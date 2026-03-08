# Triton IR 方言架构文档

## 一、概述

Triton IR 方言（`tt.*`）是 Triton 编译器的前端中间表示，位于 Python 前端与 TritonGPU IR 之间。其核心职责是提供**硬件无关的 tile 级并行抽象**，允许用户以接近数学表达式的方式描述 GPU kernel，而无需关心具体的线程映射和内存层次。

```
Python 前端 (triton.language)
      │  ast_to_ttir()
      ▼
Triton IR (tt.*)   ← 本文档的分析对象
      │  TritonToTritonGPU pass
      ▼
TritonGPU IR (ttg.*)
```

**方言标识**：`tt`，C++ 命名空间 `::mlir::triton`

**设计哲学**：
- **Tile-first**：操作对象是 tile（tensor block），而非标量或单个元素
- **硬件无关**：不包含任何 GPU 线程映射信息（encoding 为空）
- **显式内存**：指针操作显式，支持 block pointer 抽象
- **数学语义**：操作语义接近 NumPy/PyTorch，易于理解和优化

**源码位置**：
- 定义：`include/triton/Dialect/Triton/IR/`
- 实现：`lib/Dialect/Triton/IR/`
- Pass：`lib/Dialect/Triton/Transforms/`

---

## 二、类型系统

`include/triton/Dialect/Triton/IR/TritonTypes.td`

### 2.1 指针类型（`tt.ptr<T>`）

```
tt.ptr<f32>                    // 标量指针
tt.ptr<f32, 1>                 // 全局内存指针（地址空间 1）
tensor<32x!tt.ptr<f16>>        // 指针 tensor（每个元素是一个指针）
```

Triton 的指针类型是一等公民，支持：
- **标量指针**：`!tt.ptr<element_type, address_space>`
- **指针 tensor**：`tensor<shape x !tt.ptr<T>>`，表示一组指针
- **地址空间**：0=通用，1=全局，3=共享，4=常量，5=私有

与 LLVM 指针的区别：
- Triton 指针携带元素类型信息（typed pointer）
- 支持指针算术（`tt.addptr`）和指针比较
- 可以形成 tensor of pointers，用于 gather/scatter 访问模式

### 2.2 Tensor 类型

```
tensor<32x32xf16>                           // 2D tensor，无 encoding
tensor<32x32xf16, #ttg.blocked<{...}>>      // 带 TritonGPU encoding
```

在 Triton IR 阶段，tensor 类型的 `encoding` 属性为空（硬件无关）。`TritonToTritonGPU` pass 会为所有 tensor 分配初始 encoding。

### 2.3 Tensor Descriptor 类型（`tt.tensordesc<T>`）

```
!tt.tensordesc<tensor<32x32xf16>>
```

TMA（Tensor Memory Accelerator，Hopper/Blackwell）专用的 tensor 描述符类型，封装：
- 基地址指针
- shape 和 stride 信息
- swizzle 模式
- 边界检查信息

由 `tt.make_tensor_descriptor` 创建，用于 `tt.descriptor_load`/`tt.descriptor_store` 操作。

---

## 三、操作集（Ops）

`include/triton/Dialect/Triton/IR/TritonOps.td`

所有 `tt.*` Op 继承自 `TT_Op`，自动附加 `TensorSizeTrait` 和 `VerifyTensorLayoutsTrait` 两个验证 trait。

### 3.1 类型转换操作

#### `tt.int_to_ptr`

```mlir
%ptr = tt.int_to_ptr %addr : i64 -> !tt.ptr<f32>
%ptrs = tt.int_to_ptr %addrs : tensor<32xi64> -> tensor<32x!tt.ptr<f32>>
```

将整数地址转换为指针。支持标量和 tensor 操作。

Traits：`Elementwise`、`SameOperandsAndResultShape`、`SameOperandsAndResultEncoding`、`Pure`

#### `tt.ptr_to_int`

```mlir
%addr = tt.ptr_to_int %ptr : !tt.ptr<f32> -> i64
```

将指针转换为整数地址。

#### `tt.bitcast`

```mlir
%out = tt.bitcast %in : tensor<32xf32> -> tensor<32xi32>
%ptr_i32 = tt.bitcast %ptr_f32 : !tt.ptr<f32> -> !tt.ptr<i32>
```

