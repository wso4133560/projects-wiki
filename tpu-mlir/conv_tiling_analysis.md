# TPU-MLIR Conv Tiling 深度分析

## 概述

TPU-MLIR 中 Conv（卷积）操作的 tiling 逻辑是整个 LayerGroup 优化体系的核心场景。其本质问题是：卷积层的输入激活、权重、输出激活三者之和往往超过 TPU 片上本地内存（LMEM）的容量，必须将张量沿 N/H/W/C 等维度切片，分批搬入 LMEM 完成计算，再搬回全局内存（GMEM）。

整个流程分为五个层次：**分割约束收集 → 初始 shape_secs 设定 → 时间步反向传播 → LMEM 地址分配与 shape_secs 搜索 → 后端代码生成**。本文逐层展开。

---

## 一、核心数据结构

### 1.1 `shape_secs_t`

```
lib/Dialect/Tpu/Transforms/LayerGroup/LayerGroupDefs.h
```

`shape_secs_t` 描述一个 group 内所有层共用的分割方案：

```cpp
typedef struct shape_secs {
  int64_t nsecs;        // N 维（batch）分割段数
  int64_t csecs;        // C 维（输出通道）分割段数
  int64_t dsecs;        // D 维（仅 3D conv 使用）
  int64_t hsecs;        // H 维（特征图高度）分割段数
  int64_t wsecs;        // W 维（特征图宽度）分割段数
  int64_t n_slice_num;
  int64_t c_slice_num;
  int64_t h_slice_num;
} shape_secs_t;
```

`nsecs=2, hsecs=3` 意味着输入在 N 方向切 2 份、H 方向切 3 份，共 6 个时间步。

### 1.2 `slice_info_t`

```cpp
struct slice_info_t {
  std::vector<slice_pair_t> h;   // (h_idx, h_slice) 列表
  std::vector<slice_pair_t> n;
  std::vector<slice_pair_t> w;
  std::vector<slice_pair_t> d;
  std::vector<slice_pair_t> c;
};
```

每个 `slice_pair_t = (起始偏移, 切片尺寸)`。列表长度即对应维度的段数。

### 1.3 `local_sec_info_t`

传递给后端硬件 codegen 的最终切片描述：

```cpp
struct local_sec_info_t {
  int32_t n_slice, d_slice, h_slice, w_slice;  // 输入切片大小
  int32_t n_idx, h_idx, w_idx, c_idx;          // 输入起始索引
  bool    is_h_split, is_w_split, is_c_split;  // 是否沿该维分割
  int32_t out_n_slice, out_h_idx, out_h_slice; // 输出映射
  int32_t out_w_idx, out_w_slice;
  group_type_t group_type;
};
```

### 1.4 `tensor_info_t`

```cpp
struct tensor_info_t {
  TIMESTEP_LD_ST mode;            // LOAD / STORE
  slice_info_t   slice_info;      // 该张量的切片信息
  int64_t        stage;           // 软件流水线阶段（0/1/2）
  int64_t        use_3ic_opt;     // 3IC 深度可分离优化标志
  bool           eu_align;        // 是否需要 EU 对齐
  bool           hold_in_lmem;    // 是否跨时间步保持在 LMEM
};
```

---

## 二、分割约束收集：`get_group_max_secs()`

```
lib/Dialect/Tpu/Transforms/LayerGroup/LayerGroupUtil.cpp:611-722
```

在确定 group 之后，编译器首先需要知道**各维度最多能切几段**，这由每个 op 的硬件约束综合决定。

### 2.1 算法流程

对 group 内每个 op 的每个输出张量，依次计算各维度上限：

