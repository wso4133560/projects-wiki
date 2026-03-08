# Linux USB 子系统架构详细文档

> 基于 Linux 6.19.6 内核 `drivers/usb/` 源码
> 面向内核学习与驱动开发的工程参考文档

---

## 1. 整体架构

### 1.1 五层分层图

```
┌─────────────────────────────────────────────────┐
│               应用层（用户空间）                 │
│  libusb / usbfs(/dev/bus/usb) / 设备文件节点    │
└──────────────────────┬──────────────────────────┘
                       │ syscall / ioctl
┌──────────────────────▼──────────────────────────┐
│           设备驱动层（USB Class Drivers）         │
│  storage/ · serial/ · class/ · input/ · net/    │
│  各驱动通过 usb_driver 注册，操作 urb 通信        │
└──────────────────────┬──────────────────────────┘
                       │ usb_submit_urb / usb_control_msg
┌──────────────────────▼──────────────────────────┐
│               USB Core（核心层）                  │
│  hub.c · hcd.c · urb.c · driver.c · message.c   │
│  设备枚举、驱动匹配、URB 调度、带宽管理            │
└──────────────────────┬──────────────────────────┘
                       │ hcd->driver->urb_enqueue()
┌──────────────────────▼──────────────────────────┐
│             HCD 抽象层（Host Controller Driver）  │
│  struct usb_hcd / struct hc_driver               │
│  根 Hub 仿真、DMA 映射、URB giveback BH           │
└──────────────────────┬──────────────────────────┘
                       │ MMIO / DMA
┌──────────────────────▼──────────────────────────┐
│                硬件控制器                         │
│          UHCI / OHCI / EHCI / xHCI               │
└─────────────────────────────────────────────────┘
```

### 1.2 主机模式 vs 设备模式（Gadget）双向架构

```
主机模式（Host Mode）                设备模式（Gadget Mode）
─────────────────────                ───────────────────────
应用层 / 类驱动                        FunctionFS / 应用层
    ↕  urb API                             ↕  usb_request
USB Core（hub/driver/urb）           Function Drivers (f_*.c)
    ↕  hc_driver ops                       ↕  usb_ep_ops
HCD（usb_hcd）                        Composite Framework
    ↕  MMIO/DMA                            ↕  UDC Core API
硬件（xHCI/EHCI/...）                  UDC Driver（硬件）
         ↕ USB 总线物理链路 ↕
```

### 1.3 `drivers/usb/` 目录模块总览

| 子目录 | 职责 | 代码规模（行） |
|--------|------|---------------|
| `core/` | 枚举、URB、驱动匹配、HCD 框架 | ~20,000 |
| `host/` | UHCI/OHCI/EHCI/xHCI 控制器驱动 | ~55,000 |
| `gadget/` | Gadget 框架、UDC、功能驱动 | ~35,000 |
| `storage/` | USB 大容量存储（BOT/CBI/UAS） | ~10,000 |
| `serial/` | USB 串口适配层 | ~25,000 |
| `class/` | CDC-ACM/WDM/打印机/测量设备 | ~8,000 |
| `typec/` | USB Type-C 端口管理 | ~15,000 |
| `common/` | OTG FSM、公共工具 | ~1,500 |
| `dwc2/`,`dwc3/` | DesignWare 主机/设备控制器 | ~40,000 |

---

## 2. USB Core 层（`drivers/usb/core/`）

### 2.1 关键文件及职责

| 文件 | 大小 | 职责 |
|------|------|------|
| `hub.c` | 192 KB | 设备枚举、端口管理、热插拔处理 |
| `hcd.c` | 92 KB | HCD 框架、根 Hub 仿真、URB 调度分发 |
| `message.c` | 75 KB | 同步控制/批量消息封装 |
| `devio.c` | 73 KB | usbfs 用户空间接口（/dev/bus/usb） |
| `driver.c` | 60 KB | 设备/接口驱动注册与匹配绑定 |
| `urb.c` | 34 KB | URB 分配/提交/完成/取消生命周期 |
| `config.c` | 34 KB | 配置描述符解析 |

### 2.2 核心数据结构

#### `struct usb_device`（`include/linux/usb.h:658`）

设备在内核中的完整表示：

```c
struct usb_device {
    int          devnum;        /* 总线上唯一设备编号（1-127） */
    char         devpath[16];   /* 拓扑路径字符串，如 "1.2.3" */
    u32          route;         /* xHCI route string（USB 3.0） */
    enum usb_device_state state;/* 当前设备状态机状态 */
    enum usb_device_speed speed;/* LOW/FULL/HIGH/SUPER/SUPER_PLUS */
    unsigned int rx_lanes;      /* USB 3.2 接收通道数 */
    unsigned int tx_lanes;      /* USB 3.2 发送通道数 */

    struct usb_device *parent;  /* 上游 hub，根 hub 为 NULL */
    struct usb_bus   *bus;      /* 所属总线（控制器） */
    struct usb_host_endpoint ep0; /* 控制端点 0 */

    struct device dev;          /* 内嵌 Linux 设备模型节点 */

    struct usb_device_descriptor descriptor; /* 设备描述符 */
    struct usb_host_bos  *bos;             /* BOS 描述符（USB 3.0） */
    struct usb_host_config *config;         /* 所有配置数组 */
    struct usb_host_config *actconfig;      /* 当前激活配置 */
    struct usb_host_endpoint *ep_in[16];    /* IN 端点索引表 */
    struct usb_host_endpoint *ep_out[16];   /* OUT 端点索引表 */

    u32  quirks;                /* 设备特性 Quirk 标志位 */
    atomic_t urbnum;            /* 已提交 URB 计数 */
    int  slot_id;               /* xHCI 设备槽 ID */
};
```

