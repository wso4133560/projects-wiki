# MSP430 `newlib` 启动、链接与 I/O 深入分析

这份文档是 `docs/msp430-newlib-arch.md` 的续篇，专门深入 3 件事：

1. `crt0.S` 为什么能做到“按需拼装”的启动链路
2. `*.ld` 链接脚本怎样与启动代码协同
3. `printf -> write -> CIO` 与 `syscall` 两条 I/O 路径到底怎么分工

分析对象仍然是：`msp430-gcc-9.3.1.11-source-full/newlib`。

---

## 1. 启动系统的核心思想：把 CRT 拆成可垃圾回收的段

MSP430 端口没有采用“一个大 `crt0` 函数串行调用一切”的做法，而是把每个启动步骤做成独立 section：

- `__crt0_start`
- `__crt0_init_bss`
- `__crt0_movedata`
- `__crt0_move_highdata`
- `__crt0_run_preinit_array`
- `__crt0_call_main`
- `__crt0_call_exit`
- `__crt0_run_array`

底层机制由 `memmodel.h` 的两个宏提供：

- `START_CRT_FUNC number name`
- `END_CRT_FUNC name`

它们会把函数放到 `.crt_<序号><名字>` section 中。参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/memmodel.h:42-58`。

链接脚本随后用：

```ld
KEEP (*(SORT(.crt_*)))
```

把这些 section 拉进 `.text`，并按名字排序。参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/msp430-sim.ld:96-100`，以及 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/msp430xl-sim.ld:278-284`。

### 1.1 为什么这很适合 MCU

这种设计有 4 个直接好处：

- **顺序稳定**：靠 section 名中的数字控制顺序，而不是手工写调用链。
- **可裁剪**：某段启动逻辑若没有被引用或没有触发条件，可被链接器 GC 掉。
- **易插入用户逻辑**：用户可以塞一个 `.crt_0550_board_init` 之类的片段插入现有序列。
- **large/small 模型共用一套流程**：不同模型只改宏和链接脚本，不改启动框架。

### 1.2 这不是“完全动态发现”，而是“编译期条件 + 链接期回收”

`crt0.S` 中每个阶段都包在 `#if Lxxx` 里，例如：

- `#if Lbss`
- `#if Lmovedata`
- `#if Lrun_init_array`
- `#if Lcallexit`

参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/crt0.S:69-84,108-127,183-194,260-269`。

这些 `Lxxx` 宏由构建规则 `crt_%.o : crt0.S -> -DL$*` 驱动，每个 `crt_xxx.o` 都只编译出一个阶段。参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/Makefile.in:83-107`。

换句话说，启动系统不是一个源文件编译出一个目标，而是：

```text
crt0.S
  -> crt_bss.o
  -> crt_movedata.o
  -> crt_main.o
  -> crt_callexit.o
  -> crt_run_init_array.o
  ...
```

再由 `libcrt.a` 聚合。参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/Makefile.in:85-97,122-128`。

---

## 2. reset 到 `main` 的精确时序

可以把 MSP430 newlib 的缺省启动时序理解成下图：

```text
Reset
  -> VECT31
  -> .resetvec
  -> __crt0_start
  -> 设置 SP = __stack
  -> 清零 .bss
  -> (large model) 清零 .upper.bss
  -> 初始化 .data
  -> (large model) 初始化 .upper.data
  -> 执行 .preinit_array
  -> 执行 .init_array
  -> 执行 smi 位置初始化表
  -> main(argc=0)
  -> _exit/exit
  -> (如启用) fini_array
```

### 2.1 Reset 向量是怎么接到 `__crt0_start` 的

在 `crt0.S` 的 `L0` 分支中：

- 先创建 `.resetvec`
- 定义 `__msp430_resetvec_hook`
- 写入 `__crt0_start` 的地址

参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/crt0.S:22-25`。

而 `intr_vectors.ld` 明确把 `VECT31` 放在 `0xFFFE`，并让：

