# Triton 编译器后端代码生成详解

本文档详细分析 Triton 编译器的后端代码生成流程,包括 NVIDIA 和 AMD 两个后端的完整编译管道。

---

## 一、概述

Triton 编译器采用多阶段降级策略,将高层 Python 代码逐步转换为 GPU 可执行的二进制代码:

```
Python 代码
  ↓ AST 解析
Triton IR (TTIR)
  ↓ GPU 特定转换
TritonGPU IR (TTGIR)
  ↓ LLVM 转换
LLVM IR (LLIR)
  ↓ 汇编生成
PTX / AMDGCN
  ↓ 汇编器
CUBIN / HSACO
```

**关键文件**:
- `python/triton/backends/compiler.py` - 后端抽象接口
- `third_party/nvidia/backend/compiler.py` - NVIDIA 后端实现
- `third_party/amd/backend/compiler.py` - AMD 后端实现
- `python/triton/compiler/compiler.py` - 编译流程编排
- `python/src/llvm.cc` - LLVM IR 到汇编的转换

---

## 二、NVIDIA 后端编译流程

### 2.1 编译阶段定义

**文件**: `third_party/nvidia/backend/compiler.py`

NVIDIA 后端定义了 5 个编译阶段:

| 阶段 | 名称 | 输入 | 输出 | 说明 |
|------|------|------|------|------|
| 1 | ttir | Python AST | Triton IR | 前端语义分析 |
| 2 | ttgir | Triton IR | TritonGPU IR | GPU 特定优化 |
| 3 | llir | TritonGPU IR | LLVM IR | 资源分配和转换 |
| 4 | ptx | LLVM IR | PTX 汇编 | LLVM 后端生成 |
| 5 | cubin | PTX 汇编 | CUDA 二进制 | ptxas 汇编 |

### 2.2 make_ttir - 生成 Triton IR

**职责**: 从 Python AST 生成初始的 Triton IR

**关键 Pass**:
```python
def make_ttir(mod, metadata, options):
    pm = ir.pass_manager(mod.context)
    pm.enable_debug()

    # 1. 内联优化
    passes.common.add_inliner(pm)

    # 2. 重写张量指针
    passes.ttir.add_rewrite_tensor_pointer(pm, options.arch)

    # 3. 对于 SM < 90,将张量描述符转换为指针
    if options.arch < 90:
        passes.ttir.add_convert_to_ttgpuir(pm)

    # 4. 规范化和优化
    passes.common.add_canonicalizer(pm)
    passes.common.add_cse(pm)
    passes.common.add_licm(pm)
    passes.common.add_symbol_dce(pm)

    pm.run(mod)
    return mod
```

### 2.3 make_ttgir - 生成 TritonGPU IR

**职责**: 转换为 GPU 特定 IR 并应用优化

**关键参数**:
- `num_warps`: warp 数量
- `num_ctas`: CTA (Cooperative Thread Array) 数量
- `num_stages`: 流水线阶段数
- `compute_capability`: GPU 计算能力 (如 90 表示 SM90/Hopper)

**优化 Pass 流程**:

```python
def make_ttgir(mod, metadata, options, cluster_info):
    pm = ir.pass_manager(mod.context)

    # 1. 转换为 TritonGPU IR
    passes.ttir.add_convert_to_ttgpuir(
        pm,
        target=options.arch,
        num_warps=options.num_warps,
        threads_per_warp=32,
        num_ctas=options.num_ctas
    )

    # 2. 内存合并优化
    passes.ttgpuir.add_coalesce(pm)

    # 3. 矩阵乘法加速
    passes.ttgpuir.add_accelerate_matmul(pm)

    # 4. 移除布局转换
    passes.ttgpuir.add_remove_layout_conversions(pm)

    # 5. 优化 dot 操作数
    passes.ttgpuir.add_optimize_dot_operands(pm)

    # 6. Hopper (SM90+) 特定优化
    if options.arch >= 90:
        # Warp 特化
        passes.ttgpuir.add_automatic_warp_specialization(pm)

        # 软件流水线
        passes.ttgpuir.add_assign_latencies(pm)
        passes.ttgpuir.add_schedule_loops(pm)
        passes.ttgpuir.add_pipeline(pm, num_stages=options.num_stages)

    # 7. Blackwell (SM100+) 特定优化
    if options.arch >= 100:
        # TMEM 分配
        passes.ttgpuir.add_allocate_tensor_memory(pm)
        passes.ttgpuir.add_optimize_tensor_memory(pm)

    # 8. 通用优化
    passes.common.add_cse(pm)
    passes.common.add_symbol_dce(pm)

    pm.run(mod)
    return mod
```