位级重新解释，支持指针类型（`arith.bitcast` 不支持）。要求源类型和目标类型位宽相同。

#### `tt.fp_to_fp`

```mlir
%fp16 = tt.fp_to_fp %fp8 : tensor<32xf8e4nv> -> tensor<32xf16>
%fp8 = tt.fp_to_fp %fp16, rounding = rtne : tensor<32xf16> -> tensor<32xf8e4nv>
```

浮点类型转换，支持自定义类型（FP8 系列）和非默认舍入模式。

舍入模式（`TT_RoundingModeAttr`）：
- `rtne`：round to nearest even（默认）
- `rtz`：round toward zero
- `rtne_sat`：round to nearest even with saturation

### 3.2 指针操作

#### `tt.addptr`

```mlir
%new_ptr = tt.addptr %ptr, %offset : !tt.ptr<f32>, i32
%new_ptrs = tt.addptr %ptrs, %offsets : tensor<32x!tt.ptr<f32>>, tensor<32xi32>
```

指针算术，`new_ptr = ptr + offset * sizeof(element_type)`。支持标量和 tensor 操作。

#### `tt.make_block_ptr`

```mlir
%block_ptr = tt.make_block_ptr %base, %shape, %strides, %offsets, %block_shape, %order
  : (!tt.ptr<f32>, tensor<2xi64>, tensor<2xi64>, tensor<2xi32>, tensor<2xi32>, tensor<2xi32>)
  -> !tt.ptr<tensor<32x32xf32>>
```

创建 block pointer，封装对连续内存区域的结构化访问。参数：
- `base`：基地址指针
- `shape`：全局 tensor 形状
- `strides`：各维度步长（字节）
- `offsets`：当前 block 的起始偏移
- `block_shape`：block 大小
- `order`：维度顺序（最快变化轴在前）

返回类型为 `!tt.ptr<tensor<...>>`，指向一个 tensor block。

#### `tt.advance`

```mlir
%new_block_ptr = tt.advance %block_ptr, %offsets : !tt.ptr<tensor<32x32xf32>>, tensor<2xi32>
```

移动 block pointer 的偏移量，用于循环中迭代访问不同 tile。

### 3.3 内存操作

#### `tt.load`

```mlir
%data = tt.load %ptr : !tt.ptr<f32>
%data = tt.load %ptrs mask %mask other %other : tensor<32x!tt.ptr<f32>>
%data = tt.load %block_ptr boundary_check [0, 1] : !tt.ptr<tensor<32x32xf32>>
```

从内存加载数据。支持三种模式：
- **标量 load**：单个指针
- **gather load**：指针 tensor，每个线程加载不同地址
- **block load**：block pointer，加载连续 tile

可选参数：
- `mask`：掩码 tensor，控制哪些元素实际加载
- `other`：掩码为 false 时的默认值
- `cache`：缓存策略（`.ca`、`.cg`、`.cs`、`.lu`、`.cv`）
- `evict`：驱逐策略（`.evict_normal`、`.evict_first`、`.evict_last`）
- `isVolatile`：volatile 语义
- `boundary_check`：边界检查维度（block pointer 专用）

#### `tt.store`

```mlir
tt.store %ptr, %value : !tt.ptr<f32>, f32
tt.store %ptrs, %values mask %mask : tensor<32x!tt.ptr<f32>>, tensor<32xf32>
tt.store %block_ptr, %tile boundary_check [0, 1] : !tt.ptr<tensor<32x32xf32>>, tensor<32x32xf32>
```

向内存写入数据。参数与 `tt.load` 类似。

#### `tt.atomic_rmw` / `tt.atomic_cas`

```mlir
%old = tt.atomic_rmw fadd, %ptr, %val mask %mask sem = acq_rel scope = gpu
       : (!tt.ptr<f32>, f32) -> f32
%old = tt.atomic_cas %ptr, %cmp, %val sem = acq_rel scope = gpu
       : (!tt.ptr<i32>, i32, i32) -> i32
```

原子操作。`atomic_rmw` 支持的操作：`add`、`fadd`、`max`、`min`、`umax`、`umin`、`and`、`or`、`xor`、`xchg`。

内存序（`sem`）：`relaxed`、`acquire`、`release`、`acq_rel`、`seq_cst`

作用域（`scope`）：`cta`、`gpu`、`sys`

### 3.4 计算操作

#### `tt.dot`

