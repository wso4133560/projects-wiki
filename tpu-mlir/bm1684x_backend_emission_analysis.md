# BM1684X 后端指令发射详细分析

## 一、概述

BM1684X 后端是 TPU-MLIR 编译器的最终代码生成阶段，负责将 Tpu Dialect 的操作转换为 BM1684X 硬件可执行的 TIU（Tensor Integer Unit）和 GDMA（Global DMA）指令序列，并将这些指令序列化为 bmodel 二进制文件格式。

**核心组件**：
- **Codegen Pass**：遍历 MLIR 操作，调用代码生成接口
- **Backend API**：动态库接口，生成硬件指令
- **Instruction Buffer**：管理 TIU/GDMA 指令缓冲区
- **BModel Serializer**：将指令序列化为 FlatBuffers 格式

**关键文件**：
- `lib/Dialect/Tpu/Transforms/Codegen.cpp`：Codegen Pass 入口
- `lib/Dialect/Tpu/Transforms/Codegen/BM168xCodegen.cpp`（89KB）：主要代码生成逻辑
- `lib/Backend/BM168x/BM1684X.cpp`：BM1684X 后端实现
- `lib/Backend/BM168x/BM168x.cpp`（27KB）：后端基类

---

## 二、BM1684X 后端初始化

### 2.1 硬件参数配置

```cpp
// include/tpu_mlir/Backend/BM168x/BM1684X.h
class BM1684X : public BM168x {
protected:
  BM1684X() : BM168x(TypeID::get<BM1684X>()) {
    if (chip != module::Chip::BM1684X) return;
    
    code = std::make_unique<BM168x::Code>();
    
    // 硬件参数
    NPU_NUM = 64;                    // 64 个 NPU Lane
    EU_BYTES = 64;                   // 执行单元宽度 64 字节
    LMEM_BYTES = 1 << 18;            // 256KB 本地内存
    LMEM_BANKS = 16;                 // 16 个 Bank
    LMEM_BANK_BYTES = LMEM_BYTES / LMEM_BANKS;  // 16KB per Bank
    DMA_ALGN_BYTES = 512;            // DMA 对齐 512 字节
    IC_PARALLEL = 64;                // 输入通道并行度
    ALIGNMENT = 0x1000;              // 4KB 对齐
    
    // 内存地址空间
    GMEM_START_ADDR = 0x100000000ull;      // 全局内存起始地址（4GB）
    L2_SRAM_START_ADDR = 0x10000000ull;    // L2 SRAM 起始地址
    L2_SRAM_SIZE = 0x1FB000;               // L2 SRAM 大小（约 2MB）
    
    // 数据格式编码
    GDMA_VALUE_FORMAT_INT8 = 0;
    GDMA_VALUE_FORMAT_FLOAT16 = 1;
    GDMA_VALUE_FORMAT_FLOAT32 = 2;
    GDMA_VALUE_FORMAT_INT16 = 3;
    GDMA_VALUE_FORMAT_INT32 = 4;
    GDMA_VALUE_FORMAT_BFLOAT16 = 5;
    GDMA_VALUE_FORMAT_INT4 = 6;
  }
};
```

### 2.2 后端环境启动

```cpp
// lib/Backend/BM168x/BM1684X.cpp
void BM1684X::start_env() {
  BM168x::start_env();
  
  // 设置核心信息
  assert(core_num != 0);
  dl_backend_api_set_core_info(0, core_num);
  
  // 允许存储指令
  dl_allow_store_cmd();
  
  // 禁止原子 C 模型（加速编译）
  dl_forbid_atomic_cmodel();
  
  // 加载查找表（激活函数等）
  dl_load_lookup_tables();
}
```

### 2.3 指令缓冲区管理

