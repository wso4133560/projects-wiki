# LLVM TableGen 语法详细参考

> 来源：`llvm/lib/TableGen/TGLexer.h`、`llvm/lib/TableGen/TGParser.cpp`

---

## 目录

1. [词法元素](#1-词法元素)
2. [基本类型](#2-基本类型)
3. [关键字](#3-关键字)
4. [操作符](#4-操作符)
5. [语句结构](#5-语句结构)
6. [表达式](#6-表达式)
7. [模式匹配](#7-模式匹配)
8. [预处理指令](#8-预处理指令)

---

## 1. 词法元素

### 1.1 标识符

**规则：** `[a-zA-Z_][0-9a-zA-Z_]*`

```tablegen
// 合法标识符
MyClass
_private
Register32
X86_64

// 非法标识符
123abc    // 不能以数字开头
my-class  // 不能包含连字符
```

### 1.2 变量名

**规则：** `$[a-zA-Z_][0-9a-zA-Z_]*`

变量名以 `$` 开头，用于 DAG 操作数和模式匹配。

```tablegen
$dst
$src1
$src2
$_temp
```

### 1.3 字面量

#### 整数字面量

```tablegen
42          // 十进制
0x2A        // 十六进制
0b101010    // 二进制
-15         // 负数
```

#### 二进制常量

```tablegen
0b1010      // 4位二进制
{1, 0, 1, 0}  // 位集合表示
```

#### 字符串字面量

```tablegen
"hello"
"add\t{$src2, $dst|$dst, $src2}"  // 包含转义字符
```

#### 代码片段

```tablegen
[{
  // C++ 代码
  return Inst.getOperand(0).getReg();
}]
```

#### 布尔字面量

```tablegen
true
false
```

### 1.4 注释

```tablegen
// 单行注释

/* 多行注释
   可以跨越多行 */
```

### 1.5 符号

| 符号 | Token | 说明 |
|------|-------|------|
| `-` | `minus` | 减号 |
| `+` | `plus` | 加号 |
| `[` | `l_square` | 左方括号 |
| `]` | `r_square` | 右方括号 |
| `{` | `l_brace` | 左花括号 |
| `}` | `r_brace` | 右花括号 |
| `(` | `l_paren` | 左圆括号 |
| `)` | `r_paren` | 右圆括号 |
| `<` | `less` | 小于号 |
| `>` | `greater` | 大于号 |
| `:` | `colon` | 冒号 |
| `;` | `semi` | 分号 |
| `,` | `comma` | 逗号 |
| `.` | `dot` | 点号 |
| `=` | `equal` | 等号 |
| `?` | `question` | 问号 |
| `#` | `paste` | 粘贴符（宏展开） |
| `...` | `dotdotdot` | 省略号 |

---

## 2. 基本类型

### 2.1 类型关键字

| 关键字 | 说明 | 示例 |
|--------|------|------|
| `bit` | 单个位 | `bit hasREX_W = 1;` |
| `bits<n>` | n 位位向量 | `bits<8> Opcode = 0x90;` |
| `int` | 整数 | `int Size = 32;` |
| `string` | 字符串 | `string Name = "ADD";` |
| `list<T>` | 列表 | `list<Register> Regs = [EAX, EBX];` |
| `dag` | DAG 节点 | `dag Ins = (ins GR32:$src);` |
| `code` | 代码片段 | `code Predicate = [{ ... }];` |

### 2.2 类型转换

```tablegen
// 显式类型转换
!cast<Register>(SomeValue)
!cast<Instruction>(MyInst)

// 类型检查
!isa<Register>(Value)
```

---

## 3. 关键字

### 3.1 对象定义关键字

| 关键字 | 说明 | 语法 |
|--------|------|------|
| `class` | 定义类 | `class Name<params> : Parent { ... }` |
| `def` | 定义实例 | `def Name : Class<args> { ... }` |
| `defm` | 实例化 multiclass | `defm Prefix : MultiClass<args>;` |
| `multiclass` | 定义多类 | `multiclass Name<params> { ... }` |
| `defvar` | 定义局部变量 | `defvar name = value;` |
| `defset` | 定义集合 | `defset type name = { ... }` |
| `deftype` | 定义类型别名 | `deftype NewType = OldType;` |

### 3.2 控制流关键字

| 关键字 | 说明 | 语法 |
|--------|------|------|
| `let` | 设置字段值 | `let Field = Value in { ... }` |
| `foreach` | 循环 | `foreach var = list in { ... }` |
| `if` | 条件语句 | `if condition then ... else ...` |
| `then` | if 的 then 分支 | - |
| `else` | if 的 else 分支 | - |
| `assert` | 断言 | `assert condition, "message";` |

### 3.3 其他关键字

| 关键字 | 说明 |
|--------|------|
| `field` | 声明字段 |
| `in` | 用于 let 和 foreach |
| `include` | 包含其他文件 |
| `dump` | 调试输出 |

---

## 4. 操作符

### 4.1 Bang 操作符（! 开头）

TableGen 提供 50+ 个 bang 操作符，用于值计算和转换。

#### 算术操作符

| 操作符 | 语法 | 说明 |
|--------|------|------|
| `!add` | `!add(a, b)` | 加法 |
| `!sub` | `!sub(a, b)` | 减法 |
| `!mul` | `!mul(a, b)` | 乘法 |
| `!div` | `!div(a, b)` | 除法 |
| `!not` | `!not(x)` | 逻辑非 |
| `!log2` | `!log2(x)` | 对数（以2为底） |

#### 位操作符

| 操作符 | 语法 | 说明 |
|--------|------|------|
| `!and` | `!and(a, b)` | 按位与 |
| `!or` | `!or(a, b)` | 按位或 |
| `!xor` | `!xor(a, b)` | 按位异或 |
| `!shl` | `!shl(x, n)` | 左移 |
| `!srl` | `!srl(x, n)` | 逻辑右移 |
| `!sra` | `!sra(x, n)` | 算术右移 |

#### 比较操作符

| 操作符 | 语法 | 说明 |
|--------|------|------|
| `!eq` | `!eq(a, b)` | 等于 |
| `!ne` | `!ne(a, b)` | 不等于 |
| `!lt` | `!lt(a, b)` | 小于 |
| `!le` | `!le(a, b)` | 小于等于 |
| `!gt` | `!gt(a, b)` | 大于 |
| `!ge` | `!ge(a, b)` | 大于等于 |

#### 列表操作符

| 操作符 | 语法 | 说明 |
|--------|------|------|
| `!listconcat` | `!listconcat(l1, l2)` | 列表连接 |
| `!listsplat` | `!listsplat(elem, n)` | 重复元素 n 次 |
| `!listremove` | `!listremove(list, elem)` | 移除元素 |
| `!listflatten` | `!listflatten(list)` | 扁平化嵌套列表 |
| `!head` | `!head(list)` | 列表首元素 |
| `!tail` | `!tail(list)` | 列表尾部 |
| `!size` | `!size(list)` | 列表大小 |
| `!empty` | `!empty(list)` | 是否为空 |

#### 字符串操作符

| 操作符 | 语法 | 说明 |
|--------|------|------|
| `!strconcat` | `!strconcat(s1, s2)` | 字符串连接 |
| `!substr` | `!substr(str, start, len)` | 子串提取 |
| `!find` | `!find(str, pattern)` | 查找子串位置 |
| `!tolower` | `!tolower(str)` | 转小写 |
| `!toupper` | `!toupper(str)` | 转大写 |
| `!interleave` | `!interleave(list, sep)` | 用分隔符连接列表 |
| `!repr` | `!repr(value)` | 获取值的字符串表示 |

#### 类型操作符

| 操作符 | 语法 | 说明 |
|--------|------|------|
| `!cast<T>` | `!cast<Type>(value)` | 类型转换 |
| `!isa<T>` | `!isa<Type>(value)` | 类型检查 |
| `!exists` | `!exists(value)` | 检查是否存在 |
| `!initialized` | `!initialized(value)` | 检查是否已初始化 |

#### 控制流操作符

| 操作符 | 语法 | 说明 |
|--------|------|------|
| `!if` | `!if(cond, then, else)` | 条件表达式 |
| `!cond` | `!cond(c1:v1, c2:v2, ...)` | 多路条件 |
| `!foreach` | `!foreach(var, list, body)` | 遍历列表 |
| `!filter` | `!filter(var, list, pred)` | 过滤列表 |
| `!foldl` | `!foldl(init, list, acc, var, body)` | 折叠列表 |

#### DAG 操作符

| 操作符 | 语法 | 说明 |
|--------|------|------|
| `!dag` | `!dag(op, args, names)` | 构造 DAG |
| `!getdagop` | `!getdagop(dag)` | 获取 DAG 操作符 |
| `!setdagop` | `!setdagop(dag, op)` | 设置 DAG 操作符 |
| `!getdagarg` | `!getdagarg(dag, i)` | 获取第 i 个参数 |
| `!getdagname` | `!getdagname(dag, i)` | 获取第 i 个参数名 |
| `!setdagarg` | `!setdagarg(dag, i, val)` | 设置第 i 个参数 |
| `!setdagname` | `!setdagname(dag, i, name)` | 设置第 i 个参数名 |

#### 其他操作符

| 操作符 | 语法 | 说明 |
|--------|------|------|
| `!subst` | `!subst(old, new, str)` | 字符串替换 |
| `!range` | `!range(start, end, step)` | 生成范围 |
| `!match` | `!match(str, pattern)` | 正则匹配 |
| `!instances` | `!instances(class)` | 获取类的所有实例 |
| `!concat` | `!concat(a, b)` | 通用连接 |

---

## 5. 语句结构

### 5.1 class 定义

```tablegen
// 基本语法
class ClassName<TemplateParams> : ParentClass1, ParentClass2 {
  // 字段定义
  type fieldName = defaultValue;

  // 模板参数使用
  bits<8> Opcode = opcodeParam;
}

// 示例
class Register<string n> {
  string Name = n;
  string Namespace = "";
  bits<16> HWEncoding = 0;
  list<Register> SubRegs = [];
}

// 多重继承
class X86Reg<string n, bits<16> enc> : Register<n>, Printable {
  let Namespace = "X86";
  let HWEncoding = enc;
}
```

### 5.2 def 定义

```tablegen
// 基本语法
def InstanceName : ClassName<Args> {
  // 覆盖字段值
  let fieldName = newValue;
}

// 示例
def EAX : X86Reg<"eax", 0> {
  let SubRegs = [AX];
}

// 匿名定义
def : SomeClass<args> {
  // 不需要名称的实例
}
```

### 5.3 multiclass 定义

```tablegen
// 基本语法
multiclass MultiClassName<TemplateParams> {
  def Suffix1 : Class1<...>;
  def Suffix2 : Class2<...>;
  // 可以包含多个 def
}

// 示例
multiclass BinOp<bits<8> opc, string mnemonic> {
  def 8rr  : Inst<opc, (outs GR8:$dst),  (ins GR8:$src1,  GR8:$src2)>;
  def 16rr : Inst<opc, (outs GR16:$dst), (ins GR16:$src1, GR16:$src2)>;
  def 32rr : Inst<opc, (outs GR32:$dst), (ins GR32:$src1, GR32:$src2)>;
}

// 实例化
defm ADD : BinOp<0x01, "add">;
// 生成: ADD8rr, ADD16rr, ADD32rr
```

### 5.4 let 语句

```tablegen
// 单个字段
let Field = Value in {
  def Inst1 : ...;
  def Inst2 : ...;
}

// 多个字段
let Field1 = Value1, Field2 = Value2 in {
  def Inst : ...;
}

// 嵌套 let
let Namespace = "X86" in {
  let Defs = [EFLAGS] in {
    def ADD32rr : ...;
  }
}

// 字段访问
let Inst.Opcode = 0x90 in {
  def NOP : ...;
}
```

### 5.5 foreach 循环

```tablegen
// 基本语法
foreach var = list in {
  def NAME#var : ...;
}

// 示例：生成多个寄存器
foreach i = [0, 1, 2, 3] in {
  def R#i : Register<"r"#i, i>;
}
// 生成: R0, R1, R2, R3

// 嵌套循环
foreach size = [8, 16, 32, 64] in {
  foreach op = ["add", "sub", "mul"] in {
    def op#size : ...;
  }
}
```

### 5.6 defvar 局部变量

```tablegen
// 定义局部变量
defvar RegList = [EAX, EBX, ECX, EDX];
defvar NumRegs = !size(RegList);

// 使用变量
def MyClass {
  list<Register> Regs = RegList;
  int Count = NumRegs;
}
```

### 5.7 defset 集合定义

```tablegen
// 定义集合
defset list<Register> AllRegs = {
  def EAX : Register<"eax", 0>;
  def EBX : Register<"ebx", 3>;
  def ECX : Register<"ecx", 1>;
}

// AllRegs 自动包含所有定义的寄存器
```

### 5.8 assert 断言

```tablegen
// 编译时断言
assert !eq(Size, 32), "Size must be 32";
assert !ge(NumRegs, 4), "At least 4 registers required";

// 在 class 中使用
class MyClass<int n> {
  assert !gt(n, 0), "n must be positive";
  int Value = n;
}
```

---

## 6. 表达式

### 6.1 字段访问

```tablegen
// 直接访问
Inst.Opcode
Reg.HWEncoding

// 链式访问
Inst.Parent.Field
```

### 6.2 列表表达式

```tablegen
// 列表字面量
[1, 2, 3, 4]
[EAX, EBX, ECX, EDX]

// 列表连接
!listconcat([1, 2], [3, 4])  // [1, 2, 3, 4]

// 列表索引（通过 !listelem）
!listelem([10, 20, 30], 1)  // 20

// 列表切片
!listslice([1, 2, 3, 4, 5], 1, 3)  // [2, 3, 4]

// 列表生成
!range(0, 10, 2)  // [0, 2, 4, 6, 8]
```

### 6.3 条件表达式

```tablegen
// !if 表达式
!if(!eq(Size, 32), GR32, GR64)

// !cond 多路条件
!cond(!eq(Size, 8):  GR8,
      !eq(Size, 16): GR16,
      !eq(Size, 32): GR32,
      true:          GR64)
```

### 6.4 字符串表达式

```tablegen
// 字符串连接
!strconcat("add", "l")  // "addl"

// 字符串插值（通过 # 操作符）
"R"#i  // 如果 i=5，结果为 "R5"

// 子串
!substr("hello", 1, 3)  // "ell"

// 大小写转换
!tolower("HELLO")  // "hello"
!toupper("world")  // "WORLD"
```

### 6.5 DAG 表达式

```tablegen
// DAG 构造
(add GR32:$src1, GR32:$src2)

// 带名称的 DAG
(outs GR32:$dst)
(ins GR32:$src1, GR32:$src2)

// 嵌套 DAG
(set GR32:$dst, (add GR32:$src1, GR32:$src2))

// DAG 操作
!getdagop((add $a, $b))  // 返回 add
!getdagarg((add $a, $b), 0)  // 返回 $a
```


---

## 7. 模式匹配

### 7.1 基本模式

```tablegen
// 简单模式
[(set GR32:$dst, GR32:$src)]

// 算术操作模式
[(set GR32:$dst, (add GR32:$src1, GR32:$src2))]

// 内存操作模式
[(set GR32:$dst, (load addr:$src))]
[(store GR32:$src, addr:$dst)]
```

### 7.2 复杂模式

```tablegen
// 带立即数的模式
[(set GR32:$dst, (add GR32:$src, imm:$imm))]

// 嵌套模式
[(set GR32:$dst, (add (mul GR32:$src1, GR32:$src2), GR32:$src3))]

// 条件模式
[(set GR32:$dst, (select i1:$cond, GR32:$true, GR32:$false))]
```

### 7.3 模式片段

```tablegen
// 定义模式片段
def addr : ComplexPattern<iPTR, 5, "SelectAddr", [], []>;
def i32imm : Operand<i32>;

// 使用模式片段
def MOV32rm : I<0x8B, MRMSrcMem, (outs GR32:$dst), (ins i32mem:$src),
                "mov{l}\t{$src, $dst|$dst, $src}",
                [(set GR32:$dst, (load addr:$src))]>;
```

### 7.4 模式约束

```tablegen
// 类型约束
def SDTIntBinOp : SDTypeProfile<1, 2, [
  SDTCisSameAs<0, 1>,    // 结果类型与第一个操作数相同
  SDTCisSameAs<0, 2>,    // 结果类型与第二个操作数相同
  SDTCisInt<0>           // 结果必须是整数类型
]>;

// 节点属性
def add : SDNode<"ISD::ADD", SDTIntBinOp, [SDNPCommutative]>;
```

---

## 8. 预处理指令

### 8.1 条件编译

```tablegen
// #ifdef
#ifdef MACRO_NAME
  def Feature1 : ...;
#endif

// #ifndef
#ifndef MACRO_NAME
  def Feature2 : ...;
#endif

// #else
#ifdef ENABLE_FEATURE
  def FeatureA : ...;
#else
  def FeatureB : ...;
#endif
```

### 8.2 宏定义

```tablegen
// 命令行定义宏
// llvm-tblgen -DENABLE_AVX512 ...

// 在代码中检查
#ifdef ENABLE_AVX512
  defm VADD : AVX512_BinOp<...>;
#endif
```

### 8.3 文件包含

```tablegen
// 包含其他 .td 文件
include "X86RegisterInfo.td"
include "X86InstrFormats.td"
```

---

## 9. 高级特性

### 9.1 模板特化

```tablegen
// 通用模板
class BinOp<bits<8> opc, string mnemonic, RegisterClass RC> {
  bits<8> Opcode = opc;
  string Mnemonic = mnemonic;
  RegisterClass RegClass = RC;
}

// 特化
def ADD32rr : BinOp<0x01, "add", GR32>;
def ADD64rr : BinOp<0x01, "add", GR64>;
```

### 9.2 字段覆盖

```tablegen
class Base {
  int Value = 10;
  string Name = "base";
}

class Derived : Base {
  let Value = 20;        // 覆盖父类字段
  let Name = "derived";
}
```

### 9.3 名称拼接

```tablegen
// 使用 # 操作符拼接名称
foreach i = [0, 1, 2, 3] in {
  def R#i : Register<"r"#i, i>;
}
// 生成: R0, R1, R2, R3

// 在 multiclass 中使用 NAME
multiclass MyClass<string suffix> {
  def NAME#suffix : ...;
}

defm Inst : MyClass<"_v1">;
// 生成: Inst_v1
```

### 9.4 匿名记录

```tablegen
// 不需要名称的定义
def : Pat<(add GR32:$src1, GR32:$src2),
          (ADD32rr GR32:$src1, GR32:$src2)>;

// 用于模式匹配规则
def : InstAlias<"mov $src, $dst", (MOV32rr GR32:$dst, GR32:$src)>;
```

---

## 10. 常见模式和惯用法

### 10.1 寄存器定义模式

```tablegen
// 定义寄存器层次结构
class X86Reg<string n, bits<16> enc, list<Register> subregs = []> 
  : Register<n> {
  let Namespace = "X86";
  let HWEncoding = enc;
  let SubRegs = subregs;
}

// 8位寄存器
def AL : X86Reg<"al", 0>;

// 16位寄存器（包含8位子寄存器）
let SubRegIndices = [sub_8bit] in
  def AX : X86Reg<"ax", 0, [AL]>;

// 32位寄存器（包含16位子寄存器）
let SubRegIndices = [sub_16bit] in
  def EAX : X86Reg<"eax", 0, [AX]>;
```

### 10.2 指令定义模式

```tablegen
// 定义指令基类
class Inst<bits<8> opc, dag outs, dag ins, string asm, list<dag> pattern>
  : Instruction {
  bits<8> Opcode = opc;
  dag OutOperandList = outs;
  dag InOperandList = ins;
  string AsmString = asm;
  list<dag> Pattern = pattern;
}

// 定义具体指令
def ADD32rr : Inst<0x01, (outs GR32:$dst), (ins GR32:$src1, GR32:$src2),
                   "add{l}\t{$src2, $dst|$dst, $src2}",
                   [(set GR32:$dst, (add GR32:$src1, GR32:$src2))]>;
```

### 10.3 multiclass 实例化模式

```tablegen
// 定义 multiclass
multiclass ArithBinOp<bits<8> opc, string mnemonic, SDNode node> {
  def 8rr  : BinOpRR<opc, mnemonic, Xi8, node>;
  def 16rr : BinOpRR<opc, mnemonic, Xi16, node>;
  def 32rr : BinOpRR<opc, mnemonic, Xi32, node>;
  def 64rr : BinOpRR<opc, mnemonic, Xi64, node>;
}

// 批量实例化
defm ADD : ArithBinOp<0x00, "add", add>;
defm SUB : ArithBinOp<0x28, "sub", sub>;
defm AND : ArithBinOp<0x20, "and", and>;
defm OR  : ArithBinOp<0x08, "or",  or>;
defm XOR : ArithBinOp<0x30, "xor", xor>;
```

### 10.4 条件字段设置

```tablegen
// 根据条件设置字段
class MyInst<bit is64bit> {
  let Opcode = !if(is64bit, 0x48, 0x40);
  let Size = !if(is64bit, 8, 4);
}

// 使用 !cond 多路选择
class SizedInst<int size> {
  bits<8> Opcode = !cond(
    !eq(size, 8):  0x88,
    !eq(size, 16): 0x89,
    !eq(size, 32): 0x8B,
    true:          0x8D
  );
}
```

### 10.5 列表生成模式

```tablegen
// 使用 !foreach 生成列表
defvar RegList = !foreach(i, [0, 1, 2, 3], 
                          !cast<Register>("R"#i));

// 使用 !filter 过滤列表
defvar EvenRegs = !filter(r, AllRegs, 
                          !eq(!and(!cast<int>(r.HWEncoding), 1), 0));

// 使用 !foldl 累积计算
defvar TotalSize = !foldl(0, RegList, acc, reg, 
                          !add(acc, reg.Size));
```

---

## 11. 调试技巧

### 11.1 dump 语句

```tablegen
// 输出调试信息
dump "Debug: Value = " # !repr(SomeValue);

// 输出变量值
defvar x = 42;
dump "x = " # !repr(x);
```

### 11.2 assert 断言

```tablegen
// 编译时检查
assert !eq(!size(RegList), 4), "Expected 4 registers";
assert !gt(Opcode, 0), "Opcode must be positive";
```

### 11.3 类型检查

```tablegen
// 检查类型
assert !isa<Register>(Value), "Value must be a Register";

// 安全类型转换
defvar Reg = !if(!isa<Register>(Value), 
                 !cast<Register>(Value), 
                 DefaultReg);
```

---

## 12. 最佳实践

### 12.1 命名约定

- **类名**：使用 PascalCase（如 `RegisterClass`）
- **字段名**：使用 PascalCase（如 `HWEncoding`）
- **实例名**：使用 UPPERCASE 或 PascalCase（如 `EAX`、`ADD32rr`）
- **变量名**：使用 camelCase（如 `$dst`、`$src1`）

### 12.2 代码组织

```tablegen
// 1. 包含文件
include "Target.td"

// 2. 类型定义
deftype MyType = ...;

// 3. 基类定义
class BaseClass { ... }

// 4. 派生类定义
class DerivedClass : BaseClass { ... }

// 5. multiclass 定义
multiclass MyMultiClass { ... }

// 6. 实例定义
def Instance1 : ...;
def Instance2 : ...;

// 7. multiclass 实例化
defm Prefix : MyMultiClass<...>;
```

### 12.3 避免重复

```tablegen
// 不好：重复代码
def ADD8rr  : Inst<0x00, ...>;
def ADD16rr : Inst<0x01, ...>;
def ADD32rr : Inst<0x03, ...>;

// 好：使用 multiclass
multiclass ADD<bits<8> base_opc> {
  def 8rr  : Inst<base_opc, ...>;
  def 16rr : Inst<!add(base_opc, 1), ...>;
  def 32rr : Inst<!add(base_opc, 3), ...>;
}
defm ADD : ADD<0x00>;
```

### 12.4 文档注释

```tablegen
// 为类添加注释
/// Register class for 32-bit general purpose registers.
/// Includes EAX, EBX, ECX, EDX, ESI, EDI, EBP, ESP.
class GR32 : RegisterClass<"X86", [i32], 32, ...> {
  ...
}

// 为字段添加注释
class Instruction {
  /// The opcode byte for this instruction
  bits<8> Opcode = 0;
  
  /// The assembly string for this instruction
  string AsmString = "";
}
```

---

## 13. 完整示例

### 13.1 简单寄存器定义

```tablegen
// 定义寄存器基类
class MyReg<string n, bits<4> enc> : Register<n> {
  let Namespace = "MyTarget";
  let HWEncoding = enc;
}

// 定义具体寄存器
def R0 : MyReg<"r0", 0>;
def R1 : MyReg<"r1", 1>;
def R2 : MyReg<"r2", 2>;
def R3 : MyReg<"r3", 3>;

// 定义寄存器类
def GPR : RegisterClass<"MyTarget", [i32], 32, (sequence "R%u", 0, 3)>;
```

### 13.2 完整指令定义

```tablegen
// 定义指令格式
class InstFormat<bits<4> val> {
  bits<4> Value = val;
}
def FmtRR : InstFormat<0>;
def FmtRI : InstFormat<1>;

// 定义指令基类
class Inst<bits<8> opc, InstFormat fmt, dag outs, dag ins, 
           string asm, list<dag> pattern> : Instruction {
  let Namespace = "MyTarget";
  bits<8> Opcode = opc;
  InstFormat Format = fmt;
  dag OutOperandList = outs;
  dag InOperandList = ins;
  string AsmString = asm;
  list<dag> Pattern = pattern;
}

// 定义 ADD 指令
def ADDrr : Inst<0x01, FmtRR, (outs GPR:$dst), (ins GPR:$src1, GPR:$src2),
                 "add $dst, $src1, $src2",
                 [(set GPR:$dst, (add GPR:$src1, GPR:$src2))]>;

def ADDri : Inst<0x02, FmtRI, (outs GPR:$dst), (ins GPR:$src, i32imm:$imm),
                 "add $dst, $src, $imm",
                 [(set GPR:$dst, (add GPR:$src, imm:$imm))]>;
```

### 13.3 使用 multiclass 的完整示例

```tablegen
// 定义 multiclass
multiclass BinOpPat<bits<8> opc, string mnemonic, SDNode node> {
  // 寄存器-寄存器版本
  def rr : Inst<opc, FmtRR, (outs GPR:$dst), (ins GPR:$src1, GPR:$src2),
                !strconcat(mnemonic, " $dst, $src1, $src2"),
                [(set GPR:$dst, (node GPR:$src1, GPR:$src2))]>;
  
  // 寄存器-立即数版本
  def ri : Inst<!add(opc, 1), FmtRI, (outs GPR:$dst), (ins GPR:$src, i32imm:$imm),
                !strconcat(mnemonic, " $dst, $src, $imm"),
                [(set GPR:$dst, (node GPR:$src, imm:$imm))]>;
}

// 实例化所有算术指令
defm ADD : BinOpPat<0x01, "add", add>;
defm SUB : BinOpPat<0x03, "sub", sub>;
defm AND : BinOpPat<0x05, "and", and>;
defm OR  : BinOpPat<0x07, "or",  or>;
defm XOR : BinOpPat<0x09, "xor", xor>;
```

---

## 总结

TableGen 语法提供了丰富的特性来描述目标架构：

1. **类型系统**：7 种基础类型，支持复杂的类型组合
2. **关键字**：10+ 个对象定义和控制流关键字
3. **操作符**：50+ 个 bang 操作符，覆盖算术、逻辑、字符串、列表、DAG 等操作
4. **语句结构**：class、def、multiclass、let、foreach 等灵活的定义方式
5. **模式匹配**：强大的 DAG 模式匹配系统，用于指令选择
6. **预处理**：条件编译和宏定义支持

通过这些语法元素的组合，TableGen 能够以声明式的方式描述复杂的目标架构，自动生成大量重复性的 C++ 代码，极大地提高了 LLVM 后端开发的效率。
