# USB 主机控制器详解：UHCI / OHCI / EHCI / xHCI

> 基于 Linux 6.19.6 内核 `drivers/usb/host/` 源码
> 面向内核驱动开发的工程参考文档

---

## 总览对比

| 特性 | UHCI | OHCI | EHCI | xHCI |
|------|------|------|------|------|
| 规范版本 | USB 1.1 | USB 1.1 | USB 2.0 | USB 3.x（向下兼容 1.x/2.0） |
| 最高速率 | 12 Mbps（全速）/ 1.5 Mbps（低速） | 12 Mbps（全速）/ 1.5 Mbps（低速） | 480 Mbps（高速） | 5/10/20 Gbps（超速） |
| 规范起源 | Intel（1995） | Compaq / Microsoft（1999） | Intel（2000） | Intel（2010） |
| 总线接口 | PCI / I/O 端口 | PCI / MMIO | PCI / MMIO | PCIe / MMIO |
| 调度模式 | 纯软件（CPU 维护帧列表） | 硬件（ED 链表） | 混合（硬件异步 + 软件周期） | 全硬件（TRB 环） |
| 传输描述符 | QH + TD | ED + TD | QH + QTD / ITD / SITD | TRB（Transfer Request Block） |
| 中断通知 | 完成标志轮询（IOC bit） | Done Queue Head（WDH 中断） | IAA + STS_INT | 事件环（Event Ring）+ MSI-X |
| 多速度支持 | 否（需伴随 OHCI/UHCI） | 否（需伴随 OHCI/UHCI） | 否（需伴随 OHCI/UHCI） | 是（单控制器覆盖全速度） |
| Linux 驱动 | `uhci-hcd.c` | `ohci-hcd.c` | `ehci-hcd.c` | `xhci.c` |

---

## 1. UHCI（Universal Host Controller Interface）

### 1.1 背景

UHCI 由 Intel 于 1995 年随 PIIX3 南桥芯片提出，是最早的 USB 1.1 主机控制器规范之一。其设计哲学是**把尽可能多的逻辑放在软件（驱动）层**，硬件只负责最底层的包收发，因此芯片成本低，但 CPU 负担较重。

### 1.2 寄存器模型

UHCI 使用 **I/O 端口**（少数实现也支持 MMIO）访问控制寄存器，基地址由 PCI BAR4 提供：

| 偏移 | 寄存器 | 关键位 |
|------|--------|--------|
| 0x00 | `USBCMD` | RS（运行/停止）、HCRESET、GRESET、EGSM（全局挂起） |
| 0x02 | `USBSTS` | USBINT（IOC 完成）、ERROR、RD（恢复检测）、HCH（已停止） |
| 0x04 | `USBINTR` | TIMEOUT、RESUME、IOC、SP（短包）中断使能 |
| 0x06 | `USBFRNUM` | 当前帧号（只读，低 11 位有效） |
| 0x08 | `USBFLBASEADD` | 帧列表基地址（物理地址，4KB 对齐） |
| 0x10/0x12 | `USBPORTSCx` | 端口状态与控制（CCS、PE、PR、SUSP、LSDA 等） |

端口状态寄存器关键位（`uhci-hcd.h`）：

```c
#define USBPORTSC_CCS   0x0001  /* 设备连接状态 */
#define USBPORTSC_PE    0x0004  /* 端口使能 */
#define USBPORTSC_LSDA  0x0100  /* 低速设备标志 */
#define USBPORTSC_PR    0x0200  /* 端口复位 */
#define USBPORTSC_SUSP  0x1000  /* 端口挂起 */
```

### 1.3 帧列表（Frame List）

UHCI 硬件每 1 ms 推进一帧，驱动在内存中维护一张 **1024 项的帧指针数组**，每项是 32 位物理地址，指向当前帧应处理的第一个 QH 或 TD：

```
frame[0] → QH(int128) → QH(int64) → ... → QH(async) → QH(bulk) → TERM
frame[1] → QH(int64)  → ...
...
frame[1023] → ...
```

硬件从 `USBFLBASEADD + (USBFRNUM & 0x3FF) * 4` 处读取帧指针，沿链表依次处理 QH/TD。

### 1.4 传输描述符 QH 与 TD

#### `struct uhci_qh`（`uhci-hcd.h:151`）

```c
struct uhci_qh {
    /* 硬件字段（共享内存，硬件直接读写） */
    __hc32 link;       /* 链表中下一个 QH 的物理地址 */
    __hc32 element;    /* 当前 TD 头指针（硬件推进此指针） */

    /* 软件字段 */
    struct list_head queue;         /* 挂在此 QH 上的 URB 列表 */
    struct uhci_td   *dummy_td;     /* 哑 TD，防止硬件越界 */
    int    state;                   /* IDLE / UNLINKING / ACTIVE */
    int    type;                    /* CONTROL / BULK / INTR / ISO */
    int    skel;                    /* 骨架队列编号 */
    unsigned int period;            /* 中断/ISO 传输间隔（帧数） */
} __attribute__((aligned(16)));
```

#### `struct uhci_td`（`uhci-hcd.h:258`）

