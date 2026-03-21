# MSP430 链接脚本符号表与运行时用途对照

这份文档聚焦 `msp430xl-sim.ld` 与 `msp430xl-sim-rom.ld` 两份 large model 链接脚本，目的是回答两个问题：

1. 每个关键符号是谁定义的？
2. 它在运行时被谁使用、承担什么职责？

建议与以下文档配合使用：

- `docs/msp430-newlib-arch.md`
- `docs/msp430-newlib-startup-io-deepdive.md`
- `docs/msp430-newlib-ascii-maps.md`

---

## 1. 符号体系总览

MSP430 newlib 端口里，链接脚本导出的符号大致分成 6 类：

1. **启动入口类**：`_start`、`__stack`
2. **普通数据初始化类**：`__datastart`、`__romdatastart`、`__romdatacopysize`
3. **普通 BSS 清零类**：`__bssstart`、`__bsssize`
4. **high memory 初始化类**：`__high_datastart`、`__rom_highdatastart`、`__rom_highdatacopysize`、`__upper_data_init`、`__high_bssstart`、`__high_bsssize`
5. **构造/析构数组类**：`__preinit_array_start/end`、`__init_array_start/end`、`__fini_array_start/end`
6. **堆/栈边界类**：`end`、`_end`、`__stack_size`

这些符号的“消费者”主要是：

- `crt0.S`
- `_sbrk()`
- C/C++ 初始化机制

---

## 2. `msp430xl-sim.ld` 关键段与符号表

### 2.1 段布局摘要

`msp430xl-sim.ld` 是 **FRAM memory model** 示例脚本：

- 常规 `.text/.data/.bss/.rodata` 放 `FRAM`
- `.upper.*` 放 `HIFRAM`
- `heap` 与 `stack` 最终也放到 `HIFRAM`

关键定义见：
- `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/msp430xl-sim.ld:131-171`
- `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/msp430xl-sim.ld:248-275`
- `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/msp430xl-sim.ld:344-380`

### 2.2 符号对照表

| 符号 | 定义位置 | 含义 | 运行时使用者 | 用途 |
|---|---|---|---|---|
| `_start` | `msp430xl-sim.ld:281` | 程序入口地址 | 启动入口/链接器 | 作为 `ENTRY(_start)` 对应入口 |
| `__stack` | `msp430xl-sim.ld:389` 附近的 `.stack` | 初始栈顶 | `crt0.S` | `start` 阶段设置 `R1` |
| `__stack_size` | `msp430xl-sim.ld:379` | 预留栈空间估计值 | 链接断言/布局 | 用于检查 stack 余量 |
| `__datastart` | `msp430xl-sim.ld:134` | `.data` 运行地址起点 | `crt0.S` | `movedata` 的目标地址 |
| `__dataend` | `msp430xl-sim.ld:164` | `.data` 结束地址 | 运行时少直接引用 | 调试/边界观察 |
| `__romdatastart` | `msp430xl-sim.ld:170` | `.data` 加载地址 | `crt0.S` | `movedata` 的源地址 |
| `__romdatacopysize` | `msp430xl-sim.ld:171` | `.data` 拷贝长度 | `crt0.S` | `movedata` 的长度 |
| `__high_datastart` | `msp430xl-sim.ld:251` | `.upper.data` 运行地址起点 | `crt0.S` | `move_highdata` 目标/源之一 |
| `__high_dataend` | `msp430xl-sim.ld:253` | `.upper.data` 结束地址 | 调试/边界观察 | high data 边界 |
| `__bssstart` | `msp430xl-sim.ld:261` | `.bss` 起点 | `crt0.S` | `init_bss` 清零起点 |
| `__bssend` | `msp430xl-sim.ld:273` | `.bss` 结束地址 | 调试/边界观察 | 普通 BSS 边界 |
| `__bsssize` | `msp430xl-sim.ld:275` | `.bss` 清零长度 | `crt0.S` | `init_bss` 长度 |
| `__noinit_start/end` | `msp430xl-sim.ld:327-330` | `.noinit` 边界 | 用户/调试 | 表示复位不清零区 |
| `__persistent_start/end` | `msp430xl-sim.ld:338-341` | `.persistent` 边界 | 用户/调试 | 表示加载初始化但复位不重置区 |
| `__high_bssstart` | `msp430xl-sim.ld:352` | `.upper.bss` 起点 | `crt0.S` | `init_highbss` 清零起点 |
| `__high_bssend` | `msp430xl-sim.ld:355` | `.upper.bss` 结束 | 调试/边界观察 | high BSS 边界 |
| `__high_bsssize` | `msp430xl-sim.ld:357` | `.upper.bss` 长度 | `crt0.S` | `init_highbss` 长度 |
| `_end` / `end` | `msp430xl-sim.ld:365-366` | heap 起点 | `_sbrk()` | 线性堆初始位置 |
| `__etext` / `_etext` / `etext` | `msp430xl-sim.ld:303-305` | 文本结束 | 调试/兼容 | 传统 text 边界符号 |
| `__preinit_array_start/end` | `msp430xl-sim.ld:70-72` | preinit 数组边界 | `crt0.S` | `run_preinit_array` |
| `__init_array_start/end` | `msp430xl-sim.ld:74-77` | init 数组边界 | `crt0.S` | `run_init_array` |
| `__fini_array_start/end` | `msp430xl-sim.ld:79-82` | fini 数组边界 | `crt0.S` | `run_fini_array` |

