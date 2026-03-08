# TPU-MLIR Pass 列表详细分析

## 一、概述

TPU-MLIR 编译器基于 MLIR 框架构建，采用 Pass 驱动的编译流水线。整个编译过程分为三大阶段：**Top Dialect 优化**、**TopToTpu 方言转换**、**Tpu Dialect 优化与代码生成**。每个阶段由一组有序的 Pass 依次执行，共计约 40 个 Pass。

Pass 定义文件：
- `include/tpu_mlir/Dialect/Top/Transforms/Passes.td`：Top Dialect Pass
- `include/tpu_mlir/Dialect/Tpu/Transforms/Passes.td`：Tpu Dialect Pass
- `include/tpu_mlir/Conversion/Passes.td`：跨方言转换 Pass

Pass 流水线构建：`python/utils/mlir_shell.py`

---

## 二、Top Dialect Pass 详解

Top Dialect 是 TPU-MLIR 的高层硬件无关表示，对应 ONNX 抽象层次。其 Pass 的主要职责是：模块初始化、形状推导、图结构优化、量化参数注入、预/后处理融合。

### 2.1 `--init`：模块初始化

**实现**：`lib/Dialect/Top/Transforms/Init.cpp`

在整个编译流程的入口处执行，完成以下初始化工作：
- 设置目标芯片型号（chip）到模块属性中
- 配置权重存储模式（1N/2N/4N）
- 设置对数级别（log level）
- 初始化全局模块上下文，使后续 Pass 能正确读取芯片配置

所有后续 Pass 均依赖此 Pass 写入的模块属性。

### 2.2 `--deinit`：模块反初始化

编译流程末尾执行，负责清理资源：
- 可选择是否将权重数据保存到 NPZ 文件
- 释放运行时状态

### 2.3 `--processor-assign`：处理器分配

**实现**：`lib/Dialect/Top/Transforms/ProcessorAssign.cpp`

将目标芯片和量化配置写入模块：
- 支持芯片：`bm1684`、`bm1684x`、`bm1688`、`bm1690`、`cv183x`、`cv182x`、`cv181x`、`cv180x`、`sg2380` 等 15+ 种
- 量化模式：`INT8`、`INT4`、`F16`、`BF16`、`F32`、`W8F16`、`W4F16`、`W8BF16`、`W4BF16`、`F8E4M3`、`F8E5M2` 等
- 配置设备数量（num_device）和核心数量（num_core）

### 2.4 `--shape-infer`：形状推断

**实现**：`lib/Dialect/Top/Transforms/ShapeInfer.cpp`

遍历所有 Top Dialect 操作，调用每个 Op 的 `ShapeInterface::inferShape()` 方法：
- 对每个操作推导输出张量的静态形状
- 处理 broadcast、reshape、concat 等复杂形状变换
- 为后续内存估算和 tiling 提供准确的维度信息

动态 shape 场景下，此 Pass 仍会推断可知部分的形状（如 batch 维度已知的情况）。

### 2.5 `--struct-optimize`：结构优化

**实现**：`lib/Dialect/Top/Transforms/StructOptimize.cpp`

执行图结构级别的优化，包括：
- **算子融合**：Conv+BN、BN+Scale 等融合为单一 Op
- **常量折叠**：对所有输入均为常量的操作直接计算结果，替换为 WeightOp
- **冗余消除**：删除 Identity Reshape、无效 Transpose 等不改变语义的操作
- **代数化简**：利用交换律、结合律化简多操作序列

此 Pass 在 `ShapeInfer` 之后运行，以便利用已知形状信息做更精准的优化决策。

### 2.6 `--extra-optimize`：额外优化

**实现**：`lib/Dialect/Top/Transforms/ExtraOptimize.cpp`

补充 `StructOptimize` 未覆盖的优化场景：
- 进一步的算子融合（如 Attention 模式识别）
- 针对特定硬件的图变换（如某些芯片不支持的操作分解）
- LLM 模型相关的特殊优化（KV Cache 结构识别）

### 2.7 `--import-calibration-table`：导入校准表

**实现**：`lib/Dialect/Top/Transforms/ImportCalibrationTable.cpp`

量化流程的关键 Pass，读取 `run_calibration.py` 生成的校准表文件：
- 为每个操作的输出张量设置量化阈值（threshold）
- 支持对称量化（symmetric）和非对称量化（asymmetric）
- 校准表格式：`op_name  threshold  scale  zero_point`
- 阈值将在 `convert-top-to-tpu` 阶段被用于计算 scale/zero_point

