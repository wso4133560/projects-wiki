# TPU-MLIR 内存分配详细实现分析

## 一、内存层次体系概览

TPU-MLIR 面向的 SOPHGO TPU 硬件具有明确的三层内存层次：

```
GMEM（全局内存，DDR/HBM）
  ├── 权重区（Coeff Region）：COEFF_START_ADDR 起
  ├── 激活区（Activation Region）：CTX_START_ADDR 起
  └── IO 区（可选，IO_ALONE / IO_TAG 模式）

L2 SRAM（仅 BM1690/BM1690E）
  └── 主要存储 LUT 查找表（激活函数预计算）

LMEM（本地内存，片上 SRAM）
  ├── 权重缓冲（Weight Buffer）：每次 DMA 搬入一份完整权重
  ├── 激活缓冲（Activation Buffer）：ping-pong 双缓冲
  └── 临时缓冲（Operation Buffer）：算子自身申请的中间存储
```

编译器的地址分配工作分为两个独立阶段：
1. **GMEM 分配**（`AddressAssign` Pass）：为整个模型的权重和激活分配 DDR 偏移
2. **LMEM 分配**（`LmemAllocator`）：在 LayerGroup 范围内分配片上 SRAM 地址

两者使用完全不同的算法，因为内存规模和约束截然不同。

---

## 二、硬件参数：各芯片 LMEM 规格

```
lib/Backend/Arch.cpp
include/tpu_mlir/Backend/Arch.h
```

编译器的所有内存计算公式都依赖以下硬件常量（以单例模式管理）：

| 参数 | 含义 | BM1684X | BM1688 | CV183X |
|------|------|---------|--------|--------|
| `NPU_NUM` | NPU 核心数（Lane 数）| 32 | 32 | 32 |
| `EU_BYTES` | 执行单元字节宽度 | 16 | 16 | 16 |
| `LMEM_BYTES` | 片上 LMEM 总量 | 5 MB | 5 MB | 3 MB |
| `LMEM_BANKS` | LMEM Bank 数量 | 16 | 16 | 4 |
| `LMEM_BANK_BYTES` | 单 Bank 字节数 | 320 KB | 320 KB | 768 KB |
| `DMA_ALGN_BYTES` | DMA 对齐要求 | 512 B | 512 B | 64 B |
| `ALIGN_4N` | 是否 4N 对齐模式 | false | false | true |

**EU_NUM 的推导：**

```cpp
int64_t Arch::eu_num(double dbytes) {
    return EU_BYTES / dbytes;
    // INT8:  EU_NUM = 16 / 1 = 16
    // FP16:  EU_NUM = 16 / 2 = 8
    // FP32:  EU_NUM = 16 / 4 = 4
}
```

这个值决定了 H×W 维度在 LMEM 中必须对齐到的粒度。

---

## 三、张量 LMEM 字节数计算：`get_tensor_lmem_bytes()`

```
lib/Backend/Arch.cpp:119-180
```

这是整个 LMEM 规划的基础公式，被所有内存估算调用：

### 3.1 标准模式（BM168x，`ALIGN_4N=false`）

```
lmem_bytes = n × d × ⌈c/NPU_NUM⌉ × align_up(h×w, EU_NUM) × type_bytes

其中：
  EU_NUM    = EU_BYTES / type_bytes
  type_bytes = type_bits / 8
```

**物理含义：**
- `⌈c/NPU_NUM⌉`：每个 NPU Lane 负责的通道数，所有 Lane 并行计算
- `align_up(h×w, EU_NUM)`：每个 Lane 内，特征图展开为一维向量后对齐到 EU 宽度

**数值示例（FP16，C=3，H=224，W=224，N=1）：**
```
EU_NUM      = 16 / 2 = 8
c_per_npu   = ⌈3 / 32⌉ = 1
hw_aligned  = align_up(224×224, 8) = 50176
lmem_bytes  = 1 × 1 × 1 × 50176 × 2 = 100352 字节 ≈ 98 KB
```

### 3.2 4N 对齐模式（CV18xx，`ALIGN_4N=true`）

CV18xx 要求 N 维按 `4/type_bytes` 对齐，以满足 DMA 对齐约束：

```cpp
int64_t eu_num = Arch::eu_num(4);           // 固定 4 字节宽度计算 EU_NUM
int64_t n_aligned = ceiling_func(n, 4 / (int64_t)dbytes);
return n_aligned × d × c_per_npu × align_up(h×w, eu_num) × 4;
```

