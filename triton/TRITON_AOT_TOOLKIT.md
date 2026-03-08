# Triton AOT 工具包详细说明（源码对照版）

本文基于当前仓库源码，系统说明 Triton 的 AOT（Ahead-Of-Time）工具包：背景、组成、工作流、参数语义、生成产物、限制与排障。

---

## 1. 背景与定位

Triton 默认工作模式是 JIT（运行时编译并缓存）。AOT 工具包提供另一条路径：

1. 在构建阶段离线编译 Triton kernel，生成可直接集成的 C 源码与头文件。
2. 把多个 specialization 产物链接成统一入口，运行时只做轻量分发，不再依赖 Python 端编译。

AOT 典型适用场景：
- 需要将 Triton kernel 交付给纯 C/C++ 运行时。
- 需要减少线上首次请求的编译延迟。
- 需要对 kernel 版本进行静态封装和发布。

核心工具：
- `python/triton/tools/compile.py`：把 `@triton.jit` kernel 编译为后端二进制 + C stub。
- `python/triton/tools/link.py`：把多个 compile 产物链接成统一 dispatcher。

---

## 2. 工具包组成

## 2.1 Python 入口

- `python/triton/tools/compile.py`
- `python/triton/tools/link.py`

它们是脚本文件，不是 `console_scripts` 入口（`setup.py` 里没有为 compile/link 注册命令）。

## 2.2 后端模板

AOT 生成代码依赖模板文件：

- CUDA：
  - `third_party/nvidia/tools/cuda/compile.c`
  - `third_party/nvidia/tools/cuda/compile.h`
  - `third_party/nvidia/tools/cuda/link.h`
- HIP：
  - `third_party/amd/tools/hip/compile.c`
  - `third_party/amd/tools/hip/compile.h`
  - `third_party/amd/tools/hip/link.h`

运行 `compile.py/link.py` 时，代码读取路径是 `triton.tools/extra/<backend>`。
该路径由安装/开发模式下的 `setup.py` 链接或打包逻辑提供（见 `setup.py` 的 `get_package_dirs/get_packages/add_link_to_backends`）。

---

## 3. 端到端工作流

## 3.1 单个 kernel AOT 编译（compile.py）

流程：

1. 执行用户传入的 Python 文件，取出 `--kernel-name` 对应的 `@triton.jit` 对象。
2. 解析 `--signature` 与 `--grid`。
3. 将 `signature` 中常量/提示（hint）转换为 `constexpr` 和属性（如 `tt.divisibility = 16`）。
4. 通过 `triton.compile(...)` 调用标准编译流水线，拿到 `CompiledKernel`。
5. 提取后端二进制（`cubin` 或 `hsaco`），编码成十六进制字节数组写入 C 模板。
6. 生成每个 specialization 对应的一组 `*.c/*.h` 文件。

## 3.2 多 specialization 链接（link.py）

流程：

1. 读取所有 `compile.py` 产出的头文件。
2. 解析 `// tt-linker:` 与 `// tt-linker-backend:` 注释元数据。
3. 按“kernel + algo 信息”聚合 specialization。
4. 生成统一 dispatcher：
- `_default` 入口（默认 algo_id=0）
- 带 `algo_id` 的入口
- `load_/unload_` 聚合入口
- `get_num_algos` 查询接口

---

## 4. compile.py 详解

## 4.1 CLI 参数

关键参数：

- `path`：包含 kernel 的 Python 文件（会被执行）。
- `--kernel-name/-n`：函数名。
- `--signature/-s`：签名字符串。
- `--grid/-g`：网格表达式，必须是 3 段（`x,y,z`）。
- `--target/-t`：`<backend>:<arch>:<warp-size>`，如 `cuda:80:32`、`hip:gfx942:64`。
- `--num-warps/-w`、`--num-stages/-ns`：meta 参数。
- `--out-name/-on`、`--out-path/-o`：输出基名和路径。

## 4.2 signature 语法

`--signature` 逐参数逗号分隔，支持三种语义：

1. 普通类型参数：如 `*fp16`, `i32`
2. 带 hint 参数：`type:16` 或 `type:1`
- `:16` 表示 16 字节整除提示（用于 dispatch 条件和 `tt.divisibility`）
- `:1` 表示值恒等于 1（作为常量特化）
3. 常量参数：可解析成 `int`/`float` 的 token（例如 `1024`）
- 会转为 `constexpr`
- 不进入生成的 C 原型参数列表

当前实现只接受 hint 值 `{1,16}`，否则断言失败。

