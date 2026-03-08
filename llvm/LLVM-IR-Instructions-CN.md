# LLVM IR 指令集中文参考

> 来源：`llvm/docs/LangRef.rst`

---

## 一、终结指令（Terminator Instructions）

每个基本块必须以终结指令结尾，用于控制流转移。

| 指令 | 语法 | 说明 |
|------|------|------|
| `ret` | `ret <type> <value>` / `ret void` | 从函数返回，可携带返回值。控制流回到调用方。 |
| `br` | `br i1 <cond>, label <true>, label <false>` / `br label <dest>` | 条件或无条件跳转到另一个基本块。 |
| `switch` | `switch <intty> <val>, label <default> [<val>, label <dest> ...]` | 多路整数分支，类似 C 的 switch，查表跳转到对应标签。 |
| `indirectbr` | `indirectbr ptr <addr>, [label <dest1>, ...]` | 间接跳转，目标地址由运行时指针决定（来自 `blockaddress` 常量）。 |
| `invoke` | `<res> = invoke <fnty> <fn>(<args>) to label <normal> unwind label <except>` | 带异常处理的函数调用。正常返回走 normal 标签，抛出异常走 except 标签。 |
| `callbr` | `<res> = callbr <fnty> <fn>(<args>) to label <fallthrough> [indirect labels]` | 专用于 GCC 风格内联汇编的 `asm goto`，允许汇编跳转到多个标签。 |
| `resume` | `resume <type> <value>` | 恢复传播一个被 `landingpad` 捕获的飞行中异常。 |
| `catchswitch` | `<res> = catchswitch within <parent> [label <handler1>, ...] unwind ...` | 描述一组可能的 catch 处理器，由异常处理个性函数分发。 |
| `catchret` | `catchret from <token> to label <normal>` | 从 `catchpad` 退出，结束异常处理，控制流转到 normal 标签。 |
| `cleanupret` | `cleanupret from <value> unwind label <continue>` / `unwind to caller` | 从 `cleanuppad` 退出，通知个性函数清理已完成。 |
| `unreachable` | `unreachable` | 标记该位置不可达，无定义语义，用于告知优化器此路径不会执行。 |

---

## 二、一元操作（Unary Operations）

| 指令 | 语法 | 说明 |
|------|------|------|
| `fneg` | `<res> = fneg [fast-math flags] <ty> <op>` | 浮点取反，翻转操作数的符号位。支持向量类型。 |

---

## 三、二元操作（Binary Operations）

两个相同类型操作数，产生一个结果值。

| 指令 | 语法 | 说明 |
|------|------|------|
| `add` | `<res> = add [nuw] [nsw] <ty> <op1>, <op2>` | 整数加法。`nuw`=无符号不溢出，`nsw`=有符号不溢出，违反则产生 poison。 |
| `fadd` | `<res> = fadd [fast-math flags] <ty> <op1>, <op2>` | 浮点加法。 |
| `sub` | `<res> = sub [nuw] [nsw] <ty> <op1>, <op2>` | 整数减法。也用于表示取负（`sub 0, %val`）。 |
| `fsub` | `<res> = fsub [fast-math flags] <ty> <op1>, <op2>` | 浮点减法。 |
| `mul` | `<res> = mul [nuw] [nsw] <ty> <op1>, <op2>` | 整数乘法。结果宽度与操作数相同，有符号/无符号均适用。 |
| `fmul` | `<res> = fmul [fast-math flags] <ty> <op1>, <op2>` | 浮点乘法。 |
| `udiv` | `<res> = udiv [exact] <ty> <op1>, <op2>` | 无符号整数除法。除以零为未定义行为。`exact` 表示整除，否则为 poison。 |
| `sdiv` | `<res> = sdiv [exact] <ty> <op1>, <op2>` | 有符号整数除法，向零取整。溢出（如 INT_MIN / -1）为未定义行为。 |
| `fdiv` | `<res> = fdiv [fast-math flags] <ty> <op1>, <op2>` | 浮点除法。 |
| `urem` | `<res> = urem <ty> <op1>, <op2>` | 无符号整数取余。 |
| `srem` | `<res> = srem <ty> <op1>, <op2>` | 有符号整数取余，结果符号与被除数相同（非模运算）。 |
| `frem` | `<res> = frem [fast-math flags] <ty> <op1>, <op2>` | 浮点取余，等价于 `fmod`，结果符号与被除数相同。 |

---

## 四、位运算（Bitwise Binary Operations）

