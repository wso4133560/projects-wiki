# TI GCC 9.3.1.11 MSP430 优化函数逐条注释

> 目标：按“函数级别”解释 TI GCC 9.3.1.11 在 MSP430 上的优化决策。  
> 基础源码根目录：`/home/tanglin/workspace2/study/mcu/msp430/msp430-gcc-9.3.1.11-source-full/gcc`。

---

## 1. 阅读方式与主线

建议按这条主线阅读本文件：

1. `option 归一化`：`msp430_option_override` 决定 ISA/hwmult/模型约束。
2. `成本模型`：`msp430_rtx_costs` + 子成本函数决定“偏向什么代码形态”。
3. `扩展与 helper`：`msp430_expand_shift` / `msp430_expand_helper` 决定 inline vs 调库。
4. `地址与内存区域`：`msp430x_insn_required` + section hook 决定 430 或 430X 指令。
5. `运行时库耦合`：`driver-msp430.c` 选择 `-lmul_*`，最终影响乘法落地路径。

---

## 2. 函数索引（优化相关）

### 2.1 核心优化入口（`msp430.c`）

- `msp430_option_override`（`msp430.c:159`）
- `msp430_memory_move_cost`（`msp430.c:1221`）
- `msp430_costs`（`msp430.c:1261`）
- `msp430_shift_costs`（`msp430.c:1331`）
- `msp430_muldiv_costs`（`msp430.c:1411`）
- `msp430_rtx_costs`（`msp430.c:1613`）
- `msp430_insn_cost`（`msp430.c:1765`）

### 2.2 代码生成关键函数（`msp430.c`）

- `msp430_expand_helper`（`msp430.c:3351`）
- `use_helper_for_const_shift`（`msp430.c:3467`）
- `msp430_expand_shift`（`msp430.c:3517`）
- `msp430_output_asm_shift_insns`（`msp430.c:3580`）

### 2.3 硬件乘法判定与寻址判定（`msp430.c`）

- `msp430_use_f5_series_hwmult`（`msp430.c:3940`）
- `msp430_use_32bit_hwmult`（`msp430.c:3975`）
- `msp430_use_16bit_hwmult`（`msp430.c:4002`）
- `msp430_has_hwmult`（`msp430.c:4030`）
- `msp430_op_not_in_high_mem`（`msp430.c:4224`）
- `msp430x_insn_required`（`msp430.c:4262`）

### 2.4 section / 符号标记与地址空间（`msp430.c`）

- `msp430_addr_space_convert`（`msp430.c:512`）
- `msp430_encode_section_info`（`msp430.c:2277`）
- `msp430_select_section`（`msp430.c:2530`）
- `msp430_unique_section`（`msp430.c:2635`）
- `msp430_output_aligned_decl_common`（`msp430.c:2671`）

### 2.5 Driver 侧关键函数（`driver-msp430.c`）

- `msp430_select_cpu`（`driver-msp430.c:31`）
- `msp430_select_hwmult_lib`（`driver-msp430.c:55`）
- `msp430_get_linker_devices_include_path`（`driver-msp430.c:122`）
- `msp430_check_lower_region_prefix`（`driver-msp430.c:134`）

---

## 3. 选项归一化：`msp430_option_override` 深解

位置：`msp430.c:159`

### 3.1 职责

- 将用户选项转换为内部可执行状态：`msp430x`, `hwmult`, `region`, `frame-pointer` 策略。
- 提前做“非法组合”拦截，避免后续 pass 出现不一致。

### 3.2 关键决策

- `msp430x = target_cpu >= MSP430_CPU_MSP430X_DEFAULT`
- 如果给了 `-mmcu`：
  - 解析 `devices.csv` / 硬编码表
  - 校验 `-mcpu` 与 MCU 实际 ISA 是否冲突（可告警）
  - 校验 `-mhwmult` 与 MCU `hwmpy` 是否冲突（可告警）
- 约束：
  - `-mlarge` 必须配 430X
  - `-mcode-region=upper/either`、`-mdata-region=upper/either` 需要 `-mlarge`
- 对 `-fexceptions` 等选项：控制 `flag_omit_frame_pointer`
- `TARGET_OPT_SPACE && optimize < 3` 时强制 `optimize_size=1`（补偿 newlib 构建流程）