- `__interrupt_vector_reset`
- `(.resetvec)`

都落到该槽位。参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/intr_vectors.ld:33-35,69-70`。

这条链路意味着：

```text
硬件复位 -> 取 0xFFFE -> 命中 .resetvec -> __crt0_start
```

### 2.2 `start` 阶段只做最少的事

`start` 阶段本身只做了三类关键操作：

1. 建立 reset hook 引用
2. 强制拉入 `main` 与 `__crt0_call_main`
3. 设置栈顶：`mov_ #__stack, R1`

参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/crt0.S:53-60`。

注意它**没有**在这里直接写“call init_bss / call movedata / call main”。顺序其实是靠 `.crt_*` 的拼接来保证的。

---

## 3. 每个启动阶段依赖哪些链接符号

这一节最重要，因为 MSP430 端口本质上就是“启动汇编 + 链接脚本符号协议”。

### 3.1 `init_bss`

代码：

- `R12 = __bssstart`
- `R13 = 0`
- `R14 = __bsssize`
- `call_ #memset`

参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/crt0.S:73-83`。

依赖符号来自链接脚本：

- `__bssstart`
- `__bsssize`

参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/msp430-sim.ld:153-167`，以及 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/msp430xl-sim.ld:258-275`。

### 3.2 `movedata`

代码：

- `R12 = __datastart`
- `R13 = __romdatastart`
- `R14 = __romdatacopysize`
- `call_ #memmove`

参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/crt0.S:113-125`。

对应符号由脚本提供：

- `__datastart`
- `__romdatastart = LOADADDR(.data)`
- `__romdatacopysize = SIZEOF(.data)`

small 模拟脚本参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/msp430-sim.ld:116-151`；ROM model 脚本参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/msp430xl-sim-rom.ld:133-175`。

### 3.3 `move_highdata`

这是整套启动代码最复杂的一段。

代码逻辑：

1. 先检查 `__rom_highdatacopysize`
2. 若为 0，直接跳过
3. 检查 `__upper_data_init` 状态字
4. 若状态字非 0，则从“影子/ROM”复制到 `__high_datastart`
5. 若状态字为 0，则先反向复制，并把状态字设为 1

参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/crt0.S:139-166`。

它依赖 5 组关键符号：

- `__upper_data_init`
- `__rom_highdatacopysize`
- `__high_datastart`
- `__rom_highdatastart`
- `__high_bssstart/__high_bsssize`（同类 high memory 支撑）

在 `L0` 阶段，`crt0.S` 先把这些符号做成弱定义 0 值，避免没有 high memory 区域时报未定义。参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/crt0.S:27-51`。

这是一种很精巧的容错：

- 链接脚本若真的定义了 `.upper.*`，强符号覆盖弱符号
- 若没定义 `.upper.*`，启动代码仍然能链接，只是对应逻辑被短路

---

## 4. large FRAM 脚本和 large ROM 脚本的本质区别

这两个脚本都面向 MSP430X large model，但策略不同。

### 4.1 `msp430xl-sim.ld`：FRAM 模型

特点：

- `FRAM (rwx)` 同时承载代码和常规数据
- `HIFRAM (rwx)` 承载 `upper.*` 段
- `.data` 是 `> FRAM`，不是 `AT> 别处`
- `.upper.data` 也是 `> HIFRAM`，没有单独影子副本定义

参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/msp430xl-sim.ld:26-31,131-171,248-254`。

因此它更像“**运行地址本身就是可持久介质**”的模型。

### 4.2 `msp430xl-sim-rom.ld`：ROM 模型

特点：

- `.data > RAM AT> ROM`
- `.upper.data > HIFRAM AT> HIROM`
- 脚本显式导出：
  - `__upper_data_init = LOADADDR(.upper.data)`
  - `__rom_highdatacopysize = SIZEOF(.upper.data) - 2`
  - `__rom_highdatastart = LOADADDR(.upper.data) + 2`