#### `struct usb_interface`（`include/linux/usb.h:243`）

驱动绑定的基本单元，对应 USB 接口：

```c
struct usb_interface {
    struct usb_host_interface *altsetting;     /* 所有 altsetting 数组 */
    struct usb_host_interface *cur_altsetting; /* 当前激活的 altsetting */
    unsigned num_altsetting;                   /* altsetting 数量 */
    int minor;                                 /* 已绑定的 minor 号 */
    enum usb_interface_condition condition;    /* 绑定状态机 */
    struct device dev;                         /* 设备模型节点 */
};
```

#### `struct usb_host_endpoint`（`include/linux/usb.h:67`）

主机侧端点抽象：

```c
struct usb_host_endpoint {
    struct usb_endpoint_descriptor       desc;         /* 端点描述符 */
    struct usb_ss_ep_comp_descriptor     ss_ep_comp;   /* SS 伴随描述符 */
    struct list_head  urb_list;   /* 该端点上排队的 URB 链表 */
    void             *hcpriv;     /* HCD 私有数据（调度信息） */
    int               enabled;    /* 端点是否激活 */
    int               streams;    /* USB 3.0 流通道数 */
};
```

#### `struct urb`（`include/linux/usb.h:1624`）

USB 请求块，USB 通信的核心载体：

```c
struct urb {
    /* 私有区域 —— 仅 USB Core 和 HCD 访问 */
    struct kref    kref;        /* 引用计数 */
    int            unlinked;    /* 取消链接错误码 */
    void          *hcpriv;      /* HCD 私有调度数据 */
    atomic_t       use_count;   /* 并发提交计数 */
    atomic_t       reject;      /* 拒绝新提交标志 */

    /* 公共区域 —— 驱动可读写 */
    struct usb_device         *dev;              /* 目标设备 */
    unsigned int               pipe;             /* 端点管道编码 */
    unsigned int               stream_id;        /* USB 3.0 Stream ID */
    int                        status;           /* 完成状态码 */
    unsigned int               transfer_flags;   /* URB_SHORT_NOT_OK 等 */
    void                      *transfer_buffer;  /* 数据缓冲区 */
    dma_addr_t                 transfer_dma;     /* DMA 地址 */
    struct scatterlist        *sg;               /* SG 列表 */
    u32                        transfer_buffer_length; /* 缓冲区长度 */
    u32                        actual_length;    /* 实际传输长度 */
    unsigned char             *setup_packet;     /* 控制传输 Setup 包 */
    int                        start_frame;      /* ISO 起始帧号 */
    int                        number_of_packets;/* ISO 包数量 */
    int                        interval;         /* INT/ISO 传输间隔 */
    int                        error_count;      /* ISO 错误计数 */
    void                      *context;          /* complete 回调上下文 */
    usb_complete_t             complete;         /* 完成回调函数 */
    struct usb_iso_packet_descriptor iso_frame_desc[]; /* ISO 帧描述符 */
};
```

#### `struct usb_driver`（`include/linux/usb.h:1238`）

USB 接口驱动的注册描述符：

```c
struct usb_driver {
    const char *name;
    int  (*probe)(struct usb_interface *, const struct usb_device_id *);
    void (*disconnect)(struct usb_interface *);
    int  (*suspend)(struct usb_interface *, pm_message_t);
    int  (*resume)(struct usb_interface *);
    int  (*reset_resume)(struct usb_interface *);
    const struct usb_device_id *id_table; /* 支持的设备 ID 表 */
    struct device_driver driver;          /* 内嵌驱动模型节点 */
    unsigned int supports_autosuspend:1;
};
```

#### `struct usb_bus`（`include/linux/usb.h:448`）

总线/控制器的抽象表示：

```c
struct usb_bus {
    struct device *controller;  /* 主机侧硬件设备 */
    int   busnum;               /* 总线编号（注册顺序） */
    struct usb_device *root_hub;/* 根 Hub 虚拟设备 */
    DECLARE_BITMAP(devmap, 128);/* 设备地址分配位图 */
    int   bandwidth_allocated;  /* 已分配周期性带宽（μs/frame） */
    int   bandwidth_int_reqs;   /* 中断传输请求数 */
    int   bandwidth_isoc_reqs;  /* 同步传输请求数 */
};
```

### 2.3 设备状态机

定义于 `include/uapi/linux/usb/ch9.h:1211`：

```
USB_STATE_NOTATTACHED → USB_STATE_ATTACHED → USB_STATE_POWERED
                                                    ↓
USB_STATE_SUSPENDED ← USB_STATE_CONFIGURED ← USB_STATE_ADDRESS
                                                    ↑
                                         USB_STATE_DEFAULT
```

