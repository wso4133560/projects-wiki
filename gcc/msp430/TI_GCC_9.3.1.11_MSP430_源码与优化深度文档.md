# TI GCC 9.3.1.11 MSP430 源码与优化深度文档

> 面向对象：需要深入理解 `msp430-elf-gcc`（TI 9.3.1.11）后端实现、优化策略、代码生成路径的编译器/固件工程师。  
> 基于源码：`/home/tanglin/workspace2/study/mcu/msp430/msp430-gcc-9.3.1.11-source-full/gcc`。

---

## 1. 版本与实现边界

- TI 包中的 GCC 基线来自 upstream `gcc-9.3.0`，TI 补丁后版本标识为 `9.3.1`。
- MSP430 后端核心文件：
  - `gcc/config/msp430/msp430.c`
  - `gcc/config/msp430/msp430.md`
  - `gcc/config/msp430/msp430.h`
  - `gcc/config/msp430/msp430.opt`
  - `gcc/config/msp430/constraints.md`
  - `gcc/config/msp430/predicates.md`
  - `gcc/common/config/msp430/msp430-common.c`
- 运行时支持（libgcc）：
  - `libgcc/config/msp430/*`
- MCU 数据解析与 driver spec 函数：
  - `gcc/config/msp430/msp430-devices.c`
  - `gcc/config/msp430/driver-msp430.c`

---

## 2. 源码结构地图（按职责）

### 2.1 目标选项与约束

- `msp430.opt`：定义 `-mmcu/-mcpu/-mlarge/-msmall/-mhwmult/-mcode-region/-mdata-region/-mmax-inline-shift/-muse-430x-for-mem/-mtiny-printf` 等。
- `msp430-opts.h`：目标枚举类型：
  - `enum msp430_hwmult_types`
  - `enum msp430_cpu_types`
  - `enum msp430_regions`
- `msp430-common.c`：处理 `-mmcu=msp430/msp430x/msp430xv2` 这类 generic 名称，转换成 CPU 语义。

### 2.2 后端核心

- `msp430.c`：
  - option override / ABI / 调用约定
  - cost model（`TARGET_RTX_COSTS`、`TARGET_INSN_COST`、memory move cost）
  - shift/mul helper 选择
  - prologue/epilogue 展开
  - section 选择与符号低地址标记
  - builtin 展开（`__delay_cycles` 等）
- `msp430.md`：RTL 模式匹配、split、长度属性、乘法/shift模式、jump/cbranch/tablejump策略。

### 2.3 运行时支持

- `libgcc/config/msp430/t-msp430`：构建 `libmul_none/libmul_16/libmul_32/libmul_f5` 等。
- `lib2mul.c` / `lib2hw_mul.S`：软件/硬件乘法实现。
- `lib2div*.c`, `lib2shift.c`, `slli.S/srai.S/srli.S`：ABI helper 的关键实现。

### 2.4 验证用例

- `gcc/testsuite/gcc.target/msp430/*`：覆盖成本模型、shift helper 阈值、430X memory addressing、hwmult 内联策略等。

---

## 3. 编译选项如何进入后端决策链

### 3.1 入口：`msp430_option_override`

在 `msp430.c` 的 `msp430_option_override()` 中完成关键归一化：

1. 依据 `-mcpu` / `-mmcu` 推导 `msp430x`（是否启用 430X ISA）。
2. 读取 `devices.csv` 或硬编码表，验证 `ISA/hwmpy` 与 `-mhwmult` 是否冲突。
3. 强制约束：
   - `-mlarge` 需要 430X 兼容 MCU。
   - `-mcode-region` / `-mdata-region` 的 `upper/either` 需要 large model。
4. 异常处理相关：开关 `flag_omit_frame_pointer`。
5. `-mOs`（内部目标选项）触发 `optimize_size=1`（用于 newlib 构建链路中的覆盖问题）。

### 3.2 `devices.csv` 的作用

`msp430-devices.c` 支持从以下路径查找 `devices.csv`：

- `-I/-L` 搜索路径
- `MSP430_GCC_INCLUDE_DIR`
- toolchain 下 `msp430-elf/include/devices/`
- `-mdevices-csv-loc=` 显式指定

其结果决定：

- `revision`（430 / 430X / 430Xv2）
- `hwmpy`（none / 16 / 32 / f5）

并直接影响后续 `-mcpu` 推导、`-lmul_*` 选择与乘法代码路径。

### 3.3 Driver spec 函数

`driver-msp430.c` 的 spec hook：

