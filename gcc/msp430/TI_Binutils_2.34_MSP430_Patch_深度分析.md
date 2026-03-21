# TI Binutils 2.34 MSP430 Patch 深度分析

> 基准版本：Binutils 2.34（上游官方）
> 目标版本：TI msp430-gcc-9.3.1.11-source-full 中的 Binutils
> 分析范围：全量 diff（bfd / gas / ld / binutils / include）

---

## 概览

TI 对 Binutils 2.34 的修改规模远超 GDB，涉及 **2 个全新特性**、**多个重要 Bug 修复** 和 **大量 MSP430 支持扩展**。

| 改动类别 | 文件数 | 核心内容 |
|---------|--------|---------|
| **新特性 A**：符号元信息 `.symtab_meta` | 10+ | 全新 ELF section 类型 + 汇编指令 + 工具链集成 |
| **新特性 B**：ULEB128 重定位 | 5 | 跨 relaxation 的 DWARF 信息精确重定位 |
| MSP430 Relaxation 引擎优化 | 1 | JMP↔BR(A) 互转逻辑修复与增强 |
| 链接器脚本与 .em 重构 | 4 | orphan 处理、either 区间 shuffle、数组对齐检查 |
| BFD 核心修复 | 4 | 段布局 Bug、压缩节过滤、GC 修复 |
| 汇编器设备列表更新 | 1 | 新增 40+ MSP430FR 型号 |
| 工具集成 | 2 | readelf、objcopy 支持新特性 |

---

## 一、新特性 A：符号元信息机制（Symbol Meta-Information）

### 1.1 背景与动机

MSP430 嵌入式开发中有两类常见需求：
1. 某些函数/变量需要**固定放置在特定地址**（如中断向量、硬件寄存器映射的初始值块）
2. 被 `__attribute__((used))` 或 `volatile` 标记的符号需要在 `--gc-sections` 时**强制保留**

上游 Binutils 没有统一机制在 ELF 文件中携带这类信息。TI 为此设计了完整的 `.symtab_meta`（Section Type `SHT_GNU_SYMTAB_META`）基础设施。

### 1.2 新增 ELF Section 类型

**`include/elf/common.h`**：
```c
#define SHT_GNU_SYMTAB_META 0x6ffffff8   /* .symtab_meta section */
```

同时新增 Symbol Meta-Information（SMI）编解码宏，与重定位信息复用同一编码格式：
```c
#define ELF32_SMI_SYM(i)   ELF32_R_SYM(i)     // 高24位：符号索引
#define ELF32_SMI_KIND(i)  ELF32_R_TYPE(i)    // 低8位：元信息种类
```

### 1.3 数据结构设计

**`include/elf/internal.h`**：新增内部表示：
```c
typedef struct elf_internal_sym_meta_info {
  unsigned char kind;   // SMK_RETAIN 或 SMK_LOCATION
  bfd_vma value;        // 保留标志 或 目标地址
  bfd_vma index;        // 输出符号表中的索引（链接期填充）
} Elf_Internal_SymMetaInfo;
```

**`bfd/elf-bfd.h`**：完整的元信息种类定义：
```c
typedef enum {
  SMK_NONE     = 0,
  SMK_RETAIN   = 1,   // 阻止 --gc-sections 回收该符号及其所在节
  SMK_LOCATION = 2,   // 要求链接器将该符号的节放置在指定绝对地址
} bfd_elf_metasym_kind;
```

元信息哈希表（以符号名为键，支持同名不同符号的链表）：
```c
struct bfd_elf_metasym_hash_entry {
  struct bfd_hash_entry root;  // 基类
  asymbol *sym;                // 对应的 BFD 符号
  unsigned int bfd_id;         // 来源 BFD 的 ID（区分同名局部符号）
  Elf_Internal_Sym *elfsym;   // 指向 symtab_hdr->contents 中的条目（relaxation 时同步更新）
  Elf_Internal_SymMetaInfo msym;
  struct bfd_elf_metasym_hash_entry *next;  // 同名不同符号链表
};
```

遍历宏：
```c
#define FOR_MSYM_IN_MSYM_HASH_TABLE(ABFD, I, HASH_ENTRY, MSYM_HASH_ENTRY) \
  ...  // 三层 for 循环：桶 → 哈希链 → 同名链表
```

