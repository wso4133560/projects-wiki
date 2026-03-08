# Triton Autotune 原理（源码解析）

本文面向 Triton 开发者，基于当前仓库实现解释 `triton.autotune` 的运行机制、关键数据结构、缓存与失败回退路径。

- 主要源码：`python/triton/runtime/autotuner.py`
- 关联接口：`python/triton/runtime/jit.py`, `python/triton/compiler/compiler.py`
- 行为验证：`python/test/unit/runtime/test_autotuner.py`

## 1. 入口与对象模型

典型写法：

```python
@triton.autotune(configs=[...], key=[...])
@triton.jit
def kernel(...):
    ...
```

装饰器执行顺序是自下而上：

1. `@triton.jit` 先把 Python 函数包装成 `JITFunction`。
2. `@triton.autotune` 再把 `JITFunction` 包装成 `Autotuner`（`KernelInterface` 子类）。

因此最终调用 `kernel[grid](...)` 时，实际先进入 `KernelInterface.__getitem__` 返回的 launcher，再调用 `Autotuner.run(grid=..., warmup=False, ...)`。

源码锚点：

- `python/triton/runtime/autotuner.py::autotune`（decorator 返回 `Autotuner`）
- `python/triton/runtime/jit.py::KernelInterface.__getitem__`
- `python/triton/runtime/autotuner.py::Autotuner.run`

## 2. `Config` 是什么

`Config` 表示一个待比较的候选配置，包含两类信息：

- Kernel 元参数：`kwargs`（例如 `BLOCK_SIZE_M`, `BLOCK_SIZE_N`）
- 编译/调度相关参数：`num_warps`, `num_stages`, `num_ctas`, `maxnreg`, `ir_override`

`Config.all_kwargs()` 会把上述参数合并为传给 `JITFunction.run` 的关键字参数。

等价性与哈希：

- `__hash__` / `__eq__` 基于 `all_kwargs()` + `pre_hook`
- 这决定了 `Config` 作为字典 key（timing map）的行为

补充：

- 如果用户传入 `configs=[]`，`Autotuner` 会注入默认配置
  `Config({}, num_warps=4, num_stages=3, num_ctas=1)`。

源码锚点：

- `python/triton/runtime/autotuner.py::class Config`
- `python/triton/runtime/autotuner.py::Autotuner.__init__`

## 3. 运行时主流程（`Autotuner.run`）

当 `len(configs) > 1` 时，`run` 的逻辑可概括为：

```text
构造 tuning key
  -> 命中进程内 cache ? 直接用 best config
  -> 否则 prune configs
      -> benchmark 每个候选
      -> 选 timing 最小配置
      -> 对输出做一次 reset_only pre_hook（防止 bench 副作用残留）
      -> 写入进程内 cache
  -> 用 best config 正式执行一次 kernel
```

### 3.1 tuning key 构造

key 由两部分拼成 tuple：

1. `autotune(..., key=[...])` 指定的参数值（只取 `arg_names` 中存在的项）
2. 所有 kernel 实参中带 `dtype` 属性的对象，其 `str(arg.dtype)`（通常是 tensor dtype）

这意味着：即使 `key` 不显式包含 dtype，dtype 变化也会触发重新调优。

源码锚点：

- `python/triton/runtime/autotuner.py::Autotuner.run`（`key.append(str(arg.dtype))`）
- 单测：`python/test/unit/runtime/test_autotuner.py::test_kwargs`

### 3.2 bench + 选优

对每个候选 `config`：

1. `_bench` 先检查冲突：如果 launch kwargs 与 `config.kwargs` 同名，抛 `ValueError`。
2. 调用链中 hook 顺序：
   - `config.pre_hook(full_nargs)`（若配置自带 pre_hook）
   - `self.pre_hook(full_nargs)`（用户或默认 pre_hook）
   - 执行 `self.fn.run(...)`
   - `self.post_hook(full_nargs, exception=...)`
3. benchmark 函数返回 `(p50, p20, p80)`，`min(timings, key=timings.get)` 选最小（按 tuple 字典序，先比 p50）。

默认 benchmark 来源：

- 若没传 `do_bench`，使用 `driver.active.get_benchmarker()`。
- CUDA/HIP 后端默认返回 `triton.testing.do_bench`。

源码锚点：

- `python/triton/runtime/autotuner.py::Autotuner._bench`
- `python/triton/runtime/autotuner.py::Autotuner.do_bench`
- `third_party/nvidia/backend/driver.py::CudaDriver.get_benchmarker`
- `third_party/amd/backend/driver.py::HIPDriver.get_benchmarker`

### 3.3 正式执行

best config 选出后：

- 保存到 `self.best_config`
- 执行一次 `self.fn.run(*args, **kwargs, **best_config.all_kwargs())`
- 不再重复 benchmark

若 `configs` 只有一个，则跳过调优流程，直接执行该配置。

## 4. 配置裁剪：`prune_configs`

裁剪分两层，顺序固定：

1. `early_config_prune(configs, named_args, **kwargs)`：先做规则过滤
2. `perf_model` + `top_k`：再按模型估计时间排序取前 k

`top_k` 规则：

- `int`：直接使用
- `float <= 1.0`：转成 `int(len(configs) * top_k)`
- 其他类型：抛 `TypeError`

若 `early_config_prune` 返回空列表，抛 `AutotunerError`（要求至少保留一个配置）。

