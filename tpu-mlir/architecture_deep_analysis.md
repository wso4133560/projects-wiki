# TPU-MLIR 架构深度分析文档

## Context

TPU-MLIR 是 SOPHGO（算能）开源的、基于 MLIR 框架构建的深度学习编译器工具链。它的核心任务是：将来自多种深度学习框架的预训练模型，编译为可在 SOPHGO 系列 TPU 硬件上高效执行的二进制文件（bmodel）。本文档对该项目的完整架构、核心组件、编译流程、方言设计、后端支持进行系统性的深度分析。

---

## 一、项目总体概览

### 1.1 项目定位

TPU-MLIR 定位于**企业级深度学习推理编译器**，服务于从云端高性能推理到边缘端嵌入式部署的全场景需求。其核心价值在于：

- **统一前端**：通过标准化的 MLIR 中间表示，屏蔽不同训练框架（ONNX、PyTorch、TFLite、Caffe）的差异
- **多目标后端**：支持 15+ 种 SOPHGO 自研 TPU 芯片（BM168x 系列、CV18xx 系列、SG 系列）
- **全面量化**：内置从 F32 到 INT4 的完整量化链路，包含自动混合精度搜索
- **LLM 支持**：专门适配大语言模型（Qwen2、Llama 等）的编译与部署路径

### 1.2 顶层目录结构

```
tpu-mlir/
├── include/          C++ 公共头文件（方言定义、接口、工具）
├── lib/              C++ 核心库实现（方言、转换、后端）
├── tools/            命令行可执行工具（tpuc-opt、model_tool 等）
├── python/           Python 工具链（转换、校准、推理、分析）
├── third_party/      外部依赖（MLIR/LLVM、oneDNN、or-tools 等）
├── test/             功能测试与回归测试
├── unittests/        单元测试
├── regression/       端到端回归测试
├── docs/             文档
├── bindings/         Python/C API 绑定
├── capi/             C 语言 API
├── experimental/     实验性功能
├── docker/           容器化支持
└── CMakeLists.txt    构建系统入口
```

### 1.3 核心技术栈

| 层次 | 技术 | 用途 |
|------|------|------|
| 框架层 | MLIR/LLVM | 中间表示与 Pass 管理框架 |
| 图优化层 | TableGen + C++ | 方言定义与变换规则 |
| 量化层 | Python + NumPy | 校准、阈值搜索、混合精度 |
| 后端层 | C++ + 硬件库 | 指令生成与内核调用 |
| 序列化层 | FlatBuffers | bmodel 二进制格式 |
| 工具层 | Python | 端到端工作流脚本 |

---

## 二、MLIR 方言系统设计

TPU-MLIR 的核心架构是**两层方言设计**：高层 `top` 方言和低层 `tpu` 方言，分别对应硬件无关的图表示和硬件特定的操作描述。

### 2.1 Top 方言（硬件无关高层表示）

**命名空间**：`::tpu_mlir::top`
**文件位置**：`include/tpu_mlir/Dialect/Top/IR/TopOps.td`（6533 行）

#### 2.1.1 设计原则

Top 方言对标 ONNX 的抽象层次，定义了 188 个硬件无关的神经网络操作。每个操作继承 `Top_Op` 基类，自动获得三个关键接口的方法声明：

```
Top_Op<mnemonic, traits>
  ├─ InferenceInterface   → init/deinit/inference 方法
  ├─ FlopsInterface       → 计算 FLOPs
  └─ ShapeInterface       → 形状推断
```

#### 2.1.2 操作分类

| 类别 | 代表性操作 | 数量 |
|------|-----------|------|
| 基础/控制 | None, Weight, Input, Tuple, UnTuple | 8 |
| 卷积/池化 | Conv, Deconv, AvgPool, MaxPool, Depth2Space | ~15 |
| 激活函数 | Relu, Sigmoid, Tanh, SiLU, GELU, Mish, Swish | ~20 |
| 算术运算 | Add, Sub, Mul, Div, Max, Min 及 Const 变体 | ~20 |
| 张量变换 | Reshape, Permute, Flatten, Slice, Concat, Tile | ~20 |
| 降维/归约 | Squeeze, Unsqueeze, ReduceMean/Sum/Max/Min/L2 | ~12 |
| 量化操作 | RequantInt, RequantFp, DequantInt, QuantizeLinear | ~12 |
| 序列/注意力 | GRU, LSTM, Attention, MatMul, Einsum | ~10 |
| 高级专用 | SelectiveScan, MatchTemplate, NonZero, RoiPool | ~15 |
| 类型转换 | Cast, DtypeCast, Scale | ~8 |