```cpp
void BM1684X::before_codegen() {
  dl_allow_store_cmd();                      // 允许存储指令
  BM168x::before_codegen();
  dl_backend_api_clear_tpu_inst_data();      // 清空指令缓冲区
}

void BM1684X::after_codegen(int64_t flops) {
  BM168x::after_codegen(flops);
  dl_store_cmd_end();                        // 结束指令存储
  dl_forbid_store_cmd();                     // 锁定指令缓冲区
}
```

---

## 三、Codegen Pass 执行流程

### 3.1 Pass 入口

```cpp
// lib/Dialect/Tpu/Transforms/Codegen.cpp
class CodegenPass : public CodegenBase<CodegenPass> {
  void runOnOperation() override {
    auto mOp = getOperation();
    
    // 创建 BMCodegen 实例
    BMCodegen bm_codegen;
    bm_codegen.init(mOp, filename, bmodel_only);
    
    // 遍历所有模块
    auto modules = module::getAllModules();
    for (auto s : *modules) {
      bm_codegen.run(s, embed_debug_info, gdma_check);
    }
    
    // 存储 bmodel 文件
    bm_codegen.store();
  }
};
```

### 3.2 BMCodegen::run() 主循环

```cpp
// lib/Dialect/Tpu/Transforms/Codegen/BM168xCodegen.cpp
void BMCodegen::run(ModuleOp s, bool embed_debug_info, bool gdma_check) {
  // 1. 设置命令检查参数
  cmd_check_param_t check_param;
  check_param.gdma_check_en = gdma_check;
  bm168x->dl_set_cmd_check_param((void *)&check_param, gdma_check);
  
  // 2. 遍历所有子网（函数）
  s.walk<WalkOrder::PreOrder>([&](func::FuncOp func) {
    if (func == module::getMainFuncOp(s)) return;
    
    auto mode = getRunMode(func);
    
    switch (mode) {
      case RunMode::TPU_STATIC:
        // 静态编译模式
        auto subnet = CreateSubNet(s, call, run_core);
        break;
        
      case RunMode::TPU_DYNAMIC:
        // 动态编译模式
        auto subnet = CreateSubNet(s, call, subnet_ir_, context, run_core);
        break;
        
      case RunMode::CPU:
        // CPU 子网
        auto subnet = CreateCPUSubNet(s, call);
        break;
    }
  });
  
  // 3. 创建命令组
  auto cmd_group = builder.CreateVector(*cmd_group_all);
  npb.add_cmd_group(cmd_group);
}
```

### 3.3 CreateSubNet() 子网创建

```cpp
Offset<bmodel::SubNet> BMCodegen::CreateSubNet(ModuleOp s, func::CallOp call,
                                                int run_core) {
  // 1. 获取函数和输入输出
  auto func = module::getFuncOp(s, call.getCallee());
  
  // 2. 执行代码生成
  codegen(func, run_core);
  
  // 3. 创建命令组向量
  auto cmd_group = CreateCmdGroupVector();
  
  // 4. 创建输入输出张量描述
  auto input_tensor = CreateTensorVector(func, true);
  auto output_tensor = CreateTensorVector(func, false);
  
  // 5. 构建 SubNet FlatBuffer
  bmodel::SubNetBuilder snb(model_gen->Builder());
  snb.add_input_tensor(input_tensor);
  snb.add_output_tensor(output_tensor);
  snb.add_cmd_group(cmd_group);
  snb.add_subnet_mode(subnet_mode);
  
  return snb.Finish();
}
```

---

## 四、操作代码生成

### 4.1 GlobalGenInterface：全局操作

全局操作处理完整张量，不涉及 tiling。

#### Conv2D 全局代码生成

