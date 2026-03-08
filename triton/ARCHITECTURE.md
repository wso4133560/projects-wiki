# Triton 编译器架构文档

## 一、概述

### 项目定位与核心价值

Triton 是一个基于 MLIR 的 GPU 编译器框架，允许 Python 程序员以接近硬件峰值性能编写 GPU kernel，同时无需深入掌握 CUDA/HIP 底层细节。其核心价值在于：

- **生产力**：用 Python 描述 tile 级并行算法，编译器自动处理内存层次、线程映射、指令选择
- **可移植性**：同一 kernel 源码可编译到 NVIDIA（CUDA）和 AMD（HIP/ROCm）GPU
- **性能**：通过多级 IR 优化（布局分析、软件流水线、warp 特化）逼近手写 CUDA 性能

### 整体分层架构

```
┌─────────────────────────────────────────────────────────┐
│                   Python 前端                            │
│  @triton.jit  →  JITFunction  →  DependenciesFinder     │
│  triton.language (core.py / semantic.py)                 │
└────────────────────────┬────────────────────────────────┘
                         │ ast_to_ttir()
┌────────────────────────▼────────────────────────────────┐
│              Triton IR (TTIR)  —  tt.* 方言              │
│  硬件无关的 tile 级操作：load/store/dot/reduce/scan       │
└────────────────────────┬────────────────────────────────┘
                         │ TritonToTritonGPU pass
┌────────────────────────▼────────────────────────────────┐
│         TritonGPU IR (TTGIR)  —  ttg.* 方言              │
│  布局感知：blocked/mma/shared encoding + 37 个优化 Pass  │
└──────────┬─────────────────────────────┬────────────────┘
           │ NVIDIA 后端                  │ AMD 后端
┌──────────▼──────────┐       ┌──────────▼──────────────┐
│  ttnvgpu.* 方言      │       │  ttg.amd_mfma/wmma 编码  │
│  TMA/WGMMA/Cluster  │       │  Buffer ops / Pingpong   │
└──────────┬──────────┘       └──────────┬──────────────┘
           │ ToLLVM                       │ ToLLVM
┌──────────▼──────────┐       ┌──────────▼──────────────┐
│     LLVM IR (PTX)   │       │   LLVM IR (AMDGCN)      │
└──────────┬──────────┘       └──────────┬──────────────┘
           │ ptxas                        │ rocm-llvm
┌──────────▼──────────┐       ┌──────────▼──────────────┐
│       CUBIN          │       │         HSACO            │
└─────────────────────┘       └─────────────────────────┘
```

### 顶层目录结构

```
triton/
├── python/triton/          # Python 前端与运行时
│   ├── runtime/jit.py      # JITFunction、DependenciesFinder
│   ├── compiler/           # compiler.py、code_generator.py
│   ├── language/           # core.py、semantic.py
│   └── backends/compiler.py# BaseBackend、GPUTarget 抽象
├── include/triton/Dialect/
│   ├── Triton/IR/          # tt.* 方言定义（.td 文件）
│   └── TritonGPU/IR/       # ttg.* 方言定义（.td 文件）
├── lib/
│   ├── Conversion/TritonToTritonGPU/   # TTIR → TTGIR 转换
│   └── Conversion/TritonGPUToLLVM/    # TTGIR → LLVM 公共层
├── third_party/
│   ├── nvidia/             # CUDA 后端（compiler.py + C++ passes）
│   └── amd/                # HIP 后端（compiler.py + C++ passes）
└── python/src/             # pybind11 绑定（ir.cc、passes.cc、llvm.cc）
```

---

## 二、编译流水线全景

### 端到端流程

```
@triton.jit Python 函数
  │
  ├─ DependenciesFinder（AST 遍历 + SHA256 缓存键）
  │    遍历函数 AST，递归哈希所有被调用的 JITFunction 源码
  │
  ├─ ASTSource.make_ir() → ast_to_ttir()
  │    Python AST → Triton IR (TTIR, .ttir)
  │
  ├─ make_ttir()  [TTIR 清理 pass]
  │    inliner → rewrite_tensor_pointer → canonicalizer
  │    → combine → reorder_broadcast → cse → symbol_dce → loop_unroll
  │
  ├─ make_ttgir() [TritonToTritonGPU + 优化 pass 序列]
  │    convert_to_ttgpuir → coalesce → accelerate_matmul
  │    → pipeline → remove_layout_conversions → reorder_instructions
  │    → (NVIDIA: TMA lowering / AMD: buffer ops)
  │
  ├─ make_llir()  [TTGIR → LLVM IR]
  │    allocate_shared_memory → to_llvmir → nvgpu_to_llvm / amd_to_llvm
  │    → llvm.to_module() → llvm.optimize_module(O3)
  │
  ├─ make_ptx() / make_amdgcn()  [LLVM IR → 汇编]
  │    llvm.translate_to_asm()
  │
  └─ make_cubin() / make_hsaco()  [汇编 → 二进制]
       ptxas subprocess / amd.assemble_amdgcn()
```