#### 2.1.3 规范化（Canonicalize）

`lib/Dialect/Top/Canonicalize/` 下为 60+ 个操作各自定义了规范化规则，执行如下优化：
- **常量折叠**：将操作数全为常量的操作直接计算结果
- **操作融合**：将 BN+Scale、Conv+BN 等融合为单一操作
- **冗余消除**：消除 Identity Reshape、无效 Transpose 等
- **形状传播**：利用已知形状简化图结构

### 2.2 Tpu 方言（硬件特定低层表示）

**命名空间**：`::tpu_mlir::tpu`
**文件位置**：`include/tpu_mlir/Dialect/Tpu/IR/TpuOps.td`（7887 行）

#### 2.2.1 设计原则

Tpu 方言的操作直接对应 SOPHGO TPU 的硬件指令语义，定义了 212 个操作。每个操作通过 `Tpu_Op` 基类自动获得代码生成接口：

```
Tpu_Op<mnemonic, traits>
  ├─ TpuTypeRestrict          → 类型检查约束
  ├─ GlobalGenInterface       → 全局内存代码生成
  ├─ InferenceInterface       → C 模型推理执行
  └─ DynGlobalGenInterface    → 动态 shape 代码生成
```

具备本地内存优化的操作还额外实现：
- `LocalGenInterface`：本地内存（LMEM）执行代码生成
- `DynLocalGenInterface`：动态 shape 下的本地内存生成

#### 2.2.2 核心操作分类

**计算操作**：
- 卷积族：`Conv2D`, `Conv3D`, `Deconv`, `Deconv3D`
- 池化族：`Pool1D`, `Pool2D`, `Pool3D`, `MaxPoolWithMask`
- 矩阵运算：`MatMul`, `A16MatMul`, `Mlp`, `Attention`, `FAttention`
- 量化族：`RequantInt`, `RequantIntAxis`, `RequantFp`, `RequantFpAxis`, `DequantInt`

**内存操作**（反映硬件内存层级）：
- `Buffer`：声明全局或本地内存缓冲区
- `Load`/`Store`：GMEM↔LMEM 的 DMA 传输
- `LoadToL2M`：加载到 L2 缓存

**并行控制操作**（反映 TPU 的并行层级）：
- 核心级：`CoreBegin`/`CoreEnd`, `CoreParallel`, `CoreSplit`, `CoreJoin`
- 设备级：`DevBegin`/`DevEnd`, `DevParallelOp`
- 分组级：`GroupOp`, `GroupParallelOp`

**专用操作**：
- 图像处理：`Yuv2rgbFormulaOp`, `Mmap2RgbmapOp`, `PreprocessOp`, `MeanStdScaleOp`
- 专用算子：`SelectiveScanOp`（Mamba 架构）, `CorrelationOp`

---

## 三、接口与特征系统

### 3.1 接口系统（Interfaces）

`include/tpu_mlir/Interfaces/` 定义了 10 个关键 MLIR 接口，构成操作能力的契约：

| 接口 | 适用方言 | 核心方法 | 用途 |
|------|---------|---------|------|
| `InferenceInterface` | Top + Tpu | init/inference/deinit | C 模型推理仿真 |
| `FlopsInterface` | Top | getFLOPs() | 计算量统计 |
| `ShapeInterface` | Top | inferShape() | 形状推导 |
| `GlobalGenInterface` | Tpu | codegen_global_bm1684x/bm1684/cv18xx | 全局内存代码生成 |
| `LocalGenInterface` | Tpu | LocalGenSupport/assign_sec_info | 本地内存优化 |
| `DynGlobalGenInterface` | Tpu | dyn_codegen_global_bm1684x | 动态 shape 支持 |
| `DynLocalGenInterface` | Tpu | dyn_codegen_local_bm1684x | 动态 shape 本地内存 |
| `TypeInterface` | Tpu | getQuantType()/setQuantType() | 量化类型管理 |
| `IndexingMapsInterface` | Tpu | getIndexingMaps() | Linalg 索引映射 |
| `InplaceInterface` | Tpu | isInplaceOp() | 原地操作标记 |