### 2.8 `--convert-qdq-to-calibrated-dialect`：QDQ 转换

将包含 QuantizeLinear/DequantizeLinear 操作对（QDQ 格式）的模型转换为普通量化参数标注格式，使其与 `import-calibration-table` 后的 MLIR 格式统一，进入同一条 `convert-top-to-tpu` 转换路径。

### 2.9 `--processor-top-optimize`：处理器特定 Top 优化

**实现**：`lib/Dialect/Top/Transforms/ProcessorOptimize.cpp`

在方言转换前，针对特定芯片做一轮 Top Dialect 优化：
- 针对 CV18xx：某些操作需提前分解为 CV18xx 支持的子操作
- 针对 BM1684X/BM1690：利用更多硬件特性做针对性优化
- 目的是减少 TopToTpu 转换的复杂度

### 2.10 `--fuse-preprocess`：预处理融合

将图像预处理操作（归一化、色彩空间转换、均值减法等）融合进模型计算图的入口：
- 读取 `mean`、`scale`、`pixel_format`、`channel_format` 等参数
- 生成 `PreprocessOp` 替代独立的预处理步骤
- 融合后可在 TPU 上执行预处理，无需 CPU 介入

### 2.11 `--add-postprocess`：后处理添加

在模型输出端追加后处理操作（如 NMS、Decode Box 等），使推理引擎能直接输出最终检测结果，简化部署代码。

### 2.12 `--pruning`：剪枝

对 MatMul 权重做结构化剪枝，将稀疏权重压缩表示，减少权重存储量与计算量。通常与 `struct-optimize` 互斥，作为替代的第一优化步骤。

---

## 三、转换 Pass 详解

### 3.1 `--convert-top-to-tpu`：Top → Tpu 方言转换

**实现**：`lib/Conversion/TopToTpu/TopToTpuPass.cpp`（约 2000 行）

这是整个编译器最核心的 Pass，承担以下职责：

**量化参数注入**：
- 从操作的 `threshold` 属性计算 INT8 的 `scale` 和 `zero_point`
- 对激活函数、算术运算插入 `RequantInt`/`RequantFp` 操作
- 处理混合精度场景：权重 INT8/INT4，激活 FP16/BF16

**操作降级（Lowering）**：
- 按芯片选择对应的 Lowering 规则：`LoweringBM1684X.cpp`、`LoweringBM1684.cpp`、`LoweringCV18xx.cpp`
- 每条规则将一个 Top Op 替换为等价的 Tpu Op
- 处理类型转换：`top::ConvOp` → `tpu::Conv2DOp`（含量化参数）

**类型系统转换**：
- 激活值类型从浮点转换为整数（FP32 → INT8）
- 权重类型按量化模式转换（FP32 → INT8/INT4/FP16）

### 3.2 `--convert-top-to-tosa`：Top → TOSA 转换

将 Top Dialect 转换为 MLIR 标准的 TOSA（Tensor Operator Set Architecture）方言，用于非 SOPHGO 目标平台的代码生成路径。

### 3.3 `--convert-top-to-linalg`：Top → Linalg 转换

将 Top Dialect 转换为 Linalg 方言，对接 MLIR 的通用线性代数优化基础设施，用于实验性后端路径。

---

## 四、Tpu Dialect Pass 详解

Tpu Dialect 是 TPU-MLIR 的低层硬件特定表示，其 Pass 负责硬件优化、并行映射、内存规划和指令生成。

### 4.1 `--topo-sort`：拓扑排序

**实现**：`lib/Dialect/Tpu/Transforms/TopoSort.cpp`

在 LayerGroup 之前执行，对操作顺序做全局重排：
- 依据数据依赖关系进行拓扑排序
- **目标**：最小化跨 Group 的数据流，从而减少后续 LayerGroup 生成的 base_group 数量
- 将没有依赖关系的操作尽量集中，提高 LayerGroup 的融合机会

### 4.2 `--shape-optimize`：形状优化

**实现**：`lib/Dialect/Tpu/Transforms/ShapeOptimize.cpp`

处理动态 shape 场景下的形状相关优化：
- 消除不必要的 shape 计算操作
- 对已知部分静态形状进行提前折叠
- 为动态子网划分做准备

### 4.3 `--op-reorder`：操作重排序

**实现**：`lib/Dialect/Tpu/Transforms/OpReorder.cpp`