```c
struct uhci_td {
    __hc32 link;    /* 下一个 TD 或 QH 的物理地址 */
    __hc32 status;  /* 控制/状态字：ACTIVE、STALLED、BABBLE、NAK、错误计数、实际长度 */
    __hc32 token;   /* 令牌：PID、设备地址、端点号、数据翻转、期望长度 */
    __hc32 buffer;  /* 数据缓冲区物理地址 */
    /* 软件字段 */
    dma_addr_t dma_handle;
    struct list_head list;
    int frame;      /* ISO 时使用的帧号 */
} __attribute__((aligned(16)));
```

`status` 关键位：

| 位 | 含义 |
|----|------|
| [23] `TD_CTRL_ACTIVE` | TD 尚未处理 |
| [22] `TD_CTRL_STALLED` | 端点 STALL |
| [20] `TD_CTRL_BABBLE` | 数据过长 |
| [19] `TD_CTRL_NAK` | 收到 NAK |
| [18] `TD_CTRL_CRCTIMEO` | CRC/超时错误 |
| [28:27] `C_ERR` | 错误重试计数（最大 3 次） |
| [10:0] `ACTLEN` | 实际传输字节数（n-1 编码） |

### 1.5 骨架 QH 调度树

UHCI 用 11 个预置「骨架 QH」（`skelqh[UHCI_NUM_SKELQH]`）构建按间隔分层的调度树：

```
skelqh[2]  → int128 QH（每 128 帧轮询一次）
skelqh[3]  → int64 QH
...
skelqh[9]  → int1 QH（每帧一次，同时接入 async 链）
skelqh[9]  → async QH → LS_CONTROL → FS_CONTROL → BULK
skelqh[10] → term QH（自指向，防止无任务时停止带宽回收）
```

帧列表中不同帧指向该树的不同深度节点，实现周期性轮询。

### 1.6 中断处理

```
硬件中断 → uhci_irq()
  → 读 USBSTS 寄存器
  → USBSTS_USBINT：扫描完成 TD → uhci_scan_schedule()
      → 遍历 async 链和 periodic 链
      → 检查 TD status 是否已清除 ACTIVE 位
      → usb_hcd_giveback_urb()
  → USBSTS_ERROR：传输错误，报告给 URB
  → USBSTS_RD：恢复检测
  → USBSTS_HCH：控制器停止（硬件错误）
```

### 1.7 局限性

- CPU 必须每帧主动遍历帧列表和 QH 链，CPU 开销大
- 不支持高速（480 Mbps），需配合 EHCI 使用
- 每个控制器最多 2 个根 Hub 端口（Intel PIIX 限制）

---

## 2. OHCI（Open Host Controller Interface）

### 2.1 背景

OHCI 由 Compaq、Microsoft 等厂商联合制定，作为 UHCI 的竞争方案，于 1999 年发布 1.0a 规范。其设计哲学与 UHCI 相反：**把更多调度逻辑放在硬件**，驱动只需维护端点描述符（ED）链表，硬件自动遍历并完成传输。OHCI 广泛用于嵌入式 ARM/MIPS 平台（如树莓派早期版本），因为它对软件更友好。

### 2.2 寄存器模型（MMIO）

OHCI 控制器通过 MMIO 访问，`struct ohci_regs`（`ohci-hcd.h`）覆盖所有寄存器：

| 寄存器 | 用途 |
|--------|------|
| `HcRevision` | 规范版本（应为 0x10 = 1.0） |
| `HcControl` | 功能状态控制（CBSR、PLE、IE、CLE、BLE、HCFS） |
| `HcCommandStatus` | 命令触发（HCR 复位、CLF/BLF 填充列表） |
| `HcInterruptStatus` | 中断状态（WDH、SF、RHSC 等） |
| `HcInterruptEnable/Disable` | 中断使能控制 |
| `HcHCCA` | HCCA 物理基地址 |
| `HcControlHeadED` | 控制端点 ED 链表头 |
| `HcBulkHeadED` | 批量端点 ED 链表头 |
| `HcDoneHead` | 已完成 TD 链表头（由 HCCA 提供） |
| `HcFmInterval` | 帧间隔（12000 位时间 = 1 ms） |
| `HcRhDescriptorA/B` | 根 Hub 描述符 |
| `HcRhPortStatus[N]` | 各端口状态 |

`HcControl.HCFS`（Host Controller Functional State）状态机：

```
USB_RESET → USB_RESUME → USB_OPERATIONAL → USB_SUSPEND
                ↑                                ↓
                └──────── 恢复事件 ───────────────┘
```

### 2.3 HCCA（Host Controller Communications Area）

HCCA 是 256 字节的 **DMA 一致性共享内存**，是 CPU 与 OHCI 硬件通信的核心桥梁：

```c
struct ohci_hcca {
    __hc32 int_table[32];  /* 32 个周期性 ED 槽的头指针（硬件直接读取） */
    __hc32 frame_no;       /* 当前帧号（硬件更新，每帧递增） */
    __hc32 done_head;      /* 已完成 TD 链表头（硬件在 WDH 中断前填写） */
    u8     reserved[116];
    u8     what[4];
};
```

