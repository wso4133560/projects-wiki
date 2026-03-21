# TI GCC 9.3.1.11 MSP430 优化调优手册

> 版本基线：TI GCC 9.3.1.11（GCC 9.3.0 + TI patch，BASE-VER 9.3.1）  
> 适用范围：`msp430-elf-gcc` C/C++ 固件工程（small/large model，430/430X/430Xv2）

---

## 1. 使用方式（先定目标再配参数）

先在三种目标里选一个主目标：

1. 极致代码体积（Flash 紧张）
2. 平衡（大多数量产项目）
3. 极致性能（热点计算明显）

然后按本手册对应模板落地，最后做“反汇编验证 + map 验证 + 基准测试”。

---

## 2. 参数模板（可直接复制）

## 2.1 模板 A：极致代码体积（推荐起点）

```bash
CFLAGS="-Os -ffunction-sections -fdata-sections -mrelax -mmcu=<your_mcu> -mhwmult=auto"
LDFLAGS="-Wl,--gc-sections -Wl,--print-memory-usage"
```

适用：

- Flash 容量紧张
- 计算热点不明显
- 功耗和可维护性优先

可选增强：

- 需要进一步减小 `printf` 体积时加 `-mtiny-printf`（需 newlib nano formatted io 支持）

风险提醒：

- `-mtiny-printf` 非重入，不适合多线程/复杂并发输出场景

## 2.2 模板 B：平衡（默认推荐）

```bash
CFLAGS="-O2 -ffunction-sections -fdata-sections -mrelax -mmcu=<your_mcu> -mhwmult=auto"
LDFLAGS="-Wl,--gc-sections -Wl,--print-memory-usage"
```

适用：

- 大部分业务固件
- 既要控制体积，也要保持较好性能

可选调参：

- Shift 热点明显可调 `-mmax-inline-shift=6..12`（默认等效阈值为 4）

## 2.3 模板 C：极致性能（有硬件乘法）

```bash
CFLAGS="-O3 -ffunction-sections -fdata-sections -mrelax -mmcu=<your_mcu> -mhwmult=auto"
LDFLAGS="-Wl,--gc-sections -Wl,--print-memory-usage"
```

适用：

- 算法热点重（乘法/移位密集）
- 可接受更大代码尺寸

预期现象：

- widening multiply 更可能内联
- 常量 shift 更可能 inline

风险提醒：

- 体积显著上升，先确认 Flash 预算

## 2.4 模板 D：Large model 且高地址安全优先

```bash
CFLAGS="-O2 -mlarge -muse-430x-for-mem -mmcu=<your_mcu> -mhwmult=auto"
LDFLAGS="-Wl,--gc-sections"
```

适用：

- 大地址空间布局复杂
- 不希望出现“本可 430 指令但地址上界不确定”的风险

风险提醒：

- 一般会增大代码尺寸

---

## 3. 关键选项怎么选

## 3.1 `-mmcu` / `-mcpu`

建议：

- 优先只给 `-mmcu=<具体型号>`
- 让 driver 自动推导 `-mcpu` 与 `-lmul_*`

原因：

- 后端会根据 MCU 数据选择 ISA/hwmult 和链接脚本

## 3.2 `-mhwmult`

建议顺序：

1. `auto`（首选）
2. 明确指定 `16bit/32bit/f5series`（仅当你确定器件能力且希望强约束）
3. `none`（用于对比/保守验证）

## 3.3 `-mmax-inline-shift`

经验值：

- 体积优先：`2~4`
- 平衡：`4~8`
- 性能优先：`8~16`

注意：

- 该阈值对 SImode 有折算逻辑，别按 16 位直觉直接套用

## 3.4 `-mcode-region` / `-mdata-region`

建议：

- 若无强制布局需求，先用默认（让链接脚本主导）
- 要求 low/upper/either 精细布局时再启用

约束：

- `upper/either` 需要 `-mlarge`

## 3.5 `-mrelax`

建议：

- 量产默认开启
- 会传递给 assembler + linker，通常有正收益

---

## 4. 三阶段调优流程（实战）

## 4.1 第一步：建立基线

固定：

- 编译器版本
- 链接脚本
- CFLAGS/LDFLAGS
- 输入数据集

记录：

- `.text/.data/.bss` 大小
- 关键函数周期/耗时
- 功耗（如有）

## 4.2 第二步：单变量扫描

每次只改一个开关，例如：

1. `-O2 -> -Os`
2. `-mmax-inline-shift=4 -> 8`
3. `-mhwmult=auto -> f5series`
4. `-muse-430x-for-mem` 开/关

每次记录同一套指标，避免多变量耦合误判。

## 4.3 第三步：固化配置

输出：

- 一个“默认构建模板”
- 一个“性能构建模板”
- 一个“回归验证脚本”

