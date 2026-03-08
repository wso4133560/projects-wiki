# TritonGPU 到 LLVM 降低过程详细文档

## 一、概述

TritonGPU 到 LLVM 的降低过程是 Triton 编译器中最复杂的转换阶段，负责将高层的布局感知 IR（TritonGPU IR）转换为可执行的 LLVM IR。这个过程涉及：

- **布局转换**：在不同 encoding 之间移动数据
- **内存操作**：共享内存分配、加载、存储
- **计算降低**：归约、扫描、矩阵乘法
- **硬件特定优化**：利用 Tensor Core、异步拷贝等硬件特性

```
TritonGPU IR (ttg.*)
      │
      ├─ 类型转换（TypeConverter）
      ├─ 布局转换（ConvertLayoutOp）
      ├─ 内存操作（MemoryOp）
      ├─ 计算操作（Reduce/Scan/Dot）
      │
      ▼
LLVM IR (MLIR LLVM Dialect)
      │
      ▼
LLVM IR (LLVM C++ API)
```

**源码位置**：
- 公共层：`lib/Conversion/TritonGPUToLLVM/`
- NVIDIA：`third_party/nvidia/lib/TritonNVIDIAGPUToLLVM/`
- AMD：`third_party/amd/lib/TritonAMDGPUToLLVM/`

---

## 二、类型转换器（TypeConverter）

### 2.1 核心类

`lib/Conversion/TritonGPUToLLVM/TypeConverter.cpp`

```cpp
class TritonGPUToLLVMTypeConverter : public LLVMTypeConverter {
  // 继承自 MLIR 的 LLVMTypeConverter
  // 添加 TritonGPU 特定的类型转换规则
};
```

### 2.2 关键类型映射

#### Triton Pointer → LLVM Pointer

```cpp
addConversion([ctx](triton::PointerType type) -> std::optional<Type> {
  return LLVM::LLVMPointerType::get(ctx, type.getAddressSpace());
});
```

所有 Triton 指针类型转换为 LLVM 的 opaque pointer（地址空间保留）。

#### RankedTensor → LLVM Struct

```cpp
Type convertTritonTensorType(RankedTensorType type) {
  Type eltType = convertType(type.getElementType());
  unsigned numElementsPerThread = getTotalElemsPerThread(type);
  SmallVector<Type, 4> types(numElementsPerThread, eltType);
  return LLVM::LLVMStructType::getLiteral(ctx, types);
}
```

**核心思想**：Tensor 被展开为每个线程持有的元素数组。

**示例**：
```
tensor<128x128xf16, #blocked<{sizePerThread=[2,2], threadsPerWarp=[8,4], warpsPerCTA=[2,2]}>>
→ struct { f16, f16, f16, f16 }  // 每线程 4 个元素
```

#### MemDescType → LLVM Struct

```cpp
Type convertMemDescType(MemDescType type) {
  SmallVector<Type, 4> types;
  // 基地址指针
  types.push_back(LLVM::LLVMPointerType::get(ctx, addressSpace));
  // 偏移量（每个维度）
  for (auto i = 0; i < rank; i++) {
    types.push_back(IntegerType::get(ctx, 32));
  }
  return LLVM::LLVMStructType::getLiteral(ctx, types);
}
```

**MemDesc 结构**：`{ ptr, offset0, offset1, ... }`

#### FP8 类型 → i8

```cpp
// 所有 FP8 变体都转换为 i8
Float8E4M3FNUZType → i8
Float8E4M3FNType → i8
Float8E5M2Type → i8
Float8E5M2FNUZType → i8
```

---

## 三、布局转换（ConvertLayoutOp）

`lib/Conversion/TritonGPUToLLVM/ConvertLayoutOpToLLVM.cpp`

### 3.1 转换策略分类

布局转换使用 **LinearLayout** 分析来决定转换策略：

