# MSP430 Newlib 架构手册

版本基线：`msp430-gcc-9.3.1.11-source-full/newlib`

这份手册是对前面分析结果的统一收敛版本，目标是把 MSP430 端口在 newlib 中的核心设计一次讲清楚：

- 它有哪些源码组成部分
- 启动链路如何工作
- 链接脚本与运行时如何协作
- I/O / syscall / CIO 如何分工
- 哪些链接符号是理解整个端口的关键
- 后续如果继续读源码，应该按什么顺序推进

如果你只保留一份文档，我建议保留这份。

配套细分材料：

- 总览版：`docs/msp430-newlib-arch.md`
- 启动/I-O 深挖版：`docs/msp430-newlib-startup-io-deepdive.md`
- ASCII 图解版：`docs/msp430-newlib-ascii-maps.md`
- 链接符号参考表：`docs/msp430-linker-symbol-reference.md`

---

## 1. 一句话结论

**MSP430 的 newlib 端口本质上是一个“尺寸优先、裸机优先、按需链接”的最小 C 运行时系统：libc 机器层极薄，真正的运行时核心在 `libgloss/msp430`，通过片段化 `crt0`、small/large 双模型抽象、链接脚本符号协议以及 CIO 半主机 I/O，把 newlib 收缩到适合 MCU 的规模。**

---

## 2. 架构总览

### 2.1 三层结构

可以把整个 MSP430 newlib 端口分成三层：

### 第一层：CPU/ABI 适配层

位置：`newlib/newlib/libc/machine/msp430`

职责：

- `setjmp/longjmp`
- 极简 `printf`
- 极简 `puts`
- 若干 `stdio/nano-vfprintf` 内部头复用

源码非常少，只编译三个源文件：

- `setjmp.S`
- `tiny-puts.c`
- `tiny-printf.c`

参见 `msp430-gcc-9.3.1.11-source-full/newlib/newlib/libc/machine/msp430/Makefile.am:22-25`。

### 第二层：运行时装配层

位置：`newlib/libgloss/msp430`

职责：

- reset 接管
- 栈初始化
- `.data/.bss/.upper.*` 初始化
- 构造/析构数组调度
- heap/stack 边界建立
- 提供启动对象与链接脚本

关键文件：

- `crt0.S`
- `memmodel.h`
- `msp430-sim.ld`
- `msp430xl-sim.ld`
- `msp430xl-sim-rom.ld`
- `intr_vectors.ld`

### 第三层：宿主交互层

位置：`newlib/libgloss/msp430`

职责：

- 模拟器 syscall
- 无 OS 环境的默认 syscall stub
- CIO 半主机 I/O
- 最小 `sbrk`
- 裸机退出语义

关键文件：

- `syscalls.S`
- `ciosyscalls.S`
- `cio.S`
- `cio.h`
- `write.c`
- `unlink.c`
- `sbrk.c`
- `exit.c`

---

## 3. 构建策略与端口取向

`configure.host` 中，MSP430 目标会被赋予非常鲜明的默认策略：

- `-Os`
- `-mOs`
- `-DPREFER_SIZE_OVER_SPEED`
- `-DSMALL_MEMORY`
- `-ffunction-sections -fdata-sections`
- `-mhwmult=none`
- `default_newlib_nano_malloc=yes`

参见 `msp430-gcc-9.3.1.11-source-full/newlib/newlib/configure.host:79-88,232-238`。

这组配置说明：

1. **代码尺寸优先**，不是性能优先
2. **默认面向资源最紧张的器件**
3. **默认依赖链接器 GC 回收无用代码段**
4. **尽量不假定高配硬件乘法器存在**

因此，MSP430 端口从一开始就不是“完整 libc + 适配缩减”，而是“从一开始就按 MCU 最小运行时目标来设计”。

---

## 4. 数据模型与 ABI 适配

### 4.1 `small` 与 `large` memory model

`memmodel.h` 用一组抽象宏统一了两种地址模型：

- `call_` -> `CALL` / `CALLA`
- `ret_` -> `RET` / `RETA`
- `mov_` -> `MOV` / `MOVA`
- `movx_` -> `MOV` / `MOVX`
- `br_` -> `BR` / `BRA`
- `cmp_` -> `CMP` / `CMPA`
- `add_` -> `ADD` / `ADDA`
- `PTRsz` -> `2` / `4`