驱动通过 `ohci->hcca->done_head` 获取完成 TD 链，无需轮询整个描述符链。

### 2.4 端点描述符 ED（`ohci.h:26`）

ED 是 OHCI 中 QH 的对应概念，**硬件直接遍历 ED 链**：

```c
struct ed {
    /* 硬件字段（16 字节，硬件直接读写） */
    __hc32 hwINFO;    /* 端点配置：FA(设备地址)、EN(端点号)、D(方向)、S(速度)、F(格式)、MPS(最大包长) */
    __hc32 hwTailP;   /* TD 链表尾指针（驱动维护） */
    __hc32 hwHeadP;   /* TD 链表头指针（硬件推进，低位含 H/C 标志） */
    __hc32 hwNextED;  /* 下一个 ED 的物理地址 */

    /* 软件字段 */
    dma_addr_t dma;
    u8 state;         /* ED_IDLE / ED_UNLINK / ED_OPER */
    u8 type;          /* PIPE_BULK / CONTROL / INT / ISO */
    u16 interval;     /* 中断/ISO 轮询间隔 */
    u16 load;         /* 带宽占用（μs） */
} __attribute__((aligned(16)));
```

`hwHeadP` 低位标志：
- `ED_H (bit0)`：端点被 STALL 后由硬件置位，驱动清除
- `ED_C (bit1)`：数据翻转进位（carry），硬件维护

### 2.5 传输描述符 TD（`ohci.h:88`）

```c
struct td {
    /* 硬件字段（16 字节） */
    __hc32 hwINFO;    /* CC(4位完成码)、EC(错误计数)、T(数据翻转)、DP(方向/PID)、R(短包允许) */
    __hc32 hwCBP;     /* Current Buffer Pointer：当前缓冲区��置（硬件推进） */
    __hc32 hwNextTD;  /* 下一个 TD 物理地址 */
    __hc32 hwBE;      /* Buffer End：缓冲区结束地址 */

    /* 软件字段 */
    struct ed *ed;
    struct urb *urb;
    dma_addr_t td_dma;
    dma_addr_t data_dma;
} __attribute__((aligned(32)));
```

`hwINFO.CC`（Condition Code）完成码映射到 errno：

| CC | 含义 | errno |
|----|------|-------|
| 0x00 | 无错误 | 0 |
| 0x01 | CRC 错误 | `-EILSEQ` |
| 0x04 | STALL | `-EPIPE` |
| 0x05 | 设备无响应 | `-ETIME` |
| 0x08 | 数据溢出 | `-EOVERFLOW` |
| 0x09 | 数据不足 | `-EREMOTEIO` |

### 2.6 周期性调度（时间轮）

`ohci_hcd.periodic[32]` 是 32 个 1ms 槽，构成时间轮：

```
periodic[0]  → ED(1ms) → ED(2ms) → ED(4ms) → ... → NULL
periodic[1]  → ED(2ms) → ED(4ms) → ...
periodic[2]  → ED(1ms) → ED(4ms) → ...
...
```

一个间隔为 N 的中断 ED 会被插入 `32/N` 个槽，确保每 N ms 被访问一次。HCCA 的 `int_table[32]` 直接映射这 32 个槽头，硬件每帧从 `int_table[frame_no & 31]` 开始遍历。

### 2.7 中断处理

```
硬件中断 → ohci_irq()
  → 读 HcInterruptStatus
  → WDH（WritebackDoneHead）：HCCA.done_head 非空
      → dl_done_list()：遍历 Done Queue 中的已完成 TD
      → 每个 TD 映射回 URB → usb_hcd_giveback_urb()
  → SF（StartOfFrame）：帧起始，推进周期调度
  → RHSC（Root Hub Status Change）：端口热插拔
      → usb_hcd_poll_rh_status()
  → UE（Unrecoverable Error）：控制器重置
```

### 2.8 特点与适用场景

- 硬件遍历 ED 链，CPU 开销远低于 UHCI
- 被嵌入式平台广泛采用（ARM SoC 上极为常见）
- 因 HCCA 共享内存设计，调试性好
- 同样不支持 USB 2.0 高速，需配合 EHCI 使用

---

## 3. EHCI（Enhanced Host Controller Interface）

### 3.1 背景

EHCI 由 Intel 主导，于 2002 年随 USB 2.0 规范发布，支持 **480 Mbps 高速传输**。其设计基于 UHCI 的 QH/TD 结构，但大幅增强了硬件调度能力。EHCI 采用**伴随控制器**（Companion Controller）模型：自身只处理高速设备，低速/全速设备通过事务转换器（TT，Transaction Translator）桥接，或回退给同芯片的 UHCI/OHCI 伴随控制器处理。

### 3.2 寄存器模型（MMIO）

EHCI 有两级寄存器：

**能力寄存器（只读）`struct ehci_caps`**

| 寄存器 | 用途 |
|--------|------|
| `hc_capbase` | 能力寄存器长度（低 8 位）+ HCI 版本 |
| `hcs_params` | 结构参数：N_PORTS（端口数）、N_PCC（每个 CC 端口数）、N_CC（CC 数） |
| `hcc_params` | 能力参数：64 位 DMA、等待时间等 |

