# TritonGPU 方言架构文档

## 一、概述

TritonGPU 方言（`ttg.*`）是 Triton 编译器的核心中间层，位于硬件无关的 Triton IR（`tt.*`）与后端 LLVM IR 之间。其核心职责是将 tile 级操作与 GPU 线程层次绑定，通过**布局编码系统**（Layout Encoding）精确描述数据在 GPU 线程间的分布方式，并提供一套完整的优化 Pass 体系。

```
Triton IR (tt.*)
      │  TritonToTritonGPU pass
      ▼
TritonGPU IR (ttg.*)   ← 本文档的分析对象
      │  TritonGPUToLLVM pass
      ▼
LLVM IR
```

**方言标识**：`ttg`，C++ 命名空间 `::mlir::triton::gpu`

**源码位置**：
- 定义：`include/triton/Dialect/TritonGPU/IR/`
- 实现：`lib/Dialect/TritonGPU/IR/`
- Pass：`lib/Dialect/TritonGPU/Transforms/`

---

## 二、类型系统

`include/triton/Dialect/TritonGPU/IR/TritonGPUTypes.td`

TritonGPU 方言引入两个核心类型：

### 2.1 `ttg.async.token`（AsyncTokenType）

```
ttg.async.token
```

异步操作的 token 类型，用于建立异步操作之间的 SSA 依赖链。由 `ttg.async_copy_global_to_local` 产生，被 `ttg.async_commit_group` 和 `ttg.async_wait` 消费。

### 2.2 `!ttg.memdesc<shape x etype, encoding, memspace, mutable, allocShape>`（MemDescType）

```
!ttg.memdesc<16x32xf16, #ttg.swizzled_shared<{...}>, #ttg.shared_memory, mutable>
```

内存描述符类型，封装共享内存（或全局 scratch 内存）的基地址指针和布局描述。与 `RankedTensorType` 的关键区别：

| 特性 | `tensor<...>` | `!ttg.memdesc<...>` |
|------|--------------|---------------------|
| 存储位置 | 寄存器（分布式） | 共享内存 / 全局内存 |
| 多视图 | 不支持 | 支持（subslice/trans/reshape） |
| 可变性 | 不可变 SSA | 可选 mutable 标记 |
| 生命周期 | SSA 值 | 显式 alloc/dealloc |

参数说明：
- `shape`：逻辑 tensor 形状
- `encoding`：共享内存布局编码（`SharedEncodingTrait` 实现）
- `memorySpace`：`#ttg.shared_memory` 或 `#ttg.global_memory`
- `mutableMemory`：是否可写（false = 只写一次的常量缓冲）
- `allocShape`：实际分配的物理形状（含 padding 时可能大于 shape）

---

## 三、布局编码系统（Layout Encoding）

布局编码是 TritonGPU 方言的核心创新，通过 tensor 的 `encoding` attribute 精确描述数据在 GPU 线程层次（register → lane → warp → block）中的分布。

`include/triton/Dialect/TritonGPU/IR/TritonGPUAttrDefs.td`

### 3.1 接口层

`include/triton/Dialect/TritonGPU/IR/TritonGPUAttrInterfaces.td`

| 接口 | 实现者 | 核心方法 |
|------|--------|---------|
| `LayoutEncodingTrait` | 所有 encoding | `getCGALayout()`, `getRank()` |
| `DistributedEncodingTrait` | 寄存器 encoding | `toLinearLayout(shape)`, `getTotalElemsPerThread(shape)`, `getElemsPerThread(shape)` |
| `SharedEncodingTrait` | 共享内存 encoding | `getAlignment()` (默认 16) |
| `MmaEncodingTrait` | MMA encoding | `getRepOrderForOperand(opIdx)` |

### 3.2 CGAEncodingAttr（CGA 层级编码）

```
#ttg.cga_encoding<{linearLayout = ...}>
```

描述 CTA（block）在 CGA（Cooperative Grid Array）中如何映射到逻辑 tensor 维度。是所有 encoding 的公共组件，通过 `getCGALayout()` 访问。

工厂方法：
- `get1CTALayout(context, rank)`：单 CTA，空 bases
- `get1DLayout(context, numCTAs)`：1D 多 CTA 线性分布
- `fromSplitParams(CTAsPerCGA, CTASplitNum, CTAOrder)`：遗留接口，兼容旧代码

