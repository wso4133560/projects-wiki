# Linux USB Class 驱动详细文档

> 基于 Linux 6.19.6 内核源码
> 覆盖 `drivers/usb/class/`、`drivers/usb/serial/`、`drivers/usb/storage/`、`drivers/hid/usbhid/`

---

## 目录

1. [USB Class 驱动体系概述](#1-usb-class-驱动体系概述)
2. [CDC-ACM（串口/调制解调器）](#2-cdc-acm串口调制解调器)
3. [CDC-WDM（无线广域网控制）](#3-cdc-wdm无线广域网控制)
4. [USB 打印机（usblp）](#4-usb-打印机usblp)
5. [USB 测试测量（usbtmc）](#5-usb-测试测量usbtmc)
6. [USB HID（人机接口）](#6-usb-hid人机接口)
7. [USB 串口适配层（usb-serial）](#7-usb-串口适配层usb-serial)
8. [USB 大容量存储（usb-storage / UAS）](#8-usb-大容量存储usb-storage--uas)
9. [Class 驱动注册机制](#9-class-驱动注册机制)
10. [跨类驱动对比](#10-跨类驱动对比)

---

## 1. USB Class 驱动体系概述

### 1.1 分层结构

```
用户空间（/dev/ttyACM0、/dev/bus/usb、/dev/sg*、/dev/usblp0）
    ↕  字符设备 / TTY / SCSI
─────────────────────────────────────────────────────────────
USB Class 驱动层
  ┌──────────┬──────────┬──────────┬──────────┬──────────┐
  │ cdc-acm  │ cdc-wdm  │  usblp   │ usbtmc   │ usbhid   │
  │ serial/  │          │          │          │          │
  │ storage/ │          │          │          │          │
  └────┬─────┴────┬─────┴────┬─────┴────┬─────┴────┬─────┘
       │           │          │          │          │
─────────────────────────────────────────────────────────────
USB Core（urb/driver/hub）
─────────────────────────────────────────────────────────────
HCD（xHCI / EHCI / OHCI / UHCI）
─────────────────────────────────────────────────────────────
硬件
```

### 1.2 USB 设备类与内核驱动对应关系

| USB Class | 值 | 内核驱动位置 | 用户可见接口 |
|-----------|----|------------|------------|
| HID | 0x03 | `drivers/hid/usbhid/` | input 设备 / hiddev |
| Printer | 0x07 | `drivers/usb/class/usblp.c` | `/dev/usb/lpN` |
| Mass Storage | 0x08 | `drivers/usb/storage/` | `/dev/sd*` / SCSI |
| CDC ACM | 0x02 / Sub 0x02 | `drivers/usb/class/cdc-acm.c` | `/dev/ttyACM*` |
| CDC WDM | 0x02 / Sub 0x0A | `drivers/usb/class/cdc-wdm.c` | `/dev/cdc-wdm*` |
| CDC ECM/NCM/RNDIS | 0x02 | `drivers/net/usb/` | 网络接口 `eth*` / `usb*` |
| Audio | 0x01 | `sound/usb/` | ALSA 声卡 |
| Video | 0x0E | `drivers/media/usb/uvc/` | V4L2 设备 |
| USBTMC | 0xFE / Sub 0x03 | `drivers/usb/class/usbtmc.c` | `/dev/usbtmcN` |
| USB Serial | 多种 | `drivers/usb/serial/` | `/dev/ttyUSB*` |

### 1.3 Class 驱动与设备驱动的区别

- **USB Interface Driver（`usb_driver`）**：绑定到接口（`usb_interface`），一个设备有几个接口就可以绑定几个驱动
- **USB Device Driver（`usb_device_driver`）**：绑定到整个设备（`usb_device`），较少使用
- **Class 驱动**：实现某一 USB 类协议，对上层暴露标准内核接口（TTY / SCSI / input）

---

## 2. CDC-ACM（串口/调制解调器）

**文件**：`drivers/usb/class/cdc-acm.c` / `cdc-acm.h`（2129 行）
**用户接口**：`/dev/ttyACMN`（N 从 0 开始）

### 2.1 协议概述

CDC-ACM（Communications Device Class - Abstract Control Model）是 USB 标准串口/调制解调器类。实现 USB 规范 Class 0x02、SubClass 0x02、Protocol 0x00/0x01。

**接口结构**（标准 CDC 设备使用两个接口）：
```
Interface 0（控制接口，Class=0x02, SubClass=0x02）
  └── Endpoint: Interrupt IN（用于状态通知）

Interface 1（数据接口，Class=0x0A CDC-Data）
  ├── Endpoint: Bulk IN（数据接收）
  └── Endpoint: Bulk OUT（数据发送）
```

某些简单设备使用**单接口**（`combined_interfaces`），控制和数据端点合并在一个接口。

### 2.2 核心数据结构

```c
// drivers/usb/class/cdc-acm.h:57
struct acm {
    struct usb_device    *dev;          /* 对应的 USB 设备 */
    struct usb_interface *control;      /* 控制接口（Class=0x02）*/
    struct usb_interface *data;         /* 数据接口（Class=0x0A）*/
    unsigned in, out;                   /* Bulk IN / OUT 管道 */
    struct tty_port port;               /* 内嵌 TTY 端口（连接 TTY 层）*/

    /* 控制通道 */
    struct urb           *ctrlurb;      /* INT IN URB（接收串口状态通知）*/
    u8                   *ctrl_buffer;  /* 控制 URB 数据缓冲区 */

    /* 写通道（ACM_NW 个并行写 URB）*/
    struct acm_wb         wb[ACM_NW];   /* 写缓冲区数组 */
    spinlock_t            write_lock;

    /* 读通道（ACM_NR 个并行读 URB）*/
    struct urb           *read_urbs[ACM_NR]; /* Bulk IN URB 池 */
    struct acm_rb         read_buffers[ACM_NR];
    unsigned long         read_urbs_free;    /* 空闲 URB 位图 */
    spinlock_t            read_lock;

    /* 串口状态 */
    struct usb_cdc_line_coding line;    /* 波特率、停止位、校验位 */
    unsigned int ctrlin;                /* 输入控制线（DCD/DSR/RI）*/
    unsigned int ctrlout;               /* 输出控制线（DTR/RTS）*/
    struct async_icount iocount;        /* 控制线变化计数器 */

    unsigned int writesize;             /* Bulk OUT 端点最大包大小 */
    unsigned int readsize, ctrlsize;    /* 缓冲区大小 */
    unsigned int minor;                 /* 设备次设备号 */
    struct mutex mutex;
    bool disconnected;
    unsigned long quirks;               /* 设备特异性修正标志 */
    struct delayed_work dwork;          /* 延迟工作队列（恢复等）*/
};
```

### 2.3 ACM 类特定请求

通过 EP0 控制传输发送（`bRequest` 对应 CDC 规范 Table 3）：

| 请求 | 值 | 方向 | 功能 |
|------|----|------|------|
| `SET_LINE_CODING` | 0x20 | OUT | 设置波特率/停止位/校验（7字节）|
| `GET_LINE_CODING` | 0x21 | IN | 读取当前行编码 |
| `SET_CONTROL_LINE_STATE` | 0x22 | OUT | 控制 DTR/RTS（wValue[0]=DTR, [1]=RTS）|
| `SEND_BREAK` | 0x23 | OUT | 发送 Break 信号（wValue=持续时间 ms）|

`usb_cdc_line_coding` 结构（7字节）：
```c
struct usb_cdc_line_coding {
    __le32 dwDTERate;   /* 波特率（bps），如 115200 */
    __u8   bCharFormat; /* 停止位：0=1位 1=1.5位 2=2位 */
    __u8   bParityType; /* 校验：0=None 1=Odd 2=Even 3=Mark 4=Space */
    __u8   bDataBits;   /* 数据位：5/6/7/8/16 */
} __attribute__ ((packed));
```

### 2.4 ACM 通知（Interrupt IN）

设备通过 INT IN 端点异步通知主机串口状态变化，通知格式（`usb_cdc_notification`）：

| 通知类型 | 值 | 数据 |
|----------|----|------|
| `NETWORK_CONNECTION` | 0x00 | wValue=1/0（连接/断开）|
| `RESPONSE_AVAILABLE` | 0x01 | 无（设备有响应待读取）|
| `SERIAL_STATE` | 0x20 | 2字节串口状态位图 |

**串口状态位**（`bRxCarrier`/`bTxCarrier` 等）：
- bit0 = DCD（Data Carrier Detect）
- bit1 = DSR（Data Set Ready）
- bit2 = Break
- bit3 = Ring Indicator
- bit4 = Framing Error
- bit5 = Parity Error
- bit6 = Overrun

### 2.5 数据流与调用链

**发送路径（TTY write → Bulk OUT）**：

```
用户 write() → tty_write()
    → acm_tty_write(tty, buf, count)        [cdc-acm.c:814]
        → acm_wb_alloc()：从 wb[] 中分配空闲写缓冲区
        → memcpy(wb->buf, buf, count)
        → acm_start_wb(acm, wb)             [cdc-acm.c:229]
            → usb_fill_bulk_urb(wb->urb, ...)
            → usb_submit_urb(wb->urb)        [提交 Bulk OUT URB]
        → acm_write_bulk()：写完成回调      [cdc-acm.c:590]
            → acm_wb_free(wb)：释放写缓冲区
            → tty_port_tty_wakeup()：通知 TTY 层可继续写入
```

**接收路径（Bulk IN → TTY read）**：

```
硬件 Bulk IN 完成中断
    → acm_read_bulk_callback(urb)           [cdc-acm.c:516]
        → 检查 urb->status
        → process_read_urb()
            → tty_insert_flip_string()：将数据推入 TTY flip buffer
            → tty_flip_buffer_push()：通知 TTY 层有数据可读
        → acm_submit_read_urb()：重新提交 Bulk IN URB（循环接收）
```

**控制通知路径（INT IN → 状态更新）**：

```
设备发送 SERIAL_STATE 通知
    → acm_ctrl_irq(urb)                     [cdc-acm.c:371]
        → 解析 usb_cdc_notification
        → 更新 acm->ctrlin（DCD/DSR/RI/Break）
        → tty_port_tty_hangup()（若 DCD 丢失）
        → wake_up_interruptible(&acm->wioctl)
        → usb_submit_urb(acm->ctrlurb)：重新提交 INT IN
```

### 2.6 probe/disconnect 流程

```
acm_probe(intf, id)                          [cdc-acm.c:1182]
    → usb_cdc_get_union_descriptors()：解析 CDC Union 描述符
      找到控制接口 + 数据接口
    → usb_cdc_get_acm_capabilities()：解析 ACM 功能描述符（GET_LINE_CODING 等支持）
    → usb_cdc_get_calling_capabilities()：解析呼叫管理能力
    → 分配 struct acm
    → 分配读/写 URB 池和缓冲区
    → tty_port_init(&acm->port)
    → usb_fill_int_urb(acm->ctrlurb, ...)：准备 INT IN URB
    → usb_submit_urb(acm->ctrlurb)：启动控制通知接收
    → tty_register_device()：注册 /dev/ttyACMN

acm_disconnect(intf)                         [cdc-acm.c:1572]
    → usb_kill_urb(acm->ctrlurb)：停止控制 URB
    → acm_kill_urbs()：停止所有读写 URB
    → tty_unregister_device()
    → tty_port_tty_hangup()：挂断 TTY
```

---

## 3. CDC-WDM（无线广域网控制）

**文件**：`drivers/usb/class/cdc-wdm.c`（1378 行）
**用户接口**：`/dev/cdc-wdmN`
**用途**：MBIM（Mobile Broadband Interface Model）、QMI 等 4G/5G 模组控制通道

### 3.1 协议概述

CDC-WDM（Wireless Device Management）提供 USB 控制平面通道，供用户空间工具（如 ModemManager、mbimcli）发送 AT 命令或 MBIM 控制消息。

**端点结构**：
```
Interface（Class=0x02, SubClass=0x0A）
  ├── Endpoint: Control（EP0，用于发送命令）
  └── Endpoint: Interrupt IN（用于接收响应通知）
```

命令通过 **EP0 控制传输**（`SET_CONTROL_LINE_STATE` 等类请求）发送，响应通过 **INT IN** 异步接收。

### 3.2 核心数据结构

```c
// drivers/usb/class/cdc-wdm.c:76
struct wdm_device {
    u8              *inbuf;     /* INT IN 响应缓冲区 */
    u8              *outbuf;    /* 控制传输命令缓冲区 */
    u8              *sbuf;      /* 状态缓冲区 */
    u8              *ubuf;      /* 拷贝到用户空间的缓冲区 */

    struct urb      *command;   /* OUT 控制传输 URB（发命令）*/
    struct urb      *response;  /* INT IN URB（收通知）*/
    struct urb      *validity;  /* 初始化查询 URB */

    struct usb_interface    *intf;
    struct usb_ctrlrequest  *orq; /* 发送控制请求结构 */
    struct usb_ctrlrequest  *irq; /* 接收控制请求结构 */

    u16             bufsize;        /* 用户 I/O 缓冲区大小 */
    u16             wMaxCommand;    /* 最大命令大小（从描述符获取）*/
    u16             wMaxPacketSize; /* INT IN 最大包大小 */

    spinlock_t      iuspin;
    struct mutex    wlock, rlock;
    wait_queue_head_t wait;
    struct work_struct rxwork;          /* 响应处理工作队列 */
    struct work_struct service_outs_intr;

    /* WWAN 子系统集成 */
    enum wwan_port_type wwanp_type;
    struct wwan_port    *wwanp;
};
```

### 3.3 数据流

```
用户 write() （发送命令）:
    → wdm_write()
        → memcpy(desc->outbuf, user_data, count)
        → usb_fill_control_urb(desc->command, ...)
          [bRequest = CDC_SEND_ENCAPSULATED_COMMAND (0x00)]
        → usb_submit_urb(desc->command)

设备响应（收通知）:
    → INT IN 完成 → wdm_int_callback()
        → 调度 rxwork 工作队列
    → wdm_rxwork()
        → usb_control_msg(GET_ENCAPSULATED_RESPONSE, 0x01)
          从设备读取完整响应数据 → 存入 inbuf
        → wake_up(&desc->wait)

用户 read():
    → wdm_read()
        → wait_event_interruptible(desc->wait, ...)：等待响应
        → copy_to_user(ubuf, inbuf, count)
```

---

## 4. USB 打印机（usblp）

**文件**：`drivers/usb/class/usblp.c`（1487 行）
**用户接口**：`/dev/usb/lpN`（N 从 0 开始）
**USB Class**：Printer（0x07），SubClass 1，Protocol 1/2/3

### 4.1 协议与端点

USB 打印机类定义三种协议（altsetting）：

| 协议 | 协议值 | 端点 | 说明 |
|------|--------|------|------|
| Unidirectional | 1 | Bulk OUT only | 仅支持打印（发送）|
| Bidirectional | 2 | Bulk OUT + Bulk IN | 双向，可查询打印机状态 |
| 1284.4 compatible | 3 | Bulk OUT + Bulk IN | IEEE 1284.4 多路复用 |

驱动优先选择最高协议号（向下兼容）。

### 4.2 核心数据结构

```c
// drivers/usb/class/usblp.c:132
struct usblp {
    struct usb_device    *dev;
    struct mutex wmut, mut;
    spinlock_t lock;
    char *readbuf;              /* Bulk IN 读缓冲区 */
    char *statusbuf;            /* 状态查询缓冲区 */
    struct usb_anchor urbs;

    struct {
        int alt_setting;
        struct usb_endpoint_descriptor *epwrite; /* Bulk OUT 端点 */
        struct usb_endpoint_descriptor *epread;  /* Bulk IN 端点（双向模式）*/
    } protocol[USBLP_MAX_PROTOCOLS]; /* 每种协议的端点配置 */

    int current_protocol;      /* 当前使用的协议号 */
    int minor;
    unsigned char bidir;       /* 1 = 双向模式 */
    unsigned char no_paper;    /* 缺纸标志 */
    unsigned char *device_id_string; /* IEEE 1284 设备 ID 字符串 */
};
```

### 4.3 类特定请求

| 请求 | 值 | 方向 | 功能 |
|------|----|------|------|
| `GET_DEVICE_ID` | 0x00 | IN | 获取 IEEE 1284 设备 ID 字符串 |
| `GET_PORT_STATUS` | 0x01 | IN | 获取端口状态（1字节）|
| `SOFT_RESET` | 0x02 | OUT | 软件复位打印机 |

**端口状态字节**：
- bit3 = PAPER_EMPTY（缺纸）
- bit4 = SELECT（在线/离线）
- bit5 = NOT_ERROR（无错误）

### 4.4 打印数据流

```
用户 write() → usblp_write()
    → usb_fill_bulk_urb(urb, epwrite...)
    → usb_submit_urb()：向 Bulk OUT 提交打印数据
    → wait_event_interruptible(usblp->wwait, ...)：等待完成

用户 read()（双向模式）→ usblp_read()
    → usb_fill_bulk_urb(urb, epread...)
    → usb_submit_urb()：从 Bulk IN 读取打印机返回数据（如墨水状态）
    → wait_event_interruptible(usblp->rwait, ...)
```

---

## 5. USB 测试测量（usbtmc）

**文件**：`drivers/usb/class/usbtmc.c`（2608 行）
**用户接口**：`/dev/usbtmcN`（N 从 0 开始）
**标准**：USBTMC（USB Test and Measurement Class）Spec Rev 1.0
**USB Class**：Application Specific（0xFE），SubClass 0x03

### 5.1 协议概述

USBTMC 用于连接示波器、万用表、信号发生器等仪器仪表，实现 IEEE 488（GPIB）消息传输。支持 SCPI（Standard Commands for Programmable Instruments）命令集。

**端点结构**：
```
Interface
  ├── Bulk OUT（发送命令/数据到仪器）
  ├── Bulk IN（从仪器接收响应数据）
  └── Interrupt IN（可选，SRQ 服务请求通知）
```

### 5.2 消息格式

USBTMC 使用带消息头的包格式：

**DEV_DEP_MSG_OUT（主机 → 设备，Bulk OUT）**：
```
Offset  大小   字段
0       1      MsgID = 0x01
1       1      bTag（1-255，单调递增）
2       1      bTagInverse（~bTag）
3       1      保留 = 0x00
4       4      TransferSize（数据字节数，LE32）
8       1      bmTransferAttributes（bit0=EOM）
9       3      保留
--- 对齐到 4 字节后 ---
[n]     N      Payload（如 "*IDN?\n"）
```

**DEV_DEP_MSG_IN（设备 → 主机，Bulk IN）**：
```
Offset  大小   字段
0       1      MsgID = 0x02
1       1      bTag
2       1      bTagInverse
3       1      保留
4       4      TransferSize（响应数据字节数）
8       1      bmTransferAttributes（bit0=EOM）
9       3      保留
--- 对齐 ---
[n]     N      Payload（响应字符串，如 "TEKTRONIX,MSO64,...\n"）
```

### 5.3 核心数据结构

```c
// drivers/usb/class/usbtmc.c:73
struct usbtmc_device_data {
    struct usb_device    *usb_dev;
    struct usb_interface *intf;

    unsigned int bulk_in;       /* Bulk IN 管道 */
    unsigned int bulk_out;      /* Bulk OUT 管道 */

    u8  bTag;                   /* 当前消息标签（1-255 循环）*/
    u8  bTag_last_write;        /* 上次写的标签（用于 Abort）*/
    u8  bTag_last_read;         /* 上次读的标签 */
    u16 wMaxPacketSize;         /* Bulk IN 最大包大小 */

    /* Interrupt IN（SRQ 服务请求）*/
    u8             *iin_buffer;
    struct urb     *iin_urb;
    u8              iin_bTag;
    int             iin_ep_present;

    __u8 usb488_caps;           /* USB488 能力位图 */
    struct usbtmc_dev_capabilities capabilities;
    struct kref kref;
    struct mutex io_mutex;       /* 同一时刻只允许一个 I/O */
    wait_queue_head_t waitq;
    struct fasync_struct *fasync; /* 异步通知（SRQ）*/
};
```

### 5.4 USBTMC 特定请求

通过 EP0 发送（`bRequest`）：

| 请求 | 值 | 方向 | 功能 |
|------|----|------|------|
| `INITIATE_ABORT_BULK_OUT` | 0x01 | IN | 中止 Bulk OUT 传输 |
| `CHECK_ABORT_BULK_OUT_STATUS` | 0x02 | IN | 查询中止状态 |
| `INITIATE_ABORT_BULK_IN` | 0x03 | IN | 中止 Bulk IN 传输 |
| `CHECK_ABORT_BULK_IN_STATUS` | 0x04 | IN | 查询中止状态 |
| `INITIATE_CLEAR` | 0x05 | IN | 清除接口（复位）|
| `CHECK_CLEAR_STATUS` | 0x06 | IN | 查询清除状态 |
| `GET_CAPABILITIES` | 0x07 | IN | 获取设备能力 |
| `INDICATOR_PULSE` | 0x40 | IN | 使设备 LED 闪烁（定位）|

---

## 6. USB HID（人机接口）

**文件**：`drivers/hid/usbhid/hid-core.c`（1735 行）
**子驱动**：`usbkbd.c`（398 行）、`usbmouse.c`（232 行）、`hiddev.c`（945 行）
**USB Class**：HID（0x03）

### 6.1 HID 子系统架构

Linux HID 子系统分为两层：

```
输入子系统（input_dev）/ hidraw（/dev/hidrawN）/ hiddev（/dev/usb/hiddevN）
    ↕
HID Core（drivers/hid/hid-core.c）
    → 报告描述符解析 → 报告解码 → 事件分发
    ↕  hid_driver（各厂商特化驱动）
USB HID 传输层（drivers/hid/usbhid/hid-core.c）
    → URB 管理（INT IN / OUT / EP0 CTRL）
    ↕
USB Core
```

`hid_device`（HID 核心抽象）通过 `driver_data` 指向 `usbhid_device`（USB 传输私有数据）。

### 6.2 核心数据结构

```c
// drivers/hid/usbhid/usbhid.h:56
struct usbhid_device {
    struct hid_device *hid;         /* 关联的 HID 抽象设备 */
    struct usb_interface *intf;
    int ifnum;

    unsigned int bufsize;           /* URB 缓冲区大小 */

    /* 输入通道（INT IN）*/
    struct urb *urbin;              /* INT IN URB（接收报告）*/
    char *inbuf;                    /* INT IN 数据缓冲区 */
    dma_addr_t inbuf_dma;

    /* 控制通道（EP0）*/
    struct urb *urbctrl;            /* 控制 URB */
    struct usb_ctrlrequest *cr;     /* 控制请求 */
    struct hid_control_fifo ctrl[HID_CONTROL_FIFO_SIZE]; /* 控制 FIFO */
    unsigned char ctrlhead, ctrltail;
    char *ctrlbuf;
    dma_addr_t ctrlbuf_dma;

    /* 输出通道（INT OUT 或 EP0）*/
    struct urb *urbout;             /* OUT URB（发送报告）*/
    struct hid_output_fifo out[HID_CONTROL_FIFO_SIZE];
    unsigned char outhead, outtail;
    char *outbuf;
    dma_addr_t outbuf_dma;

    /* 同步 */
    struct mutex mutex;
    spinlock_t lock;
    unsigned long iofl;             /* I/O 状态标志位 */
    struct timer_list io_retry;     /* 重试定时器 */
    struct work_struct reset_work;  /* 设备复位工作队列 */
    wait_queue_head_t wait;
};
```

**`iofl` 状态标志**：
```
HID_CTRL_RUNNING(1)   — 控制传输进行中
HID_OUT_RUNNING(2)    — 输出传输进行中
HID_IN_RUNNING(3)     — 输入 URB 已提交（正在等待数据）
HID_RESET_PENDING(4)  — 设备复位待处理
HID_SUSPENDED(5)      — 设备已挂起
HID_IN_POLLING(14)    — 正在轮询输入
```

### 6.3 HID 报告描述符

HID 设备通过报告描述符（Report Descriptor）描述数据格式，由 HID Core 在 probe 时通过 `GET_DESCRIPTOR(HID_DT_REPORT)` 获取并解析：

```
GET_DESCRIPTOR(type=0x22, Report Descriptor)
    → hid_parse_report()：解析报告格式树
        → 每个 Input/Output/Feature 报告的字段格式
        → Usage Page + Usage → 映射到具体输入类型（鼠标移动/按键/etc）
```

**报告描述符示例（标准鼠标）**：
```
Usage Page (Generic Desktop)       // 用途页：通用桌面
Usage (Mouse)                       // 用途：鼠标
Collection (Application)
  Usage (Pointer)
  Collection (Physical)
    Usage Page (Button)
    Usage Minimum (1), Usage Maximum (3)
    Logical Minimum (0), Logical Maximum (1)
    Report Count (3), Report Size (1)
    Input (Data, Variable, Absolute)  // 3个按键（1bit each）
    Report Count (1), Report Size (5)
    Input (Constant)                  // 填充5bits
    Usage Page (Generic Desktop)
    Usage (X), Usage (Y)
    Logical Minimum (-127), Logical Maximum (127)
    Report Size (8), Report Count (2)
    Input (Data, Variable, Relative)  // X/Y 相对移动（各1字节）
  End Collection
End Collection
```

### 6.4 HID 类请求

| 请求 | 值 | 方向 | 功能 |
|------|----|------|------|
| `GET_REPORT` | 0x01 | IN | 获取当前报告（Input/Output/Feature）|
| `GET_IDLE` | 0x02 | IN | 获取输入报告的空闲速率 |
| `GET_PROTOCOL` | 0x03 | IN | 获取协议（Boot=0, Report=1）|
| `SET_REPORT` | 0x09 | OUT | 发送报告（Output/Feature）|
| `SET_IDLE` | 0x0A | OUT | 设置空闲速率（0=无限制，仅报告变化时）|
| `SET_PROTOCOL` | 0x0B | OUT | 切换 Boot/Report 协议 |

**Boot Protocol**（BIOS 兼容）：键盘/鼠标设备在 BIOS 阶段使用固定格式，不依赖报告描述符。

### 6.5 数据流

**接收路径（输入报告）**：

```
INT IN 完成 → usbhid_irq_in(urb)
    → hid_input_report(hid, HID_INPUT_REPORT, data, len, 1)
        → HID Core 报告解析器
            → hid_process_event()：根据报告描述符解码字段
            → hidinput_report_event()：发给 input 子系统
                → input_report_abs() / input_report_key()
                → input_sync()
    → usb_submit_urb(usbhid->urbin)：重提交，等待下一个报告
```

**发送路径（输出报告，如 LED 状态）**：

```
input_led_event() → led_work 工作队列
    → usbhid_submit_report(hid, report, USB_DIR_OUT)
        → 若有 INT OUT 端点：usb_submit_urb(urbout)
        → 否则：usb_submit_urb(urbctrl)  [EP0 SET_REPORT]
```

---

## 7. USB 串口适配层（usb-serial）

**文件**：`drivers/usb/serial/usb-serial.c`（1553 行）+ 各厂商驱动
**用户接口**：`/dev/ttyUSBN`

### 7.1 两级架构

USB 串口子系统由核心层 + 厂商驱动层构成：

```
TTY 子系统 (/dev/ttyUSBN)
    ↕  tty_driver / tty_port
usb-serial core（usb-serial.c）
    → 端点发现、URB 分配、TTY 端口管理
    ↕  usb_serial_driver 钩子
厂商驱动（ftdi_sio.c / cp210x.c / ch341.c / ...）
    → 特定寄存器配置、波特率计算、流控处理
    ↕  usb_driver
USB Core
```

### 7.2 核心数据结构

#### `struct usb_serial`（设备级）

```c
// include/linux/usb/serial.h:141
struct usb_serial {
    struct usb_device       *dev;       /* USB 设备 */
    struct usb_serial_driver *type;     /* 厂商驱动指针 */
    struct usb_interface    *interface; /* USB 接口 */
    struct usb_interface    *sibling;   /* 兄弟接口（多接口设备）*/

    unsigned char num_ports;            /* 该设备的串口数量 */
    unsigned char num_bulk_in;          /* Bulk IN 端点数 */
    unsigned char num_bulk_out;         /* Bulk OUT 端点数 */
    unsigned char num_interrupt_in;     /* INT IN 端点数 */
    unsigned char num_interrupt_out;    /* INT OUT 端点数 */

    struct usb_serial_port *port[MAX_NUM_PORTS]; /* 串口数组 */
    struct kref kref;
    struct mutex disc_mutex;
    void *private;                      /* 厂商驱动私有数据 */
};
```

#### `struct usb_serial_port`（端口级）

```c
// include/linux/usb/serial.h:66
struct usb_serial_port {
    struct usb_serial *serial;
    struct tty_port port;           /* 内嵌 TTY 端口 */
    spinlock_t lock;
    u32 minor;                      /* 次设备号 */
    u8  port_number;                /* 在 serial->port[] 中的索引 */

    /* INT IN（部分设备用于接收状态）*/
    unsigned char *interrupt_in_buffer;
    struct urb    *interrupt_in_urb;
    __u8 interrupt_in_endpointAddress;

    /* Bulk IN（接收数据，支持双缓冲）*/
    unsigned char *bulk_in_buffer;
    struct urb    *read_urb;
    unsigned char *bulk_in_buffers[2];
    struct urb    *read_urbs[2];
    unsigned long  read_urbs_free;   /* 空闲读 URB 位图 */
    __u8 bulk_in_endpointAddress;

    /* Bulk OUT（发送数据，支持双缓冲）*/
    unsigned char *bulk_out_buffer;
    struct urb    *write_urb;
    struct kfifo   write_fifo;       /* 写数据 FIFO */
    unsigned char *bulk_out_buffers[2];
    struct urb    *write_urbs[2];
    unsigned long  write_urbs_free;
    __u8 bulk_out_endpointAddress;

    struct async_icount icount;
    struct work_struct work;
    struct device dev;
};
```

### 7.3 `usb_serial_driver` 钩子接口

```c
// include/linux/usb/serial.h:233
struct usb_serial_driver {
    const char *description;
    const struct usb_device_id *id_table;
    unsigned char num_ports;

    /* 生命周期 */
    int  (*probe)(struct usb_serial *, const struct usb_device_id *);
    int  (*attach)(struct usb_serial *);         /* 接口完整初始化后 */
    int  (*calc_num_ports)(struct usb_serial *, struct usb_serial_endpoints *);
    void (*disconnect)(struct usb_serial *);
    void (*release)(struct usb_serial *);

    /* 端口生命周期 */
    int  (*port_probe)(struct usb_serial_port *);
    void (*port_remove)(struct usb_serial_port *);

    /* TTY 操作（对应 struct tty_operations）*/
    int  (*open)(struct tty_struct *, struct usb_serial_port *);
    void (*close)(struct usb_serial_port *);
    int  (*write)(struct tty_struct *, struct usb_serial_port *,
                  const unsigned char *buf, int count);
    void (*set_termios)(struct tty_struct *, struct usb_serial_port *,
                        const struct ktermios *old);
    int  (*break_ctl)(struct tty_struct *, int break_state);
    int  (*tiocmget)(struct tty_struct *);
    int  (*tiocmset)(struct tty_struct *, unsigned int set, unsigned int clear);

    /* URB 回调（USB 事件处理）*/
    void (*read_bulk_callback)(struct urb *);   /* Bulk IN 完成 */
    void (*write_bulk_callback)(struct urb *);  /* Bulk OUT 完成 */
    void (*read_int_callback)(struct urb *);    /* INT IN 完成 */

    /* 电源管理 */
    int  (*suspend)(struct usb_serial *, pm_message_t);
    int  (*resume)(struct usb_serial *);
};
```

### 7.4 主要厂商驱动

| 驱动文件 | 芯片/用途 | 特点 |
|----------|-----------|------|
| `ftdi_sio.c` | FTDI FT232/FT2232 | 最广泛，波特率可精确计算 |
| `cp210x.c` | Silicon Labs CP2102 | USB 转 UART，物联网常用 |
| `ch341.c` | WinChipHead CH340/341 | 国产低成本 USB 串口芯片 |
| `option.c` | 4G/5G 模组（Sierra/Huawei）| 多端口，AT 命令 + 数据 |
| `pl2303.c` | Prolific PL2303 | 历史悠久，有较多 quirk |
| `f81232.c` | Fintek F81232 | RS-232 串口转换 |
| `ark3116.c` | ArkMicro ARK3116 | USB 转 RS-232 |
| `garmin_gps.c` | Garmin GPS 设备 | 专用协议 |
| `ir-usb.c` | IrDA 红外 USB 适配器 | 红外通信 |

### 7.5 发送/接收调用链

**发送（write → Bulk OUT）**：

```
tty write() → usb_serial_generic_write()
    → kfifo_in_locked(port->write_fifo, buf, count)
    → usb_serial_generic_write_start()
        → 若有空闲 write_urb：
        → usb_serial_generic_prepare_write_buffer()
          [或 driver->prepare_write_buffer()]
        → usb_fill_bulk_urb(urb, bulk_out_endpointAddress...)
        → usb_submit_urb(urb)
    → 完成回调 usb_serial_generic_write_bulk_callback()
        → 清除位图中的忙标志
        → usb_serial_generic_write_start()：继续发送剩余数据
        → tty_port_tty_wakeup()
```

**接收（Bulk IN → tty）**：

```
USB Core → usb_serial_generic_read_bulk_callback()
    [或 driver->read_bulk_callback()]
    → usb_serial_generic_process_read_urb()
      [或 driver->process_read_urb()]
    → tty_insert_flip_string(port, data, count)
    → tty_flip_buffer_push()
    → usb_fill_bulk_urb(urb, ...) + usb_submit_urb()  重新提交
```

---

## 8. USB 大容量存储（usb-storage / UAS）

### 8.1 子系统结构

```
SCSI 子系统（struct scsi_cmnd）
    ↕  struct Scsi_Host / struct scsi_host_template
usb-storage（scsiglue.c）
    → SCSI 命令队列 → ctl_thread 控制线程
    ↕  trans_cmnd 函数指针（BOT / CBI / CB）
传输层（transport.c）
    → URB 打包（CBW）→ 数据阶段 → 状态（CSW）
    ↕
USB Core（urb API）
```

### 8.2 核心数据结构（`struct us_data`）

```c
// drivers/usb/storage/usb.h:87
struct us_data {
    /* 设备信息 */
    struct mutex dev_mutex;
    struct usb_device    *pusb_dev;
    struct usb_interface *pusb_intf;
    const struct us_unusual_dev *unusual_dev; /* quirk 条目 */
    u64  fflags;                              /* 固定 quirk 标志 */

    /* 管道缓存 */
    unsigned int send_bulk_pipe;    /* Bulk OUT 管道 */
    unsigned int recv_bulk_pipe;    /* Bulk IN 管道 */
    unsigned int send_ctrl_pipe;    /* EP0 OUT 管道 */
    unsigned int recv_ctrl_pipe;    /* EP0 IN 管道 */
    unsigned int recv_intr_pipe;    /* INT IN 管道（CBI 使用）*/

    /* 协议信息 */
    char *transport_name;           /* "Bulk-only" / "CB" / "CBI" */
    char *protocol_name;            /* "Transparent SCSI" / "ATAPI" 等 */
    u8   subclass;                  /* USB 子类：SCSI/ATAPI/UFI/RBC 等 */
    u8   protocol;                  /* USB 协议：BOT/CBI/CB */
    u8   max_lun;                   /* 最大 LUN 数（SCSI 逻辑单元号）*/

    /* 函数指针 */
    trans_cmnd   transport;         /* 传输函数（BOT/CB/CBI）*/
    trans_reset  transport_reset;   /* 传输复位函数 */
    proto_cmnd   proto_handler;     /* 协议处理器（SCSI/ATAPI/UFI）*/

    /* SCSI 集成 */
    struct scsi_cmnd  *srb;          /* 当前 SCSI 命令 */
    unsigned int tag;                /* BOT 命令块标签 */

    /* URB 管理 */
    struct urb          *current_urb;
    struct usb_ctrlrequest *cr;
    struct usb_sg_request current_sg;
    unsigned char *iobuf;            /* I/O 缓冲区（CBW/CSW 等）*/
    dma_addr_t iobuf_dma;

    /* 控制线程同步 */
    struct task_struct *ctl_thread;
    struct completion cmnd_ready;
    struct completion notify;
    wait_queue_head_t delay_wait;
};
```

### 8.3 BOT（Bulk-Only Transport）协议

BOT 是最主流的 USB 存储传输协议，用 Bulk OUT/IN 传输 SCSI 命令和数据。

**CBW（Command Block Wrapper，31字节）**：

```c
struct bulk_cb_wrap {
    __le32 Signature;          /* 固定 = 0x43425355 ("USBC") */
    __u32  Tag;                /* 标签，用于匹配 CSW */
    __le32 DataTransferLength; /* 数据阶段期望传输字节数 */
    __u8   Flags;              /* bit7=方向（1=IN，0=OUT）*/
    __u8   Lun;                /* 目标 LUN（0-15）*/
    __u8   Length;             /* CDB 长度（1-16）*/
    __u8   CDB[16];            /* SCSI 命令描述块 */
};
```

**CSW（Command Status Wrapper，13字节）**：

```c
struct bulk_cs_wrap {
    __le32 Signature;    /* 固定 = 0x53425355 ("USBS") */
    __u32  Tag;          /* 对应 CBW 的 Tag */
    __le32 Residue;      /* 未传输的字节数 */
    __u8   Status;       /* 0=Good, 1=Failed, 2=Phase Error */
};
```

**BOT 事务时序**：

```
主机 →[Bulk OUT: CBW(31B)]→ 设备
    ↓  数据阶段（根据 CBW.DataTransferLength 和 Flags 决定方向）
主机 ←→[Bulk IN/OUT: Data]→ 设备（可能为空）
    ↓  状态阶段
主机 ←[Bulk IN: CSW(13B)]← 设备
```

**`usb_stor_Bulk_transport()` 核心流程**（`transport.c:1111`）：

```
1. 构建 CBW（填入 SCSI CDB、LUN、方向、长度）
2. usb_stor_bulk_transfer_buf(us, send_bulk_pipe, bcb, CBW_SIZE)
   → 向 Bulk OUT 发送 31 字节 CBW
3. 数据阶段：
   if (srb->sc_data_direction == DMA_FROM_DEVICE)
       usb_stor_bulk_srb(us, recv_bulk_pipe, srb)  // IN
   else
       usb_stor_bulk_srb(us, send_bulk_pipe, srb)  // OUT
4. usb_stor_bulk_transfer_buf(us, recv_bulk_pipe, bcs, CSW_SIZE)
   → 从 Bulk IN 接收 13 字节 CSW
5. 验证 CSW.Signature、CSW.Tag
6. 根据 CSW.Status 返回结果
```

**BOT 错误恢复**（Phase Error）：

```
Phase Error → Bulk-Only Mass Storage Reset（EP0 bRequest=0xFF）
    → 等待 150ms
    → Clear Halt on Bulk IN / Bulk OUT（CLEAR_FEATURE ENDPOINT_HALT）
```

**BOT 子类与协议对应**（USB 子类字段）：

| SubClass | 命令集 | 设备类型 |
|----------|--------|---------|
| 0x01 | RBC | Flash 存储 |
| 0x02 | ATAPI（MMC-5）| CD/DVD |
| 0x04 | UFI | 软盘（已废弃）|
| 0x05 | SFF-8070i | ATAPI 软盘 |
| 0x06 | SCSI Transparent | 最常见（U 盘/移动硬盘）|

### 8.4 CBI/CB 协议

旧式传输协议，主要用于软盘（已基本淘汰）：

| 协议 | 命令发送 | 状态获取 |
|------|---------|---------|
| CBI（Control/Bulk/Interrupt）| EP0 控制传输 | INT IN 接收状态 |
| CB（Control/Bulk）| EP0 控制传输 | 无状态端点，靠应答包 |

```
CBI 流程：
  EP0 OUT → 发送 SCSI CDB（12字节）
  Bulk OUT/IN → 数据传输
  INT IN → 接收命令完成状态（2字节：ASC/ASCQ）
```

### 8.5 UAS（USB Attached SCSI）

**文件**：`drivers/usb/storage/uas.c`（1304 行）
**特点**：USB 3.0 专用，支持并行命令、流传输，基于 USB Stream

#### 8.5.1 四管道设计

```
端点分配（Pipe Usage Descriptor，USB_DT_PIPE_USAGE）：
  CMD_PIPE_ID   = 1  → Bulk OUT（发送命令 IU）
  STATUS_PIPE_ID = 2 → Bulk IN（接收状态/Sense IU）
  DATA_IN_PIPE_ID = 3 → Bulk IN Stream（数据读取）
  DATA_OUT_PIPE_ID = 4 → Bulk OUT Stream（数据写入）

所有管道使用 USB 3.0 Stream，每个 tag 对应一个 Stream ID。
```

#### 8.5.2 IU（Information Unit）格式

定义于 `include/linux/usb/uas.h`，所有 IU 以 4 字节公共头开始：

```c
struct iu {
    __u8  iu_id;    /* IU 类型 */
    __u8  rsvd1;
    __be16 tag;     /* 命令标签（与 Stream ID 对应）*/
};
```

| IU 类型 | 值 | 方向 | 描述 |
|---------|-----|------|------|
| `IU_ID_COMMAND` | 0x01 | 主机→设备 | SCSI 命令（含 CDB）|
| `IU_ID_STATUS` | 0x03 | 设备→主机 | 命令完成状态（含 Sense）|
| `IU_ID_RESPONSE` | 0x04 | 设备→主机 | 对任务管理命令的响应 |
| `IU_ID_TASK_MGMT` | 0x05 | 主机→设备 | SCSI 任务管理（Abort/Reset）|
| `IU_ID_READ_READY` | 0x06 | 设备→主机 | 设备就绪接收数据（DATA OUT 前）|
| `IU_ID_WRITE_READY` | 0x07 | 设备→主机 | 设备准备发送数据（DATA IN 前）|

**Command IU 结构**（`include/linux/usb/uas.h:46`）：

```c
struct command_iu {
    __u8  iu_id;       /* = 0x01 */
    __u8  rsvd1;
    __be16 tag;        /* 命令标签 */
    __u8  prio_attr;   /* SCSI 命令优先级和排队属性 */
    __u8  rsvd5;
    __u8  len;         /* CDB 长度 */
    __u8  rsvd7;
    struct scsi_lun lun;
    __u8  cdb[16];     /* SCSI Command Descriptor Block */
} __attribute__((__packed__));
```

**Sense IU 结构**（`include/linux/usb/uas.h:72`）：

```c
struct sense_iu {
    __u8  iu_id;       /* = 0x03 */
    __u8  rsvd1;
    __be16 tag;
    __be16 status_qual;
    __u8  status;      /* SCSI 状态字节（GOOD/CHECK_CONDITION 等）*/
    __u8  rsvd7[7];
    __be16 len;        /* Sense 数据长度 */
    __u8  sense[SCSI_SENSE_BUFFERSIZE]; /* SCSI Sense 数据 */
} __attribute__((__packed__));
```

#### 8.5.3 UAS 事务流程

**READ 命令（数据 IN 方向）**：

```
主机 → 设备 CMD Pipe（Stream tag=N）: Command IU（READ(10) CDB）
设备 → 主机 STATUS Pipe（Stream tag=N）: Read Ready IU
主机 → 提交 DATA IN URB（Stream tag=N）
设备 → 主机 DATA IN Pipe（Stream tag=N）: 实际数据
设备 → 主机 STATUS Pipe（Stream tag=N）: Sense IU（完成状态）
```

**多命令并行**（UAS 的核心优势）：

```
主机同时发送 N 个 Command IU（不同 tag）
设备并行处理（SSD 内部并行）
各自通过各自的 Stream 返回数据和状态
乱序完成，通过 tag 对应
```

相较 BOT 的串行执行（必须等待上一命令的 CSW），UAS 可以：
- 消除命令排队延迟（Command Queue Depth 可达 256）
- 数据传输和状态接收并行
- 适合高速 NVMe USB 外壳设备

### 8.6 Quirk 机制

`us_unusual_dev` 表（`unusual_devs.h`）记录需要特殊处理的设备：

```c
/* 示例条目 */
UNUSUAL_DEV(0x090c, 0x6000, 0x0000, 0xffff,
    "Feitian Technologies",
    "ePass Token",
    USB_SC_DEVICE, USB_PR_DEVICE,
    NULL,
    US_FL_IGNORE_DEVICE),  /* 忽略此设备，不作为存储驱动绑定 */
```

常见 Quirk 标志（`US_FL_*`）：

| 标志 | 含义 |
|------|------|
| `US_FL_SINGLE_LUN` | 仅 LUN 0 有效，忽略其他 |
| `US_FL_FIX_INQUIRY` | 修正 INQUIRY 命令响应 |
| `US_FL_NO_WP_DETECT` | 不检测写保护 |
| `US_FL_IGNORE_DEVICE` | 不绑定（让其他驱动处理）|
| `US_FL_NOT_LOCKABLE` | 设备不支持 PREVENT ALLOW MEDIUM REMOVAL |
| `US_FL_CAPACITY_HEURISTICS` | 修正容量计算错误 |
| `US_FL_MAX_SECTORS_64` | 限制最大扇区数为 64 |

---

## 9. Class 驱动注册机制

### 9.1 `usb_driver` 注册

所有 USB Class 驱动最终都注册一个或多个 `usb_driver`：

```c
static struct usb_driver acm_driver = {
    .name          = "cdc_acm",
    .probe         = acm_probe,
    .disconnect    = acm_disconnect,
    .suspend       = acm_suspend,
    .resume        = acm_resume,
    .reset_resume  = acm_reset_resume,
    .id_table      = acm_ids,           /* 支持的 VID/PID/Class 表 */
    .supports_autosuspend = 1,
};

module_usb_driver(acm_driver);
/* 等价于 module_init(acm_init) 调用 usb_register(&acm_driver) */
```

### 9.2 ID 匹配表（`usb_device_id`）

匹配字段优先级（越具体越优先）：

```c
/* 完全匹配：VID + PID */
{ USB_DEVICE(0x0483, 0x5740) }

/* 类匹配：Class + SubClass + Protocol */
{ USB_DEVICE_AND_INTERFACE_INFO(0, USB_CLASS_COMM, USB_CDC_SUBCLASS_ACM, 0) }

/* 接口类匹配 */
{ USB_INTERFACE_INFO(USB_CLASS_COMM, USB_CDC_SUBCLASS_ACM, USB_CDC_ACM_PROTO_AT_V25TER) }

/* 厂商类匹配（某些 4G 模块）*/
{ USB_DEVICE_AND_INTERFACE_INFO(0x12d1, 0x1001, 0xff, 0x01, 0x01) }
```

### 9.3 usb_serial 的双注册机制

USB 串口子系统同时需要注册 `usb_driver` 和 `usb_serial_driver`：

```c
/* 以 FTDI 为例 */
static struct usb_serial_driver ftdi_sio_device = {
    .driver         = { .name = "ftdi_sio" },
    .description    = "FTDI USB Serial Device",
    .id_table       = id_table_combined,
    .num_ports      = 1,
    .probe          = ftdi_sio_probe,
    .open           = ftdi_sio_open,
    .set_termios    = ftdi_set_termios,
    .read_bulk_callback = ftdi_read_bulk_callback,
    ...
};

static struct usb_driver ftdi_driver = {
    .name       = "ftdi_sio",
    .probe      = usb_serial_probe,     /* 通用入口 */
    .disconnect = usb_serial_disconnect,
    .id_table   = id_table_combined,
    .supports_autosuspend = 1,
};

module_init(ftdi_sio_init);  /* 同时调用 usb_serial_register + usb_register */
```

### 9.4 probe 调用栈（以 cdc-acm 为例）

```
插入设备 → hub_irq() → hub_event() → hub_port_connect()
    → usb_new_device() → device_add()
    → usb_probe_interface()             [USB Core]
        → usb_match_one_id_intf()       [检查 id_table]
        → acm_probe(intf, id)           [驱动实现]
            → usb_cdc_get_descriptors() [解析 CDC 描述符]
            → 分配 struct acm
            → 初始化 URB 池
            → tty_port_init()
            → tty_register_device()     → 创建 /dev/ttyACMN
            → usb_submit_urb(ctrlurb)   [启动 INT IN]
```

---

## 10. 跨类驱动对比

### 10.1 各类驱动关键特性对比

| 特性 | cdc-acm | usblp | usbtmc | usbhid | usb-serial | usb-storage |
|------|---------|-------|--------|--------|------------|-------------|
| 用户接口 | TTY | 字符设备 | 字符设备 | input / hidraw | TTY | SCSI/块设备 |
| 内核抽象层 | tty_port | — | — | input_dev | tty_port | scsi_host |
| 控制传输 | CDC 类请求 | 打印机类请求 | USBTMC 类请求 | HID 类请求 | 厂商命令 | BOT Reset |
| 数据端点 | Bulk + INT | Bulk OUT (+IN) | Bulk IN + OUT | INT IN + OUT | Bulk IN/OUT | Bulk IN/OUT |
| 并行命令 | 无 | 无 | 无 | 无 | 无 | UAS 支持 |
| Quirk 机制 | quirks 位标志 | quirks 位 | — | HID quirks | unusual_devs | unusual_devs |
| 多端口 | 单端口 | 单端口 | 单端口 | 单设备 | 多端口(厂商) | 多 LUN |
| 错误恢复 | CLEAR_HALT | Reset | Abort Bulk | Reset | 重置 | BOT Reset |

### 10.2 URB 并发策略对比

| 驱动 | 读 URB 数 | 写 URB 数 | 并发策略 |
|------|----------|----------|---------|
| cdc-acm | ACM_NR（16）个 Bulk IN | ACM_NW（16）个 Bulk OUT | 位图管理空闲 URB |
| usb-serial | 2 个 Bulk IN（双缓冲）| 2 个 Bulk OUT（双缓冲）| 位图管理，FIFO 溢出写入等待 |
| usbhid | 1 个 INT IN | 1 个 INT OUT（可选）| 单 URB，顺序重提交 |
| usb-storage | 1 个 Bulk（单线程）| 1 个 Bulk | ctl_thread 串行执行 |
| UAS | N×4 个（每 tag 各一组）| N×4 个 | Stream 并行，最多 depth=65535 |

### 10.3 内核子系统集成关系

```
                    ┌─────────────────────────────────────┐
                    │           应用层 / 用户空间           │
                    └──┬──────┬──────┬──────┬──────┬──────┘
                       │      │      │      │      │
                   TTY层   块设备   input  字符设备  网络层
                    /dev   /dev/sd  框架   /dev    eth*/usb*
                   /ttyACM  /dev/sg        /dev/lp
                   /ttyUSB               /dev/hidraw
                       │      │      │      │      │
              ┌────────┴──┐ ┌─┴──┐ ┌─┴─┐ ┌┴──┐ ┌─┴──┐
              │ cdc-acm   │ │SCSI│ │HID│ │usb│ │CDC │
              │ usb-serial│ │层  │ │Core│ │lp │ │ECM │
              └────────┬──┘ └─┬──┘ └─┬─┘ └┬──┘ └─┬──┘
                       │      │      │     │      │
              ┌─────────────────────────────────────────┐
              │              USB Core（urb/driver）       │
              └─────────────────────────────────────────┘
```

### 10.4 关键源文件索引

| 文件路径 | 行数 | 职责 |
|----------|------|------|
| `drivers/usb/class/cdc-acm.c` | 2129 | CDC-ACM 主体实现 |
| `drivers/usb/class/cdc-acm.h` | 116 | ACM 数据结构定义 |
| `drivers/usb/class/cdc-wdm.c` | 1378 | CDC-WDM MBIM/QMI 控制 |
| `drivers/usb/class/usblp.c` | 1487 | USB 打印机 |
| `drivers/usb/class/usbtmc.c` | 2608 | USB 测量仪器 |
| `drivers/hid/usbhid/hid-core.c` | 1735 | USB HID 传输层 |
| `drivers/hid/usbhid/usbhid.h` | 97 | usbhid_device 定义 |
| `drivers/hid/usbhid/usbkbd.c` | 398 | USB 键盘（Boot Protocol）|
| `drivers/hid/usbhid/usbmouse.c` | 232 | USB 鼠标（Boot Protocol）|
| `drivers/hid/usbhid/hiddev.c` | 945 | HID 字符设备接口 |
| `drivers/usb/serial/usb-serial.c` | 1553 | USB 串口核心层 |
| `include/linux/usb/serial.h` | ~320 | usb_serial / usb_serial_port / usb_serial_driver |
| `drivers/usb/serial/ftdi_sio.c` | 2877 | FTDI 串口芯片驱动 |
| `drivers/usb/serial/option.c` | 2694 | 4G/5G 模组串口驱动 |
| `drivers/usb/storage/transport.c` | 1462 | BOT/CBI/CB 传输层 |
| `drivers/usb/storage/transport.h` | ~80 | 传输函数声明 |
| `drivers/usb/storage/scsiglue.c` | 699 | SCSI-USB 粘合层 |
| `drivers/usb/storage/usb.c` | 1258 | usb-storage 主入口 |
| `drivers/usb/storage/usb.h` | ~200 | us_data 定义 |
| `drivers/usb/storage/uas.c` | 1304 | UAS 驱动 |
| `include/linux/usb/uas.h` | 110 | UAS IU 结构定义 |