```cpp
// lib/Dialect/Tpu/Interfaces/BM1684X/Conv2D.cpp
void tpu::Conv2DOp::codegen_global_bm1684x() {
  auto attr = parseParam();
  
  // 1. 获取输入输出规格
  auto op = getOperation();
  auto input_spec = BM168x::get_input_spec(op);
  auto output_spec = BM168x::get_output_spec(op);
  
  // 2. 填充卷积参数结构
  conv_global_spec_t spec = {0};
  spec.common.input_c = attr.ic;
  spec.common.output_c = attr.oc;
  spec.common.groups = attr.groups;
  spec.common.kh = attr.kh;
  spec.common.kw = attr.kw;
  spec.common.dh = attr.dh;
  spec.common.dw = attr.dw;
  spec.common.stride_h = attr.sh;
  spec.common.stride_w = attr.sw;
  spec.common.pad_h_t = attr.pht;
  spec.common.pad_h_b = attr.phb;
  spec.common.pad_w_l = attr.pwl;
  spec.common.pad_w_r = attr.pwr;
  spec.common.has_bias = attr.has_bias;
  spec.common.if_relu = attr.do_relu;
  spec.common.relu_limit = attr.relu_limit;
  
  // 3. 量化参数
  if (module::isUniformQuantized(getInput(), getOutput())) {
    spec.rshift = module::getI64Array(getRshift())->at(0);
    spec.merge_coeff = getCoeffMerged();
  }
  
  // 4. 调用后端 API 生成 TIU 指令
  BM168x::call_global_func("backend_api_conv_global", &spec, sizeof(spec),
                           input_spec->data(), output_spec->data());
}
```

#### MatMul 全局代码生成

```cpp
// lib/Dialect/Tpu/Interfaces/BM1684X/MatMul.cpp
void tpu::MatMulOp::codegen_global_bm1684x() {
  auto p = parseParam();
  
  // 1. 填充 MatMul 参数
  batch_matmul_common_spec_t spec{0};
  spec.Y_dtype = BM168x::getDataType(getOutput());
  spec.L_trans = p.left_transpose;
  spec.R_trans = p.right_transpose;
  spec.hdim_is_batch = p.hdim_is_batch;
  spec.has_bias = p.with_bias;
  spec.if_relu = p.do_relu;
  spec.relu_limit = p.relu_limit;
  
  // 2. 量化参数
  if (module::isUniformQuantized(getOutput())) {
    auto rshift_v = module::getI64Array(getRshift());
    spec.rshift_num = rshift_v->size();
    for (int i = 0; i < spec.rshift_num; i++) {
      spec.rshift[i] = rshift_v->at(i);
    }
  }
  
  // 3. 调用后端 API
  auto op = getOperation();
  auto input_spec = BM168x::get_input_spec(op);
  auto output_spec = BM168x::get_output_spec(op);
  
  BM168x::call_global_func("backend_api_batch_matmul_global", &spec,
                           sizeof(spec), input_spec->data(), output_spec->data());
}
```

### 4.2 LocalGenInterface：局部操作

局部操作处理切片后的张量，用于 LayerGroup 优化。

#### Pool2D 局部代码生成

```cpp
// lib/Dialect/Tpu/Interfaces/BM1684X/Pooling2D.cpp
void tpu::Pool2DOp::codegen_local_bm1684x_kernel(
    std::vector<group_info_t> &in_group_infos,
    std::vector<group_info_t> &out_group_infos,
    local_sec_info_t &sec_info,
    std::shared_ptr<std::vector<tensor_spec_t>> input_spec,
    std::shared_ptr<std::vector<tensor_spec_t>> output_spec) {
  
  auto attr = parseParam();
  auto gi = out_group_infos[0];
  
  // 1. 填充 Pooling 局部参数
  pooling_local_spec_t spec = {0};
  spec.buffer_addr = gi.buffer_addr;
  
  // 2. Padding 处理（仅在边界切片）
  spec.common.pad_h_t = (sec_info.h_idx == 0 ? attr.pad_h : 0);
  spec.common.pad_h_b = (sec_info.out_h_idx + sec_info.out_h_slice == attr.oh 
                         ? attr.pad_h_after : 0);
  spec.common.pad_w_l = (sec_info.w_idx == 0 ? attr.pad_w : 0);
  spec.common.pad_w_r = (sec_info.out_w_idx + sec_info.out_w_slice == attr.ow 
                         ? attr.pad_w_after : 0);
  
  // 3. 其他参数
  spec.common.kh = attr.kh;
  spec.common.kw = attr.kw;
  spec.common.stride_h = attr.sh;
  spec.common.stride_w = attr.sw;
  spec.common.is_avg_pooling = (attr.pool_mode == tpu::PoolMode::Avg);
  
  // 4. 调用后端 API 生成局部指令
  BM168x::call_local_func("backend_api_pooling_local", &spec, sizeof(spec),
                          &sec_info, input_spec->data(), output_spec->data());
}
```