`GlobalGenInterface` 是代码生成的关键枢纽，它为每个操作分别提供面向 BM1684、BM1684X 和 CV18xx 的代码生成实现，使得同一 Tpu 方言的操作能透明地映射到不同硬件。

### 3.2 特征系统（Traits）

`include/tpu_mlir/Traits/Traits.td` 定义了 11 个操作特征，描述操作的静态语义性质：

| 特征 | 含义 | 优化价值 |
|------|------|---------|
| `SupportFuseRelu` | 可与后续 ReLU 融合 | 减少一次激活的内存访问 |
| `SupportConstant` | 支持常量输入折叠 | 编译时预计算 |
| `SupportPermuteMove` | 支持 Permute 穿越移动 | 消除冗余转置 |
| `SupportElementwise` | 逐元素操作 | 支持 broadcasting 优化 |
| `SupportEarlyStride` | 支持早期步长 | 减少中间张量大小 |
| `ShapeProducer/Consumer` | 参与动态 shape 传播 | 动态 shape 正确性保障 |
| `ScalarProducer/Consumer` | 产生/消费标量值 | 标量优化路径 |
| `TpuTypeRestrict` | 类型转换有限制 | 量化安全性保障 |
| `InOutSameShape` | 输入输出形状相同 | shape 推导加速 |

---

## 四、完整编译流程

TPU-MLIR 的端到端编译分为**两个主要阶段**，每个阶段由独立的 Python 脚本驱动。

### 4.1 阶段一：模型转换（model_transform.py）

**目的**：将各种框架的原始模型转换为 Top Dialect MLIR

```
输入模型 (ONNX/TFLite/PyTorch/Caffe/TpuLang)
    │
    ▼
[前端转换器 - python/transform/]
    ├─ OnnxConverter.py          → onnx → top
    ├─ TFLiteConverter.py        → tflite → top
    ├─ TorchConverter.py         → pytorch → top
    ├─ CaffeConverter.py         → caffe → top
    └─ TpuLangConverter.py       → custom → top
    │
    ▼ (via MLIRImporter)
[TOP Dialect MLIR] + 权重 NPZ 文件
    │
    ▼
[tpuc-opt: Top 优化 Pass 序列]
    ├─ Init               → 模块初始化，注入芯片目标信息
    ├─ ShapeInfer         → 静态形状推导
    ├─ StructOptimize     → 图结构优化（融合、消除）
    ├─ ExtraOptimize      → 额外优化（算子融合等）
    └─ FusePreprocess     → 将预处理操作融合进图
    │
    ▼
${model_name}.mlir  （Top Dialect 最终形态）
```

**支持的前端格式**：

| 格式 | 典型场景 | 核心文件大小 |
|------|---------|------------|
| ONNX | 通用场景，最广泛支持 | OnnxConverter.py ~164KB |
| PyTorch | torch.export 直接导出 | TorchConverter.py |
| TFLite | 移动端/边缘端模型 | TFLiteConverter.py |
| Caffe | 经典 CV 模型 | CaffeConverter.py |
| TpuLang | 自定义 Op 扩展 | TpuLangConverter.py |

### 4.2 阶段二：模型部署（model_deploy.py）

**目的**：将 Top Dialect MLIR 编译为目标芯片的 bmodel 二进制