### 3.3 分布式 Encoding（寄存器布局）

#### BlockedEncodingAttr

```
#ttg.blocked<{sizePerThread=[2,2], threadsPerWarp=[8,4], warpsPerCTA=[1,2], order=[1,0]}>
```

最基础的分布式 encoding，每个 warp 拥有 tensor 的连续区域，促进 load/store coalescing。

参数语义：
- `sizePerThread`：每线程持有的元素数（各维度）
- `threadsPerWarp`：每 warp 的线程分布（各维度，乘积 = warp_size）
- `warpsPerCTA`：每 CTA 的 warp 分布（各维度，乘积 = num_warps）
- `order`：最快变化轴在前（`[1,0]` 表示列优先）

布局函数 `L(i)`：对于 tensor 元素索引 `i`，返回持有该元素的所有（lane, warp）对的集合。当 tensor 比 layout 大时 wrap-around（多值/线程），当 tensor 比 layout 小时 broadcast（多线程/值）。

#### LinearEncodingAttr

```
#ttg.linear<{register=[[0,1],[8,0],[0,8],[64,0]], lane=[[0,2],[0,4],[1,0],[2,0],[4,0]], warp=[[16,0],[32,0]], block=[]}>
```

通用线性 encoding，用 GF(2) 上的矩阵（LinearLayout）精确描述任意数据分布。是 `BlockedEncoding` 的泛化形式，也是所有 encoding 的统一数学表示。

参数：`linearLayout`（`LinearLayout` 对象，包含 register/lane/warp/block 四个维度的基向量）

#### SliceEncodingAttr

```
#ttg.slice<{dim=0, parent=#ttg.blocked<{...}>}>
```

从高维 encoding 沿指定维度切片降维，用于 `tt.reduce` 的输出 tensor。`dim` 指定被归约的维度，`parent` 是原始高维 encoding。

#### DotOperandEncodingAttr

```
#ttg.dot_op<{opIdx=0, parent=#ttg.nvidia_mma<{...}>, kWidth=4}>
```

dot 操作数专用 encoding，携带操作数在 MMA 指令中的角色信息：
- `opIdx`：0 = 矩阵 A（左操作数），1 = 矩阵 B（右操作数）
- `parent`：对应的 MMA encoding（决定具体的寄存器布局）
- `kWidth`：K 维度的向量化宽度

#### NvidiaMmaEncodingAttr

```
#ttg.nvidia_mma<{versionMajor=3, versionMinor=0, warpsPerCTA=[4,1], instrShape=[16,256,16]}>
```

NVIDIA MMA 指令布局，对应 Ampere（MMAv2）和 Hopper（MMAv3/WGMMA）的矩阵乘累加指令输出格式。

- `versionMajor=2`：Ampere `mma.sync.aligned.m16n8k16`
- `versionMajor=3`：Hopper `wgmma.mma_async`（warpgroup 级别）
- `instrShape`：`[M, N, K]` 单条 MMA 指令处理的 tile 大小

#### AMDMfmaEncodingAttr

```
#ttg.amd_mfma<{version=3, warpsPerCTA=[2,2], instrShape=[32,32,8], isTransposed=false}>
```

AMD CDNA 架构的 MFMA（Matrix Fused Multiply-Add）指令布局。

- `version`：1=gfx908, 2=gfx90a, 3=gfx942, 4=gfx950
- `instrShape`：`[M, N, K]`，如 `[32,32,8]`（f32）或 `[16,16,16]`（f16）
- `isTransposed`：用于 Flash-Attention 等 chained dot 场景
- `tilesPerWarp`：每 warp 连续 tile 数（默认全 1）

#### AMDWmmaEncodingAttr

```
#ttg.amd_wmma<{version=1, warpsPerCTA=[2,2]}>
```

AMD RDNA 架构的 WMMA（Wave Matrix Multiply-Accumulate）指令布局。

- `version`：1=gfx11xx（RDNA3），2=gfx12xx（RDNA4）

### 3.4 共享内存 Encoding

#### SwizzledSharedEncodingAttr（主流）

```
#ttg.swizzled_shared<{vec=4, perPhase=2, maxPhase=4, order=[1,0]}>
```

通过 XOR swizzle 消除 shared memory bank conflict。