---

## 五、后端 API 调用机制

### 5.1 函数指针类型定义

```cpp
// lib/Backend/BM168x/BM168x.cpp

// 全局 API 签名
typedef int (*global_backend_api_t)(
    void *params,      // 操作参数结构体
    int param_size,    // 参数大小
    void *input,       // 输入张量规格数组
    void *output,      // 输出张量规格数组
    void *pid_node     // 命令 ID 节点（用于周期计算）
);

// 局部 API 签名
typedef int (*local_backend_api_t)(
    void *params,      // 操作参数结构体
    int param_size,    // 参数大小
    void *input,       // 输入张量规格数组
    void *info,        // 切片信息（local_sec_info_t）
    void *output,      // 输出张量规格数组
    void *pid_node     // 命令 ID 节点
);
```

### 5.2 API 调用包装函数

```cpp
void BM168x::call_global_func(const char *symbolName, void *params,
                              int param_size, void *input, void *output) {
  // 1. 从动态库获取函数指针
  auto func = instance()->CastToFPtr<global_backend_api_t>(symbolName);
  
  // 2. 调用后端函数，传入 cmdid_node 用于指令追踪
  func(params, param_size, input, output, (*instance())->cmdid_node);
}

void BM168x::call_local_func(const char *symbolName, void *params,
                             int param_size, void *info, void *input,
                             void *output) {
  // 1. 从动态库获取函数指针
  auto func = instance()->CastToFPtr<local_backend_api_t>(symbolName);
  
  // 2. 调用后端函数，传入 bdc_node 用于局部指令追踪
  func(params, param_size, info, input, output, (*instance())->bdc_node);
}
```

### 5.3 命令 ID 节点管理

```cpp
// 分割命令 ID 节点（用于并行）
void BM168x::divide_sync_id() {
  dl_cmd_id_divide(code->cmdid_node, code->bdc_node, code->gdma_node);
}

// 合并命令 ID 节点（用于同步）
void BM168x::merge_sync_id() {
  dl_cmd_id_merge(code->cmdid_node, code->bdc_node, code->gdma_node);
}
```

---

## 六、GroupOp 代码生成

GroupOp 是 LayerGroup 优化后的操作组，需要按时间步（timestep）迭代生成指令。

### 6.1 codegen_for_group() 函数