| 指令 | 语法 | 说明 |
|------|------|------|
| `shl` | `<res> = shl [nuw] [nsw] <ty> <op1>, <op2>` | 左移，等价于 `op1 * 2^op2`。移位量 ≥ 位宽则为 poison。 |
| `lshr` | `<res> = lshr [exact] <ty> <op1>, <op2>` | 逻辑右移，高位补零。`exact` 表示移出位全为零，否则为 poison。 |
| `ashr` | `<res> = ashr [exact] <ty> <op1>, <op2>` | 算术右移，高位补符号位。`exact` 表示移出位全为零，否则为 poison。 |
| `and` | `<res> = and <ty> <op1>, <op2>` | 按位与。 |
| `or` | `<res> = or [disjoint] <ty> <op1>, <op2>` | 按位或。`disjoint` 表示两操作数无公共置位位，违反则为 poison。 |
| `xor` | `<res> = xor <ty> <op1>, <op2>` | 按位异或。`xor %V, -1` 等价于按位取反（`~%V`）。 |

---

## 五、向量操作（Vector Operations）

| 指令 | 语法 | 说明 |
|------|------|------|
| `extractelement` | `<res> = extractelement <n x ty> <val>, <ty2> <idx>` | 从向量中提取指定下标的标量元素。越界返回 poison。 |
| `insertelement` | `<res> = insertelement <n x ty> <val>, <ty> <elt>, <ty2> <idx>` | 向向量指定位置插入一个标量元素，返回新向量。 |
| `shufflevector` | `<res> = shufflevector <n x ty> <v1>, <n x ty> <v2>, <m x i32> <mask>` | 按掩码从两个向量中选取元素，构造新向量（排列/混洗）。 |

---

## 六、聚合操作（Aggregate Operations）

| 指令 | 语法 | 说明 |
|------|------|------|
| `extractvalue` | `<res> = extractvalue <aggregate type> <val>, <idx>{, <idx>}*` | 从结构体或数组中提取指定字段/元素的值（编译期常量索引）。 |
| `insertvalue` | `<res> = insertvalue <aggregate type> <val>, <ty> <elt>, <idx>{, <idx>}*` | 向结构体或数组指定位置插入值，返回新的聚合值。 |

---

## 七、内存访问与寻址操作（Memory Access and Addressing Operations）

| 指令 | 语法 | 说明 |
|------|------|------|
| `alloca` | `<res> = alloca [inalloca] <type> [, <ty> <NumElements>] [, align <n>]` | 在当前函数栈帧上分配内存，函数返回时自动释放，返回指针。 |
| `load` | `<res> = load [volatile] <ty>, ptr <ptr> [, align <n>]` | 从内存地址读取值。支持 `volatile`（禁止重排）和 `atomic`（原子加载）。 |
| `store` | `store [volatile] <ty> <val>, ptr <ptr> [, align <n>]` | 将值写入内存地址。支持 `volatile` 和 `atomic`，返回 void。 |
| `fence` | `fence [syncscope(...)] <ordering>` | 内存屏障，在操作间建立 happens-before 关系。支持 acquire/release/acq_rel/seq_cst。 |
| `cmpxchg` | `cmpxchg [weak] [volatile] ptr <ptr>, <ty> <cmp>, <ty> <new> <success-ord> <fail-ord>` | 原子比较并交换：读取内存值，若等于 cmp 则写入 new，返回 `{原值, 成功标志}`。 |
| `atomicrmw` | `atomicrmw [volatile] <op> ptr <ptr>, <ty> <val> <ordering>` | 原子读-修改-写操作。支持 add/sub/and/or/xor/xchg/max/min 等操作，返回旧值。 |
| `getelementptr` | `<res> = getelementptr [inbounds] <ty>, ptr <ptr>, <idx1> [, <idx2> ...]` | 计算聚合类型或数组中元素的地址偏移，不访问内存，仅做指针算术。`inbounds` 越界为 poison。 |

---

## 八、类型转换操作（Conversion Operations）

| 指令 | 语法 | 说明 |
|------|------|------|
| `trunc` | `<res> = trunc [nuw] [nsw] <ty> <val> to <ty2>` | 整数截断，丢弃高位比特，目标类型必须更窄。 |
| `zext` | `<res> = zext [nneg] <ty> <val> to <ty2>` | 零扩展，高位补零，目标类型必须更宽。`nneg` 表示操作数非负。 |
| `sext` | `<res> = sext <ty> <val> to <ty2>` | 符号扩展，高位补符号位，目标类型必须更宽。 |
| `fptrunc` | `<res> = fptrunc [fast-math flags] <ty> <val> to <ty2>` | 浮点截断，从更宽的浮点类型转换为更窄的类型（如 double → float）。 |
| `fpext` | `<res> = fpext [fast-math flags] <ty> <val> to <ty2>` | 浮点扩展，从更窄的浮点类型转换为更宽的类型（如 float → double）。 |
| `fptoui` | `<res> = fptoui <ty> <val> to <ty2>` | 浮点转无符号整数，向零取整。超出范围返回 poison。 |
| `fptosi` | `<res> = fptosi <ty> <val> to <ty2>` | 浮点转有符号整数，向零取整。超出范围返回 poison。 |
| `uitofp` | `<res> = uitofp [nneg] <ty> <val> to <ty2>` | 无符号整数转浮点。无法精确表示时按默认舍入模式处理。 |
| `sitofp` | `<res> = sitofp <ty> <val> to <ty2>` | 有符号整数转浮点。无法精确表示时按默认舍入模式处理。 |
| `ptrtoint` | `<res> = ptrtoint <ty> <val> to <ty2>` | 指针转整数，通过零扩展或截断对齐位宽。 |
| `inttoptr` | `<res> = inttoptr <ty> <val> to <ty2>` | 整数转指针，通过零扩展或截断对��位宽。 |
| `bitcast` | `<res> = bitcast <ty> <val> to <ty2>` | 位重解释转换，不改变任何比特，仅改变类型（no-op cast）。源和目标位宽必须相同。 |
| `addrspacecast` | `<res> = addrspacecast <pty> <val> to <pty2>` | 指针地址空间转换，在不同地址空间（如 GPU 全局/共享内存）之间转换指针。 |