```cpp
// N 维：受 batch 对齐约束
if (AllowDataSplit(dim_N, group_type)) {
    n_align = 32 / dtype_bytes;        // 例如 INT8 时 n_align=32
    max_nsecs = min(max_nsecs, ceiling_func(n, n_align));
} else {
    max_nsecs = 1;                     // 该 op 不允许 N 维分割
}

// H 维：受特征图高度约束
if (AllowDataSplit(dim_H, group_type)) {
    max_hsecs = min(max_hsecs, h);
}

// W 维：受 DMA 对齐约束（BM1684X/BM1690）
if (BM1684X && AllowDataSplit(dim_W, group_type)) {
    if (w * dtype_bytes <= DMA_ALGN_BYTES) {
        max_wsecs = 1;                 // 行宽度不足一个 DMA 对齐单位，禁止 W 分割
    } else {
        max_wsecs = min(max_wsecs, w);
    }
}

// C 维：仅 GROUP_MM / GROUP_SMALL_C 类型
if ((GROUP_MM || GROUP_SMALL_C) && AllowDataSplit(dim_C, group_type)) {
    max_csecs = min(max_csecs, ceiling_func(c, Arch::NPU_NUM));
}
```

### 2.2 Conv 的维度分割限制

对于标准 `Conv2D`（`GROUP_NORMAL`）：
- **N 维**：允许，但受 batch 对齐约束
- **H 维**：允许，是主要分割维度
- **W 维**：BM1684X 及以上支持，但受 DMA 对齐约束
- **C 维**：标准 Conv 不在 C 维分割输入，仅在 GROUP_MM 类型下开放
- **D 维**：仅 3D Conv 且 BM1684X/BM1690 上支持

---

## 三、初始 shape_secs 设定：`init_group_data_secs()`

```
lib/Dialect/Tpu/Transforms/LayerGroup/LayerGroupUtil.cpp:749-826
```

该函数从 LMEM 容量约束出发，**自底向上**推算出使整个 group 恰好能放入 LMEM 的最小分割段数。

### 3.1 总内存需求估算

对 group 内每个 op：

```cpp
in0_size   = Arch::get_tensor_lmem_bytes(input[0], n/nsecs, c, h, w);
out0_size  = Arch::get_tensor_lmem_bytes(output[0], ...);
buf_size   = op->getBufferSize(in0_size, out0_size, ...);  // Conv 额外缓冲
weight_sz  = Arch::get_tensor_lmem_bytes(weight, ...);     // 权重整体加载

total_size = in0_size + out0_size + buf_size + weight_sz;
```

### 3.2 反推最小分割数

```cpp
// 需要多少段才能放入 LMEM
total_secs = ceiling_func(total_size, Arch::LMEM_BYTES);

// 依次消化到各维度（N → C → D → H → W）
shape_secs.nsecs = max(min(total_secs, max_nsecs), shape_secs.nsecs);
total_secs = ceiling_func(total_secs, shape_secs.nsecs);

// H 维是主战场
shape_secs.hsecs = max(total_secs, shape_secs.hsecs);

// H 超上限则溢出到 W
if (shape_secs.hsecs > max_hsecs) {
    shape_secs.wsecs = ceiling_func(shape_secs.hsecs, max_hsecs);
    if (shape_secs.wsecs > max_wsecs) return false;  // LMEM 彻底不足
    shape_secs.hsecs = max_hsecs;
}
```

这一步得到的 `shape_secs` 是**下界估计**，是后续搜索的起点。

---

## 四、Conv 的 H/W 向后推导：`BackwardH()` / `BackwardW()`

```
lib/Dialect/Tpu/Impl/Conv2D.cpp:346-376
```

给定输出切片 `(out_idx, out_slice)`，计算对应的输入切片 `(in_idx, in_slice)`，需要考虑 stride、dilation、padding：

### 4.1 H 方向反推

```cpp
int kh_ext = (kh - 1) * dh + 1;    // 空洞卷积等效核高

in_slice = (out_slice - 1) * sh + max(kh_ext, sh);
in_idx   = out_idx * sh - pht;      // 减去 top padding

LocalGenInterface::fixSlice(in_idx, in_slice, ih, is_last);
```

`fixSlice()` 处理边界情况：当 `in_idx < 0`（第一个切片碰到 top padding）或 `in_idx + in_slice > ih`（最后切片超出底部）时，做裁剪并记录实际有效区域。

### 4.2 W 方向反推（类似）

```cpp
int kw_ext = (kw - 1) * dw + 1;

in_slice = (out_slice - 1) * sw + max(kw_ext, sw);
in_idx   = out_idx * sw - pwl;
```

### 4.3 输入切片放大效应

对于 `stride=1, kh=3, kh_ext=3` 的典型 3×3 Conv，若输出切 `out_slice=H/4`：