```mlir
%c = tt.dot %a, %b, %acc {inputPrecision = tf32, maxNumImpreciseAcc = 0}
     : tensor<32x16xf32> * tensor<16x32xf32> -> tensor<32x32xf32>
```

矩阵乘累加：`C = A @ B + acc`。

参数：
- `inputPrecision`：输入精度（`tf32`、`ieee`、`bf16x3`、`bf16x6`）
- `maxNumImpreciseAcc`：允许的最大不精确累加次数（0 = 无限制）

`TritonGPU` 阶段会将此操作替换为硬件 MMA 指令（Tensor Core / MFMA / WMMA）。

#### `tt.dot_scaled`

```mlir
%c = tt.dot_scaled %a, %b, %acc, %a_scale, %b_scale
     : tensor<32x16xf8e4nv> * tensor<16x32xf8e4nv> -> tensor<32x32xf32>
```

带缩放因子的矩阵乘，用于量化推理。`C = (A * a_scale) @ (B * b_scale) + acc`。

#### `tt.reduce`

```mlir
%sum = tt.reduce %src {axis = 1, redOp = 0 : i32}
       : tensor<32x32xf32> -> tensor<32xf32>
```

沿指定轴归约。`redOp` 枚举：
- `0`：`add`（求和）
- `1`：`mul`（求积）
- `2`：`max`
- `3`：`min`
- `4`：`umax`
- `5`：`umin`
- `6`：`fadd`
- `7`：`fmax`
- `8`：`fmin`
- `9`：`xor`（XOR 归约）

#### `tt.scan`

```mlir
%prefix_sum = tt.scan %src {axis = 0, reverse = false}
              : tensor<32x32xf32> -> tensor<32x32xf32>
```

前缀扫描（prefix scan / cumulative sum）。`reverse = true` 时为后缀扫描。

#### `tt.histogram`

```mlir
tt.histogram %src, %dst : tensor<32xi32>, !tt.ptr<tensor<256xi32>>
```

直方图操作，将 `src` 中的值作为索引，对 `dst` 对应位置原子递增。

### 3.5 形状操作

#### `tt.reshape`

```mlir
%out = tt.reshape %in {allow_reorder = false} : tensor<32x32xf32> -> tensor<1024xf32>
```

重塑 tensor 形状。`allow_reorder = true` 时允许改变元素顺序（用于优化）。

#### `tt.broadcast`

```mlir
%out = tt.broadcast %in : tensor<32xf32> -> tensor<32x32xf32>
```

广播 tensor 到更高维度。

#### `tt.trans`

```mlir
%out = tt.trans %in {order = [1, 0]} : tensor<32x16xf32> -> tensor<16x32xf32>
```

转置 tensor。`order` 指定维度重排顺序。

#### `tt.expand_dims`

```mlir
%out = tt.expand_dims %in {axis = 1} : tensor<32xf32> -> tensor<32x1xf32>
```

在指定位置插入大小为 1 的维度。

#### `tt.cat`

```mlir
%out = tt.cat %a, %b {axis = 0} : tensor<32x16xf32>, tensor<32x16xf32> -> tensor<64x16xf32>
```

沿指定轴拼接 tensor。

#### `tt.join` / `tt.split`

```mlir
%out = tt.join %a, %b : tensor<32xf32>, tensor<32xf32> -> tensor<32x2xf32>
%a, %b = tt.split %in : tensor<32x2xf32> -> tensor<32xf32>, tensor<32xf32>
```

`join` 将两个 tensor 合并为一个新维度，`split` 是其逆操作。

### 3.6 TMA 操作（Hopper/Blackwell）

#### `tt.make_tensor_descriptor`

```mlir
%desc = tt.make_tensor_descriptor %base, %shape, %strides, %offsets, %block_shape, %order
        : (!tt.ptr<f32>, ...) -> !tt.tensordesc<tensor<32x32xf16>>
```

创建 TMA 描述符，封装 tensor 的内存布局信息。

#### `tt.descriptor_load` / `tt.descriptor_store`

```mlir
%data = tt.descriptor_load %desc : !tt.tensordesc<tensor<32x32xf16>> -> tensor<32x32xf16>
tt.descriptor_store %desc, %data : !tt.tensordesc<tensor<32x32xf16>>, tensor<32x32xf16>
```