参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/memmodel.h:18-38`。

这让同一份：

- `crt0.S`
- `syscalls.S`
- `cio.S`

可以同时兼容 16-bit 与 20-bit 地址模型。

### 4.2 newlib 内部对 MSP430 的通用配置

`sys/config.h` 针对 MSP430 定义：

- `_REENT_SMALL`
- `__BUFSIZ__ = 256`
- `__SMALL_BITFIELDS`
- `_POINTER_INT = int`（small）或 `__int20`（large）

参见 `msp430-gcc-9.3.1.11-source-full/newlib/newlib/libc/include/sys/config.h:153-166`。

这说明 newlib 自身也在内部为 MSP430 做了“小 reent、小缓冲、可容纳 20-bit 指针整数”的内存约束适配。

### 4.3 `jmp_buf` 语义

`machine/setjmp.h` 为 MSP430 定义：

- `_JBLEN = 9`
- small model: `_JBTYPE = unsigned short`
- large model: `_JBTYPE = unsigned long`

参见 `msp430-gcc-9.3.1.11-source-full/newlib/newlib/libc/include/machine/setjmp.h:308-316`。

与之配套的 `setjmp.S` 保存：

- PC (`r0`)
- SP (`r1`)
- `r4-r10`

参见 `msp430-gcc-9.3.1.11-source-full/newlib/newlib/libc/machine/msp430/setjmp.S:15-27,31-105`。

这体现的是“最小 C 调用上下文恢复”，而不是完整 CPU 快照。

---

## 5. 启动系统：`crt0.S` 不是单函数，而是启动流水线

### 5.1 片段化 `crt0`

MSP430 端口最有特色的设计，是把启动逻辑拆成多个独立 `.crt_*` section。

`memmodel.h` 定义：

- `START_CRT_FUNC number name`
- `END_CRT_FUNC name`

每个启动步骤都进入单独的 `.crt_XXXXname` 段。参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/memmodel.h:42-58`。

构建规则又把同一个 `crt0.S` 多次编译成不同对象：

- `crt_bss.o`
- `crt_movedata.o`
- `crt_main.o`
- `crt_run_init_array.o`
- ...

参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/Makefile.in:83-107`。

最后，链接脚本用：

```ld
KEEP(*(SORT(.crt_*)))
```

把它们拼回 `.text`，同时保持编号顺序。参见：
- `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/msp430-sim.ld:96-100`
- `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/msp430xl-sim.ld:278-284`
- `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/msp430xl-sim-rom.ld:365-370`

### 5.2 这样做的价值

这种设计带来 4 个收益：

1. **启动顺序稳定**：靠 section 名排序保证顺序
2. **可按需回收**：不用的阶段不必进入最终镜像
3. **便于插入用户启动逻辑**：用户可以插 `.crt_05xx_*`
4. **便于 small/large 共用**：流程结构不变，只换指令宏和脚本符号

---

## 6. 从 Reset 到 `main` 的完整启动链路

完整时序可以概括为：

```text
Reset
 -> VECT31
 -> .resetvec
 -> __crt0_start
 -> 设置 SP = __stack
 -> init_bss
 -> init_highbss (large)
 -> movedata
 -> move_highdata (large)
 -> run_preinit_array
 -> run_init_array
 -> run_smi_location_init_array
 -> call_main (argc=0)
 -> call_exit
 -> run_fini_array (如果启用)
```

### 6.1 reset 向量接管

`crt0.S` 在 `L0` 分支中建立 `.resetvec`，写入 `__crt0_start`。参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/crt0.S:22-25,53-60`。

`intr_vectors.ld` 把 `VECT31` 放在 `0xFFFE`，并允许：

- `__interrupt_vector_reset`
- `.resetvec`

都落入其中。参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/intr_vectors.ld:33-35,69-70`。

### 6.2 栈初始化

`start` 阶段做的最关键动作是：

```asm
mov_ #__stack, R1
```

参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/crt0.S:53-60`。

因此 `__stack` 是整个运行时最关键的链接脚本符号之一。

### 6.3 `.bss` / `.data` 初始化

- `init_bss` 用 `__bssstart` + `__bsssize` 调 `memset`
- `movedata` 用 `__datastart` + `__romdatastart` + `__romdatacopysize` 调 `memmove`

参见：
- `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/crt0.S:73-83`
- `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/crt0.S:113-125`

### 6.4 large model 的 `.upper.*`

在 `__MSP430X_LARGE__` 下，还会出现：

- `init_highbss`
- `move_highdata`

其中 `move_highdata` 最复杂，它依赖：

- `__upper_data_init`
- `__high_datastart`
- `__rom_highdatastart`
- `__rom_highdatacopysize`

参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/crt0.S:130-168`。

### 6.5 `main` 入口非常 bare-metal

`call_main` 中只做：

- `clr.w R12` -> `argc = 0`
- `call_ #main`