参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/msp430xl-sim-rom.ld:252-265`。

这里的 `-2` 与 `+2` 来自状态字的布局：`.upper.data` 的加载镜像头部先放一个 2 字节状态字，真实数据紧随其后。参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/msp430xl-sim-rom.ld:252-260`。

### 4.3 为什么 `.either.*` 在 large model 中经常被故意省略

在 `msp430xl-sim.ld` 和 `msp430xl-sim-rom.ld` 中，多次强调：

- 不要在脚本里主动定义 `.either.rodata`
- 不要主动定义 `.either.data`
- 不要主动定义 `.either.bss`
- 不要主动定义 `.either.text`

因为链接器的自动放置算法依赖“`.upper.*` 存在，而 `.either.*` 未被脚本抢先消费”。参见：
- `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/msp430xl-sim.ld:48-61,145-148,266-269,294-297`
- `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/msp430xl-sim-rom.ld:50-63,147-150,277-280`

这说明 `-mdata-region=either` 不是简单 section rename，而是工具链和链接脚本协同完成的自动分配机制。

---

## 5. heap / stack 布局在不同脚本下的差异

### 5.1 普通 `msp430-sim.ld`

在小模型脚本里：

- `_end = .; PROVIDE(end = .);`
- `__stack` 放在 `RAM` 顶部

参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/msp430-sim.ld:193-210`。

所以 `_sbrk()` 从 `end` 向上长堆，而栈从 RAM 顶部向下长。参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/sbrk.c:23-37`。

### 5.2 `msp430xl-sim-rom.ld`

在 large ROM 模型里，脚本专门创建一个 `.heap_start` 放在 `HIFRAM`，哪怕 `.upper.bss` 为空也要保证 `end` 落在 HIFRAM。参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/msp430xl-sim-rom.ld:325-334`。

同时它明确说明：

- `.stack` 也放在 HIFRAM 顶部
- 这样 heap 和 stack 都在高地址区，更不容易与低地址 RAM 已分配数据冲突

参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/msp430xl-sim-rom.ld:347-356`。

这意味着 large ROM model 下，`_sbrk` 的内存语义更接近：

```text
低 RAM   : .data/.bss
高 HIFRAM: heap + stack + upper.bss/upper.data
```

---

## 6. `run_array` 为什么能同时支持 init 与 fini

`run_preinit_array` / `run_init_array` / `run_fini_array` 最后都落到 `__crt0_run_array`。

区别只在于传参：

- preinit/init：
  - `R4 = start`
  - `R5 = end`
  - `R6 = +PTRsz`
- fini：
  - `R4 = end - PTRsz`
  - `R5 = start - PTRsz`
  - `R6 = -PTRsz`

参见：
- `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/crt0.S:170-194`
- `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/crt0.S:274-285`

`run_array` 本体只做三件事：

1. 比较 `R4` 与 `R5`
2. 取 `*R4` 作为函数指针到 `R7`
3. `R4 += R6` 后调用它，再循环

参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/crt0.S:288-303`。

因此 `run_array` 被设计成了一个**方向可配置**的通用遍历器：

- `+PTRsz` 就是正向 init
- `-PTRsz` 就是逆向 fini

这是个非常省代码的实现。

---

## 7. `run_smi_location_init_array` 是什么

这部分在很多端口文档里容易被忽略，但它其实说明 MSP430 启动系统考虑了“按表初始化离散对象”的需求。

`run_smi_location_init_array` 按固定三元组遍历一张表：

- `vma`
- `lma`
- `size`

每条记录大小固定 3 个地址项；当 `size == 0` 时结束。参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/crt0.S:213-246`。

其中：

- 若 `lma == 0xFFFF`，则视为 BSS 风格对象，调用 `memset`
- 否则调用 `memmove`

这相当于允许链接器或上层工具链生成一张“散点初始化清单”，由启动代码统一执行。