- `msp430_select_cpu`：根据 MCU 自动注入 `-mcpu=...`
- `msp430_select_hwmult_lib`：选择 `-lmul_none/-lmul_16/-lmul_32/-lmul_f5`
- `msp430_get_linker_devices_include_path`：给 linker 增加设备目录
- `msp430_check_lower_region_prefix`：控制 lower region 是否传播给 linker

---

## 4. 优化总览：核心思想

MSP430 后端的优化并非单一“更快”或“更小”，而是以下几层叠加：

1. `optimize_size` 与 `optimize>=2` 双轨决策。
2. RTL cost model（表达式成本）驱动中端/后端选择。
3. `.md` 模式条件（`optimize > 2`、`msp430_has_hwmult()` 等）控制是否内联。
4. 指令长度模型驱动 `TARGET_INSN_COST`。
5. 链接阶段 `--gc-sections` 与 `--relax` 再优化。

---

## 5. 成本模型（重点）

### 5.1 `TARGET_MEMORY_MOVE_COST`

`msp430_memory_move_cost()`：

- speed 模式：`!optimize_size && optimize >= 2`
- size 模式：否则
- 选择不同成本表（cycle vs size）
- 返回值按 `TARGET_REGISTER_MOVE_COST=2` 归一

含义：同一条 move 在 `-Os` 与 `-O2/O3` 下“被认为”的代价不同，影响寄存器分配和表达式重写。

### 5.2 基础成本 `msp430_costs`

按 source/destination addressing 组合计价：

- reg->reg, imm->reg, mem->reg, mem->mem, pc 相关跳转
- 对 `TRUNCATE` 到 `QI/HI/PSI` 视为“通常免费副作用”
- 根据 `speed` 选择不同成本表

### 5.3 shift 成本 `msp430_shift_costs`

重点逻辑：

- 非常量 shift：直接劣化成本（通常会倾向 helper）
- 常量 shift：先走 `use_helper_for_const_shift()` 决策
- 430（非 X）模式：按 `amt` 线性叠加
- 430X：`amt<=4` 可用 `RRAM/RRCM/RRUM/RLAM`，>4 走 `RPT + X 指令`

### 5.4 乘除成本 `msp430_muldiv_costs`

- 乘法与除法共入口，但策略不同：
  - 除法始终重罚（软件 helper）
  - 乘法在有 hwmult 时降成本
- `DImode` 在 16-bit hwmult 上特别昂贵（helper/序列开销大）
- speed 模式下 heavily disparage 软件乘除，迫使优先硬件路径/更优表达式。

### 5.5 总成本钩子 `TARGET_RTX_COSTS`

`msp430_rtx_costs()` 对 `SET/PLUS/MULT/SHIFT/CALL` 等分类处理：

- 递归抓“真正目的操作数”(`msp430_get_inner_dest_code`)
- `SImode` / `DImode` 成本乘系数（2/4）
- 对 `MEM` 的 `PSI/SI/DI` 额外惩罚

### 5.6 指令成本 `TARGET_INSN_COST`

`msp430_insn_cost()` 当前用 `get_attr_length(insn)/2` 近似，说明：

- 指令长度属性在 MSP430 后端非常关键。
- size 优化路径与 speed 路径都受长度模型显著影响。

---

## 6. 指令长度/扩展字模型（与优化强关联）

`msp430.md` 顶部定义了 `type/extension/length` 三层机制：

- `type`: `single/double/triple/cmp`
- `extension`: `none/x/a/m`
  - `x` 通常表示 430X 扩展字
  - `m` 对 `RRAM/RRCM` 等 M extension
- `length`: 根据 operand 是否 cheap、是否高地址扩展等动态计算

这会直接反馈到：

- `TARGET_INSN_COST`
- instruction selection 的偏好
- 与 `-Os/-O3` 路径上的 tradeoff

---

## 7. Shift 优化机制（最重要之一）

### 7.1 入口路径

`.md` 使用 `define_expand "<shift_insn><mode>3"` 调用 `msp430_expand_shift()`。

### 7.2 `use_helper_for_const_shift()` 决策

关键点：

- 默认 inline 阈值 4（`msp430_max_inline_shift` 默认 sentinel=65，再映射为 4）
- 用户可用 `-mmax-inline-shift=N` 覆盖
- `SImode` 阈值按 1/2 折算
- 在 `-O2+` 且非 `-Os` 时，某些模式倾向 always inline
- `DImode` 默认走 helper

### 7.3 helper 选择

`msp430_expand_shift()` 会在需要 helper 时映射到：