参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/crt0.S:250-255`。

这说明它默认服务的是：

```c
int main(void)
```

风格的 MCU 程序，而不是完整命令行环境。

---

## 7. 链接脚本：它不是附属品，而是运行时协议的一部分

MSP430 newlib 端口的一个关键理解点是：

**链接脚本不是“把段摆进去”这么简单，而是和 `crt0.S` 共同组成完整的运行时协议。**

### 7.1 普通符号协议

最基础的一组：

- `__stack`
- `__datastart`
- `__romdatastart`
- `__romdatacopysize`
- `__bssstart`
- `__bsssize`
- `end`

这些符号被：

- `crt0.S`
- `_sbrk()`

直接消费。

### 7.2 构造/析构数组协议

脚本还必须导出：

- `__preinit_array_start/end`
- `__init_array_start/end`
- `__fini_array_start/end`

由 `run_preinit_array` / `run_init_array` / `run_fini_array` 使用。参见：
- `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/crt0.S:170-194`
- `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/crt0.S:274-285`

### 7.3 `run_array` 的通用遍历器设计

`run_array` 本体只通过 `(current, stop, stride)` 遍历函数指针数组：

- 正向遍历：用于 preinit/init
- 逆向遍历：用于 fini

参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/crt0.S:288-303`。

这是个高度节省代码的设计。

---

## 8. `msp430xl-sim.ld` 与 `msp430xl-sim-rom.ld` 的差别

### 8.1 `msp430xl-sim.ld`：FRAM 模型

特点：

- 常规 `.text/.data/.bss/.rodata` 放 `FRAM`
- `.upper.*` 放 `HIFRAM`
- `.data` 是 `> FRAM`
- `.upper.data` 也是直接 `> HIFRAM`

参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/msp430xl-sim.ld:131-171,248-275`。

这更像“运行数据本身就在持久可写区”的模型。

### 8.2 `msp430xl-sim-rom.ld`：ROM 模型

特点：

- `.text/.rodata` 在 `ROM/HIROM`
- `.data > RAM AT> ROM`
- `.upper.data > HIFRAM AT> HIROM`
- `heap/stack` 在 `HIFRAM`

参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/msp430xl-sim-rom.ld:133-175,252-265,325-363`。

这意味着：

- 普通 `.data` 必须从 ROM 搬到 RAM
- `upper.data` 也必须从 HIROM 搬到 HIFRAM
- 运行时对链接符号的依赖更强

### 8.3 `.either.*` 的特殊点

在这两份 large model 脚本里，多次强调：不要主动定义 `.either.data/.either.bss/.either.text/.either.rodata`，否则会破坏链接器对 `-mdata-region=either` 的自动放置策略。参见：
- `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/msp430xl-sim.ld:145-148,266-269,294-297`
- `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/msp430xl-sim-rom.ld:147-150,277-280,381-384`

这再次说明：链接脚本在这个端口里是“主动逻辑参与者”，不是静态摆放器。

---

## 9. I/O 架构：syscall 与 CIO 双通道并存

### 9.1 为什么 `write` 是核心

MSP430 端口里的：

- `puts`
- `printf`
- `_sbrk` 错误消息

最终都依赖 `write()`。参见：
- `msp430-gcc-9.3.1.11-source-full/newlib/newlib/libc/machine/msp430/tiny-puts.c:8-18`
- `msp430-gcc-9.3.1.11-source-full/newlib/newlib/libc/machine/msp430/tiny-printf.c:86-94,258-271`
- `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/sbrk.c:29-33`

所以这个端口最重要的 I/O 问题不是“有没有文件系统”，而是“有没有最小输出能力”。

### 9.2 模拟器 syscall 路径

`syscalls.S` 把：

- `_exit`
- `exit`
- `open`
- `close`
- `read`
- `fstat`
- `lseek`
- `kill`

映射为低地址：`0x180 + SYS_xxx`。参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/syscalls.S:31-51`。

模拟器通过捕获对这片低地址的 `CALL` 来实现 syscall。

### 9.3 真实硬件默认 syscall stub

`ciosyscalls.S` 里：

- `isatty()` 固定返回 1
- `getpid()` 固定返回 42
- 大多数 syscall 设置 `errno = ENOSYS` 并返回 `-1`

参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/ciosyscalls.S:49-78`。

### 9.4 为什么 `write` 不走普通 syscall

无论 `syscalls.S` 还是 `ciosyscalls.S`，都故意注释掉了 `SC(write)`。参见：
- `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/syscalls.S:48`
- `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/ciosyscalls.S:75`