```
in_slice = (H/4 - 1) * 1 + 3 = H/4 + 2
```

即每段输入比输出多 `kh_ext - sh = 2` 行（halo），相邻段之间有重叠数据，LMEM 需多分配这部分 halo。

---

## 五、`assign_sec_info_kernel()`：填充硬件切片描述

```
lib/Dialect/Tpu/Impl/Conv2D.cpp:378-420
```

确定切片方案后，将其翻译为后端 codegen 所需的 `local_sec_info_t`：

```cpp
sec_info.n_slice = in_gi.n_slice;
sec_info.h_slice = in_gi.h_slice;
sec_info.w_slice = in_gi.w_slice;

sec_info.is_h_split = !(in_gi.h_idx == 0 && in_gi.h_slice == attr.ih);
sec_info.is_w_split = !(in_gi.w_idx == 0 && in_gi.w_slice == attr.iw);

// BM1684X：将 padding 信息编码进 h_idx（负值表示 top padding）
if (!module::isCV18xx()) {
    int64_t pad_h_b = (in_gi.h_idx + in_gi.h_slice == attr.ih) ? attr.phb : 0;
    if (sec_info.is_h_split) {
        sec_info.h_idx = (in_gi.h_idx == 0) ? -attr.pht : in_gi.h_idx;
        sec_info.h_slice += (sec_info.h_idx < 0) ? (-sec_info.h_idx) : 0;
        sec_info.h_slice += pad_h_b;   // 最后一段加 bottom padding
    }
}

sec_info.out_h_idx   = gi.h_idx;
sec_info.out_h_slice = gi.h_slice;
```

**负 `h_idx` 编码**是一个关键设计：`h_idx = -pht` 告知硬件"该切片需要 `pht` 行 top padding 填 0，实际数据从 GMEM 偏移 0 开始读"，避免了在 LMEM 中显式存储 padding。

---

## 六、Conv 的 LMEM 缓冲区估算：`getBufferSize_bm1684x()`

```
lib/Dialect/Tpu/Impl/Conv2D.cpp（BM1684X 实现段）
```

Conv 除了 input/weight/output 三个主缓冲区外，还需要若干辅助缓冲区：

### 6.1 4-bit 权重解压缓冲

当权重精度为 INT4 时，执行前需解压到 INT8：

```cpp
if (getWeightBits() == 4) {
    int kh_ext = use_3ic_optimize & 0x1 ? 1 : kh;
    int kw_ext = use_3ic_optimize & 0x2 ? 1 : kw;
    sz += oc_per_npu * align_up(ic_ext / groups, ...) * kh_ext * kw_ext;
}
```

### 6.2 Group Conv 中间缓冲

分组卷积（`groups > 1`）需要额外的中间激活缓冲：

```cpp
if (groups > 1) {
    sz += n_slice * ic_per_npu * align_up(h_slice * w_slice, eu_num) * type_len;
    sz += ic_per_npu * 2 * type_len;  // per-NPU 额外工作空间
}
```

### 6.3 3IC 深度可分离优化缓冲

当启用 `use_3ic_optimize`（把三个输入通道合并为一次操作）时：

```cpp
if (use_3ic_optimize) {
    // 合并 kh/kw 维度的输出中间缓冲
    dw_out_size = merge_both  ? out_h_slice * out_w_slice
                : merge_kh   ? out_h_slice * in_w_slice
                : merge_kw   ? in_h_slice  * out_w_slice
                :              0;

    sz += align_up(dw_out_size, eu_num)
        * ceiling_func(in_c_slice * kernel_len, Arch::NPU_NUM)
        * n_slice * in_type_len;
}
```

---

## 七、shape_secs 搜索：`assignLmemAddrWithSecs()`

```
lib/Dialect/Tpu/Transforms/LayerGroup/LmemAllocator.cpp:1349-1524
```

初始 `shape_secs` 只是下界，真正的最优方案需要在合法空间内搜索**使周期最短**的分割组合。

### 7.1 搜索策略选择