状态说明：
- **NOTATTACHED**：设备未连接（内核扩展状态）
- **ATTACHED**：检测到物理连接
- **POWERED**：总线供电就绪
- **DEFAULT**：复位完成，响应地址 0
- **ADDRESS**：SET_ADDRESS 完成，有唯一总线地址
- **CONFIGURED**：SET_CONFIGURATION 完成，功能完整可用
- **SUSPENDED**：低功耗挂起（可从任意状态进入）

### 2.4 接口绑定状态机

定义于 `include/linux/usb.h:97`：

```
UNBOUND → BINDING → BOUND → UNBINDING → UNBOUND
              ↑ probe()        ↑ disconnect()
```

状态值：`USB_INTERFACE_UNBOUND=0`, `BINDING=1`, `BOUND=2`, `UNBINDING=3`

### 2.5 三大调用链

#### 调用链一：URB 提交/完成

```
usb_submit_urb(urb, GFP_KERNEL)           [urb.c:368]
  → usb_hcd_submit_urb(urb, mem_flags)    [hcd.c:1519]
      → map_urb_for_dma()                 [DMA 映射]
      → hcd->driver->urb_enqueue(hcd, urb, mem_flags)  [HCD 实现]
          → 硬件描述符构建 + 门铃写入/帧列表更新
          → 中断触发：usb_hcd_irq()       [hcd.c:2471]
      → usb_hcd_giveback_urb(hcd, urb, status) [hcd.c:1735]
          → 入队 high_prio_bh 或 low_prio_bh
          → 工作队列执行 → urb->complete(urb)
```

#### 调用链二：设备枚举

```
hub_irq(urb)                              [hub.c:773]
  → 提交状态解析 → schedule hub_event work
hub_event(work)                           [hub.c:5874]
  → hub_port_connect_change()
      → hub_port_connect(hub, port1, ...)  [hub.c:5390]
          → usb_alloc_dev() → usb_new_device(udev) [hub.c:2642]
              → usb_enumerate_device()
                  → hub_set_address()  [SET_ADDRESS 控制传输]
                  → usb_get_descriptor() [GET_DESCRIPTOR]
                  → usb_set_configuration()
              → device_add(&udev->dev)   [触发驱动匹配]
```

#### 调用链三：驱动绑定

```
usb_register_driver(driver)               [driver.c:1045]
  → driver_register(&new_driver->driver) [driver.c:1078]
      → bus_probe_device() → device_attach()
          → usb_probe_interface(dev)      [driver.c:318]
              → usb_match_one_id_intf()   [driver.c:687]
              → driver->probe(intf, id)   [驱动自身 probe]
```

### 2.6 URB 生命周期

```
ALLOCATED (usb_alloc_urb)
    ↓  usb_fill_*_urb() + 字段填充
SUBMITTED (usb_submit_urb)
    ↓  usb_hcd_submit_urb → HCD enqueue
IN-FLIGHT (硬件传输中)
    ↓  完成中断 / usb_kill_urb() 取消
COMPLETED (urb->complete 回调)
    ↓  usb_free_urb() 当 kref 归零
FREED
```

驱动可通过 `usb_kill_urb()` 同步取消，或 `usb_unlink_urb()` 异步取消。

### 2.7 带宽管理

周期性传输（INT/ISO）在 `SET_INTERFACE` 或 `SET_CONFIGURATION` 时分配带宽：

- **Full/Low speed**：帧周期 1 ms，最多保留 90% = 900 μs/frame
- **High speed**：微帧 125 μs，最多保留 80% = 6400 μs/s
- 带宽由 `usb_bus.bandwidth_allocated` 跟踪（microseconds/frame 单位）
- HCD 通过 `hc_driver->add_endpoint()` / `drop_endpoint()` 钩子参与调度计算

### 2.8 关键同步原语

| 同步原语 | 位置 | 保护对象 |
|----------|------|----------|
| `usb_bus.devnum_next_mutex` | `usb_bus` | 设备地址分配 |
| `usb_hcd.bandwidth_mutex` | `usb_hcd` | 带宽变更操作 |
| `usb_hcd.address0_mutex` | `usb_hcd` | SET_ADDRESS 序列化 |
| `giveback_urb_bh.lock` | `giveback_urb_bh` | giveback 队列 |
| `usb_device` 内嵌 `device` lock | `usb_device` | PM 状态变更 |
| hub `mutex` | `usb_hub` | 端口操作序列化 |

---

## 3. HCD 抽象层

### 3.1 `struct usb_hcd`（`include/linux/usb/hcd.h:68`）

```c
struct usb_hcd {
    struct usb_bus      self;         /* HCD 即是总线，继承 usb_bus */
    const struct hc_driver *driver;   /* HW 特定操作钩子表 */
    int                 speed;        /* 根 Hub 速度 */

    struct timer_list   rh_timer;     /* 根 Hub 状态轮询定时器 */
    struct urb         *status_urb;   /* 当前 hub 状态 URB */
    struct work_struct  died_work;    /* 控制器死亡处理 */

    unsigned long       flags;        /* HCD_FLAG_* 原子标志 */
    unsigned int        irq;          /* 已分配的 IRQ 号 */
    void __iomem       *regs;         /* MMIO 寄存器基地址 */

    struct giveback_urb_bh high_prio_bh; /* 高优先级 giveback 工作队列 */
    struct giveback_urb_bh low_prio_bh;  /* 低优先级 giveback 工作队列 */

    struct mutex       *address0_mutex;  /* SET_ADDRESS 互斥锁 */
    struct mutex       *bandwidth_mutex; /* 带宽分配互斥锁 */
    struct usb_hcd     *shared_hcd;      /* 共享硬件的伴生 HCD（xHCI） */
};
```