```
${model_name}.mlir (Top Dialect)
    │
    ▼
[TopToTpu 转换 Pass - lib/Conversion/TopToTpu/]
    ├─ TopToTpuPass.cpp       → 主转换逻辑（~2000行）
    ├─ TopLowering.cpp        → 量化参数处理
    ├─ LoweringBM1684X.cpp    → BM1684X 特定降级规则
    ├─ LoweringBM1684.cpp     → BM1684 特定降级规则
    └─ LoweringCV18xx.cpp     → CV18xx 特定降级规则
    │
    ▼ （量化参数注入、类型转换）
${model_name}_tpu.mlir (Tpu Dialect)
    │
    ▼
[Tpu 优化 Pass 序列 - lib/Dialect/Tpu/Transforms/]
    ├─ TopoSort           → 确保拓扑执行顺序
    ├─ ShapeOptimize      → 动态 shape 优化
    ├─ OpReorder          → 操作重排（减少内存占用）
    ├─ WeightReorder      → 权重内存布局重排（硬件友好格式）
    ├─ SubnetDivide       → 动态 shape 子网分割
    ├─ DevParallel        → 多设备任务分配
    ├─ CoreParallel       → 多核任务分配
    │
    ├─ LayerGroup         → 层组划分（核心优化，见下节）
    │
    ├─ AddressAssign      → 全局/本地内存地址分配
    │   ├─ basic          → 基础分配
    │   ├─ io_alone       → IO 独立分配
    │   ├─ io_tag         → IO 标签分配
    │   └─ io_reuse       → IO 内存复用
    │
    └─ Codegen            → 硬件指令代码生成
    │
    ▼
${model_name}.bmodel  （FlatBuffers 格式二进制）
```

### 4.3 层组优化（LayerGroup）—— 核心优化 Pass

LayerGroup 是 TPU-MLIR 最关键的优化，负责解决 TPU 本地内存（LMEM）容量有限与网络层间大张量传输的矛盾。

```
LayerGroup 内部流程（lib/Dialect/Tpu/Transforms/LayerGroup/）
    │
    ├─ LayerGroupUtil.cpp (~3000行)
    │   → 构建数据流依赖图
    │   → 分析算子的内存访问模式
    │
    ├─ GroupMethod.cpp (~3500行)
    │   → 动态规划：求解最优分组方案
    │   → 目标：最大化 LMEM 利用率，最小化 GMEM↔LMEM 搬运次数
    │
    ├─ LmemAllocator.cpp (~2500行)
    │   → 组内 LMEM 动态分配
    │   → 张量生命周期管理（活跃区间分析）
    │
    ├─ CycleCalculator.cpp (~2000行)
    │   → 计算每个算子的执行周期
    │   → 识别计算/DMA 瓶颈
    │
    └─ BasicTimeStep.cpp (~1000行)
        → 生成双缓冲（double-buffering）时间步调度
        → 实现计算与 DMA 的流水并行
```

**核心参数**（通过 `--opt` 控制优化级别）：
- `opt=1`：基础层组，按依赖关系分组
- `opt=2`：启用 LMEM 复用与循环切分
- `opt=3`：最激进优化，启用 compress_weight

---

## 五、量化流程

### 5.1 量化校准（run_calibration.py）

```
未量化 MLIR + 校准数据集
    │
    ▼
[ShapeOps 识别] → shape-only 层保持浮点
    │
    ▼
[MatchPattern] → 识别 Transformer 块，特殊处理 Attention/FFN
    │
    ▼
[激活值统计] - ActivationCalibrator
    ├─ 前向推理收集各层激活分布
    └─ 直方图统计
    │
    ▼
[阈值搜索] - SearchThreshold
    ├─ kl (KL 散度) ← 默认，精度最优
    ├─ mse (均方误差)
    ├─ max (最大值，最激进)
    ├─ percentile (百分位数)
    └─ aciq_laplace/gauss (ACIQ 算法)
    │
    ▼
[混合精度搜索] - SearchQtable（可选）
    ├─ 按敏感度自动选择层的精度
    ├─ wi8ai8_fp：权重8位激活8位混合浮点
    ├─ wi4ai4_wi8ai8：4/8混合整数
    └─ wf8af8_fp：8位浮点混合
    │
    ▼
[SmoothQuant 后处理]（可选）
    → 平衡权重和激活的量化难度
    │
    ▼
calibration_table  （每层阈值/scale/zero_point）
```

### 5.2 支持的量化模式