### 3.3 INT4 特殊处理

INT4 张量需保证按 2 个元素对齐（因为 INT4 以 byte 为最小存储单位，两个元素共享一个 byte）：

```cpp
if (type_bits == 4) {
    return align_up(n × d × c_per_npu × eu_aligned, 2L) × 0.5;
}
```

---

## 四、全局内存（GMEM）分配

```
lib/Dialect/Tpu/Transforms/AddressAssign/BMAddressAssign.cpp
```

GMEM 分配分为权重分配和激活分配两个阶段，按固定顺序依次执行。

### 4.1 权重分配（Weight Allocation）

权重从 `COEFF_START_ADDR` 开始连续排列，静态权重在前、动态权重在后：

```cpp
auto addr = start_addr;   // = BM168x::COEFF_START_ADDR

// 两轮：is_static=true 先，is_static=false 后
for (auto is_static : {true, false}) {
    func.walk([&](top::WeightOp op) {
        // 1. 确定存储模式
        auto stmode = op.getStoreMode().has_value()
            ? parseStoreMode(op.getStoreModeAttr())
            : STORE_MODE_1N;

        // 2. 计算字节数
        auto [n_factor, byte_factor] = stmode_map.at(stmode);
        // 1N: (1, type_bytes), 2N: (2/type_bytes, 2), 4N: (4/type_bytes, 4)
        int64_t bytes = ceiling_func(n, n_factor) * byte_factor * c * h * w;
        bytes = ceiling_func(bytes, 8L);  // INT4 对齐

        // 3. MatMul 右矩阵特殊处理（确保列访问对齐）
        if (byte_factor == 4 && module::IsRightMat(out_value)) {
            int64_t bytes_2 = ceiling_func(K, NPU_NUM) * N * byte_factor;
            bytes = std::max(bytes, bytes_2);
        }

        // 4. 设置地址并前进
        module::setAddress(out_value, addr);
        addr = align_up(addr + bytes, BM168x::ALIGNMENT);
    });
}
module::setCoeffAddr(m, start_addr);
module::setCoeffSize(m, addr - start_addr);
```

**三种存储模式的内存布局：**

| 模式 | 适用场景 | N 方向排布 | 每元素字节 |
|------|---------|-----------|----------|
| 1N | 通用 | 实际 N �� | type_bytes |
| 2N | FP16/BF16 权重 | 每 2 个 batch 共一组 | 2 |
| 4N | INT8/FP32 权重 | 每 4 个 batch 共一组 | 4 |

### 4.2 激活分配（Activation Allocation）

激活张量采用**活跃区间（Live Range）分析**，最大化地址复用：

#### 4.2.1 活跃区间构建

```cpp
// 反向扫描所有 op，构建每个张量的 [start_op_idx, end_op_idx]
for (auto iter = all_ops.rbegin(); iter != all_ops.rend(); ++iter) {
    auto op = *iter;
    updateLiveRangeofBMOps(op, ops_loc, liveRange, common_ops, inplace_ops);
}
```

活跃区间规则：
- **Input/ReturnOp**：`[0, 0xFFFFFFFF]`，整个推理生命周期
- **普通 Op**：`[定义处 op_idx, 最后使用处 op_idx]`
- **InplaceOp（Concat/Reshape/Slice 等）**：与前驱共享地址，不独立分配

#### 4.2.2 GmemAllocator 首次适配分配

```cpp
// 1. 按活跃开始时间排序
GmemAllocator::sortOpByLiveStart(common_ops, liveRange);

// 2. 带复用的贪心分配
auto gmemUsed = allocator.assignGaddr(common_ops, liveRange,
                                      reuse_addr=true, start_addr);
```

首次适配算法维护一个空闲块列表：对每个待分配张量，找第一个 `size >= tensor_size` 的空闲块；活跃区间不重叠的张量可以复用同一地址块。

### 4.3 就地操作（Inplace Op）地址分配

就地操作不独立分配内存，而是复用输入地址：