## 4.3 常量与属性转换

- 常量通过 `ASTSource(..., constexprs=...)` 进入编译。
- `:16` 会注入 `attrs={(i,): [["tt.divisibility", 16]]}`。
- `:1` 会被加入 constants，并在 stub 参数里按 `i32` 常量约束处理。

## 4.4 产物命名规则

每个 specialization 的函数名：

`<out_name>_<sig_hash>_<suffix>`

其中：
- `sig_hash` = `sha256(signature + meta_sig)[:8]`
- `meta_sig` = `warps{num_warps}xstages{num_stages}`
- `suffix` 用于编码 hint：
  - `id`：第 i 个参数有 `:16`
  - `ic`：第 i 个参数有 `:1`

输出文件名模式：

`<out_path>.<sig_hash>_<suffix>.c/.h`

## 4.5 关键限制

`compile.py` 显式拒绝以下 kernel：

- `global_scratch_size > 0`
- `profile_scratch_size > 0`

即：当前 AOT 产物尚不支持带 global/profile scratch 的 kernel。

## 4.6 生成 C stub 的核心行为

- 把 `cubin/hsaco` 内联为 `unsigned char[]`。
- 暴露 `load_<kernel>()` / `unload_<kernel>()`。
- launcher 会按 `gridX/gridY/gridZ` 和 `num_warps * warp_size` 调用底层驱动 API：
  - CUDA：`cuLaunchKernel`
  - HIP：`hipModuleLaunchKernel`

---

## 5. link.py 详解

## 5.1 元信息来源

`compile.h` 中含两类注释：

- `// tt-linker-backend: <backend>`
- `// tt-linker: <kernel_name>:<full_signature>:<algo_info>`

`link.py` 通过正则提取它们，并校验：

- 所有输入头文件 backend 一致。
- 同一逻辑 kernel 的 C 参数类型签名一致。
- suffix 格式合法（仅 index + `c/d`）。

## 5.2 分发策略

link 生成的分发逻辑分两层：

1. hint 分发（对齐 `:1/:16`）
- `:16` 生成条件：`((uintptr_t)arg % 16 == 0)`
- `:1` 生成条件：`(arg == 1)`
- 条件按“特化参数数量降序”尝试，优先更具体 specialization。
- 全不匹配返回 `TT_ERROR_INVALID_VALUE`。

2. algo 分发（对齐多组 tile/meta）
- 生成 `kernel_func_t kernel_kernels[]` 表。
- `kernel(..., int algo_id)` 根据索引调用。
- 生成 `kernel_default(...)` 等价 `algo_id=0`。

同时生成：
- `load_kernel()/unload_kernel()`：批量加载/卸载所有 specialization。
- `kernel_get_num_algos()`：查询 algo 数量。

## 5.3 `--prefix` 现状

`link.py` CLI 定义了 `--prefix` 参数，但当前生成逻辑未使用该参数。属于预留/未接线状态。

---

## 6. CUDA 与 HIP 差异

## 6.1 类型映射差异

`compile.py` 生成参数类型时调用 `driver.active.map_python_to_cpp_type`，映射定义在各后端 `driver.py`：

- CUDA 指针类型映射为 `CUdeviceptr`，descriptor 映射为 `CUtensorMap`。
- HIP 指针类型映射为 `hipDeviceptr_t`，descriptor 映射为 `TDMDescriptor`。

注意：标量 `fp16/bf16/fp32/fp64` 在当前映射里都转为 `double`（与 launcher 参数抽取实现有关），这是现有实现行为。

## 6.2 模板差异

- CUDA 模板在 `load_` 中包含 dynamic shared memory opt-in 处理（`CU_FUNC_ATTRIBUTE_MAX_DYNAMIC_SHARED_SIZE_BYTES`）。
- HIP 模板返回类型是 `hipError_t`，无 CUDA 对应的 shared opt-in 分支。

---

## 7. 与 Triton 编译核心的关系

AOT 工具不是独立编译器，而是复用 `triton.compile` 主流程：

1. `compile.py` 构造 `ASTSource`。
2. 调用 `triton.compile(src, target, options)`。
3. 通过 `CompiledKernel.asm[binary_ext]` 读取最终后端二进制。

因此 AOT 的正确性和性能与 JIT 编译管线一致，区别主要在于“产物封装形式”和“部署时机”。

---

## 8. 用法示例（源码测试对照）

## 8.1 编译多个 specialization

示例（概念化）：