| 量化模式 | 精度 | 典型场景 |
|---------|------|---------|
| F32 | 最高 | 调试、精度敏感 |
| F16 | 高 | BM168x GPU-like 场景 |
| BF16 | 高（训练友好） | LLM 推理 |
| INT8 | 中（主流） | 需要校准表 |
| INT4 | 低（极致压缩） | 超大模型 |
| W8F16/W8BF16 | 权重压缩 | LLM 量化 |
| W4F16/W4BF16 | 极致权重压缩 | 超大 LLM |
| F8E4M3/F8E5M2 | 8位浮点 | 新兴量化格式 |
| *DYN 系列 | 动态 shape | 变长序列模型 |

---

## 六、硬件后端架构

### 6.1 支持的硬件目标

TPU-MLIR 以单例模式管理当前目标架构（`lib/Backend/Arch.cpp`），支持以下平台：

#### BM168x 系列（SOPHGO 高性能计算）

| 芯片 | 特点 | 内存 |
|------|------|------|
| BM1684 | 第一代，成熟稳定 | 12GB LPDDR4 |
| BM1684X | 增强版，多核 NPU | 16GB LPDDR4 |
| BM1688 | 高集成，A2 架构 | - |
| BM1690 | 新一代旗舰 | 大容量 HBM |
| BM1690E | 嵌入式增强版 | - |

#### CV18xx 系列（SOPHGO 边缘嵌入式）

| 芯片 | 定位 |
|------|------|
| CV186X | CV 系列高端（兼容 BM1688） |
| CV184X | CV 系列中端 |
| CV183X/CV182X/CV181X/CV180X | 轻量级边缘推理 |

#### 其他系列

| 芯片 | 说明 |
|------|------|
| SG2380 | SOPHGO SG 系列 |
| SGTPUV8 | VPU 变体 |
| SG2262 | 商用方案（兼容 BM1690） |

### 6.2 后端架构关键参数

每个硬件目标在 `Arch.h` 中定义以下静态参数，LayerGroup 和 AddressAssign 据此进行内存规划：

```cpp
static int64_t NPU_NUM;           // NPU 执行核心数
static int64_t EU_BYTES;          // 执行单元宽度（字节）
static int64_t LMEM_BYTES;        // 本地内存总量
static int64_t LMEM_BANKS;        // LMEM 分组数（用于双缓冲）
static int64_t LMEM_BANK_BYTES;   // 每组大小
static int64_t DMA_ALGN_BYTES;    // DMA 对齐要求（通常 512B）
static bool    ALIGN_4N;          // 是否要求 4N 对齐
```

### 6.3 代码生成层次

```
Tpu Dialect (IR)
    │
    ├─ [BM168x 路径]
    │   ├─ codegen_global_bm1684x() → 调用 BM168x 运行时 API
    │   ├─ codegen_local_bm1684x()  → 生成 LMEM 操作序列
    │   └─ lib/Backend/BM168x/      → 底层 API 包装
    │
    ├─ [CV18xx 路径]
    │   ├─ codegen_global_cv18xx()  → 调用 CV18xx 内核
    │   ├─ codegen_local_cv18xx()   → 生成 CV18xx 本地操作
    │   └─ lib/Backend/CV18xx/Kernel/  → 专用算子内核（最大 930KB+）
    │
    └─ [CPU 仿真路径]
        └─ InferenceInterface → DNNL/oneDNN 参考实现
```

---

## 七、工具链与 Python 生态

### 7.1 核心命令行工具

| 工具 | 位置 | 功能 |
|------|------|------|
| `tpuc-opt` | `tools/tpuc-opt/` | MLIR 优化器，支持所有 Pass 的独立调用 |
| `model_tool` | `tools/model_tool/` | bmodel 查看、修改、合并、切分 |
| `tpuc-tool` | `tools/tpuc-tool/` | TPU 工具集（推理、调试） |
| `chiprunner` | `tools/chiprunner/` | 芯片运行时（链接实际 TPU） |
| `cvimodel_debug` | `tools/cvimodel_debug/` | CV18xx 模型调试 |

### 7.2 Python 工具矩阵（python/tools/，50+ 脚本）

**编译链核心**：
- `model_transform.py`：前端转换入口
- `model_deploy.py`（35KB）：后端编译入口，支持全部量化选项
- `run_calibration.py`：INT8 量化校准
- `llm_convert.py`：LLM 专用编译路径（支持 KV Cache、动态 shape）