- `__mspabi_slli/srai/srli`
- `__mspabi_slll/sral/srll`
- `__mspabi_sllll/srall/srlll`
- PSI 对应 `__gnu_mspabi_*`

并在非 `-Os` 尝试 const 变体（`helper_n` 版本）以换速度。

### 7.4 430 vs 430X 发码差异

`msp430_output_asm_shift_insns()`：

- 430 HI 常量 shift：循环发 `RLA/RRA/CLRC+RRC`
- 430X 可用 `RPT + RLAX/RRAX/RRUX` 或 `RLAM/RRAM/RRUM`
- 估长时显式考虑 `msp430x_insn_required(operand)`

### 7.5 testsuite 佐证

- `max-inline-shift-430.c`
- `max-inline-shift-430-no-opt.c`
- `max-inline-shift-430x.c`

这些测试直接验证阈值与 helper 调用行为。

---

## 8. 乘法/除法优化机制（重点）

### 8.1 hw multiplier 能力判定

`msp430_use_f5_series_hwmult()/msp430_use_32bit_hwmult()/msp430_use_16bit_hwmult()/msp430_has_hwmult()`：

- 先看显式 `-mhwmult=`
- `auto` 时根据 `mmcu` + `devices.csv` 的 `hwmpy` 推断
- 对未知 MCU，默认保守（无 hwmult）

### 8.2 helper 调用与符号重定向

`msp430_expand_helper()`：

- 将参数装入 ABI 约定寄存器（R12/R13/...）
- 对 mpy 系 helper 根据 hwmult 选择后缀：
  - `_hw`
  - `_hw32`
  - `_f5hw`
- 通过 call usage 明确寄存器占用，减少后续优化误判。

### 8.3 O3 下 widening multiply 内联

`msp430.md` 中：

- `mulhisi3/umulhisi3/mulsidi3/umulsidi3`
- 条件 `optimize > 2 && msp430_has_hwmult()` 时可进入内联模式
- 内联模板直接访问硬件乘法寄存器地址，并包裹中断关开序列

### 8.4 中断安全性

硬件乘法 helper（libgcc `lib2hw_mul.S`）会保存/关闭/恢复中断状态，保证 ISR 场景可安全调用。

### 8.5 testsuite 佐证

- `rtx-cost-O3-f5series.c` 验证 `-O3 -mhwmult=f5series` 下的内联倾向与 helper 抑制。

---

## 9. 内存模型与 430X 指令选择

### 9.1 Small/Large 模型

- `-msmall`: 16-bit pointers
- `-mlarge`: 20-bit pointers (`PSImode`)
- `Pmode = TARGET_LARGE ? PSImode : HImode`

### 9.2 低地址符号标记

`msp430_encode_section_info()` 会为可判定低地址变量打 `SYMBOL_FLAG_LOW_MEM`，后续用于判断是否可安全使用 430（非 X）寻址。

### 9.3 `msp430x_insn_required()` 决策

触发 430X 指令的条件（任一成立）：

- 操作数是 `PSImode`
- 操作数可能位于高地址
- 开启 `-muse-430x-for-mem` 且为 MEM

### 9.4 `-muse-430x-for-mem`

默认关闭；开启后强制内存操作走 430X，降低高地址误判风险，代价是代码尺寸与某些路径性能。

### 9.5 约束层辅助

`constraints.md` 的 `Ys/Ya` 与 `predicates.md` 的高地址判定 predicate，共同配合模式匹配控制发码合法性与大小。

---

## 10. section/region 机制与优化影响

### 10.1 选项与 attribute 联动

- 全局策略：`-mcode-region` / `-mdata-region`
- 局部属性：`lower/upper/either/noinit/persistent/section`

### 10.2 `gen_prefix()` + section hook

`msp430_select_section()` / `msp430_unique_section()`：

- 根据 region 加前缀 `.lower/.upper/.either`
- large 模式下 ISR 强制进 `.lowtext`（向量表要求 16-bit 地址）

### 10.3 与 `--gc-sections` 的协作

`msp430.h` 默认 linker spec 在非 `-g`、非 `-r` 且未 `-mno-gc` 时启用 `--gc-sections`。

配合 `-ffunction-sections -fdata-sections` 可显著降尺寸。

---

## 11. 跳转/分支相关优化

- `tablejump` 在 `-mlarge` 下默认抑制，除非更偏 speed（`optimize > 2 || !TARGET_LARGE`）。
- 原因：
  - `__int20` 指针转换与 jump table 元素宽度导致额外 code bloat。
