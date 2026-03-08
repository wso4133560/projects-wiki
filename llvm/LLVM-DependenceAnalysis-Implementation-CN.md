# LLVM `DependenceAnalysis` (`da`) 实现详解（源码导读）

> 阅读定位：本文聚焦 LLVM 中传统循环依赖分析 pass —— `DependenceAnalysis`（pass 名 `da`），基于当前仓库源码进行实现级拆解。
>
> 目标读者：已经熟悉 LLVM IR / PassManager / SCEV 基本概念，希望快速建立“从 API 到算法再到调用方”的完整实现心智模型。

---

## 1. 范围与源码入口

### 1.1 本文覆盖范围

仅覆盖 `da`：

- 分析对象：函数内两条内存访问指令（主要是 load/store）是否存在依赖，若存在给出方向/距离等信息。
- 不覆盖：`MemoryDependenceAnalysis`（`memdep`）、`LoopAccessAnalysis` 的向量化专用依赖检查。

### 1.2 关键文件

- 接口与数据结构：`llvm/include/llvm/Analysis/DependenceAnalysis.h`
- 实现主体：`llvm/lib/Analysis/DependenceAnalysis.cpp`
- New PM 注册：`llvm/lib/Passes/PassRegistry.def`
- Printer 选项解析：`llvm/lib/Passes/PassBuilder.cpp`
- 主要测试：`llvm/test/Analysis/DependenceAnalysis/*`

### 1.3 设计背景（源码自述）

`DependenceAnalysis.cpp` 文件头明确说明：当前实现基于 PLDI'91 的 *Practical Dependence Testing*，并且是“**不完整实现**（incomplete）”，保守性优先于激进结论。

源码中明确提到的限制包括：

- 无法在“耦合 RDIV”之间传播约束；
- 缺少多下标 MIV 的联合测试；
- 因此会产生保守结果（通常表现为 `confused` 或方向不够精确），但不会破坏正确性。

---

## 2. Pass 形态、注册与运行

### 2.1 New Pass Manager（主路径）

`DependenceAnalysis` 在 New PM 下是 `FunctionAnalysis`：

- 在 `PassRegistry.def` 中注册：
  - `FUNCTION_ANALYSIS("da", DependenceAnalysis())`
- 运行入口：
  - `DependenceAnalysis::run(Function &F, FunctionAnalysisManager &FAM)`
  - 内部获取依赖分析输入：
    - `AAManager`
    - `ScalarEvolutionAnalysis`
    - `LoopAnalysis`
  - 返回 `DependenceInfo`（按值结果对象）

### 2.2 Printer Pass

用于调试输出的 pass：

- `FUNCTION_PASS("print<da>", DependenceAnalysisPrinterPass(errs()))`
- 带参数形式：
  - `print<da><normalized-results>`
  - 参数解析函数：`parseDependenceAnalysisPrinterOptions`

`normalized-results` 含义：打印前尝试把“负方向向量”规范化（通过 `D->normalize(SE)`）。

### 2.3 Legacy PM 兼容层

`DependenceAnalysisWrapperPass` 提供旧 PM 访问方式：

- `INITIALIZE_PASS_BEGIN/END(..., "da", "Dependence Analysis", true, true)`
- `getAnalysisUsage` 声明传递依赖：`AAResultsWrapperPass` / `ScalarEvolutionWrapperPass` / `LoopInfoWrapperPass`
- `runOnFunction` 内部构造 `DependenceInfo`

说明：当前主线用户应优先使用 New PM 接口，但 legacy 包装层仍保留。

---

## 3. 对外 API 与核心数据模型

### 3.1 `Dependence`：最小依赖描述

`Dependence` 表示两个内存访问之间“存在依赖但信息可能很粗”的结果。

关键能力：

- 基础类型判断：
  - `isInput()` / `isOutput()` / `isFlow()` / `isAnti()`
- 顺序性：
  - `isOrdered()`（output/flow/anti）
  - `isUnordered()`（input）
- 精度状态：
  - 默认 `isConfused() = true`
- 图链接字段：
  - `NextPredecessor` / `NextSuccessor`（为依赖图构建预留）

### 3.2 `FullDependence`：带方向/距离向量的精细结果

`FullDependence` 扩展 `Dependence`，额外携带：

- `Levels`：共同循环层数；
- `DVEntry[]`：每层方向/距离信息；
- `LoopIndependent`：是否可能 loop-independent；
- `Consistent`：分析是否一致（若用了保守近似可能置为 false）。

### 3.2.1 `DVEntry` 语义

每层循环有一个 `DVEntry`：

- 方向集合：`LT` / `EQ` / `GT` 及其并集（`LE` `GE` `NE` `ALL`）
- `Distance`：可选 SCEV 距离
- 变换提示标志：
  - `PeelFirst`
  - `PeelLast`
  - `Splitable`
- `Scalar`：该层是否“标量层”（该层归纳变量未参与下标）

### 3.2.2 负方向与规范化

- `isDirectionNegative()`：如果最左非 `EQ` 方向是 `GT/GE`，认为方向向量“负”。
- `normalize(SE)`：交换 `Src/Dst`，翻转方向与距离符号，把结果规范到“非负”方向。

注意：源码里对 `<>` / `<=>` / `*` 做了注释说明，规范化判定故意保守简化。

### 3.3 `Dependence::dump` 输出格式

打印逻辑在 `Dependence::dump(raw_ostream&)`：