### 多级缓存机制

缓存键由以下部分组成（`get_cache_key()`）：

- `src.hash()`：源码 SHA256（`DependenciesFinder` 递归计算所有依赖）
- `backend.hash()`：后端标识（NVIDIA: ptxas 版本 + arch；AMD: HIPOptions 字段哈希）
- `options.__dict__`：编译选项（num_warps、num_stages、arch 等）
- `env_vars`：影响编译的环境变量

缓存命中时直接返回 `CompiledKernel`，跳过所有编译阶段。缓存目录默认为 `~/.triton/cache/<hash>/`，存储各阶段 IR 文件（`.ttir`、`.ttgir`、`.llir`、`.ptx`/`.amdgcn`、`.cubin`/`.hsaco`）及元数据 JSON。

### `BaseBackend.add_stages()` 的 stages 字典模型

`stages` 是一个有序字典，键为 IR 扩展名，值为转换函数：

```python
# NVIDIA 后端（compiler.py:558-565）
stages["ttir"]  = lambda src, metadata: self.make_ttir(src, metadata, options, capability)
stages["ttgir"] = lambda src, metadata: self.make_ttgir(src, metadata, options, capability)
stages["llir"]  = lambda src, metadata: self.make_llir(src, metadata, options, capability)
stages["ptx"]   = lambda src, metadata: self.make_ptx(src, metadata, options, capability)
stages["cubin"] = lambda src, metadata: self.make_cubin(src, metadata, options, capability)

# AMD 后端（compiler.py:534-541）
stages["ttir"]   = lambda src, metadata: self.make_ttir(src, metadata, options)
stages["ttgir"]  = lambda src, metadata: self.make_ttgir(src, metadata, options)
stages["llir"]   = lambda src, metadata: self.make_llir(src, metadata, options)
stages["amdgcn"] = lambda src, metadata: self.make_amdgcn(src, metadata, options)
stages["hsaco"]  = lambda src, metadata: self.make_hsaco(src, metadata, options)
```

`compiler.compile()` 按插入顺序依次执行各阶段，每阶段接收上一阶段输出，并可向 `metadata` 字典写入信息（如 `shared`、`num_warps`、`name`）。`IRSource` 输入时从对应阶段之后开始执行，支持从任意 IR 层级介入编译。

---

## 三、Python 前端

### `@triton.jit` 装饰器

`python/triton/runtime/jit.py`

`@triton.jit` 将 Python 函数包装为 `JITFunction`。核心组件：

- **`JITFunction`**：持有原始 Python 函数引用、参数名列表、`cache_key`（SHA256）。调用时触发编译或缓存查找，最终通过 `CompiledKernel` 启动 GPU kernel。
- **`JITCallable`**：`JITFunction` 的基类，提供 `cache_key` 属性。
- **`DependenciesFinder`**（`jit.py:34`）：`ast.NodeVisitor` 子类，遍历函数 AST，递归哈希所有被调用的 `JITFunction` 源码（`hashlib.sha256`），并追踪全局变量引用。支持的内置模块：`triton.language`、`triton.experimental.gluon.language`、`copy`、`math`。

### 类型系统与语言核心（`triton.language`）

`python/triton/language/core.py`

| 类型分类 | 示例 |
|---------|------|
| 标量整数 | `int1`、`int8`、`int16`、`int32`、`int64`、`uint8`…`uint64` |
| 标量浮点 | `float16`、`bfloat16`、`float32`、`float64`、`float8e4nv`… |
| 指针类型 | `pointer_type(element_ty)` |
| Tensor 类型 | `block_type(element_ty, shape)` |

核心内置操作分类：

- **内存**：`load`、`store`、`atomic_add`、`atomic_cas`、`make_block_ptr`
- **算术**：`dot`、`sum`、`max`、`min`、`xor_sum`（归约）
- **形状**：`reshape`、`broadcast`、`trans`、`expand_dims`、`cat`
- **程序控制**：`program_id`、`num_programs`、`multiple_of`、`max_contiguous`
- **调试**：`static_print`、`device_print`、`static_assert`

### 语义分析层

`python/triton/language/semantic.py`

`semantic.py` 是 Python AST 到 TTIR 的翻译核心，职责：

- 类型推断与隐式类型转换（整数提升、指针算术）
- 将 `triton.language` 内置函数调用映射到对应的 MLIR 操作构造
- 维护编译时常量（`constexpr`）的折叠与传播
- 处理 `if/for/while` 控制流到 MLIR `scf` 方言的映射