**操作寄存器（读写）`struct ehci_regs`**

| 寄存器 | 用途 |
|--------|------|
| `command` | RS、HCRESET、FLS（帧列表大小）、PSE（周期调度使能）、ASE（异步调度使能）、IAA |
| `status` | STS_ASS、STS_PSS、STS_INT（传输完成）、STS_ERR、STS_PCD、STS_IAA |
| `intr_enable` | 各中断使能 |
| `frame_index` | 当前微帧号（每微帧 125 μs，8 微帧 = 1 ms） |
| `async_next` | 异步 QH 环当前指针 |
| `periodic_base` | 周期性表物理基地址 |
| `port_status[N]` | 端口状态与控制 |

### 3.3 核心数据结构

#### `struct ehci_hcd`（`ehci.h:111`）

```c
struct ehci_hcd {
    /* 高精度定时器，管理 12 类 hrtimer 事件 */
    struct hrtimer          hrtimer;
    enum ehci_hrtimer_event next_hrtimer_event;

    struct ehci_caps __iomem *caps;
    struct ehci_regs __iomem *regs;
    spinlock_t lock;

    /* 异步调度 */
    struct ehci_qh  *async;          /* 异步 QH 环形链表头 */
    struct ehci_qh  *dummy;          /* AMD quirk 哑 QH */
    struct list_head async_unlink;   /* 正在卸载的 QH */
    unsigned         async_count;    /* 当前活跃异步 QH 数量 */

    /* 周期性调度 */
    unsigned    periodic_size;       /* 微帧指针表大小（默认 1024） */
    __hc32     *periodic;            /* 硬件微帧指针表（DMA） */
    dma_addr_t  periodic_dma;
    struct list_head intr_qh_list;   /* 中断 QH 链表 */

    /* ISO 传输缓存 */
    struct list_head cached_itd_list;   /* 高速 ISO ITD 列表 */
    struct list_head cached_sitd_list;  /* 全速 ISO SITD 列表 */

    /* DMA 池 */
    struct dma_pool *qh_pool;   /* QH：每活跃 URB 一个 */
    struct dma_pool *qtd_pool;  /* QTD：每个 QH 若干个 */
    struct dma_pool *itd_pool;  /* ITD：高速 ISO */
    struct dma_pool *sitd_pool; /* SITD：全速 ISO（分裂事务） */
};
```

#### `struct ehci_qh`（Queue Head）

EHCI QH 对应一个端点，包含端点能力信息和当前活跃 QTD 指针：

```c
struct ehci_qh_hw {             /* 硬件部分，DMA 可访问 */
    __hc32 hw_next;             /* 下一个 QH 物理地址（低位含类型标志） */
    __hc32 hw_info1;            /* RL(NAK 重试)、C(控制端点)、MaxPkt、H(头部QH)、DTC、EPS(速度)、EP、Device */
    __hc32 hw_info2;            /* Mult(事务乘数)、PortNum、HubAddr、C-mask、S-mask */
    __hc32 hw_current;          /* 当前 QTD 物理地址（硬件推进） */
    /* 硬件叠加区：当前 QTD 的内容镜像（硬件填写） */
    __hc32 hw_qtd_next;
    __hc32 hw_alt_next;
    __hc32 hw_token;
    __hc32 hw_buf[5];
    __hc32 hw_buf_hi[5];        /* 64 位 DMA 高 32 位 */
} __aligned(32);
```

`hw_info1` 关键字段：
- `EPS[13:12]`：端点速度（00=全速, 01=低速, 10=高速）
- `MaxPkt[26:16]`：端点最大包长
- `H[15]`：Head of Reclamation List（异步链表头标志）
- `DTC[14]`：数据翻转控制

#### `struct ehci_qtd`（Queue Transfer Descriptor）

```c
struct ehci_qtd_hw {            /* 硬件部分 */
    __hc32 hw_next;             /* 下一个 QTD 物理地址 */
    __hc32 hw_alt_next;         /* 错误时备用 QTD 地址 */
    __hc32 hw_token;            /* Status(8bit)、PID(2bit)、Cerr(2bit)、C_Page(3bit)、IOC、TotalBytes、DT */
    __hc32 hw_buf[5];           /* 最多 5 个 4KB 页物理地址 */
    __hc32 hw_buf_hi[5];        /* 64 位高位 */
} __aligned(32);
```

`hw_token` 关键字段：
- `Status[7:0]`：0x80=ACTIVE, 0x40=HALTED, 0x20=DBE, 0x10=BabbleDetected, 0x08=Transaction Error, 0x04=Missed MF, 0x02=Split State, 0x01=Ping State
- `TotalBytes[30:16]`：剩余字节数（倒计数）
- `PID[9:8]`：00=OUT, 01=IN, 10=SETUP
- `IOC[15]`：完成中断使能

### 3.4 ISO 传输描述符

**ITD（Isochronous Transfer Descriptor）**：用于高速 ISO 端点，一个 ITD 可覆盖一个微帧内最多 3072 字节（多事务），包含 8 个微帧独立事务描述字段（`hw_transaction[8]`）。