### 2.4 make_llir - 生成 LLVM IR

**职责**: 转换为 LLVM IR 并分配资源

**关键步骤**:

```python
def make_llir(src, metadata, options, cluster_info):
    mod = src.parse()
    pm = ir.pass_manager(mod.context)

    # 1. 分配共享内存
    passes.convert.add_allocate_shared_memory_nv(pm)

    # 2. 分配张量内存 (Blackwell)
    if options.arch >= 100:
        passes.convert.add_allocate_tensor_memory(pm)

    # 3. 转换为 LLVM IR
    passes.convert.add_to_llvmir(pm, options.arch)

    # 4. 规范化
    passes.common.add_canonicalizer(pm)
    passes.common.add_cse(pm)
    passes.common.add_symbol_dce(pm)

    pm.run(mod)

    # 5. 转换为 LLVM Module
    llvm_mod = llvm.to_module(mod, metadata)

    # 6. LLVM 优化 (O3)
    llvm.optimize_module(llvm_mod, llvm.OPTIMIZE_O3)

    # 7. 提取元数据
    metadata["shared"] = mod.get_int_attr("triton_gpu.shared")
    metadata["num_regs"] = mod.get_int_attr("triton_gpu.num-regs")
    metadata["global_scratch_size"] = mod.get_int_attr("triton_gpu.global_scratch_size")

    return llvm_mod
```

**LLVM 优化 Pass** (O3 级别):
- 函数内联
- 常量传播和折叠
- 死代码消除
- 循环优化 (展开、向量化)
- 指令选择和调度
- 寄存器分配

### 2.5 make_ptx - 生成 PTX 汇编

**职责**: 使用 LLVM 后端生成 PTX 汇编代码

**实现**:

```python
def make_ptx(src, metadata, options, cluster_info):
    llvm_mod = src.parse()

    # 1. 确定 PTX 版本
    ptx_version = get_ptx_version(options.arch)

    # 2. 转换为 PTX
    ptx_code = llvm.translate_to_asm(
        llvm_mod,
        triple='nvptx64-nvidia-cuda',
        proc=f'sm_{options.arch}',
        features=f'+ptx{ptx_version}',
        flags=["nvptx-mad-wide-opt"],
        enable_fp_fusion=options.enable_fp_fusion
    )

    # 3. 后处理: 更新 PTX 版本和目标架构
    ptx_code = re.sub(r'\.version \d+\.\d+', f'.version {ptx_version}', ptx_code)
    ptx_code = re.sub(r'\.target sm_\d+', f'.target sm_{options.arch}', ptx_code)

    return ptx_code
```

**PTX 版本映射**:
- SM 90 (Hopper) → PTX 8.0+
- SM 100 (Blackwell) → PTX 9.0+

**关键 LLVM 参数**:
- `triple`: 目标三元组,指定 NVPTX 后端
- `proc`: 处理器类型,如 `sm_90a`
- `features`: PTX 特性,如 `+ptx90`
- `flags`: 编译标志,如 `nvptx-mad-wide-opt` (融合乘加优化)

### 2.6 make_cubin - 生成 CUDA 二进制

**职责**: 使用 ptxas 汇编器将 PTX 编译为 CUBIN

**实现**:

```python
def make_cubin(src, metadata, options, cluster_info):
    ptx_code = src.parse()

    # 1. 构建 ptxas 命令
    ptxas_args = [
        'ptxas',
        f'--gpu-name=sm_{options.arch}',
        '--output-file', cubin_path,
        ptx_path
    ]

    # 2. 添加调试选项
    if options.debug:
        ptxas_args.append('-lineinfo')

    # 3. 添加优化选项
    if options.opt_level == 0:
        ptxas_args.append('--opt-level 0')

    # 4. 寄存器分配优化
    ptxas_args.append('--regAllocOptLevel=2')

    # 5. 禁用融合乘加 (如果需要)
    if not options.enable_fp_fusion:
        ptxas_args.append('--fmad=false')

    # 6. 执行 ptxas
    result = subprocess.run(ptxas_args, capture_output=True)

    if result.returncode != 0:
        raise RuntimeError(f"ptxas failed: {result.stderr.decode()}")

    # 7. 读取 CUBIN
    with open(cubin_path, 'rb') as f:
        cubin_data = f.read()

    return cubin_data
```