- `BRANCH_COST=3` + `LOGICAL_OP_NON_SHORT_CIRCUIT=0`：倾向保留短路逻辑，兼顾尺寸与性能。

---

## 12. Prologue/Epilogue 与函数属性优化

### 12.1 保存策略

`msp430_preserve_reg_p()` 针对 ISR/leaf/non-leaf 做差异保存策略，避免不必要压栈。

### 12.2 helper epilogue

在非 430X 某些场景可触发 `epilogue_helper`（`__mspabi_func_epilog_N`）减少序列冗余。

### 12.3 特殊属性

- `interrupt`
- `wakeup`
- `critical`
- `reentrant`
- `naked`

这些属性直接改变 prologue/epilogue 发码与中断位处理路径。

---

## 13. libgcc 运行时实现对优化的影响

`libgcc/config/msp430/t-msp430` 中的构建规则决定可链接 helper 的粒度：

- 按功能拆分（div/mul/shift）减少无关代码进入最终镜像。
- `HOST_LIBGCC2_CFLAGS += -Os -ffunction-sections -fdata-sections -mhwmult=none -DNO_DI_MODE`

这体现 TI 工具链对“尺寸优先”场景的现实取舍。

---

## 14. testsuite 解读：可用作回归基准

推荐重点看以下用例：

- `rtx-cost-O3-default.c`
- `rtx-cost-Os-default.c`
- `rtx-cost-O3-f5series.c`
- `use-430x-for-mem.c`
- `max-inline-shift-430*.c`

用途：

- 验证成本模型是否按预期驱动 codegen。
- 验证 helper 与 inline 的边界点。
- 验证内存寻址是否切换到 430X。

---

## 15. 面向实践的优化配置建议

### 15.1 极致代码体积（通用 MCU）

- 推荐：`-Os -ffunction-sections -fdata-sections -Wl,--gc-sections -mrelax`
- 视项目选择：`-mtiny-printf`
- 若 shift helper 过多导致性能问题，可小幅增大 `-mmax-inline-shift`

### 15.2 平衡型（速度+体积）

- 推荐：`-O2 -mrelax`
- 对 shift 热点按 profile 调整 `-mmax-inline-shift`
- `mhwmult=auto`（配合有效 `mmcu`/`devices.csv`）

### 15.3 追求速度（且有硬件乘法）

- 推荐：`-O3 -mhwmult=auto`（或明确 `f5series/32bit`）
- 注意：
  - widening multiply 可能内联，体积上升
  - tablejump 在 large 模型下才会更积极启用

### 15.4 高地址敏感工程

- 在 large model 且内存布局复杂时，考虑 `-muse-430x-for-mem` 规避潜在高地址寻址歧义。

---

## 16. 常见误区

1. `ChangeLog` 出现 `9.2.0 released` 不代表当前基线是 9.2.0。要看 `BASE-VER` 与 patch 基线。
2. `-mmcu` 只影响宏，不影响优化：错误。它会影响 ISA、hwmult、链接脚本和代码生成路径。
3. `-O3` 一定更快：在 MSP430 上，代码尺寸膨胀可能反过来影响系统行为（flash/缓存/总线时间），需实测。
4. `-muse-430x-for-mem` 一定更优：它是“保守正确性+一致性”选项，不是普适性能选项。

---

## 17. 源码阅读建议顺序（最快建立整体模型）

1. `msp430.opt`（先理解旋钮）
2. `msp430_option_override`（看旋钮如何归一化）
3. `msp430_rtx_costs` + `msp430_shift_costs` + `msp430_muldiv_costs`
4. `msp430.md` 的 shift/mul/tablejump/prologue 相关模式
5. `msp430x_insn_required` 与 section/low-memory 相关函数
6. `libgcc/config/msp430/*` 对应 helper 实现
7. testsuite 用例反证你的理解

---

## 18. 附：关键源码文件清单（可直接跳转）

- `gcc/config/msp430/msp430.c`
- `gcc/config/msp430/msp430.md`
- `gcc/config/msp430/msp430.h`
- `gcc/config/msp430/msp430.opt`
- `gcc/config/msp430/msp430-opts.h`
- `gcc/config/msp430/constraints.md`
- `gcc/config/msp430/predicates.md`
- `gcc/config/msp430/msp430-devices.c`
- `gcc/config/msp430/driver-msp430.c`
- `libgcc/config/msp430/t-msp430`
- `libgcc/config/msp430/lib2hw_mul.S`
- `libgcc/config/msp430/lib2mul.c`
- `gcc/doc/invoke.texi`（MSP430 选项章节）