**调试与分析**：
- `model_runner.py`（28KB）：跨平台模型推理（C模型/芯片）
- `bmodel_dis.py`：bmodel 反汇编分析
- `compare_visualizer.py`（41KB）：层间精度对比可视化
- `op_locator.py`：量化精度问题定位
- `gen_layer_group_config.py`：手动配置层组策略

**性能分析**：
- `run_bmprofile.py`：芯片性能剖析
- `pmu_dump.py`：性能监测单元数据采集
- `PerfAI/`：AI 辅助性能分析模块

### 7.3 Python 转换器架构

```python
BaseConverter (python/transform/BaseConverter.py)
    ├─ 管理模型名称、输入形状、预处理参数
    ├─ 维护权重字典 WeightToNpz()
    ├─ 持有 MLIRImporter 实例（生成 Top 操作）
    │
    ├─ OnnxConverter      → op_factory: 150+ ONNX 算子映射
    ├─ TFLiteConverter    → TFLite 算子映射
    ├─ TorchConverter     → PyTorch aten 算子映射
    │   └─ MaskRCNNConverter（继承，添加 RoI 相关处理）
    ├─ CaffeConverter     → Caffe Layer 映射
    └─ TpuLangConverter   → TpuLang 自定义算子
```

---

## 八、关键数据格式

### 8.1 bmodel 格式（FlatBuffers）

```
bmodel 文件结构：
├─ Header（魔数 + 版本 + 芯片类型）
├─ Net（网络描述）
│   ├─ 子网列表（静态/动态子网）
│   ├─ 输入/输出描述
│   └─ 参数描述
├─ Kernel（硬件指令序列）
│   └─ 各子网的 GDMA/TIU 指令
└─ Params（权重数据）
    └─ 量化后的权重（INT8/INT4/FP16 等）
```

### 8.2 校准表格式

```
calibration_table 文本格式：
# op_name    threshold    scale    zero_point
input        0.99834      ...      ...
conv1/Conv   1.25431      ...      ...
...
```

### 8.3 编译中间产物

```
${prefix}_top.mlir           → Top Dialect（前端输出，调试用）
${prefix}_tpu.mlir           → Tpu Dialect（量化后，优化前）
${prefix}_tpu_opt.mlir       → 优化后 Tpu Dialect
${prefix}_final.mlir         → 代码生成后最终 MLIR
${prefix}.bmodel             → 可部署二进制
tensor_location.json         → 张量-内存地址映射（调试工具用）
*.layer_group_cache.json     → 层组结果缓存（加速重编译）
```

---

## 九、可扩展性设计

### 9.1 添加新硬件后端

需要完成以下步骤：
1. 在 `lib/Backend/` 新建芯片目录，实现继承 `Arch` 的芯片类
2. 在 `include/tpu_mlir/Interfaces/` 的 `GlobalGenInterface` 中添加新方法声明
3. 为每个 Tpu 操作实现新芯片的代码生成方法
4. 在 `lib/Dialect/Tpu/Transforms/LayerGroup/` 注册新芯片内存参数
5. 在 `lib/Backend/Arch.cpp` 的 `Arch::init()` 中注册芯片

### 9.2 添加新前端格式

只需在 `python/transform/` 新建继承 `BaseConverter` 的转换器类，实现 `op_factory` 字典（算子映射表）即可接入完整的 Top → Tpu → bmodel 编译链。

### 9.3 添加新自定义操作（TpuLang）

通过 `python/samples/tpulang/` 示例，用户可以：
1. 定义 Python 层的自定义算子接口
2. 在 C++ 层注册对应的 Top/Tpu 操作
3. 提供硬件后端的代码生成实现

---

## 十、架构总结

### 10.1 整体架构分层图