并在 CI 中固定，防止参数漂移。

---

## 5. 你必须做的验证

## 5.1 反汇编验证（确认优化是否发生）

建议命令：

```bash
msp430-elf-objdump -drwC firmware.elf > firmware.S
```

重点检查：

- shift 是否调用 `__mspabi_*` 还是 inline
- 乘法是否走 `*_hw/_hw32/_f5hw` 路径
- large model 下关键内存访问是否出现 `MOVX/430X` 形式

## 5.2 map 验证（确认体积收益点）

建议命令：

```bash
msp430-elf-gcc ... -Wl,-Map,firmware.map
```

重点看：

- `libmul_*` 实际链接了哪套
- 哪些对象占用了主要 `.text`
- `--gc-sections` 是否有效剔除了 unused section

## 5.3 功能验证（优化安全性）

必须覆盖：

- ISR 路径
- 启动与低功耗唤醒路径
- 关键算术边界（移位位数、乘法溢出附近输入）

---

## 6. 典型问题与处理

## 6.1 体积突然暴涨

先查：

1. 是否从 `-Os` 切到了 `-O3`
2. `-mmax-inline-shift` 是否设太大
3. widening multiply 是否触发 inline
4. `-muse-430x-for-mem` 是否打开

处理：

- 先回到 `-O2` 或 `-Os`
- 降低 `-mmax-inline-shift`
- 检查 `-mhwmult` 是否误设

## 6.2 性能不升反降

常见原因：

- 代码体积增加导致指令路径更差
- 关键热点未实际命中目标优化路径

处理：

- 用反汇编核对热点函数
- 对热点文件单独 `-O3`，全局仍 `-O2`/`-Os`

## 6.3 large model 下寻址异常或行为不稳

处理：

- 开启 `-muse-430x-for-mem` 做安全对比
- 检查 section/region 与 linker 脚本匹配

## 6.4 `printf` 体积过大

处理：

- 优先 `-mtiny-printf`（满足库前提时）
- 把日志路径改为轻量输出接口

---

## 7. 推荐配置矩阵（快速决策）

| 场景 | 推荐优化级别 | 关键选项 | 风险 |
|---|---|---|---|
| 低端 Flash 紧张 | `-Os` | `-ffunction-sections -fdata-sections -mrelax -Wl,--gc-sections` | 峰值性能一般 |
| 通用量产 | `-O2` | 同上 + `-mhwmult=auto` | 需要做回归验证 |
| 算法热点重 | `-O3` | `-mhwmult=auto` + 调 `-mmax-inline-shift` | 体积增大明显 |
| 大地址安全优先 | `-O2` | `-mlarge -muse-430x-for-mem` | 代码尺寸增加 |
| 日志体积受限 | `-Os` | `-mtiny-printf` | 非重入限制 |

---

## 8. Makefile 片段（可直接改）

```make
# profile: size / balanced / speed / large-safe
PROFILE ?= balanced
MMCU    ?= msp430f5529

COMMON_CFLAGS := -mmcu=$(MMCU) -ffunction-sections -fdata-sections -mrelax -mhwmult=auto
COMMON_LDFLAGS := -Wl,--gc-sections -Wl,-Map,firmware.map -Wl,--print-memory-usage

ifeq ($(PROFILE),size)
  CFLAGS += -Os $(COMMON_CFLAGS)
  LDFLAGS += $(COMMON_LDFLAGS)
endif

ifeq ($(PROFILE),balanced)
  CFLAGS += -O2 $(COMMON_CFLAGS)
  LDFLAGS += $(COMMON_LDFLAGS)
endif

ifeq ($(PROFILE),speed)
  CFLAGS += -O3 $(COMMON_CFLAGS) -mmax-inline-shift=12
  LDFLAGS += $(COMMON_LDFLAGS)
endif

ifeq ($(PROFILE),large-safe)
  CFLAGS += -O2 $(COMMON_CFLAGS) -mlarge -muse-430x-for-mem
  LDFLAGS += $(COMMON_LDFLAGS)
endif
```

---

## 9. 与源码文档联动阅读

建议配合这两份文档：

- `TI_GCC_9.3.1.11_MSP430_源码与优化深度文档.md`
- `TI_GCC_9.3.1.11_MSP430_优化函数逐条注释.md`

推荐阅读顺序：

1. 先看本手册直接落参数
2. 再看“逐条注释”定位具体函数逻辑
3. 最后看“深度文档”建立全局模型

---

## 10. 最小落地清单（给团队）

1. 固定 `PROFILE=size/balanced/speed` 三套配置。
2. 每套配置都保留 map + objdump 产物。
3. 在 CI 做尺寸门限与关键函数汇编模式检查。
4. 每次升级 toolchain 都做同口径 A/B 对比。

