# MSP430 `newlib` ASCII 架构图

这份文档把前两份分析文档压缩成可快速阅读的 ASCII 图，适合：

- 快速回忆整体架构
- 给别人讲解 MSP430 newlib 端口
- 做后续源码导读时的“导航页”

配套文档：

- 总体分析：`docs/msp430-newlib-arch.md`
- 深入分析：`docs/msp430-newlib-startup-io-deepdive.md`

---

## 1. 总体分层图

```text
+-------------------------------------------------------------+
|                  MSP430 GCC / newlib 端口                   |
+-------------------------------------------------------------+
|  第 1 层：libc CPU/ABI 适配                                 |
|  目录: newlib/newlib/libc/machine/msp430                    |
|                                                             |
|   - setjmp.S            上下文保存/恢复                     |
|   - tiny-printf.c       极简 printf                         |
|   - tiny-puts.c         极简 puts                           |
|   - nano-vfprintf*.h    复用 nano printf 内核               |
+-------------------------------------------------------------+
|  第 2 层：运行时装配 / CRT / 链接脚本                       |
|  目录: newlib/libgloss/msp430                               |
|                                                             |
|   - crt0.S              启动片段流水线                      |
|   - memmodel.h          small/large 模型抽象               |
|   - msp430-*.ld         段布局 / 启动符号协议              |
|   - intr_vectors.ld     中断向量与 reset 向量映射          |
+-------------------------------------------------------------+
|  第 3 层：宿主交互 / syscall / 半主机 I/O                   |
|  目录: newlib/libgloss/msp430                               |
|                                                             |
|   - syscalls.S          模拟器低地址 syscall               |
|   - ciosyscalls.S       无 OS 默认桩                       |
|   - cio.S / cio.h       CIO 协议与缓冲区                   |
|   - write.c             调试输出主通路                     |
|   - unlink.c            宿主文件操作最小支持               |
|   - sbrk.c              堆增长后端                         |
|   - exit.c              裸机退出语义                       |
+-------------------------------------------------------------+
```

---

## 2. 构建产物关系图

```text
configure.host
   |
   |-- 识别 host_cpu = msp430*
   |-- 追加 -Os -mOs -DPREFER_SIZE_OVER_SPEED -DSMALL_MEMORY
   |-- 追加 -ffunction-sections -fdata-sections
   |-- 追加 -mhwmult=none
   |-- 选择 machine_dir = msp430
   v

+-----------------------------------+
| newlib/libc/machine/msp430        |
| -> lib.a                          |
|    - setjmp.o                     |
|    - tiny-puts.o                  |
|    - tiny-printf.o                |
+-----------------------------------+

+-----------------------------------+
| libgloss/msp430                   |
| -> crt0.o / gcrt0.o               |
| -> libcrt.a                       |
| -> libsim.a                       |
| -> libnosys.a                     |
| -> msp430-sim.ld                  |
| -> msp430xl-sim.ld                |
| -> msp430xl-sim-rom.ld            |
+-----------------------------------+
```

---

## 3. `crt0.S` 片段化启动图

### 3.1 源码到对象文件

```text
crt0.S
  |
  |-- -DL0                    -> crt0.o           (start / resetvec)
  |-- -DLbss                  -> crt_bss.o        (init_bss)
  |-- -DLhigh_bss             -> crt_high_bss.o   (init_highbss)
  |-- -DLmovedata             -> crt_movedata.o   (copy .data)
  |-- -DLmove_highdata        -> crt_move_highdata.o
  |-- -DLmain                 -> crt_main.o       (call main)
  |-- -DLcallexit             -> crt_callexit.o   (call _exit)
  |-- -DLrun_preinit_array    -> crt_run_preinit_array.o
  |-- -DLrun_init_array       -> crt_run_init_array.o
  |-- -DLrun_fini_array       -> crt_run_fini_array.o
  |-- -DLrun_array            -> crt_run_array.o
  |-- -DLrun_smi_location_init_array -> crt_run_smi_location_init_array.o
  v
libcrt.a
```

### 3.2 链接时怎么拼起来

```text
libcrt.a 中的对象
   |
   v
各对象内部 section：

  .crt_0000start
  .crt_0100init_bss
  .crt_0200init_highbss
  .crt_0300movedata
  .crt_0400move_highdata
  .crt_0500run_preinit_array
  .crt_0600run_init_array
  .crt_0710run_smi_location_init_array
  .crt_0800call_main
  .crt_0900call_exit
  .crt_1000run_fini_array
  .crt_1100run_array

   |
   v
链接脚本： KEEP(*(SORT(.crt_*)))
   |
   v
最终 .text 中按编号顺序排列
```