关键标志位：
- `HCD_FLAG_HW_ACCESSIBLE(0)`：硬件可访问（全功率）
- `HCD_FLAG_POLL_RH(2)`：需要轮询根 Hub 状态
- `HCD_FLAG_DEAD(6)`：控制器已死亡

### 3.2 `struct hc_driver` 操作接口（`include/linux/usb/hcd.h:237`）

| 钩子 | 签名 | 用途 |
|------|------|------|
| `irq` | `irqreturn_t (*irq)(struct usb_hcd *)` | 中断处理入口 |
| `reset` | `int (*reset)(struct usb_hcd *)` | 控制器初始化 |
| `start` | `int (*start)(struct usb_hcd *)` | 启动控制器 |
| `stop` | `void (*stop)(struct usb_hcd *)` | 停止控制器 |
| `urb_enqueue` | `int (*urb_enqueue)(struct usb_hcd *, struct urb *, gfp_t)` | 入队 URB |
| `urb_dequeue` | `int (*urb_dequeue)(struct usb_hcd *, struct urb *, int)` | 取消 URB |
| `endpoint_disable` | `void (*endpoint_disable)(struct usb_hcd *, struct usb_host_endpoint *)` | 端点资源释放 |
| `hub_status_data` | `int (*hub_status_data)(struct usb_hcd *, char *)` | 根 Hub 状态 |
| `hub_control` | `int (*hub_control)(struct usb_hcd *, u16, u16, u16, char *, u16)` | 根 Hub 控制 |
| `add_endpoint` | `int (*add_endpoint)(struct usb_hcd *, struct usb_device *, struct usb_host_endpoint *)` | 带宽预留 |
| `drop_endpoint` | `int (*drop_endpoint)(...)` | 带宽释放 |
| `check_bandwidth` | `int (*check_bandwidth)(...)` | 带宽合法性检查 |
| `alloc_dev` | `int (*alloc_dev)(struct usb_hcd *, struct usb_device *)` | xHCI 设备槽分配 |
| `free_dev` | `void (*free_dev)(...)` | xHCI 设备槽释放 |

### 3.3 根 Hub 仿真机制

USB Core 将控制器端口抽象为虚拟根 Hub 设备（`usb_hcd.self.root_hub`）：

- **虚拟描述符**：Hub 类描述符由 `hub_control()` 软件生成，响应 `GetHubDescriptor` 请求
- **端口状态映射**：`hub_status_data()` 读取硬件端口寄存器，转为标准 Hub 状态字节
- **轮询定时器**：对不支持中断模式的控制器（`uses_new_polling=0`），用 `rh_timer` 定期轮询端口状态
- **中断 URB**：Hub 驱动通过 INT URB 订阅根 Hub 状态变化，完成时触发 `hub_irq`

### 3.4 URB giveback BH 机制

```c
struct giveback_urb_bh {
    bool            running;
    bool            high_prio;
    spinlock_t      lock;
    struct list_head head;         /* 待 giveback 的 URB 链表 */
    struct work_struct bh;         /* 工作队列项 */
    struct usb_host_endpoint *completing_ep;
};
```

`usb_hcd_giveback_urb()` 将 URB 入队 `high_prio_bh` 或 `low_prio_bh`，由工作队列上下文统一调用 `urb->complete()`，避免在中断上下文中执行耗时操作。

### 3.5 中断处理流程

```
硬件中断
  → usb_hcd_irq(irq, __hcd)           [hcd.c:2471]
      → hcd->driver->irq(hcd)         [HCD 特定实现]
          → 读取状态寄存器
          → 处理完成描述符 / 事件环
          → usb_hcd_giveback_urb()     [入队 giveback BH]
      → return IRQ_HANDLED
```

---

## 4. 主机控制器驱动（`drivers/usb/host/`）

### 4.1 四代控制器对比

| 特性 | UHCI | OHCI | EHCI | xHCI |
|------|------|------|------|------|
| USB 版本 | 1.1 | 1.1 | 2.0 | 3.x（兼容 1.x/2.0） |
| 厂商起源 | Intel | Compaq/MS | Intel | Intel |
| 调度模式 | 软件（CPU 遍历） | 硬件 | 混合（硬件异步+软件周期） | 硬件（TRB 环） |
| 传输描述符 | QH + TD | ED + TD | QH + QTD（异步）+ ITD/SITD（ISO） | TRB（Transfer Request Block） |
| 帧/周期结构 | 1024 帧指针列表 | 32 周期 ED 列表 | 1024 微帧指针表 | 环形传输环（Segment Ring） |
| 中断触发 | 完成标志轮询 | Done Queue（WDH） | IAA + STS_INT | 事件环（Event Ring） |
| DMA 池 | td_pool + qh_pool | td_cache + ed_cache | qtd_pool + qh_pool | device_pool + segment_pool |