### 2.3 FRAM 模型里 high data 的特点

在 `msp430xl-sim.ld` 中：

- `__high_datastart` 存在
- 但 `__upper_data_init`、`__rom_highdatastart`、`__rom_highdatacopysize` **没有真实强定义**

见 `msp430xl-sim.ld:248-254`。

这意味着：

- `crt0.S` 中 `move_highdata` 仍然可能被链接进来
- 但如果链接脚本没有提供 high data 初始化镜像符号，则会退回到 `crt0.S` 里的 weak 0 值
- 启动逻辑因此被安全短路

对应 weak 定义见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/crt0.S:139-146` 之前的 `L0` 弱符号区，以及 `move_highdata` 的长度检查见 `crt0.S:144-146`

这正是 FRAM 脚本与 ROM 脚本的最大差别之一：

- FRAM 模式强调“数据可能天然就在可执行/可写持久区里”
- ROM 模式强调“运行前必须搬运镜像”

---

## 3. `msp430xl-sim-rom.ld` 关键段与符号表

### 3.1 段布局摘要

`msp430xl-sim-rom.ld` 是 **ROM memory model** 示例脚本：

- `.text/.rodata` 在 `ROM/HIROM`
- `.data` 在 `RAM`，但加载镜像在 `ROM`
- `.upper.data` 在 `HIFRAM`，但加载镜像在 `HIROM`
- `heap` 与 `stack` 放在 `HIFRAM`

关键定义见：
- `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/msp430xl-sim-rom.ld:133-175`
- `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/msp430xl-sim-rom.ld:252-265`
- `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/msp430xl-sim-rom.ld:310-363`

### 3.2 符号对照表

| 符号 | 定义位置 | 含义 | 运行时使用者 | 用途 |
|---|---|---|---|---|
| `_start` | `msp430xl-sim-rom.ld:367` | 程序入口地址 | 启动入口/链接器 | 程序入口 |
| `__stack` | `msp430xl-sim-rom.ld:357` | 初始栈顶 | `crt0.S` | `R1 = __stack` |
| `__stack_size` | `msp430xl-sim-rom.ld:345` | 栈预留大小 | 链接断言 | 校验 heap/stack 间距 |
| `__datastart` | `msp430xl-sim-rom.ld:136` | `.data` 的 VMA | `crt0.S` | `movedata` 目标地址 |
| `__romdatastart` | `msp430xl-sim-rom.ld:174` | `.data` 的 LMA | `crt0.S` | `movedata` 源地址 |
| `__romdatacopysize` | `msp430xl-sim-rom.ld:175` | `.data` 拷贝大小 | `crt0.S` | `movedata` 长度 |
| `__upper_data_init` | `msp430xl-sim-rom.ld:256` | upper.data 状态字地址 | `crt0.S` | 控制 `move_highdata` 拷贝方向 |
| `__high_datastart` | `msp430xl-sim-rom.ld:259` | `.upper.data` 的 VMA | `crt0.S` | `move_highdata` 目标/源之一 |
| `__high_dataend` | `msp430xl-sim-rom.ld:261` | `.upper.data` 结束 | 调试/边界观察 | high data 边界 |
| `__rom_highdatacopysize` | `msp430xl-sim-rom.ld:264` | `.upper.data` 实际镜像长度 | `crt0.S` | `move_highdata` 长度 |
| `__rom_highdatastart` | `msp430xl-sim-rom.ld:265` | `.upper.data` 镜像数据起点 | `crt0.S` | `move_highdata` 源地址 |
| `__bssstart` | `msp430xl-sim-rom.ld:272` | `.bss` 起点 | `crt0.S` | `init_bss` 起点 |
| `__bssend` | `msp430xl-sim-rom.ld:284` | `.bss` 结束 | 调试/边界观察 | 普通 BSS 边界 |
| `__bsssize` | `msp430xl-sim-rom.ld:286` | `.bss` 长度 | `crt0.S` | `init_bss` 长度 |
| `__noinit_start/end` | `msp430xl-sim-rom.ld:293-296` | `.noinit` 边界 | 用户/调试 | 复位不清零区 |
| `__persistent_start/end` | `msp430xl-sim-rom.ld:304-307` | `.persistent` 边界 | 用户/调试 | HIFRAM 持久区 |
| `__high_bssstart` | `msp430xl-sim-rom.ld:318` | `.upper.bss` 起点 | `crt0.S` | `init_highbss` 起点 |
| `__high_bssend` | `msp430xl-sim-rom.ld:321` | `.upper.bss` 结束 | 调试/边界观察 | high BSS 边界 |
| `__high_bsssize` | `msp430xl-sim-rom.ld:323` | `.upper.bss` 长度 | `crt0.S` | `init_highbss` 长度 |
| `_end` / `end` | `msp430xl-sim-rom.ld:331-332` | heap 起点 | `_sbrk()` | 高地址堆起点 |
| `__etext` / `_etext` / `etext` | `msp430xl-sim-rom.ld:390-392` | 文本结束 | 调试/兼容 | 传统 text 边界符号 |
| `__preinit_array_start/end` | `msp430xl-sim-rom.ld:72-74` | preinit 数组边界 | `crt0.S` | `run_preinit_array` |
| `__init_array_start/end` | `msp430xl-sim-rom.ld:76-79` | init 数组边界 | `crt0.S` | `run_init_array` |
| `__fini_array_start/end` | `msp430xl-sim-rom.ld:81-84` | fini 数组边界 | `crt0.S` | `run_fini_array` |

### 3.3 `__upper_data_init` / `__rom_highdatastart` / `__rom_highdatacopysize` 三件套

这 3 个符号是 large ROM model 最关键的协议：

- `__upper_data_init`：状态字位置
- `__rom_highdatastart`：upper data 镜像的真实数据起点
- `__rom_highdatacopysize`：需要复制的字节数

对应 `crt0.S` 的 `move_highdata` 逻辑：

| `crt0.S` 操作 | 依赖符号 | 说明 |
|---|---|---|
| `mov.w #__rom_highdatacopysize, R14` | `__rom_highdatacopysize` | 长度为 0 则直接跳过 |
| `cmpx.w #0, &__upper_data_init` | `__upper_data_init` | 判断当前初始化状态 |
| `mov_ #__high_datastart, R12` | `__high_datastart` | upper.data 的目标地址 |
| `mov_ #__rom_highdatastart, R13` | `__rom_highdatastart` | ROM/影子镜像源地址 |
| `call_ #memmove` | 上述三者 | 完成初始化搬运 |