- `confused` 直接输出 `confused!`
- 否则输出：
  - 一致性前缀 `consistent`（可选）
  - 依赖类型（flow/output/anti/input）
  - 各层方向/距离/peel 信息，如 `[<= <>]`、`[S]` 等
  - 若 loop-independent，尾部带 `|<`
  - 若可分裂，尾部带 `splitable`

这也是大多数 `print<da>` 回归测试的直接对比文本来源。

### 3.4 `DependenceInfo`：分析驱动器

`DependenceInfo` 是真正做判定的对象，核心公开方法：

- `std::unique_ptr<Dependence> depends(Instruction *Src, Instruction *Dst)`
- `const SCEV *getSplitIteration(const Dependence &Dep, unsigned Level)`
- `bool invalidate(Function&, const PreservedAnalyses&, Invalidator&)`

构造依赖项：

- `AAResults *AA`
- `ScalarEvolution *SE`
- `LoopInfo *LI`
- `Function *F`

---

## 4. `depends()` 主流程逐段导读

这一节按 `DependenceInfo::depends` 实际执行顺序讲。

### 4.1 早退与可分析性过滤

1. 若任一指令不访问内存：`return nullptr`
2. 若不是“简单 load/store”（例如 call/invoke、或原子/volatile）：返回 `Dependence`（confused）
3. 计算指针 `SrcPtr/DstPtr`

其中“简单 load/store”的定义由 `isLoadOrStore()` 给出：

- 仅 `LoadInst::isUnordered()` / `StoreInst::isUnordered()` 为真时可进入精细分析。

### 4.2 AA 预筛：`underlyingObjectsAlias`

这是 `depends()` 前半段最关键的剪枝。

流程：

1. 用 `BatchAAResults`（开启 cross-iteration mode）先查 `NoAlias`；
2. 再看底层对象是否同一 identified object；
3. 若可证明不同 identified object -> `NoAlias`；
4. 其余情况 -> `MayAlias` / `PartialAlias` / `MustAlias`。

`depends()` 对结果处理：

- `NoAlias` -> 无依赖，直接 `nullptr`
- `MayAlias` / `PartialAlias` -> 无法精确，返回 `confused`
- `MustAlias` -> 进入后续 SCEV/方向向量分析

### 4.3 循环层级映射

调用 `establishNestingLevels(Src, Dst)`，建立：

- `CommonLevels`
- `SrcLevels`
- `MaxLevels`

作用：把“Src 专属循环 + 公共循环 + Dst 专属循环”统一映射到一个层级编号空间，为后续子脚本分类、方向向量更新、约束传播提供一致索引。

### 4.4 初始下标表达式与 base 校验

- `SrcSCEV = SE->getSCEV(SrcPtr)`
- `DstSCEV = SE->getSCEV(DstPtr)`

关键防御：

- 若 `SE->getPointerBase(SrcSCEV) != SE->getPointerBase(DstSCEV)`
- 直接返回 `confused`

原因：不同 pointer base 时，后续 `getMinusSCEV` 等比较可能不可计算，源码明确用该判断避免崩溃与伪精确结论。

### 4.5 Delinearization（可选）

由开关控制：

- `-da-delinearize`（默认 true）
- `-da-disable-delinearization-checks`（关闭安全检查，可能导致某些语言模型下错误方向向量）

如果 `tryDelinearize` 成功，单一线性访问会拆成多维下标对，后续按“每个 subscript pair”逐个处理，尽量避免昂贵且不精确的 MIV 测试。

### 4.6 子脚本预处理与分类

每个下标对执行：

1. `removeMatchingExtensions`：剥离可配对的同构扩展（zext/sext）
2. `classifyPair`：分类为
   - `ZIV`
   - `SIV`
   - `RDIV`
   - `MIV`
   - `NonLinear`
3. 收集循环位图 `Loops` / `GroupLoops` / `Group`

### 4.7 separable / coupled 分组

源码实现的是“最小耦合组”划分：

- `ZIV` 默认可分离（separable）；
- `SIV/RDIV/MIV` 根据 `GroupLoops` 交集决定是否耦合；
- `NonLinear` 不参与精确分解，但会影响一致性（`Result.Consistent = false`）。

结果：

- `Separable`：可独立测试
- `Coupled`：需约束联立 + 传播

### 4.8 可分离子脚本测试

对 `Separable` 中每项，根据分类调用：

- `testZIV`
- `testSIV`
- `testRDIV`
- `testMIV`

返回约定非常关键：

- 这些 `test*` 返回 `true` 表示“**已证伪依赖**”，`depends()` 立刻返回 `nullptr`；
- 返回 `false` 表示“依赖仍可能存在”，继续收敛方向/约束。

### 4.9 耦合组测试 + 约束传播

若存在 `Coupled` 组：

1. 初始化每层 `Constraint` 为 `Any`
2. 优先处理组内 `SIV`，将新约束并入层级约束向量
3. 如果约束变化，执行 `propagate` 尝试把 MIV 降级为 SIV/ZIV
4. 新变成 ZIV/SIV 的项立刻再测试
5. 剩余 RDIV 执行 `testRDIV`
6. 剩余 MIV 执行 `testMIV`
7. 用 `updateDirection` 把约束回写到 `Result.DV`

若某层方向更新成 `NONE`，说明可证伪，返回 `nullptr`。

### 4.10 收尾修正