```cpp
// lib/Dialect/Tpu/Transforms/Codegen/BM168xCodegen.cpp
void BMCodegen::codegen_for_group(GroupOp gOp, Operation *prev_op, Operation *next_op) {
  // 1. 解析 group 结构
  auto body = gOp.getBody();
  auto timestep_table = /* 从 flow 属性恢复 */;
  int64_t timestep_num = timestep_table.size();
  
  // 2. 构建操作 ID 到操作的映射
  std::vector<Operation *> group_ops(max_id + 1);
  body.
walk([&](Operation *op) {
    if (auto lgOp = dyn_cast<LocalGenInterface>(op)) {
      auto ginfo = lgOp.getGroupInfo(0, 0, 0, 0, 0);
      group_ops[ginfo.id] = op;
    }
  });
  
  // 3. 获取切片配置
  auto shape_secs = module::getI64Array(gOp.getShapeSecs());
  int64_t nstep = shape_secs->at(0);
  int64_t cstep = shape_secs->at(1);
  int64_t hstep = shape_secs->at(3);
  int64_t dstep = shape_secs->at(2);
  int64_t wstep = shape_secs->at(4);
  
  // 4. 遍历时间步
  for (int64_t ts = 0; ts < timestep_num; ++ts) {
    bm168x->divide_sync_id();  // 分割命令 ID 节点
    
    // 5. 遍历该时间步的所有操作
    for (auto id : timestep_table[ts]) {
      auto op = group_ops[id];
      auto lgOp = cast<LocalGenInterfaceDecorator>(op);
      
      // 6. 设置命令 ID 前缀（用于调试）
      auto pid_node = (CMD_ID_NODE *)(*BM168x::instance())->bdc_node;
      if (isa<LoadOp, StoreOp>(op)) {
        pid_node = (CMD_ID_NODE *)(*BM168x::instance())->gdma_node;
      }
      BM168x::instance()->dl_set_cmd_id_prefix(pid_node, gen_op_id(op).c_str());
      
      // 7. 生成局部指令
      lgOp.codegen_local_bm168x(nstep, cstep, hstep, dstep, wstep,
                                group_type, sec_info);
    }
    
    bm168x->merge_sync_id();  // 合并命令 ID 节点
  }
}
```

---

## 七、指令缓冲区提取与序列化

### 7.1 CreateCmdGroupVector() 函数

```cpp
std::shared_ptr<std::vector<Offset<bmodel::CmdGroup>>>
BMCodegen::CreateCmdGroupVector() {
  auto cmd_group_v = std::make_shared<std::vector<Offset<bmodel::CmdGroup>>>();
  
  // 1. 从后端获取指令缓冲区指针
  auto gdma_ptr = (uint8_t *)bm168x->get_inst_data("gdma:0:0");
  auto bdc_ptr = (uint8_t *)bm168x->get_inst_data("tiu:0:0");
  
  int bdc_offset = 0, gdma_offset = 0;
  
  // 2. 遍历每个指令组
  for (int group_idx = 0; group_idx < bm168x->get_group_number(); group_idx++) {
    // 3. 获取指令数量
    auto bdc_num = bm168x->get_inst_number_per_group("tiu:0:0", group_idx);
    auto gdma_num = bm168x->get_inst_number_per_group("gdma:0:0", group_idx);
    
    // 4. 获取指令长度
    auto bdc_len = bm168x->get_bdc_len(bdc_num, group_idx);
    auto gdma_len = bm168x->get_gdma_len(gdma_num, group_idx);
    
    // 5. 创建 TIU 指令二进制
    bmodel::Binary binary_bdc;
    if (bdc_num != 0) {
      binary_bdc = model_gen->WriteBinary(bdc_len, bdc_ptr + bdc_offset);
      bdc_offset += bdc_len;
    }
    
    // 6. 创建 GDMA 指令二进制
    bmodel::Binary binary_gdma;
    std::shared_ptr<std::vector<bmodel::RelEntry>> cmd_reloc_entries;
    if (gdma_num != 0) {
      binary_gdma = model_gen->WriteBinary(gdma_len, gdma_ptr + gdma_offset);
      
      // 7. 创建重定位条目（IO_RELOC 模式）
      cmd_reloc_entries = CreateGdmaRelEntryVector(gdma_num, gdma_ptr + gdma_offset);
      gdma_offset += gdma_len;
    }
    
    // 8. 构建 CmdGroup FlatBuffer
    bmodel::CmdGroupBuilder cgb(model_gen->Builder());
    cgb.add_bdc_num(bdc_num);
    cgb.add_gdma_num(gdma_num);
    cgb.add_bdc_cmd_byte(bdc_len);
    cgb.add_gdma_cmd_byte(gdma_len);
    
    if (bdc_num != 0) {
      cgb.add_binary_bdc(&binary_bdc);
    }
    if (gdma_num != 0) {
      cgb.add_binary_gdma(&binary_gdma);
      if (cmd_reloc_entries && !cmd_reloc_entries->empty()) {
        auto reloc_entries_v = model_gen->Builder().CreateVector(*cmd_reloc_entries);
        cgb.add_reloc_entries(reloc_entries_v);
      }
    }
    
    cmd_group_v->push_back(cgb.Finish());
  }
  
  return cmd_group_v;
}
```