### 编译入口

`python/triton/compiler/compiler.py`

- **`ASTSource`**（`compiler.py:52`）：封装 `JITFunction`，`make_ir()` 调用 `ast_to_ttir()` 生成初始 TTIR 模块。`hash()` 基于函数 `cache_key`、签名、常量、属性计算。
- **`IRSource`**（`compiler.py:87`）：从磁盘加载已有 IR 文件（`.ttir`/`.ttgir`/`.llir`/`.ptx`），支持从任意阶段介入编译流水线。
- **`CompiledKernel`**：持有编译产物路径和元数据，提供 kernel 启动接口。

---

## 四、MLIR 方言栈

### 4.1 Triton 方言（`tt.*`）

`include/triton/Dialect/Triton/IR/`

**类型系统**：

- `tt.ptr<T>`：指向类型 `T` 的指针，可以是标量指针或 tensor of pointers
- `RankedTensorType + encoding`：带布局编码的 ranked tensor，编码在 TTIR 阶段为空（硬件无关）

**核心操作分类**（`TritonOps.td`）：

| 分类 | 操作 |
|------|------|
| 类型转换 | `tt.int_to_ptr`、`tt.ptr_to_int`、`tt.bitcast`、`tt.fp_to_fp` |
| 算术 | `tt.clampf`、`tt.precise_sqrt`、`tt.precise_divf`、`tt.mulhiui` |
| 内存 | `tt.load`、`tt.store`、`tt.atomic_rmw`、`tt.atomic_cas` |
| 形状 | `tt.reshape`、`tt.broadcast`、`tt.trans`、`tt.expand_dims`、`tt.cat`、`tt.join`、`tt.split` |
| 计算 | `tt.dot`、`tt.dot_scaled`、`tt.reduce`、`tt.scan`、`tt.histogram` |
| 程序控制 | `tt.get_program_id`、`tt.get_num_programs`、`tt.call`、`tt.func`、`tt.return` |
| TMA（描述符） | `tt.make_tensor_descriptor`、`tt.reinterpret_tensor_descriptor`、`tt.descriptor_load`、`tt.descriptor_store`、`tt.descriptor_gather`、`tt.descriptor_scatter` |
| 调试 | `tt.print`、`tt.assert` |

**Traits 与 Interfaces**：

- `Elementwise`：逐元素操作，保持 shape 和 encoding
- `SameOperandsAndResultEncoding`：操作数与结果共享 encoding
- `TensorSizeTrait`、`VerifyTensorLayoutsTrait`：所有 `TT_Op` 自动附加的验证 trait
- `GlobalMemory`：标记全局内存访问的 resource

### 4.2 TritonGPU 方言（`ttg.*`）

`include/triton/Dialect/TritonGPU/IR/`

TritonGPU 方言的核心创新是**布局编码系统**（Layout Encoding），通过 tensor 的 encoding attribute 精确描述数据在 GPU 线程间的分布方式。

**分布式编码（Distributed Encoding）**：

| 编码 | 助记符 | 用途 |
|------|--------|------|
| `BlockedEncodingAttr` | `#ttg.blocked` | 默认分块布局，参数：`sizePerThread`、`threadsPerWarp`、`warpsPerCTA`、`order` |
| `LinearEncodingAttr` | `#ttg.linear` | 基于线性布局的通用编码，统一描述任意数据重排 |
| `SliceEncodingAttr` | `#ttg.slice` | 从高维 encoding 切片降维，用于 reduce 输出 |
| `DotOperandEncodingAttr` | `#ttg.dot_op` | dot 操作数专用，携带 `opIdx`（0=A, 1=B）和 `kWidth` |
| `NvidiaMmaEncodingAttr` | `#ttg.nvidia_mma` | NVIDIA MMA 指令布局（Ampere/Hopper WGMMA） |
| `AMDMfmaEncodingAttr` | `#ttg.amd_mfma` | AMD MFMA 矩阵指令布局 |
| `AMDWmmaEncodingAttr` | `#ttg.amd_wmma` | AMD WMMA 指令布局 |

**共享内存编码（Shared Memory Encoding）**：

| 编码 | 助记符 | 用途 |
|------|--------|------|
| `SwizzledSharedEncodingAttr` | `#ttg.swizzled_shared` | XOR swizzle 消除 bank conflict，参数：`vec`、`perPhase`、`maxPhase`、`order` |
| `PaddedSharedEncodingAttr` | `#ttg.padded_shared` | padding + 线性重映射消除 bank conflict |
| `NvidiaMmaSharedEncodingAttr` | `#ttg.nvmma_shared` | Hopper TMA/WGMMA 专用共享内存布局 |