### 4.2 各控制器关键数据结构

#### UHCI（`drivers/usb/host/uhci-hcd.h:383`）

```c
struct uhci_hcd {
    unsigned long io_addr;          /* I/O 端口基地址 */
    void __iomem *regs;             /* MMIO 寄存器（内存映射时） */
    struct dma_pool *qh_pool;       /* QH DMA 池 */
    struct dma_pool *td_pool;       /* TD DMA 池 */
    struct uhci_qh *skelqh[UHCI_NUM_SKELQH]; /* 骨架 QH 数组（按带宽间隔分层） */
    __hc32 *frame;                  /* 硬件帧列表（1024项，DMA可访问） */
    void  **frame_cpu;              /* CPU 侧帧列表镜像 */
    spinlock_t lock;
    unsigned int frame_number;      /* 当前帧号 */
    short load[MAX_PHASE];          /* 各相位周期性带宽占用 */
};
```

**骨架 QH 设计**：UHCI 用软件维护一组预置的 `skelqh[]`，形成按轮询间隔（1/2/4/.../128 ms）分层的链表树，周期性 QH 插入对应层级，异步 QH 在骨架末尾循环。

#### OHCI（`drivers/usb/host/ohci.h:362`）

```c
struct ohci_hcd {
    spinlock_t lock;
    struct ohci_regs __iomem *regs;
    struct ohci_hcca *hcca;         /* Host Controller Communications Area，硬件共享内存 */
    struct ed *ed_controltail;      /* 控制端点 ED 链表尾 */
    struct ed *ed_bulktail;         /* 批量端点 ED 链表尾 */
    struct ed *periodic[NUM_INTS];  /* 32 个周期性 ED 槽（时间轮） */
    struct dma_pool *td_cache;      /* TD DMA 池 */
    struct dma_pool *ed_cache;      /* ED DMA 池 */
    struct td *td_hash[TD_HASH_SIZE]; /* Done List TD 哈希表 */
    int load[NUM_INTS];             /* 各周期槽带宽负载 */
};
```

**HCCA**：一块 256 字节的 DMA 一致性共享内存，包含 32 项周期性 ED 头指针和 Done Head，是 CPU 与 OHCI 硬件的通信区。

#### EHCI（`drivers/usb/host/ehci.h:111`）

```c
struct ehci_hcd {
    struct hrtimer  hrtimer;        /* 高精度定时器（替代 frame timer） */
    struct ehci_regs __iomem *regs;
    spinlock_t lock;

    /* 异步调度 */
    struct ehci_qh *async;          /* 异步 QH 环形链表头 */
    struct ehci_qh *dummy;          /* 哑 QH（AMD 兼容 quirk） */

    /* 周期性调度 */
    unsigned       periodic_size;   /* 微帧表大小（默认 1024） */
    __hc32        *periodic;        /* 硬件微帧指针表（DMA） */
    dma_addr_t     periodic_dma;
    struct list_head intr_qh_list;  /* INT QH 链表 */

    /* ISO 传输 */
    struct list_head cached_itd_list;  /* 高速 ISO ITD */
    struct list_head cached_sitd_list; /* 全速 ISO SITD */
    unsigned uframe_periodic_max;   /* 每微帧最大周期性时间（μs） */
};
```

**IAA（Interrupt on Async Advance）**：EHCI 异步调度采用 IAA 机制——软件设置 `USBCMD.IAA` 位后，硬件在下一个异步调度推进点触发中断，确认 QH 已安全从链表中移除。

#### xHCI（`drivers/usb/host/xhci.h:1501`）

```c
struct xhci_hcd {
    struct usb_hcd *main_hcd;       /* USB 2.0 根 Hub HCD */
    struct usb_hcd *shared_hcd;     /* USB 3.x 根 Hub HCD */
    struct xhci_cap_regs __iomem *cap_regs;
    struct xhci_op_regs  __iomem *op_regs;
    struct xhci_run_regs __iomem *run_regs;
    struct xhci_doorbell_array __iomem *dba; /* 门铃寄存器阵列 */

    struct xhci_device_context_array *dcbaa; /* 设备上下文基址数组 */
    struct xhci_interrupter **interrupters;  /* 中断器数组（支持 MSI-X） */
    struct xhci_ring *cmd_ring;              /* 命令环 */
    struct xhci_virt_device *devs[MAX_HC_SLOTS]; /* 虚拟设备数组 */

    /* DMA 池 */
    struct dma_pool *device_pool;    /* 设备/端点上下文 */
    struct dma_pool *segment_pool;   /* 环段（TRB 段） */
    struct dma_pool *small_streams_pool;
    struct dma_pool *medium_streams_pool;

    spinlock_t lock;
    unsigned long long quirks;       /* 硬件 Quirk 位图 */
};
```

**xHCI 核心概念**：
- **TRB（Transfer Request Block）**：16 字节的传输描述单元，多个 TRB 串联成环形传输环
- **命令环**：主机向控制器发送命令（Enable Slot/Address Device/Configure Endpoint）
- **事件环**：控制器通知主机传输完成和命令完成
- **门铃寄存器**：驱动写入门铃触发控制器处理对应端点的传输环

