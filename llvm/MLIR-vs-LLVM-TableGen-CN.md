# MLIR TableGen 与 LLVM TableGen 对比分析

> 来源：`mlir/include/mlir/IR/`、`llvm/lib/Target/X86/`

---

## 核心差异概览

| 维度 | LLVM TableGen | MLIR TableGen |
|------|---------------|---------------|
| **抽象层次** | 硬件指令级（接近机器码） | IR 操作级（抽象语义） |
| **核心目标** | 描述目标架构 | 描述 IR 方言 |
| **命名空间** | 单一目标（如 X86） | 多方言系统（Arith/Func/SCF 等） |
| **类型系统** | 固定（MVT：i8/i16/i32/i64/f32/f64） | 可扩展（TypeDef 自定义类型） |
| **接口支持** | 无 | OpInterface/TypeInterface/AttrInterface |
| **汇编格式** | 字符串模板 | 声明式 DSL（assemblyFormat） |

---

## 1. 设计目标对比

### 1.1 LLVM TableGen：描述硬件架构

**目标：生成代码生成器**

```
LLVM IR (目标无关)
    ↓
[LLVM TableGen 描述的目标架构]
    ↓ 指令选择
    ↓ 寄存器分配
    ↓ 指令调度
    ↓ 机器码生成
    ↓
机器码 (x86/ARM/RISCV)
```

**关键任务：**
- 定义指令编码（opcode、ModRM、REX 前缀）
- 定义寄存器层次结构（AL→AX→EAX→RAX）
- 定义指令选择模式（DAG 模式匹配）
- 定义调度模型（延迟、吞吐量）

### 1.2 MLIR TableGen：描述 IR 方言

**目标：生成 IR 基础设施**

```
高层 MLIR 方言 (TF/Torch)
    ↓
[MLIR TableGen 描述的方言转换]
    ↓ Linalg 方言
    ↓ Affine 方言
    ↓ SCF 方言
    ↓ LLVM 方言
    ↓
LLVM IR
```

**关键任务：**
- 定义操作语义（加法、循环、函数调用）
- 定义类型系统（Tensor、MemRef、Index）
- 定义接口（ShapeInference、SideEffect）
- 定义重写规则（模式匹配和替换）

---

## 2. 核心概念对比

### 2.1 LLVM TableGen 核心概念

| 概念 | 说明 | 示例 |
|------|------|------|
| `Instruction` | 指令基类 | `class X86Inst : Instruction` |
| `Register` | 寄存器定义 | `def EAX : X86Reg<"eax", 0>` |
| `RegisterClass` | 寄存器类 | `def GR32 : RegisterClass<...>` |
| `Pattern` | DAG 模式 | `[(set GR32:$dst, (add GR32:$src1, GR32:$src2))]` |
| `Predicate` | 指令谓词 | `Requires<[HasAVX]>` |
| `SchedModel` | 调度模型 | `def HaswellModel : SchedMachineModel` |

### 2.2 MLIR TableGen 核心概念

| 概念 | 说明 | 示例 |
|------|------|------|
| `Dialect` | 方言定义 | `def Arith_Dialect : Dialect` |
| `Op` | 操作定义 | `def Arith_AddIOp : Op<Arith_Dialect, "addi">` |
| `TypeDef` | 类型定义 | `def TensorType : TypeDef<Builtin_Dialect, "Tensor">` |
| `AttrDef` | 属性定义 | `def IntegerAttr : AttrDef<Builtin_Dialect, "Integer">` |
| `OpInterface` | 操作接口 | `def InferTypeOpInterface : OpInterface` |
| `Trait` | 操作特征 | `Commutative`, `NoMemoryEffect` |
| `DRR` | 声明式重写 | `Pattern<(AddIOp $a, $b), (...)>` |

---

## 3. TableGen 类定义对比

### 3.1 LLVM 指令定义