### 3.3 优化含义

这一步决定了后面几乎所有优化分支是否可达：

- 是否允许 `PSImode` 指针优化路径
- 是否启用 430X 指令模式
- 是否能走硬件乘法快速路径
- section 放置是否允许 upper/either

---

## 4. 成本模型主干（最核心）

## 4.1 `msp430_memory_move_cost`（`msp430.c:1221`）

### 功能

评估寄存器与内存之间 move 成本，作为 RA/中端决策输入。

### 逻辑

- speed 模式判定：`!optimize_size && optimize >= 2`
- speed 与 size 使用不同成本表（cycle_cost vs size_cost）
- 返回值按 `TARGET_REGISTER_MOVE_COST=2` 归一

### 优化影响

同样代码在 `-Os` 与 `-O2/O3` 可能有完全不同的寄存器驻留策略。

## 4.2 `msp430_costs`（`msp430.c:1261`）

### 功能

给定 `(src, dst, outer_rtx)`，按寻址模式组合计算基础成本。

### 关键点

- `TRUNCATE` 到 `QI/HI/PSI` 常视为免费。
- 主要按 `reg/imm/mem(indirect vs non-indirect)/pc` 组合计价。
- MSP430 成本高度依赖寻址方式而非算子名称。

## 4.3 `msp430_shift_costs`（`msp430.c:1331`）

### 功能

为 shift/rotate 系操作定价。

### 关键分支

- 非常量 shift：成本劣化（倾向 helper）。
- 常量 shift：先调用 `use_helper_for_const_shift()`。
- 430：按 `amt` 线性叠加。
- 430X：`<=4` 有更便宜模式（`RRAM/RRUM/...`），否则 `RPT + X`。

### 关键语义

该函数不仅“算成本”，还隐含向中端传达“helper 更合理”或“inline 更合理”的偏好。

## 4.4 `msp430_muldiv_costs`（`msp430.c:1411`）

### 功能

对 `MULT/DIV/MOD/UDIV/UMOD` 成本建模。

### 关键分支

- `!speed`：保守尺寸导向，乘法按 hwmult 能力轻重分层，除法统一较高惩罚。
- `speed`：
  - 无 hwmult 或不适配模式时强惩罚（尤其 DIV/MOD）
  - 有 hwmult 时加入参数搬运成本后定价
- `DImode` + 16-bit hwmult 视为代价很大。

### 优化结果

- `-O3 + 有 hwmult` 更容易触发 inline multiply 序列。
- `-Os` 更偏向 helper 或更短路径。

## 4.5 `msp430_rtx_costs`（`msp430.c:1613`）

### 功能

`TARGET_RTX_COSTS` 的真实入口，驱动 GCC 对 RTL 表达式的全局偏好。

### 核心机制

- 分类处理 `SET/PLUS/MULT/SHIFT/CALL/...`
- 递归定位“内层真实目的操作数”
- 对 `SImode`、`DImode` 加倍率
- 对 `MEM` 的 `PSI/SI/DI` 额外加权

### 一句话总结

这是真正决定“GCC 后续愿意选什么形态”的主调函数。

## 4.6 `msp430_insn_cost`（`msp430.c:1765`）

### 功能

`TARGET_INSN_COST`：使用 `get_attr_length(insn)/2` 作为相对成本。

### 含义

- 指令长度属性对后端决策影响极大。
- `msp430.md` 的 `length/type/extension` 规则是优化地基。

---

## 5. Shift 优化：从中端到发码

## 5.1 `use_helper_for_const_shift`（`msp430.c:3467`）

### 输入

- 模式（QI/HI/PSI/SI/DI）
- 常量移位量 `amt`

### 决策关键

- 默认阈值：4（除非用户显式 `-mmax-inline-shift`）
- `SImode` 阈值折半
- `-O2+ 且非 -Os` 时倾向 always inline（但有上限）
- `DImode` 强制 helper

### 伪代码

```text
if mode == DI: use helper
if mode in {QI,HI,PSI}:
  if msp430x or amt <= max_inline or always_inline: inline
  else helper
if mode == SI:
  if amt <= max_inline/2 or (amt % 16)<=max_inline/2 or (always_inline && amt<=15): inline
  else helper
```