```cpp
// Concat：输出复用第一输入地址，后续输入顺序排列
if (auto concatOp = dyn_cast<tpu::ConcatOp>(op)) {
    int64_t addr = module::getAddress(concatOp.getInputs()[0]);
    module::setAddress(concatOp.getOutput(), addr);
    int64_t offset = module::getBytes(inputs[0]);
    for (int i = 1; i < inputs.size(); i++) {
        module::setAddress(inputs[i], addr + offset);
        offset += module::getBytes(inputs[i]);
    }
}

// Reshape / Identity：地址直接透传
// Slice：输出地址 = 输入地址 + 切片偏移量
// Insert：输入输出共享同一地址
```

---

## 五、LMEM 分配核心系统

### 5.1 关键数据结构

```
include/tpu_mlir/Dialect/Tpu/Transforms/LayerGroup/LayerGroupDefs.h
include/tpu_mlir/Dialect/Tpu/Transforms/LayerGroup/LmemAllocator.h
```

#### `mem_buffer_key_t`：缓冲区身份标识

```cpp
typedef struct mem_buffer_key {
    lmem_type_t type;     // LMEM_WEIGHT / LMEM_ACTIVATION / LMEM_OPERATION
    Value       value;    // 对应的 MLIR Value（激活/权重）
    Operation  *op;       // 对应的 Op（临时缓冲）
    int64_t     conflict; // 冲突标记（排序用，运行时动态更新）

    bool operator<(const mem_buffer_key &other) const {
        // 先按 type，同 type 按指针排序
        if (type != other.type) return type < other.type;
        if (type == LMEM_OPERATION) return op < other.op;
        return value.getImpl() < other.value.getImpl();
    }
} mem_buffer_key_t;
```

#### `mem_buffer_value_t`：缓冲区分配状态

```cpp
typedef struct mem_buffer_value {
    int64_t start_ts;    // 缓冲区首次被使用的时间步索引
    int64_t end_ts;      // 缓冲区最后一次被使用的时间步索引
    int64_t addr;        // 已分配的 LMEM 偏移地址（字节）
    int64_t size;        // 缓冲区字节数
    int64_t align_bytes; // 对齐要求（EU_BYTES 或 DMA_ALGN_BYTES）
} mem_buffer_value_t;
```

#### `avail_space_t`：每个缓冲区视角下的可用空间

```cpp
typedef struct {
    std::list<MemBlock>    avail_lmems;   // 可用内存块列表（地址, 大小）
    std::set<int64_t>      exclude_banks; // 因 bank conflict 排除的 bank 编号
} avail_space_t;

using BufferAvailSpace = std::map<mem_buffer_key_t, avail_space_t>;
```

每个待分配缓冲区都维护自己的可用空间视图，避免重复计算全局可用空间。

#### `membuf_sort_std_t`：排序键

```cpp
typedef struct membuf_sort_standard {
    int64_t area;       // 生命周期长度 × 字节数（面积）
    int64_t start_ts;   // 生命周期起始时间步
    int64_t idx;        // 缓冲区索引（保证稳定排序）
} membuf_sort_std_t;

using MemBufSortStd = std::pair<mem_buffer_key_t, membuf_sort_std_t>;
```

### 5.2 缓冲区排序算法

```
lib/Dialect/Tpu/Transforms/LayerGroup/LmemAllocator.cpp:29-48
```

分配前对所有待分配缓冲按四级优先级排序：

```
优先级 1（最高）: conflict 值高者优先
    ↓ 相等时
优先级 2: area（时间步数 × 字节数）大者优先
    ↓ 相等时
优先级 3: start_ts 小者优先（生命周期早开始的先分配）
    ↓ 相等时
优先级 4: idx 小者优先（保证确定性排序）
```

**面积的计算：**

```cpp
// 正常情况（start_ts <= end_ts）
area = (end_ts - start_ts + 1) × size;

// 循环情况（跨时间步边界，常见于软件流水线）
area = ((end_ts + 1) + (timestep_num - start_ts)) × size;
```

这个"面积"度量了缓冲区对 LMEM 空间的"压力"：生命周期越长、体积越大的缓冲区越早分配，以减少后续缓冲区因其存在而受到的限制。

### 5.3 可用内存块管理：`update_avail_lmems()`

```
lib/Dialect/Tpu/Transforms/LayerGroup/LmemAllocator.cpp:239-304
```

LMEM 可用空间以**不重叠的有序区间列表**表示，每次分配后需更新该列表。根据新占用区间与现有可用块的关系，分四种情况处理：