```cpp
LinearLayout conversion = minimalCvtLayout(srcTy, dstTy);
auto dims = conversion.getInDimNames();

if (llvm::is_contained(dims, kBlock) || llvm::is_contained(dims, kWarp)) {
  // Case 1: 跨 CTA 或跨 Warp 转换 → 使用共享内存
  transferSwizzlingLocalMem(op, adaptor.getSrc(), rewriter);
} else if (llvm::is_contained(dims, kLane)) {
  // Case 2: Warp 内转换 → 使用 Warp Shuffle
  if (cvtNeedsWarpShuffle(srcTy, dstTy))
    return transferWithinWarp(op, adaptor, rewriter);
  else
    transferSwizzlingLocalMem(op, adaptor.getSrc(), rewriter);
} else if (llvm::is_contained(dims, kRegister)) {
  // Case 3: 线程内转换 → 寄存器重排
  return transferWithinThread(op, conversion, adaptor, rewriter);
} else {
  // Case 4: 等价布局 → 直接替换
  rewriter.replaceOp(op, adaptor.getSrc());
}
```

### 3.2 线程内转换（Register Reordering）

```cpp
LogicalResult transferWithinThread(ConvertLayoutOp op,
                                   const LinearLayout &conversion,
                                   OpAdaptor adaptor,
                                   ConversionPatternRewriter &rewriter) {
  auto inVals = unpackLLElements(loc, adaptor.getSrc(), rewriter);
  SmallVector<Value> outVals(conversion.getInDimSize(kRegister));

  // 根据 LinearLayout 映射重排寄存器
  for (int i = 0; i < outVals.size(); i++) {
    auto srcIdx = conversion.apply({{kRegister, i}}).begin()->second;
    outVals[i] = inVals[srcIdx];
  }

  Value result = packLLElements(loc, getTypeConverter(), outVals,
                                rewriter, op.getType());
  rewriter.replaceOp(op, result);
  return success();
}
```

**示例**：
```
输入: [a0, a1, a2, a3]  (blocked layout)
映射: {0→2, 1→0, 2→3, 3→1}
输出: [a1, a3, a0, a2]  (mma layout)
```

### 3.3 Warp Shuffle 转换

**核心思想**：使用 `__shfl_xor` 和 `__shfl_idx` 在 Warp 内交换数据。

```cpp
LogicalResult transferWithinWarp(ConvertLayoutOp op, OpAdaptor adaptor,
                                 ConversionPatternRewriter &rewriter) {
  // 1. 分解转换为 P = P_mixed ∘ P_lane ∘ P_reg
  auto factors = getWarpLayoutConvertDecomposition(srcTy, dstTy, bitwidth);
  auto &[pReg, pLane, mixedTranspositions, nPack] = factors;

  // 2. 应用寄存器排列 P_reg
  for (const auto &[i, v] : llvm::enumerate(inVals))
    newInVals[pReg.apply({{kReg, i}})[0].second] = v;

  // 3. 打包寄存器（如果可能）
  if (elemsPerVec > 1) {
    packedVals.emplace_back(packLLVector(loc, slice, rewriter));
  }

  // 4. 执行混合转置（Swap 或 Ship 方法）
  if (m == 1 && pLaneIsTrivial && isShippable(mixedTranspositions[0])) {
    outVals = transferWithinWarpShipImpl(...);
  } else {
    outVals = transferWithinWarpSwapImpl(...);
  }

  // 5. 解包并恢复广播
  if (elemsPerVec > 1) {
    unpackedVals = unpackVal(v);
  }
  if (!removeBroadcastDst.isIdentity())
    outVals = broadcastAs(outVals, dstLayout);
}
```

**Swap 实现（三阶段转置）**：

```cpp
// 实现 (r_i l_j) 转置：交换第 i 个寄存器位和第 j 个 lane 位
// 分解为三个阶段：
// 1. r_i ^= l_j (selp)
// 2. l_j ^= r_i (shfl)
// 3. r_i ^= l_j (selp)

// Stage 1: 选择性排列
for (const auto &t : mixedTranspositions)
  vals = applySwap(t, /*preShuf=*/true);

// Stage 2: Shuffle
for (int r = 0; r < numRegs; ++r) {
  int mask = computeMask(mixedTranspositions, r);
  if (mask != 0)
    vals[r] = targetInfo.shuffleXor(rewriter, loc, vals[r], mask);
}

// Stage 3: 选择性排列
for (const auto &t : mixedTranspositions)
  vals = applySwap(t, /*preShuf=*/false);
```

**PTX 示例**：
```ptx
// Shuffle XOR
shfl.sync.bfly.b32 %r1, %r0, 0x1, 0x1f, 0xffffffff;

// Select (selp)
selp.b32 %r2, %r1, %r0, %p;
```

### 3.4 共享内存转换（Swizzling）