我的判断：这是为了支持某些特殊存储位置、特殊 section 或小对象定点初始化，而不必把它们硬编码进 `.data/.bss/.upper.data/.upper.bss` 的主流程。

---

## 8. I/O 路径：为什么 `write` 既不走标准 OS syscall，也不直接碰 UART

### 8.1 `write` 的真实角色

在 MSP430 newlib 里，`write` 是最关键的 I/O 汇聚点：

- `tiny puts` 调 `write(1, ...)`
- `tiny printf` 最终调 `write(1, ...)`
- `_sbrk` 出错也调 `write(1, "Heap and stack collision\n", ...)`

参见：
- `msp430-gcc-9.3.1.11-source-full/newlib/newlib/libc/machine/msp430/tiny-puts.c:8-18`
- `msp430-gcc-9.3.1.11-source-full/newlib/newlib/libc/machine/msp430/tiny-printf.c:86-94`
- `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/sbrk.c:29-33`

所以只要 `write` 通了，最基本的调试输出就通了。

### 8.2 `write` 为什么不在 `syscalls.S`/`ciosyscalls.S` 里

你会发现两边都刻意注释掉了 `SC(write)`：

- `syscalls.S:48`
- `ciosyscalls.S:75`

也就是说，`write` 没有被当成普通 syscall 桩，而是由 `write.c` 单独实现。这是一个有意的架构决策：**输出路径和普通文件 syscall 路径分离**。

好处是：

- 即使没有 OS、没有文件系统，`printf` 仍然能工作
- 宿主调试器只要认识 CIO 协议，就能接收输出
- 避免把最常用的调试输出依赖在不稳定的 syscall 仿真上

### 8.3 CIO 调用序列

`write.c` 的单次分块发送逻辑是：

```text
write()
  -> write_chunk()
     -> 填 __CIOBUF__.length
     -> 填 __CIOBUF__.parms[0] = CIO_WRITE
     -> 把 fd/len 等参数编码进 parms
     -> memcpy 到 __CIOBUF__.buf
     -> _libgloss_cio_hook()
     -> 从 parms[0..1] 读返回值
```

源码参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/write.c:18-32`。

如果长度大于 `CIO_BUF_SIZE`，则 `write()` 做循环分块，默认每块 64 字节。参见：
- `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/cio.h:17-25`
- `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/write.c:49-58`

### 8.4 `_libgloss_cio_hook` 本身几乎不做事

`cio.S` 里它只是：

```asm
C$$IO$$:
    nop
    ret_
```

参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/cio.S:28-33`。

这说明：

- CIO 的真正处理者不在目标程序里
- 真实语义来自调试器、仿真器或宿主侧工具
- 目标程序只负责把协议帧放进 `__CIOBUF__` 并触发钩子

所以 CIO 更像一种 **debug monitor ABI**，而不是本地驱动 API。

### 8.5 `unlink` 复用同一协议

`unlink.c` 和 `write.c` 完全同一个模式：

- 填长度
- 填命令号 `CIO_UNLINK`
- 拷文件名进缓冲
- 调 `_libgloss_cio_hook`
- 从 `parms[0..1]` 取返回值

参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/unlink.c:5-22`。

这说明 CIO 协议被设计成：

- 不止面向 stdout/stderr
- 而是面向一组最小宿主服务

---

## 9. 模拟器 syscall 路径与 CIO 路径如何共存

可以把两条路径理解成：

### 9.1 路径 A：模拟器 syscall

适用于：

- `open/close/read/fstat/lseek/kill/_exit/exit`

机制：

- 函数地址直接绑定到 `0x180 + SYS_xxx`
- 模拟器拦截 CALL 到这些低地址的动作

参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/syscalls.S:14-18,35-51`。

### 9.2 路径 B：CIO 半主机协议

适用于：

- `write`
- `unlink`
- 可能扩展到更多 CIO 命令