```
情况 1：完全覆盖 → 删除该可用块
  avail: |--------|
  excl: |----------|

情况 2：右侧重叠 → 缩短可用块右边界
  avail:   |--------|
  excl:        |---------|
  result:  |---|

情况 3：左侧重叠 → 缩短可用块左边界
  avail:       |--------|
  excl:  |--------|
  result:         |---|

情况 4：内部切割 → 将可用块一分为二
  avail: |--------------|
  excl:       |-----|
  result:|---|       |---|
```

```cpp
bool LmemAllocator::update_avail_lmems(std::list<MemBlock> &avail_lmems,
                                        const MemBlock &exclude_lmem) {
    for (auto avail_iter = avail_lmems.begin();
         avail_iter != avail_lmems.end(); ) {
        int64_t a_s = avail_iter->first;
        int64_t a_e = a_s + avail_iter->second;
        int64_t e_s = exclude_lmem.first;
        int64_t e_e = e_s + exclude_lmem.second;

        if (a_s >= e_s && a_e <= e_e) {
            avail_iter = avail_lmems.erase(avail_iter); // 情况1
        } else if (a_s < e_s && a_e > e_s && a_e <= e_e) {
            avail_iter->second = e_s - a_s;  // 情况2
            ++avail_iter;
        } else if (a_s >= e_s && a_s < e_e && a_e > e_e) {
            avail_iter->first  = e_e;
            avail_iter->second = a_e - e_e;  // 情况3
            ++avail_iter;
        } else if (a_s < e_s && a_e > e_e) {
            // 情况4：插入右半段
            avail_lmems.insert(std::next(avail_iter),
                               {e_e, a_e - e_e});
            avail_iter->second = e_s - a_s;
            ++avail_iter;
        } else {
            ++avail_iter;
        }
    }
}
```

### 5.4 Bank Conflict 回避机制

```
lib/Dialect/Tpu/Transforms/LayerGroup/LmemAllocator.cpp:651-713
```

TPU 的 LMEM 在物理上被划分为多个 Bank（如 BM1684X：16 个 Bank，每 Bank 320 KB）。当 GDMA 搬运操作和 BDC（NPU）计算操作在**同一个时间步**访问**同一个 Bank** 时，会产生访问冲突，导致一方等待。

**Bank 编号计算：**

```cpp
void LmemAllocator::find_used_banks(std::set<int64_t> &used_banks,
                                     int64_t lmem_addr, int64_t lmem_size) {
    int64_t bank_size  = Arch::LMEM_BANK_BYTES;
    int64_t start_bank = lmem_addr / bank_size;
    int64_t end_bank   = (lmem_addr + lmem_size - 1) / bank_size;
    for (int64_t i = start_bank; i <= end_bank; ++i)
        used_banks.insert(i);
}
```

**排除 Bank 的决策逻辑：**

对缓冲区 B 的生命周期内每个时间步 `ts`：
1. 若 B 被 GDMA 使用（Load/Store），则找出该时间步内 NPU 使用的最近一次已分配缓冲所占的 Bank → 将这些 Bank 加入 B 的 `exclude_banks`
2. 若 B 被 NPU 使用（BDC 计算），则对称地找 GDMA 使用的 Bank

```cpp
void LmemAllocator::update_exclude_banks(...) {
    for (int64_t ts = buffer_value.start_ts; /* 遍历生命周期 */; ) {
        bool is_npu_use  = is_buffer_used_by_npu(buffer_key, getLayers(ts));
        bool is_gdma_use = is_buffer_used_by_gdma(buffer_key, getTensors(ts));

        if (is_gdma_use || is_npu_use) {
            // 找最近分配的缓冲在此时间步是否被另一类执行单元使用
            for (auto op : getLayers(ts)) {
                if (is_relate_op(recent_buffer_allocated, op, ts, ...)) {
                    find_used_banks(recent_used_banks,
                                    recent_buffer_value.addr,
                                    recent_buffer_value.size);
                    break;
                }
            }
        }
        if (!recent_used_banks.empty()) {
            exclude_banks.insert(recent_used_banks.begin(),
                                 recent_used_banks.end());
            break;
        }
        ts = (ts + 1) % timestep_num;
    }
}
```

分配时，先从 `avail_lmems` 中过滤掉所有 `exclude_banks` 覆盖的地址区间，再在剩余空间中查找首个满足大小要求的块。若找不到（Bank 约束过强），则允许 Bank Conflict 回退分配——正确性优先于性能。