```cpp
void transferSwizzlingLocalMem(ConvertLayoutOp op, Value src,
                               ConversionPatternRewriter &rewriter) {
  auto srcLayout = toLinearLayout(srcTy);
  auto dstLayout = toLinearLayout(dstTy);

  // 1. 计算最优 swizzling 模式
  auto smem = optimalSwizzlingLdSt(srcLayout, dstLayout, bitwidth);

  // 2. 分解为多个 tile（如果需要）
  auto nReps = smem.getInDimSize(kReps);

  for (int i = 0; i < nReps; ++i) {
    if (i > 0) emitBarrier();

    // 3. 存储到共享内存
    lowerLdStShared(loc, ctx, storeCvt, tileInVals, llvmElemTy,
                    smemBase, ...);

    emitBarrier();

    // 4. 从共享内存加载
    auto tileOutVals = lowerLdStShared(loc, ctx, loadCvt, {}, llvmElemTy,
                                       smemBase, ...);
    llvm::append_range(outVals, tileOutVals);
  }
}
```

**Swizzle 地址计算**：
```cpp
// XOR swizzle: addr = base + (row ^ (col / vec)) * stride + (col % vec)
Value swizzledOffset = b.xor_(rowIdx, b.lshr(colIdx, vecLog2));
Value addr = b.add(base, b.mul(swizzledOffset, stride));
addr = b.add(addr, b.and_(colIdx, vecMask));
```

---

## 四、归约操作（ReduceOp）

`lib/Conversion/TritonGPUToLLVM/ReduceOpToLLVM.cpp`

### 4.1 归约流程

```cpp
LogicalResult matchAndRewrite(triton::ReduceOp op, OpAdaptor adaptor,
                              ConversionPatternRewriter &rewriter) {
  // 1. 线程内归约
  std::tie(regLl, accs) = reduceWithinThreads(op, regLl, accs, rewriter);

  // 2. Warp 内归约
  std::tie(regLl, accs) = reduceWithinWarps(op, regLl, accs, rewriter);

  // 3. 跨 Warp/Block 归约（如果需要）
  while (regLl.getOutDimSize(kAxis) != 1) {
    LinearLayout tmpLl = ReduceOpHelper::getInterLayout(regLl, axis);

    if (i > 0) sync(rewriter, loc, lastCvtCrossesCTAs);

    // 通过共享内存转换布局
    accs = convertLayoutValues(loc, rewriter, op, regLl, tmpLl, accs);

    // 再次 Warp 内归约
    std::tie(regLl, accs) = reduceWithinWarps(op, tmpLl, accs, rewriter);
    ++i;
  }

  // 4. 打包结果
  packResults(op, accs, rewriter);
  return success();
}
```

### 4.2 线程内归约（向量化）

```cpp
std::pair<LinearLayout, SmallVector<SmallVector<Value>>>
reduceWithinThreads(triton::ReduceOp op, LinearLayout layout,
                    SmallVector<SmallVector<Value>> accs,
                    ConversionPatternRewriter &rewriter) {
  // 1. 检查是否可以向量化
  auto vectorizeKind = helper.getInThreadVectorizeOpKind(axisPack, ...);
  bool vectorize = (vectorizeKind != None);

  // 2. 移动 axis 基到前面
  auto perm = ReduceOpHelper::moveAxisBasesToFront(layout, axis, vectorize);
  if (!perm.isIdentity()) {
    layout = perm.apply(layout);
    for (auto &vals : accs) vals = perm.apply(vals);
  }

  // 3. 打包为向量（如果可以）
  if (vectorize) packVectorized(accs, rewriter);

  // 4. 树形归约
  for (unsigned regBase = 0; regBase < regs; regBase += axisPack) {
    SmallVector<SmallVector<Value>> vals;
    for (unsigned i = 0; i < axisPack; ++i) {
      vals.push_back(accs[opIdx][regBase + i]);
    }
    auto acc = treeReduceBinary(loc, rewriter, combineRegion, vals);
    reduced[opIdx].push_back(acc[opIdx]);
  }

  // 5. 解包向量
  if (vectorize) unpackVectorized(loc, accs, rewriter, reduceAfterUnpacking);

  return {layout, accs};
}
```

**树形归约示例**：
```
输入: [a0, a1, a2, a3, a4, a5, a6, a7]

Level 1: [a0+a1, a2+a3, a4+a5, a6+a7]
Level 2: [a0+a1+a2+a3, a4+a5+a6+a7]
Level 3: [a0+a1+a2+a3+a4+a5+a6+a7]
```