机制：

- 编码 `__CIOBUF__`
- 调 `_libgloss_cio_hook`
- 宿主/调试器解释命令

参见：
- `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/cio.h:31-40`
- `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/write.c:18-32`
- `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/unlink.c:13-22`

### 9.3 路径 C：真实硬件无 OS 桩

适用于：

- 没有模拟器 syscall，也没有真正 OS 的情形

机制：

- 大多数接口直接 `ENOSYS`
- 但保留 `write`/`unlink` 的 CIO 出口

参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/ciosyscalls.S:61-78`。

这 3 条路径组合起来，就形成了 MSP430 newlib 的 I/O 策略：

```text
普通文件/进程语义     -> syscall 或 ENOSYS
最重要的调试输出语义   -> CIO
```

---

## 10. `_sbrk` 与 `malloc` 的真实边界

`_sbrk()` 只是把 `_sbrk_heap` 线性上推。参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/sbrk.c:23-37`。

它做的唯一保护是：

```c
if (_sbrk_heap + adj > sp) {
    write(... "Heap and stack collision\n" ...);
    abort();
}
```

参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/sbrk.c:29-34`。

这意味着它依赖 3 个前提：

1. 当前函数局部变量地址可以近似当前栈顶
2. 栈单调向低地址生长
3. heap 和 stack 位于同一线性地址空间中并会相向增长

对普通 bare-metal 单栈应用，这是够用的；但对以下场景就不太够：

- 中断栈与线程栈分离
- RTOS 多任务堆栈
- 多内存池/多 region 分配
- 需要高水位统计或碎片感知

所以在实际项目里，MSP430 newlib 往往需要用户重写 `_sbrk`。

---

## 11. `exit` 为什么是死循环

`exit.c` 里的实现看起来很“土”，但其实非常符合 MCU 现实：

```c
void __attribute__((naked, weak)) exit (int status) {
    while (1);
}
```

参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/exit.c:1-10`。

这样做的原因通常有 3 个：

- bare-metal 没有父进程可返回
- 让调试器能稳定停在程序结束态
- 避免退出后跑飞到未知地址

在模拟器环境里，如果启用了低地址 `_exit` syscall 映射，则也可能由模拟器接管进程终止语义。参见 `msp430-gcc-9.3.1.11-source-full/newlib/libgloss/msp430/syscalls.S:43`。

---

## 12. 推荐把整套端口理解成三层

最后，把 MSP430 的 newlib 端口抽象成三层最清晰：

### 第 1 层：ABI/CPU 适配层

位置：`newlib/newlib/libc/machine/msp430`

职责：

- `setjmp/longjmp`
- tiny stdio
- small/large 地址模型下的最少 libc 适配

### 第 2 层：运行时装配层

位置：`newlib/libgloss/msp430/crt0.S` + `memmodel.h` + `*.ld`

职责：

- reset 接管
- 栈初始化
- `.bss/.data/.upper.*` 初始化
- 构造/析构数组调度
- heap/stack 起点定义

### 第 3 层：宿主交互层

位置：`syscalls.S` / `ciosyscalls.S` / `cio.S` / `write.c` / `unlink.c`

职责：

- 模拟器 syscall
- 真实硬件无 OS stub
- 调试器半主机 I/O

这三层叠起来，就是完整的 MSP430 bare-metal C 运行时。

---

## 13. 这一轮深入分析的最关键结论

如果只记 4 句话，可以记这 4 句：

1. **MSP430 的 `crt0` 不是单函数，而是 `.crt_*` 启动片段流水线。**
2. **large model 的关键不只是 20-bit 指令，还有 `.upper.*` 段与链接脚本符号协议。**
3. **`write` 被从普通 syscall 体系中刻意拆出来，单独走 CIO 半主机协议。**
4. **这套端口的真正灵魂是“尺寸优先 + 裸机优先 + 调试可观测优先”。**