在 LayerGroup 前后都可能执行的轻量级重排：
- 将 WeightOp（常量权重）的位置调整到其使用者附近
- 减少权重在 LMEM 中的保留时间
- 降低同时驻留 LMEM 的张量峰值数量

### 4.4 `--weight-reorder`：权重重排序

**实现**：`lib/Dialect/Tpu/Transforms/WeightReorder.cpp`

将权重的内存布局从 NHWC/NCHW 转换为 TPU 硬件要求的格式：
- Conv 权重：转换为 `(OC, IC/NPU_NUM, kH, kW, NPU_NUM)` 格式，使每个 NPU Lane 连续访问属于自己的权重块
- Depthwise Conv 权重：特殊的通道对齐处理
- MatMul 权重：列优先/行优先格式选择
- INT4 权重：打包为双通道 INT8 格式

此 Pass 直接影响 LMEM 的 DMA 访问效率。

### 4.5 `--weight-fold`：权重折叠

**实现**：`lib/Dialect/Tpu/Transforms/WeightFold.cpp`

若某操作的**所有输入**均为 WeightOp（常量），则在编译期执行该操作，将结果折叠为新的 WeightOp：
- 减少运行时的不必要计算
- 常见于 Reshape、Transpose、Slice 等作用在纯权重上的操作

### 4.6 `--op-divide`：操作分割

**实现**：`lib/Dialect/Tpu/Transforms/OpDivide.cpp`

将超大全局操作（单次无法放入 LMEM）分割为多个较小操作：
- 对超大 MatMul（M 或 N 维度极大）按行/列分块
- 对超大 Concat 按输入数量分批
- 分割后的子操作可各自独立进行 LayerGroup 优化

### 4.7 `--subnet-divide`：子网划分

**实现**：`lib/Dialect/Tpu/Transforms/SubnetDivide.cpp`

将包含动态 shape 操作的计算图划分为静态子网和动态子网：
- **静态子网**：所有张量 shape 在编译期已知，走标准 LayerGroup 路径
- **动态子网**：包含可变 shape 操作（如变长序列 Attention），走动态编译路径
- 划分边界处插入子网间通信操作

### 4.8 `--layer-group`：层分组（核心优化）

**实现**：`lib/Dialect/Tpu/Transforms/LayerGroup.cpp` + `lib/Dialect/Tpu/Transforms/LayerGroup/`（18 个文件，约 12000 行）

LayerGroup 是 Tpu Dialect 中最重要的 Pass，详见独立分析文档。核心功能：
- 用动态规划算法将连续层划分为最优 Group
- 在 Group 内通过 LMEM tiling 消除 GMEM 访问
- 通过三阶段软件流水线实现计算与 DMA 的重叠
- 支持 `opt=1`（基础）和 `opt=2`（激进，含权重压缩）两种优化级别

### 4.9 `--dev-parallel`：多设备并行

**实现**：`lib/Dialect/Tpu/Transforms/DevParallel.cpp`

将模型分发到多个 TPU 设备（chips）上并行执行：
- 在 `num_device > 1` 时激活
- 对矩阵运算按输出维度分片，分配到不同设备
- 插入 `DevBegin`/`DevEnd` 控制操作标记设备边界
- 为跨设备张量插入 AllReduce/AllGather 同步操作

### 4.10 `--core-parallel`：多核并行

**实现**：`lib/Dialect/Tpu/Transforms/CoreParallel.cpp`

在单个 TPU 设备内利用多 NPU 核心并行：
- 在 `num_core > 1` 时激活（如 BM1688：2 核，BM1690：8 核）
- 对 LayerGroup 内的操作做核间切片分配
- 插入 `CoreBegin`/`CoreEnd`/`CoreSplit`/`CoreJoin` 控制原语
- 确保核间数据同步的正确性

### 4.11 `--address-assign`：内存地址分配

**实现**：`lib/Dialect/Tpu/Transforms/AddressAssign/`

为全局内存（GMEM）中所有张量分配偏移地址，支持四种策略：
- `basic`：顺序分配，不复用
- `io_alone`：输入/输出张量独立分配在专属 IO 区域
- `io_tag`：通过标签机制动态绑定 IO 地址（零拷贝接口）
- `io_reuse`：IO 张量地址允许与激活张量复用（极致内存压缩）

内部通过活跃区间分析（Live Range Analysis）和首次适配贪心算法实现内存复用。