---

## 4. Reset 到 `main` 的时序图

```text
[硬件 Reset]
     |
     v
[VECT31 @ 0xFFFE]
     |
     v
[.resetvec]
     |
     v
[__crt0_start]
     |
     |-- R1 = __stack
     |
     |-- if present: __crt0_init_bss
     |      `-> memset(__bssstart, 0, __bsssize)
     |
     |-- if large+present: __crt0_init_highbss
     |      `-> memset(__high_bssstart, 0, __high_bsssize)
     |
     |-- if present: __crt0_movedata
     |      `-> memmove(__datastart, __romdatastart, __romdatacopysize)
     |
     |-- if large+present: __crt0_move_highdata
     |      |-> 检查 __rom_highdatacopysize
     |      |-> 检查 __upper_data_init 状态字
     |      `-> 决定 upper.data 的复制方向
     |
     |-- if present: __crt0_run_preinit_array
     |
     |-- if present: __crt0_run_init_array
     |
     |-- if present: __crt0_run_smi_location_init_array
     |
     |-- __crt0_call_main
     |      `-> main(argc=0)
     |
     |-- if linked: __crt0_call_exit
     |      `-> _exit / exit
     |
     `-- if linked: __crt0_run_fini_array
```

---

## 5. `run_array` 通用遍历器图

```text
                +-----------------------+
                |   __crt0_run_array    |
                +-----------------------+
                           |
                           | 参数:
                           |   R4 = current
                           |   R5 = stop
                           |   R6 = stride
                           v
                 +----------------------+
                 | current == stop ?    |
                 +----------------------+
                    | yes           | no
                    v               v
                  [ret]      R7 = *R4 (函数指针)
                                 |
                                 v
                            R4 += R6
                                 |
                                 v
                              call R7
                                 |
                                 v
                              回到自身
```

用法：

```text
init/preinit : R6 = +PTRsz   -> 正向遍历
fini         : R6 = -PTRsz   -> 逆向遍历
```

---

## 6. small / large memory model 指令抽象图

```text
+--------------------+----------------------+----------------------+
| 抽象宏             | small model          | large model          |
+--------------------+----------------------+----------------------+
| call_              | CALL                 | CALLA                |
| ret_               | RET                  | RETA                 |
| mov_               | MOV                  | MOVA                 |
| movx_              | MOV                  | MOVX                 |
| br_                | BR                   | BRA                  |
| cmp_               | CMP                  | CMPA                 |
| add_               | ADD                  | ADDA                 |
| PTRsz              | 2                    | 4                    |
+--------------------+----------------------+----------------------+
```

这意味着同一份 `crt0.S` / `syscalls.S` / `cio.S` 可以跨 small/large 模型复用。

---

## 7. 链接脚本符号协同图

### 7.1 普通 `.data/.bss`

```text
链接脚本导出:                        crt0.S 使用:

__datastart -----------------------> movedata: 目标地址
__romdatastart --------------------> movedata: 源地址
__romdatacopysize -----------------> movedata: 拷贝长度

__bssstart ------------------------> init_bss: 起始地址
__bsssize -------------------------> init_bss: 清零长度

__stack ---------------------------> start: 初始化 R1
end / _end ------------------------> _sbrk: heap 起点
```

### 7.2 large model 的 `.upper.*`

```text
链接脚本导出:                        crt0.S 使用:

__high_datastart ------------------> move_highdata: upper.data VMA
__rom_highdatastart ---------------> move_highdata: ROM/影子源地址
__rom_highdatacopysize ------------> move_highdata: 拷贝长度
__upper_data_init -----------------> move_highdata: 状态字

__high_bssstart -------------------> init_highbss: 起始地址
__high_bsssize --------------------> init_highbss: 清零长度
```

### 7.3 弱符号兜底图

```text
crt0.S (L0)
  |
  |-- weak __upper_data_init      = 0
  |-- weak __rom_highdatacopysize = 0
  |-- weak __high_datastart       = 0
  |-- weak __rom_highdatastart    = 0
  |-- weak __high_bssstart        = 0
  `-- weak __high_bsssize         = 0

        |
        +-- 如果链接脚本给出强定义 -> 使用脚本值
        `-- 如果脚本没定义         -> 启动流程自动短路/跳过