### 4.10.1 Scalar 位修正

合并所有 subscript 的 loop 参与信息，修正各层 `DV[Level].Scalar`。

### 4.10.2 `LoopIndependent` 修正

- 对 `PossiblyLoopIndependent=true` 场景：
  - 若任一层方向不含 `EQ`，则 `LoopIndependent=false`
- 对 `Src==Dst` 场景（初始设为 false）：
  - 若全部层方向都是 `EQ`，判定“无依赖”，返回 `nullptr`

### 4.10.3 最终返回

- `nullptr`：可证伪依赖
- `Dependence`：存在依赖但信息混乱（confused）
- `FullDependence`：包含方向/距离/peel/split 信息

---

## 5. 关键测试函数族（逐函数组解读）

### 5.1 `testZIV`

处理“零归纳变量”场景（常量下标对常量下标）。

- 成功证伪：直接无依赖；
- 否则通常只给出有限方向信息。

### 5.2 `testSIV` 及其分支

`testSIV` 是入口分发器，按系数关系落到不同子测试：

- `strongSIVtest`
- `exactSIVtest`
- `weakCrossingSIVtest`
- `weakZeroSrcSIVtest`
- `weakZeroDstSIVtest`

这些函数的共同特征：

- 尝试更新方向与距离；
- 不可完全证明时可能标记 `Result.Consistent = false`；
- 某些场景会设置 `PeelFirst/PeelLast/Splitable`。

### 5.2.1 `weakCrossingSIVtest` 的 split 语义

这是 `getSplitIteration()` 的来源之一：

- 当检测到可通过分裂循环打破依赖时，`DVEntry.Splitable = true`。
- split 迭代点不常驻存储，后续通过 `getSplitIteration` 按需重算。

### 5.3 `testRDIV`、`exactRDIVtest`、`symbolicRDIVtest`

`RDIV` 处理“不同归纳变量相互关系”场景，两个子测试互补：

- `exactRDIVtest`：某些结构精度更高
- `symbolicRDIVtest`：可覆盖 exact 处理不到的符号场景

源码明确写到：二者互补，且 RDIV 结果目前无法像 SIV 一样完整传播。

### 5.4 `testMIV`、`gcdMIVtest`、`banerjeeMIVtest`

MIV 是最复杂场景：

- `gcdMIVtest`：便宜、可快速证伪部分方向
- `banerjeeMIVtest`：更系统的上下界与方向推导

`banerjeeMIVtest` 依赖下列函数族：

- `collectCoeffInfo`
- `findBoundsALL/EQ/LT/GT`
- `testBounds`
- `exploreDirections`
- `getLowerBound` / `getUpperBound`

并受 `-da-miv-max-level-threshold` 限制递归探索深度（防止组合爆炸）。

---

## 6. 约束系统与传播机制

### 6.1 `Constraint` 五态

`Constraint` 在实现里有 5 个 kind：

- `Any`
- `Line`
- `Distance`
- `Point`
- `Empty`

并集/交集核心在 `intersectConstraints`：

- 用于把新得到的层级约束合并进全局约束向量；
- 若合并后为 `Empty`，可立即判无依赖。

### 6.2 `propagate*` 系列

- `propagate`：总控
- `propagateDistance`
- `propagateLine`
- `propagatePoint`

传播后可能发生：

- 子脚本表达式简化（`Src/Dst` 被重写）
- 分类从 MIV 降级到 SIV/ZIV
- 若传播是保守近似，会置 `Consistent=false`

这也是耦合组可逐步“解耦”的关键。

---

## 7. Delinearization 代码路径

核心函数：

- `tryDelinearize`
- `tryDelinearizeFixedSize`
- `tryDelinearizeParametricSize`

关键点：

- Clang 可能把多维下标线性化；DA 尝试反线性化恢复多维结构，避免把所有问题推到 MIV。
- fixed-size 与 parametric-size 两条路径分别处理静态维度和符号维度。
- 默认有“合法性检查”来避免跨维下标溢出/借位导致错误方向推断；可通过 `-da-disable-delinearization-checks` 关闭，但会带来正确性风险。

---

## 8. `getSplitIteration()`：按需重算策略

`getSplitIteration(const Dependence &Dep, unsigned SplitLevel)` 并不读取缓存，而是“重走精简版分析流程”推导分裂点。

源码特别强调该函数必须与 `depends()` 保持语义同步：

- 如果后续维护改了 `depends()` 主流程，却忘记同步 `getSplitIteration()`，可能导致 split 提示与实际依赖判定不一致。

---

## 9. 分析失效机制（New PM）

`DependenceInfo::invalidate` 逻辑：

1. 若 `DependenceAnalysis` 自身未被保留，直接失效；
2. 否则检查传递依赖：
   - `AAManager`
   - `ScalarEvolutionAnalysis`
   - `LoopAnalysis`

只要其中任一失效，`da` 结果也失效。

对应测试：`new-pm-invalidation.ll`，验证了 `invalidate<scalar-evolution>` 会连带使 `DependenceAnalysis` 失效并重算。

---

## 10. 调用方如何消费 `da` 结果

### 10.1 LoopInterchange

`LoopInterchange.cpp` 中典型模式：

1. `if (auto D = DI->depends(Src, Dst))`
2. `D->normalize(SE)` 先归一化负方向
3. 读取每层 `getDirection()` 映射到 `<,>,=,*`
4. 构造依赖矩阵并做词典序合法性判定