使用 TMA 描述符进行批量数据传输（全局内存 ↔ 共享内存）。

#### `tt.descriptor_gather` / `tt.descriptor_scatter`

```mlir
%data = tt.descriptor_gather %desc, %indices : !tt.tensordesc<...>, tensor<32xi32> -> tensor<32xf16>
tt.descriptor_scatter %desc, %indices, %data : !tt.tensordesc<...>, tensor<32xi32>, tensor<32xf16>
```

使用 TMA 描述符进行 gather/scatter 访问。

### 3.7 程序控制操作

#### `tt.get_program_id` / `tt.get_num_programs`

```mlir
%pid = tt.get_program_id x : i32
%num = tt.get_num_programs y : i32
```

获取当前 program（grid 中的 block）的 ID 和总数。轴参数：`x`、`y`、`z`。

#### `tt.func` / `tt.call` / `tt.return`

```mlir
tt.func @kernel(%arg0: !tt.ptr<f32>, %arg1: i32) {
  %data = tt.load %arg0 : !tt.ptr<f32>
  tt.return
}

tt.call @helper(%ptr) : (!tt.ptr<f32>) -> ()
```

函数定义、调用和返回。`tt.func` 支持 `noinline` 属性。

### 3.8 调试操作

#### `tt.print`

```mlir
tt.print "value = " {hex = false} : %val : tensor<32xf32>
```

设备端打印（device printf）。可选 `hex = true` 以十六进制格式打印。

#### `tt.assert`

```mlir
tt.assert %cond, "assertion failed", "file.py", "func", 42 : i1
```

设备端断言。参数：条件、消息、文件名、函数名、行号。

### 3.9 其他操作

#### `tt.clampf` / `tt.precise_sqrt` / `tt.precise_divf`

```mlir
%out = tt.clampf %in, %min, %max propagateNan = all : tensor<32xf32>
%out = tt.precise_sqrt %in : tensor<32xf32>
%out = tt.precise_divf %a, %b : tensor<32xf32>
```

精确数学操作，保证 IEEE 754 语义（不使用快速数学优化）。

#### `tt.mulhiui` / `tt.mulhisi`

```mlir
%hi = tt.mulhiui %a, %b : tensor<32xi32>
```

乘法高位（`(a * b) >> 32`），用于大整数运算。

---

## 四、属性系统

`include/triton/Dialect/Triton/IR/TritonAttrDefs.td`

### 4.1 `tt.divisibility`

```mlir
%ptr {tt.divisibility = 16 : i32}
```

标记指针或整数值可被某个 2 的幂整除，用于优化内存访问对齐。

### 4.2 `tt.contiguity`

```mlir
%ptr {tt.contiguity = dense<[1, 32]> : tensor<2xi32>}
```

标记指针 tensor 的连续性信息，用于向量化优化。值表示各维度上连续元素的最大数量。

### 4.3 `tt.constancy`

```mlir
%val {tt.constancy = dense<[1, 0]> : tensor<2xi32>}
```

标记 tensor 各维度上的常量性。`1` 表示该维度上所有元素相同，`0` 表示可能不同。

### 4.4 `tt.max_contiguous`

```mlir
%ptr {tt.max_contiguous = 8 : i32}
```

标记指针 tensor 的最大连续访问宽度，用于选择最优的 load/store 指令。

### 4.5 `tt.multiple_of`

```mlir
%val {tt.multiple_of = 16 : i32}
```

标记整数值是某个常量的倍数，用于循环展开和边界检查优化。

---

## 五、Traits 与 Interfaces

`include/triton/Dialect/Triton/IR/TritonInterfaces.td`

### 5.1 Op Traits

| Trait | 说明 |
|-------|------|
| `Elementwise` | 逐元素操作，保持 shape 和 encoding |
| `SameOperandsAndResultShape` | 操作数与结果 shape 相同 |
| `SameOperandsAndResultEncoding` | 操作数与结果 encoding 相同 |
| `SameOperandsAndResultElementType` | 操作数与结果元素类型相同 |
| `TensorSizeTrait` | 自动附加到所有 `TT_Op`，验证 tensor 大小 |
| `VerifyTensorLayoutsTrait` | 自动附加到所有 `TT_Op`，验证 encoding 一致性 |

### 5.2 Op Interfaces