### 4.3 Warp 内归约

```cpp
void warpReduce(triton::ReduceOp op, unsigned reduceLaneIdMask,
                SmallVector<Value> &acc, ConversionPatternRewriter &rewriter) {
  // 尝试使用硬件 redux 指令
  if (targetInfo.warpReduce(rewriter, op.getLoc(), acc, op, reduceLaneIdMask)) {
    return;
  }

  // 回退到 shuffle 实现
  for (int bit = llvm::Log2_32(warpSize) - 1; bit >= 0; --bit) {
    unsigned mask = 1u << bit;
    if ((reduceLaneIdMask & mask) == 0) continue;

    SmallVector<Value> shfl(op.getNumOperands());
    for (unsigned i = 0; i < op.getNumOperands(); ++i) {
      shfl[i] = targetInfo.shuffleXor(rewriter, op.getLoc(), acc[i], mask);
    }
    accumulate(op.getLoc(), rewriter, op.getCombineOp(), acc, shfl);
  }
}
```

**PTX 示例**（硬件 redux）：
```ptx
// FP32 sum reduction
redux.sync.add.f32 %r0, %r1, 0xffffffff;

// INT32 max reduction
redux.sync.max.s32 %r0, %r1, 0xffffffff;
```

---

## 五、扫描操作（ScanOp）

`lib/Conversion/TritonGPUToLLVM/ScanOpToLLVM.cpp`

### 5.1 扫描流程

```cpp
LogicalResult emitFastScan(triton::ScanOp op, triton::ScanOpAdaptor adaptor,
                           ConversionPatternRewriter &rewriter,
                           const TargetInfoBase &targetInfo) {
  // 1. 解包输入
  auto srcValues = unpackInputs(loc, op, adaptor, rewriter, nElems,
                                removeBroadcastRegs);

  // 2. 如果是反向扫描，翻转数据
  if (op.getReverse()) {
    warpIdAxis = b.sub(b.i32_val(axisNumWarps - 1), warpIdAxis);
    srcValues = flipSrcValues(loc, op, rewriter, targetInfo, srcValues,
                              iWarpSize);
  }

  // 3. 线程内扫描
  scanThreadContiguousElements(srcValues, rewriter, helper);

  // 4. Warp 内扫描
  warpScan(srcValues, rewriter, targetInfo, helper, laneIdAxis);

  // 5. 跨 Warp 扫描（如果需要）
  if (axisNumWarps > 1) {
    // 慢路径：使用共享内存
    storeWarpAccumulator(srcValues, rewriter, helper, ...);
    b.barrier(triton::gpu::AddrSpace::Local);
    AddPartialReduce(srcValues, rewriter, targetInfo, helper, ...);
  } else if (srcValues.size() > 1) {
    // 快路径：单 Warp
    AddPartialReduceOneWarp(srcValues, rewriter, targetInfo, helper, ...);
  }

  // 6. 如果是反向扫描，再次翻转
  if (op.getReverse()) {
    srcValues = flipSrcValues(loc, op, rewriter, targetInfo, srcValues,
                              iWarpSize);
  }

  // 7. 打包结果
  for (unsigned i = 0; i < op.getNumOperands(); ++i) {
    results[i] = packLLElements(loc, getTypeConverter(),
                                valuesTransposed[i], rewriter, resultTy);
  }
  rewriter.replaceOp(op, results);
  return success();
}
```

### 5.2 Warp 内扫描

```cpp
static void warpScan(SmallVector<SmallVector<Value>> &srcValues,
                     ConversionPatternRewriter &rewriter,
                     const TargetInfoBase &targetInfo,
                     ScanLoweringHelper &helper, Value laneIdAxis) {
  unsigned scanDim = helper.getAxisNumThreadsPerWarpWithUniqueData();

  for (unsigned srcIndex = 0; srcIndex < srcValues.size(); srcIndex++) {
    // 只处理每个连续块的最后一个元素
    if (elementIdx != scanElementsPerThreads - 1) continue;

    SmallVector<Value> acc = srcValues[srcIndex];
    // 使用 shuffle up 进行扫描
    for (unsigned i = 1; i <= scanDim / 2; i <<= 1) {
      SmallVector<Value> shfl(acc.size());
      for (unsigned j = 0; j < acc.size(); ++j) {
        shfl[j] = targetInfo.shuffleUp(rewriter, loc, acc[j], i * threadStride);
      }
      Value mask = b.icmp_sge(laneIdAxis, b.i32_val(i));
      SmallVector<Value> tempAcc = accumulate(helper, rewriter, shfl, acc, mask);
      for (unsigned j = 0; j < acc.size(); ++j) {
        acc[j] = b.select(mask, tempAcc[j], acc[j]);
      }
    }
    srcValues[srcIndex] = std::move(acc);
  }
}
```