要点：Interchange 对方向向量“符号形状”高度敏感，因此 normalize 非常关键。

### 10.2 LoopFuse / CodeMoverUtils

二者都把 `da` 当作“可移动性守门员”：

- 若发现 `flow/anti/output` 依赖，则禁止 hoist/sink/move
- `input` 依赖通常不阻止重排

因此，`da` 返回 `confused` 时，消费方往往会走保守路径，降低变换激进性。

### 10.3 LoopUnrollAndJam

`LoopUnrollAndJamPass` 直接构造 `DependenceInfo` 参与 legality 检查，属于“变换 pass 内嵌 DA”而非“由分析管理器缓存复用”的使用方式。

---

## 11. 命令行与调试实践

### 11.1 最常用命令

```bash
opt < input.ll -disable-output "-passes=print<da>" -aa-pipeline=basic-aa
```

打印规范化结果：

```bash
opt < input.ll -disable-output -passes='print<da><normalized-results>' \
  -aa-pipeline=basic-aa
```

强制要求分析并观察失效：

```bash
opt < input.ll -disable-output \
  -passes='require<da>,invalidate<scalar-evolution>,print<da>' \
  -debug-pass-manager
```

### 11.2 常用 `da` 选项

- `-da-delinearize`（默认 true）
- `-da-disable-delinearization-checks`
- `-da-miv-max-level-threshold=<N>`（默认 7）

---

## 12. 测试用例地图（按能力归类）

`llvm/test/Analysis/DependenceAnalysis/` 下可以按功能快速定位：

- 基础打印/输出形态：
  - `Dump.ll`
  - `ZIV.ll`
- SIV 族：
  - `StrongSIV.ll`
  - `ExactSIV.ll`
  - `WeakCrossingSIV.ll`
  - `WeakZeroSrcSIV.ll`
  - `WeakZeroDstSIV.ll`
  - `SymbolicSIV.ll`
- RDIV：
  - `ExactRDIV.ll`
  - `SymbolicRDIV.ll`
- MIV / Banerjee / GCD：
  - `Banerjee.ll`
  - `GCD.ll`
  - `MIVCheckConst.ll`
  - `MIVMaxLevelThreshold.ll`
- 耦合与传播：
  - `Coupled.ll`
  - `Propagating.ll`
  - `Constraints.ll`
  - `Separability.ll`
- Delinearization：
  - `DADelin.ll`
  - `PreliminaryNoValidityCheckFixedSize.ll`
  - `SimpleSIVNoValidityCheck*.ll`
- 稳定性/回归：
  - `BasePtrBug.ll`
  - `NonCanonicalizedSubscript.ll`
  - `PR21585.ll` / `PR31848.ll` / `PR51512.ll`
  - `new-pm-invalidation.ll`

建议阅读顺序：`ZIV -> Strong/Exact SIV -> Weak SIV -> RDIV -> MIV -> Coupled/Propagating -> Delinearization -> invalidation`。

---

## 13. 已知限制与维护注意事项

### 13.1 源码明确限制（非推测）

- 仍是“work in progress”；
- 耦合 RDIV 约束传播不足；
- 多下标 MIV 测试不完整；
- 某些路径标注“temporary”；
- 多处 `llvm_unreachable` 假设输入已满足分类前提。

### 13.2 工程维护时的高风险点

1. 修改 `depends()` 时必须同步检查 `getSplitIteration()`。
2. 调整 SCEV 化简/扩展行为时，要重点回归：
   - `NonCanonicalizedSubscript.ll`
   - `BasePtrBug.ll`
   - `PR21585.ll`
3. 调整 delinearization 策略时，必须同时回归“开启/关闭检查”两组 case。

---

## 14. 一句话总结

`da` 的工程价值不在”永远精确”，而在”以 AA + SCEV + 循环层级 + 约束传播为骨架，尽可能证明无依赖；证明不了就保守返回”，并通过 `FullDependence` 把可用于循环变换的方向/距离/peel/split 信号暴露给上层优化 pass。

---

# 附录：DependenceAnalysis.cpp 函数逐一实现报告

> 本附录以函数为单位，对 `llvm/lib/Analysis/DependenceAnalysis.cpp` 的全部函数进行实现级分析，并标注对应论文出处。

## 参考论文

| 论文 | 引用位置 |
|---|---|
| **Goff, Kennedy, Tseng, “Practical Dependence Testing”, PLDI 1991** | 整体框架、ZIV/SIV/RDIV/MIV 分类、约束传播（Figure 4、5） |
| **Banerjee, “Dependence Analysis for Supercomputing”, Kluwer Academic, 1988** | `exactSIVtest`（Algorithm 6.2.1 case 2.5）、`banerjeeMIVtest`（Banerjee 不等式） |
| **Wolfe, “Optimizing Supercompilers for Supercomputers”, MIT Press, 1989** | `findGCD`（Program 2.1, p.29 Kirch 算法）、`banerjeeMIVtest`（Section 2.5.2，Extreme Value Test，p.25） |

---

## A.1 Pass 注册与入口

### `DependenceAnalysis::run`

**New Pass Manager** 入口。从 `FAM` 获取三个分析结果（`AAManager`、`ScalarEvolutionAnalysis`、`LoopAnalysis`），构造并返回 `DependenceInfo` 对象。

### `DependenceAnalysisWrapperPass::runOnFunction`