参数语义：
- `vec`：向量化宽度（swizzle 粒度，单位：元素数）
- `perPhase`：每个 phase 包含的行数
- `maxPhase`：最大 phase 数（xor 值上限为 maxPhase-1）
- `order`：最快变化轴在前

Swizzle 规则：行 `r` 的列 `c` 处的元素，其物理地址列索引为 `(c/vec ^ (r/perPhase) % maxPhase) * vec + c%vec`。

#### PaddedSharedEncodingAttr

```
#ttg.padded_shared<[2:+2] {offset=[[0,1],[0,2],[1,0],[2,0]], block=[]}>
```

通过 padding + 线性重映射消除 bank conflict。适用于 swizzle 无法处理的特殊访问模式。

参数：
- `intervals`/`paddings`：每 `interval` 个元素后插入 `padding` 个填充（均为 2 的幂）
- `linearComponent`：`LinearLayout`，描述 shared memory offset → 逻辑 tensor 索引的映射

#### PartitionedSharedEncodingAttr

```
#ttg.partitioned_shared<{numPartitions=2, numGroups=4, partitionDim=0, partitionLayout=#ttg.swizzled_shared<{...}>}>
```

将 tensor 沿 `partitionDim` 分割为 `numPartitions * numGroups` 个逻辑 piece，分配到 `numPartitions` 个物理 buffer（保证不同 partition 的 buffer 在不同物理 shared memory slot），减少 partition conflict。

#### SharedLinearEncodingAttr

```
#ttg.shared_linear<{linearLayout=..., layoutAlignment=16}>
```

线性共享内存 encoding，用 `LinearLayout` 直接描述 shared memory offset → 逻辑 tensor 索引的映射。

#### NVMMASharedEncodingAttr

```
#ttg.nvmma_shared<{swizzlingByteWidth=128, transposed=false, elementBitWidth=16, fp4Padded=false}>
```

Hopper/Blackwell MMAv3/MMAv5 专用共享内存布局，对应 PTX 文档中的 warpgroup-level matrix shared memory layout。

- `swizzlingByteWidth`：32/64/128 字节 swizzle 宽度（由 shape/eltTy 自动推导）
- `transposed`：最外维连续（用于 B 矩阵）
- `fp4Padded`：MMAv5 fp4 特殊 padding 格式（`mxf8f6f4` 指令）

#### AMDRotatingSharedEncodingAttr

```
#ttg.amd_rotating_shared<{vec=4, perPhase=1, maxPhase=8, order=[1,0]}>
```

AMD 专属旋转 swizzle，每个 block 的起始 phase 不同（旋转），用于无原生 shared memory transpose 支持的 RDNA 架构。

Swizzle 规则：`outColId = inColId ^ (phase ^ blockNo)`，其中 `blockNo = (inRowId / (perPhase * maxPhase)) % maxPhase`。

---

## 四、操作集（Ops）

`include/triton/Dialect/TritonGPU/IR/TritonGPUOps.td`

所有 `ttg.*` Op 继承自 `TTG_Op`，自动附加 `VerifyTensorLayoutsTrait` 和 `VerifyMemDescLayoutsTrait` 两个验证 trait。

### 4.1 布局转换

#### `ttg.convert_layout`

```mlir
%out = ttg.convert_layout %src : tensor<32x32xf16, #blocked> -> tensor<32x32xf16, #mma>
```

在不同 encoding 之间转换 tensor 布局。这是 TritonGPU 中最关键的操作，可能产生 shared memory 中转（当两个 encoding 之间无法直接寄存器重排时）。

Traits：`SameOperandsAndResultShape`、`SameOperandsAndResultElementType`、`Pure`、`hasCanonicalizer`

### 4.2 共享内存管理

#### `ttg.local_alloc`

```mlir
%buf = ttg.local_alloc : () -> !ttg.memdesc<32x32xf16, #swizzled_shared, #shared_memory, mutable>
%buf = ttg.local_alloc %src : (tensor<32x32xf16, #blocked>) -> !ttg.memdesc<...>
```

在共享内存中分配 buffer，返回 `MemDescType`。可选 `src` 操作数用于初始化（等价于 alloc + store）。可选 `alignment` 属性指定对齐要求。

#### `ttg.local_dealloc`

```mlir
ttg.local_dealloc %buf : !ttg.memdesc<...>
```

显式释放共享内存 buffer（可选，编译器会在最后一次使用后自动推断释放点）。

#### `ttg.local_load`