```
┌─────────────────────────────────────────────────────────────┐
│                     用户工作流层                              │
│   model_transform.py  model_deploy.py  run_calibration.py   │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                  前端转换层（Python）                          │
│  OnnxConverter / TFLiteConverter / TorchConverter / Caffe   │
│                     MLIRImporter                             │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│              Top Dialect（高层硬件无关 IR）                   │
│          188 个操作 + Canonicalize + Top Transforms          │
│    InferenceInterface / FlopsInterface / ShapeInterface      │
└──────────────────────────┬──────────────────────────────────┘
                           │ TopToTpu Conversion Pass
                           │ （量化参数注入 + 芯片特定降级规则）
┌──────────────────────────▼──────────────────────────────────┐
│              Tpu Dialect（低层硬件特定 IR）                   │
│          212 个操作 + 并行控制原语 + 内存操作                  │
│   GlobalGenInterface / LocalGenInterface / TypeInterface     │
└──────────────────────────┬──────────────────────────────────┘
                           │ Tpu 优化 Pass 序列
                           │ LayerGroup → AddressAssign → Codegen
┌──────────────────────────▼──────────────────────────────────┐
│                   硬件后端层（C++）                            │
│   BM1684 │ BM1684X │ BM1688 │ BM1690 │ CV18xx │ SG2380     │
│              Kernel 实现 / 指令序列生成                        │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                  bmodel 二进制输出                             │
│            FlatBuffers 格式 + 权重 + 硬件指令序列              │
└─────────────────────────────────────────────────────────────┘
```

### 10.2 核心设计特点

1. **两层方言分离**：Top（抽象/框架无关）→ Tpu（具体/硬件特定），通过 `TopToTpu` Pass 桥接，保证前后端解耦
2. **接口驱动**：所有跨模块能力通过 MLIR 接口协议表达，新增功能无需修改核心框架
3. **特征辅助优化**：静态特征（Traits）为编译器提供无需分析即可使用的语义信息，提高优化速度
4. **LayerGroup 是性能关键**：通过最优化 LMEM 利用率和 DMA/计算流水，实现接近理论峰值的 TPU 利用率
5. **量化链路完整**：从训练后量化（PTQ）的 KL 校准，到自动混合精度搜索，再到 QAT 部署，覆盖全场景
6. **LLM 路径专门化**：`llm_convert.py` 针对大模型的动态 shape、KV Cache、多设备分布做了专项优化
7. **多芯片统一管理**：通过 `Arch` 单例和 `GlobalGenInterface` 多分发，15+ 芯片共用同一套 MLIR IR

### 10.3 代码规模

| 模块 | 语言 | 规模 |
|------|------|------|
| Tpu Dialect 定义 | TableGen | 7887 行 |
| Top Dialect 定义 | TableGen | 6533 行 |
| CV18xx 内核库 | C++ | 930KB+ |
| Python 工具 | Python | 50+ 脚本，200+ 模块文件 |
| LayerGroup 优化 | C++ | 18 个文件，~12000 行 |
| 前端转换器 | Python | 100KB+ |

---

## 验证方法

本分析文档基于对以下关键文件的直接阅读，无需执行代码即可验证：

**关键文件路径**（可直接 `cat` 验证）：
- `include/tpu_mlir/Dialect/Top/IR/TopOps.td` — Top 方言操作定义
- `include/tpu_mlir/Dialect/Tpu/IR/TpuOps.td` — Tpu 方言操作定义
- `include/tpu_mlir/Interfaces/*.td` — 接口定义
- `include/tpu_mlir/Traits/Traits.td` — 特征定义
- `lib/Backend/Arch.cpp` — 硬件初始化逻辑
- `lib/Conversion/TopToTpu/TopToTpuPass.cpp` — 核心转换 Pass
- `python/tools/model_deploy.py` — 部署工具（量化参数）
- `python/transform/OnnxConverter.py` — ONNX 转换器

**端到端功能验证**（参考 README.md）：
```bash
# 1. 转换 ONNX 模型
model_transform.py --model_name yolov5s \
  --model_def yolov5s.onnx \
  --input_shapes [[1,3,640,640]] \
  --mlir yolov5s.mlir

# 2. INT8 量化校准
run_calibration.py yolov5s.mlir \
  --dataset ./COCO2017 --input_num 100 \
  -o yolov5s_cali_table

# 3. 编译为 bmodel
model_deploy.py --mlir yolov5s.mlir \
  --quantize INT8 \
  --calibration_table yolov5s_cali_table \
  --chip bm1684x \
  --model yolov5s_int8.bmodel
```