### 1.4 汇编器指令 `.sym_meta_info`

**`gas/config/obj-elf.c`**：新增伪指令：
```
.sym_meta_info NAME, KIND, VALUE
```

用法示例（来自测试文件 `ld-msp430-elf/metasym-loc-1.s`）：
```asm
loc808_data:
    .short 2056
    .sym_meta_info loc808_data, SMK_LOCATION, 0x808    ; 要求放到 0x808

loc9000_text:
    NOP
    RET
    .sym_meta_info loc9000_text, SMK_LOCATION, 0x9000  ; 代码放到 0x9000
```

解析逻辑（`obj_elf_sym_meta_info()`）：
1. 读取符号名，验证符号已声明
2. 读取 kind 字符串（`SMK_RETAIN` 或 `SMK_LOCATION`）
3. 读取数值（hex/dec 均可）
4. 调用 `bfd_elf_add_metasym()` 将条目写入 BFD 的哈希表

**`gas/config/tc-msp430.c`**：最终汇编完成后，若存在 `SMK_LOCATION` 元信息，自动定义符号 `__crt0_run_smi_location_init_array`，触发 crt0 中特殊的初始化路径（按指定地址搬移数据）。

### 1.5 BFD 层实现

**`bfd/elf.c`**：
- `bfd_elf_slurp_symtab_meta()`：从 `.symtab_meta` 节读取元信息，填充哈希表
- `bfd_elf_write_symtab_meta()`：链接完成后将哈希表序列化写入输出文件

`.symtab_meta` 不会被 BFD 创建为普通的 BFD section，而是专门由 `elf_symtabmeta` 索引管理，防止被自动复制到输出文件。

**`bfd/elflink.c`**：三个关键介入点：

1. **输入 BFD 读取阶段**（`_bfd_elf_link_read_relocs_from_section` 附近）：读入 `.symtab_meta`，解析为内部哈希表。

2. **GC 标记阶段**：对所有 `SMK_RETAIN` 的元符号，强制设置 `h->mark = 1` 和 `SEC_KEEP`：
```c
if (msym->msym.kind == SMK_RETAIN)
  {
    h->root.type = bfd_link_hash_defined;
    h->mark = 1;
    h->root.u.def.section->flags |= SEC_KEEP;
  }
```

3. **输出符号表索引更新**（`_bfd_elf_link_adjust_metasym()`）：在 relaxation 修改符号值时，同步更新元信息中的 `index` 字段，确保最终写出时有正确的符号表索引。

### 1.6 工具集成

**`binutils/objcopy.c`**：ELF→ELF 复制时，自动读取并复制 `.symtab_meta`：
```c
if (!bfd_elf_slurp_symtab_meta (ibfd, NULL, isympp, symcount, NULL)
    || !bfd_elf_copy_metasyms (obfd, ibfd))
  { /* error */ }
```

**`binutils/readelf.c`**：
- `SHT_GNU_SYMTAB_META` 节类型显示为 `"GNU_SYMTAB_META"`
- 显示 `.symtab_meta` 节中每个条目的 kind/value/index

---

## 二、新特性 B：ULEB128 差值重定位

### 2.1 背景

DWARF 调试信息大量使用 `.uleb128 label1 - label0` 来表示长度/偏移量，例如：
```asm
.Ldebug_line_start:
  ...
.uleb128 .Lend - .Lstart   // 表示一段区间的字节长度
.Lend:
```

在 MSP430 的链接时 relaxation（指令压缩/扩展）过程中，`.Lstart` 和 `.Lend` 之间的指令数量可能改变，导致这个差值在汇编时计算出的值到链接完成时已经不正确，从而产生错误的 DWARF 信息。

### 2.2 新增重定位类型

**`include/elf/msp430.h`**：
```c
RELOC_NUMBER (R_MSP430_SET_ULEB128,  11)   /* GNU only */
RELOC_NUMBER (R_MSP430_SUB_ULEB128,  12)   /* GNU only */
RELOC_NUMBER (R_MSP430X_SET_ULEB128, 22)   /* GNU only */
RELOC_NUMBER (R_MSP430X_SUB_ULEB128, 23)   /* GNU only */
```

**`bfd/reloc.c`**：对应的 `BFD_RELOC_MSP430_SET_ULEB128` 和 `BFD_RELOC_MSP430_SUB_ULEB128`。