```mlir
%out = ttg.local_load %buf : !ttg.memdesc<32x32xf16, #shared, #shmem> -> tensor<32x32xf16, #blocked>
%out = ttg.local_load %buf token %tok : !ttg.memdesc<...> -> tensor<...>
```

从共享内存 buffer 加载到分布式 tensor（寄存器）。可选 `token` 操作数用于等待异步操作完成。

#### `ttg.local_store`

```mlir
ttg.local_store %src, %buf : tensor<32x32xf16, #blocked> -> !ttg.memdesc<...>
```

将分布式 tensor（寄存器）写入共享内存 buffer。

#### `ttg.local_gather`

```mlir
%out = ttg.local_gather %buf[%indices] : !ttg.memdesc<...>, tensor<32xi32, #blocked> -> tensor<32xf16, #blocked>
```

沿指定轴从共享内存 gather 元素，等价于 `tt.gather` 但操作对象是 `MemDescType`。

#### `ttg.local_scatter`

```mlir
ttg.local_scatter %buf[%indices], %values : !ttg.memdesc<...>, tensor<32xi32, #blocked>, tensor<32xf16, #blocked>
```

`local_gather` 的逆操作，将值 scatter 写入共享内存的运行时计算地址。

### 4.3 MemDesc 视图操作

#### `ttg.memdesc_index`

```mlir
%slice = ttg.memdesc_index %buf[%i] : !ttg.memdesc<2x32x32xf16, ...> -> !ttg.memdesc<32x32xf16, ...>
```

沿第 0 维取第 `i` 个切片，不影响底层内存。用于流水线中的 stage 索引。

#### `ttg.memdesc_subslice`

```mlir
%sub = ttg.memdesc_subslice %buf[0, 0] : !ttg.memdesc<32x32xf16, ...> -> !ttg.memdesc<16x32xf16, ...>
```

取任意子视图（多维偏移），不影响底层内存。

#### `ttg.memdesc_trans`

```mlir
%t = ttg.memdesc_trans %buf {order=[1,0]} : !ttg.memdesc<32x16xf16, ...> -> !ttg.memdesc<16x32xf16, ...>
```

转置 MemDesc 视图，不影响底层内存。

#### `ttg.memdesc_reshape`

```mlir
%r = ttg.memdesc_reshape %buf : !ttg.memdesc<32x32xf16, ...> -> !ttg.memdesc<1024xf16, ...>
```

重塑 MemDesc 视图，不影响底层内存（要求原始 MemDesc 连续）。

#### `ttg.memdesc_reinterpret`

```mlir
%r = ttg.memdesc_reinterpret %buf : !ttg.memdesc<32x32xf16, ...> -> !ttg.memdesc<32x16xi32, ...>
```

将 MemDesc 重新解释为不同的 shape 和元素类型（要求原始 MemDesc 连续）。

### 4.4 异步拷贝

#### `ttg.async_copy_global_to_local`

```mlir
%tok = ttg.async_copy_global_to_local %src, %dst mask %mask other %other
       cacheModifier = .ca evictionPolicy = .evict_normal
       : tensor<32x32x!tt.ptr<f16>, #blocked> -> !ttg.memdesc<32x32xf16, #shared, #shmem, mutable>
```

异步将全局内存拷贝到共享内存（对应 NVIDIA `cp.async` 指令，SM80+）。返回 `async.token`。

参数：
- `src`：全局内存指针 tensor
- `result`：目标共享内存 MemDesc（必须 mutable）
- `mask`/`other`：可选掩码和默认值
- `contiguity`：可向量化的最大元素数（用于选择 cp.async 宽度：4/8/16 字节）

#### `ttg.async_commit_group`

```mlir
%tok = ttg.async_commit_group tokens %tok1, %tok2
```

关闭当前批次的异步拷贝操作，允许后续 `async_wait` 等待。

#### `ttg.async_wait`

```mlir
%tok = ttg.async_wait %tok1, %tok2 {num = 0}
```

等待直到最多 `num` 个异步拷贝组仍在进行中（`num=0` 等待全部完成）。不提供 CTA 级同步，需配合 `ttg.barrier` 使用。

### 4.5 同步与控制流

#### `ttg.barrier`

```mlir
ttg.barrier local
ttg.barrier local|global_read|global_write
ttg.barrier all
```