参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/crt0.S:139-166`。

ROM 脚本显式定义这些值，才能使 `move_highdata` 真正生效。

---

## 4. `crt0.S` 与链接脚本的符号消费矩阵

### 4.1 启动阶段 -> 链接脚本符号

| 启动阶段 | `crt0.S` 行号 | 依赖符号 | 来源 |
|---|---|---|---|
| `start` | `crt0.S:53-60` | `__stack` | `.stack` 段 |
| `init_bss` | `crt0.S:73-83` | `__bssstart`, `__bsssize` | `.bss` |
| `init_highbss` | `crt0.S:92-103` | `__high_bssstart`, `__high_bsssize` | `.upper.bss` |
| `movedata` | `crt0.S:113-125` | `__datastart`, `__romdatastart`, `__romdatacopysize` | `.data` + `LOADADDR/SIZEOF` |
| `move_highdata` | `crt0.S:139-166` | `__upper_data_init`, `__high_datastart`, `__rom_highdatastart`, `__rom_highdatacopysize` | `.upper.data` 协议 |
| `run_preinit_array` | `crt0.S:173-180` | `__preinit_array_start/end` | `.preinit_array` |
| `run_init_array` | `crt0.S:186-193` | `__init_array_start/end` | `.init_array` |
| `run_fini_array` | `crt0.S:278-285` | `__fini_array_start/end` | `.fini_array` |
| `run_smi_location_init_array` | `crt0.S:214-245` | `__smi_location_init_array_start` | 特殊初始化表 |

### 4.2 非启动路径 -> 链接脚本符号

| 运行时代码 | 行号 | 依赖符号 | 说明 |
|---|---|---|---|
| `_sbrk()` | `sbrk.c:17-21` | `end` | 堆起点 |
| `_sbrk()` | `sbrk.c:27-34` | 当前栈地址（局部变量近似） | 检查 heap/stack 冲突 |

参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/sbrk.c:17-40`。