**核心操作**：

| 操作 | 说明 |
|------|------|
| `ttg.convert_layout` | 在不同 encoding 间转换，可能产生 shared memory 中转 |
| `ttg.local_alloc` | 在共享内存中分配 tensor |
| `ttg.local_load` | 从共享内存加载到寄存器 |
| `ttg.local_store` | 从寄存器写入共享内存 |
| `ttg.async_copy_global_to_local` | 异步全局内存 → 共享内存拷贝 |
| `ttg.async_commit_group` / `ttg.async_wait` | 异步拷贝同步原语 |
| `ttg.warp_specialize` | Warp 特化控制流，将 warp 分组执行不同代码路径 |
| `ttg.memdesc_subslice` | 共享内存描述符切片 |

### 4.3 TritonNvidiaGPU 方言（`ttnvgpu.*`）

`third_party/nvidia/include/`

NVIDIA 专属操作，对应 Hopper/Blackwell 硬件特性：

- **TMA**：TMA 描述符操作、`ttnvgpu.tensor_memory_base`
- **WGMMA**：`ttnvgpu.wgmma_mma_raw_op`、`ttnvgpu.wgmma_fence`、`ttnvgpu.wgmma_commit_group`、`ttnvgpu.wgmma_wait_group`
- **Cluster**：`ttnvgpu.cluster_arrive`、`ttnvgpu.cluster_wait`、`ttnvgpu.cluster_id`
- **TMEM**（Tensor Memory，Blackwell）：`ttnvgpu.tmem_alloc`、`ttnvgpu.tmem_load`、`ttnvgpu.tmem_store`
- **mbarrier**：`ttnvgpu.init_barrier`、`ttnvgpu.arrive_barrier`、`ttnvgpu.wait_barrier`

### 4.4 Gluon 方言（`gluon.*`）

`python/triton/experimental/gluon/`

Gluon 是 Triton 的实验性扩展方言，用于布局推断辅助。核心操作：

- `gluon.set_auto_layout`：标记需要自动推断布局的 tensor，由 `add_infer_coalesced_encodings` 和 `add_resolve_auto_encodings` pass 填充具体 encoding

Gluon 语言（`Language.GLUON`）走独立的编译路径：`gluon_to_ttgir()` 而非 `make_ttgir()`。

### 4.5 TritonInstrument 方言

用于性能分析与 profiling 插桩，通过 `CUDABackend.instrumentation` / `HIPBackend.instrumentation` 钩子在 `ttgpuir_to_llvmir` 和 `llvmir_to_llvm` 阶段注入 pass。支持模式：`fpsan`（浮点 sanitizer）、`consan`（并发 sanitizer）、`iisan`。

---

## 五、TritonToTritonGPU 转换

`lib/Conversion/TritonToTritonGPU/`（3 个文件）

| 文件 | 职责 |
|------|------|
| `TritonToTritonGPUPass.cpp` | Pass 入口，调用类型转换器和 pattern rewriter |
| `TritonGPUConversion.cpp` | 类型转换器：为 tensor 类型附加初始 `#ttg.blocked` encoding |
| `RelayoutTritonGPU.cpp` | 处理需要重新布局的特殊情况 |

**转换职责**：将硬件无关的 TTIR 转换为布局感知的 TTGIR。

**初始布局分配策略**：

转换时通过 `passes.ttir.add_convert_to_ttgpuir(pm, target, num_warps, warp_size, num_ctas)` 触发。所有 tensor 默认分配 `#ttg.blocked` encoding，参数由 `num_warps`、`warp_size`、tensor shape 共同决定：

- `warpsPerCTA`：根据 tensor 维度和 warp 数量启发式分配
- `sizePerThread`：根据元素类型和访问模式确定每线程处理元素数
- `order`：按 tensor 维度的连续性排列（最内层维度优先）

后续优化 pass（`accelerate_matmul`、`coalesce` 等）会将 `#ttg.blocked` 替换为更优的 encoding（如 `#ttg.nvidia_mma`、`#ttg.dot_op`）。

---

## 六、TritonGPU 优化 Pass 体系

Pass 按功能分组，以下为 NVIDIA SM80-90 路径的完整执行序列（`make_ttgir()`）：

### 第一阶段：初始转换与基础优化

```
convert_to_ttgpuir      → 分配初始 blocked encoding
coalesce                → 优化内存访问连续性，调整 sizePerThread
f32_dot_tc              → 将 f32 dot 转换为 tensor core 操作（emuTF32）
plan_cta                → 规划 CTA cluster 布局（NVIDIA 专属）
remove_layout_conversions → 消除冗余的 convert_layout 操作
optimize_thread_locality  → 优化线程局部性，减少跨 warp 通信
```

