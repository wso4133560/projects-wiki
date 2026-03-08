# NVIDIA SM 分支 Pass 序列深度分析

> 基于 `third_party/nvidia/backend/compiler.py`（commit `526cdf4`）完整源码逐行解析。
> 三条编译路径：SM7x（Volta/Turing）/ SM8x-9x（Ampere/Hopper）/ SM10x+（Blackwell）

---

## 目录

1. [架构判断逻辑与分支条件](#1-架构判断逻辑与分支条件)
2. [make_ttir 阶段差异](#2-make_ttir-阶段差异)
3. [make_ttgir 完整 Pass 序列对照](#3-make_ttgir-完整-pass-序列对照)
4. [各 Pass 详细说明](#4-各-pass-详细说明)
   - [4.1 共享前段 Pass（所有 SM）](#41-共享前段-pass所有-sm)
   - [4.2 SM7x 专属路径](#42-sm7x-专属路径)
   - [4.3 SM8x/9x 专属路径（Hopper Warp Spec）](#43-sm8x9x-专属路径hopper-warp-spec)
   - [4.4 SM10x+ 专属路径（Blackwell TMEM）](#44-sm10x-专属路径blackwell-tmem)
   - [4.5 共享后段 Pass（所有 SM）](#45-共享后段-pass所有-sm)
5. [make_llir 阶段差异](#5-make_llir-阶段差异)
6. [parse_options 中的类型与能力差异](#6-parse_options-中的类型与能力差异)
7. [Warp 专用化机制对比（SM8x-9x vs SM10x+）](#7-warp-专用化机制对比sm8x-9x-vs-sm10x)
8. [TMEM（张量内存）完整流程（仅 SM10x+）](#8-tmem张量内存完整流程仅-sm10x)
9. [软件流水线机制对比](#9-软件流水线机制对比)
10. [各 SM 特性能力矩阵](#10-各-sm-特性能力矩阵)

---

## 1. 架构判断逻辑与分支条件

### 1.1 分支判断语句（原始代码）

```python
# third_party/nvidia/backend/compiler.py, make_ttgir()

emuTF32 = (capability // 10 >= 8)   # SM80+: f32 dot 用 TF32 近似

if capability // 10 in [8, 9]:      # SM8x 和 SM9x（Ampere + Hopper）
    # Hopper Warp Spec + 流水线
    ...
elif capability // 10 >= 10:        # SM10x+（Blackwell）
    # TMEM + 新式 Warp Spec + 流水线
    ...
else:                               # SM7x（Volta + Turing）
    passes.ttir.add_triton_licm(pm)  # 仅此一个 Pass
```

### 1.2 SM 代号与 GPU 映射

| `capability` 值 | `capability // 10` | GPU 代号 | 典型产品 |
|----------------|-------------------|---------|---------|
| 70 | 7 | Volta SM70 | V100 |
| 75 | 7 | Turing SM75 | RTX 20, T4 |
| 80 | 8 | Ampere SM80 | A100, A30 |
| 86 | 8 | Ampere SM86 | RTX 30 |
| 87 | 8 | Ampere SM87 | Jetson Orin |
| 89 | 8 | Ada Lovelace SM89 | RTX 40 |
| 90 | 9 | Hopper SM90 | H100, H800 |
| 100 | 10 | Blackwell SM100 | B100, B200 |
| 120 | 12 | Blackwell SM120 | RTX 50 |

**关键边界说明**：
- `capability >= 80`（SM80+）：启用 HW 转置优化（`optimize_dot_operands` 第二参数为 `True`）
- `capability >= 89`（SM89+，Ada）：原生 FP8 e4m3 支持
- `capability >= 90`（SM90+，Hopper）：TMA 降级、异步流水线、FP8 e4b15 废弃
- `capability // 10 >= 10`（SM100+，Blackwell）：TMEM、新 Warp Spec、MMAv5

---

## 2. make_ttir 阶段差异

```python
@staticmethod
def make_ttir(mod, metadata, opt, capability):
    pm = ir.pass_manager(mod.context)
    pm.enable_debug()

    passes.common.add_inliner(pm)                    # 所有 SM：函数内联
    passes.ttir.add_rewrite_tensor_pointer(pm)       # 所有 SM：指针规范化

    # ★ 关键分支：能否保留原生张量描述符
    if capability // 10 < 9:                         # SM7x/SM8x
        passes.ttir.add_rewrite_tensor_descriptor_to_pointer(pm)
    # SM9x+：保留 tt.tensordesc，供 TMA 路径使用

    passes.common.add_canonicalizer(pm)
    passes.ttir.add_combine(pm)
    passes.ttir.add_reorder_broadcast(pm)
    passes.common.add_cse(pm)
    passes.common.add_symbol_dce(pm)
    passes.ttir.add_loop_unroll(pm)
    pm.run(mod, 'make_ttir')
    return mod
```

**`add_rewrite_tensor_descriptor_to_pointer` 的作用**：

在 SM7x/SM8x 中，`tt.load(tensordesc, coord)` 无法映射到 TMA 硬件，必须将其展开为普通的索引计算 + `tt.load(ptr)`。SM9x+ 保留 `tensordesc`，后续由 `add_tma_lowering` 降级为 `ttng.async_tma_copy_global_to_local`。

---

## 3. make_ttgir 完整 Pass 序列对照

以下完整展现三个分支的 Pass 序列。**粗体**为该分支专有；正常字体为共享 Pass。

```
make_ttgir() 调用顺序
══════════════════════════════════════════════════════════════════════════════
序号  Pass 函数调用                               SM7x   SM8x-9x  SM10x+
══════════════════════════════════════════════════════════════════════════════
# ──── 共享前段（所有 SM 均执行）────────────────────────────────────────────
 1   add_convert_to_ttgpuir(...)                  ✓       ✓       ✓
 2   add_coalesce(pm)                             ✓       ✓       ✓
 3   add_f32_dot_tc(pm, emuTF32)                  ✓(F)    ✓(T)    ✓(T)
 4   add_plan_cta(pm)                [NVIDIA]     ✓       ✓       ✓
 5   add_remove_layout_conversions(pm)            ✓       ✓       ✓
 6   add_optimize_thread_locality(pm)             ✓       ✓       ✓
 7   add_accelerate_matmul(pm)                    ✓       ✓       ✓
 8   add_remove_layout_conversions(pm)            ✓       ✓       ✓
 9   add_optimize_dot_operands(pm, cap>=80)       ✓(F)    ✓(T)    ✓(T)
10   add_optimize_descriptor_encoding(pm)[NVIDIA] ✓       ✓       ✓
11   add_loop_aware_cse(pm)                       ✓       ✓       ✓

# ──── 中段分支（三路互斥）──────────────────────────────────────────────────
     ── SM7x 唯一 ──
12a  add_triton_licm(pm)                          ✓

     ── SM8x/9x：Hopper Warp Spec ──
12b  add_fuse_nested_loops(pm)                            ✓
13b  add_canonicalizer(pm)                                ✓
14b  add_triton_licm(pm)                                  ✓
15b  add_canonicalizer(pm)                                ✓
16b  add_combine_tensor_select_and_if(pm)                 ✓
17b  add_hopper_warpspec(pm, stages, dump)  [Hopper专]    ✓
18b  add_assign_latencies(pm, num_stages)                 ✓
19b  add_schedule_loops(pm)                               ✓
20b  add_pipeline(pm, num_stages, dump)                   ✓

     ── SM10x+：Blackwell TMEM Warp Spec ──
12c  add_fuse_nested_loops(pm)                                    ✓
13c  add_canonicalizer(pm)                                        ✓
14c  add_triton_licm(pm)                                          ✓
15c  add_optimize_accumulator_init(pm)             [新]            ✓
16c  add_hoist_tmem_alloc(pm, False)               [新]            ✓
17c  add_promote_lhs_to_tmem(pm)               [Blackwell专]       ✓
18c  add_assign_latencies(pm, num_stages)                         ✓
19c  add_schedule_loops(pm)                                       ✓
20c  add_warp_specialize(pm, num_stages)         [新式]             ✓
21c  add_pipeline(pm, num_stages, dump)                           ✓
22c  add_optimize_partition_warps(pm)              [新]            ✓
23c  add_combine_tensor_select_and_if(pm)                         ✓
24c  add_hoist_tmem_alloc(pm, True)              [二次]            ✓
25c  add_remove_tmem_tokens(pm)                  [新]              ✓

# ──── 共享后段（所有 SM 均执行）────────────────────────────────────────────
26   add_canonicalizer(pm)                        ✓       ✓       ✓
27   add_loop_aware_cse(pm)                       ✓       ✓       ✓
28   add_prefetch(pm)                             ✓       ✓       ✓
29   add_optimize_dot_operands(pm, cap>=80)       ✓(F)    ✓(T)    ✓(T)
30   add_coalesce_async_copy(pm)                  ✓       ✓       ✓
31   add_optimize_tmem_layouts(pm)   [实质SM10x+] ✓(空)   ✓(空)   ✓(有效)
32   add_tma_lowering(pm) [仅 cap//10>=9]         ✗       ✓(9x)   ✓
33   add_remove_layout_conversions(pm)            ✓       ✓       ✓
34   add_interleave_tmem(pm)         [实质SM10x+] ✓(空)   ✓(空)   ✓(有效)
35   add_reduce_data_duplication(pm)              ✓       ✓       ✓
36   add_reorder_instructions(pm)                 ✓       ✓       ✓
37   add_loop_aware_cse(pm)                       ✓       ✓       ✓
38   add_symbol_dce(pm)                           ✓       ✓       ✓
39   add_fence_insertion(pm, cap)      [参数化]   ✓       ✓       ✓
40   add_lower_mma(pm)                            ✓       ✓       ✓
41   add_sccp(pm)                                 ✓       ✓       ✓
42   add_cse(pm)                                  ✓       ✓       ✓
43   add_canonicalizer(pm)                        ✓       ✓       ✓
══════════════════════════════════════════════════════════════════════════════
总 Pass 数（含重复）                             13      23      30
```

---

## 4. 各 Pass 详细说明

### 4.1 共享前段 Pass（所有 SM）

#### `add_convert_to_ttgpuir(pm, f"cuda:{capability}", num_warps, 32, num_ctas)`

将设备无关的 TTIR 转换为 TTGIR，同时分配初始 layout encoding：

- 为所有 `tt.dot` 的结果张量分配 `BlockedEncoding`（稍后由 `accelerate_matmul` 改为 MMA encoding）
- 根据 `num_warps`、`warp_size=32`、`num_ctas` 计算每个张量的初始分布
- 插入 `ttg.convert_layout` 以维护数据一致性

---

#### `add_coalesce(pm)` — `tritongpu-coalesce`

**目的**：优化全局内存访问的 layout，使同一 warp 的线程访问连续地址（coalesced access）。

**工作方式**：
1. 分析所有 `tt.load` / `tt.store` 的 tensor 类型和当前 layout
2. 若当前 layout 导致非合并访问，插入新的 `ttg.convert_layout`，将 load/store 前后的 encoding 替换为对合并友好的 `BlockedEncoding`

**对所有 SM 的影响**：此 Pass 与架构无关，通用优化。

---

#### `add_f32_dot_tc(pm, emuTF32)` — `tritongpu-F32DotTC`

**目的**：处理 `f32 × f32 → f32` 的矩阵乘，将其映射到 TensorCore。

**两种模式**（由 `emuTF32` 控制）：

| 模式 | 触发条件 | 实现方式 |
|------|---------|---------|
| `emuTF32=False`（SM7x） | capability < 80 | 分解为 3 个 f16 DotOp + 4 个 pointwise op（CUTLASS 方案） |
| `emuTF32=True`（SM8x+） | capability >= 80 | 将 f32 输入截断为 TF32（19-bit），直接走 TC 路径，精度略损 |

**SM7x 的分解方式**（来自 Pass 文档注释，参考 CUTLASS `#385`）：
```
f32 a×b = f16(a)×f16(b) + (a-f16(a))×f16(b) + f16(a)×(b-f16(b))
```
这样可以在 SM7x 的 f16 TensorCore 上完成 f32 精度计算。

---

#### `add_plan_cta(pm)` — `triton-nvidia-gpu-plan-cta` **[NVIDIA 专有]**

**目的**：计算并应用"最优"CTA tiling，影响 `DotOp`、`ReduceOp`、`StoreLikeOps`。

**具体行为**：
- 分析每个 `tt.dot` 的张量形状
- 根据 warp 数量和 SM 的共享内存容量，选择最优的 CTA 内 tile 划分
- 将计算结果写入 `tt.dot` 的 layout attribute（`num_warps`、分块维度）

**为何在 `coalesce` 之后运行**：代码注释写明这是 TODO，理论上应在 `coalesce` 之前以避免 layout 不一致。

---

#### `add_remove_layout_conversions(pm)` — `tritongpu-remove-layout-conversions`

**目的**：消除冗余的 `ttg.convert_layout` op。

**工作方式**：
1. 分析每个 `ttg.convert_layout` 的源/目标 encoding
2. 若 source 和 target encoding 相同，消除该 op
3. 若两个连续的 convert_layout 可以合并（A→B→C 变为 A→C），则合并
4. 优先选择对 load/store 友好的 `BlockedEncoding`，对 TC 操作优先 `MmaEncoding`

**运行位置**：第一次在 `plan_cta` 之后、`accelerate_matmul` 之前；第二次在 `accelerate_matmul` 之后清理新插入的转换。

---

#### `add_optimize_thread_locality(pm)` — `tritongpu-optimize-thread-locality`

**目的**：减少跨线程通信，优化 reduction 和 gather 操作。

**具体优化**：
1. **Reduction 优化**：对循环内 yield 的 reduction，延迟 reduce 到循环结束后再跨线程规约，使线程内部积累避免反复 shuffle
2. **Gather 优化**：为 gather op 选择使整个 warp 协同完成的 layout，触发 warp-synchronous gather 代码路径

---

#### `add_accelerate_matmul(pm)` — `tritongpu-accelerate-matmul`

**目的**：这是最关键的 Pass，将 `tt.dot` 的 layout 从 `BlockedEncoding` 替换为 `MmaEncoding`，触发 TensorCore 代码路径。

**核心决策逻辑**（按架构选择 MMA 版本）：

| SM 版本 | MMA 版本 | 指令集 | 支持的类型 |
|---------|---------|-------|-----------|
| SM70-75 | MMAv1 | `wmma.*` | f16, i8 |
| SM80-86 | MMAv2 | `mma.sync.aligned.*` | f16, bf16, tf32, i8 |
| SM89 | MMAv2 | 同上 + fp8路径 | + fp8e4m3 |
| SM90 | MMAv3（WGMMA） | `wgmma.mma_async.*` | f16, bf16, tf32, fp8 |
| SM100+ | MMAv5（TCGen5） | `tcgen05.mma.*` | f16, bf16, fp8, fp4 |

**运行后 IR 变化**（示例）：
```mlir
# 之前（BlockedEncoding）：
%c = tt.dot %a, %b, %acc : tensor<128x64xf16, #blocked> * ... -> tensor<128x128xf32, #blocked>

# 之后（NvidiaMmaEncoding，SM90）：
%c = tt.dot %a, %b, %acc : tensor<128x64xf16, #shared> * ... -> tensor<128x128xf32, #mma<{v=3,...}>>
```

---

#### `add_optimize_dot_operands(pm, capability >= 80)` — `tritongpu-optimize-dot-operands`

**目的**：重排矩阵操作数的 layout，利用硬件转置能力。

**参数含义**：
- `hoistLayoutConversion=False`（SM7x）：不利用硬件转置，仅做标准 layout 优化
- `hoistLayoutConversion=True`（SM80+）：将 `convert_layout` 提前，跨越 elementwise op，利用 SM80+ 的硬件级转置能力（TensorCore 可在不额外转置的情况下读取转置后的操作数）

---

#### `add_optimize_descriptor_encoding(pm)` — `triton-nvidia-optimize-descriptor-encoding` **[NVIDIA 专有]**

**目的**：为 TMA 描述符（`tt.tensordesc` / `ttng.tensordesc_im2col`）设置最优的共享内存 encoding，决定 swizzling 模式和消息大小。

**触发条件**：仅在 SM9x+（有 TMA 硬件）时有实质效果；在 SM7x/SM8x 中 tensordesc 已被 `add_rewrite_tensor_descriptor_to_pointer` 消除，此 Pass 为空操作。

---

#### `add_loop_aware_cse(pm)`

**目的**：Loop-aware 公共子表达式消除。与普通 CSE 不同，它理解循环语义，可消除跨循环迭代的重复计算。

---

### 4.2 SM7x 专属路径

#### `add_triton_licm(pm)` — 仅在 SM7x 中段执行

**目的**：循环不变代码提升（Loop-Invariant Code Motion）。

**为何 SM7x 单独运行 LICM**：SM7x 没有软件流水线，没有 Warp 专用化，需要在此处通过 LICM 减少循环内的冗余计算。SM8x+ 的 LICM 在流水线相关 Pass 之后作为后段 Pass 运行（统一在 `add_triton_licm` 调用）。

**SM7x 中段 Pass 数量**：仅 1 个，这也反映了 SM7x 的编译优化复杂度远低于 Hopper/Blackwell。

---

### 4.3 SM8x/9x 专属路径（Hopper Warp Spec）

#### `add_fuse_nested_loops(pm)` — `tritongpu-fuse-nested-loops`

**目的**：识别并融合可以流水线化的嵌套循环，将其合并为单个循环，使流水线 Pass 能处理。

**工作方式**：
1. 分析循环嵌套中有无数据依赖阻止融合
2. 将外层循环变量展开为索引，把内层循环 body 合并到外层
3. 结果：一个包含所有操作的单一循环，适合后续的 `assign_latencies` + `pipeline`

---

#### `add_combine_tensor_select_and_if(pm)` — `tritongpu-combine-tensor-select-and-if`

**目的**：将与同一 `if` 条件相同的 `select` 合并到 `if` 语句内部。

**示例转换**：
```mlir
# 之前：
%result = arith.select %cond, %a, %b : tensor<...>
scf.if %cond { ... } else { ... }

# 之后：
%result = scf.if %cond -> tensor<...> {
  scf.yield %a
} else {
  scf.yield %b
}
```

**为何 SM8x/9x 在流水线之前运行**：流水线 Pass 需要清晰的 if/select 结构才能正确调度条件加载。

---

#### `add_hopper_warpspec(pm, opt.num_stages, dump_enabled)` — `nvgpu-warp-specialization` **[Hopper 专有，位于 `hopper/lib/Transforms/`]**

**目的**：将 kernel 中的操作自动分区为生产者（Loader）warp 和消费者（MMA）warp，实现计算/内存重叠。

**实现机制**（Meta 的 AutoWS，与 Blackwell 的不同）：

```
输入（普通循环）：
  for i in range(N):
    A = load(ptr_A + i*stride)       # 内存操作
    B = load(ptr_B + i*stride)
    acc = dot(A, B, acc)             # 计算操作

↓ hopper_warpspec 分析并分区

输出（warp 专用化，转换为 NVWS Dialect 中间形式）：
  nvws.warp_group(num_warps=4) {     # 生产者 warp 组
    A_buf = load(ptr_A + ...)        # 负责 TMA 加载
    B_buf = load(ptr_B + ...)
    signal(barrier)
  }
  nvws.warp_group(num_warps=4) {     # 消费者 warp 组
    wait(barrier)
    acc = warp_group_dot(A_buf, B_buf, acc)  # 负责 WGMMA 计算
  }
```

**内部分 5 步执行**（来自 `hopper/lib/Transforms/WarpSpecialization/`）：
1. `WSTaskPartition`：将 op 分配到不同异步任务组（通过 `async_task_id` 属性标注）
2. `WSTaskIdPropagate`：将 task_id 属性传播到依赖 op
3. `WSDataPartition`：将操作拆分为更小单元以适配 warp 组
4. `WSCodePartition`：生成实际的 warp 专用代码
5. `NVWSLowerWarpGroup`：将 `nvws.warp_group` 转换为 `ttg.warp_specialize`

**`num_stages` 参数**：控制 producer-consumer 之间的缓冲数量（double/triple buffering）。

**与 SM10x+ `add_warp_specialize` 的区别**：
- `add_hopper_warpspec`（SM8x/9x）：Meta 的 AutoWS 实现，通过 NVWS Dialect 作为中间层，代码在 `hopper/` 目录下
- `add_warp_specialize`（SM10x+）：通用实现 `tritongpu-automatic-warp-specialization`，直接生成 `ttg.warp_specialize`，代码在 `lib/Dialect/TritonGPU/Transforms/`

---

#### `add_assign_latencies(pm, opt.num_stages)` — `tritongpu-assign-latencies`

**目的**：为循环中的"高延迟操作"打上延迟标注（`tt.latency` 属性），供流水线调度使用。

**延迟分配逻辑**（`AssignLatencies.cpp`）：

```python
# 伪代码描述算法
loadLatency = (num_stages - 1) / (maxIndirectionLevel + 1)
# maxIndirectionLevel = 依赖层次（直接 load = 0，load of load = 1, 以此类推）

# 示例：num_stages=3, 直接 load
loadLatency = (3 - 1) / (0 + 1) = 2   # load 比消费者早 2 个阶段
```

**触发条件**：
- `forOp` 不包含更大 loop distance（避免处理多重循环依赖）
- `forOp` 不是外层循环（只对内层循环流水线化）

**标注的 op 类型**：
- `tt.load`（全局内存 load）
- `ttg.async_copy_global_to_local`（异步拷贝）
- `ttng.async_tma_copy_global_to_local`（TMA 异步拷贝，SM9x+）
- `tt.dot` / `ttng.warp_group_dot`（矩阵乘）

---

#### `add_schedule_loops(pm)` — `tritongpu-schedule-loops`

**目的**：根据 `tt.latency` 标注，生成最优的调度方案，确定每个 op 属于流水线的哪个阶段。

**调度结果**：每个 op 获得 `stage` 属性（0, 1, 2, ... num_stages-1），表示在流水线的哪个阶段执行。

**依赖关系**：
```
Stage 0: load A[i], load B[i]      ← 最早开始
Stage 1: (等待 load 完成)
Stage 2: dot(A[i-2], B[i-2], acc) ← 使用两轮前的数据（num_stages=3）
```

---

#### `add_pipeline(pm, opt.num_stages, dump_enabled)` — `tritongpu-pipeline`

**目的**：根据调度方案重写循环，插入 prologue（预加载）和 epilogue（尾部处理），实现真正的软件流水线。

**变换示意**：
```mlir
# 输入（带 stage 标注的循环）：
for i in [0, N):
  %A_tile = load(%A, i)  {stage = 0}
  %B_tile = load(%B, i)  {stage = 0}
  %acc = dot(%A_tile, %B_tile, %acc)  {stage = 2}

# 输出（流水线展开）：
# Prologue（预加载前 num_stages-1 轮）：
%A0 = load(%A, 0)  # 异步启动
%B0 = load(%B, 0)
%A1 = load(%A, 1)
%B1 = load(%B, 1)

# Main loop：
for i in [num_stages-1, N):
  %A_next = load(%A, i)      # 下一轮预加载
  %B_next = load(%B, i)
  %acc = dot(%A[i-2], %B[i-2], %acc)  # 使用缓冲中的数据

# Epilogue（消耗剩余缓冲）：
%acc = dot(%A[N-2], ...)
%acc = dot(%A[N-1], ...)
```

**多缓冲分配**：Pipeline Pass 会自动将 `ttg.local_alloc` 扩展为 `num_stages` 个缓冲（multi-buffering），配合 mbarrier 实现无锁生产消费。

---

### 4.4 SM10x+ 专属路径（Blackwell TMEM）

SM10x+ 相比 SM8x/9x 多了 TMEM 相关的 7 个新 Pass，以及不同的 Warp Spec 机制。

#### `add_optimize_accumulator_init(pm)` — `tritongpu-optimize-accumulator-init` **[新]**

**目的**：消除矩阵乘累加器的显式 zero-initialization，替换为 `useC=false` 标志。

**问题背景**：在 Blackwell MMAv5（`tcgen05.mma`）中，累加器存放在 TMEM 中。如果每次矩阵乘都先用 `store(zeros)` 清零累加器，会产生不必要的 TMEM 写操作。

**优化**：分析循环中首次使用累加器的位置，将 `ttng.warp_group_dot(%a, %b, zeros, useC=true)` 替换为 `ttng.warp_group_dot(%a, %b, prev_acc, useC=false)`。当 `useC=false` 时，硬件 MMA 指令自动将累加器初始化为 0（`tcgen05.mma.cta_group::1 ... useC=false`）。

---

#### `add_hoist_tmem_alloc(pm, False)` — `tritongpu-hoist-tmem-alloc` **[新，第一次调用]**

**目的**：将 TMEM 分配操作（`ttng.tmem_alloc`）从循环内部提升到循环外部，减少 per-iteration 的 TMEM 生命周期冲突。

**参数含义**：
- `hoistOutOfIf=False`：只提升出循环，不提升出 if 语句（第一次保守调用）
- `hoistOutOfIf=True`：也提升出 if 语句（第二次激进调用，在 24c 处）

**原理**：
```mlir
# 之前（TMEM alloc 在循环内）：
scf.for i in [0, N):
  %tmem_buf, %token = ttng.tmem_alloc : !ttg.memdesc<...>
  %result = ttng.tc_gen5_mma(%a, %b, %tmem_buf, %token, ...)

# 之后（TMEM alloc 提升到循环外）：
%tmem_buf, %token_init = ttng.tmem_alloc : !ttg.memdesc<...>
scf.for i in [0, N) iter_args(%token = %token_init):
  %token_new = ttng.tc_gen5_mma(%a, %b, %tmem_buf, %token, ...)
  scf.yield %token_new
```

---

#### `add_promote_lhs_to_tmem(pm)` — `tritongpu-promote-lhs-to-tmem` **[Blackwell 专有]**

**目的**：将 MMAv5 (`tc_gen5_mma`) 的左操作数（LHS/A 矩阵）存储到张量内存（TMEM）中，而不是共享内存。

**背景**：在 Blackwell (SM100+) 中，`tc_gen5_mma` 指令可以从 TMEM 读取 A 矩阵操作数，带来以下优势：
- 共享内存带宽压力减小（A 矩阵不占用 shared mem bank）
- TMEM 有更高的读取带宽，适合 A 矩阵的连续读取模式
- 支持更大的 tile size

**Pass 行为**：
1. 识别 `ttng.warp_group_dot` 中 A 操作数从共享内存读取的情况
2. 插入 `ttng.tmem_alloc` + `ttng.tmem_store`（将 A 写入 TMEM）
3. 将 `warp_group_dot` 的 A 操作数替换为 TMEM 描述符

---

#### `add_warp_specialize(pm, opt.num_stages)` — `tritongpu-automatic-warp-specialization` **[新式，SM10x+]**

**目的**：自动 Warp 专用化，将 kernel 拆分为多个 warp partition，每个 partition 执行不同的任务。

**与 `add_hopper_warpspec` 的本质区别**：

| 特性 | Hopper (`add_hopper_warpspec`) | Blackwell (`add_warp_specialize`) |
|------|-------------------------------|----------------------------------|
| 代码位置 | `hopper/lib/Transforms/WarpSpecialization.cpp` | `lib/Dialect/TritonGPU/Transforms/` |
| 中间表示 | 经过 NVWS Dialect（nvws.warp_group） | 直接生成 ttg.warp_specialize |
| 分区数量 | 通常 2（loader + compute） | 可变，由调度分析决定 |
| TMEM 支持 | 无 | 有（支持 TMEM 在 partition 间传递） |
| 通用性 | NVIDIA Hopper 专用 | 通用实现（AMD 也可用此框架） |

**分区策略（SM100+ 典型）**：
```
Warp Group 0 (default, 4 warps):  主控制流
Warp Group 1 (partition 0, 1 warp): TMA 数据加载（producer）
Warp Group 2 (partition 1, 8 warps): TCGen5 MMA 计算（consumer）
```

---

#### `add_pipeline(pm, opt.num_stages, dump_enabled)` — 流水线（SM10x+ 版）

SM10x+ 的流水线 Pass 与 SM8x/9x 调用同一函数，但内部行为差异显著：

- **MMAv5 异步路径**：`tc_gen5_mma` 本身是异步的，等待通过 TMEM token 机制而不是 mbarrier
- **多级 TMEM 缓冲**：TMEM 多缓冲直接在片上，无需 shared mem multi-buffering
- **NVWS Aref 机制**：通过 `nvws.aref` 抽象管理 producer-consumer 之间的缓冲所有权

```
SoftwarePipeliner.cpp 中对 SM10x+ 的特殊路径：
  ��� pipelineWgmma()：将 WGMMAv3 (SM9x) 中的 asyncLaunchDots 逻辑扩展到 MMAv5
  → hasMMAv5WaitsInLastStage()：检查 TMEM token 是否在最后阶段等待，调整流水线结构
```

---

#### `add_optimize_partition_warps(pm)` — `tritongpu-optimize-partition-warps` **[新]**

**目的**：分析 `ttg.warp_specialize` 中各 partition 的实际资源需求，减少分配给每个 partition 的 warp 数量，降低寄存器使用，提高 occupancy。

**工作方式**：
1. 分析每个 partition 内的 op 类型（load-only partition 不需要太多 warp）
2. 重新计算所需的 warp 数（通常 1-2 warps 足够处理 TMA load）
3. 更新 `ttg.warp_specialize` 的 `partitionNumWarps` 属性
4. 附带寄存器使用优化：根据 partition 特性设置 `requestedRegisters`（默认 232 for compute warp groups）

---

#### `add_hoist_tmem_alloc(pm, True)` — 第二次调用 **[激进版]**

**目的**：第二次调用，参数 `hoistOutOfIf=True`，允许将 TMEM alloc 提升出 if 语句。

**为何第一次不做**：Warp 专用化之前，if 语句可能带条件 TMEM 分配，不安全提升。Warp 专用化之后，if 语句已经按 partition 分割，现在可以安全分析。

**配合 `NVWSHoistTmemStore`**（`nvws-hoist-tmem-store`）：当 TMEM 的清零初始化（`tmem_store(zeros)`）位于嵌套循环内时，进一步提升到顶层，消除内层循环每次迭代都要清零的开销。

---

#### `add_remove_tmem_tokens(pm)` — `triton-nvidia-gpu-remove-tmem-tokens` **[新]**

**目的**：在流水线优化完成后，清理不再需要的 TMEM memory dependency token。

**背景**：TMEM 操作（`tmem_alloc`, `tmem_store`, `tc_gen5_mma`）通过 token 链维护内存依赖顺序。这些 token 是 IR 层的虚拟依赖，在流水线和 Warp 专用化完成后，它们要么被 mbarrier/NVWS aref 取代，要么已经冗余，需要清除。

**不清除的危险**：LLVM 会把遗留的 token 当作实际的 `i1` 值处理，产生多余的条件分支。

---

### 4.5 共享后段 Pass（所有 SM）

#### `add_prefetch(pm)` — `tritongpu-prefetch`

**目的**：在循环体中，对 `tt.dot` 的操作数进行 shared memory 级别的预取。

**工作方式**：
1. 识别循环中的 `DotOp`，其操作数来自 `ttg.local_load`（共享内存）
2. 将 `DotOp` 分解为多个更小的 finer-grained `DotOp`
3. 在循环的上一次迭代末尾，预取下一次迭代的操作数
4. 在循环的 prologue 中预取第一次迭代数据

**与软件流水线的关系**：`add_prefetch` 是共享内存级别的预取，`add_pipeline` 是全局内存级别的预取。两者协同工作：全局 → (pipeline) → 共享内存 → (prefetch) → 寄存器 → TensorCore。

---

#### `add_optimize_tmem_layouts(pm)` — `triton-nvidia-optimize-tmem-layouts` **[实质仅 SM10x+]**

**目的**：为 TMEM 分配选择最优的 layout，使能更好的 subtiling 和 reduction 性能。

**在 SM7x/SM8x/SM9x 上**：无 TMEM op 存在，此 Pass 为空操作。

**在 SM10x+ 上**：
- 分析 `ttng.tmem_alloc` 的使用方式（用于 MMA 累加器 vs 用于 epilogue reduction）
- 选择对 `tc_gen5_mma` 最优的 TMEM layout（通常是 column-major）
- 调整 layout 以支持更高效的 TMEM 到寄存器的数据转换

---

#### `add_tma_lowering(pm)` — `triton-nvidia-tma-lowering` **[仅 SM9x+]**

**条件**：`if capability // 10 >= 9`

**目的**：将高层的 `tt.load(tensordesc, coord)` 降级为 NVIDIA 专有的 TMA（Tensor Memory Accelerator）操作。

**变换示例**：
```mlir
# 输入（高层描述符 load）：
%result = tt.load %descriptor, %coord : !tt.tensordesc<tensor<64x64xf16>>

# 输出（TMA 异步拷贝）：
ttng.init_barrier %barrier, 1 : ...
ttng.barrier_expect %barrier, 8192, %pred : ...  # 8192 = 64*64*2 bytes
ttng.async_tma_copy_global_to_local %descriptor[%coord], %shared_buf, %barrier, %pred : ...
ttng.wait_barrier %barrier, %phase : ...
```

**在 SM8x 上**：`tensordesc` 已在 `make_ttir` 中被 `add_rewrite_tensor_descriptor_to_pointer` 替换为普通指针，此时 `tensordesc` op 不存在，此 Pass 不运行（通过条件 `capability // 10 >= 9` 跳过）。

---

#### `add_interleave_tmem(pm)` — `triton-nvidia-interleave-tmem` **[实质仅 SM10x+]**

**目的**：通过下沉 TMEM load 和提升 TMEM store，以及交织两者，降低寄存器压力。

**具体行为**：
- 将 `ttng.tmem_load`（TMEM → 寄存器）尽量向下移动（延迟物化）
- 将 `ttng.tmem_store`（寄存器 → TMEM）尽量向上移动（提前归还寄存器）
- 若顺序允许，交织 load 和 store 以填充延迟空隙

**与 `add_reorder_instructions` 的协同**：`reorder_instructions` 在整体 IR 层面重排，`interleave_tmem` 专门处理 TMEM 相关的细粒度调度。

---

#### `add_fence_insertion(pm, capability)` — `triton-nvidia-gpu-fence-insertion` **[参数化]**

**目的**：在异步操作的正确位置插入内存栅栏（`ttng.fence_async_shared`）。

**基于 `capability` 的差异化行为**：

| SM 版本 | capability 传入值 | 栅栏类型 | 插入场景 |
|---------|-----------------|---------|---------|
| SM7x | 70/75 | 无（无异步操作） | 极少插入 |
| SM8x | 80-89 | `fence.proxy.async` | 异步拷贝前后 |
| SM9x | 90 | `fence.proxy.async.global` + mbarrier | TMA 前后 |
| SM10x | 100+ | TMEM fence + TMA fence | TMEM 和 TMA 边界 |

**Pass 的两个版本**：
- `add_fence_insertion`（在 `make_ttgir` 末尾）：**优化版**，分析最优插入位置
- `add_proxy_fence_insertion`（在 `make_llir` 中）：**保守版**，确保所有功能性需求满足

---

#### `add_lower_mma(pm)` — `triton-nvidia-mma-lowering`

**目的**：将 TTGIR 中的高层 MMA 表示（带 `MmaEncoding` 的 `tt.dot`）降级为 NVGPU Dialect 中的底层指令。

**按 MMA 版本的降级目标**：

| 输入 | 输出 | 指令集 | SM |
|-----|-----|-------|-----|
| `tt.dot` with `MmaV1Encoding` | `nvgpu.wmma.*` | WMMA | SM7x |
| `tt.dot` with `MmaV2Encoding` | `nvgpu.mma.sync.*` inline ASM | mma.sync | SM8x |
| `ttng.warp_group_dot` | `nvgpu.wgmma` | WGMMA | SM9x |
| `ttng.tc_gen5_mma` | 保留（稍后由 `TensorMemoryToLLVM.cpp` 处理） | TCGen5 | SM10x+ |

---

## 5. make_llir 阶段差异

`make_llir` 相对于 `make_ttgir` 分支更少，但仍有关键差异：

```python
def make_llir(self, src, metadata, options, capability):
    ptx_version = get_ptx_version_from_options(options, self.target.arch)
    mod = src
    pm = ir.pass_manager(mod.context)
    pm.enable_debug()

    # 所有 SM：
    passes.ttgpuir.add_combine_tensor_select_and_if(pm)  # 再次运行（warp spec 后可能有新机会）
    passes.ttgpuir.add_allocate_warp_groups(pm)           # 为 warp spec 分配 warp group ID
    passes.convert.add_scf_to_cf(pm)                      # SCF → CF
    passes.gluon.add_inliner(pm)                          # Gluon 专用内联（处理 warp spec 区域）

    # NVIDIA 专有：共享内存分配（capability 和 ptxVersion 决定 swizzling 选项）
    nvidia.passes.ttgpuir.add_allocate_shared_memory_nv(pm, capability, ptx_version)

    # NVIDIA 专有：TMEM 分配（SM10x+ 有效，其他为空）
    nvidia.passes.ttnvgpuir.add_allocate_tensor_memory(pm)

    # NVIDIA 专有：验证 Two-CTA matmul 一致性（SM10x+ 特性）
    nvidia.passes.ttnvgpuir.add_check_matmul_two_cta(pm)

    # 所有 SM：全局临时内存分配
    passes.ttgpuir.add_allocate_global_scratch_memory(pm)

    # NVIDIA 专有：代理栅栏插入（保守版本，功能性保证）
    nvidia.passes.ttnvgpuir.add_proxy_fence_insertion(pm, capability)

    # NVIDIA 专有：核心 Lowering（TritonGPU → LLVM IR）
    nvidia.passes.ttgpuir.add_to_llvmir(pm, capability, ptx_version)
    # = ConvertTritonGPUToLLVMPass，触发所有 *ToLLVM.cpp 中的 patterns

    passes.ttgpuir.add_canonicalize_llvm_ir(pm)            # LLVM IR 规范化
    passes.common.add_cse(pm)

    # NVIDIA 专有：Warp 专用化 → LLVM 降级
    nvidia.passes.ttnvgpuir.add_warp_specialize_to_llvm(pm)
    # = ConvertWarpSpecializeToLLVM：生成共享内存通信 + barrier 代码

    # NVIDIA 专有：NVGPU Dialect → LLVM IR（处理 nvgpu.wgmma 等）
    nvidia.passes.ttnvgpuir.add_nvgpu_to_llvm(pm)

    # 所有 SM：标准化清理
    passes.common.add_canonicalizer(pm)
    passes.common.add_cse(pm)
    passes.common.add_symbol_dce(pm)
    passes.convert.add_nvvm_to_llvm(pm)                    # NVVM Dialect → LLVM intrinsics
    passes.llvmir.add_di_scope(pm)                         # Debug info

    pm.run(mod, 'make_llir')

    # LLVM-C API 阶段
    llvm.init_targets()
    context = llvm.context()
    llvm_mod = llvm.to_module(mod, context)
    proc = sm_arch_from_capability(capability)    # 例如 "sm_90a" for SM90
    features = get_features(options, self.target.arch)   # 例如 "+ptx86"
    triple = 'nvptx64-nvidia-cuda'
    nvidia.set_short_ptr()                         # NVPTX 使用 32bit 指针优化
    llvm.attach_datalayout(llvm_mod, triple, proc, features)
    nvidia.set_nvvm_reflect_ftz(llvm_mod)          # FTZ 控制（刷新 denormal 到 0）

    # 链接设备库（仅当存在外部依赖时）
    if options.extern_libs and nvidia.has_extern_deps(llvm_mod):
        paths = [path for (name, path) in options.extern_libs]
        llvm.link_extern_libs(llvm_mod, paths)     # 链接 libdevice.10.bc

    llvm.optimize_module(llvm_mod, llvm.OPTIMIZE_O3)  # LLVM O3 优化

    # 收集元数据
    metadata["num_warps"]           = src.get_int_attr("ttg.total-num-warps")
    metadata["shared"]              = src.get_int_attr("ttg.shared")
    metadata["tmem_size"]           = src.get_int_attr("ttg.tensor_memory_size")   # SM10x+
    metadata["global_scratch_size"] = src.get_int_attr("ttg.global_scratch_memory_size")
```

**`sm_arch_from_capability` 的逻辑**：
```python
def sm_arch_from_capability(capability: int):
    suffix = "a" if capability >= 90 else ""  # SM90 用 sm_90a，SM80 用 sm_80
    return f"sm_{capability}{suffix}"
```

---

## 6. parse_options 中的类型与能力差异

```python
def parse_options(self, opts) -> CUDAOptions:
    args = {'arch': f"sm{self.target.arch}"}
    capability = self.target.arch  # int

    # num_ctas 限制
    if args.get("num_ctas", 1) > 1 and capability < 90:
        raise ValueError("num_ctas > 1 requires SM90+ (Hopper)")

    # FP8 类型支持
    supported_fp8 = set(CUDAOptions.supported_fp8_dtypes)   # 默认：fp8e5, fp8e4b15
    if capability >= 89:
        supported_fp8.add("fp8e4nv")     # Ada/Hopper/Blackwell: OCP FP8 e4m3
    args["supported_fp8_dtypes"] = tuple(sorted(supported_fp8))

    # FP8 废弃类型（SM90+ 废弃旧格式）
    if capability >= 90:
        args["deprecated_fp8_dot_operand_dtypes"] = ("fp8e4b15",)

    # imprecise accumulator（SM90 Hopper 矩阵乘的近似优化）
    args["max_num_imprecise_acc_default"] = 2**30 if capability == 90 else 0
```

**`max_num_imprecise_acc_default = 2**30` 的含义**（仅 SM90）：

SM90 的 WGMMA 指令在 fp8 矩阵乘中，累加器使用有限精度（imprecise）的方式累加多个 k 块的结果，最多允许 `2^30` 次不精确累加，换取更高吞吐。SM100+ 的 TCGen5 有更完善的精度控制，不需要此设置。

**类型支持汇总**：

| 类型 | SM70-75 | SM80-86 | SM87-89 | SM90 | SM100+ |
|-----|---------|---------|---------|------|--------|
| f16 × f16 → f32 | ✓ wmma | ✓ mma.sync | ✓ | ✓ wgmma | ✓ tc_gen5 |
| bf16 × bf16 → f32 | ✗ | ✓ | ✓ | ✓ | ✓ |
| tf32 × tf32 → f32 | ✗ | ✓(emu) | ✓(emu) | ✓(emu) | ✓(emu) |
| fp8e5m2 × fp8e5m2 → f32 | ✗ | ✗ | ✗ | ✓ | ✓ |
| fp8e4m3 × fp8e4m3 → f32 | ✗ | ✗ | ✓(SM89) | ✓ | ✓ |
| fp8e4b15（旧） | ✗ | ✗ | ✓ | ✓(deprecated) | ✗ |
| fp4 × fp4 → f32 | ✗ | ✗ | ✗ | ✗ | ✓ |
| i8 × i8 → i32 | ✓ | ✓ | ✓ | ✓ | ✓ |
| num_ctas > 1 | ✗ | ✗ | ✗ | ✓ | ✓ |
| TMA 异步加载 | ✗ | ✗ | ✗ | ✓ | ✓ |
| TMEM | ✗ | ✗ | ✗ | ✗ | ✓ |

---

## 7. Warp 专用化机制对比（SM8x-9x vs SM10x+）

### SM8x-9x：Hopper AutoWS（Meta 实现）

```
代码入口：hopper/lib/Transforms/WarpSpecialization.cpp
Python 调用：nvidia.passes.hopper.add_hopper_warpspec(pm, num_stages, dump)
C++ Pass：mlir::createNVGPUWarpSpecialization
```

**实现流程（6 步）**：
1. **Task Partition**（`WSTaskPartition`）：根据 op 类型（load=producer, dot=consumer），将 op 分配到不同 task group，打上 `async_task_id` 属性
2. **Task Id Propagate**（`WSTaskIdPropagate`）：将 task_id 沿数据依赖链传播
3. **Data Partition**（`WSDataPartition`）：将跨 task 的 op（如同时被 producer/consumer 使用的 tensor）拆分
4. **Code Partition**（`WSCodePartition`）：为每个 task group 生成独立的代码区域，通过共享内存 buffer 传递数据
5. **NVWS Lower Warp Group**（`NVWSLowerWarpGroup`）：将 `nvws.warp_group` 转换为 `ttg.warp_specialize`
6. **NVWS Lower Aref**（`NVWSLowerAref`）：将 `nvws.aref.*` 转换为 `ttng.*barrier*` 同步原语

**典型 warp 分配（flash attention, SM90）**：
```
Total: 8 warps per CTA
Partition 0 (TMA Loader): 1 warp  → 负责 TMA 加载 Q/K/V
Partition 1 (WGMMA): 7 warps → 负责矩阵乘和 softmax
```

### SM10x+：通用 Warp Spec（新式）

```
代码入口：lib/Dialect/TritonGPU/Transforms/WarpSpecialize.cpp
Python 调用：passes.ttgpuir.add_warp_specialize(pm, num_stages)
C++ Pass：mlir::triton::gpu::createTritonGPUAutomaticWarpSpecialization
```

**与 Hopper 版本的关键差别**：

| 方面 | Hopper AutoWS | Blackwell WarpSpec |
|-----|--------------|-------------------|
| 中间 Dialect | 经过 NVWS | 直接生成 ttg.warp_specialize |
| TMEM 处理 | 不处理 TMEM 所有权 | 通过 NVWS 管理 TMEM partition 所有权 |
| 后处理 Pass | 无 | `add_optimize_partition_warps` 优化 warp 数 |
| combine_select | 在 warpspec 之前 | 在 warpspec 之后（24c 处） |
| token 清理 | 无 | `add_remove_tmem_tokens` |
| `hoist_tmem_alloc` | 无 | 两次（含 `hoistOutOfIf=True`） |

---

## 8. TMEM（张量内存）完整流程（仅 SM10x+）

### 8.1 TMEM 的架构背景

Blackwell SM100 引入了片上张量内存（Tensor Memory，TMEM），用于存放矩阵乘的累加器，特点：
- 位于 SM 内部，访问速度远高于 L1 cache
- 容量固定（per-SM），通过 `ttng.tmem_alloc` 分配
- 通过 token 链维护写后读（RAW）依赖顺序
- 支持 TCGen5 指令直接读写

### 8.2 TMEM 相关 Pass 的执行顺序与依赖

```
make_ttgir 中的 TMEM Pass 执行顺序：

[第 15c] add_optimize_accumulator_init
         → 目标：消除 tmem_alloc 之前的 zero-init store
         → 前提：dot op 的 useC flag 可以替换 zero-init

[第 16c] add_hoist_tmem_alloc(hoistOutOfIf=False)
         → 目标：将 tmem_alloc 提升出 for 循环
         → 约束：不能提升出 if（需要静态分配确定性）

[第 17c] add_promote_lhs_to_tmem
         → 目标：将 MMAv5 的 A 操作数从 shared mem 移入 TMEM
         → 结果：插入 tmem_alloc + tmem_store(A_data)

[第 18c-21c] 流水线阶段
         → Pipeline Pass 识别 TMEM op，将其纳入 async 调度

[第 22c] add_optimize_partition_warps
         → 根据 TMEM 使用情况调整各 partition 的 warp 数

[第 23c] add_combine_tensor_select_and_if
         → 在 Warp Spec 后合并 select/if（Warp Spec 可能引入新的 if）

[第 24c] add_hoist_tmem_alloc(hoistOutOfIf=True)
         → 第二次提升：现在 if 已经被 warp spec 分区，可以安全提升出 if

[第 25c] add_remove_tmem_tokens
         → 流水线完成后，TMEM token 不再需要，清除

── make_llir 中的 TMEM Pass ──────────────────────────

[make_llir] add_allocate_tensor_memory
         → 实际分配 TMEM 地址空间，为每个 tmem_alloc 分配偏移
         → 写入 module 属性：ttg.tensor_memory_size

[make_llir] TensorMemoryToLLVM.cpp（通过 add_to_llvmir 触发）
         → 将 ttng.tmem_alloc → LLVM ptr<6>（address space 6）
         → 将 ttng.tmem_store → tcgen05.st.* intrinsic
         → 将 ttng.tc_gen5_mma → tcgen05.mma.* intrinsic
```

### 8.3 TMEM 的 IR 变化示例

```mlir
# make_ttgir 之后（TMEM 在 IR 中的样子）：

%tmem_buf, %token0 = ttng.tmem_alloc
    : !ttg.memdesc<128x128xf32, #tmem, #tensor_mem>

# A 矩阵提升到 TMEM（add_promote_lhs_to_tmem 产生）：
%tmem_a, %tok_a = ttng.tmem_alloc : !ttg.memdesc<64x64xf16, #tmem, #tensor_mem>
%tok_a2 = ttng.tmem_store %A_shared, %tmem_a[%tok_a], %pred
    : !ttg.memdesc<...> into !ttg.memdesc<...>

# TCGen5 MMA（SM100 矩阵乘）：
%tok1 = ttng.tc_gen5_mma %tmem_a, %B_shared, %tmem_buf[%token0], %useC, %pred
    : !ttg.memdesc<...>, !ttg.memdesc<...>, !ttg.memdesc<...>

# make_llir → TensorMemoryToLLVM 之后（LLVM IR 层）：
%tmem_ptr = llvm.inttoptr %base_addr : i64 -> !llvm.ptr<6>  # address space 6 = TMEM
call @tcgen05.mma.cta_group::1.kind::f16(...)               # PTX tcgen05 指令
```

---

## 9. 软件流水线机制对比

| 特性 | SM7x | SM8x（Ampere） | SM9x（Hopper） | SM10x+（Blackwell） |
|-----|------|--------------|--------------|-------------------|
| 流水线存在 | 否 | 是 | 是 | 是 |
| 内存操作类型 | 同步 `tt.load` | `async_copy_global_to_local` | TMA async | TMA async |
| 等待机制 | 无 | `ttg.async_wait` | mbarrier | mbarrier |
| 双缓冲 | 否 | shared mem multi-buffer | shared mem multi-buffer | TMEM multi-buffer |
| Warp 专用化 | 否 | 否（Ampere）/ 是（Hopper实验） | 是（autoWS） | 是（新式） |
| num_stages 默认 | 3 | 3 | 3 | 3 |
| 流水线代码文件 | — | `SoftwarePipeliner.cpp` | `SoftwarePipeliner.cpp` + `WGMMAPipeline.cpp` | `SoftwarePipeliner.cpp` + `MMAv5PipelineUtility.cpp` |

---

## 10. 各 SM 特性能力矩阵

| 能力 / 特性 | SM7x | SM8x | SM9x | SM10x+ |
|------------|------|------|------|--------|
| TF32 模拟 | ✗ | ✓ | ✓ | ✓ |
| HW 转置操作数 | ✗ | ✓ | ✓ | ✓ |
| 异步拷贝 `cp.async` | ✗ | ✓ SM80 | ✓ | ✓ |
| Tensor Descriptor 原生支持 | ✗ | ✗ | ✓(TMA) | ✓(TMA) |
| 软件流水线 | ✗ | ✓ | ✓ | ✓ |
| Warp 专用化（Hopper AutoWS） | ✗ | ✗ | ✓ | ✗（换用新版） |
| Warp 专用化（新式） | ✗ | ✗ | ✗ | ✓ |
| TMEM 张量内存 | ✗ | ✗ | ✗ | ✓ |
| MMA 版本 | v1 | v2 | v3(WGMMA) | v5(TCGen5) |
| FP8 原生 | ✗ | ✗ | ✓(SM89+) | ✓ |
| FP4 | ✗ | ✗ | ✗ | ✓ |
| Cluster（multi-CTA） | ✗ | ✗ | ✓ | ✓ |
| Two-CTA matmul | ✗ | ✗ | ✗ | ✓ |
| `make_ttgir` Pass 总数 | 13 | 23 | 23+TMA | 30 |