---

## 5. 哪些符号在 FRAM 脚本与 ROM 脚本中语义不同

| 符号 | FRAM 脚本 | ROM 脚本 | 影响 |
|---|---|---|---|
| `__romdatastart` | 与 `.data` 运行区接近，更多是形式上的 `LOADADDR(.data)` | 真正指向 ROM 镜像 | `movedata` 在 ROM model 中是实质拷贝 |
| `__romdatacopysize` | 一般可为 `.data` 大小，但数据可能本就在 FRAM | 明确是从 ROM 搬入 RAM 的字节数 | `movedata` 的必要性不同 |
| `__upper_data_init` | 通常无强定义，依赖 weak 0 | 明确定义在 `.upper.data` 头部 | `move_highdata` 在 ROM model 才真正起作用 |
| `__rom_highdatastart` | 通常无强定义 | 明确指向 HIROM 中的 upper.data 镜像 | high data 初始化是否真实发生 |
| `__rom_highdatacopysize` | 通常无强定义 | 明确定义为 `SIZEOF(.upper.data)-2` | high data 的复制长度 |
| `end` | HIFRAM 中 `.heap_start` 后 | 同样位于 HIFRAM `.heap_start` 后 | `_sbrk` 都从高地址区起堆 |

---

## 6. 反查源码时怎么用这张表

如果你在源码里看到一个符号，可以这样反查：

### 6.1 看到 `__romdatacopysize`

反查路径：

```text
crt0.S movedata
   -> 查链接脚本里的 SIZEOF(.data)
   -> 再看 .data 是 > FRAM 还是 > RAM AT> ROM
   -> 就知道这次是“形式搬运”还是“真实搬运”
```

### 6.2 看到 `__upper_data_init`

反查路径：

```text
crt0.S move_highdata
   -> 查脚本是否真的强定义了 __upper_data_init
   -> 如果没有，大概率是 FRAM/无镜像场景
   -> 如果有，再查 __rom_highdatastart / __rom_highdatacopysize
```

### 6.3 看到 `end`

反查路径：

```text
sbrk.c
   -> 查脚本里 end / _end 的位置
   -> 确认 heap 实际从哪个 memory region 开始
   -> 再看 __stack 在哪，判断 heap/stack 是否同域
```

---

## 7. 最值得记住的符号清单

如果只想背最重要的一组，我建议记这 10 个：

- `__stack`
- `__datastart`
- `__romdatastart`
- `__romdatacopysize`
- `__bssstart`
- `__bsssize`
- `__high_datastart`
- `__rom_highdatastart`
- `__rom_highdatacopysize`
- `end`

理解了这 10 个，MSP430 newlib 启动和内存初始化逻辑基本就通了。