### 第二阶段：矩阵乘加速

```
accelerate_matmul       → 将 tt.dot 替换为 MMA 指令布局
                          (NvidiaMmaEncoding / AMDMfmaEncoding)
remove_layout_conversions → 再次消除冗余布局转换
optimize_dot_operands   → 为 dot 操作数选择最优 shared memory 布局
                          (DotOperandEncoding + SwizzledSharedEncoding)
optimize_descriptor_encoding → 优化 TMA 描述符编码（NVIDIA 专属）
```

### 第三阶段：软件流水线（SM80/90 路径）

```
fuse_nested_loops           → 融合嵌套循环
canonicalizer / triton_licm → 循环不变量外提
combine_tensor_select_and_if → 合并条件选择
hopper_warpspec             → Hopper warp 特化预处理（SM90）
assign_latencies            → 为操作分配延迟注解（num_stages 参数）
schedule_loops              → 循环调度，生成软件流水线结构
pipeline                    → 实际插入流水线 prologue/epilogue/mainloop
```

### 第四阶段：Warp 特化（SM100+ Blackwell 路径）

```
optimize_accumulator_init   → 优化累加器初始化
hoist_tmem_alloc            → 提升 TMEM 分配到循环外
promote_lhs_to_tmem         → 将矩阵乘左操作数提升到 Tensor Memory
assign_latencies / schedule_loops / warp_specialize / pipeline
optimize_partition_warps    → 优化 warp 分区
hoist_tmem_alloc (allow_if) → 允许跨 if 语句提升
remove_tmem_tokens          → 清理 TMEM token
```

### 第五阶段：收尾优化

```
prefetch                    → 预取操作插入
optimize_dot_operands       → 再次优化 dot 操作数
coalesce_async_copy         → 合并异步拷贝操作
optimize_tmem_layouts       → 优化 TMEM 布局（NVIDIA 专属）
tma_lowering                → TMA 操作降低（SM90+）
remove_layout_conversions   → 最终消除冗余布局转换
interleave_tmem             → TMEM 交错访问优化
reduce_data_duplication     → 减少数据重复
reorder_instructions        → 指令重排以提高 ILP
fence_insertion             → 插入内存 fence（NVIDIA 专属）
lower_mma                   → MMA 操作最终降低
sccp / cse / canonicalizer  → 标准清理 pass
```

### AMD 专属 Pass 序列差异

AMD 路径在 `make_ttgir()` 中使用 `amd.passes.ttgpuir.*` 命名空间的专属 pass：

- `add_accelerate_matmul(arch, matrix_instr_nonkdim, kpack)`：MFMA/WMMA 加速
- `add_optimize_epilogue`：epilogue 优化
- `add_hoist_layout_conversions` / `add_sink_layout_conversions`：布局转换提升/下沉
- `add_schedule_loops(num_stages)` / `add_pipeline(use_async_copy, use_block_pingpong)`：AMD 软件流水线
- `add_block_pingpong(num_stages)`：乒乓缓冲（gfx942/gfx950）
- `add_convert_to_buffer_ops`：将全局内存访问转换为 buffer 操作（AMD 专属硬件特性）
- `add_in_thread_transpose`：线程内转置（gfx942）

---

## 七、后端抽象模型

`python/triton/backends/compiler.py`

### `GPUTarget` 数据类

```python
@dataclass(frozen=True)
class GPUTarget:
    backend: str        # "cuda" 或 "hip"
    arch: Union[int, str]  # CUDA: 90（compute capability）；HIP: "gfx940"
    warp_size: int      # CUDA: 32；HIP CDNA: 64；HIP RDNA: 32
```

### `BaseBackend` 抽象接口

| 方法 | 签名 | 职责 |
|------|------|------|
| `supports_target` | `(target: GPUTarget) -> bool` | 静态方法，判断是否支持目标 |
| `hash` | `() -> str` | 返回后端唯一标识（用于缓存键） |
| `parse_options` | `(opts: dict) -> object` | 将选项字典转换为后端特定的 Options 对象 |
| `add_stages` | `(stages: dict, options, language) -> None` | 填充编译阶段字典 |
| `load_dialects` | `(context) -> None` | 向 MLIR context 加载后端专属方言 |
| `get_module_map` | `() -> Dict[str, ModuleType]` | 返回设备库模块映射（如 libdevice） |
| `get_codegen_implementation` | `(options) -> dict` | 返回代码生成辅助函数（类型转换、min_dot_size） |

### `DriverBase` 抽象接口

`python/triton/runtime/driver.py`

提供设备管理、kernel 启动、内存分配等运行时操作的抽象接口，由 CUDA/HIP 驱动实现。

---

## 八、NVIDIA 后端