| Interface | 实现者 | 核心方法 |
|-----------|--------|---------|
| `MemoryEffectOpInterface` | load/store/atomic | `getEffects()`：声明内存副作用 |
| `InferTypeOpInterface` | reshape/broadcast | `inferReturnTypes()`：推断返回类型 |
| `DestinationStyleOpInterface` | histogram | `getDpsInits()`：获取目标操作数 |

### 5.3 Resource

```tablegen
def GlobalMemory : Resource<"::mlir::triton::GlobalMemory">;
```

`tt.load`、`tt.store`、`tt.atomic_*` 操作声明访问 `GlobalMemory` resource，用于内存依赖分析和优化。

---

## 六、优化 Pass 体系

`lib/Dialect/Triton/Transforms/`

| Pass 文件 | Pass 名称 | 职责 |
|-----------|-----------|------|
| `Combine.cpp` | `triton-combine` | 模式匹配优化（常量折叠、代数简化、死代码消除） |
| `RewriteTensorPointer.cpp` | `triton-rewrite-tensor-pointer` | 将 block pointer 重写为标量指针 + 偏移计算（旧架构兼容） |
| `RewriteTensorDescriptorToPointer.cpp` | `triton-rewrite-tensor-descriptor-to-pointer` | 将 TMA 描述符重写为普通指针（<SM90 兼容） |
| `ReorderBroadcast.cpp` | `triton-reorder-broadcast` | 将 broadcast 操作下沉到使用点，减少中间 tensor 大小 |
| `LoopUnroll.cpp` | `triton-loop-unroll` | 循环展开，支持部分展开和完全展开 |
| `LoopInvariantCodeMotion.cpp` | `triton-licm` | 循环不变量外提（LICM） |
| `LoopAwareCSE.cpp` | `triton-loop-aware-cse` | 循环感知的公共子表达式消除 |
| `LoopPeeling.cpp` | `triton-loop-peeling` | 循环剥离，将首尾迭代分离出来 |
| `ArithTypeConversion.cpp` | `triton-arith-type-conversion` | 算术类型转换（i1 → i8 等） |
| `FunctionTypeConversion.cpp` | `triton-function-type-conversion` | 函数签名类型转换 |

### 6.1 Combine Pass 优化模式

`Combine.cpp` 实现了大量模式匹配优化，包括：

**常量折叠**：
- `addptr(ptr, 0)` → `ptr`
- `broadcast(splat(x))` → `splat(x)`
- `reshape(reshape(x))` → `reshape(x)`

**代数简化**：
- `dot(a, b, 0)` → `dot(a, b)`（零累加器消除）
- `reduce(broadcast(x))` → `x`（广播后归约消除）
- `trans(trans(x))` → `x`（双重转置消除）

**死代码消除**：
- 未使用的 `load` 操作
- 未使用的 `make_block_ptr` 操作

### 6.2 Tensor Pointer 重写

`RewriteTensorPointer.cpp` 将高级 block pointer API 降低为标量指针算术：

```mlir
// 重写前
%block_ptr = tt.make_block_ptr %base, %shape, %strides, %offsets, [32, 32], [1, 0]
%data = tt.load %block_ptr

// 重写后
%ptr0 = tt.addptr %base, %offset0
%ptr1 = tt.addptr %ptr0, %offset1
...
%data = tt.load %ptrs mask %mask
```

这是为了兼容不支持 TMA 的旧架构（<SM90）。

### 6.3 循环优化

**LoopUnroll**：
- 支持 `#pragma unroll` 注解
- 自动展开小循环（trip count < 8）
- 部分展开大循环（unroll factor = 4）

**LICM**：
- 外提循环不变的 `load` 操作
- 外提循环不变的指针计算
- 外提循环不变的 `make_block_ptr` 操作

**LoopAwareCSE**：
- 跨循环迭代的公共子表达式消除
- 识别循环归纳变量的线性关系
- 消除冗余的边界检查

---

## 七、Python 前端绑定

### 7.1 `triton.language` 核心 API

`python/triton/language/core.py`