CTA 级同步屏障，同步执行并确保指定地址空间的内存操作对所有线程可见。

`addrSpace` 位掩码：
- `none`：仅控制流同步（无内存序）
- `local`：共享内存操作完成
- `global_read`/`global_write`：全局内存读/写完成
- `tensor_read`/`tensor_write`：Tensor Memory 读/写完成（Blackwell）
- `all`：所有地址空间

#### `ttg.warp_specialize`

```mlir
%out = ttg.warp_specialize(%a, %b)
default {
  %r = some_op(%a)
  ttg.warp_yield %r : i32
}
partition0(%arg0: i32, %arg1: i32) num_warps(8) {
  async_dispatch(%arg0, %arg1)
  ttg.warp_return
}
partition1(%arg0: i32, %arg1: i32) num_warps(1) {
  async_dispatch(%arg0, %arg1)
  ttg.warp_return
} : (i32, i32) -> i32
```

Warp 特化控制流，将 CTA 内的 warp 分组，各组同时执行不同代码路径。

- `default` region：当前 warp 组执行的代码（可隐式捕获外部值）
- `partition` regions：独立 warp 组执行的代码（`IsolatedFromAbove`，通过显式参数传值）
- `partitionNumWarps`：各 partition 的 warp 数
- `warpGroupStartIds`：各 partition 的起始 warp ID（可选）
- `requestedRegisters`/`actualRegisters`：寄存器数量提示

#### `ttg.warp_id`

```mlir
%wid = ttg.warp_id : i32
```

返回当前线程的 warp ID（硬件寄存器或 thread_id / warp_size）。

### 4.6 流水线辅助

#### `ttg.predicate_stage`

```mlir
%pred = ttg.predicate_stage %iv, %ub, %step maxStage 3 stage 1 : index -> i1
```

软件流水线的 stage 谓词，判断当前循环迭代是否应执行指定 stage 的操作。

#### `ttg.mask` / `ttg.mask.return`

```mlir
%vals = ttg.mask(%pred) {
  %v = some_op(...)
  ttg.mask.return %v : f32
} : i1 -> f32
```

流水线 mask 操作，在谓词为 false 时跳过内部操作。

### 4.7 其他操作

#### `ttg.fp4_to_fp`

```mlir
%out = ttg.fp4_to_fp %src {axis=1} : tensor<32x16xi8, #blocked> -> tensor<32x32xf16, #blocked>
```

将 packed fp4（e2m1）上转型为浮点数。低 4 位为第一个 fp4 元素，高 4 位为第二个。

#### `ttg.global_scratch_alloc`

```mlir
%ptr = ttg.global_scratch_alloc {nbytes=1024, alignment=16} : !tt.ptr<i8>
```

分配 kernel 私有的全局内存 scratch buffer（每个 program 独立）。

#### `ttg.warp_yield` / `ttg.warp_return`

`warp_specialize` 的 terminator：`warp_yield` 从 default region 返回值，`warp_return` 从 partition region 返回（无返回值）。

---

## 五、线性布局系统（Linear Layout）

`include/triton/Tools/LinearLayout.h`

线性布局是 TritonGPU 编码系统的数学基础，用 GF(2) 上的线性代数统一描述所有 encoding 间的数据映射关系。

### 5.1 核心概念

LinearLayout 定义了从**输入维度**（GPU 线程层次）到**输出维度**（tensor 逻辑索引）的线性映射：

```
输入维度（in-dims）：register, lane, warp, block
输出维度（out-dims）：dim0, dim1, ..., dimN-1
```

每个输入维度有一组**基向量**（bases），每个基向量是一个输出维度的位向量。映射规则：对于输入坐标 `(r, l, w, b)`，输出坐标通过对所有激活位的基向量做 XOR 得到。

### 5.2 示例

```
#ttg.linear<{
  register = [[0,1],[8,0],[0,8],[64,0]],  // 4 个 register 位 → (dim0, dim1)
  lane     = [[0,2],[0,4],[1,0],[2,0],[4,0]],  // 5 个 lane 位（32 线程）
  warp     = [[16,0],[32,0]],             // 2 个 warp 位（4 warps）
  block    = []                           // 单 CTA
}>
```

解读：
- `register[0] = [0,1]`：register bit 0 控制 dim1 的 bit 0（即每线程持有 dim1 方向相邻的 2 个元素）
- `lane[2] = [1,0]`：lane bit 2 控制 dim0 的 bit 0（即 lane 4-7 与 lane 0-3 在 dim0 方向相邻）