`third_party/nvidia/backend/compiler.py`

### 编译阶段

```
ttir → ttgir → llir → ptx → cubin
```

### `CUDAOptions` 关键参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `num_warps` | 4 | 每 CTA 的 warp 数（必须为 2 的幂） |
| `num_ctas` | 1 | CTA cluster 大小（SM90+ 支持 >1） |
| `num_stages` | 3 | 软件流水线级数 |
| `warp_size` | 32 | NVIDIA 固定 32 |
| `maxnreg` | None | 最大寄存器数（对应 PTX `.maxnreg`） |
| `ptx_version` | None | PTX 版本（自动从 CUDA 版本推导） |
| `default_dot_input_precision` | `"tf32"` | dot 操作默认精度 |
| `arch` | `"sm90"` 等 | 目标 SM 架构字符串 |

### make_ttir()（TTIR 清理）

```
inliner → rewrite_tensor_pointer
→ [rewrite_tensor_descriptor_to_pointer（<SM90）]
→ canonicalizer → combine → reorder_broadcast
→ cse → symbol_dce → loop_unroll
```

### make_ttgir()（核心优化，按 capability 分支）

- **SM70-79**：仅 `triton_licm`，无流水线
- **SM80-90**：完整 Hopper warp 特化 + 软件流水线（见第六章）
- **SM100+（Blackwell）**：TMEM + warp 特化 + 更激进的流水线

### make_llir()（TTGIR → LLVM IR）

关键步骤：

1. `allocate_warp_groups`：分配 warp 组
2. `scf_to_cf`：结构化控制流 → 基本块
3. `allocate_shared_memory_nv`：计算并分配共享内存（`ttg.shared` 属性）
4. `allocate_tensor_memory`：分配 TMEM（Blackwell）
5. `nvidia.passes.ttgpuir.add_to_llvmir`：主要 TTGIR → LLVM IR 转换
6. `warp_specialize_to_llvm`：warp 特化控制流降低
7. `nvgpu_to_llvm`：NVGPU 操作降低
8. `nvvm_to_llvm`：NVVM intrinsic 降低
9. `llvm.to_module()` + `llvm.optimize_module(O3)`：MLIR LLVM → LLVM C++ API

元数据提取：`shared`、`tmem_size`、`global_scratch_size`、`num_warps`（warp 特化后可能变化）

### make_ptx()

`llvm.translate_to_asm(src, "nvptx64-nvidia-cuda", proc, features, flags, ...)`

- `proc`：`sm_90a`、`sm_80` 等
- `features`：`+ptx86` 等（从 CUDA 版本推导，上限 PTX 9.0）
- 后处理：正则替换 `.version` 和 `.target` 字段

### make_cubin()

调用 `ptxas` 子进程：

```
ptxas -lineinfo --fmad=... -v --regAllocOptLevel=2 --gpu-name=sm_90a input.ptx -o output.cubin
```

---

## 九、AMD 后端

`third_party/amd/backend/compiler.py`

### 编译阶段

```
ttir → ttgir → llir → amdgcn → hsaco
```

### `HIPOptions` 关键参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `num_warps` | 4 | 每 CTA 的 wavefront 数 |
| `waves_per_eu` | 0 | 每 EU 的 wave 数（0 = 不限制） |
| `num_stages` | 2 | 软件流水线级数 |
| `arch` | `"gfx940"` 等 | 目标 GCN 架构字符串 |
| `warp_size` | 64（CDNA）/ 32（RDNA） | 由 arch 自动推导 |
| `matrix_instr_nonkdim` | 0 | MFMA 非 K 维度大小 |
| `kpack` | 1 | K 维度打包因子（gfx950 已废弃） |
| `schedule_hint` | `"none"` | 指令调度提示（`"attention"` 等） |

### make_ttir()

与 NVIDIA 路径基本相同，差异：

- AMD 支持 `add_triton_licm`（循环不变量外提）
- `rewrite_tensor_descriptor_to_pointer` 仅在不支持 TDM 的架构上执行

### make_ttgir()（AMD 专属优化序列）

```
convert_to_ttgpuir → coalesce → f32_dot_tc
→ remove_layout_conversions → optimize_thread_locality
→ amd.accelerate_matmul → remove_layout_conversions
→ amd.optimize_epilogue → amd.optimize_dot_operands
→ amd.hoist_layout_conversions → amd.sink_layout_conversions
→ fuse_nested_loops → canonicalizer → triton_licm
→ amd.schedule_loops → amd.pipeline(use_async_copy, use_block_pingpong)
→ [amd.coalesce_async_copy（gfx950/gfx1250）]
→ amd.convert_to_tensor_ops
→ [amd.insert_instruction_sched_hints]
→ remove_layout_conversions → reduce_data_duplication
→ [amd.in_thread_transpose（gfx942）]
→ amd.move_up_prologue_loads
→ [amd.block_pingpong（gfx942/gfx950）]
→ [amd.canonicalize_pointers → amd.convert_to_buffer_ops → amd.optimize_buffer_op_ptr]
→ amd.fold_true_cmpi → amd.prepare_if_combining
→ canonicalizer → cse → symbol_dce
```