| Python API | 对应 TTIR Op | 说明 |
|------------|--------------|------|
| `tl.load(ptr, mask, other)` | `tt.load` | 加载数据 |
| `tl.store(ptr, val, mask)` | `tt.store` | 存储数据 |
| `tl.dot(a, b, acc)` | `tt.dot` | 矩阵乘累加 |
| `tl.sum(x, axis)` | `tt.reduce` | 求和归约 |
| `tl.max(x, axis)` | `tt.reduce` | 最大值归约 |
| `tl.cumsum(x, axis)` | `tt.scan` | 前缀和 |
| `tl.trans(x)` | `tt.trans` | 转置 |
| `tl.broadcast_to(x, shape)` | `tt.broadcast` | 广播 |
| `tl.make_block_ptr(...)` | `tt.make_block_ptr` | 创建 block pointer |
| `tl.advance(ptr, offsets)` | `tt.advance` | 移动 block pointer |
| `tl.atomic_add(ptr, val)` | `tt.atomic_rmw` | 原子加 |
| `tl.program_id(axis)` | `tt.get_program_id` | 获取 program ID |
| `tl.num_programs(axis)` | `tt.get_num_programs` | 获取 program 总数 |
| `tl.multiple_of(val, n)` | 附加 `tt.multiple_of` 属性 | 标记倍数关系 |
| `tl.max_contiguous(ptr, n)` | 附加 `tt.max_contiguous` 属性 | 标记连续性 |

### 7.2 语义分析（AST → TTIR）

`python/triton/language/semantic.py`

**类型推断**：
- 整数字面量 → `i32`
- 浮点字面量 → `f32`
- 二元操作自动类型提升（`i32 + i64` → `i64`）
- 指针算术自动缩放（`ptr + 1` → `ptr + sizeof(T)`）

**控制流映射**：
- Python `if` → `scf.if`
- Python `for` → `scf.for`
- Python `while` → `scf.while`

**常量折叠**：
- `constexpr` 参数在编译时求值
- 常量表达式折叠（`2 + 3` → `5`）
- 常量传播（`x = 5; y = x + 1` → `y = 6`）

**属性推导**：
- 自动推导 `tt.divisibility`（`ptr = base + i * 16` → `divisibility = 16`）
- 自动推导 `tt.contiguity`（`arange(0, 32)` → `contiguity = 32`）
- 自动推导 `tt.constancy`（`broadcast(x)` → `constancy = [1, 0]`）

---

## 八、与其他方言的关系

```
┌─────────────────────────────────────────────────────────┐
│                Python 前端 (triton.language)             │
│  tl.load / tl.store / tl.dot / tl.sum / ...             │
└──────────────────────────┬──────────────────────────────┘
                           │ ast_to_ttir()
                           │ (semantic.py)
┌──────────────────────────▼──────────────────────────────┐
│                  Triton IR (tt.*)                        │
│  硬件无关的 tile 级操作，encoding 为空                    │
│                                                          │
│  依赖方言：                                               │
│  - mlir::arith（算术操作）                                │
│  - mlir::scf（结构化控制流）                              │
│  - mlir::cf（控制流）                                     │
└──────────────────────────┬──────────────────────────────┘
                           │ TritonToTritonGPU pass
                           │ (分配初始 #ttg.blocked encoding)
┌──────────────────────────▼──────────────────────────────┐
│                TritonGPU IR (ttg.*)                      │
│  布局感知，带 encoding 的 tensor                          │
└──────────────────────────────────────────────────────────┘
```

**依赖的标准方言**：
- `arith`：算术操作（`arith.addi`、`arith.mulf` 等）
- `scf`：结构化控制流（`scf.if`、`scf.for`、`scf.while`）
- `cf`：控制流（`cf.br`、`cf.cond_br`）
- `func`：函数定义（`func.func`、`func.call`、`func.return`）

**Triton IR 的独特性**：
- 不依赖 `linalg` 方言（Triton 有自己的 tile 抽象）
- 不依赖 `tensor` 方言（Triton tensor 是分布式的，不是 SSA 值）
- 显式指针操作（不使用 `memref`）

---

## 九、编译流程中的位置

```
Python 源码
  │
  ├─ DependenciesFinder（AST 遍历 + SHA256 缓存键）
  │
  ├─ ast_to_ttir()  ← 生成 Triton IR
  │    └─ semantic.py：类型推断、控制流映射、常量折叠
  │
  ├─ make_ttir()  ← Triton IR 清理 Pass
  │    ├─ inliner：内联函数调用
  │    ├─ rewrite_tensor_pointer：block pointer 重写（可选）
  │    ├─ canonicalizer：规范化
  │    ├─ combine：模式匹配优化
  │    ├─ reorder_broadcast：broadcast 下沉
  │    ├─ cse：公共子表达式消除
  │    ├─ symbol_dce：符号死代码消除
  │    └─ loop_unroll：循环展开
  │
  ├─ TritonToTritonGPU pass  ← 转换为 TritonGPU IR
  │    └─ 为所有 tensor 分配初始 #ttg.blocked encoding
  │
  └─ make_ttgir()  ← TritonGPU 优化 Pass（见 TritonGPU 文档）
```