### 2.3 汇编器插入 Fixup（`gas/config/tc-msp430.c`）

新函数 `msp430_insert_uleb128_fixes()`，在 `md_finish()` 中调用：
```c
// 对每个 .uleb128 expr1 - expr2 的 frag，插入两个 fixup：
fix_new_exp (fragP, fragP->fr_fix, 0,
             exp_dup_with_op_symbol, 0, BFD_RELOC_MSP430_SUB_ULEB128);
fix_new_exp (fragP, fragP->fr_fix, 0,
             exp_dup_with_add_symbol, 0, BFD_RELOC_MSP430_SET_ULEB128);
```

**先写 SUB（减数），再写 SET（被减数）**，这样 BFD 在处理时会先记录基准值，最后用 SET 写出正确结果。

### 2.4 BFD Relocation 处理（`bfd/elf32-msp430.c`）

**`write_uleb128()`**：新增辅助函数，将一个无符号整数以 ULEB128 格式写入内存，返回写后指针。

**处理逻辑**（在 `msp430_elf_perform_relocation()` 中）：
```c
case R_MSP430_SET_ULEB128:
  relocation += rel->r_addend;  // 加上被减数
  // fall through：
  // 1. 用 _bfd_read_unsigned_leb128 读出当前位置 ULEB128 的字节长度
  // 2. 清零该区域（保持长度不变，仅清内容）
  // 3. 用 write_uleb128 写入最终差值
  // 4. 若新值编码比原来短，设置最高位 0x80 填充（不缩短长度）
  // 5. 若新值编码比原来长，报错
```

**HOWTO 条目**：使用 `msp430_elf_ignore_reloc` 作为 special handler（实际处理在 perform_relocation 中），`bitsize=0` 表示长度可变。

### 2.5 readelf 支持（`binutils/readelf.c`）

原来 MSP430 重定位处理假设所有重定位大小为 2 字节（或 R_MSP430_32 为 4 字节）。现在增加对 ULEB128 重定位的变长处理：
```c
case 11: /* R_MSP430_SET_ULEB128 */
case 22: /* R_MSP430X_SET_ULEB128 */
    read_leb128 (start + reloc->r_offset, end, FALSE, &reloc_size, &leb_ret);
    break;
```

---

## 三、MSP430 Relaxation 引擎优化（`bfd/elf32-msp430.c`）

### 3.1 调试输出开关

```c
static bfd_boolean debug_relocs = 0;
```

全局调试开关，在 relaxation 的关键路径（删除字节、调整符号、growing/shrinking 决策）插入了大量 `printf`，方便排查 relaxation 错误。

### 3.2 函数泛化：`add_two_words` → `add_words`

原函数 `msp430_elf_relax_add_two_words()` 固定插入 2 个 word（4 字节）。

新函数 `msp430_elf_relax_add_words(num_words, word1, word2)` 支持插入 1 或 2 个 word：
```c
// 核心改动
int num_bytes = num_words * 2;
contents = bfd_realloc (contents, sec_end + num_bytes);
memmove (contents + addr + num_bytes, contents + addr, sec_end - addr);
if (num_words == 2)
    bfd_put_16 (abfd, word2, contents + addr + 2);
sec->size += num_bytes;
```

### 3.3 JMP → BR(A) Growing 修复

**场景**：条件跳转 `JMP label` 超出 10-bit 范围，需要扩展为：
```
J<cond> +2       ; 跳过下面的无条件跳转
BRA/BR label     ; 无条件跳到目标
```

**原始问题**：
```c
// 原代码：无论何种情况，总是插入 2 words（BRA opcode + 地址）
contents = msp430_elf_relax_add_two_words(abfd, sec, irel->r_offset + 2, 0x0080, 0x0000);
```

**修复后**：当 opcode 已经是 `0x0080`（即从 JMP 直接转来的情况，opcode 已写入），只需插入 **1 word**（地址）：
```c
if (opcode == 0x0080)  // JMP → BRA：BRA opcode 已存在，只加地址 word
    contents = msp430_elf_relax_add_words(abfd, sec, irel->r_offset + 2, 1, 0x0000, 0);
else                   // 其他条件跳转：需要完整的 2 words
    contents = msp430_elf_relax_add_words(abfd, sec, irel->r_offset + 2, 2, 0x0080, 0x0000);
```