**Prefix Sum 示例**：
```
输入: [1, 2, 3, 4, 5, 6, 7, 8]

Step 1 (offset=1): [1, 1+2, 2+3, 3+4, 4+5, 5+6, 6+7, 7+8]
                 = [1, 3, 5, 7, 9, 11, 13, 15]

Step 2 (offset=2): [1, 3, 1+5, 3+7, 5+9, 7+11, 9+13, 11+15]
                 = [1, 3, 6, 10, 14, 18, 22, 26]

Step 3 (offset=4): [1, 3, 6, 10, 1+14, 3+18, 6+22, 10+26]
                 = [1, 3, 6, 10, 15, 21, 28, 36]
```

---

## 六、共享内存操作（MemoryOp）

`lib/Conversion/TritonGPUToLLVM/MemoryOpToLLVM.cpp`

### 6.1 LocalAlloc（共享内存分配）

```cpp
LogicalResult matchAndRewrite(triton::gpu::LocalAllocOp op, OpAdaptor adaptor,
                              ConversionPatternRewriter &rewriter) {
  // 1. 获取共享内存基地址
  SmallVector<Value> smemBases = LLVM::getSharedMemoryBases(loc, rewriter,
                                                            targetInfo, op);

  // 2. 创建共享内存对象
  auto smemObj = SharedMemoryObject(smemBases, llvmElemTy, memDescTy.getRank(),
                                   loc, rewriter);

  // 3. 如果有初始值，存储到共享内存
  if (op.getSrc()) {
    auto inVals = unpackLLElements(loc, adaptor.getSrc(), rewriter);
    lowerLocalStore(loc, ctx, op.getSrc(), memDescTy, smemObj, inVals,
                   typeConverter, rewriter, targetInfo);
  }

  // 4. 返回共享内存描述符
  auto retVal = getStructFromSharedMemoryObject(loc, smemObj, rewriter);
  rewriter.replaceOp(op, retVal);
  return success();
}
```

### 6.2 LocalLoad（从共享内存加载）

```cpp
LogicalResult matchAndRewrite(LocalLoadOp op, OpAdaptor adaptor,
                              ConversionPatternRewriter &rewriter) {
  // 1. 获取共享内存对象
  auto smemObj = LLVM::getSharedMemoryObjectFromStruct(loc, adaptor.getSrc(),
                                                       llvmElemTy, rewriter);

  // 2. 计算布局转换
  auto regLayout = toLinearLayout(regTy);
  auto sharedLayout = isPaddedEncoding(memDescTy.getEncoding())
                          ? paddedLinearLayout(memDescTy)
                          : toLinearLayout(memDescTy);
  auto cvt = regLayout.invertAndCompose(sharedLayout);

  // 3. 执行加载
  auto outVals = lowerLocalLdSt(loc, ctx, cvt, {}, llvmElemTy, memDescTy,
                                smemObj, rewriter, targetInfo, op);

  // 4. 打包结果
  Value result = packLLElements(loc, typeConverter, outVals, rewriter, regTy);
  rewriter.replaceOp(op, result);
  return success();
}
```

**PTX 示例**：
```ptx
// 向量化加载（128 位）
ld.shared.v4.b32 {%r0, %r1, %r2, %r3}, [%addr];

// 标量加载
ld.shared.b32 %r0, [%addr];
```

---

---

## 七、NVIDIA 特定 Lowering

### 7.1 MMAv2 (Ampere Tensor Cores)

**文件路径**: `third_party/nvidia/lib/TritonNVIDIAGPUToLLVM/DotOpToLLVM/MMAv2.cpp`

**支持的 PTX 指令**:

**Turing 架构**:
```ptx
mma.sync.aligned.m16n8k8.row.col.f32.f16.f16.f32
mma.sync.aligned.m8n8k16.row.col.satfinite.s32.s8.s8.s32
mma.sync.aligned.m16n8k8.row.col.f16.f16.f16.f16
```