**Legacy Pass Manager** 适配器，功能等同上者，返回 `false`（不修改 IR）。

### `DependenceAnalysisWrapperPass::getAnalysisUsage`

声明依赖 `AAResultsWrapperPass`、`ScalarEvolutionWrapperPass`、`LoopInfoWrapperPass`，标记为 `setPreservesAll()`。

### `dumpExampleDependence`（静态辅助）

测试/打印用途：枚举函数中所有内存访问指令对，调用 `depends()` 并格式化打印依赖向量，支持 `normalize()`。

---

## A.2 `Dependence` 类（基类）

### `isInput / isOutput / isFlow / isAnti`

判断依赖类型：

- **Input**：Src 读 + Dst 读
- **Output**：Src 写 + Dst 写（WAW）
- **Flow（真依赖）**：Src 写 + Dst 读（RAW）
- **Anti**：Src 读 + Dst 写（WAR）

### `isScalar(unsigned level)`

基类默认返回 `false`，在 `FullDependence` 中覆盖为读取 `DV[level-1].Scalar`。

---

## A.3 `FullDependence` 类

### 构造函数

分配 `CommonLevels` 个 `DVEntry` 数组 `DV`，初始 `Consistent = true`，每个方向向量初始为 `ALL`（全方向）。

### `isDirectionNegative()`

遍历 `DV`，在第一个非 `EQ` 的层次上判断：若方向是 `GT` 或 `GE`，认为依赖向量为”负向”。

### `normalize(ScalarEvolution *SE)`

若 `isDirectionNegative()` 为真：

1. 交换 `Src` 和 `Dst`
2. 翻转方向位（LT ↔ GT，EQ 保持）
3. 将距离取负（`SE->getNegativeSCEV(Distance)`）

**目的**：使方向向量第一个有方向的 Level 分量为 `<` 或 `=`，形成规范化表示。

### `getDirection / getDistance / isScalar / isPeelFirst / isPeelLast / isSplitable`

简单 getter，访问 `DV[Level-1]` 中对应字段，带 Level 范围断言。

---

## A.4 `Constraint` 类（约束表示）

Constraint 用于 **Delta 测试**中表示下标关系：

| 类型 | 含义 | 表达形式 |
|---|---|---|
| `Any` | 无约束（初始状态） | — |
| `Distance` | 距离约束 | `X - Y = d`（化为 `1·X + (-1)·Y = -d`） |
| `Line` | 线性约束 | `A·X + B·Y = C` |
| `Point` | 点约束（精确解） | `<X, Y>` |
| `Empty` | 矛盾（不可满足） | — |

### `setDistance(const SCEV *D, const Loop *CurLoop)`

将距离 `d` 编码为 Line 形式：`A=1, B=-1, C=-d`。`getD()` 返回 `SE->getNegativeSCEV(C)`（即 `d`）。

---

## A.5 `intersectConstraints` —— Delta 约束合并

**参考**：Goff et al. PLDI 1991, Figure 4

将约束 `Y` 与 `X` 求交，结果存入 `X`，返回 `true` 表示 `X` 改变。

**各情形处理：**

1. **Any ∩ Any** → 无变化
2. **Any ∩ Y** → `X = Y`
3. **Empty** → 不变
4. **X ∩ Empty** → `X = Empty`
5. **Distance ∩ Distance**：已知相等不变，已知不等 → `Empty`
6. **Line ∩ Line**：
   - 斜率相同（平行线）：检查是否重合，不重合 → `Empty`
   - 斜率不同（相交）：Cramer 法则求整数交点 `(Xq, Yq)`；余数非零或越界 → `Empty`；否则设 `Point(Xq, Yq)`
7. **Point ∩ Line**：点不满足线方程 → `Empty`

---

## A.6 辅助工具函数

### `underlyingObjectsAlias`（静态）

三段式别名判断：① `BAA.isNoAlias()` → `NoAlias`；② 底层对象相同 → `MustAlias`；③ 均为已识别对象但不同 → `NoAlias`；④ 其余 → `MayAlias`。使用 `enableCrossIterationMode()` 支持跨迭代别名识别。

### `isLoadOrStore`（静态）

只接受**无序（unordered）**的 `LoadInst` 和 `StoreInst`，拒绝 atomic、volatile 操作。

### `establishNestingLevels`

建立 Src/Dst 的循环嵌套共同级别映射：

```
级别:  1 ... CommonLevels ... SrcLevels ... MaxLevels
对应:  外层公共循环          Src特有循环    Dst特有循环
```

算法：从各自最深循环向外爬升直到相遇，相遇深度即 `CommonLevels`。

### `mapSrcLoop / mapDstLoop`

- `mapSrcLoop`：直接返回 `getLoopDepth()`
- `mapDstLoop`：深度超过 `CommonLevels` 时映射到 Dst 专属区间 `D - CommonLevels + SrcLevels`

### `isLoopInvariant`

“不在任何循环中”视为循环不变量（分析的是访问点的 SCEV，而非整个函数）。

### `collectCommonLoops`

遍历 `LoopNest`，对每个 Level ≤ `CommonLevels` 且 SCEV 依赖该循环的层次，在 `Loops` 位图置位。

### `unifySubscriptType`

统一下标对的类型宽度：找最宽整数类型，sign-extend 所有更窄下标（防止类型不一致导致分析失误）。

### `removeMatchingExtensions`