```cpp
if (is_best_shape_secs) {
    sc_method_use_best_shape_secs(...);     // 直接使用缓存
} else if (shape_secs_search_level == 1) {
    if (GROUP_MM)
        sc_method_search_better_v2(...);    // MatMul 专用
    else
        sc_method_search_better_v1(...);    // Conv 通用
} else {
    sc_method_quick_search(...);            // 快速搜索（默认）
}
```

### 7.2 `sc_method_search_better_v1()` 的枚举搜索

```
LmemAllocator.cpp:1646-1785
```

五维网格搜索（D × W × C × N × H），每维设置连续失败阈值（默认 3 次）来剪枝：

```cpp
for _d in [1..max_dsecs]:
  for _w in [1..max_wsecs]:
    for _c in [1..max_csecs]:
      for _n in [1..max_nsecs]:
        for _h in [1..max_hsecs]:
            cur = {_n, _c, _d, _h, _w}

            // 跳过总段数低于下界的无意义组合
            if (real_nsecs * real_csecs * ... <= total_secs_lower_bound_)
                continue;

            ret = try_this_shape_secs(cur, ...);

            // 连续 3 次无改进则停止该维度增长
            if (dim_fail_count[dim_H] >= 3) break;
```

### 7.3 `try_this_shape_secs()`：每次尝试的完整流程

```cpp
// 1. 反向传播 slice_info
status = time_step->assignTimeStep(lg_info, shape_secs, true);
if (!status) return SECS_TIMESTEP_INVALID;

// 2. 分配 LMEM 地址
status = assignLmemAddr(lg_info, time_step, shape_secs, ...);
if (!status) return SECS_LMEM_INVALID;

// 3. 估算周期成本
_group_cost = cycle_calculator_->getGroupCycle(time_step, shape_secs, type);

// 4. 更新最优
if (_group_cost < min_group_costs_) {
    min_group_costs_ = _group_cost;
    min_shape_secs_  = shape_secs;
    return SECS_VALID_AND_BETTER;
}
return SECS_VALID;
```

每次"尝试"都包含完整的切片信息传播、LMEM 地址分配和周期估算三步，保证了最终选出的 `shape_secs` 在 LMEM 合法且周期最优。

---

## 八、时间步分配与双缓冲：`BasicTimeStep`

```
lib/Dialect/Tpu/Transforms/LayerGroup/BasicTimeStep.cpp
```

确定 `shape_secs` 之后，`assignTimeStep()` 将各张量的切片信息分配到时间步，并通过 `SwPipeline` 产生三阶段软件流水线：

```
Stage 0: GDMA Load（从 GMEM 预取下一切片的 input）
Stage 1: BDC Compute（当前切片的 Conv 计算）
Stage 2: GDMA Store（将上一切片的 output 写回 GMEM）
```

### 8.1 双缓冲内存布局

```
LMEM 布局示意（以 H 方向 2 段双缓冲为例）：

[buf_A: input_slice_0]  ← Stage-0 加载 slice_0
[buf_B: input_slice_1]  ← Stage-0 加载 slice_1（预取）
[weight]                ← 整个权重（不随切片变化）
[out_buf_A: output_0]   ← Stage-1 写 slice_0 的结果
[out_buf_B: output_1]   ← Stage-1 写 slice_1 的结果
```

权重**始终完整**驻留 LMEM（不随 H 分割），仅激活按切片轮换，实现计算与 DMA 的流水并行。

### 8.2 `hold_in_lmem` 标志

当某层的输出是下一时间步的 group 内输入（而非写回 GMEM）时，`hold_in_lmem=true`，跳过 Store 步骤，直接复用 LMEM 地址，减少不必要的 GDMA 传输。

---

## 九、动态规划分组：`GroupMethod`

```
lib/Dialect/Tpu/Transforms/LayerGroup/GroupMethod.cpp:1380-1640
```

Conv tiling 是 LayerGroup 体系的子问题；外层还需要决定**哪些连续的算子应该合并为一个 group** 做 tiling。这由动态规划求解。

### 9.1 DP 状态定义

```
cost_table[i][j] = 将 op[i..j] 合并为一个 group 的最小周期成本
cut_points[i][j] = 达到最优成本时的分割点（用于回溯）
```

### 9.2 DP 转移