### 5.3 在 convert_layout 中的应用

`lib/Conversion/TritonGPUToLLVM/ConvertLayoutOpToLLVM.cpp`

当 `ttg.convert_layout` 需要在两个 encoding 之间转换时：

1. 将源 encoding 和目标 encoding 都转换为 `LinearLayout`
2. 计算两个 LinearLayout 之间的变换矩阵
3. 判断是否需要 shared memory 中转：
   - 若变换可以通过 warp shuffle 完成 → 直接寄存器重排
   - 若需要跨 warp 通信 → 通过 shared memory 中转（`local_alloc` + `local_store` + `barrier` + `local_load`）

### 5.4 LinearLayout 与 Encoding 的转换

`include/triton/Dialect/TritonGPU/IR/LinearLayoutConversions.h`

- `toLinearLayout(shape, encoding)` → `LinearLayout`：将任意 encoding 转换为 LinearLayout 表示
- `LinearEncodingAttr::get(linearLayout)`：从 LinearLayout 创建 encoding
- 方言级缓存：`TritonGPU_Dialect` 内置 `LinearLayoutCache` 和 `LinearEncodingCache`，避免重复计算

---

## 六、优化 Pass 体系

`lib/Dialect/TritonGPU/Transforms/`

### 6.1 Pass 总览

| Pass 文件 | Pass 名称 | 职责 |
|-----------|-----------|------|
| `Coalesce.cpp` | `tritongpu-coalesce` | 优化内存访问连续性，调整 `sizePerThread` 使 load/store 向量化 |
| `AccelerateMatmul.cpp` | `tritongpu-accelerate-matmul` | 将 `tt.dot` 替换为 MMA 指令布局（NvidiaMma/AMDMfma） |
| `OptimizeDotOperands.cpp` | `tritongpu-optimize-dot-operands` | 为 dot 操作数选择最优 shared memory 布局（SwizzledShared/NVMMAShared） |
| `RemoveLayoutConversions.cpp` | `tritongpu-remove-layout-conversions` | 消除冗余的 `ttg.convert_layout` 操作 |
| `OptimizeThreadLocality.cpp` | `tritongpu-optimize-thread-locality` | 优化线程局部性，减少跨 warp 数据通信 |
| `F32DotTC.cpp` | `tritongpu-f32-dot-tc` | 将 f32 dot 转换为 tensor core 操作（emuTF32） |
| `ReorderInstructions.cpp` | `tritongpu-reorder-instructions` | 指令重排以提高 ILP，将 load 提前到使用点之前 |
| `ReduceDataDuplication.cpp` | `tritongpu-reduce-data-duplication` | 减少 convert_layout 产生的数据重复 |
| `Prefetch.cpp` | `tritongpu-prefetch` | 将 shared memory load 提前到循环前一次迭代 |
| `FuseNestedLoops.cpp` | `tritongpu-fuse-nested-loops` | 融合嵌套循环，为流水线创造条件 |
| `CoalesceAsyncCopy.cpp` | `tritongpu-coalesce-async-copy` | 合并相邻的 `async_copy_global_to_local` 操作 |
| `CombineTensorSelectAndIf.cpp` | `tritongpu-combine-tensor-select-and-if` | 将 `arith.select` 与 `scf.if` 合并，减少分支 |
| `OptimizeAccumulatorInit.cpp` | `tritongpu-optimize-accumulator-init` | 优化 MMA 累加器初始化（Blackwell TMEM） |
| `HoistTMEMAlloc.cpp` | `tritongpu-hoist-tmem-alloc` | 将 TMEM 分配提升到循环外（Blackwell） |
| `DecomposeScaledBlocked.cpp` | `tritongpu-decompose-scaled-blocked` | 分解 scaled dot 操作 |

### 6.2 软件流水线子系统

`lib/Dialect/TritonGPU/Transforms/Pipeliner/`