**Ampere 架构**:
```ptx
mma.sync.aligned.m16n8k16.row.col.f32.f16.f16.f32      // FP16
mma.sync.aligned.m16n8k16.row.col.f32.bf16.bf16.f32    // BF16
mma.sync.aligned.m16n8k8.row.col.f32.tf32.tf32.f32     // TF32
mma.sync.aligned.m16n8k32.row.col.f32.e5m2.e4m3.f32    // FP8
mma.sync.aligned.m8n8k4.row.col.f64.f64.f64.f64        // FP64
```

**Hopper 架构**:
```ptx
mma.sync.aligned.m16n8k16.row.col.f64.f64.f64.f64      // FP64 优化
```

**操作数打包**:
- FP16/BF16: 每个寄存器打包 2 个元素 (32 位)
- FP8: 每个寄存器打包 4 个元素
- 支持 `largeK` 模式,将大 K 维度分割成多个子 MMA

### 7.2 WGMMA (Hopper Warpgroup Matrix Multiply)

**文件路径**: `third_party/nvidia/lib/TritonNVIDIAGPUToLLVM/DotOpToLLVM/WGMMA.cpp`

**WGMMAOp 结构**:
```cpp
triton::nvgpu::WGMMAOp {
  opA: 操作数 A (寄存器或 descriptor)
  opB: 操作数 B (必须是 descriptor)
  opC: 累加器 C (可选)
  m, n, k: 矩阵维度
  eltTypeA, eltTypeB, eltTypeC: 数据类型
  layoutA, layoutB: 布局 (row/col)
}
```

**支持的数据类型**:
- s8, s32 (整数)
- e4m3, e5m2 (FP8)
- f16, bf16 (半精度)
- tf32, f32 (单精度)

**PTX 指令格式**:
```ptx
wgmma.mma_async.sync.aligned.m64n128k16.f32.f16.f16
  {%0,%1,%2,%3}, {%4,%5}, %6, 1, 1, 1, 0, 1;
```

**支持的形状**:
- FP16/BF16: M=64, N=8-256, K=16
- TF32: M=64, N=8-256, K=8
- FP8: M=64, N=8-256, K=32
- INT8: M=64, N=8-224, K=32

**同步操作**:
```ptx
wgmma.fence.aligned;
wgmma.wait_group.sync.aligned #pendings;
```

### 7.3 MMAv5 (Blackwell TCGen5)

**文件路径**: `third_party/nvidia/lib/TritonNVIDIAGPUToLLVM/DotOpToLLVM/MMAv5.cpp`

**PTX 指令**:
```ptx
tcgen05.mma.cta_group::1.kind::f16                              // FP16/BF16
tcgen05.mma.cta_group::1.kind::tf32                             // TF32
tcgen05.mma.cta_group::1.kind::f8f6f4                           // FP8
tcgen05.mma.cta_group::1.kind::mxf8f6f4.block_scale.scale_vec::1X  // Scaled FP8
tcgen05.mma.cta_group::1.kind::mxf4.block_scale.scale_vec::2X      // Scaled FP4
```

**特性**:
- 支持 2-CTA 模式 (`cta_group::2`)
- 使用指令描述符 (32 位) 编码参数
- 支持 Tensor Memory (TMEM) 作为操作数
- 使用 `tcgen05.commit` 进行 barrier 同步

### 7.4 异步拷贝 (cp.async)

**文件路径**: `third_party/nvidia/lib/TritonNVIDIAGPUToLLVM/LoadStoreOpToLLVM.cpp`

**TMA 拷贝指令**:
```ptx
// TILED 模式
cp.async.bulk.tensor.{1,2,3,4,5}d.cta_group::1.shared::cta.global.mbarrier::complete_tx::bytes

// IM2COL 模式 (卷积)
cp.async.bulk.tensor.{rank}d.global.shared::cta.im2col.mbarrier::complete_tx::bytes

// GATHER/SCATTER
cp.async.bulk.tensor.2d.tile::gather4.shared::cta.global.mbarrier::complete_tx::bytes
cp.async.bulk.tensor.2d.tile::scatter4.global.shared::cta.bulk_group
```