```cpp
// 单层初始化
cost_table[i][i] = getGlobalLayerCycle(op[i]);   // 单层全局执行成本

// 区间扩展
for len = 2 to n:
  for start = 0 to n - len:
    end = start + len - 1

    // 选项 1：整体合并为一个 group
    cost = is_layer_group_valid(ops[start..end]) ? group_cost : MAX_COST

    // 选项 2：在某点分割，两段各自 tiling
    for sweep = start to end-1:
        temp = cost_table[start][sweep] + cost_table[sweep+1][end]
        if temp < cost:
            cost = temp
            cut_points[start][end] = sweep

    cost_table[start][end] = cost
```

`is_layer_group_valid()` 内部完整执行第四至第七节描述的 shape_secs 搜索，返回最小 `group_cost`。这使得 DP 的代价函数直接反映实际硬件执行周期，而非启发式估计。

---

## 十、BM1684X Conv 后端代码生成

完整的 `local_sec_info_t` 最终传入 `codegen_local_bm1684x()`，生成 TIU（Tensor Integer Unit）和 TDMA 指令序列。后端根据 `sec_info` 中的以下字段选择硬件路径：

| 字段 | 含义 | 影响 |
|------|------|------|
| `is_h_split` | H 维是否分割 | 决定是否需要 halo padding |
| `h_idx < 0` | 有 top padding | 硬件自动在上边界补零 |
| `use_3ic_opt` | 3IC 优化标志 | 切换 depthwise 融合内核 |
| `out_h_slice` | 输出 H 切片大小 | 决定 TIU 的输出地址步幅 |

---

## 十一、完整流程总结

```
GroupMethod（动态规划）
    │
    ├─ 对每个候选 group 调用 is_layer_group_valid()
    │       │
    │       ├─ get_group_max_secs()
    │       │       └─ 收集所有 op 的维度分割上限（Conv: H 主分割，W 受 DMA 约束）
    │       │
    │       ├─ init_group_data_secs()
    │       │       └─ 从 LMEM 容量反推初始 shape_secs 下界
    │       │
    │       ├─ assignTimeStep()（反向传播）
    │       │       └─ Conv BackwardH/W：output slice → input slice（含 halo 计算）
    │       │
    │       └─ assignLmemAddrWithSecs()（搜索 + 分配）
    │               ├─ 枚举 shape_secs 候选（N×C×D×H×W 五维，阈值剪枝）
    │               ├─ try_this_shape_secs()：assignTimeStep + assignLmemAddr + getGroupCycle
    │               └─ 选出使周期最小且 LMEM 合法的 shape_secs
    │
    ├─ DP 选出全局最优分组方案
    │
    └─ assign_sec_info_kernel()
            └─ 填充 local_sec_info_t（含负 h_idx padding 编码）
            └─ codegen_local_bm1684x() → TIU/TDMA 指令
```

### 关键设计取舍

1. **H 优先分割**：初始化时先耗尽 N，再耗尽 H，最后才用 W，是因为 H 分割的 halo overhead 最小（仅两端各多 `kh-1` 行），而 W 分割 halo overhead 与 kw 成比例且存在 DMA 对齐问题。

2. **权重不分割**（GROUP_NORMAL）：Conv 权重 `(Cout, Cin/g, kH, kW)` 随输入 H 分割时**全部留在 LMEM**，反复复用。若 LMEM 不足以放下完整权重，会切换到多 group 或调整 C 分组策略，而不是切分权重。

3. **负 h_idx 编码 padding**：避免在 LMEM 中预分配 padding 行，节约宝贵的片上内存，由硬件在 TIU 边界自动补零。

4. **搜索成本实测**：`try_this_shape_secs()` 每次调用都完整地分配 LMEM 地址并估算周期，计算精确但开销较大，因此通过"连续失败 3 次则剪枝"策略控制搜索空间。

---

---

## 十二、LMEM 地址分配：`assignLmemAddr()`

```
lib/Dialect/Tpu/Transforms/LayerGroup/LmemAllocator.cpp:990-1347
```

在确定 `shape_secs` 之后，需要给每个张量缓冲区在 LMEM 中分配具体的地址偏移。

### 12.1 缓冲区描述结构