```tablegen
// llvm/lib/Target/X86/X86InstrFormats.td
class X86Inst<bits<8> opcod, Format f, ImmType i, dag outs, dag ins,
              string AsmStr, Domain d = GenericDomain>
  : Instruction {
  let Namespace = "X86";

  // 硬件编码字段
  bits<8> Opcode = opcod;           // 操作码
  Format Form = f;                   // 指令格式（MRMDestReg/MRMSrcMem 等）

  // 编码前缀
  bit hasREX_W = 0;                  // REX.W 前缀（64位操作）
  bit hasVEX_4V = 0;                 // VEX.VVVV 字段
  bit hasVEX_L = 0;                  // VEX.L 位（256位向量）
  bit hasEVEX_K = 0;                 // EVEX 掩码寄存器

  // 操作数和汇编
  dag OutOperandList = outs;
  dag InOperandList = ins;
  string AsmString = AsmStr;
}

// 具体指令定义
def ADD32rr : I<0x01, MRMDestReg, (outs GR32:$dst), (ins GR32:$src1, GR32:$src2),
                "add{l}\t{$src2, $dst|$dst, $src2}",
                [(set GR32:$dst, (add GR32:$src1, GR32:$src2))]>;
```

**特点：**
- 直接映射到机器码（opcode: 0x01）
- 包含大量硬件编码细节
- 汇编字符串手动指定

### 3.2 MLIR 操作定义

```tablegen
// mlir/include/mlir/IR/OpBase.td
class Op<Dialect dialect, string mnemonic, list<Trait> props = []> {
  Dialect opDialect = dialect;
  string opName = mnemonic;

  // 操作数和结果（抽象类型）
  dag arguments = (ins);
  dag results = (outs);

  // 区域和后继块（支持嵌套结构）
  dag regions = (region);
  dag successors = (successor);

  // 声明式汇编格式
  string assemblyFormat = "";

  // 文档
  string summary = "";
  code description = "";

  // 生成选项
  bit hasVerifier = 0;
  bit hasFolder = 0;
  bit hasCanonicalizer = 0;

  // Trait 列表
  list<Trait> traits = props;
}

// 具体操作定义
def Arith_AddIOp : Arith_Op<"addi", [Commutative, NoMemoryEffect]> {
  let summary = "integer addition operation";

  let arguments = (ins SignlessIntegerOrIndexLike:$lhs,
                       SignlessIntegerOrIndexLike:$rhs);
  let results = (outs SignlessIntegerOrIndexLike:$result);

  // 声明式汇编格式
  let assemblyFormat = "$lhs `,` $rhs attr-dict `:` type($result)";

  let hasFolder = 1;
  let hasCanonicalizer = 1;
}
```

**特点：**
- 无机器码映射，纯语义描述
- 支持区域和后继块（嵌套结构）
- 声明式汇编格式（自动生成解析器/打印器）
- 支持 Folder 和 Canonicalizer

---

## 4. 后端生成器对比

### 4.1 LLVM TableGen 后端

**位置：** `llvm/utils/TableGen/`

| 后端 | 生成内容 | 输出文件 |
|------|----------|----------|
| `InstrInfoEmitter` | 指令信息表 | `*GenInstrInfo.inc` |
| `RegisterInfoEmitter` | 寄存器信息 | `*GenRegisterInfo.inc` |
| `CodeEmitterGen` | 机器码编码器 | `*GenMCCodeEmitter.inc` |
| `AsmMatcherEmitter` | 汇编解析器 | `*GenAsmMatcher.inc` |
| `AsmWriterEmitter` | 汇编输出器 | `*GenAsmWriter.inc` |
| `DAGISelEmitter` | DAG 指令选择器 | `*GenDAGISel.inc` |
| `SubtargetEmitter` | 子目标信息 | `*GenSubtargetInfo.inc` |

### 4.2 MLIR TableGen 后端

**位置：** `mlir/tools/mlir-tblgen/`

| 后端 | 生成内容 | 输出文件 |
|------|----------|----------|
| `OpDefinitionsGen` | 操作类定义 | `*Ops.h.inc` / `*Ops.cpp.inc` |
| `DialectGen` | 方言类定义 | `*Dialect.h.inc` / `*Dialect.cpp.inc` |
| `AttrOrTypeDefGen` | 属性/类型定义 | `*AttrDefs.h.inc` / `*TypeDefs.h.inc` |
| `OpInterfacesGen` | 接口定义 | `*OpInterfaces.h.inc` |
| `OpFormatGen` | 汇编格式解析器 | 嵌入 `*Ops.cpp.inc` |
| `RewriterGen` | 模式重写代码 | `*Patterns.inc` |
| `PassGen` | Pass 定义 | `*Passes.h.inc` |