### make_llir()（AMD TTGIR → LLVM IR）

关键步骤：

1. `amd.update_async_wait_count`：更新异步等待计数
2. `amd.warp_pipeline_conversion`：warp 流水线转换
3. `scf_to_cf` / `index_to_llvmir`：控制流降低
4. `amd.allocate_shared_memory`：共享内存分配（LDS）
5. `amd.to_llvmir(arch, __HIP_FTZ)`：主要 TTGIR → LLVM IR 转换
6. `amd.warp_specialize_to_llvm`：warp 特化降低
7. `cf_to_llvmir` / `arith_to_llvmir`：标准方言降低
8. `llvm.to_module()` + 设置 AMDGPU 函数属性：
   - `amdgpu-flat-work-group-size`、`amdgpu-waves-per-eu`
   - `amdgpu-cluster-dims`（multi-CTA）
   - `denormal-fp-math-f32`
9. 链接外部库（ocml、ockl）
10. `llvm.optimize_module(O3, arch)`

### make_amdgcn()

`llvm.translate_to_asm(src, TARGET_TRIPLE, arch, features, ...)` 生成 GCN 汇编文本。

### make_hsaco()

`amd.assemble_amdgcn()` + `amd.link_hsaco()` 生成最终 HSACO 二进制。

---

## 十、TritonGPUToLLVM 公共转换层

`lib/Conversion/TritonGPUToLLVM/`

公共层负责将 `ttg.*` 操作降低为 LLVM IR，被 NVIDIA 和 AMD 后端共享。

### 核心转换文件职责

| 文件 | 职责 |
|------|------|
| `ConvertLayoutOpToLLVM.cpp` | `ttg.convert_layout` → shared memory 中转或寄存器重排 |
| `ReduceOpToLLVM.cpp` | `tt.reduce` → warp shuffle + shared memory 归约 |
| `ScanOpToLLVM.cpp` | `tt.scan` → 前缀扫描（warp 级 + block 级） |
| `DotOpToLLVM.cpp` | `tt.dot` → 通用 dot 降低（非 MMA 路径） |
| `LoadStoreOpToLLVM.cpp` | `tt.load`/`tt.store` → LLVM 内存操作 |
| `AtomicOpToLLVM.cpp` | `tt.atomic_rmw`/`tt.atomic_cas` → LLVM 原子操作 |
| `ElementwiseOpToLLVM.cpp` | 逐元素操作 → LLVM 算术指令 |
| `HistogramOpToLLVM.cpp` | `tt.histogram` → shared memory 直方图 |
| `PrintOpToLLVM.cpp` | `tt.print` → device printf |
| `AssertOpToLLVM.cpp` | `tt.assert` → device assert |
| `MemoryOpToLLVM.cpp` | `ttg.local_alloc/load/store` → shared memory 操作 |
| `AsyncCopyOpToLLVM.cpp` | `ttg.async_copy_global_to_local` → cp.async / ds_read |
| `SPMDOpToLLVM.cpp` | `tt.get_program_id`/`tt.get_num_programs` → 线程 ID 计算 |
| `FuncOpToLLVM.cpp` | `tt.func` → LLVM function，处理 kernel 属性 |
| `ViewOpToLLVM.cpp` | `tt.reshape`/`tt.broadcast`/`tt.trans` → 寄存器重排 |
| `TensorPtrOpsToLLVM.cpp` | tensor pointer 操作降低 |
| `TypeConverter.cpp` | TTGIR 类型 → LLVM 类型转换 |
| `Utility.cpp` | 公共工具函数（线程 ID 计算、shared memory 地址计算） |

### 线性布局（Linear Layout）系统

`include/triton/Dialect/TritonGPU/IR/LinearLayout.h`

线性布局是 TritonGPU 编码系统的数学基础，用统一的线性代数框架描述任意 encoding 间的数据重排：

- 每个 encoding 可表示为从（lane, warp, block）坐标到 tensor 元素索引的线性映射（GF(2) 上的矩阵）
- `ConvertLayoutOpToLLVM` 通过计算两个 encoding 的线性布局之间的变换矩阵，确定数据是否需要经过 shared memory 中转，以及中转时的地址计算方式
- 支持的维度：`register`（线程内寄存器）、`lane`（warp 内线程）、`warp`（CTA 内 warp）、`block`（CTA cluster 内 CTA）