| 文件 | 职责 |
|------|------|
| `AssignLatencies.cpp` | `tritongpu-assign-latencies`：为操作标注延迟（load=3, mma=1 等），决定流水线深度 |
| `ScheduleLoops.cpp` | `tritongpu-schedule-loops`：基于延迟注解生成循环调度方案（stage 分配） |
| `SoftwarePipeliner.cpp` | `tritongpu-pipeline`：实际变换循环，插入 prologue/epilogue/mainloop |
| `PipelineExpander.cpp` | 将流水线调度展开为具体的 IR 变换 |
| `PipeliningUtility.cpp` | 流水线工具函数（stage 谓词、mask 生成） |
| `Schedule.cpp` | 调度数据结构和算法 |
| `WGMMAPipeline.cpp` | Hopper WGMMA 专用流水线（MMAv3 异步 warpgroup 矩阵乘） |
| `MMAv5PipelineUtility.cpp` | Blackwell MMAv5 流水线工具（TMEM 相关） |
| `TMAStoresPipeline.cpp` | TMA store 流水线（Hopper/Blackwell） |
| `LowerLoops.cpp` | 流水线循环的最终降低 |

**软件流水线工作原理**：

```
原始循环（num_stages=3）：
  for i in range(N):
    A = load(ptr_a + i)   # latency=3
    B = load(ptr_b + i)   # latency=3
    C += dot(A, B)

流水线变换后：
  # Prologue（预热）
  A0 = load(ptr_a + 0)
  B0 = load(ptr_b + 0)
  A1 = load(ptr_a + 1)
  B1 = load(ptr_b + 1)
  # Mainloop
  for i in range(2, N):
    Ai = load(ptr_a + i)   # stage 0
    Bi = load(ptr_b + i)   # stage 0
    C += dot(A[i-2], B[i-2])  # stage 2
  # Epilogue（排空）
  C += dot(A[N-2], B[N-2])
  C += dot(A[N-1], B[N-1])
```

### 6.3 Warp 特化子系统

`lib/Dialect/TritonGPU/Transforms/WarpSpecialization/`

| 文件 | 职责 |
|------|------|
| `AutomaticWarpSpecialization.cpp` | `tritongpu-automatic-warp-specialization`（Hopper）：自动识别可特化的 warp 组 |
| `Partition.cpp` | `tritongpu-warp-specialize`（Blackwell）：将循环分区为 producer/consumer warp 组 |
| `PartitionLoops.cpp` | `tritongpu-partition-loops`：将循环体分割为多个 warp partition |
| `PartitionScheduling.cpp` | Partition 调度策略 |
| `PartitionBuilder.cpp` | 构建 `ttg.warp_specialize` IR 结构 |
| `OptimizePartitionWarps.cpp` | `tritongpu-optimize-partition-warps`：优化各 partition 的 warp 数量分配 |

**Warp 特化模式**（Blackwell MMAv5）：

```
ttg.warp_specialize(...)
default {                    // Consumer warps（执行 MMA）
  %c = wgmma(...)
  ttg.warp_yield %c
}
partition0(...) num_warps(4) {  // Producer warps（执行 TMA load）
  tma_load(...)
  ttg.warp_return
}
```

---

## 七、方言级工具与接口

### 7.1 方言扩展方法

`include/triton/Dialect/TritonGPU/IR/Dialect.h`

```cpp
class TritonGPU_Dialect {
  // 将任意 encoding 转换为 LinearLayout（带缓存）
  LinearLayout toLinearLayout(ArrayRef<int64_t> shape, Attribute layout);

  // 将任意 encoding 转换为 LinearEncodingAttr（带缓存）
  LinearEncodingAttr toLinearEncoding(ArrayRef<int64_t> shape, Attribute layout);

  // 从 ModuleOp 读取 CTA 数量（ttg.num-ctas 属性）
  static int getNumCTAs(ModuleOp mod);

  // 从 ModuleOp 读取每 warp 线程数（ttg.threads-per-warp 属性）
  static int getThreadsPerWarp(ModuleOp mod);
};
```

### 7.2 模块级属性

TritonGPU 在 `ModuleOp` 上附加以下属性：

| 属性 | 类型 | 说明 |
|------|------|------|
| `ttg.num-warps` | i32 | 每 CTA 的 warp 数 |
| `ttg.num-ctas` | i32 | CTA cluster 大小 |
| `ttg.threads-per-warp` | i32 | 每 warp 的线程数（32 或 64） |
| `ttg.shared` | i32 | 共享内存使用量（字节，由 allocate_shared_memory pass 填充） |
| `ttg.total-num-warps` | i32 | warp 特化后的总 warp 数 |
| `ttg.tensor_memory_size` | i32 | TMEM 使用量（Blackwell） |
| `ttg.global_scratch_memory_size` | i32 | 全局 scratch 内存大小 |
| `ttg.maxnreg` | i32 | 最大寄存器数（对应 PTX `.maxnreg`） |

