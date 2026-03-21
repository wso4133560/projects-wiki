# TI GCC 9.3.1.11（MSP430）在 `-O2` / `-O3` 下的性能优化源码分析

## 1. 分析范围与结论

- 代码基线：`msp430-gcc-9.3.1.11-source-full/gcc/gcc/config/msp430/`
- 重点文件：`msp430.c`、`msp430.md`、`msp430.h`
- 关注点：MSP430 后端里**真正由 `optimize`（即 `-O` 等级）直接控制**的代码生成行为。

结论先行：

1. 对 MSP430 后端来说，`-O2` 与 `-O3` 的很多行为是相同的（都属于 speed 导向）。
2. 但存在几处明确的 `optimize > 2` 分支，导致 `-O3` 才触发的关键路径：
- `tablejump`（在 `-mlarge` 下）
- 硬件乘法相关 widening multiply 的 inline 序列
3. 对有硬件乘法器（尤其 `-mhwmult=f5series`）的芯片，`-O3` 在乘法热点路径上更有机会减少 helper call，获得性能收益。
4. 对 `-mlarge` 代码，`-O3` 倾向以代码体积换取分支分发表速度。

---

## 2. O2/O3 判定门槛总览（源码级）

| 判定条件 | 源码位置 | 含义 | 对 O2 | 对 O3 |
|---|---|---|---|---|
| `optimize >= 2 && !optimize_size` | `msp430.c:1229` | 速度导向 cost model | 开启 | 开启 |
| `optimize >= 2 && !optimize_size` | `msp430.c:3480-3481` | 常量移位 `always_inline` | 开启 | 开启 |
| `optimize > 2` | `msp430.md:1381` | `tablejump` 在 `-mlarge` 下可用 | 关闭（large） | 开启（large） |
| `optimize > 2 && msp430_has_hwmult()` | `msp430.md:1919/1933/1947/1961` | widening multiply inline 序列 | 关闭 | 开启（需 hwmult） |
| `!(optimize > 2 && msp430_has_hwmult())` | `msp430.md:1861/1876/1891/1906` | widening multiply 回落 helper | 使用 helper | 尽量 inline |
| `optimize >= 2 && !optimize_size` | `msp430.h:266` | `NO_FUNCTION_CSE` | 开启 | 开启 |

---

## 3. `-O2` / `-O3` 共同行为（MSP430 后端）

## 3.1 Cost model 进入 speed 模式

源码：`msp430.c:1227-1232`

```c
/* Optimize with a code size focus by default, unless -O2 or above is
   specified.  */
bool speed = (!optimize_size && optimize >= 2);
cost_p = (speed ? &cycle_cost_double_op_mov : &size_cost_double_op);
```

解释：

- `-O2` 和 `-O3` 都让 `speed=true`（前提不是 `-Os` 路径），后端成本评估从偏 code-size 切到偏 cycle。
- 这影响寄存器/内存 move、算术、复合表达式选择（通过 `msp430_rtx_costs`/`msp430_costs` 链路传播）。

性能含义：

- O2 与 O3 在大量普通算术/搬运上差距并不大，二者都不是“省代码优先”。

## 3.2 常量移位：`always_inline` 在 O2 即生效

源码：`msp430.c:3478-3481`

```c
int always_inline = (!user_set_max_inline
                     && (optimize >= 2 && !optimize_size));
```

解释：

- 只要到 O2（且不走 size 优先），后端就倾向把常量移位直接展开，而不是 helper。
- O3 不会在这里再增加新的阈值门槛。

补充：`msp430_expand_shift` 中 const helper 变体选择是 `!optimize_size`（`msp430.c:3529`），不是 `optimize > 2`。

结论：

- 移位这条线上，O2 与 O3 行为总体接近，更多差异来自上层通用优化（如更激进内联/向量化尝试）而非 MSP430 target 钩子本身。

## 3.3 函数地址 CSE 策略：O2/O3 同步

源码：`msp430.h:266`

```c
#define NO_FUNCTION_CSE (optimize >= 2 && !optimize_size)
```

解释：