---

## 十一、Python 绑定层

`python/src/`

### pybind11 绑定架构

所有 C++ MLIR 操作通过 pybind11 暴露给 Python，编译为 `triton._C.libtriton` 共享库。

### `ir.cc`（~86KB）

`python/src/ir.cc`

最核心的绑定文件，封装：

- **MLIR 基础设施**：`ir.context()`、`ir.module`、`ir.function`、`ir.block`、`ir.region`、`ir.builder`
- **类型构造**：`get_int1_ty`、`get_fp16_ty`、`get_ptr_ty`、`get_block_ty` 等
- **TTIR 操作构造**：所有 `tt.*` 操作的 Python 接口（`create_load`、`create_store`、`create_dot`、`create_reduce` 等）
- **TTGIR 操作构造**：`create_convert_layout`、`create_local_alloc`、`create_async_copy` 等
- **Pass Manager**：`ir.pass_manager`，`pm.run(mod, stage_name)`
- **模块解析**：`ir.parse_mlir_module`、`ir.load_dialects`

### `gluon_ir.cc`（~59KB）

`python/src/gluon_ir.cc`

Gluon 方言的 Python 绑定，提供 Gluon 操作构造接口和布局推断 pass 接口。

### `passes.cc`

`python/src/passes.cc`

所有 MLIR pass 的 Python 接口，按命名空间组织：

- `passes.common.*`：`add_inliner`、`add_canonicalizer`、`add_cse`、`add_sccp`、`add_symbol_dce`
- `passes.ttir.*`：`add_convert_to_ttgpuir`、`add_combine`、`add_rewrite_tensor_pointer`、`add_loop_unroll`、`add_triton_licm`、`add_loop_aware_cse`
- `passes.ttgpuir.*`：`add_coalesce`、`add_accelerate_matmul`、`add_pipeline`、`add_assign_latencies`、`add_schedule_loops`、`add_remove_layout_conversions`、`add_reorder_instructions`、`add_prefetch`、`add_warp_specialize` 等
- `passes.convert.*`：`add_scf_to_cf`、`add_index_to_llvmir`、`add_arith_to_llvmir`、`add_nvvm_to_llvm`
- `passes.llvmir.*`：`add_di_scope`、`add_di_local_variable`
- `passes.gluon.*`：`add_inliner`、`add_infer_coalesced_encodings`、`add_resolve_auto_encodings`

### `llvm.cc`（~34KB）

`python/src/llvm.cc`

LLVM IR 生成与优化绑定：

- `llvm.context()`：创建 LLVM context
- `llvm.to_module(mlir_mod, context)`：MLIR LLVM 方言 → LLVM C++ Module
- `llvm.attach_datalayout(mod, triple, proc, features)`：设置目标数据布局
- `llvm.optimize_module(mod, level, arch, ...)`：运行 LLVM 优化 pipeline（O3）
- `llvm.translate_to_asm(src, triple, proc, features, flags, ...)`：LLVM IR → 汇编文本
- `llvm.link_extern_libs(mod, paths)`：链接外部 bitcode 库（libdevice、ocml、ockl）
- `llvm.init_targets()`：初始化 LLVM 目标（NVPTX、AMDGPU）

---

## 关键文件路径速查

| 组件 | 路径 |
|------|------|
| Python 入口 | `python/triton/__init__.py` |
| JIT 运行时 | `python/triton/runtime/jit.py` |
| 编译器主体 | `python/triton/compiler/compiler.py` |
| 代码生成 | `python/triton/compiler/code_generator.py` |
| 语言核心 | `python/triton/language/core.py` |
| 语义分析 | `python/triton/language/semantic.py` |
| 后端抽象 | `python/triton/backends/compiler.py` |
| Triton 方言 ops | `include/triton/Dialect/Triton/IR/TritonOps.td` |
| Triton 方言类型 | `include/triton/Dialect/Triton/IR/TritonTypes.td` |
| TritonGPU 编码 | `include/triton/Dialect/TritonGPU/IR/TritonGPUAttrDefs.td` |
| TritonToTritonGPU | `lib/Conversion/TritonToTritonGPU/` |
| TritonGPUToLLVM | `lib/Conversion/TritonGPUToLLVM/` |
| NVIDIA 后端 | `third_party/nvidia/backend/compiler.py` |
| AMD 后端 | `third_party/amd/backend/compiler.py` |
| Python 绑定（IR） | `python/src/ir.cc` |
| Python 绑定（Pass） | `python/src/passes.cc` |
| Python 绑定（LLVM） | `python/src/llvm.cc` |
| Python 绑定（Gluon） | `python/src/gluon_ir.cc` |