### 5.5 主分配循环：`assignLmemAddr()`

```
lib/Dialect/Tpu/Transforms/LayerGroup/LmemAllocator.cpp:990-1347
```

核心算法是一个**贪心的逐步最小地址分配循环**：

```
初始化：
  membuf_list      ← 所有待分配缓冲，按 area/start_ts/idx 计算初始排序键
  npu_membuf_heap  ← 各时间步内被 NPU 使用的缓冲集合（按时间步索引）
  gdma_membuf_heap ← 各时间步内被 GDMA 使用的缓冲集合
  buffer_avail_space ← 每个缓冲的独立可用空间视图（初始为全 LMEM）

主循环 while (!membuf_list.empty()):
  Step 1. 更新冲突标记
    → 凡在 npu_membuf_heap 或 gdma_membuf_heap 中出现的缓冲，conflict=1

  Step 2. 对 membuf_list 重新排序
    → conflict > area > start_ts > idx

  Step 3. 遍历 membuf_list，为每个候选缓冲查询可分配的最低地址
    → 第一次分配：直接从地址 0 开始（最简单情况）
    → 后续分配：调用 global_find_avail_lmem_location()
        1. update_exclude_banks()   ← 更新不可使用的 Bank
        2. update_avail_lmems()     ← 排除已占用和生命周期重叠的地址段
        3. find_avail_lmem_location() ← 在剩余可用块中找最低地址

  Step 4. 选出地址最低的候选缓冲作为本轮分配目标 tgt_membuf
    → 贪心策略：最低地址意味着 LMEM 碎片化程度最低

  Step 5. 执行分配
    → time_step->set_lmem_addr(tgt_membuf, tgt_min_address)
    → 更新 lmem_occupy = max(lmem_occupy, tgt_addr + tgt_size)

  Step 6. 清理
    → membuf_list.erase(tgt_membuf)
    → conflict_heap_delete(npu_membuf_heap, gdma_membuf_heap, tgt_membuf)
    → buffer_avail_space.erase(tgt_membuf)

结束：
  time_step->set_lmem_occupy(lmem_occupy)
  assignL2memAddr(lg_info, time_step)   ← L2 SRAM（LUT）分配
```

**每个缓冲区的独立可用空间视图设计是关键：** `buffer_avail_space[buf]` 记录的是"从 buf 的视角看，哪些 LMEM 地址是可用的"。这避免了每次查找都从头重算全局可用空间，是算法效率的核心来源。

---

## 六、冲突堆（Conflict Heap）管理

```
lib/Dialect/Tpu/Transforms/LayerGroup/LmemAllocator.cpp:840-990
```

冲突堆记录了"当前仍未完成分配的缓冲区在各时间步的使用情况"，用于快速判断某缓冲是否处于"冲突状态"（即同时被 NPU 和 GDMA 关注）。

```cpp
// membuf_heap_create：为每个时间步建立集合
void membuf_heap_create(
    std::vector<std::set<mem_buffer_key_t *>> &npu_membuf_heap,
    std::vector<std::set<mem_buffer_key_t *>> &gdma_membuf_heap,
    std::list<MemBufSortStd> &membuf_list,
    BasicTimeStepPtr &time_step) {

    npu_membuf_heap.resize(timestep_num);
    gdma_membuf_heap.resize(timestep_num);

    for (auto &mem_buf : membuf_list) {
        auto &buf_val = time_step->get_lmem_buffer_value(mem_buf.first);
        // 在 [start_ts, end_ts] 范围内登记
        for (int64_t ts = buf_val.start_ts; ts <= buf_val.end_ts; ++ts) {
            if (is_buffer_used_by_npu(mem_buf.first, getLayers(ts)))
                npu_membuf_heap[ts].insert(&mem_buf.first);
            if (is_buffer_used_by_gdma(mem_buf.first, getTensors(ts)))
                gdma_membuf_heap[ts].insert(&mem_buf.first);
        }
    }
}
```

每完成一个缓冲区的分配，立即从冲突堆中删除，防止已分配缓冲继续影响后续决策。

---

## 七、缓冲区生命周期追踪：`gen_all_mem_buffer_ts()`

```
lib/Dialect/Tpu/Transforms/LayerGroup/BasicTimeStep.cpp:376-557
```

LMEM 分配前，`BasicTimeStep` 通过前向扫描时间步表为每个缓冲区确定 `start_ts` 和 `end_ts`：