若 Src 和 Dst 的 SCEV 都是相同方向的整数扩展（同为 sext 或 zext）且操作数类型一致，去掉扩展层简化分析。

### `checkSubscript / checkSrcSubscript / checkDstSubscript`

递归检查 SCEV 是否为**仿射形式**（线性 `AddRecExpr` 链），步长必须循环不变，`AddRec` 必须属于当前 `LoopNest` 的某个循环。同时更新 `Loops` 位图。

### `classifyPair`

根据 Src 和 Dst 共同涉及的循环数量分类：

| 循环数量 | 分类 | 全称 |
|---|---|---|
| N == 0 | **ZIV** | Zero Index Variable |
| N == 1 | **SIV** | Single Index Variable |
| N == 2，满足单变量条件 | **RDIV** | Restricted Double Index Variable |
| 其余 | **MIV** | Multiple Index Variable |
| 非线性 | **NonLinear** | — |

### `isKnownPredicate`

增强版谓词判断：① 剥除同向 sign/zero-extend；② 调用 `SE->isKnownPredicate`；③ 失败则计算 `Delta = X - Y` 用 SE 符号判断。

### `isKnownLessThan / isKnownNonNegative`

用于去线性化合法性验证。`isKnownLessThan` 对 `AddRec` 额外利用 `BackedgeTakenCount` 进行评估。

### `collectUpperBound / collectConstantUpperBound`

获取循环次数上界（`BackedgeTakenCount`），`collectConstantUpperBound` 进一步转型为 `SCEVConstant`，失败返回 `nullptr`。

---

## A.7 核心测试函数

### `testZIV` —— ZIV 测试

**适用形式**：`[c1]` vs `[c2]`（两者均循环不变）

- `c1 == c2` → 依赖
- `c1 != c2` → 独立
- 无法判断 → 设 `Consistent = false`

---

### `strongSIVtest` —— 强 SIV 测试

**参考**：Goff et al. PLDI 1991, Section 4.2.1

**适用形式**：`[c1 + a·i]` vs `[c2 + a·i]`（**相同**系数 `a`）

**依赖距离**：`d = (c1 - c2) / a`

**步骤**：① `|Delta| > UB·|a|` → 独立；② 常量情形整除判断求精确距离；③ Delta=0 → 距离=0，方向=EQ；④ Coeff=1 → 距离=Delta；⑤ 其他符号情形分析可能方向集合。

---

### `weakCrossingSIVtest` —— 弱交叉 SIV 测试

**参考**：Goff et al. PLDI 1991, Section 4.2.2

**适用形式**：`[c1 + a·i]` vs `[c2 - a·i]`（**相反**系数）

**交叉点**（令 `i = i'`）：`i = (c2 - c1) / (2a)`

**步骤**：① Delta=0 → 只保留 `=` 方向；② 计算 `SplitIter`；③ Delta<0 → 独立；④ `Delta > 2·a·UB` → 独立；⑤ `a` 不整除 Delta → 独立；⑥ `2a` 不整除 Delta → 去掉 `=` 方向。设 `Splitable = true`。

---

### `findGCD`（静态）—— Kirch 算法

**参考**：Wolfe, “Optimizing Supercompilers”, Program 2.1, p.29

计算 `gcd(AM, BM)` 并求解 `AM·x - BM·y = gcd`（扩展 Euclidean 算法）。返回 `true` 表示 gcd 不整除 Delta（无依赖）。

### `floorOfQuotient / ceilingOfQuotient`（静态）

带符号整数的向下/向上取整除法（APInt 版），用于 `exactSIVtest` 的参数范围计算。

---

### `exactSIVtest` —— 精确 SIV 测试

**参考**：Banerjee 1988, Algorithm 6.2.1 (case 2.5)

**适用形式**：`[c1 + a1·i]` vs `[c2 + a2·i]`（一般系数，同一循环变量）

**步骤**：① GCD 整除验证；② 求参数解 `TX, TY`，在循环范围约束下求参数 `T` 的范围 `[TL, TU]`；③ `TL > TU` → 独立；④ 枚举距离范围确定方向集合。

---

### `weakZeroSrcSIVtest / weakZeroDstSIVtest` —— 弱零 SIV 测试

**参考**：Goff et al. PLDI 1991, Section 4.2.2

**适用形式**：`[c1]` vs `[c2 + a·i]`（一侧系数为零）

**步骤**：① `c1==c2` → 方向 `>=`，`PeelFirst=true`；② `Delta > UB·|a|` → 独立；③ `Delta == UB·|a|` → 方向 `<=`，`PeelLast=true`；④ `Delta < 0` → 独立；⑤ `a` 不整除 Delta → 独立。Src/Dst 变体方向语义（GE/LE）相反。

---

### `exactRDIVtest` —— 精确 RDIV 测试

**适用形式**：`[c1 + a·i]` vs `[c2 + b·j]`（**不同**循环变量）

算法与 `exactSIVtest` 相似，分别对 `i` 和 `j` 的范围约束求参数 `T` 的范围，`TL > TU` → 独立（不返回方向，因不同循环变量无法定义统一距离）。

---

### `symbolicRDIVtest` —— 符号 RDIV 测试（极值测试）

**参考**：Goff et al. PLDI 1991, Section 4.5（Banerjee Inequalities 特例）

**适用形式**：`[c1 + a1·i]` vs `[c2 + a2·j]`（含符号参数）