```cpp
struct mem_buffer_value_t {
    int64_t start_ts;     // 该缓冲区首次被使用的时间步
    int64_t end_ts;       // 该缓冲区最后一次被使用的时间步
    int64_t addr;         // 分配的 LMEM 偏移地址
    int64_t size;         // 缓冲区字节数
    int64_t align_bytes;  // 对齐要求（EU_BYTES 或 DMA_ALGN_BYTES）
};
```

所有缓冲区按以下优先级排序后依次分配：
**冲突数（conflict）降序 → 面积（area）降序 → 起始时间步升序 → 索引升序**

"冲突数"指与该缓冲区在同一时间步并发使用、且由不同执行单元（GDMA 或 NPU）访问的其他缓冲区数量，冲突越多的缓冲区越需要优先放置以避免 bank conflict。

### 12.2 Bank Conflict 回避机制

LMEM 由若干等大的 Bank 组成（如 BM1684X：64 Banks，每 Bank 4 KB）。若 GDMA 操作的缓冲区与 BDC（NPU）操作的缓冲区落在同一 Bank，会产生 bank conflict 导致停顿。

```cpp
// 对每个时间步 ts：
for (auto &buf : buffers_at_ts) {
    if (buf.is_gdma) {
        // 找出同 ts 内 NPU 使用的所有 bank，排除之
        exclude_banks |= npu_banks_at(ts);
    } else {
        // 找出同 ts 内 GDMA 使用的所有 bank，排除之
        exclude_banks |= gdma_banks_at(ts);
    }
}

// 分配时跳过 excluded bank 范围
for (auto &space : avail_lmems) {
    // 过滤掉落入 excluded bank 的空间段
    ...
    // 在剩余空间中找第一个 >= buffer.size 的空闲区间
}
```

若排除 bank 后找不到合法空间，则回退到允许 bank conflict 的位置分配（正确性优先于性能）。

### 12.3 双缓冲的实现方式

双缓冲并非在此阶段显式分配两块空间，而是通过 `tensor_info_t.stage` 字段在时间步调度阶段隐式实现：

- `stage=0`：对应软件流水线第 0 阶段（预取下一切片）
- `stage=1`：对应第 1 阶段（当前切片计算）
- `stage=2`：对应第 2 阶段（回写上一切片）

LMEM 分配器为每个 `(tensor, stage)` 组合分配独立地址。在稳态流水中，阶段 0（预取）与阶段 1（计算）的缓冲区同时存在于 LMEM，自然形成 ping-pong 双缓冲效果。

---

## 十三、时间步构建与切片后向传播：`BasicTimeStep`

```
lib/Dialect/Tpu/Transforms/LayerGroup/BasicTimeStep.cpp:376-557
```

### 13.1 `gen_all_mem_buffer_ts()` —— 缓冲区生命周期分析

该函数遍历所有时间步，为每个张量缓冲区确定 `start_ts` 和 `end_ts`：

```
对每个时间步 ts：
  对该 ts 的层操作（BDC）：
    → 输出张量：新建缓冲，start_ts = ts，end_ts = 待定
    → 输入张量：更新已有缓冲，end_ts = ts

  对该 ts 的 GDMA 操作：
    → LOAD 操作：新建缓冲，start_ts = ts，end_ts = 待定
    → STORE 操作：更新已有缓冲，end_ts = ts
```

三阶段软件流水线中：
- 权重（`hold_in_lmem=true`）：`start_ts=0`，`end_ts=total_ts-1`，横跨全部时间步
- 激活切片（动态 Load/Store）：每隔 `swpipl_stage_num` 个时间步轮换
- 中间缓冲（`start_ts == end_ts`）：每个时间步独立分配与释放

### 13.2 切片后向传播：`stripe_mine_idx_slice()`

对 Conv 链（如 `Input → Conv1 → BN → ReLU → Conv2 → Output`），已知输出的 `shape_secs`，需反推每一层对应的输入切片索引与大小。

传播方向：从 group 最后一个 op 向第一个 op 逆向推导，每个 op 实现自己的 `BackwardH/BackwardW` 方法：

```
Conv2 output slice → Conv2 input slice（含 halo）
    ↓
BN/ReLU：逐元素操作，input_slice = output_slice（透传）
    ↓
Conv1 output slice → Conv1 input slice（含 halo）
    ↓
Input 张量的最终加载范围
```