```
时间步表（timestep_table_）结构：
  每行（TimestepRow）= { tpu0_ts_field: [op列表], gdma0_ts_field: [tensor列表] }

前向扫描规则：
  遇到层操作（BDC）的输出张量：
    → 新建缓冲条目，start_ts = current_ts，end_ts = 待定

  遇到层操作的输入张量：
    → 更新已有缓冲，end_ts = current_ts

  遇到 GDMA LOAD 张量：
    → 新建缓冲条目，start_ts = current_ts（数据从此时开始在 LMEM 中存在）

  遇到 GDMA STORE 张量：
    → 更新缓冲，end_ts = current_ts（数据不再需要后释放）
```

**特殊生命周期：权重（`hold_in_lmem=true`）：**

若某权重在整个 group 生命周期内都不改变，则标记为 `hold_in_lmem`，其 `start_ts=0`，`end_ts=total_timesteps-1`，始终占用 LMEM，避免反复 DMA 搬运。

---

## 八、软件流水线中的内存分配

### 8.1 三阶段流水线与双缓冲

TPU-MLIR 使用三阶段软件流水线实现计算与 DMA 的重叠：

```
Stage 0（预取）: GDMA Load → 将 slice_i+1 从 GMEM 搬入 LMEM
Stage 1（计算）: BDC Compute → 对 slice_i 执行 Conv/MatMul 等
Stage 2（回写）: GDMA Store → 将 slice_i-1 的输出写回 GMEM
```

每个张量的 `tensor_info_t.stage` 字段（0/1/2）标注它属于哪个流水阶段。

### 8.2 双缓冲的内存开销

稳态时 Stage 0 和 Stage 1 的激活缓冲同时存在于 LMEM，所以实际 LMEM 需要预留**两倍**激活切片的空间：

```
LMEM 布局（H 方向 2 段，双缓冲）：

[0    ]: input_buf_A  ← Stage-1 计算当前切片
[size ]: input_buf_B  ← Stage-0 预取下一切片
[2×size]: weight      ← 整个 group 生命周期内不动
[3×size]: output_buf_A ← Stage-1 写当前输出
[4×size]: output_buf_B ← Stage-2 准备写回的上一输出
```

这不是在分配阶段显式区分，而是因为 `(tensor, stage_0)` 和 `(tensor, stage_1)` 在时间步上的生命周期不重叠，分配器自然为它们分配不同地址。

### 8.3 `is_tensor_hold_in_lmem()` 判断

若一个张量被多个连续的层操作共享（如 GroupOp 内部的中间层特征图），则标记为 `hold_in_lmem=true`：

```cpp
// 持久化到 LMEM 的条件：该 Value 在 group 内被多个时间步的 BDC 操作使用
bool hold_in_lmem = (use_count_in_group > 1) &&
                    !tensor_info.mode.has(TIMESTEP_STORE);
```

持久化张量跨所有时间步均有效，分配器会为其选取不被其他短生命周期缓冲覆盖的地址区域。

---

## 九、L2 SRAM 分配：`assignL2memAddr()`

```
lib/Dialect/Tpu/Transforms/LayerGroup/LmemAllocator.cpp
```

BM1690/BM1690E 配备了额外的 L2 SRAM，主要用于 LUT（Look-Up Table）激活函数（如 `sigmoid`、`tanh`、`exp` 的分段线性近似）。

```cpp
void LmemAllocator::assignL2memAddr(LgInfo &lg_info,
                                     BasicTimeStepPtr &time_step) {
    auto &l2mem_buffer = time_step->get_l2mem_buffer();
    int64_t l2_addr = Arch::L2_SRAM_START_ADDR;

    for (auto &kv : l2mem_buffer) {
        time_step->set_l2mem_addr(kv.first, l2_addr);
        l2_addr = align_up(l2_addr + kv.second.size, Arch::ALIGNMENT);
    }

    // 检查是否超出 L2 SRAM 容量
    if (l2_addr - Arch::L2_SRAM_START_ADDR > Arch::L2_SRAM_SIZE) {
        // 降级：将 LUT 数据放入 LMEM
        assignLmemForL2(lg_info, time_step);
    }
}
```

---

## 十、GMEM IO 内存模式（IO_ALONE / IO_TAG）

```
lib/Dialect/Tpu/Transforms/AddressAssign/BMAddressAssign.cpp
lib/Backend/BM168x/BM168x.h
```