## 5.2 `msp430_expand_shift`（`msp430.c:3517`）

### 功能

在 expand 阶段决定：

- 直接走 helper 调用路径（并 `DONE`）
- 或留给 `.md` pattern 匹配 inline 指令序列

### helper 名称映射

- HI: `__mspabi_slli/srai/srli`
- SI: `__mspabi_slll/sral/srll`
- DI: `__mspabi_sllll/srall/srlll`
- PSI: `__gnu_mspabi_sllp/srap/srlp`

### 优化细节

- 非 `-Os` 倾向常量特化 helper（`_n` 变体）
- `-Os` 会关闭 const helper 变体，优先代码尺寸

## 5.3 `msp430_output_asm_shift_insns`（`msp430.c:3580`）

### 功能

真正输出或估算 shift 指令序列长度。

### 关键点

- 430 HI：重复 `RLA/RRA/CLRC+RRC`
- SI 模式在 430/430X 使用双字序列
- 估长时显式加上 `msp430x_insn_required(op)` 的扩展代价

---

## 6. 乘法优化：helper 与 inline 的边界

## 6.1 `msp430_expand_helper`（`msp430.c:3351`）

### 功能

通用 ABI helper 调用器，服务于 shift/mul/div 等。

### 做了什么

- 把参数搬入 ABI 寄存器（R12/R13/...）
- 可生成 const helper 变体名
- 对 mpy helper 根据 hwmult 类型改后缀：
  - `_hw`, `_hw32`, `_f5hw`
- 构造 `call_insn` 并附加寄存器 usage 信息

### 优化含义

它不是“普通调用封装”，而是连接“目标能力判定”与“最终 helper 选择”的核心枢纽。

## 6.2 `msp430_use_*hwmult` / `msp430_has_hwmult`

- `msp430_use_f5_series_hwmult`（`3940`）
- `msp430_use_32bit_hwmult`（`3975`）
- `msp430_use_16bit_hwmult`（`4002`）
- `msp430_has_hwmult`（`4030`）

### 共同特点

- 首先尊重显式 `-mhwmult=`。
- `auto` 时从 `mmcu` 解析 `hwmpy`。
- 使用静态缓存避免重复解析，降低编译期开销。

### 安全策略

未知 MCU 默认保守：无 hwmult（避免错误生成硬件乘法访问序列）。

---

## 7. 430 vs 430X 寻址判定（优化与正确性边界）

## 7.1 `msp430_encode_section_info`（`msp430.c:2277`）

### 功能

根据变量与 section 信息打 `SYMBOL_FLAG_LOW_MEM`。

### 作用

为后续“是否可以不用 430X”提供静态证据。

## 7.2 `msp430_op_not_in_high_mem`（`msp430.c:4224`）

### 功能

判断一个操作数是否可被认为“不在高地址风险区”。

### 关键判断

- 非 large 模式通常直接 true
- `SYMBOL_FLAG_LOW_MEM` 为 true
- 某些 `plus/index` 且偏移在 16-bit 安全范围
- 常量绝对地址在 16-bit 范围

## 7.3 `msp430x_insn_required`（`msp430.c:4262`）

满足任一即需要 430X：

- 操作数模式为 `PSImode`
- `!msp430_op_not_in_high_mem(op)`
- `MEM && -muse-430x-for-mem`

### 优化含义

这是“节省代码尺寸（尽量 430）”与“高地址安全（必要时 430X）”的平衡点。

---

## 8. Section/Region 优化链路

## 8.1 `msp430_select_section`（`msp430.c:2530`）

### 功能

按 `attribute + -mcode/-mdata-region + large/small` 选择 section。

### 关键策略

- large 模式下 interrupt 函数强制 `.lowtext`（向量表地址限制）
- 支持 `.lower/.upper/.either` 前缀策略
- 对 `noinit/persistent` 专门分区

## 8.2 `msp430_unique_section`（`msp430.c:2635`）

### 功能

unique section 场景下继续保持 region 前缀语义一致。