---

## 5. 完整示例对比

### 5.1 LLVM：X86 ADD 指令

```tablegen
// X86InstrArithmetic.td
multiclass ArithBinOp<bits<8> BaseOpc, string Mnemonic, SDNode OpNode> {
  // 8位寄存器-寄存器
  let Defs = [EFLAGS] in
  def 8rr : BinOpRR<BaseOpc, Mnemonic, Xi8, OpNode>;

  // 32位寄存器-寄存器
  let Defs = [EFLAGS] in
  def 32rr : BinOpRR<BaseOpc, Mnemonic, Xi32, OpNode>, OpSize32;

  // 32位寄存器-内存
  let Defs = [EFLAGS], mayLoad = 1 in
  def 32rm : BinOpRM<BaseOpc, Mnemonic, Xi32, OpNode>, OpSize32;
}

// 实例化
defm ADD : ArithBinOp<0x00, "add", add>;
// 生成: ADD8rr, ADD32rr, ADD32rm, ...
```

### 5.2 MLIR：Arith AddI 操作

```tablegen
// ArithOps.td
def Arith_AddIOp : Arith_IntBinaryOp<"addi", [Commutative]> {
  let summary = "integer addition operation";
  let description = [{
    The `arith.addi` operation takes two operands and returns one result,
    each of these is required to be the same type.
  }];

  let arguments = (ins SignlessIntegerOrIndexLike:$lhs,
                       SignlessIntegerOrIndexLike:$rhs);
  let results = (outs SignlessIntegerOrIndexLike:$result);

  let assemblyFormat = "$lhs `,` $rhs attr-dict `:` type($result)";

  let hasFolder = 1;
  let hasCanonicalizer = 1;
}
```

---

## 6. 关键差异总结

| 维度 | LLVM TableGen | MLIR TableGen |
|------|---------------|---------------|
| **抽象层次** | 机器指令（接近硬件） | IR 操作（抽象语义） |
| **核心类** | `Instruction`, `Register`, `Pattern` | `Op`, `Dialect`, `Interface`, `TypeDef` |
| **命名空间** | 单一目标架构（X86/ARM） | 多方言系统（Arith/Func/SCF） |
| **类型系统** | 固定 MVT（i8/i16/i32/i64） | 可扩展 TypeDef（自定义类型） |
| **属性系统** | 无 | AttrDef（自定义属性） |
| **接口系统** | 无 | OpInterface/TypeInterface/AttrInterface |
| **汇编格式** | 字符串模板（手动） | 声明式 DSL（自动生成） |
| **模式匹配** | DAG 模式（用于指令选择） | DRR（声明式重写规则） |
| **验证** | 硬件约束 | 语义约束（类型检查、区域结构） |
| **后端输出** | 编码器、选择器、寄存器分配器 | 操作类、方言类、接口类 |
| **应用场景** | 目标架构代码生成 | IR 方言定义和变换 |

---

## 7. 共同点

尽管应用场景不同，MLIR TableGen 和 LLVM TableGen 共享相同的底层技术：

1. **相同的 TableGen 语言**：class/def/multiclass/let/foreach 等语法
2. **相同的操作符**：!if/!foreach/!cast/!strconcat 等
3. **相同的类型系统**：bit/bits/int/string/list/dag
4. **相同的解析器**：TGParser.cpp 和 TGLexer.cpp
5. **相同的 Record 系统**：RecordKeeper 管理所有定义

**关键区别在于：**
- LLVM TableGen 的 **Record 表示硬件指令**
- MLIR TableGen 的 **Record 表示 IR 操作**

---

## 总结

MLIR TableGen 和 LLVM TableGen 是同一工具在不同抽象层次的应用：

- **LLVM TableGen**：描述"如何将 IR 转换为机器码"
  - 关注硬件细节（编码、寄存器、调度）
  - 生成代码生成器（CodeGen）

- **MLIR TableGen**：描述"如何定义和变换 IR"
  - 关注语义抽象（操作、类型、接口）
  - 生成 IR 基础设施（Dialect/Op/Interface）

两者互补，共同构成 LLVM 编译器基础设施的完整工具链。