**SITD（Split-transaction Isochronous Transfer Descriptor）**：用于通过 TT 的全速 ISO 端点，描述分裂事务的 S-mask/C-mask 时序和缓冲区信息。

### 3.5 异步调度与 IAA 机制

异步传输（Control + Bulk）使用**环形 QH 链**：

```
async → QH(ctrl) → QH(bulk1) → QH(bulk2) → async（回到头部）
           ↓              ↓
         QTD链           QTD链
```

**IAA（Interrupt on Async Advance）机制**——安全卸载 QH 的关键：

1. 驱动从环中摘除目标 QH（软件操作）
2. 写 `USBCMD.IAA` 位，请求硬件完成当前异步推进轮次
3. 硬件在推进到 dummy QH 后触发 `STS_IAA` 中断
4. 驱动收到 IAA 中断后确认硬件不再访问该 QH，安全释放

若不等 IAA，可能出现 UAF（Use After Free）竞争。

### 3.6 周期性调度

周期性表（`periodic[1024]`）每项对应一个 125 μs 微帧，指向 QH 或 ITD 链表。

- 中断端点：按 `2^n` 微帧间隔插入到对应周期表项中（类似 UHCI 骨架树）
- `ehci_per_sched.cs_mask`：S-mask（发起分裂）和 C-mask（完成分裂）位图控制 TT 事务时序
- `uframe_periodic_max`：每微帧最多允许占用的周期性带宽（μs），默认 100 μs（125 μs 的 80%）

### 3.7 hrtimer 事件体系

EHCI 用高精度定时器替代帧定时器，管理 12 类异步事件（`enum ehci_hrtimer_event`）：

| 事件 | 触发时机 |
|------|----------|
| `EHCI_HRTIMER_POLL_ASS` | 轮询异步调度是否已关闭 |
| `EHCI_HRTIMER_POLL_PSS` | 轮询周期调度是否已关闭 |
| `EHCI_HRTIMER_UNLINK_INTR` | 等待中断 QH 卸载完成 |
| `EHCI_HRTIMER_ASYNC_UNLINKS` | 批量卸载空闲异步 QH |
| `EHCI_HRTIMER_IAA_WATCHDOG` | 看门狗：检测丢失的 IAA 中断 |
| `EHCI_HRTIMER_IO_WATCHDOG` | 检测丢失的传输完成中断 |

### 3.8 中断处理

```
ehci_irq()
  → 读 STS 寄存器
  → STS_IAA：IAA 确认 → 安全释放已卸载 QH
  → STS_INT：传输完成
      → scan_async()：扫描异步 QH 链，giveback 完成 URB
      → scan_intr()：扫描中断 QH 链
      → scan_isoc()：扫描 ITD/SITD
  → STS_PCD：端口状态变化 → usb_hcd_poll_rh_status()
  → STS_ERR：传输错误（含 STS_INT）
  → STS_FLR：帧列表翻转（帧号溢出）
```

---

## 4. xHCI（eXtensible Host Controller Interface）

### 4.1 背景

xHCI 由 Intel 主导，随 USB 3.0 于 2010 年发布，是目前最现代的 USB 主机控制器规范。其革命性改进在于：**一个控制器同时支持 USB 1.x/2.0/3.x 全速度**，彻底消除了伴随控制器的需求。此外，xHCI 将调度逻辑完全移入硬件，驱动只需构造 TRB 并写入门铃，硬件自动完成所有调度。

### 4.2 寄存器模型

xHCI 寄存器区域分为四块：

| 寄存器组 | 结构体 | 内容 |
|---------|--------|------|
| 能力寄存器 | `struct xhci_cap_regs` | 版本、能力参数（MaxSlots、MaxPorts、MaxInterrupters）、能力扩展链 |
| 操作寄存器 | `struct xhci_op_regs` | USBCMD、USBSTS、DCBAAP、CRCR（命令环）、端口状态 |
| 运行寄存器 | `struct xhci_run_regs` | MF_INDEX（微帧索引）、中断器寄存器数组 |
| 门铃寄存器 | `struct xhci_doorbell_array` | 每个设备槽/端点独立的门铃（写入触发处理） |

`USBCMD` 关键位：

| 位 | 含义 |
|----|------|
| `RS` | 运行/停止 |
| `HCRST` | 控制器复位 |
| `INTE` | 中断使能 |
| `HSEE` | 主机系统错误使能 |
| `EWE` | 远端唤醒使能 |

`USBSTS` 关键位：

| 位 | 含义 |
|----|------|
| `HCH` | 控制器已停止 |
| `CNR` | 控制器未就绪 |
| `HSE` | 主机系统错误 |
| `EINT` | 事件中断 |
| `PCD` | 端口状态变化 |

### 4.3 核心数据结构

#### `struct xhci_hcd`（`xhci.h:1501`）