**ptxas 选项说明**:
- `--gpu-name`: 目标 GPU 架构
- `--lineinfo`: 生成行号信息 (调试)
- `--opt-level`: 优化级别 (0-3)
- `--regAllocOptLevel`: 寄存器分配优化级别
- `--fmad`: 是否启用融合乘加

**CUBIN 格式**:
- ELF 格式的二进制文件
- 包含 SASS (Streaming ASSembler) 机器码
- 包含符号表和调试信息
- 可使用 `cuobjdump` 或 `nvdisasm` 反汇编

---

## 三、AMD 后端编译流程

### 3.1 编译阶段定义

**文件**: `third_party/amd/backend/compiler.py`

AMD 后端定义了 5 个编译阶段:

| 阶段 | 名称 | 输入 | 输出 | 说明 |
|------|------|------|------|------|
| 1 | ttir | Python AST | Triton IR | 前端语义分析 |
| 2 | ttgir | Triton IR | TritonGPU IR | GPU 特定优化 |
| 3 | llir | TritonGPU IR | LLVM IR | 资源分配和转换 |
| 4 | amdgcn | LLVM IR | AMDGCN 汇编 | LLVM 后端生成 |
| 5 | hsaco | AMDGCN 汇编 | HSA Code Object | ROCm 汇编 |

### 3.2 make_ttgir - AMD 特定优化

**关键差异**:

```python
def make_ttgir(mod, metadata, options, cluster_info):
    pm = ir.pass_manager(mod.context)

    # 1. 转换为 TritonGPU IR
    passes.ttir.add_convert_to_ttgpuir(
        pm,
        target=options.arch,
        num_warps=options.num_warps,
        threads_per_warp=options.warp_size,  # AMD: 32 或 64
        num_ctas=options.num_ctas
    )

    # 2. AMD 特定的矩阵乘法加速
    passes.ttgpuir.add_accelerate_matmul(
        pm,
        arch=options.arch,
        matrix_instr_nonkdim=options.matrix_instr_nonkdim,
        kpack=options.kpack
    )

    # 3. 异步拷贝优化 (gfx950, gfx1250)
    if options.arch in ['gfx950', 'gfx1250']:
        passes.ttgpuir.add_optimize_async_copy(pm)

    # 4. Block pingpong 调度 (gfx942, gfx950)
    if options.arch in ['gfx942', 'gfx950']:
        passes.ttgpuir.add_block_pingpong_schedule(pm)

    # 5. Buffer 操作转换
    passes.ttgpuir.add_convert_to_buffer_ops(pm)

    pm.run(mod)
    return mod
```

**AMD 特定参数**:
- `warp_size`: 32 (RDNA) 或 64 (CDNA)
- `matrix_instr_nonkdim`: MFMA/WMMA 指令的非 K 维度
- `kpack`: K 维度打包因子

### 3.3 make_llir - AMD 资源分配

**关键步骤**:

```python
def make_llir(src, metadata, options, cluster_info):
    mod = src.parse()
    pm = ir.pass_manager(mod.context)

    # 1. 分配共享内存 (LDS)
    passes.convert.add_allocate_shared_memory_amd(pm)

    # 2. 转换为 LLVM IR
    passes.convert.add_to_llvmir(pm, options.arch)

    pm.run(mod)

    # 3. 转换为 LLVM Module
    llvm_mod = llvm.to_module(mod, metadata)

    # 4. 设置 AMD 控制常量
    set_amd_control_constants(llvm_mod, options)

    # 5. 链接外部库
    link_amd_libraries(llvm_mod, options.arch)

    # 6. LLVM 优化
    llvm.optimize_module(
        llvm_mod,
        llvm.OPTIMIZE_O3,
        options.arch,
        '',
        [],
        options.enable_fp_fusion
    )

    # 7. 设置内核属性
    set_kernel_attributes(llvm_mod, options, cluster_info)

    return llvm_mod
```

**AMD 控制常量**:
```cpp
@__oclc_finite_only_opt = constant i8 0
@__oclc_correctly_rounded_sqrt32 = constant i8 1
@__oclc_wavefrontsize64 = constant i8 1  // 或 0 (取决于 warp_size)
```

