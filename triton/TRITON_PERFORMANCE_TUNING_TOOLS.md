# Triton 性能调优工具指南

本文总结 Triton 仓库内可直接使用的性能调优工具，覆盖背景、功能、典型用法和实战流程。

## 1. 背景与工具分层

Triton 的性能调优通常分三层：

1. 参数搜索层：`triton.autotune` / `triton.heuristics`
2. 基准评估层：`triton.testing.do_bench` / `do_bench_cudagraph` / `perf_report`
3. 性能剖析层：`triton.profiler`（Proton）

建议把它们组合使用：

- `do_bench` 做 baseline
- `autotune` 找参数最优点
- `proton` 看瓶颈来源（kernel 级或 scope 级）

---

## 2. 工具总览

| 工具 | 主要目标 | 适用场景 | 关键能力 |
|---|---|---|---|
| `triton.autotune` | 自动选择最优 kernel 配置 | 同一 kernel 有多组 `BLOCK_SIZE/num_warps/...` 候选 | 在线 benchmark、自动选优、缓存结果 |
| `triton.heuristics` | 规则化推导元参数 | 配置可由输入形状直接推导，不想付 autotune 成本 | 运行前按规则填充 `tl.constexpr` |
| `triton.testing.do_bench` | 稳定测量 kernel 时间 | 微基准对比、回归检查 | 预热 + 重复测量 + 统计结果 |
| `triton.testing.do_bench_cudagraph` | 降低 host 开销干扰 | 小 kernel 或 launch overhead 明显 | CUDAGraph 回放测量 |
| `triton.testing.perf_report` | 结构化性能扫描与画图 | 多维参数 sweep、生成曲线图 | DataFrame/CSV/PNG 输出 |
| `triton.profiler` (Proton) | 性能剖析和路径归因 | 要看时间分布、scope、元数据、trace | scope/state、hook、backend/mode、CLI |

---

## 3. `triton.autotune` 与 `triton.heuristics`

### 3.1 背景

GPU kernel 性能通常依赖参数组合（如 tile 大小、warps/stages）。固定参数在不同输入形状或 dtype 下很难始终最优。`autotune` 通过“候选配置 + 实测”选最优，`heuristics` 通过“规则推导”避免搜索成本。

### 3.2 主要功能

`triton.autotune`（源码：`python/triton/runtime/autotuner.py`）支持：

- 多配置候选：`triton.Config(...)`
- 触发键：`key=[...]`（键变化时重新调优）
- 剪枝：
  - `early_config_prune`
  - `perf_model + top_k`
- 副作用控制：
  - `reset_to_zero`
  - `restore_value`
  - `pre_hook` / `post_hook`
- 失败回退：`OutOfResources` 等失败配置记为 `inf`，不中断整体选优
- 缓存：
  - 进程内缓存（`Autotuner.cache`）
  - 磁盘 timing 缓存（`cache_results=True` 或 `TRITON_CACHE_AUTOTUNING=1`）

`triton.heuristics` 支持：

- 按输入参数计算元参数（如 `BLOCK_SIZE = next_power_of_2(N)`）
- 适用于可规则化、无需 benchmark 搜索的场景

### 3.3 基础用法

```python
import triton
import triton.language as tl

@triton.autotune(
    configs=[
        triton.Config({"BLOCK_SIZE": 128}, num_warps=4, num_stages=2),
        triton.Config({"BLOCK_SIZE": 256}, num_warps=8, num_stages=3),
    ],
    key=["N"],  # N 变化时重新调优
)
@triton.jit
def kernel(x_ptr, y_ptr, N, BLOCK_SIZE: tl.constexpr):
    ...
```

`heuristics` 示例：

```python
@triton.heuristics(values={
    "BLOCK_SIZE": lambda args: triton.next_power_of_2(args["N"])
})
@triton.jit
def kernel(x_ptr, N, BLOCK_SIZE: tl.constexpr):
    ...
```

### 3.4 进阶用法

1. 自定义剪枝

```python
def early_prune(configs, named_args, **kwargs):
    N = kwargs["N"]
    return [c for c in configs if c.kwargs["BLOCK_SIZE"] <= N]

@triton.autotune(
    configs=[...],
    key=["N"],
    prune_configs_by={"early_config_prune": early_prune},
)
```