同理，MSP430（非 X）的 `BR` 情况也做了对应修复。

### 3.4 BR → JMP Shrinking 重大改进

**场景**：先前 growing 生成了 `J<cond> +2; BR[A] label`，之后如果 label 距离近了，应该 shrink 回去。

**原始问题**：原代码只能将 BR 还原为 JMP，节省 2 字节，但忽略了前面的 `J<cond> +2` 现在可以直接反转并删除整个 BR，节省 **4 字节**：
```
原始：  J<cond> +2; BRA label
shrink：JMP label          ; 若 label 近，直接反转 J<cond> 并删 BRA，节省 4 字节
```

**新增的反转表**：
```c
case 0x3802: opcode = 0x3401; break;  // Jl  → Jge（反转）
case 0x3402: opcode = 0x3801; break;  // Jge → Jl
case 0x2c02: opcode = 0x2801; break;  // Jhs → Jlo
case 0x2802: opcode = 0x2c01; break;  // Jlo → Jhs
case 0x2402: opcode = 0x2001; break;  // Jeq → Jne
case 0x2002: opcode = 0x2401; break;  // Jne → Jeq
case 0x3002: continue;                // Jn 无法反转，跳过
default:     opcode = 0x3c00; break;  // 普通情况，直接 JMP
```

若检测到可反转的 `J<cond> +2`，则：
1. 写入反转后的 `J<cond>`（opcode）
2. 将 `r_offset` 向前移 4
3. 删除 4 字节（整个 BR 指令）

### 3.5 新增 `0x3c00` (JMP) 作为 growing case

原代码中，`msp430_elf_relax_adjust_locals()` 只处理条件跳转的 growing，现在也支持 `JMP`（opcode `0x3c00`）的 growing，允许将超范围的 JMP 扩展为 BRA/BR。

---

## 四、链接器（ld/）重构与修复

### 4.1 `msp430.em`：orphan section 处理重构

#### 新增：中断向量 orphan 警告
```c
static bfd_boolean
report_removed_interrupt_vector_section (asection *s)
{
  const char *vect_str = "__interrupt_vector_";
  if (strncmp (s->name, vect_str, strlen (vect_str)) == 0)
    {
      einfo ("%P: warning: linker script does not recognize interrupt "
             "vector section name %s\n", s->name);
      return TRUE;
    }
  return FALSE;
}
```

**背景**：GCC 允许用户在 `interrupt` 属性中传任意字符串，如 `__attribute__((interrupt("TIMER0_A1")))` 会生成 `__interrupt_vector_TIMER0_A1` 节。若链接脚本中没有对应规则，该节会被 orphan 处理，**但用户很难发现中断向量被忽略了**。现在会给出明确警告。

#### `msp430_place_orphan()` 重构
- 原来的逻辑：若找不到 `.lower.X` 和 `.upper.X`，直接报 error 退出
- 修复后：
  - 若无 `.lower.X`，尝试用无前缀的 `.X`（如 `.text` 代替 `.lower.text`）
  - 若仍找不到，改为 warning（非 error），并 fall through 到 `ldelf_place_orphan()`
  - 返回值从 `lower` 节改为 `ldelf_place_orphan()` 的结果

#### `add_wild_for_either()` 新函数
在 `.either` 节 shuffle 时，需要在新的 output section 中插入 wild statement，且每次插入后必须重新做 relaxation 评估（不能用完整的 `lang_relax_sections`，因为 shuffle 尚未完成）：
```c
static lang_wild_statement_type *
add_wild_for_either (lang_statement_list_type *list)
{
  ...
  intermediate_relax_sections ();  // 只做 relax，不做 size 评估
  return wild;
}
```

#### `change_output_section()` 签名修复
新增 `old_os` 参数，在移动 input section 后修正旧 output section 的 tail 指针：
```c
// 修复前：旧 output section 的 tail 悬空指向已移走的节
// 修复后：
lang_statement_list_type *old_list = &old_os->children;
if (old_list->tail == (lang_statement_union_type **)curr)
    old_list->tail = (lang_statement_union_type **)prev;
```