取而代之的是 `write.c` 走 **CIO 半主机协议**。

这是一个非常关键的架构决策：

- 普通文件/process 语义可以没有
- 但调试输出必须优先保住

### 9.5 CIO 协议

`cio.h` 定义：

- `__CIOBUF__` 缓冲布局
- `CIO_WRITE`
- `CIO_UNLINK`
- 其他宿主命令码

参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/cio.h:17-41`。

`write.c` 的流程是：

1. 把命令和参数写入 `__CIOBUF__`
2. 调 `_libgloss_cio_hook()`
3. 由宿主/调试器识别 `C$$IO$$`
4. 返回结果写回 `parms[]`

参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/write.c:18-32,37-60` 与 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/cio.S:18-33`。

这意味着 MSP430 newlib 的最关键输出路径是：

```text
printf/puts -> write -> CIO buffer -> 调试器/仿真宿主
```

而不是 UART 驱动。

### 9.6 `unlink` 复用 CIO

`unlink.c` 也完全复用同一套 CIO 协议。参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/unlink.c:5-22`。

所以 CIO 不是单纯 stdout 通道，而是一组最小宿主服务接口。

---

## 10. Heap 与 Exit：典型 bare-metal 语义

### 10.1 `_sbrk()`

`sbrk.c` 的实现非常小：

- `end` 作为堆起点
- `_sbrk_heap` 记录当前堆顶
- 用局部变量地址近似当前 SP
- 发现 heap/stack 冲突后，打印错误并 `abort()`

参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/sbrk.c:17-40`。

它只适用于：

- 单栈
- 线性地址空间
- 资源简单的 bare-metal 场景

### 10.2 `exit()`

`exit.c` 里：

- `exit()` 是 `naked weak`
- 默认进入死循环
- `_exit` 只是别名

参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/exit.c:1-12`。

这不是“缺少实现”，而是 MCU 世界里最典型的退出语义：程序结束即停住，等待调试器或人工复位。

---

## 11. 必须记住的 10 个关键链接符号

如果只想把这套端口真正记住，我建议先记住这 10 个：

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

理解它们，就能串起：

- 启动
- 数据搬运
- high memory 初始化
- heap 建立

---

## 12. 推荐阅读顺序

如果后续你要继续深入源码，建议按这个顺序：

1. `newlib/newlib/configure.host`
   - 先理解 MSP430 端口的构建取向
2. `newlib/libgloss/msp430/memmodel.h`
   - 先掌握 small/large 抽象
3. `newlib/libgloss/msp430/crt0.S`
   - 启动系统的核心
4. `newlib/libgloss/msp430/msp430xl-sim.ld`
   - 看 FRAM 模型
5. `newlib/libgloss/msp430/msp430xl-sim-rom.ld`
   - 看 ROM 模型
6. `newlib/libgloss/msp430/syscalls.S` / `ciosyscalls.S` / `write.c` / `cio.S`
   - 理解宿主交互与调试输出
7. `newlib/newlib/libc/machine/msp430/setjmp.S` / `tiny-printf.c`
   - 看 ABI 与极简 libc 适配

---

## 13. 如何使用这份手册与附录文档

如果你的目标不同，可以这样组合使用：

- **想快速理解整体**：先看本手册，再看 `docs/msp430-newlib-ascii-maps.md`
- **想深挖启动与链接**：本手册第 5~8 章 + `docs/msp430-newlib-startup-io-deepdive.md`
- **想查某个符号是干什么的**：直接看 `docs/msp430-linker-symbol-reference.md`
- **想做 PPT/分享/复习**：优先使用 `docs/msp430-newlib-ascii-maps.md`

所以这份手册适合当“正文”，其余几份更像“附录”和“索引页”。

---

## 14. 最终总结

MSP430 newlib 端口的核心并不在“实现了多少 libc 功能”，而在于它如何把一个桌面世界里的 C 运行时，压缩成适合 MCU 的最小系统：

- 用极薄的 `libc/machine/msp430` 做 ABI 补位
- 用 `libgloss/msp430` 承担真实运行时职责
- 用片段化 `crt0` 做可裁剪启动流水线
- 用链接脚本导出符号构成运行时协议
- 用 CIO 半主机通道优先保障调试输出
- 用最小 `_sbrk` / `exit` 保持裸机语义清晰

**从这个角度看，MSP430 的 newlib 端口不是“缩小版 POSIX 环境”，而是一套围绕 MCU 启动、初始化、最小输出与调试可观测性精心组织的 bare-metal C runtime。**