```c
struct xhci_hcd {
    struct usb_hcd *main_hcd;    /* USB 2.0 根 Hub 对应的 HCD */
    struct usb_hcd *shared_hcd;  /* USB 3.x 根 Hub 对应的 HCD */

    /* 寄存器映射 */
    struct xhci_cap_regs      __iomem *cap_regs;
    struct xhci_op_regs       __iomem *op_regs;
    struct xhci_run_regs      __iomem *run_regs;
    struct xhci_doorbell_array __iomem *dba;

    /* 全局数据结构 */
    struct xhci_device_context_array *dcbaa;   /* 设备上下文基址数组（物理地址表） */
    struct xhci_interrupter **interrupters;     /* 中断器数组（支持 MSI-X 多中断） */
    struct xhci_ring *cmd_ring;                 /* 命令环（Host → 控制器命令） */
    struct xhci_virt_device *devs[MAX_HC_SLOTS]; /* 每个设备槽的虚拟设备 */

    /* DMA 池 */
    struct dma_pool *device_pool;         /* 设备/输入/端点上下文（2KB/4KB） */
    struct dma_pool *segment_pool;        /* TRB 环段（4KB） */
    struct dma_pool *small_streams_pool;  /* 小流上下文数组 */
    struct dma_pool *medium_streams_pool; /* 中流上下文数组 */

    spinlock_t lock;
    unsigned long long quirks;            /* 硬件 Quirk 位图（超过 40 个已知问题） */
};
```

#### TRB（Transfer Request Block）

TRB 是 xHCI 的最小传输单元，固定 **16 字节**，按功能分为三类环：

**传输 TRB（Transfer TRB）**，构成端点传输环：

```
┌─────────────────────────────────────────────┐
│ Data Buffer Pointer Lo [31:0]               │ +0
├─────────────────────────────────────────────┤
│ Data Buffer Pointer Hi [31:0]               │ +4
├─────────────────────────────────────────────┤
│ TRB Transfer Length[16:0] | TD Size[21:17]  │ +8
│ Interrupter Target[31:22]                   │
├─────────────────────────────────────────────┤
│ C | ENT | ISP | NS | CH | IOC | IDT | ...  │ +12
│ TRB Type[15:10] | DIR[16] | ...             │
└─────────────────────────────────────────────┘
```

关键标志位：
- `C`（Cycle）：与环的 PCS（Producer Cycle State）配合，实现无锁环操作
- `IOC`（Interrupt On Completion）：该 TRB 完成后触发事件
- `CH`（Chain）：与下一个 TRB 属于同一个 TD（Transfer Descriptor）
- `ISP`（Interrupt on Short Packet）：短包时触发事件

TRB 类型（`TRB Type[15:10]`）：

| 值 | TRB 类型 | 用途 |
|----|---------|------|
| 1 | Normal | 普通数据传输 |
| 2 | Setup Stage | 控制传输 Setup 阶段 |
| 3 | Data Stage | 控制传输数据阶段 |
| 4 | Status Stage | 控制传输状态阶段 |
| 5 | Isoch | ISO 传输 |
| 6 | Link | 环段链接（跳到下一段） |
| 7 | Event Data | 事件数据 TRB |
| 33 | Transfer Event | 完成事件（事件环使用） |
| 34 | Command Completion Event | 命令完成事件 |
| 35 | Port Status Change Event | 端口状态变化事件 |

**命令 TRB（Command TRB）**，构成命令环（Host → 控制器）：

| 命令 | 用途 |
|------|------|
| Enable Slot | 为新设备分配槽 ID |
| Disable Slot | 释放设备槽 |
| Address Device | 执行 SET_ADDRESS（Block/No-BSR 模式） |
| Configure Endpoint | 更新端点上下文（SET_CONFIGURATION） |
| Evaluate Context | 更新端点参数（不改变调度） |
| Reset Endpoint | 清除端点 HALT 状态 |
| Stop Endpoint | 停止端点传输环 |
| Set TR Dequeue Pointer | 更新传输环出队指针（用于取消 URB） |

**事件 TRB（Event TRB）**，构成事件环（控制器 → Host）：通过 ERDP（Event Ring Dequeue Pointer）和 EREP（Event Ring Enqueue Pointer）实现无锁生产者消费者模型。

#### `struct xhci_ring`（传输/命令/事件环）

```c
struct xhci_ring {
    struct xhci_segment *first_seg;   /* 环的第一个段 */
    struct xhci_segment *last_seg;    /* 环的最后一个段 */
    union xhci_trb      *enqueue;     /* 生产者写入位置 */
    struct xhci_segment *enq_seg;
    union xhci_trb      *dequeue;     /* 消费者读取位置 */
    struct xhci_segment *deq_seg;
    unsigned int cycle_state;         /* PCS/CCS：0 或 1，控制 C bit */
    unsigned int stream_id;           /* 流 ID（Bulk Streams 使用） */
    unsigned int num_segs;            /* 环段数量 */
    unsigned int num_trbs_free;       /* 可用 TRB 数量 */
    enum xhci_ring_type type;         /* TYPE_CTRL/BULK/INTR/ISOC/COMMAND/EVENT */
    bool last_td_was_short;
};

struct xhci_segment {
    union xhci_trb  *trbs;           /* TRB 数组（一段 = 256 个 TRB = 4KB） */
    struct xhci_segment *next;       /* 下一段（形成环形链） */
    dma_addr_t dma;                  /* 该段的物理基地址 */
    struct xhci_dma_buf *bounce_buf; /* bounce buffer（用于 USB 3.1 SSP） */
};
```