- O2/O3 都激活这个宏路径；对“函数常量地址 vs 寄存器间接调用”的体积/性能权衡策略一致。

---

## 4. 仅 O3 才触发的关键行为

## 4.1 `tablejump` 在 `-mlarge` 下由 O3 放开

源码：`msp430.md:1369-1382`

```lisp
; tablejump is costly for -mlarge so we disable it unless optimizing for speed.
(define_insn "tablejump"
  ...
  "!(flag_pic || flag_pie) && (optimize > 2 || !TARGET_LARGE)"
  "BR%Q0\t%0 ; tablejump")
```

关键点：

- 条件是 `optimize > 2 || !TARGET_LARGE`。
- 对 small model（`!TARGET_LARGE`）来说，O2/O3都可使用。
- 对 large model：
- `-O2` 不满足 `optimize > 2`，通常禁用表跳转。
- `-O3` 满足，允许表跳转。

性能影响：

- 大型 `switch` 在 O3 + large model 下更可能走 jump-table，降低分支链比较开销。
- 代价是 jump-table 条目（large 指针）带来的体积增长。

适用场景：

- 高频状态机/协议解析器：优先评估 O3。
- Flash 紧张场景：需测量 map/section 增量再决定。

## 4.2 widening multiply（HI->SI, SI->DI）在 O3 才偏向 inline 硬件序列

源码（expand 回落 helper）：

- `msp430.md:1861`
- `msp430.md:1876`
- `msp430.md:1891`
- `msp430.md:1906`

共同模式：

```lisp
if (!(optimize > 2 && msp430_has_hwmult ()))
{
  msp430_expand_helper (...);
  DONE;
}
```

源码（inline insn 启用条件）：

- `msp430.md:1919`
- `msp430.md:1933`
- `msp430.md:1947`
- `msp430.md:1961`

共同模式：

```lisp
"optimize > 2 && msp430_has_hwmult ()"
```

解释：

- O2：即使有 hwmult，也更常走 helper call。
- O3：若有 hwmult，优先使用内联硬件乘法访问序列。

性能影响：

- 热点乘法路径可减少函数调用与 ABI 保存恢复开销。
- 在 ISR 密集或 tight loop 中，吞吐改进通常更明显。

注意点：

- inline 模式里包含对乘法寄存器的访问序列，并在模板中出现 `PUSH.W sr { DINT ... POP.W sr`（见 `msp430.md:1921-1925` 等），体现了对临界区和寄存器访问时序的保护。
- 因此收益与副作用都依赖具体 MCU、时钟和中断负载，需要实测。

---

## 5. `-Os`/`-mopt-space` 与 O2/O3 的交互

源码：`msp430.c:284-289`

```c
if (TARGET_OPT_SPACE && optimize < 3)
  optimize_size = 1;
```

解释：

- 这是 TI/该分支中为 newlib 构建流程加入的“空间优化纠偏”逻辑。
- 当 `TARGET_OPT_SPACE` 生效且优化等级低于 O3 时，强制进入 size 导向。

推导：

- 如果你的构建链路开启了该目标选项，`-O2` 可能被重新导向 `optimize_size=1`，从而改变上文很多判断（如 `speed`、`always_inline`）。
- `-O3` 下该分支不触发，可能“恢复”性能导向行为。

建议：

- 做 O2/O3 对比时必须固定 `-mopt-space` / `-Os` / specs 配置，否则结论会被 size 模式污染。

---

## 6. 与硬件乘法器类型相关的 O3 收益边界

硬件能力判定入口：`msp430.c:4030-4057`（`msp430_has_hwmult`）

收益边界可概括为：

1. `-mhwmult=none` 或自动识别到无乘法器：
- O3 的 widening multiply inline 通道不可用，O2/O3差异主要回到通用优化层。

2. `-mhwmult=16bit/32bit/f5series` 且识别成功：
- O3 可触发更多 inline multiply 模式，理论上更容易领先 O2。

3. `-mhwmult=auto` + `-mmcu` 不可识别：
- 后端安全回退为“无 hwmult”假设（`msp430.c:4055-4057`），O3优势会被削弱。

---