对于多个连续 3×3 Conv（`stride=1, pad=1`），每过一层 halo 各增加 2 行，因此同一 group 内串联 Conv 越多，输入切片相对输出切片的放大比例越大，LMEM 效率越低。这也是 GroupMethod DP 中将过长 Conv 链拆分为多个 group 的根本动因。

---

## 十四、周期估算：`CycleCalculator::getGroupCycle()`

```
lib/Dialect/Tpu/Transforms/LayerGroup/CycleCalculator.cpp:859-1108
```

周期估算是 `try_this_shape_secs()` 中选择最优 `shape_secs` 的核心依据，采用三阶段软件流水线模型：

### 14.1 三阶段流水线模型

```
total_cycle = filling_cycle + kernel_cycle × kernel_loop + draining_cycle

其中：
  loop_num    = nsecs × csecs × hsecs × dsecs × wsecs
  kernel_loop = max(loop_num - swpipl_stage_num, 0)
```

- **Filling phase**（流水填充）：前 `swpipl_stage_num` 次迭代，流水线尚未满载
- **Kernel phase**（稳态）：中间 `kernel_loop` 次迭代，每次迭代的 BDC 计算与 GDMA 搬运完全重叠
- **Draining phase**（流水排空）：最后几次迭代，流水线逐步排空

### 14.2 每时间步的周期计算

```cpp
// 对每个时间步 ts：
ts_bdc_cycle  = sum of layer cycles in this ts;   // BDC 执行时间
ts_gdma_cycle = sum of GDMA cycles in this ts;    // GDMA 传输时间

// 取较大者（BDC 与 GDMA 并行执行）
ts_cycle = max(ts_bdc_cycle, ts_gdma_cycle);
```

若 `calc_bdc_slack=true`（激进模式）：

```cpp
// BDC slack：BDC 比 GDMA 多出的空闲时间，可被 GDMA 利用
bdc_slack_cycle = bdc_cycle - max(bdc_cycle, gdma_cycle);
```

### 14.3 多核周期折减

BM1688（2 核）和 BM1690（8 核）的 `loop_num` 按核数折半，但 GDMA 带宽也相应降低（24 GB/s → 15 GB/s），实际加速比小于核数：

```cpp
if (num_core == 2) {
    loop_num = ceiling_func(loop_num, 2);
    // GDMA bandwidth: 24 → 15 GB/s（其余核占用部分带宽）
}
```

### 14.4 Conv 的周期估算精度

BDC 周期通过调用 `lgOp.lg_codegen_local_bm1684x()` 生成伪代码（不写入寄存器），从硬件性能模型获取精确周期数；GDMA 周期由数据量除以带宽估算。两者综合后的精度足以指导 `shape_secs` 搜索做出正确的性能排序。

---

## 十五、快速搜索：`sc_method_quick_search()`

```
lib/Dialect/Tpu/Transforms/LayerGroup/LmemAllocator.cpp:2118-2190
```

在默认编译模式（`shape_secs_search_level=0`）下，使用快速搜索而非全量五维枚举，适用于大多数 Conv 模型：

```
初始 shape_secs（来自 init_group_data_secs）
    │
    ▼
while try_num < MAX_TRY_NUM(=20) && shape_secs <= max_shape_secs:
    ret = try_this_shape_secs(shape_secs)

    if ret == SECS_VALID 或 SECS_VALID_AND_BETTER:
        break   ← 找到合法方案即停止

    elif ret == SECS_LMEM_INVALID:
        inc_slice_num()  ← 按 N→C→D→H→W 顺序增加一段

    else:
        break   ← 时间步非法，无法继续

    try_num++
```

**与 `sc_method_search_better_v1` 的区别**：快速搜索在找到第一个合法方案后立即退出，不保证全局最优；而 `search_better` 系列会继续搜索更优的 `shape_secs`，代价是更高的编译时间。可通过 `--shape_secs_search_level 1` 开启强化搜索。

---

## 十六、各组件协作时序

以单个 3×3 Conv（BM1684X，H 维三段切分）为例，完整协作流程如下：