### 4.4 设备上下文体系

#### DCBAA（Device Context Base Address Array）

```c
struct xhci_device_context_array {
    __le64 dev_context_ptrs[MAX_HC_SLOTS]; /* 每个槽的设备上下文物理地址 */
    dma_addr_t dma;
};
```

槽 0 指向 Scratchpad Buffer Array（控制器私有内存）。槽 1～MaxSlots 各对应一个已分配的 USB 设备。

#### 设备上下文（Device Context）

每个设备的上下文包含：
- **Slot Context**（32 字节）：设备状态、路由字符串、速度、端口号、Hub 信息
- **Endpoint Context × 31**（每个 32 字节）：端点状态（Stopped/Running/Halted）、传输环 TRDP（Dequeue Pointer）、最大 ESIT 负载、间隔、最大突发数

端点上下文（`EP Context`）关键字段：
- `EP State[2:0]`：0=Disabled, 1=Running, 2=Halted, 3=Stopped, 4=Error
- `TR Dequeue Pointer`：指向传输环当前出队位置（含 DCS bit）
- `Max Burst Size`：超速端点的最大突发包数（0～15）
- `Interval`：中断/ISO 端点的调度间隔（2^n 微帧）
- `Max ESIT Payload`：每个 ESIT（End-to-end Service Interval Time）的最大数据量

#### 输入上下文（Input Context）

在配置端点时，驱动构造输入上下文（Input Context = Input Control Context + Device Context），通过 Configure Endpoint 命令提交给控制器。Input Control Context 中的 `add_flags` 和 `drop_flags` 位图指示哪些端点上下文需要更新。

### 4.5 中断器（Interrupter）与 MSI-X

xHCI 支持多个中断器（最多 `MaxInterrupters` 个），每个中断器有独立的事件环和寄存器：

```c
struct xhci_interrupter {
    struct xhci_ring      *event_ring;    /* 该中断器的事件环 */
    struct xhci_erst       erst;          /* Event Ring Segment Table */
    struct xhci_intr_reg __iomem *ir_set; /* 中断器寄存器（IMAN/IMOD/ERSTSZ/ERDP/ERSTBA） */
    u32    s3_irq_pending;                /* S3 挂起状态保存 */
    u32    s3_irq_control;
    u32    s3_erst_size;
    u64    s3_erst_base;
    u64    s3_erdp;
};
```

中断器寄存器：
- `IMAN`（Interrupt Management）：IP（中断挂起）、IE（中断使能）
- `IMOD`（Interrupt Moderation）：IMODI（间隔，单位 250ns）、IMODC（计数器）
- `ERSTBA`：Event Ring Segment Table 物理基地址
- `ERDP`：Event Ring Dequeue Pointer（驱动更新，通知控制器已消费的位置）

### 4.6 传输流程

#### URB 提交（以 Bulk 为例）

```
xhci_urb_enqueue(hcd, urb, mem_flags)
  → 分配 xhci_td（URB 对应的 TD 描述符）
  → xhci_queue_bulk_tx()
      → 计算需要的 TRB 数量（数据长度 / 每 TRB 最大 64KB）
      → 逐个入队 Normal TRB 到端点传输环
          → 最后一个 TRB 设 IOC 位
          → 中间 TRB 设 CH（Chain）位
      → 写门铃（writel(ep_index, &xhci->dba->doorbell[slot_id])）
  → 硬件处理传输环，发送 USB 包
  → 完成后写事件 TRB 到事件环 → 触发中断
```

#### 事件处理

```
xhci_irq(hcd)
  → 读 IMAN.IP，确认中断 pending
  → xhci_handle_event(xhci, ir)：消费事件环中的 TRB
      → Transfer Event TRB：
          → 找到对应的 xhci_td
          → 记录完成状态（Completion Code）
              → 0=Invalid, 1=Success, 2=Data Buffer Error
              → 4=USB Transaction Error, 5=TRB Error
              → 6=Stall, 13=Short Packet, ...
          → xhci_giveback_urb_in_irq() → usb_hcd_giveback_urb()
      → Command Completion Event TRB：
          → 唤醒等待命令完成的线程（complete(&xhci->current_cmd->...)）
      → Port Status Change Event TRB：
          → usb_hcd_poll_rh_status()
  → 更新 ERDP（通知控制器已消费的事件位置）
```

### 4.7 流（Streams）支持

USB 3.0 引入 Bulk Streams，允许一个 Bulk 端点并发多个独立数据流（Stream ID 1～65534）。xHCI 为每个流维护独立的传输环，通过 Stream Context Array 管理：

```
端点上下文.Stream Context Array 物理地址
    → StreamCtx[1].TR Dequeue Pointer → 传输环(stream 1)
    → StreamCtx[2].TR Dequeue Pointer → 传输环(stream 2)
    ...
```

Linux 驱动用 `small_streams_pool`（≤2 流）和 `medium_streams_pool`（3～16 流）分配 Stream Context Array。

### 4.8 Quirk 系统