2. 自定义 benchmark 函数

```python
@triton.autotune(configs=[...], key=["N"], do_bench=my_bench_fn)
```

`my_bench_fn` 需要兼容签名：`fn(kernel_call, quantiles)`。

3. 缓存和可观测性

- `TRITON_PRINT_AUTOTUNING=1`：打印每次选优日志
- `TRITON_CACHE_AUTOTUNING=1`：启用 autotune timing 磁盘缓存

### 3.5 适用边界与注意事项

- `autotune` 会多次执行 kernel。若 kernel 会写输出，必须考虑 `reset_to_zero` / `restore_value`。
- `key` 设计过细会导致缓存碎片，过粗会错失最优配置。
- `top_k` 仅接受 `int` 或 `<=1.0` 的 `float`（比例）。
- `warmup/rep/use_cuda_graph` 参数在 `autotune` 中为兼容路径，已标记 deprecated。

---

## 4. 基准工具：`do_bench` / `do_bench_cudagraph` / `perf_report`

### 4.1 `do_bench`

源码：`python/triton/testing.py::do_bench`

特性：

- 自动估计单次耗时，计算 warmup/repeat 次数
- 每次测量前清 L2 cache（通过 backend driver）
- 支持 `quantiles` 或 `return_mode`（`min/max/mean/median/all`）

```python
import triton.testing as tt

ms = tt.do_bench(lambda: kernel[grid](...), warmup=25, rep=100, return_mode="mean")
p50, p20, p80 = tt.do_bench(lambda: kernel[grid](...), quantiles=[0.5, 0.2, 0.8])
```

### 4.2 `do_bench_cudagraph`

源码：`python/triton/testing.py::do_bench_cudagraph`

特性：

- 用 CUDAGraph 捕获并 replay，降低 host 侧 launch 干扰
- 对短小 kernel 或 launch-overhead 敏感场景更稳定
- NVIDIA CUDA 场景最常用

```python
ms = tt.do_bench_cudagraph(lambda: kernel[grid](...), rep=100)
```

### 4.3 `perf_report`

源码：`python/triton/testing.py::{Benchmark, perf_report}`

用途：

- 扫描多个 x 轴和 line 参数
- 自动产出 DataFrame/CSV/PNG
- 便于做算法版本对比和回归曲线

```python
import triton.testing as tt

@tt.perf_report(
    tt.Benchmark(
        x_names=["N"],
        x_vals=[2**i for i in range(10, 21)],
        line_arg="provider",
        line_vals=["triton", "torch"],
        line_names=["Triton", "Torch"],
        plot_name="vec_add",
        args={},
        ylabel="ms",
    )
)
def bench(N, provider):
    ...
    return ms, min_ms, max_ms

bench.run(print_data=True, save_path="bench_out")
```

---

## 5. Proton Profiler（`triton.profiler`）

Proton 是 Triton 的轻量 profiler，支持 kernel 路径、scope 指标、trace/tree 导出，以及 Triton launch metadata hook。

安装构建注意：

- 默认构建 Triton 时会构建 Proton。
- 若安装时设置 `TRITON_BUILD_PROTON=OFF`，`triton.profiler` 将不可用。

实现入口：

- Python API：`third_party/proton/proton/`
- CLI：`python -m triton.profiler.proton`

### 5.1 核心 API

从 `third_party/proton/proton/profile.py`、`scope.py` 可见主要接口：

- `proton.start(...)`
- `proton.activate(...)` / `proton.deactivate(...)`
- `proton.finalize(...)`
- `proton.profile(...)`（装饰器）
- `proton.scope(...)`
- `proton.cpu_timed_scope(...)`
- `proton.state(...)`

### 5.2 最小示例

```python
import triton.profiler as proton

sid = proton.start("my_profile", context="shadow", hook="triton")

with proton.scope("forward", {"bytes": 1_000_000}):
    kernel[grid](...)

proton.deactivate(sid)
proton.finalize(sid)  # 或 proton.finalize() 结束所有 session
```

### 5.3 backend / mode

`start()` 支持：

- backend：`cupti`（NVIDIA）/ `roctracer`（AMD）/ `instrumentation`
- mode：
  - `cupti`：默认或 `pcsampling`
  - `roctracer`：默认
  - `instrumentation`：可传 mode 字符串或 `triton.profiler.mode.*` 对象