### 4.3 传输调度机制

#### 异步传输（Control + Bulk）

| 控制器 | 机制 |
|--------|------|
| UHCI | 骨架 QH 链末尾挂接 Bulk/Control QH，CPU 轮询完成标志 |
| OHCI | `ed_controltail`/`ed_bulktail` 链表，硬件自动遍历，WDH 中断通知 |
| EHCI | `async` 环形 QH 链 + IAA 机制确认卸载；`STS_INT` 标志传输完成 |
| xHCI | 每个端点独立传输环，写门铃触发；事件环返回完成 TRB |

#### 周期性传输（Interrupt + ISO）

| 控制器 | 机制 |
|--------|------|
| UHCI | `skelqh[]` 按 2^n 间隔分层的四叉树，帧列表中多个帧指向同一 QH |
| OHCI | `periodic[32]` 时间轮（32 个 1ms 槽），ED 按间隔选择合适槽 |
| EHCI | `periodic[1024]` 微帧指针表 + hrtimer 管理 split transaction；ITD 用于高速 ISO |
| xHCI | 端点上下文中配置间隔（Interval），硬件自动按微帧调度；无需软件维护周期列表 |

### 4.4 DMA 内存管理

各控制器通过 `dma_pool` 管理硬件描述符的小块一致性内存：

| 控制器 | DMA 池 | 块大小 |
|--------|--------|--------|
| UHCI | `qh_pool` + `td_pool` | sizeof(uhci_qh) / sizeof(uhci_td) |
| OHCI | `td_cache` + `ed_cache` | sizeof(td) / sizeof(ed) |
| EHCI | `qtd_pool` + `qh_pool` | 32B / 96B |
| xHCI | `segment_pool`（TRB 环段）+ `device_pool`（设备上下文） | 4KB / 2KB |

### 4.5 中断处理流程

#### EHCI（`drivers/usb/host/ehci-hcd.c`）

```
ehci_irq()
  → 读 STS 寄存器
  → STS_IAA：异步卸载完成（URB cancel 路径）
  → STS_INT：传输完成 → scan_async() + scan_intr()
  → STS_PCD：端口状态变化 → usb_hcd_poll_rh_status()
  → STS_ERR：传输错误
```

#### OHCI（`drivers/usb/host/ohci-hcd.c`）

```
ohci_irq()
  → 读 INTRSTATUS 寄存器
  → INTRENABLE_WDH：Done Queue Head 更新 → dl_done_list() 处理完成 TD
  → INTRENABLE_SF：帧起始 → 推进周期调度
  → INTRENABLE_RHSC：根 Hub 状态变化
```

#### xHCI（`drivers/usb/host/xhci.c`）

```
xhci_irq()
  → 读 IMAN 寄存器（中断挂起位）
  → xhci_handle_event()：遍历事件环
      → Transfer Event TRB → xhci_handle_tx_event() → giveback URB
      → Command Completion Event → 唤醒命令等待者
      → Port Status Change Event → usb_hcd_poll_rh_status()
```

---

## 5. USB Gadget 框架（`drivers/usb/gadget/`）

### 5.1 分层架构

```
┌─────────────────────────────────────────────┐
│        应用层 / FunctionFS（f_fs.c）          │
│   用户空间通过文件系统接口实现 USB 功能         │
└────────────────────┬────────────────────────┘
                     │
┌────────────────────▼────────────────────────┐
│      功能驱动层 Function Drivers（f_*.c）     │
│  f_mass_storage / f_uac2 / f_hid / f_ncm    │
└────────────────────┬────────────────────────┘
                     │ usb_function 接口
┌────────────────────▼────────────────────────┐
│    Composite 框架（gadget/composite.c）       │
│  管理多功能组合、配置描述符、setup 请求分发    │
└────────────────────┬────────────────────────┘
                     │ usb_gadget / usb_ep 接口
┌────────────────────▼────────────────────────┐
│       UDC Core（gadget/udc/core.c）           │
│    端点抽象、请求队列、事件分发、sysfs 管理    │
└────────────────────┬────────────────────────┘
                     │ usb_ep_ops
┌────────────────────▼────────────────────────┐
│         UDC Driver（硬件相关实现）             │
│  dwc2/dwc3/chipidea/renesas_usb3/...         │
└─────────────────────────────────────────────┘
```

### 5.2 核心数据结构

#### `struct usb_request`（`include/linux/usb/gadget.h:102`）

设备侧传输请求，与主机侧 URB 对应：

```c
struct usb_request {
    void           *buf;         /* 数据缓冲区 */
    unsigned        length;      /* 请求数据长度 */
    dma_addr_t      dma;         /* DMA 地址 */
    struct scatterlist *sg;      /* SG 列表 */
    unsigned        num_sgs;
    unsigned        stream_id:16;/* USB 3.0 Stream ID */
    unsigned        zero:1;      /* 发送零长包（ZLP） */
    unsigned        short_not_ok:1;

    void (*complete)(struct usb_ep *ep, struct usb_request *req);
    void           *context;
    int             status;      /* 完成状态 */
    unsigned        actual;      /* 实际传输字节数 */
};
```