## 7. 测试用例证据（上游 testsuite）

- `gcc.target/msp430/rtx-cost-O3-default.c`
- `gcc.target/msp430/rtx-cost-O3-f5series.c`

关键点：

- 前者固定 `-O3` 验证默认 ISA/hwmult 下的成本模型产物。
- 后者固定 `-O3 -mhwmult=f5series`，并检查不要落回某些 helper（如 `__mspabi_slll`、`__mspabi_mpyl_f5hw`），与 O3 下更激进 inline 路径一致。

---

## 8. O2 vs O3 的实战判断矩阵（MSP430）

| 场景 | 推荐 | 理由 |
|---|---|---|
| 大 `switch` + `-mlarge` | 倾向 O3 | O3 才放开 large 模型下 tablejump |
| 乘法热点 + 有 hwmult | 倾向 O3 | O3 才优先 widening multiply inline |
| 无 hwmult 或乘法很少 | O2 常常足够 | target 特有 O3 增益点较少 |
| 强 code size 约束 | 优先 O2/Os 并实测 | O3 可能引入 jump-table 与内联膨胀 |
| 混合目标（速度+体积） | 双构建 profile 对比 | MSP430 后端有多处分支受 `optimize_size` 影响 |

---

## 9. 建议的可复现实验（建议直接落地）

## 9.1 编译与中间文件保留

```bash
msp430-elf-gcc -mmcu=<your_mcu> -O2 -save-temps=obj -fverbose-asm -c foo.c -o foo.O2.o
msp430-elf-gcc -mmcu=<your_mcu> -O3 -save-temps=obj -fverbose-asm -c foo.c -o foo.O3.o
```

## 9.2 反汇编对比 helper 调用与 tablejump

```bash
msp430-elf-objdump -drwC foo.O2.o > foo.O2.S.txt
msp430-elf-objdump -drwC foo.O3.o > foo.O3.S.txt
rg "__mspabi_|tablejump|BR" foo.O2.S.txt foo.O3.S.txt
```

## 9.3 代码体积与段布局

```bash
msp430-elf-size foo.O2.o foo.O3.o
msp430-elf-readelf -S foo.O2.o > sec.O2.txt
msp430-elf-readelf -S foo.O3.o > sec.O3.txt
```

## 9.4 若怀疑 `optimize_size` 被隐式打开

```bash
msp430-elf-gcc -mmcu=<your_mcu> -O2 -Q --help=target | rg "opt|space|hwmult"
msp430-elf-gcc -mmcu=<your_mcu> -O3 -Q --help=target | rg "opt|space|hwmult"
```

---

## 10. 最终结论（工程决策版）

1. 在 TI GCC 9.3.1.11 的 MSP430 target 代码里，O2 与 O3 并非“全面不同”，核心差异集中在少数 `optimize > 2` 门槛。
2. 最有价值的 O3 增益点是：
- `-mlarge` 下的 tablejump
- hwmult 条件满足时 widening multiply inline
3. 若你的 workload 不是“分支分发表 + 乘法热点”，O3 相比 O2 的收益可能有限，且体积风险更高。
4. 评估时务必固定 `-mopt-space/-Os/specs`，避免被 `optimize_size` 路径干扰后误判。

---

## 11. 关键源码定位清单

- `msp430.c:284-289`（`TARGET_OPT_SPACE` 对 `optimize_size` 的影响）
- `msp430.c:1227-1232`（speed cost model 门槛）
- `msp430.c:3467-3508`（常量移位 helper/inline 启发式）
- `msp430.c:3517-3530`（shift expand 与 const variant）
- `msp430.c:4030-4057`（`msp430_has_hwmult`）
- `msp430.md:1377-1382`（tablejump 的 O3/large 条件）
- `msp430.md:1854-1912`（widening multiply expand helper 回落）
- `msp430.md:1915-1969`（widening multiply inline insn 条件）
- `msp430.h:266`（`NO_FUNCTION_CSE`）
- `testsuite/gcc.target/msp430/rtx-cost-O3-default.c`
- `testsuite/gcc.target/msp430/rtx-cost-O3-f5series.c`