某些芯片（`SUPPORT_MEM_TAG=true`）支持将模型的输入/输出张量分配在独立的 IO 内存区，与计算激活张量隔离，方便零拷贝接口调用：

| 模式 | 策略 | 适用场景 |
|------|------|---------|
| `basic` | 输入输出与激活共享地址空间 | 通用，无特殊要求 |
| `io_alone` | 输入输出分配在独立 IO 区域 | 减少数据拷贝次数 |
| `io_tag` | 通过标签机制动态绑定 IO 地址 | 运行时动态替换 |
| `io_reuse` | IO 张量地址可被激活重用 | 极致内存压缩 |

```cpp
if (BM168x::SUPPORT_MEM_TAG) {
    addr = BM168x::CTX_START_ADDR;   // 激活区独立起点
    // IO 张量分配在 IO_ADDR[] 数组指定的位置
    for (auto input : net_inputs) {
        module::setAddress(input, BM168x::IO_ADDR[io_idx++]);
    }
}
```

---

## 十一、全流程总结

```
model_deploy.py → tpuc-opt → AddressAssign Pass
                                    │
                    ┌───────────────┴──────────────────┐
                    │                                  │
            权重分配（GMEM）                      激活分配（GMEM）
            COEFF_START_ADDR 起                  活跃区间分析
            连续顺序排列                          GmemAllocator 首次适配
            支持 1N/2N/4N 存储模式               就地操作地址复用
                    │                                  │
                    └───────────────┬──────────────────┘
                                    │
                          LayerGroup Pass
                                    │
                    ┌───────────────┴──────────────────┐
                    │                                  │
            shape_secs 搜索                    LMEM 分配（LmemAllocator）
            确定最优切片方案                    ├─ 生命周期分析（BasicTimeStep）
                                               ├─ 缓冲排序（conflict>area>ts>idx）
                                               ├─ 可用块管理（4种情况区间分割）
                                               ├─ Bank Conflict 排除
                                               ├─ 贪心最低地址分配循环
                                               └─ L2 SRAM 分配（BM1690）
                                    │
                          codegen → bmodel
```

### 核心设计取舍

1. **独立可用空间视图**：每个缓冲区维护自己的 `avail_space_t`，避免全局重算，是时间复杂度的关键优化

2. **面积优先的贪心排序**：生命周期×体积大的缓冲优先分配，降低后续分配的约束；bank conflict 缓冲最高优先，消除性能瓶颈

3. **Bank Conflict 软约束**：Bank Conflict 回避是"尽力而为"（best-effort），LMEM 不足时允许回退到有冲突分配，保证编译成功率

4. **就地操作零开销复用**：Concat/Reshape/Slice 等操作通过地址偏移复用前驱内存，无需额外分配，在激活内存峰值估算中贡献显著

5. **权重驻留 LMEM 策略**：在 group 内权重不切片，整体加载一次后通过 `hold_in_lmem=true` 跨所有时间步复用，以 LMEM 占用换取 DMA 次数减少

---

## 十二、核心文件索引

| 模块 | 文件路径 | 关键内容 |
|------|---------|---------|
| LMEM 字节数公式 | `lib/Backend/Arch.cpp:119-180` | `get_tensor_lmem_bytes()` |
| 数据结构定义 | `include/.../LayerGroupDefs.h:52-220` | key/value/avail_space/timestep |
| 排序函数 | `lib/.../LmemAllocator.cpp:29-48` | `membuf_sort_std_cmp()` |
| 区间分割 | `lib/.../LmemAllocator.cpp:239-304` | `update_avail_lmems()` |
| Bank Conflict 排除 | `lib/.../LmemAllocator.cpp:651-713` | `update_exclude_banks()` |
| 主分配循环 | `lib/.../LmemAllocator.cpp:990-1347` | `assignLmemAddr()` |
| 生命周期追踪 | `lib/.../BasicTimeStep.cpp:376-557` | `gen_all_mem_buffer_ts()` |
| GMEM 权重分配 | `lib/.../BMAddressAssign.cpp:555-627` | 权重地址顺序排列 |
| GMEM 激活分配 | `lib/.../BMAddressAssign.cpp:632-751` | 活跃区间 + 首次适配 |
| 就地操作复用 | `lib/.../BMAddressAssign.cpp:185-389` | `assignAfter()` |