### 7.2 GDMA 指令重定位

```cpp
std::shared_ptr<std::vector<bmodel::RelEntry>>
BMCodegen::CreateGdmaRelEntryVector(u32 cmd_num, u8 *cmd_buffer) {
  if (!module::isAddrMode(module::AddrMode::IO_RELOC))
    return nullptr;
  
  auto reloc_entries_v = std::make_shared<std::vector<bmodel::RelEntry>>();
  
  // GDMA 指令长度（BM1684X）
  auto get_gdma_cmd_len = []() -> u32 { return 96; };
  
  u32 cmd_cur = 0;
  for (u32 cmd_idx = 0; cmd_idx < cmd_num - 1; cmd_idx++) {
    u32 cmd_bytes = get_gdma_cmd_len();
    u32 *cmd = reinterpret_cast<u32 *>(cmd_buffer + cmd_cur);
    
    // 提取源地址（cmd[16-17]）
    u64 src_addr = (u64)(cmd[17] & 0xff) << 32 | ((u64)cmd[16]);
    try_add_reloc_entry(reloc_entries_v, src_addr, cmd_cur + 64);
    
    // 提取目标地址（cmd[18-19]）
    u64 dst_addr = (u64)(cmd[19] & 0xff) << 32 | ((u64)cmd[18]);
    try_add_reloc_entry(reloc_entries_v, dst_addr, cmd_cur + 72);
    
    // 特定命令类型的索引地址（cmd[20-21]）
    u32 cmd_type = cmd[1] & 0xf;
    if (cmd_type == 2 || cmd_type == 7 || cmd_type == 8) {
      u64 index_addr = ((u64)(cmd[21] & 0xff) << 32 | ((u64)cmd[20]));
      try_add_reloc_entry(reloc_entries_v, index_addr, cmd_cur + 80);
    }
    
    cmd_cur += cmd_bytes;
  }
  
  return reloc_entries_v;
}
```

---

## 八、周期计算

### 8.1 周期查询接口

```cpp
// lib/Backend/BM168x/BM168x.cpp

int64_t BM168x::get_gdma_cycle() {
  return dl_get_cmd_id_cycle(code->gdma_node);
}

int64_t BM168x::get_bdc_cycle() {
  return dl_get_cmd_id_cycle(code->bdc_node);
}

int64_t BM168x::get_cmd_cycle() {
  return dl_get_cmd_id_cycle(code->cmdid_node);
}
```

### 8.2 周期信息嵌入

周期信息在后端 API 调用时自动嵌入到命令 ID 节点中：
- **cmdid_node**：全局操作的周期
- **bdc_node**：TIU 指令的周期
- **gdma_node**：GDMA 指令的周期

这些周期信息用于 LayerGroup 的 shape_secs 搜索，选择最优的切片配置。

---

## 九、BModel 文件格式

### 9.1 文件头结构

```cpp
typedef struct {
  uint32_t magic;              // 0xFF55AAEE（魔数）
  uint32_t header_size;        // 头部大小
  uint32_t flatbuffers_size;   // FlatBuffers 元数据大小
  uint64_t binary_size;        // 二进制指令数据大小
  uint32_t reserved[11];       // 保留字段
} MODEL_HEADER_T;
```

### 9.2 内存信息结构

```cpp
typedef struct {
  uint64_t bd_cmd_mem_size;      // TIU 指令总大小
  uint64_t gdma_cmd_mem_size;    // GDMA 指令总大小
  uint64_t hau_cmd_mem_size;     // HAU 指令总大小（硬件加速单元）
  uint64_t sdma_cmd_mem_size;    // SDMA 指令总大小（系统 DMA）
  uint64_t dynamic_ir_mem_size;  // 动态 IR 总大小
  uint64_t neuron_mem_size;      // 神经元内存总大小
  uint64_t coeff_mem_size;       // 系数内存总大小
  uint64_t middle_buffer_size;   // 最大输入/输出缓冲区大小
  uint64_t host_neuron_mem_size; // CPU 层 IO 在主机上的大小
  uint64_t host_coeff_mem_size;  // CPU 层系数在主机上的大小
} bmodel_mem_info_t;
```