```bash
python python/triton/tools/compile.py \
  -n kernel \
  -s "*fp32:16,*fp16:16,*fp16:16,i32,i32,i32,i32:16,i32:1,i32:16,i32:1,i32:16,i32:1,16,16,16" \
  -g "M/16,N/16,1" \
  -w 1 \
  -o matmul_fp16 \
  /path/to/kernel.py
```

## 8.2 链接为统一入口

```bash
python python/triton/tools/link.py /tmp/aot/*.h -o kernel
```

会生成：
- `kernel.h`
- `kernel.c`

并提供：
- `matmul_fp16_default(...)`
- `matmul_fp16(..., algo_id)`
- `matmul_fp16_get_num_algos()`
- `load_matmul_fp16()/unload_matmul_fp16()`

## 8.3 algo 调度

测试用例展示了两种调用：
- 默认路径：`*_default(...)`
- 指定路径：`*(..., algo_id)`

并通过多组 tile size 编译后，用不同 `algo_id` 验证结果一致性。

---

## 9. 常见坑与约束

1. `compile.py` 会执行输入 Python 文件
- 该文件有副作用时会在编译阶段触发。

2. `grid` 必须是三元表达式
- 只做 `split(',')` + `len==3` 校验。
- 表达式文本会直接写入生成 C 代码。

3. hint 仅支持 `1` 和 `16`
- 其他值会触发断言失败。

4. 目前不支持 scratch 型 kernel 的 AOT 封装
- 会抛 `RuntimeError`。

5. 模板路径依赖安装/链接状态
- 若 `triton.tools.extra/<backend>` 不存在，`compile.py/link.py` 无法读取模板。
- 在源码开发环境通常通过 `setup.py` 的 `add_links` 逻辑建立软链。

6. `link.py --prefix` 当前无效
- 参数存在但未参与输出命名。

7. 文案与文件名存在历史不一致
- `compile.py` 描述里提到 `linker.py`，实际脚本文件名是 `link.py`。

---

## 10. 源码索引（快速定位）

## AOT 主入口

- `python/triton/tools/compile.py:30-54`（AOT 编译说明）
- `python/triton/tools/compile.py:57-77`（CLI 定义）
- `python/triton/tools/compile.py:80-147`（签名/常量/编译与限制）
- `python/triton/tools/compile.py:163-207`（stub 生成与模板渲染）

- `python/triton/tools/link.py:29-133`（头文件解析与校验）
- `python/triton/tools/link.py:173-211`（hint dispatcher 生成）
- `python/triton/tools/link.py:214-254`（algo 分发与计数接口）
- `python/triton/tools/link.py:268-335`（link 主流程）

## 模板与注释协议

- `third_party/nvidia/tools/cuda/compile.h:11-16`
- `third_party/amd/tools/hip/compile.h:13-18`
- `third_party/nvidia/tools/cuda/compile.c:57-69`
- `third_party/amd/tools/hip/compile.c:53-67`

## 类型映射

- `third_party/nvidia/backend/driver.py:98-120`
- `third_party/amd/backend/driver.py:199-220`

## 编译核心复用链路

- `python/triton/compiler/compiler.py:52-84`（`ASTSource`）
- `python/triton/compiler/compiler.py:226-363`（`triton.compile` 主流程）
- `python/triton/compiler/compiler.py:407-431`（`CompiledKernel` 与 `asm`）

## 安装/打包模板注入

- `setup.py:392-397`, `415-419`（将后端 tools 安装到 `triton.tools.extra`）
- `setup.py:442-449`（开发态软链接到 `python/triton/tools/extra`）

## 测试用例（真实工作流）

- `python/test/unit/tools/test_aot.py:307-317`（调用 compile.py）
- `python/test/unit/tools/test_aot.py:358-363`（调用 link.py）
- `python/test/unit/tools/test_aot.py:391-543`（编译、链接、执行、algo_id 验证）
- `python/test/unit/tools/test_aot.py:459-494`（无匹配 specialization 的失败场景）

---

## 11. 结论

Triton AOT 工具包本质是“编译封装器 + specialization 链接器”：

- `compile.py` 负责把单个 kernel specialization 固化为 C 可集成产物。
- `link.py` 负责把多产物聚合为统一 API（含 hint 分发 + algo 分发）。
- 整体复用 Triton 现有后端编译流水线，因此与 JIT 在生成质量上保持一致。

当前最关键的工程约束是：scratch kernel 支持缺失、模板路径依赖安装态、`--prefix` 未接线。若要把 AOT 用于生产规模部署，通常先围绕这三点做增强。