源码锚点：

- `python/triton/runtime/autotuner.py::Autotuner.prune_configs`
- 单测：`test_prune_configs`, `test_prune_all_configs`

## 5. Hook 与“多次执行副作用”控制

autotune 本质上要运行多个候选配置，可能重复写输出。框架提供两类保护：

1. `reset_to_zero=[...]`
2. `restore_value=[...]`

当用户未自定义 hook 时：

- 默认 pre_hook：
  - 每次执行前先对 `reset_to_zero` 指定 tensor 执行 `zero_()`
  - 非 `reset_only` 模式下，为 `restore_value` 做 `clone()`
- 默认 post_hook：
  - 将 `restore_value` tensor 回写 clone 备份

另外，在选出最佳配置后，会额外调用一次
`self.pre_hook(full_nargs, reset_only=True)`，清理 benchmark 过程可能留下的写入。

源码锚点：

- `python/triton/runtime/autotuner.py::Autotuner.__init__`（默认 hook 构造）
- `python/triton/runtime/autotuner.py::Autotuner.run`（`reset_only=True`）
- 单测：`test_restore`, `test_hooks`

## 6. 异常与失败回退策略

`_bench` 对以下异常采用“软失败”：

- `OutOfResources`
- `CompileTimeAssertionFailure`
- `PTXASError`

处理方式：该配置 timing 记为 `[inf, inf, inf]`，继续比较其他配置。

这样一个候选编译/资源失败，不会直接终止整个 autotune（只要还有可用配置）。

注意：其他异常不会在 `_bench` 外层被吞掉，会向上抛出。

源码锚点：

- `python/triton/runtime/autotuner.py::Autotuner._bench`
- `python/triton/runtime/errors.py`
- 单测：`test_exceed_tmem`, `test_exceed_threads`

## 7. 两级缓存：内存缓存 + 磁盘 timing 缓存

### 7.1 进程内缓存

- 结构：`self.cache: Dict[Tuple, Config]`
- key：本次运行构造的 tuning key
- value：最优 `Config`

命中后直接走正式执行，不再 benchmark。

### 7.2 磁盘缓存（可选）

启用条件：

- `autotune(..., cache_results=True)` 或 `TRITON_CACHE_AUTOTUNING=1`
- 且非 interpreter 模式

`check_disk_cache` 逻辑：

1. 先构造磁盘缓存 key（SHA256），组成包括：
   - `triton_key()`（Triton 代码版本指纹）
   - backend hash
   - `fn.cache_key`
   - cache-invalidating 环境变量集合
   - tuning key
   - 所有候选配置字符串
2. 命中文件 `<fn>.autotune.json` 时，读取 `configs_timings`，恢复 timing map，直接选最优配置。
3. 未命中则执行 benchmark，并把 `configs_timings` 写入磁盘。

限制：

- 只要任一配置含 `pre_hook`，就放弃磁盘缓存（hook 不可序列化）。

源码锚点：

- `python/triton/runtime/autotuner.py::Autotuner.check_disk_cache`
- `python/triton/runtime/cache.py::get_cache_manager`, `triton_key`
- `python/triton/knobs.py::autotuning_knobs`

## 8. `ir_override` 如何接入 autotune

`Config` 支持 `ir_override` 字段，把某个候选配置绑定到外部 IR 文件（`.ttir/.ttgir/.ptx/.amdgcn/.llir`）。

生效路径：

1. `Config.all_kwargs()` 把 `ir_override` 注入 launch kwargs
2. 编译 pipeline 每个阶段检查 metadata 中的 `ir_override`
3. 当扩展名匹配当前阶段时，替换当前 IR 模块继续下游编译

因此 `ir_override` 可以作为 autotune 候选维度的一部分（例如某个配置走自定义 IR）。

源码锚点：

- `python/triton/runtime/autotuner.py::Config.all_kwargs`
- `python/triton/compiler/compiler.py`（注释 *Users can override kernels at scale by setting `ir_override` in autotune config*）
- 单测：`test_override_ttir`, `test_override_ttgir`, `test_override_ptx`

## 9. 兼容与行为细节

- `warmup`, `rep`, `use_cuda_graph` 参数已标记 deprecated，但仍保留兼容路径（会发 `DeprecationWarning` 并走旧 benchmark 接口）。
- `TRITON_PRINT_AUTOTUNING=1` 会打印每次调优耗时和 best config。
- `self.best_config` 提供最近一次运行选出的配置，便于调试观测。

源码锚点：

- `python/triton/runtime/autotuner.py::Autotuner.__init__`
- `python/triton/runtime/autotuner.py::Autotuner.run`
- `python/triton/knobs.py::autotuning_knobs`

## 10. 开发者实践建议（基于实现约束）

1. `key` 只放真正会改变最优配置的维度，避免 key 爆炸。
2. 优先用 `early_config_prune` 过滤明显非法/无效配置，再用 `perf_model + top_k` 做精筛。
3. 有写输出副作用的 kernel 必须配 `reset_to_zero` 或 `restore_value`，否则 benchmark 会污染结果。
4. 若需要跨进程复用调优结果，启用 `cache_results`（或 `TRITON_CACHE_AUTOTUNING`），并避免在 config 上挂不可序列化 pre_hook。
5. 遇到单配置 out-of-resource 时，优先减 `BLOCK_*` 或 `num_stages`，保留至少一个可行候选。