#### `struct usb_ep_ops`（`include/linux/usb/gadget.h:140`）

端点操作接口，由 UDC Driver 实现：

```c
struct usb_ep_ops {
    int (*enable)(struct usb_ep *, const struct usb_endpoint_descriptor *);
    int (*disable)(struct usb_ep *);
    struct usb_request *(*alloc_request)(struct usb_ep *, gfp_t);
    void (*free_request)(struct usb_ep *, struct usb_request *);
    int (*queue)(struct usb_ep *, struct usb_request *, gfp_t); /* 提交请求 */
    int (*dequeue)(struct usb_ep *, struct usb_request *);      /* 取消请求 */
    int (*set_halt)(struct usb_ep *, int value);   /* STALL/CLEAR STALL */
    int (*set_wedge)(struct usb_ep *);
    void (*fifo_flush)(struct usb_ep *);
};
```

### 5.3 ConfigFS 动态配置机制

通过 `gadget/configfs.c` 将 Gadget 配置暴露为 ConfigFS 目录树：

```
/sys/kernel/config/usb_gadget/
  └── g1/
      ├── idVendor, idProduct
      ├── strings/0x409/
      ├── configs/c.1/
      │   ├── MaxPower
      │   └── <function symlink>  →  functions/acm.0/
      └── functions/
          ├── acm.0/
          └── mass_storage.0/
```

写 `UDC` 属性触发 `usb_gadget_probe_driver()`，将 Gadget 绑定到具体 UDC 硬件。

### 5.4 主要功能驱动

| 文件 | 功能 | USB Class |
|------|------|-----------|
| `f_mass_storage.c` | 大容量存储（BOT 协议） | 0x08 |
| `f_hid.c` | HID 设备 | 0x03 |
| `f_uac2.c` | USB 音频 Class 2.0 | 0x01 |
| `f_midi2.c` | USB MIDI 2.0 | 0x01 |
| `f_ncm.c` | 网络控制模型（以太网） | 0x02 |
| `f_rndis.c` | RNDIS（Windows 网络） | 0xEF |
| `f_serial.c` | 串口（Generic Serial） | 0x0A |
| `f_acm.c` | CDC ACM（调制解调器） | 0x02 |
| `f_fs.c` | FunctionFS（用户空间实现） | 任意 |
| `f_printer.c` | 打印机 | 0x07 |
| `f_uvc.c` | USB 视频 Class | 0x0E |

### 5.5 多速度描述符机制

每个功能驱动为不同 USB 速度维护独立的描述符集：

```c
/* 示例：f_hid.c */
static struct usb_descriptor_header *fs_hid_desc[] = { ... }; /* USB 1.1 Full Speed */
static struct usb_descriptor_header *hs_hid_desc[] = { ... }; /* USB 2.0 High Speed */
static struct usb_descriptor_header *ss_hid_desc[] = { ... }; /* USB 3.0 SuperSpeed */
static struct usb_descriptor_header *ssp_hid_desc[]= { ... }; /* USB 3.1 SuperSpeed+ */
```

Composite 框架在 `SET_CONFIGURATION` 时根据当前连接速度选取对应描述符集。

### 5.6 OTG 有限状态机（`drivers/usb/common/usb-otg-fsm.c`）

```
A 设备侧：
A_IDLE → A_WAIT_VRISE → A_WAIT_BCON → A_HOST ←→ A_SUSPEND
              ↑ VBUS 上电                          ↓ HNP
B 设备侧：                               B_PERIPHERAL
B_IDLE → B_SRP_INIT → B_PERIPHERAL ←→ B_WAIT_ACON → B_HOST
```

OTG FSM 通过 `otg_fsm` 结构体维护状态，事件（ID pin / VBUS / HNP）驱动状态转换，控制 VBUS 开关和角色切换（Host ↔ Peripheral）。

---

## 6. 上层设备驱动

### 6.1 USB 大容量存储（`drivers/usb/storage/`）

核心结构 `struct us_data`（`drivers/usb/storage/usb.h:87`）：

```c
struct us_data {
    struct usb_device   *pusb_dev;    /* 目标 USB 设备 */
    struct usb_interface *pusb_intf;  /* 绑定接口 */
    u64    fflags;                    /* 固定功能标志 */
    trans_cmnd   transport;           /* 传输函数指针（BOT/CBI/CB） */
    proto_cmnd   proto_handler;       /* 协议处理器（SCSI/ATAPI/UFI） */
    struct scsi_cmnd *srb;            /* 当前 SCSI 命令 */
    struct urb   *current_urb;        /* 当前传输 URB */
    struct task_struct *ctl_thread;   /* 控制线程（处理 SCSI 命令） */
    struct completion cmnd_ready;     /* 线程同步信号 */
};
```

**传输协议层次**：

| 协议 | 文件 | 说明 |
|------|------|------|
| Bulk-Only Transport (BOT) | `transport.c` | 最主流，CBW + 数据 + CSW |
| Control/Bulk/Interrupt (CBI) | `transport.c` | 旧式软盘 |
| USB Attached SCSI (UAS) | `uas.c` | 并行命令，高性能 |