#### `.either` shuffle 逻辑修正
```c
// 原逻辑（错误）：无论 upper 是否溢出，都往 lower 移
if (*lower_size + s->size < lower->region->length)
    change_output_section (..., lower);

// 修复后：仅在 upper 溢出且 lower 有空间时才移动
if (*upper_size > upper->region->length
    && *lower_size + s->size < lower->region->length)
    change_output_section (..., lower, upper);
```

### 4.2 `msp430.em`：finish hook 新增检查

新增完整的 `gld${EMULATION_NAME}_finish()` 函数，替代原来的 `finish_default`：

1. **`check_array_section_alignment()`**：检查 `__init_array_start`、`__preinit_array_start`、`__fini_array_start` 是否字节对齐（MSP430 要求 2 字节对齐），否则 crt0 遍历数组时会崩溃：
   ```c
   if (sym->u.def.value % 2)
       einfo ("%P: warning: \"%s\" symbol is not word aligned\n", name);
   ```

2. **`check_interrupt_vector_used()`**：当 `--gc-sections` 启用时，被 GC 掉的中断向量节也会触发警告（GC 路径下 orphan 检测走不到 `msp430_place_orphan()`）。

### 4.3 链接脚本模板 `elf32msp430.sc` 重构

| 改动 | 原始 | 修改后 |
|------|------|--------|
| 入口点 | 无 `ENTRY` | `ENTRY (_start)`（relocating 模式下） |
| init 节 | `.init0`~`.init9`（手写顺序） | `KEEP(*(SORT(.crt_*)))`（新 crt0 框架） |
| fini 节 | `.fini0`~`.fini9` | 移除（新 crt0 不用 finiX） |
| rel.init/fini | 保留 | 移除（MSP430 不使用这些节） |
| 数组对齐 | 无 ALIGN | `ALIGN(2)` 在 `__init_array_start` 等符号前 |
| .bss 匹配 | `*(.bss)` | `*(.bss.* .bss)`（包含 per-object 节） |

**关键含义**：旧的 `.init0`~`.init9` 约定是 avr-libc 风格的初始化顺序控制方式；新的 `SORT(.crt_*)` 允许通过文件名排序控制初始化顺序，更灵活，与新版 newlib 的 crt0 配合使用。

### 4.4 `ldlang.c` / `ldlang.h`

新增访问器函数：
```c
lang_memory_region_type *get_memory_region_list (void)
{
    return lang_memory_region_list;
}
```

供 `msp430.em` 遍历内存区域（用于 SMK_LOCATION 的地址合法性检查）。

---

## 五、BFD 核心修复

### 5.1 `bfd/elf.c`：段布局 Bug 修复

**`TOEND()` 宏修复**：
```c
// 原始：SEC_THREAD_LOCAL 节（.tbss）会被推到段末
#define TOEND(x) (((x)->flags & (SEC_LOAD | SEC_THREAD_LOCAL)) == 0)

// 修复：零大小节不应影响段布局
#define TOEND(x) (((x)->flags & (SEC_LOAD | SEC_THREAD_LOCAL)) == 0 \
                  && (x)->size != 0)
```

**`SHT_NOBITS`（`.bss` 类）节的 `sh_offset` 修复**（PR 12921、PR 25662）：
原来的逻辑混淆了 `SHT_NOBITS` 和 `PT_LOAD` 的处理顺序，导致某些情况下 `.bss` 的 `sh_offset` 被错误调整。修复后明确区分：先检查是否 `PT_LOAD`，再检查是否 `SHT_NOBITS`，并在 `SHT_NOBITS` 时不累加 `off_adjust`。

### 5.2 `bfd/compress.c`：PR 24753

修复压缩节生成时的过滤条件，排除没有内容的节（`SEC_HAS_CONTENTS` 未设置）：
```c
&& (bfd_section_flags (sec) & SEC_HAS_CONTENTS) != 0
```

避免将 `.bss`、`.tbss` 等零大小节纳入 DWARF 压缩处理。

### 5.3 `bfd/elflink.c`：num_group 赋值分裂

```c
// 原始（潜在 signed/unsigned 转换问题）
elf_tdata (abfd)->num_group = num_group = -1;

// 修复（分开赋值，先设局部变量）
num_group = (unsigned) -1;
elf_tdata (abfd)->num_group = -1;
```

---

## 六、GAS 汇编器：设备列表更新（`gas/config/tc-msp430.c`）