通过极值直接判断 `a1·i - a2·j` 的范围是否包含 `c2 - c1`：

| 系数符号 | 极值约束 |
|---|---|
| a1 ≥ 0, a2 ≥ 0 | `-a2·N2 ≤ c2-c1 ≤ a1·N1` |
| a1 ≥ 0, a2 ≤ 0 | `0 ≤ c2-c1 ≤ a1·N1 - a2·N2` |
| a1 ≤ 0, a2 ≥ 0 | `a1·N1 - a2·N2 ≤ c2-c1 ≤ 0` |
| a1 ≤ 0, a2 ≤ 0 | `a1·N1 ≤ c2-c1 ≤ -a2·N2` |

`c2-c1` 在极值范围外 → 独立。作为 SIV/RDIV 的**后备测试**。

---

### `testSIV` —— SIV 测试分发器

| 情形 | 调用 |
|---|---|
| 两者均有 AddRec，系数相同 | `strongSIVtest` |
| 系数互为相反数 | `weakCrossingSIVtest` |
| 系数不同 | `exactSIVtest` |
| 只有 Src 是 AddRec | `weakZeroDstSIVtest` |
| 只有 Dst 是 AddRec | `weakZeroSrcSIVtest` |

无论哪种，都追加调用 `gcdMIVtest` 和/或 `symbolicRDIVtest` 作为辅助。

### `testRDIV` —— RDIV 测试分发器

解析三种 RDIV 结构，调用 `exactRDIVtest` + `gcdMIVtest` + `symbolicRDIVtest`。

### `testMIV` —— MIV 测试

`gcdMIVtest(Src, Dst, Result) || banerjeeMIVtest(Src, Dst, Loops, Result)`，均为保守测试（只能证伪，不能证明精确距离）。

---

## A.8 MIV 测试实现

### `gcdMIVtest` —— GCD MIV 测试

**参考**：Wolfe, “High Performance Compilers for Parallel Computing”, p.235

**步骤**：① 从 Src/Dst 所有 `AddRec` 系数提取常量，计算累积 GCD；② `ConstDelta % GCD ≠ 0` → 独立；③ **等号方向优化**：对每个循环变量 `k` 假设 `i_k = j_k`，重新计算 GCD；若 `ConstDelta % RunningGCD ≠ 0`，清除该层 `=` 方向。

能处理含符号量情形（如 `10*i + 5*N*j + 15*M + 6`），常作为 SIV/RDIV 的后备测试。

---

### `banerjeeMIVtest` —— Banerjee 不等式 MIV 测试

**参考**：Wolfe, Section 2.5.2（Extreme Value Test）

**框架**：

1. `collectCoeffInfo` 收集 Src/Dst 系数和循环上界
2. `findBoundsALL` 计算全方向（`*`）下每层 `[LB, UB]`
3. `testBounds(ALL, 0, ...)` 检查 Delta 是否在范围内
4. `exploreDirections` 递归枚举方向组合（`<`, `=`, `>` 的笛卡尔积）
5. 汇总存活方向到 `Result.DV`

**剪枝**：`MIVMaxLevelThreshold`（默认 7），超过该层数直接返回 `ALL`，避免 O(3^n) 指数爆炸。

---

### `exploreDirections` —— 方向向量层次枚举

对每个共同循环 Level 依次尝试 `<`、`=`、`>` 三个方向，递归枚举所有组合，通过 `testBounds` 剪掉不可行组合。可行组合的方向 OR 进 `Bound[K].DirSet`，最终收紧 `Result.DV`。

### `testBounds`

检查当前方向设置下 `LowerBound ≤ Delta ≤ UpperBound`，不满足则剪枝。

### `findBoundsALL / findBoundsEQ / findBoundsLT / findBoundsGT`

计算 Banerjee 不等式在各方向下的上下界（基于 Wolfe 公式，已对规范化循环 L=0, N=1 简化）：

| 方向 | 下界 LB | 上界 UB |
|---|---|---|
| `*` | `(A⁻ - B⁺) · U` | `(A⁺ - B⁻) · U` |
| `=` | `(A-B)⁻ · U` | `(A-B)⁺ · U` |
| `<` | `((A⁻-B)⁻ · (U-1)) - B` | `((A⁺-B)⁺ · (U-1)) - B` |
| `>` | `((A-B⁺)⁻ · (U-1)) + A` | `((A-B⁻)⁺ · (U-1)) + A` |

其中 `A⁺ = max(A, 0)`, `A⁻ = min(A, 0)` 分别由 `getPositivePart / getNegativePart` 实现（`smax / smin`）。

### `collectCoeffInfo`

逐层拆解 `AddRecExpr` 链，填充 `CoefficientInfo` 数组：系数、正部、负部、循环上界。

### `getLowerBound / getUpperBound`

对各层 `Bound[K].Lower/Upper[Direction]` 求 SCEV 加法（`nullptr` 表示 ±∞）。

---

## A.9 Delta 约束传播（对应论文 Figure 5）

### `findCoefficient`

从 SCEV 中提取特定循环的步长系数，���归查找 `AddRec` 链，无则返回 0。

### `zeroCoefficient`

将 SCEV 中某循环的系数清零（去掉该 `AddRec` 层，保留其起点）。

### `addToCoefficient`

向 SCEV 的某循环系数加一个值 `Value`。结果为 0 则消去该层，无该层则新建 `AddRec`。

### `propagate`