xHCI 包含超过 40 个已知硬件问题的软件 Workaround，通过 PCI VID/DID 匹配并设置 `xhci->quirks` 位图：

| Quirk | 含义 |
|-------|------|
| `XHCI_NEC_HOST` | NEC 控制器固件命令序列特殊处理 |
| `XHCI_EP_LIMIT_QUIRK` | Intel 控制器端点上下文数量限制 |
| `XHCI_BROKEN_MSI` | MSI 中断不可用，回退到线中断 |
| `XHCI_RESET_ON_RESUME` | S3 恢复后必须完整复位控制器 |
| `XHCI_LPM_SUPPORT` | 支持 U1/U2 链路电源管理 |
| `XHCI_TRUST_TX_LENGTH` | 信任控制器报告的实际传输长度 |

---

## 5. 四代控制器架构演进总结

```
UHCI (1995)         OHCI (1999)         EHCI (2002)         xHCI (2010)
USB 1.1             USB 1.1             USB 2.0             USB 3.x
12 Mbps             12 Mbps             480 Mbps            5/10/20 Gbps

软件主导调度         硬件遍历 ED         硬件异步+           全硬件 TRB 环
CPU 遍历帧列表       软件维护 HCCA       软件管理周期表       门铃触发调度
1024 帧指针          32 周期槽           1024 微帧表         无需维护调度表

TD 链表             TD 链表             QTD 链              TRB 环形结构
QH 骨架树           ED 时间轮           QH 环 + IAA         事件环通知
轮询完成标志         WDH Done Queue      STS_INT + IAA       MSI-X 事件环

IO 端口寄存器        MMIO 寄存器         MMIO 寄存器          MMIO 四组寄存器
PCI                  PCI / SoC          PCI                  PCIe / SoC

需伴随 UHCI/OHCI    需伴随 OHCI/UHCI    需伴随 OHCI/UHCI    单控制器全覆盖
处理低速/全速        处理低速/全速        处理低速/全速（TT）  所有速度
```

### 设备支持策略演变

```
USB 2.0 时代（EHCI + OHCI/UHCI 共存）：

PCIe 总线
  ├── EHCI 控制器（处理高速设备）
  │     └── 根 Hub 端口
  │           └── USB 2.0 设备 → 直连 EHCI
  │           └── USB 1.1 设备 → TT 降速 → EHCI 调度
  └── OHCI/UHCI 控制器（Companion Controller，处理无 TT 时的低速/全速）
        └── 通过 Port Routing 接管被 EHCI 释放的端口

USB 3.x 时代（xHCI 单控制器）：

PCIe 总线
  └── xHCI 控制器（单芯片）
        ├── USB 3.x 根 Hub（shared_hcd）→ SuperSpeed 端口
        └── USB 2.0 根 Hub（main_hcd）→ HighSpeed/FullSpeed/LowSpeed 端口
              （同一物理端口向下兼容）
```

---

## 6. Linux 内核驱动关键文件索引

| 文件 | 行数 | 职责 |
|------|------|------|
| `drivers/usb/host/uhci-hcd.c` | ~2400 | UHCI 主体：初始化、IRQ、调度 |
| `drivers/usb/host/uhci-hcd.h` | ~440 | UHCI 结构体/寄存器定义 |
| `drivers/usb/host/uhci-q.c` | ~1600 | UHCI QH/TD 管理与 URB 入队 |
| `drivers/usb/host/uhci-debug.c` | ~400 | UHCI 调试接口（debugfs） |
| `drivers/usb/host/ohci-hcd.c` | ~1300 | OHCI 主体：初始化、IRQ |
| `drivers/usb/host/ohci.h` | ~480 | OHCI 结构体/寄存器定义 |
| `drivers/usb/host/ohci-q.c` | ~1300 | OHCI ED/TD 管理与 URB 入队 |
| `drivers/usb/host/ohci-mem.c` | ~140 | OHCI DMA 池管理 |
| `drivers/usb/host/ehci-hcd.c` | ~1500 | EHCI 主体：初始化、IRQ |
| `drivers/usb/host/ehci.h` | ~560 | EHCI 结构体/寄存器定义 |
| `drivers/usb/host/ehci-q.c` | ~1700 | EHCI 异步 QH/QTD 管理 |
| `drivers/usb/host/ehci-sched.c` | ~2491 | EHCI 周期性调度（最复杂） |
| `drivers/usb/host/ehci-timer.c` | ~400 | EHCI hrtimer 事件处理 |
| `drivers/usb/host/xhci.c` | ~5705 | xHCI 主体：初始化、IRQ、命令处理 |
| `drivers/usb/host/xhci.h` | ~2000+ | xHCI 所有结构体定义 |
| `drivers/usb/host/xhci-ring.c` | ~4487 | xHCI 环管理：TRB 入队、事件处理 |
| `drivers/usb/host/xhci-mem.c` | ~2200 | xHCI 内存管理：上下文、环、流 |
| `drivers/usb/host/xhci-hub.c` | ~1400 | xHCI 根 Hub 模拟与端口管理 |
| `drivers/usb/host/xhci-pci.c` | ~600 | xHCI PCI 驱动（含 Quirk 识别） |