设备数据库从 **1.194 版（2016 年 9 月）** 更新至 **1.210 版（2020 年 5 月）**，新增约 **40 个** MSP430FR 系列型号：

| 新增型号系列 | 典型型号 |
|------------|---------|
| FR2000/FR2100 | msp430fr2000, msp430fr2100 |
| FR2153/FR2155/FR2353/FR2355 | MSP430FR2x5x 系列 |
| FR2422/FR2475/FR2476/FR2512/FR2522 | MSP430FR2x2x 系列 |
| FR2672~FR2676 | MSP430FR267x 系列 |
| FR5041/FR5043/FR50431 | MSP430FR504x 系列 |
| FR6005~FR6047 | MSP430FR60xx 系列（11 个型号） |

---

## 七、工具集成

### 7.1 `binutils/readelf.c`

- 新 section 类型：`SHT_GNU_SYMTAB_META` → 显示为 `"GNU_SYMTAB_META"`
- 新 MSP430 重定位类型展示（11/12/22/23 号重定位）
- ULEB128 重定位的变长 size 计算：通过 `read_leb128` 动态确定字节数，而非硬编码 2 或 4

### 7.2 `binutils/objcopy.c`

ELF-to-ELF 复制时，自动读取源 BFD 的 `.symtab_meta` 并写入目标 BFD。

### 7.3 `binutils/dwarf.c`

微修复：在 `strncpy` 后对 `newFileName[MAX_FILENAME_LENGTH]` 显式置 `0`，避免 GCC 10 的字符串截断警告（-Werror=stringop-truncation）。

---

## 八、ELF 格式扩展（`bfd/elfcode.h` / `bfd/elf-bfd.h`）

新增 metasym 的序列化/反序列化钩子函数：
```c
bfd_boolean (*swap_metasym_in)  (bfd *, const void *, Elf_Internal_SymMetaInfo *);
void        (*swap_metasym_out) (bfd *, const Elf_Internal_SymMetaInfo *, void *);
```

32/64 位均有对应的实现（`bfd_elf32_swap_metasym_in/out`、`bfd_elf64_swap_metasym_in/out`），通过 `gdbarch` 钩子表注册。

---

## 九、全局配置

### `configure.ac`

```c
// 添加 texinfo 到 host_tools 列表
host_tools="texinfo flex bison binutils ..."
```

确保构建时优先使用源码树中的 texinfo，避免依赖系统版本（文档格式兼容性问题）。

### `ld/configure.tgt`

对 MSP430 目标，移除 `ldelfgen.o` 的额外链接（该文件的功能已被 `genelf.em` 内联或不再需要）。

---

## 十、影响总结

```
┌──────────────────────────────────────────────────────────────────────┐
│  特性 / 修复              影响范围                  工程意义          │
├──────────────────────────────────────────────────────────────────────┤
│  .symtab_meta             全工具链贯通               全新特性         │
│  ULEB128 重定位           DWARF 正确性              全新特性         │
│  JMP/BR relaxation        代码尺寸 & 正确性          重要 Bug 修复    │
│  .either shuffle 修复     内存布局正确性             重要 Bug 修复    │
│  中断向量 orphan 警告     用户友好性                 体验改善         │
│  数组对齐检查             运行时安全                 防御性检查       │
│  linker script 重构       crt0 框架迁移              架构升级         │
│  设备列表更新             新器件支持                 功能扩展         │
│  TOEND/SHT_NOBITS 修复   ELF 格式正确性             上游 Bug 修复    │
└──────────────────────────────────────────────────────────────────────┘
```

### 核心设计哲学

1. **`.symtab_meta`** 是这批 patch 中技术深度最高的特性，实现了从汇编指令（`.sym_meta_info`）→ 目标文件（`.symtab_meta` section）→ 链接器（GC 保护 + 地址固定）→ 调试工具（readelf 显示）的完整闭环，是 TI 为 MSP430 嵌入式调试场景定制的专有扩展。

2. **ULEB128 重定位** 解决了 relaxation 导致 DWARF 信息失效的根本问题，是正确支持 -Os 优化调试的必要条件。

3. **Relaxation 修复** 中 JMP↔BR(A) 的双向转换逻辑涉及复杂的指令编码细节，新代码在 shrinking 路径中额外节省 2 字节（4→6 字节总节省），对 Flash 受限的 MSP430 意义显著。