### 9.3 文件布局

```
+---------------------------+
| MODEL_HEADER_T            | 64 字节
+---------------------------+
| FlatBuffers Metadata      | flatbuffers_size 字节
|  - Net 信息               |
|  - SubNet 信息            |
|  - Tensor 描述            |
|  - CmdGroup 描述          |
+---------------------------+
| Binary Data               | binary_size 字节
|  - TIU 指令二进制         |
|  - GDMA 指令二进制        |
|  - 权重数据               |
+---------------------------+
```

---

## 十、关键数据结构

### 10.1 tensor_spec_t

```cpp
typedef struct {
  u64 addr;                    // 内存地址
  u32 dtype;                   // 数据类型
  u32 shape[MAX_SHAPE_DIMS];   // 张量形状
  u32 stride[MAX_SHAPE_DIMS];  // 内存步幅
  u32 dims;                    // 维度数量
  bool is_const;               // 是否为常量
} tensor_spec_t;
```

### 10.2 local_sec_info_t

```cpp
typedef struct {
  int32_t n_slice, c_slice, d_slice, h_slice, w_slice;  // 切片大小
  int32_t n_idx, c_idx, d_idx, h_idx, w_idx;            // 切片索引
  int32_t out_n_slice, out_c_slice, out_h_slice, out_w_slice;  // 输出切片
  int32_t out_n_idx, out_c_idx, out_h_idx, out_w_idx;          // 输出索引
  bool is_h_split, is_w_split, is_c_split;              // 分割标志
  group_type_t group_type;                               // 分组类型
} local_sec_info_t;
```

### 10.3 group_info_t

```cpp
typedef struct {
  int64_t id;                  // 操作 ID
  int64_t stage;               // 流水线阶段
  int64_t out_addr;            // 输出地址
  int64_t buffer_addr;         // 缓冲区地址
  int64_t n_slice, c_slice, d_slice, h_slice, w_slice;  // 切片大小
  int64_t n_idx, c_idx, d_idx, h_idx, w_idx;            // 切片索引
  bool overstepped;            // 是否越界
} group_info_t;
```

---

## 十一、完整指令发射流程图

```
ModuleOp (MLIR)
    │
    ├─ CodegenPass::runOnOperation()
    │
    ├─ BMCodegen::init()
    │  ├─ 加载后端动态库（libbackend_1684x.so）
    │  ├─ 初始化 ModelGen（FlatBuffers）
    │  └─ 调用 BM1684X::start_env()
    │
    ├─ BMCodegen::run()
    │  ├─ 遍历所有子网（函数）
    │  │
    │  ├─ For each subnet:
    │  │  ├─ BM1684X::before_codegen()
    │  │  │  └─ 清空指令缓冲区
    │  │  │
    │  │  ├─ codegen(func)
    │  │  │  ├─ 遍历所有操作
    │  │  │  │
    │  │  │  ├─ For GlobalGenInterface ops:
    │  │  │  │  └─ codegen_global_bm1684x()
    │  │  │  │     └─ BM168x::call_global_func("backend_api_*")
    │  │  │  │        └─ 后端库生成 TIU 指令 → cmdid_node
    │  │  │  │
    │  │  │  └─ For GroupOp:
    │  │  │     └─ codegen_for_group()
    │  │  │        ├─ 遍历时间步
    │  │  │        ├─ divide_sync_id()
    │  │  │        ├─ For each op in timestep:
    │  │  │        │  └─ codegen_local_bm1684x_kernel()
    │  │  │        │     └─ BM168x::call_local_func("backend_api_*_local")
    │  │  │        │        └─ 后端库生成 GDMA + TIU 指令 → bdc_node, gdma_node
    │  │  │        └─ merge_sync_id()
    │  │  │
    │  │  ├─ BM1684X::after_codegen()
    │  │  │  └─ 锁定指令缓冲区
    │  │  │
    │  │  └─ CreateCmdGroupVector()
    │  │     ├─ get_inst_data("tiu:0:0") → 提取 TIU 指令
    │  │     ├─ get_inst_data("gdma:0:0") → 提取 GDMA 指令
    │  │     ├─ get_inst_number_per_group() → 获取指令数量
    │  │     ├─ get_bdc_len() / get_gdma_len() → 获取指令大小
    │  │     ├─ CreateGdmaRelEntryVector() → 解析 GDMA 重定位
    │  │     └─ CmdGroupBuilder → 构建 FlatBuffers CmdGroup
    │  │
    │  └─ 创建 SubNet FlatBuffer
    │
    └─ BMCodegen::store()
       ├─ 写入 MODEL_HEADER_T
       ├─ 写入 FlatBuffers 元数据
       └─ 写入二进制指令数据
              ↓
         ${model}.bmodel
```