## 8.3 `msp430_output_aligned_decl_common`（`msp430.c:2671`）

### 功能

common symbol 输出时考虑 `mdata-region` 与 attribute，必要时改走 `.bss` 族细分 section。

### 优化含义

影响链接器最终布局，也会反向影响寻址模式与 430X 扩展字需求。

---

## 9. 地址空间转换与 mode 成本

## 9.1 `msp430_addr_space_convert`（`msp430.c:512`）

- far->near：需要截断（潜在不可预测）
- near->far：零扩展，通常安全

这与 `__attribute__((address_space))` 或近/远指针代码生成强相关。

---

## 10. Prologue/Epilogue 与性能/尺寸

- `msp430_expand_prologue`（`3001`）
- `msp430_expand_epilogue`（`3123`）

关键点：

- 根据函数属性（`interrupt/critical/reentrant/naked`）切换保存序列。
- 430X 支持 `PUSHM/POPM` 批量寄存器操作，优化长度与周期。
- 某些 430 场景可走 `epilogue_helper`，减少重复序列。

---

## 11. `.md` 模式对优化的直接控制

文件：`gcc/config/msp430/msp430.md`

关键模式位置：

- shift 扩展入口：`1146`
- 430 常量 shift：`1159`, `1171`
- 430X shift：`1182`, `1196`, `1210`
- tablejump：`1377`
- mul/mulwiden expand：`1822` 起
- O3 inline widening mul：`1915`, `1929`, `1943`, `1957`

### 典型条件

- `optimize > 2 && msp430_has_hwmult()`：允许内联 widening multiply
- `!(flag_pic || flag_pie) && (optimize > 2 || !TARGET_LARGE)`：tablejump 开关

---

## 12. Driver 与 libmul 选择（最终落地）

## 12.1 `msp430_select_hwmult_lib`（`driver-msp430.c:55`）

输入来源：

- `-mhwmult=...`
- 或 `-mmcu=...` 推导

输出：

- `-lmul_none`
- `-lmul_16`
- `-lmul_32`
- `-lmul_f5`

这一步决定最终链接到哪一套乘法实现。

## 12.2 `msp430_select_cpu`（`driver-msp430.c:31`）

根据 MCU 自动注入 `-mcpu=...`，避免前端/后端 ISA 不一致。

---

## 13. 与文档/选项手册对应关系

参考 `gcc/doc/invoke.texi` 的 MSP430 章节（约 `22890+` 行）：

- `-mhwmult=`
- `-mmax-inline-shift=`
- `-muse-430x-for-mem`
- `-mcode-region=` / `-mdata-region=`
- `-mrelax`
- `-mtiny-printf`

这些公开选项和本文件函数分析是 1:1 对应关系。

---

## 14. 优化调参速查

### 14.1 代码体积优先

- `-Os -ffunction-sections -fdata-sections -Wl,--gc-sections -mrelax`
- shift helper 边界太激进可调：`-mmax-inline-shift=2~6`

### 14.2 速度优先（且有 hwmult）

- `-O3 -mhwmult=auto`（或显式 `f5series/32bit`）
- 观察 widening multiply 是否被 inline（可用反汇编检查）

### 14.3 高地址安全优先

- large model 且布局复杂时：`-muse-430x-for-mem`
- 注意代码尺寸上涨，需和 flash 预算平衡

---

## 15. 建议的深挖顺序（按收益）

1. `msp430_option_override`
2. `msp430_rtx_costs` + `msp430_shift_costs` + `msp430_muldiv_costs`
3. `use_helper_for_const_shift` + `msp430_expand_shift`
4. `msp430x_insn_required` + `msp430_encode_section_info`
5. `msp430.md` 的 shift/mul/tablejump 模式
6. `driver-msp430.c` 与 `libgcc/config/msp430/t-msp430`

---

## 16. 你最该重点盯的三处

1. `msp430_shift_costs` / `use_helper_for_const_shift`：决定 shift 热点代码形态。  
2. `msp430_muldiv_costs` + `msp430_has_hwmult`：决定乘法是否走硬件快路径。  
3. `msp430x_insn_required` + section/low_mem 标记：决定 430 与 430X 切换，直接影响尺寸与正确性边界。