`third_party/proton/proton/mode.py` 中的典型 mode 类：

- `PCSampling(interval=...)`
- `Default(...)`
- `MMA(...)`

### 5.4 命令行用法

```bash
python -m triton.profiler.proton -n profile_name script.py arg1 arg2
python -m triton.profiler.proton -n profile_name pytest -k my_test
```

命令行模式下会自动 `start/finalize`，且仅支持单 session。

### 5.5 输出与查看

- 默认常见输出：`*.hatchet`
- 查看：
  - `proton-viewer -m time/s profile.hatchet`
  - 可配合 `--print-sorted`
- `data="trace"` 可导出 trace，便于 `chrome://tracing` / perfetto 查看

### 5.6 重要环境与约束

来自 `python/triton/knobs.py::proton_knobs` 与 Proton 代码：

- `TRITON_PROTON_DISABLE`
- `TRITON_ENABLE_NVTX`
- `TRITON_PROFILE_BUFFER_SIZE`
- `TRITON_ENABLE_HW_TRACE`
- `TRITON_CUPTI_LIB_PATH`
- `TRITON_CUPTI_LIB_BLACKWELL_PATH`

AMD 注意事项（`profile.py::_check_env`）：

- 在 AMD 上使用 Proton 时，不支持 `HIP_VISIBLE_DEVICES` / `CUDA_VISIBLE_DEVICES` 组合，建议使用 `ROCR_VISIBLE_DEVICES`。

---

## 6. 与性能调优强相关的 Knobs

### 6.1 Autotuning

- `TRITON_PRINT_AUTOTUNING`
- `TRITON_CACHE_AUTOTUNING`

### 6.2 Profiling

- `TRITON_PROTON_DISABLE`
- `TRITON_ENABLE_NVTX`
- `TRITON_PROFILE_BUFFER_SIZE`
- `TRITON_ENABLE_HW_TRACE`

### 6.3 调优辅助（编译分析）

虽然不属于“自动调优工具”，但对性能定位很有用：

- `TRITON_KERNEL_DUMP`：导出编译产物便于分析
- `TRITON_KERNEL_OVERRIDE`：配合 IR override 试验替换版本

---

## 7. 推荐调优流程（可直接执行）

1. 建立 baseline
- 用 `do_bench` 测当前实现的 p50/p20/p80。

2. 做参数搜索
- 引入 `autotune`，先给一组合理候选（覆盖不同 `BLOCK_SIZE/num_warps/num_stages`）。
- 加 `prune_configs_by`，避免无效组合浪费时间。

3. 验证收益稳定性
- 用 `perf_report` 扫多组输入尺寸，确认不是单点优化。

4. 做瓶颈归因
- 用 Proton 看 scope 与 kernel 分布，确认是算力、访存还是 launch 开销主导。

5. 固化结果
- 开启 `TRITON_CACHE_AUTOTUNING`（或 `cache_results=True`）减少重复调优成本。
- 把关键 benchmark 与 profile 命令写入项目脚本。

---

## 8. 常见问题

### Q1: 什么时候用 `heuristics`，什么时候用 `autotune`？

- 元参数可由输入稳定推导：优先 `heuristics`
- 存在硬件/shape 相关性能交互：优先 `autotune`

### Q2: 为什么 autotune 后结果被“污染”？

- 你的 kernel 在调优过程中被执行多次，输出被重复写。加 `reset_to_zero` 或 `restore_value`。

### Q3: `do_bench` 和 `do_bench_cudagraph` 如何选？

- 常规场景：先 `do_bench`
- 非常短小 kernel、launch 开销占比高：尝试 `do_bench_cudagraph`

### Q4: Proton 和 Nsight 的关系？

- Proton 更轻量、可移植性更好，适合日常迭代与 Triton 元数据关联分析。
- Nsight 更偏深度平台分析（尤其 NVIDIA 指标细节）。

---

## 9. 源码索引（便于进一步阅读）

- `python/triton/runtime/autotuner.py`
- `python/triton/testing.py`
- `python/triton/knobs.py`
- `third_party/proton/proton/profile.py`
- `third_party/proton/proton/scope.py`
- `third_party/proton/proton/mode.py`
- `third_party/proton/README.md`