### 4.12 `--processor-tpu-optimize`：处理器特定 Tpu 优化

在 Tpu Dialect 上针对具体芯片的优化：
- BM1684X：利用多核 NPU 特性做操作合并
- CV18xx：某些 Tpu Op 需替换为 CV18xx 专有实现
- BM1690：HBM 带宽感知的操作调度优化

### 4.13 `--opt-post-processor`：LayerGroup 后图优化

在 LayerGroup 完成、地址分配之前运行的轻量优化：
- 合并相邻的细粒度 Reshape/Permute 操作
- 对 LayerGroup 边界张量做额外的内存布局优化
- 为 AddressAssign 提供更规整的计算图输入

### 4.14 `--after-layergroup-weight-reorder`：LayerGroup 后权重重排

LayerGroup 可能改变某些权重的访问模式（如索引型操作），此 Pass 在 LayerGroup 后对这类权重做二次重排，确保 LMEM 访问的连续性。

### 4.15 `--strip-io-quant`：剥离 I/O 量化

对于 INT8 模型，在模型的输入端自动插入 FP32→INT8 转换，在输出端插入 INT8→FP32 转换：
- 使模型接口对外仍为浮点，内部为整数
- 可选择剥离（使用外部量化）或保留（内置量化转换）

### 4.16 `--codegen`：代码生成

**实现**：`lib/Dialect/Tpu/Transforms/Codegen/`

最终 Pass，遍历所有 Tpu Op，为每个 Op 调用 `GlobalGenInterface` 或 `LocalGenInterface` 生成硬件指令序列：
- BM168x 路径：生成 TIU（张量整数单元）和 TDMA 指令
- CV18xx 路径：生成 TIU 和 TDMA 指令（格式不同）
- 将指令序列、权重数据、模型元信息序列化为 FlatBuffers 格式的 bmodel 文件
- 支持嵌入调试信息（`embed_debug_info`）用于精度对比

### 4.17 辅助/调试 Pass

| Pass | 功能 |
|------|------|
| `--show-address` | 打印所有张量的 GMEM 地址分配结果 |
| `--net-statistic` | 统计模型的 FLOPs、参数量、内存占用 |
| `--trunc-io` | 按输入/输出名称截取模型子图（调试用） |
| `--trunc-layer` | 按层名截取模型子图（精度定位用） |
| `--cut-final-mlir` | 按配置文件切割最终 MLIR（分段部署） |
| `--time-fixed-subnet` | 按固定时间间隔划分子网 |

---

## 五、完整 Pass 流水线

```
model_transform.py（生成 Top MLIR）
    │
    ├── --init
    ├── --shape-infer
    ├── --struct-optimize（或 --pruning）
    ├── --canonicalize
    └── --extra-optimize
              ↓
         ${model}.mlir（Top Dialect）

model_deploy.py（编译为 bmodel）
    │
    阶段 1：Lowering
    ├── --processor-assign
    ├── --import-calibration-table（INT8 时）
    ├── --processor-top-optimize
    ├── --fuse-preprocess（需要时）
    ├── --convert-top-to-tpu         ← 核心转换
    ├── --canonicalize
    └── --weight-fold
              ↓
         ${model}_tpu.mlir（Tpu Dialect）

    阶段 2：Tpu 优化
    ├── --strip-io-quant
    ├── --processor-tpu-optimize
    ├── --dev-parallel（多卡时）
    ├── --weight-reorder
    ├── --op-divide（大模型时）
    ├── --subnet-divide（动态 shape 时）
    ├── --op-reorder
    ├── --topo-sort
    ├── --layer-group                ← 关键性能优化
    ├── --core-parallel（多核时）
    ├── --opt-post-processor
    ├── --after-layergroup-weight-reorder
    └── --address-assign
              ↓
         ${model}_final.mlir

    阶段 3：代码生成
    └── --codegen
              ↓
         ${model}.bmodel
```

---

## 六、统计汇总

| 类别 | Pass 数量 |
|------|----------|
| Top Dialect Pass | 12 |
| 跨方言转换 Pass | 3 |
| Tpu Dialect Pass | 21 |
| 实验性 Pass | 4 |
| **合计** | **40** |

核心性能 Pass（对最终执行效率影响最大）：`--layer-group`、`--weight-reorder`、`--core-parallel`、`--address-assign`

核心正确性 Pass（对量化精度影响最大）：`--import-calibration-table`、`--convert-top-to-tpu`、`--strip-io-quant`