### 7.3 Op Interfaces

`include/triton/Dialect/TritonGPU/IR/TritonGPUOpInterfaces.td`

- `LocalLoadTrait`：标记从共享内存加载的操作（`local_load`、`local_gather`）
- `MemDescViewTrait`：标记 MemDesc 视图操作（不影响底层内存）
- `MemWaitOpTrait`：标记内存等待操作（`async_wait`）
- `AsyncRegions`：标记含异步 region 的操作（`warp_specialize`）
- `InferMemDescTypeOpWithLayoutEquivalence`：推断 MemDesc 返回类型

---

## 八、与其他方言的关系

```
┌─────────────────────────────────────────────────────────────┐
│                    Triton IR (tt.*)                          │
│  tt.dot / tt.load / tt.store / tt.reduce / tt.scan          │
└──────────────────────────┬──────────────────────────────────┘
                           │ TritonToTritonGPU
                           │ (分配初始 #ttg.blocked encoding)
┌──────────────────────────▼──────────────────────────────────┐
│                  TritonGPU IR (ttg.*)                        │
│  布局编码 + 共享内存操作 + 异步拷贝 + warp 特化              │
│                                                              │
│  依赖方言：                                                   │
│  - triton::TritonDialect（tt.* 操作）                        │
│  - mlir::gpu::GPUDialect（GPU 线程 ID）                      │
│  - mlir::scf（结构化控制流，流水线前）                        │
└──────┬───────────────────────────────────────┬──────────────┘
       │ NVIDIA 后端                            │ AMD 后端
┌──────▼──────────────┐             ┌──────────▼──────────────┐
│ ttnvgpu.* 方言       │             │ AMD 专属 encoding        │
│ TMA/WGMMA/TMEM      │             │ AMDMfma/AMDWmma          │
│ mbarrier/Cluster    │             │ AMDRotatingShared        │
└──────┬──────────────┘             └──────────┬──────────────┘
       │                                        │
┌──────▼────────────────────────────────────────▼──────────────┐
│              TritonGPUToLLVM 公共转换层                        │
│  ConvertLayoutOp / ReduceOp / ScanOp / LoadStoreOp / ...     │
└──────────────────────────────────────────────────────────────┘
```

---

## 九、关键文件速查

| 文件 | 说明 |
|------|------|
| `include/triton/Dialect/TritonGPU/IR/TritonGPUDialect.td` | 方言定义 |
| `include/triton/Dialect/TritonGPU/IR/TritonGPUAttrDefs.td` | 所有 encoding 定义（~1500 行） |
| `include/triton/Dialect/TritonGPU/IR/TritonGPUAttrInterfaces.td` | encoding 接口定义 |
| `include/triton/Dialect/TritonGPU/IR/TritonGPUOps.td` | 所有 Op 定义 |
| `include/triton/Dialect/TritonGPU/IR/TritonGPUTypes.td` | AsyncToken / MemDescType |
| `include/triton/Dialect/TritonGPU/IR/CGAEncodingAttr.td` | CGA encoding |
| `include/triton/Dialect/TritonGPU/IR/LinearLayoutConversions.h` | LL ↔ encoding 转换 |
| `include/triton/Tools/LinearLayout.h` | LinearLayout 核心数学工具 |
| `include/triton/Dialect/TritonGPU/Transforms/Passes.h` | 所有 Pass 声明 |
| `lib/Dialect/TritonGPU/Transforms/Coalesce.cpp` | 内存访问合并 |
| `lib/Dialect/TritonGPU/Transforms/AccelerateMatmul.cpp` | MMA 加速 |
| `lib/Dialect/TritonGPU/Transforms/RemoveLayoutConversions.cpp` | 冗余布局转换消除 |
| `lib/Dialect/TritonGPU/Transforms/Pipeliner/SoftwarePipeliner.cpp` | 软件流水线核心 |
| `lib/Dialect/TritonGPU/Transforms/WarpSpecialization/Partition.cpp` | Warp 特化核心 |
| `lib/Conversion/TritonGPUToLLVM/ConvertLayoutOpToLLVM.cpp` | convert_layout 降低 |