SCSI 命令由 `usb-storage` 的 SCSI 适配层翻译为 URB，通过 `us_data.transport()` 函数发送。

### 6.2 USB 串行（`drivers/usb/serial/`）

两级结构：

```
struct usb_serial（设备级）
  └── struct usb_serial_port[N]（端口级）
        → tty_port（TTY 层接入点）
```

- `usb_serial` 封装整个 USB 设备，管理端口数组和设备级资源
- `usb_serial_port` 对应每个物理串口，内嵌 `tty_port` 接入 TTY 子系统
- 各厂商驱动（`ftdi_sio.c`/`pl2303.c`/`ch341.c` 等）实现 `usb_serial_driver` 的 `open/close/write/read_bulk_callback` 等钩子

### 6.3 USB 类驱动（`drivers/usb/class/`）

| 驱动 | 文件 | 功能 |
|------|------|------|
| CDC ACM | `cdc-acm.c` | 调制解调器/串口（AT 命令） |
| CDC WDM | `cdc-wdm.c` | 控制设备消息（移动宽带等） |
| 打印机 | `usblp.c` | USB 打印机（/dev/usb/lpN） |
| USB 测量设备 | `usbtmc.c` | USBTMC 协议（仪器仪表） |

---

## 7. 完整数据流路径

### 7.1 主机 OUT 传输路径

```
用户空间 write() / libusb
    ↓
设备驱动 usb_submit_urb(urb)          [drivers/usb/core/urb.c]
    ↓
USB Core usb_hcd_submit_urb()         [drivers/usb/core/hcd.c]
    ↓  DMA 映射 + 入队
HCD urb_enqueue()（如 xhci_urb_enqueue）
    ↓  构建 TRB / QH+TD / ED+TD
写门铃 / 更新帧列表                   [硬件触发传输]
    ↓  USB 总线物理传输
硬件 → 设备（OUT 数据包）
    ↓  完成中断
usb_hcd_giveback_urb()                [入队 giveback BH]
    ↓
urb->complete() 回调                  [设备驱动处理结果]
```

### 7.2 Gadget 设备 IN 传输路径（设备侧）

```
功能驱动 usb_ep_queue(ep, req)        [gadget/udc/core.c]
    ↓  ep->ops->queue()
UDC Driver（如 dwc3_ep_queue）
    ↓  构建硬件传输描述符
硬件等待主机 IN 令牌
    ↓  USB 总线物理传输
主机接收数据
    ↓  完成中断
UDC Driver → req->complete(ep, req)   [功能驱动处理完成]
```

---

## 8. 关键源文件索引

| 文件路径 | 职责 |
|----------|------|
| `drivers/usb/core/hub.c` | 设备枚举、热插拔、端口管理 |
| `drivers/usb/core/hcd.c` | HCD 框架、根 Hub 仿真、URB 分发 |
| `drivers/usb/core/urb.c` | URB 生命周期（alloc/submit/cancel/free） |
| `drivers/usb/core/driver.c` | 驱动注册、接口匹配绑定 |
| `drivers/usb/core/message.c` | 同步控制消息（usb_control_msg 等） |
| `drivers/usb/host/uhci-hcd.c` | UHCI 驱动主体 |
| `drivers/usb/host/uhci-hcd.h` | UHCI 结构体定义（uhci_hcd/qh/td） |
| `drivers/usb/host/ohci-hcd.c` | OHCI 驱动主体 |
| `drivers/usb/host/ohci.h` | OHCI 结构体定义（ohci_hcd/ed/td） |
| `drivers/usb/host/ehci-hcd.c` | EHCI 驱动初始化/IRQ |
| `drivers/usb/host/ehci-sched.c` | EHCI 周期性调度（2491 行） |
| `drivers/usb/host/ehci-q.c` | EHCI 异步 QH/QTD 管理 |
| `drivers/usb/host/ehci.h` | EHCI 结构体定义（ehci_hcd/qh/qtd） |
| `drivers/usb/host/xhci.c` | xHCI 驱动主体（5705 行） |
| `drivers/usb/host/xhci-ring.c` | xHCI 环管理（TRB 入队/事件处理，4487 行） |
| `drivers/usb/host/xhci.h` | xHCI 所有结构体定义 |
| `drivers/usb/gadget/composite.c` | Gadget 组合框架（2802 行） |
| `drivers/usb/gadget/udc/core.c` | UDC 核心抽象层（1949 行） |
| `drivers/usb/gadget/configfs.c` | ConfigFS 动态配置 |
| `drivers/usb/storage/transport.c` | USB 存储传输协议（BOT/CBI） |
| `drivers/usb/storage/usb.h` | us_data 结构体定义 |
| `drivers/usb/common/usb-otg-fsm.c` | OTG 有限状态机 |
| `include/linux/usb.h` | 主机端公共 API（usb_device/urb/usb_driver） |
| `include/linux/usb/hcd.h` | HCD 接口定义（usb_hcd/hc_driver） |
| `include/linux/usb/gadget.h` | Gadget 公共 API（usb_request/usb_ep_ops） |
| `include/uapi/linux/usb/ch9.h` | USB 协议标准定义（描述符/状态枚举） |