遍历 `Loops` 中每个位，分发到 `propagateDistance`、`propagateLine` 或 `propagatePoint`。**目的**：利用已知约束化简耦合下标组中的 MIV 下标，可能将其降级为 SIV 或 ZIV。

### `propagateDistance`

已知距离约束（`i' - i = d`）：从 Src 减去 `A_K·d` 并清零第 K 层，向 Dst 加入 `-A_K`。Dst 第 K 层非零 → `Consistent = false`。

### `propagateLine`

线性约束 `A·i + B·i' = C` 的传播，四种情形：① `A=0`；② `B=0`；③ `A=B`；④ 一般情形（乘以 `A` 消去分数）。

### `propagatePoint`

精确解约束 `<X, Y>`：将 `A_K·X - A'_K·Y` 加到 Src 常量部分，清零 Src/Dst 第 K 层。

### `updateDirection`

根据约束类型更新 `DVEntry`：Distance 用 SE 符号信息确定方向；Point 通过 X/Y 比较确定方向；Line 不改变（由测试函数已设置）。

---

## A.10 去线性化（Delinearization）

> **背景**：Clang 将多维数组下标线性化（如 `A[i*N + j]`），导致 MIV 分析精度低。去线性化还原为 `A[i][j]`，逐维应用 SIV 测试。

### `tryDelinearize`

主控：验证 Src/Dst 来自**同一基地址**，依次尝试 `tryDelinearizeFixedSize` 和 `tryDelinearizeParametricSize`，成功则扩展 `Pair` 为多维下标对。

### `tryDelinearizeFixedSize`

从 GEP 恢复固定大小多维下标，合法性验证（非禁用时）：各维下标 ≥ 0 且 < 维度大小。

### `tryDelinearizeParametricSize`

针对 `A[M][N]` 类运行时大小数组：① `collectParametricTerms` 收集参数项；② `findArrayDimensions` 推断各维大小；③ `computeAccessFunctions` 计算各维下标 SCEV；④ 合法性验证。

---

## A.11 主接口函数

### `depends(Instruction *Src, Instruction *Dst)` —— 主入口

**参考**：Goff et al. PLDI 1991, Section 3.1

**完整执行流程**：

```
1. 快速过滤
   ├── 非内存指令 → nullptr
   ├── 非 Load/Store（atomic/volatile）→ 保守依赖
   └── 别名检查
       ├── NoAlias → nullptr
       ├── MayAlias/PartialAlias → 保守依赖
       └── MustAlias → 继续

2. 建立循环嵌套 (establishNestingLevels)
3. 构造 FullDependence 和 Subscript 对
4. 尝试去线性化 (tryDelinearize)
5. 对每个下标对分类 (classifyPair)
   并划分为 可分离(Separable) / 耦合(Coupled) 组

6. 测试可分离下标
   ├── ZIV → testZIV
   ├── SIV → testSIV
   ├── RDIV → testRDIV
   └── MIV → testMIV
   （任一证伪 → nullptr）

7. 测试耦合下标组（Delta 测试）
   ├── 组内 SIV 测试收集约束
   ├── 约束变化时对 MIV 传播（可降级为 ZIV/SIV 再测）
   ├── 剩余 RDIV → testRDIV
   ├── 剩余 MIV → testMIV
   └── 从约束向量更新方向向量

8. 修正 Scalar 标志
9. 确定 LoopIndependent 标志
10. 返回 FullDependence
```

### `getSplitIteration(const Dependence &Dep, unsigned SplitLevel)`

按需重新计算分割迭代值（不预先存储以节省空间）。流程与 `depends()` 完全平行，只关注指定 Level 的 `SplitIter`（由 `weakCrossingSIVtest` 计算）。

**示例**：`for (i=0; i<10; i++) { A[i]=...; ...=A[11-i]; }` 分割点为 `i=5`。

### `invalidate`

检查自身或依赖的 AA/SE/LoopInfo 是否失效，决定缓存的 `DependenceInfo` 是否需要丢弃重算。

---

## A.12 统计计数器汇总

Pass 维护约 30 个 `STATISTIC` 计数器：

| 类别 | 计数器 |
|---|---|
| ZIV | 应用次数、证伪次数 |
| Strong SIV | 应用次数、成功次数、证伪次数 |
| Weak-Crossing SIV | 应用次数、成功次数、证伪次数 |
| Exact SIV | 应用次数、成功次数、证伪次数 |
| Weak-Zero SIV | 应用次数、成功次数、证伪次数 |
| Exact RDIV | 应用次数、证伪次数 |
| Symbolic RDIV | 应用次数、证伪次数 |
| Delta 测试 | 应用次数、成功次数、证伪次数、传播次数 |
| GCD 测试 | 应用次数、成功次数、证伪次数 |
| Banerjee 测试 | 应用次数、证伪次数、成功次数 |

---

## A.13 已知局限（代码注释说明）

1. **耦合 RDIV 约束传播**：无法在 coupled group 中传播 RDIV 约束（注释：”I don't yet understand how to propagate RDIV results”）
2. **多下标 MIV 联合测试**：耦合组中 MIV 逐个独立测试，非联合测试（标注为临时代码）
3. **整数溢出风险**：APInt 的 Add/Sub/Mul 可能溢出（文件顶部 TODO）
4. **SCEV 简化弱点**：含 sign/zero-extend 的情形 SE 化简不充分，影响分析精度