**Triton IR 的职责边界**：
- **输入**：Python AST（经过 `semantic.py` 类型检查）
- **输出**：硬件无关的 tile 级 IR，准备进行布局分配
- **不包含**：线程映射、共享内存分配、warp 特化、软件流水线（这些在 TritonGPU 阶段）

---

## 十、关键设计决策

### 10.1 为什么需要 Block Pointer？

传统 GPU 编程需要手动计算每个线程的指针偏移：

```python
# 传统方式（CUDA）
tid = threadIdx.x + blockIdx.x * blockDim.x
ptr = base + tid * stride
data = load(ptr)
```

Block Pointer 抽象了这些细节：

```python
# Triton 方式
block_ptr = tl.make_block_ptr(base, shape, strides, offsets, [32, 32], [1, 0])
data = tl.load(block_ptr)  # 自动处理边界检查和掩码
```

优势：
- **简洁**：一行代码替代数十行指针计算
- **安全**：自动边界检查，避免越界访问
- **优化**：编译器可以识别访问模式，生成最优代码

### 10.2 为什么 Tensor 不是 SSA 值？

MLIR 标准 `tensor` 类型是不可变的 SSA 值，但 Triton tensor 是**分布式的可变状态**：

```mlir
// MLIR 标准 tensor（SSA）
%t1 = tensor.empty() : tensor<32x32xf32>
%t2 = linalg.fill ins(%cst) outs(%t1) : tensor<32x32xf32>

// Triton tensor（分布式）
%t = tt.load %ptr : tensor<32x32xf32, #blocked>
// 每个线程持有 tensor 的一部分（由 encoding 决定）
```

Triton tensor 的 encoding 决定了数据在 GPU 线程间的分布，这是编译时确定的静态信息，而非运行时的动态值。

### 10.3 为什么需要 TMA 描述符？

Hopper/Blackwell 架构引入了 TMA（Tensor Memory Accelerator），可以在一条指令中传输整个 tile（最大 256KB）：

```mlir
// 传统方式（cp.async，每个线程拷贝一部分）
%tok = ttg.async_copy_global_to_local %ptrs, %buf : tensor<32x!tt.ptr<f16>> -> !ttg.memdesc<...>

// TMA 方式（单条指令拷贝整个 tile）
%desc = tt.make_tensor_descriptor %base, %shape, %strides, ...
%data = tt.descriptor_load %desc : !tt.tensordesc<tensor<32x32xf16>> -> tensor<32x32xf16>
```

TMA 的优势：
- **带宽**：接近理论峰值（~2TB/s）
- **延迟**：单周期发射，异步执行
- **简洁**：一条指令替代数百条 load 指令

---

## 十一、关键文件速查

| 文件 | 说明 |
|------|------|
| `include/triton/Dialect/Triton/IR/TritonDialect.td` | 方言定义 |
| `include/triton/Dialect/Triton/IR/TritonTypes.td` | 类型定义（ptr、tensordesc） |
| `include/triton/Dialect/Triton/IR/TritonOps.td` | 所有操作定义（~2000 行） |
| `include/triton/Dialect/Triton/IR/TritonAttrDefs.td` | 属性定义（divisibility、contiguity 等） |
| `include/triton/Dialect/Triton/IR/TritonInterfaces.td` | 接口定义 |
| `lib/Dialect/Triton/IR/Ops.cpp` | 操作实现（验证、类型推断） |
| `lib/Dialect/Triton/Transforms/Combine.cpp` | 模式匹配优化 |
| `lib/Dialect/Triton/Transforms/RewriteTensorPointer.cpp` | Block pointer 重写 |
| `python/triton/language/core.py` | Python API（~3000 行） |
| `python/triton/language/semantic.py` | AST → TTIR 转换（~2000 行） |
| `python/triton/compiler/code_generator.py` | IR 生成入口 |