```
GroupMethod::is_layer_group_valid([Conv])
│
├─ get_group_max_secs()
│    → max_nsecs=N, max_hsecs=H, max_wsecs=1（行宽不满足 DMA 对齐）
│
├─ init_group_data_secs()
│    → total_size = input(N×C×H/3×W) + weight(Cout×Cin×3×3) + output + buf
│    → 若超出 LMEM，hsecs 增加
│    → 返回初始 shape_secs = {1, 1, 1, 3, 1}
│
├─ assignTimeStep(shape_secs={1,1,1,3,1}, gen_idx=false)
│    → stripe_mine_max_slice()：计算各维最大 slice 大小
│    → gen_all_mem_buffer_ts()：建立缓冲区生命周期表
│    → 共 3 个时间步：ts0(slice_h0), ts1(slice_h1), ts2(slice_h2)
│
├─ assignLmemAddrWithSecs()
│    ├─ sc_method_quick_search()
│    │    ├─ try_this_shape_secs({1,1,1,3,1})
│    │    │    ├─ assignTimeStep(gen_idx=true)
│    │    │    │    → BackwardH 计算每段 input slice：
│    │    │    │      out_slice=H/3 → in_slice=H/3+2（halo 各 1 行）
│    │    │    ├─ assignLmemAddr()
│    │    │    │    → 按 conflict→area→start_ts 排序分配
│    │    │    │    → weight 全程驻留，input/output 双缓冲轮换
│    │    │    │    → bank conflict 检测，adjust addr
│    │    │    └─ getGroupCycle()
│    │    │         → filling: ts0+ts1 两步流水填充
│    │    │         → kernel: ts0~tsN 稳态（max(BDC, GDMA) per step）
│    │    │         → draining: 最后两步流水排空
│    │    └─ LMEM 合法 → 返回 shape_secs={1,1,1,3,1}，group_cost=T
│    └─ 更新 lg_info.shape_secs, group_cost
│
└─ return true（group 合法，最优 shape_secs 已确定）

         ↓

assign_sec_info_kernel() 为每个时间步填充 local_sec_info_t
codegen_local_bm1684x() → TIU 指令 + TDMA 指令序列
```

---

## 验证入口

可通过以下文件直接阅读核心实现：

| 功能 | 文件路径 |
|------|---------|
| shape_secs_t / tensor_info_t 定义 | `include/tpu_mlir/Dialect/Tpu/Transforms/LayerGroup/LayerGroupDefs.h` |
| local_sec_info_t 定义 | `include/tpu_mlir/Interfaces/LocalGenInterface.h:44-88` |
| 分割约束收集 | `lib/Dialect/Tpu/Transforms/LayerGroup/LayerGroupUtil.cpp:611-722` |
| 初始 shape_secs 反推 | `lib/Dialect/Tpu/Transforms/LayerGroup/LayerGroupUtil.cpp:749-826` |
| H/W 反推公式 | `lib/Dialect/Tpu/Impl/Conv2D.cpp:346-420` |
| padding 负索引编码 | `lib/Dialect/Tpu/Impl/Conv2D.cpp:378-420（assign_sec_info_kernel）` |
| LMEM 缓冲估算 | `lib/Dialect/Tpu/Impl/Conv2D.cpp（getBufferSize_bm1684x）` |
| LMEM 地址分配 | `lib/Dialect/Tpu/Transforms/LayerGroup/LmemAllocator.cpp:990-1347` |
| shape_secs 枚举搜索 | `lib/Dialect/Tpu/Transforms/LayerGroup/LmemAllocator.cpp:1646-1785` |
| shape_secs 快速搜索 | `lib/Dialect/Tpu/Transforms/LayerGroup/LmemAllocator.cpp:2118-2190` |
| 动态规划分组 | `lib/Dialect/Tpu/Transforms/LayerGroup/GroupMethod.cpp:1380-1640` |
| 缓冲区生命周期构建 | `lib/Dialect/Tpu/Transforms/LayerGroup/BasicTimeStep.cpp:376-557` |
| 三阶段流水线周期估算 | `lib/Dialect/Tpu/Transforms/LayerGroup/CycleCalculator.cpp:859-1108` |