**同步机制**:
```ptx
fence.proxy.tensormap::generic.acquire.gpu
cp.async.bulk.commit_group
cp.async.bulk.wait_group.read 0
```

### 7.5 ldmatrix/stmatrix

**文件路径**: `third_party/nvidia/lib/TritonNVIDIAGPUToLLVM/MemoryOpToLLVM.cpp`

**ldmatrix 指令**:
```ptx
ldmatrix.sync.aligned.m8n8.x4.shared.b16 {%r0,%r1,%r2,%r3}, [%addr];
ldmatrix.sync.aligned.m8n8.x4.trans.shared.b16 {%r0,%r1,%r2,%r3}, [%addr];
```

**特性**:
- 支持 `.x1`, `.x2`, `.x4` 向量化
- 支持 `.trans` 转置模式
- 每次加载 8x8 tile,每个线程获取 4 个元素
- 使用 `LinearLayout` 进行寄存器映射

---

## 八、AMD 特定 Lowering

### 8.1 MFMA (Matrix Fused Multiply-Add)

**文件路径**: `third_party/amd/lib/TritonAMDGPUToLLVM/DotOpToLLVM/MFMA.cpp`

**支持的指令**:
```llvm
llvm.amdgcn.mfma.f32.32x32x8f16    // FP16, 32x32x8
llvm.amdgcn.mfma.f32.16x16x16f16   // FP16, 16x16x16
llvm.amdgcn.mfma.f32.32x32x4bf16   // BF16, 32x32x4
llvm.amdgcn.mfma.f32.16x16x2f32    // FP32, 16x16x2
llvm.amdgcn.mfma.f64.16x16x4f64    // FP64, 16x16x4
llvm.amdgcn.mfma.i32.32x32x16i8    // INT8, 32x32x16
```

**操作数布局**:
- A 操作数: 每个线程持有多个元素
- B 操作数: 每个线程持有多个元素
- C 累加器: 分布在 wavefront 的所有线程中

### 8.2 WMMA (Wave Matrix Multiply-Accumulate)

**文件路径**: `third_party/amd/lib/TritonAMDGPUToLLVM/DotOpToLLVM/WMMA.cpp`

**支持的指令** (RDNA3+):
```llvm
llvm.amdgcn.wmma.f32.16x16x16.f16   // FP16
llvm.amdgcn.wmma.f32.16x16x16.bf16  // BF16
llvm.amdgcn.wmma.i32.16x16x16.iu8   // INT8
```

**特性**:
- 专为 RDNA 架构优化
- 支持 wave32 和 wave64 模式
- 更高的寄存器效率

### 8.3 Buffer 操作

**文件路径**: `third_party/amd/lib/TritonAMDGPUToLLVM/LoadStoreOpToLLVM.cpp`

**Buffer Load/Store**:
```llvm
llvm.amdgcn.raw.buffer.load.v4f32
llvm.amdgcn.raw.buffer.store.v4f32
```

**优势**:
- 32 位指针范围优化
- 硬件边界检查
- 更好的缓存行为

### 8.4 LDS (Local Data Share)

**共享内存访问**:
```llvm
// 向量化加载
load <4 x float>, ptr addrspace(3) %lds_ptr, align 16

// 原子操作
atomicrmw add ptr addrspace(3) %lds_ptr, i32 %value seq_cst
```

**Bank Conflict 避免**:
- 使用 padding 避免 bank conflict
- 支持 swizzle 模式

---

## 九、关键文件速查

| 文件 | 说明 |
|------|------|
| `TypeConverter.cpp` | 类型转换器 |
| `ConvertLayoutOpToLLVM.cpp` | 布局转换核心 |
| `ReduceOpToLLVM.cpp` | 归约操作降低 |
| `ScanOpToLLVM.cpp` | 扫描操作降低 |
| `MemoryOpToLLVM.cpp` | 共享内存操作 |
| `LoadStoreOpToLLVM.cpp` | 全局内存访问 |
| `DotOpToLLVM/FMA.cpp` | 通用矩阵乘 |
| `nvidia/lib/.../MMAv2.cpp` | Ampere MMA |
| `nvidia/lib/.../WGMMA.cpp` | Hopper WGMMA |
| `nvidia/lib/.../MMAv5.cpp` | Blackwell TCGen5 |
| `amd/lib/.../MFMA.cpp` | AMD MFMA |
| `amd/lib/.../WMMA.cpp` | AMD WMMA |