**链接的外部库**:
- `ocml.bc`: OpenCL 数学库
- `ockl.bc`: OpenCL 内核库

**内核属性**:
```llvm
!{!"amdgpu-cluster-dims", i32 %cluster_x, i32 %cluster_y, i32 %cluster_z}
!{!"amdgpu-flat-work-group-size", i32 %min, i32 %max}
!{!"amdgpu-waves-per-eu", i32 %min, i32 %max}
```

### 3.4 make_amdgcn - 生成 AMDGCN 汇编

**实现**:

```python
def make_amdgcn(src, metadata, options, cluster_info):
    llvm_mod = src.parse()

    # 1. 确定特性
    features = []
    if options.arch.startswith('gfx11'):
        features.append('-real-true16')

    # 2. 转换为 AMDGCN
    asm_code = llvm.translate_to_asm(
        llvm_mod,
        triple='amdgcn-amd-amdhsa',
        proc=options.arch,
        features=','.join(features),
        flags=[],
        enable_fp_fusion=options.enable_fp_fusion
    )

    # 3. 可选: MIR 交换路径 (调度优化)
    if options.use_mir_exchange:
        asm_code = mir_exchange_optimize(asm_code, options)

    return asm_code
```

### 3.5 make_hsaco - 生成 HSA Code Object

**实现**:

```python
def make_hsaco(src, metadata, options, cluster_info):
    asm_code = src.parse()

    # 1. 汇编 AMDGCN
    obj_data = amd.assemble_amdgcn(
        asm_code,
        options.arch
    )

    # 2. 链接生成 HSACO
    hsaco_data = amd.link_hsaco(
        obj_data,
        options.arch
    )

    return hsaco_data
```

**HSACO 格式**:
- HSA Code Object v3/v4 格式
- 包含 GCN/RDNA 机器码
- 包含元数据和符号表
- 可使用 `llvm-objdump` 查看

---

## 四、编译流程编排

### 4.1 compile 函数

**文件**: `python/triton/compiler/compiler.py`

**完整流程**:

```python
def compile(src, target, options):
    # 1. 选择后端
    backend = get_backend(target)

    # 2. 计算缓存键
    cache_key = compute_cache_key(src, backend, options)

    # 3. 检查缓存
    if cache_manager.has(cache_key):
        return cache_manager.get(cache_key)

    # 4. 创建 IR 上下文
    context = ir.context()
    backend.load_dialects(context)

    # 5. 生成初始 IR
    ir_module = src.make_ir(context, options)

    # 6. 获取编译阶段
    stages = {}
    backend.add_stages(stages, options)

    # 7. 顺序执行各阶段
    for stage_name in ['ttir', 'ttgir', 'llir', 'ptx/amdgcn', 'cubin/hsaco']:
        if stage_name in stages:
            stage_func = stages[stage_name]
            ir_module = stage_func(ir_module, metadata, options)

            # 缓存中间结果
            cache_manager.put(f"{cache_key}_{stage_name}", ir_module)

    # 8. 生成元数据
    metadata = generate_metadata(ir_module, options)

    # 9. 创建 CompiledKernel
    kernel = CompiledKernel(
        binary=ir_module,
        metadata=metadata,
        target=target
    )

    # 10. 缓存最终结果
    cache_manager.put(cache_key, kernel)

    return kernel
```

### 4.2 缓存机制

**缓存键组成**:

```python
def compute_cache_key(src, backend, options):
    key_parts = [
        triton_key(),           # Triton 版本 + 源文件哈希
        src.hash(),             # 源代码哈希
        backend.hash(),         # 后端哈希
        options.hash(),         # 编译选项哈希
        env_vars_hash()         # 环境变量哈希
    ]

    key = '-'.join(key_parts)
    return hashlib.sha256(key.encode()).hexdigest()
```

**triton_key() 包含**:
- Triton 版本号
- 所有 Python 源文件的 SHA256 哈希
- `libtriton.so` 共享库的哈希
- 语言模块的哈希

**缓存目录结构**:
```
~/.triton/cache/
└── {hash}/
    ├── ttir
    ├── ttgir
    ├── llir
    ├── ptx (或 amdgcn)
    ├── cubin (或 hsaco)
    └── metadata.json
```