---

## 九、其他操作（Other Operations）

| 指令 | 语法 | 说明 |
|------|------|------|
| `icmp` | `<res> = icmp <cond> <ty> <op1>, <op2>` | 整数比较，返回 `i1`。条件码：`eq/ne/ugt/uge/ult/ule/sgt/sge/slt/sle`。 |
| `fcmp` | `<res> = fcmp [fast-math flags] <cond> <ty> <op1>, <op2>` | 浮点比较，返回 `i1`。支持有序（`o*`）和无序（`u*`）比较，处理 NaN 情况。 |
| `phi` | `<res> = phi <ty> [<val0>, <label0>], [<val1>, <label1>], ...` | SSA φ 节点，根据控制流来自哪个前驱基本块选择对应值。必须位于基本块开头。 |
| `select` | `<res> = select [fast-math flags] i1 <cond>, <ty> <val1>, <ty> <val2>` | 条件选择，无分支地根据条件返回两个值之一，类似三目运算符 `cond ? val1 : val2`。 |
| `freeze` | `<res> = freeze <ty> <val>` | 冻结 undef/poison 值，将其固定为某个任意但确定的值，阻止未定义行为传播。 |
| `call` | `<res> = [tail\|musttail\|notail] call [fast-math flags] [cconv] <fnty> <fn>(<args>) [fn attrs]` | 函数调用。`tail`/`musttail` 提示/强制尾调用优化。 |
| `va_arg` | `<res> = va_arg ptr <arglist>, <ty>` | 从可变参数列表中读取下一个指定类型的参数。 |
| `landingpad` | `<res> = landingpad <ty> [cleanup] [catch <ty> <val>]* [filter <ty> <val>]*` | 异常处理着陆点，必须是 `invoke` 异常目标块的第一条非 phi 指令。 |
| `catchpad` | `<res> = catchpad within <catchswitch> [<args>]` | 在 `catchswitch` 分发后进入的 catch 处理块入口，用于 Windows SEH 风格异常处理。 |
| `cleanuppad` | `<res> = cleanuppad within <parent> [<args>]` | 清理块入口，用于执行析构函数等清理操作（Windows SEH / C++ 析构）。 |

---

## 附：icmp / fcmp 条件码速查

### icmp 条件码

| 条件码 | 含义 |
|--------|------|
| `eq` | 等于 |
| `ne` | 不等于 |
| `ugt` | 无符号大于 |
| `uge` | 无符号大于等于 |
| `ult` | 无符号小于 |
| `ule` | 无符号小于等于 |
| `sgt` | 有符号大于 |
| `sge` | 有符号大于等于 |
| `slt` | 有符号小于 |
| `sle` | 有符号小于等于 |

### fcmp 条件码

| 条件码 | 含义 |
|--------|------|
| `false` | 始终返回 false |
| `oeq` | 有序且相等（两者均非 NaN） |
| `ogt` | 有序且大于 |
| `oge` | 有序且大于等于 |
| `olt` | 有序且小于 |
| `ole` | 有序且小于等于 |
| `one` | 有序且不等于 |
| `ord` | 有序（两者均非 NaN） |
| `ueq` | 无序或相等（任一为 NaN 或相等） |
| `ugt` | 无序或大于 |
| `uge` | 无序或大于等于 |
| `ult` | 无序或小于 |
| `ule` | 无序或小于等于 |
| `une` | 无序或不等于 |
| `uno` | 无序（任一为 NaN） |
| `true` | 始终返回 true |

---

## 附：atomicrmw 支持的操作

| 操作 | 说明 |
|------|------|
| `xchg` | 交换，写入新值返回旧值 |
| `add` | 原子加 |
| `sub` | 原子减 |
| `and` | 原子按位与 |
| `nand` | 原子按位与非 |
| `or` | 原子按位或 |
| `xor` | 原子按位异或 |
| `max` | 有符号最大值 |
| `min` | 有符号最小值 |
| `umax` | 无符号最大值 |
| `umin` | 无符号最小值 |
| `fadd` | 浮点原子加 |
| `fsub` | 浮点原子减 |
| `fmax` | 浮点原子最大值 |
| `fmin` | 浮点原子最小值 |