---

## 十二、GDMA 指令格式（BM1684X）

### 12.1 指令结构（96 字节）

```
Offset  | Size | Field
--------|------|------------------
0-3     | 4B   | 命令头
4-7     | 4B   | 命令类型（bits 0-3）
8-63    | 56B  | 命令参数
64-71   | 8B   | 源地址（cmd[16-17]）
72-79   | 8B   | 目标地址（cmd[18-19]）
80-87   | 8B   | 索引地址（cmd[20-21]，特定类型）
88-95   | 8B   | 附加参数
```

### 12.2 命令类型

- **Type 0**：普通传输
- **Type 2**：带索引的传输（需要索引地址）
- **Type 7**：Gather 操作（需要索引地址）
- **Type 8**：Scatter 操作（需要索引地址）

---

## 十三、关键函数索引

| 函数 | 文件 | 行号 | 功能 |
|------|------|------|------|
| `CodegenPass::runOnOperation()` | Codegen.cpp | 19-57 | Pass 入口 |
| `BMCodegen::run()` | BM168xCodegen.cpp | 127-318 | 主循环 |
| `BMCodegen::codegen()` | BM168xCodegen.cpp | 1493-1666 | 操作遍历 |
| `BMCodegen::codegen_for_group()` | BM168xCodegen.cpp | 733-970 | Group 代码生成 |
| `BM168x::call_global_func()` | BM168x.cpp | 395-399 | 全局 API 调用 |
| `BM168x::call_local_func()` | BM168x.cpp | 403-408 | 局部 API 调用 |
| `BMCodegen::CreateCmdGroupVector()` | BM168xCodegen.cpp | 527-577 | 指令提取 |
| `BMCodegen::CreateGdmaRelEntryVector()` | BM168xCodegen.cpp | 579-621 | GDMA 重定位 |
| `BM1684X::before_codegen()` | BM1684X.cpp | 50-54 | 代码生成前准备 |
| `BM1684X::after_codegen()` | BM1684X.cpp | 56-60 | 代码生成后清理 |

---

## 十四、总结

BM1684X 后端的指令发射机制展示了一个清晰的分层架构：

1. **MLIR 层**：定义操作语义和接口（GlobalGenInterface/LocalGenInterface）
2. **Codegen 层**：遍历操作，调用后端 API，管理指令缓冲区
3. **Backend 层**：动态库接口，生成硬件指令，计算周期
4. **Serialization 层**：提取指令，序列化为 FlatBuffers 格式

这种设计实现了编译器前端与硬件后端的解耦，使得 TPU-MLIR 能够灵活支持多种芯片架构（BM1684/BM1684X/BM1688/BM1690/CV18xx），同时保持统一的编译流程。