**元数据内容**:
```json
{
  "hash": "abc123...",
  "target": {"backend": "cuda", "arch": 90},
  "num_warps": 4,
  "num_ctas": 1,
  "shared": 49152,
  "num_regs": 128,
  "triton_version": "3.0.0",
  "env_vars": {...}
}
```

---

## 五、LLVM IR 到汇编的转换

### 5.1 translateLLVMIRToASM 函数

**文件**: `python/src/llvm.cc`

**实现**:

```cpp
std::string translateLLVMIRToASM(
    llvm::Module *module,
    const std::string &triple,
    const std::string &proc,
    const std::string &features,
    const std::vector<std::string> &flags,
    bool enable_fp_fusion
) {
    // 1. 内联所有函数
    llvm::legacy::PassManager pm;
    pm.add(llvm::createAlwaysInlinerLegacyPass());
    pm.run(*module);

    // 2. 创建 TargetMachine
    llvm::TargetOptions options;
    options.AllowFPOpFusion = enable_fp_fusion
        ? llvm::FPOpFusion::Fast
        : llvm::FPOpFusion::Strict;

    auto target_machine = llvm::EngineBuilder()
        .setTargetOptions(options)
        .selectTarget(
            llvm::Triple(triple),
            "",
            proc,
            llvm::SmallVector<std::string>(features)
        );

    // 3. 设置数据布局
    module->setDataLayout(target_machine->createDataLayout());
    module->setTargetTriple(triple);

    // 4. 生成汇编
    llvm::SmallString<0> asm_buffer;
    llvm::raw_svector_ostream asm_stream(asm_buffer);

    llvm::legacy::PassManager codegen_pm;
    target_machine->addPassesToEmitFile(
        codegen_pm,
        asm_stream,
        nullptr,
        llvm::CodeGenFileType::AssemblyFile
    );

    codegen_pm.run(*module);

    return asm_stream.str().str();
}
```

**关键步骤**:
1. **内联**: 将所有函数调用内联,简化代码生成
2. **TargetMachine**: 创建目标机器,配置处理器和特性
3. **数据布局**: 设置目标平台的数据布局
4. **代码生成**: 运行 LLVM 后端生成汇编代码

### 5.2 LLVM 优化 Pass

**optimize_module 函数**:

```cpp
void optimize_module(
    llvm::Module *module,
    int opt_level,
    const std::string &arch,
    const std::string &features,
    const std::vector<std::string> &flags,
    bool enable_fp_fusion
) {
    llvm::PassBuilder pb;
    llvm::LoopAnalysisManager lam;
    llvm::FunctionAnalysisManager fam;
    llvm::CGSCCAnalysisManager cgam;
    llvm::ModuleAnalysisManager mam;

    // 注册分析 Pass
    pb.registerModuleAnalyses(mam);
    pb.registerCGSCCAnalyses(cgam);
    pb.registerFunctionAnalyses(fam);
    pb.registerLoopAnalyses(lam);
    pb.crossRegisterProxies(lam, fam, cgam, mam);

    // 创建优化 Pass 管道
    llvm::ModulePassManager mpm;
    if (opt_level == 3) {
        mpm = pb.buildPerModuleDefaultPipeline(llvm::OptimizationLevel::O3);
    }

    // 运行优化
    mpm.run(*module, mam);
}
```

**O3 优化包括**:
- 函数内联
- 常量传播和折叠
- 死代码消除
- 循环展开和向量化
- 指令合并和重排序
- 寄存器分配优化

---

## 六、总结

Triton 编译器的后端代码生成采用了清晰的分层架构:

1. **前端**: Python AST → Triton IR
2. **中端**: Triton IR → TritonGPU IR (GPU 特定优化)
3. **后端**: TritonGPU IR → LLVM IR → PTX/AMDGCN → CUBIN/HSACO

**关键特性**:
- **多后端支持**: 统一的抽象接口,支持 NVIDIA 和 AMD
- **多阶段编译**: 每个阶段独立,便于调试和优化
- **缓存机制**: 避免重复编译,提高开发效率
- **LLVM 集成**: 利用 LLVM 强大的优化和代码生成能力

**性能优化**:
- GPU 特定优化 (内存合并、矩阵加速、流水线)
- LLVM O3 优化
- 硬件特定指令生成 (Tensor Cores、MFMA)
- 资源分配优化 (共享内存、寄存器)