```

---

## 8. 3 种典型内存布局图

### 8.1 `msp430-sim.ld`：统一 RAM 示例

```text
低地址
  |
  |  .rodata
  |  .rodata2
  |  .text
  |  .data
  |  .bss
  |  .noinit
  |  .persistent
  |  end  ---> heap start
  |   ^
  |   | heap 向上增长
  |
  |                stack 向下增长
  |                     v
  |                 __stack
  |
高地址 (RAM 顶部)
```

### 8.2 `msp430xl-sim.ld`：FRAM + HIFRAM

```text
FRAM:
  .rodata
  .rodata2
  .data
  .bss
  .text
  .noinit
  .persistent

HIFRAM:
  .upper.rodata
  .upper.data
  .upper.text
  ...
```

特点：代码和常规数据都可放 FRAM，高地址对象放 HIFRAM。

### 8.3 `msp430xl-sim-rom.ld`：ROM + RAM + HIFRAM + HIROM

```text
ROM:
  .rodata
  .rodata2
  .text
  .data 的加载镜像 (AT> ROM)

RAM:
  .data 的运行地址
  .bss
  .noinit

HIROM:
  .upper.rodata
  .upper.data 的加载镜像 (AT> HIROM)

HIFRAM:
  .upper.data 的运行地址
  .persistent
  .upper.bss
  .heap_start -> end
  ... heap ...
                     ... stack ...
                           __stack
```

---

## 9. `write` / CIO / syscall 路径图

### 9.1 调试输出主路径

```text
printf / puts
    |
    v
 tiny-printf / tiny-puts
    |
    v
 write(fd=1, buf, len)
    |
    v
 write_chunk()
    |
    |-- __CIOBUF__.length = len
    |-- __CIOBUF__.parms[0] = CIO_WRITE
    |-- __CIOBUF__.parms[...] = fd/len/...
    |-- memcpy(__CIOBUF__.buf, buf, len)
    v
 _libgloss_cio_hook()
    |
    v
 调试器 / 仿真器 / 宿主识别 C$$IO$$
    |
    v
 在宿主侧显示输出 / 返回状态
```

### 9.2 模拟器 syscall 路径

```text
open / close / read / lseek / fstat / kill / _exit
    |
    v
符号地址 = 0x180 + SYS_xxx
    |
    v
CALL 到低地址区
    |
    v
模拟器捕获该 CALL
    |
    v
模拟器完成 syscall 语义
```

### 9.3 真实硬件缺省路径

```text
open / close / read / lseek / fstat / kill
    |
    v
ciosyscalls.S
    |
    |-- __errno() = ENOSYS
    `-- return -1

write / unlink
    |
    v
仍然走 CIO 通道
```

---

## 10. `write` 为什么被单独拿出来

```text
普通想法：
  printf -> write -> syscall(write)

MSP430 实际做法：
  printf -> write -> CIO
                
  open/read/... -> syscall or ENOSYS
```

这样拆分后：

- 最常用的调试输出不依赖 OS
- 即使大多数 syscall 都不可用，`printf` 仍然可用
- 输出链路可以稳定对接调试器/模拟器

---

## 11. `sbrk` / heap / stack 冲突图

```text
          低地址                                      高地址
            |                                           |
            |  end = 初始 heap 起点                     |
            |   |                                       |
            |   v                                       |
            | [ heap 已分配区域 ]                       |
            |                                           |
            |                  (空闲区)                 |
            |                                           |
            |                       [ 栈帧 / 局部变量 ] |
            |                               ^           |
            |                               |           |
            |                         当前近似 SP       |
            |                                           |

判断条件：
  if (_sbrk_heap + adj > sp) -> Heap and stack collision
```

---

## 12. 代码阅读路线图

```text
第一步：总览
  docs/msp430-newlib-arch.md
       |
       v
第二步：看图建立脑图
  docs/msp430-newlib-ascii-maps.md
       |
       v
第三步：细抠启动/I-O
  docs/msp430-newlib-startup-io-deepdive.md
       |
       v
第四步：回源码
  1) configure.host
  2) memmodel.h
  3) crt0.S
  4) msp430-sim.ld / msp430xl-*.ld
  5) syscalls.S / ciosyscalls.S / write.c / cio.S
  6) setjmp.S / tiny-printf.c
```

---

## 13. 最短记忆版

```text
MSP430 newlib =

  极薄 libc 机器层
+ 片段化 CRT 启动系统
+ small/large 双模型抽象
+ 链接脚本驱动的数据初始化协议
+ syscall 与 CIO 并存的宿主交互
+ 尺寸优先的 bare-metal 运行时
```

