# Linux USB Gadget 框架深度分析

> 基于 Linux 6.19.6 内核源码
> 关键路径：`drivers/usb/gadget/`、`include/linux/usb/gadget.h`、`include/linux/usb/composite.h`

---

## 1. 框架概述

USB Gadget 框架使 Linux 设备能以 **USB 从设备（Peripheral）** 角色与主机通信，例如将开发板模拟为 U 盘、网卡、串口、HID 设备等。

### 1.1 五层架构

```
┌─────────────────────────────────────────────────────────────┐
│          应用层 / FunctionFS (userfaultfs)                   │
│    /dev/usb-ffs/xxx — 用户空间直接操作 USB 端点              │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────┐
│              功能驱动层  Function Drivers                    │
│   function/f_*.c (22个)                                      │
│   f_mass_storage / f_hid / f_uac2 / f_ncm / f_rndis …      │
│   每个驱动实现 struct usb_function 的回调接口                │
└───────────────────────────┬─────────────────────────────────┘
                            │  usb_add_function() / usb_add_config()
┌───────────────────────────▼─────────────────────────────────┐
│          Composite 框架   composite.c                        │
│   多功能/多配置管理、Setup 请求路由、描述符汇总              │
│   struct usb_composite_driver / usb_composite_dev            │
└───────────────────────────┬─────────────────────────────────┘
                            │  usb_gadget_register_driver()
┌───────────────────────────▼─────────────────────────────────┐
│           UDC Core        udc/core.c                         │
│   端点抽象、驱动注册/注销、vbus 管理                         │
│   struct usb_udc / usb_ep / usb_request                      │
└───────────────────────────┬─────────────────────────────────┘
                            │  usb_gadget_ops
┌───────────────────────────▼─────────────────────────────────┐
│           UDC 驱动        udc/dwc2.c / udc/dwc3/  …         │
│   硬件设备控制器（DWC2/DWC3/ChipIdea/musb 等）              │
│   struct usb_gadget（硬件实例）                               │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 目录结构总览

| 目录/文件 | 规模 | 职责 |
|-----------|------|------|
| `composite.c` | ~2700行 | Composite 框架核心，Setup 路由，描述符组装 |
| `configfs.c` | ~1900行 | ConfigFS 动态配置接口，/sys/kernel/config 绑定 |
| `epautoconf.c` | ~230行 | 端点自动分配（usb_ep_autoconfig） |
| `udc/core.c` | ~2100行 | UDC 注册、端点 I/O 包装、vbus 管理 |
| `function/f_*.c` | 22个文件 | 各类功能驱动实现 |
| `function/f_fs.c` | ~3200行 | FunctionFS（用户空间功能驱动桥接） |
| `function/f_mass_storage.c` | ~3300行 | USB 大容量存储（BOT 协议） |
| `legacy/` | 若干文件 | 旧式单功能 Gadget 驱动（zero/serial/printer） |

---

## 2. 核心数据结构

### 2.1 struct usb_gadget — 硬件控制器抽象

> `include/linux/usb/gadget.h:442`

```c
struct usb_gadget {
    struct work_struct      work;        /* sysfs_notify 工作队列 */
    struct usb_udc         *udc;         /* 所属 UDC */
    const struct usb_gadget_ops *ops;    /* 硬件操作接口 */
    struct usb_ep          *ep0;         /* 控制端点 0 */
    struct list_head        ep_list;     /* 全部非控制端点链表 */
    enum usb_device_speed   speed;       /* 当前连接速度 */
    enum usb_device_speed   max_speed;   /* 控制器最高支持速度 */
    enum usb_ssp_rate       ssp_rate;    /* SuperSpeed Plus 速率 */
    enum usb_ssp_rate       max_ssp_rate;
    enum usb_device_state   state;       /* 设备枚举状态 */
    spinlock_t              state_lock;
    bool                    teardown;
    const char             *name;        /* 控制器名称（诊断用） */
    struct device           dev;
    /* OTG 标志位 */
    unsigned                is_otg:1;
    unsigned                is_a_peripheral:1;
    unsigned                b_hnp_enable:1;
    /* Quirk 标志 */
    unsigned                quirk_ep_out_aligned_size:1;
    unsigned                quirk_altset_not_supp:1;
    unsigned                quirk_zlp_not_supp:1;
    unsigned                lpm_capable:1;
    unsigned                wakeup_capable:1;
    unsigned                wakeup_armed:1;
    int                     irq;
};
```

`struct usb_gadget` 是 UDC 驱动向上暴露的硬件抽象。功能驱动只通过 `ops` 和端点接口访问硬件，无需关心底层控制器型号。

### 2.2 struct usb_gadget_ops — 控制器操作接口

> `include/linux/usb/gadget.h:337`

```c
struct usb_gadget_ops {
    int  (*get_frame)(struct usb_gadget *);          /* 读取当前帧号 */
    int  (*wakeup)(struct usb_gadget *);             /* 远程唤醒 */
    int  (*func_wakeup)(struct usb_gadget *, int);   /* 函数级唤醒(USB3) */
    int  (*set_selfpowered)(struct usb_gadget *, int);
    int  (*vbus_session)(struct usb_gadget *, int);  /* VBUS 变化通知 */
    int  (*vbus_draw)(struct usb_gadget *, unsigned); /* 设置电流消耗 */
    int  (*pullup)(struct usb_gadget *, int);        /* 控制 D+ 上拉 */
    int  (*udc_start)(struct usb_gadget *, struct usb_gadget_driver *);
    int  (*udc_stop)(struct usb_gadget *);
    void (*udc_set_speed)(struct usb_gadget *, enum usb_device_speed);
    void (*udc_set_ssp_rate)(struct usb_gadget *, enum usb_ssp_rate);
    void (*udc_async_callbacks)(struct usb_gadget *, bool);
    struct usb_ep *(*match_ep)(struct usb_gadget *,  /* 端点匹配 */
                               struct usb_endpoint_descriptor *,
                               struct usb_ss_ep_comp_descriptor *);
    int  (*check_config)(struct usb_gadget *);
};
```

### 2.3 struct usb_gadget_driver — Gadget 驱动接口

> `include/linux/usb/gadget.h:766`

```c
struct usb_gadget_driver {
    char                    *function;    /* 功能描述字符串 */
    enum usb_device_speed    max_speed;
    int  (*bind)(struct usb_gadget *, struct usb_gadget_driver *);
    void (*unbind)(struct usb_gadget *);
    int  (*setup)(struct usb_gadget *, const struct usb_ctrlrequest *);
    void (*disconnect)(struct usb_gadget *);
    void (*suspend)(struct usb_gadget *);
    void (*resume)(struct usb_gadget *);
    void (*reset)(struct usb_gadget *);
    struct device_driver     driver;
    char                    *udc_name;    /* 绑定特定 UDC（ConfigFS 填写）*/
    bool                     is_bound:1;
};
```

Composite 框架将自身嵌入到 `usb_composite_driver.gadget_driver`，对 UDC Core 呈现为标准 `usb_gadget_driver`，UDC Core 不感知 Composite 层的存在。

### 2.4 struct usb_udc — UDC 注册管理

> `drivers/usb/gadget/udc/core.c:53`

```c
struct usb_udc {
    struct usb_gadget_driver *driver;       /* 当前绑定的 gadget 驱动 */
    struct usb_gadget        *gadget;       /* 硬件控制器实例 */
    struct device             dev;          /* 在 /sys/class/udc/ 下的节点 */
    struct list_head          list;         /* 全局 udc_list 链表 */
    bool                      vbus;         /* 当前 VBUS 状态 */
    bool                      started;      /* UDC 已启动 */
    bool                      allow_connect;
    struct work_struct        vbus_work;    /* VBUS 变化处理工作队列 */
    struct mutex              connect_lock;
};
```

全局 `udc_list` 和 `udc_lock` 串联系统中所有 UDC，`usb_gadget_register_driver()` 在此链表中搜索 `udc_name` 匹配的控制器并绑定。

### 2.5 struct usb_composite_driver — 复合设备驱动

> `include/linux/usb/composite.h:386`

```c
struct usb_composite_driver {
    const char                          *name;
    const struct usb_device_descriptor  *dev;       /* 设备描述符模板 */
    struct usb_gadget_strings          **strings;
    enum usb_device_speed               max_speed;
    unsigned                            needs_serial:1;

    int  (*bind)(struct usb_composite_dev *cdev);   /* 必须实现 */
    int  (*unbind)(struct usb_composite_dev *);
    void (*disconnect)(struct usb_composite_dev *);
    void (*suspend)(struct usb_composite_dev *);
    void (*resume)(struct usb_composite_dev *);

    struct usb_gadget_driver gadget_driver; /* 嵌入，对 UDC 透明 */
};
```

注册入口：`usb_composite_probe(driver)` → 内部调用 `usb_gadget_register_driver(&driver->gadget_driver)`。

### 2.6 struct usb_composite_dev — 运行时复合设备

> `include/linux/usb/composite.h:459`

```c
struct usb_composite_dev {
    struct usb_gadget         *gadget;
    struct usb_request        *req;           /* ep0 控制响应缓冲 */
    struct usb_request        *os_desc_req;   /* MS OS 描述符缓冲 */
    struct usb_configuration  *config;        /* 当前激活配置 */

    /* MS OS String 支持 */
    u8                         qw_sign[14];
    u8                         b_vendor_code;
    struct usb_configuration  *os_desc_config;
    unsigned int               use_os_string:1;

    /* WebUSB 支持 */
    u16                        bcd_webusb_version;
    u8                         b_webusb_vendor_code;
    unsigned int               use_webusb:1;

    /* 内部字段 */
    struct usb_device_descriptor desc;         /* 运行时设备描述符 */
    struct list_head             configs;       /* 全部配置链表 */
    struct usb_composite_driver *driver;
    u8                           next_string_id;
    unsigned                     deactivations; /* 延迟使能计数 */
    int                          delayed_status; /* 延迟状态计数 */
    spinlock_t                   lock;
    unsigned int                 setup_pending:1;
    unsigned int                 os_desc_pending:1;
};
```

### 2.7 struct usb_configuration — 配置对象

> `include/linux/usb/composite.h:303`

```c
struct usb_configuration {
    const char                          *label;
    struct usb_gadget_strings          **strings;
    const struct usb_descriptor_header **descriptors; /* 配置前置描述符 */

    void (*unbind)(struct usb_configuration *);
    int  (*setup)(struct usb_configuration *, const struct usb_ctrlrequest *);

    /* 配置描述符字段 */
    u8   bConfigurationValue;
    u8   iConfiguration;
    u8   bmAttributes;       /* USB_CONFIG_ATT_SELFPOWER / WAKEUP */
    u16  MaxPower;           /* mA，框架自动换算为 bMaxPower */

    struct usb_composite_dev *cdev;

    /* 内部 */
    struct list_head   list;       /* 挂入 cdev->configs */
    struct list_head   functions;  /* 所有 usb_function */
    u8                 next_interface_id;
    unsigned           superspeed:1;
    unsigned           highspeed:1;
    unsigned           fullspeed:1;
    unsigned           superspeed_plus:1;
    struct usb_function *interface[32]; /* 接口号 → 功能驱动映射 */
};
```

`interface[]` 数组实现了接口号到功能驱动的 O(1) 分发——`composite_setup()` 收到针对特定接口的请求时直接通过此数组定位对应的 `usb_function`。

### 2.8 struct usb_function — 功能驱动接口

> `include/linux/usb/composite.h:189`

```c
struct usb_function {
    const char                    *name;
    struct usb_gadget_strings    **strings;

    /* 多速度描述符（NULL 表示该速度不支持） */
    struct usb_descriptor_header **fs_descriptors;  /* USB 1.1 Full Speed  */
    struct usb_descriptor_header **hs_descriptors;  /* USB 2.0 High Speed  */
    struct usb_descriptor_header **ss_descriptors;  /* USB 3.0 Super Speed */
    struct usb_descriptor_header **ssp_descriptors; /* USB 3.1 SS+         */

    struct usb_configuration      *config;
    struct usb_os_desc_table      *os_desc_table;  /* MS OS 描述符 */
    unsigned                       os_desc_n;

    /* 配置管理回调 */
    int  (*bind)(struct usb_configuration *, struct usb_function *);
    void (*unbind)(struct usb_configuration *, struct usb_function *);
    void (*free_func)(struct usb_function *);
    struct module *mod;

    /* 运行时管理回调 */
    int  (*set_alt)(struct usb_function *, unsigned intf, unsigned alt);
    int  (*get_alt)(struct usb_function *, unsigned intf);
    void (*disable)(struct usb_function *);
    int  (*setup)(struct usb_function *, const struct usb_ctrlrequest *);
    bool (*req_match)(struct usb_function *, const struct usb_ctrlrequest *,
                      bool config0);
    void (*suspend)(struct usb_function *);
    void (*resume)(struct usb_function *);

    /* USB 3.0 扩展 */
    int  (*get_status)(struct usb_function *);
    int  (*func_suspend)(struct usb_function *, u8 suspend_opt);
    bool  func_suspended;
    bool  func_wakeup_armed;

    /* 内部 */
    struct list_head    list;           /* 挂入 config->functions */
    DECLARE_BITMAP(endpoints, 32);      /* 已分配端点位图 */
    struct usb_function_instance *fi;   /* 对应的 ConfigFS 实例 */
    unsigned int        bind_deactivated:1;
};
```

#### usb_function 回调语义

| 回调 | 调用时机 | 必须实现 |
|------|----------|---------|
| `bind` | 配置绑定时，分配端点/字符串/描述符 | 推荐 |
| `unbind` | 配置解绑时，释放资源 | 推荐 |
| `set_alt` | SET_INTERFACE 请求，启用/切换替代设定 | **必须** |
| `disable` | 配置切换或主机断连时禁用功能 | **必须** |
| `setup` | 收到针对此功能接口的 Class/Vendor 请求 | 按需 |
| `req_match` | 在 setup 前判断此请求是否由该功能处理 | 按需 |
| `suspend/resume` | 主机挂起/恢复通知 | 可选 |
| `func_suspend` | USB 3.0 函数级挂起（SET_FEATURE FUNCTION_SUSPEND） | 按需 |

### 2.9 struct usb_ep — 端点抽象

> `include/linux/usb/gadget.h:230`

```c
struct usb_ep {
    void                              *driver_data;  /* 功能驱动私有 */
    const char                        *name;         /* "ep1in-bulk" 等 */
    const struct usb_ep_ops           *ops;
    const struct usb_endpoint_descriptor     *desc;
    const struct usb_ss_ep_comp_descriptor   *comp_desc;
    struct list_head                   ep_list;      /* 挂入 gadget->ep_list */
    struct usb_ep_caps                 caps;         /* 端点能力位图 */
    bool                               claimed;      /* 已被功能驱动占用 */
    bool                               enabled;
    unsigned                           mult:2;       /* ISO 传输 mult */
    unsigned                           maxburst:5;   /* SS Burst 大小 */
    u8                                 address;      /* 端点地址 */
    u16                                maxpacket;    /* 当前最大包大小 */
    u16                                maxpacket_limit; /* 硬件上限 */
    u16                                max_streams;  /* SS Streams 数量 */
};
```

### 2.10 struct usb_ep_ops — 端点操作接口

> `include/linux/usb/gadget.h:140`

```c
struct usb_ep_ops {
    int  (*enable)(struct usb_ep *, const struct usb_endpoint_descriptor *);
    int  (*disable)(struct usb_ep *);
    void (*dispose)(struct usb_ep *);

    struct usb_request *(*alloc_request)(struct usb_ep *, gfp_t);
    void (*free_request)(struct usb_ep *, struct usb_request *);

    int  (*queue)(struct usb_ep *, struct usb_request *, gfp_t);
    int  (*dequeue)(struct usb_ep *, struct usb_request *);

    int  (*set_halt)(struct usb_ep *, int value);      /* 0=clear, 1=set */
    int  (*set_wedge)(struct usb_ep *);                /* 永久 STALL */
    int  (*fifo_status)(struct usb_ep *);              /* 查询 FIFO 剩余 */
    void (*fifo_flush)(struct usb_ep *);               /* 清空 FIFO */
};
```

UDC Core 将这些操作包装为带 tracing 和校验的公共 API（`usb_ep_enable()` / `usb_ep_queue()` 等），功能驱动只调用公共 API，不直接调用 `ops->*`。

### 2.11 struct usb_request — I/O 请求

> `include/linux/usb/gadget.h:102`

```c
struct usb_request {
    void            *buf;           /* 数据缓冲区（驱动分配） */
    unsigned         length;        /* 请求长度 */
    dma_addr_t       dma;          /* DMA 地址（UDC 驱动填写） */
    struct scatterlist *sg;         /* Scatter-gather 列表 */
    unsigned         num_sgs;
    unsigned         num_mapped_sgs;

    unsigned         stream_id:16;  /* USB 3.0 Streams */
    unsigned         is_last:1;     /* 最后一个 SG 请求 */
    unsigned         no_interrupt:1;/* 不产生完成中断 */
    unsigned         zero:1;        /* 长度对齐时补 ZLP */
    unsigned         short_not_ok:1;/* 短包视为错误 */
    unsigned         dma_mapped:1;

    void (*complete)(struct usb_ep *, struct usb_request *);
    void            *context;       /* complete 回调私有数据 */
    struct list_head list;          /* UDC 内部队列 */

    unsigned         actual;        /* 实际传输字节数（完成后） */
    int              status;        /* 完成状态：0 / -ESHUTDOWN 等 */
};
```

---

## 3. ConfigFS 动态配置机制

ConfigFS 是 Gadget 框架最重要的用户空间接口，允许在运行时动态组合 USB 功能，无需重新编译内核。

### 3.1 目录结构

```
/sys/kernel/config/usb_gadget/
└── g1/                         ← gadget_info（一个复合设备）
    ├── idVendor                ← 0x1234
    ├── idProduct               ← 0x5678
    ├── bcdDevice
    ├── bcdUSB
    ├── MaxPower
    ├── strings/
    │   └── 0x409/             ← 英语字符串
    │       ├── manufacturer
    │       ├── product
    │       └── serialnumber
    ├── configs/
    │   └── c.1/               ← config_usb_cfg（配置1）
    │       ├── bmAttributes
    │       ├── MaxPower
    │       ├── strings/0x409/
    │       │   └── configuration
    │       └── hid.usb0 → ../../functions/hid.usb0  ← 软链接绑定功能
    ├── functions/
    │   └── hid.usb0/          ← usb_function_instance（功能实例）
    │       ├── protocol
    │       ├── subclass
    │       └── report_length
    ├── os_desc/               ← MS OS 描述符（可选）
    └── UDC                    ← 写入 UDC 名称触发绑定
```

### 3.2 struct gadget_info — ConfigFS 顶层节点

> `drivers/usb/gadget/configfs.c:37`

```c
struct gadget_info {
    struct config_group      group;            /* ConfigFS 根节点 */
    struct config_group      functions_group;  /* functions/ 子树 */
    struct config_group      configs_group;    /* configs/ 子树 */
    struct config_group      strings_group;    /* strings/ 子树 */
    struct config_group      os_desc_group;
    struct config_group      webusb_group;

    struct mutex             lock;
    struct usb_gadget_strings *gstrings[MAX_USB_STRING_LANGS + 1];
    struct list_head          string_list;
    struct list_head          available_func;  /* 已创建的功能实例 */

    struct usb_composite_driver composite;     /* 嵌入的 composite 驱动 */
    struct usb_composite_dev    cdev;          /* 嵌入的复合设备 */
    bool                        use_os_desc;
    char                        b_vendor_code;
    char                        qw_sign[14];
    bool                        use_webusb;
    spinlock_t                  spinlock;
    bool                        unbind;
};
```

`gadget_info` 将 ConfigFS 树、Composite 驱动和复合设备实例三者合一，是 ConfigFS 机制的中心数据结构。

### 3.3 绑定触发流程

写入 `UDC` 属性触发设备绑定：

```c
/* configfs.c:271 */
static ssize_t gadget_dev_desc_UDC_store(struct config_item *item,
                                          const char *page, size_t len)
{
    struct gadget_info *gi = to_gadget_info(item);
    /* ... */
    gi->composite.gadget_driver.udc_name = name;  /* 记录目标 UDC 名 */
    ret = usb_gadget_register_driver(&gi->composite.gadget_driver);
    /* UDC Core 在 udc_list 中查找匹配的 UDC，调用 composite_bind() */
}
```

解绑：写入空字符串 → `unregister_gadget()` → `usb_gadget_unregister_driver()`。

### 3.4 功能类型注册机制

```c
/* composite.h:571 */
struct usb_function_driver {
    const char *name;                /* "hid"/"mass_storage" 等 */
    struct module *mod;
    struct list_head list;
    struct usb_function_instance *(*alloc_inst)(void);  /* 创建实例 */
    struct usb_function          *(*alloc_func)(struct usb_function_instance *);
};

struct usb_function_instance {
    struct config_group group;      /* ConfigFS 子目录 */
    struct list_head    cfs_list;
    struct usb_function_driver *fd;
    int  (*set_inst_name)(struct usb_function_instance *, const char *);
    void (*free_func_inst)(struct usb_function_instance *);
};
```

功能驱动模块初始化时调用 `usb_function_register(driver)` 将自身注册到全局列表；用户在 ConfigFS `functions/` 目录下 `mkdir hid.usb0` 时，框架查找名为 `hid` 的 `usb_function_driver` 并调用 `alloc_inst()` 创建实例。

---

## 4. Composite 框架（composite.c）

### 4.1 composite_bind() — 绑定入口

> `drivers/usb/gadget/composite.c:2527`

```
UDC Core 发现匹配的 gadget_driver
        ↓
composite_bind(gadget, gdriver)
        ↓
   kzalloc(struct usb_composite_dev)
   cdev->gadget = gadget
   set_gadget_data(gadget, cdev)           ← 将 cdev 绑定到 gadget
        ↓
   composite_dev_prepare(composite, cdev)
   ├── 分配 ep0 控制缓冲区（USB_COMP_EP0_BUFSIZ = 4096B）
   └── 初始化字符串 ID 分配器
        ↓
   composite->bind(cdev)                   ← 调用应用层 bind（ConfigFS 模式下为 configfs_composite_bind）
   ├── usb_add_config(cdev, config, config_bind)
   │   └── config_bind(config)
   │       └── usb_add_function(config, func)
   │           ├── func->config = config
   │           ├── list_add_tail(&func->list, &config->functions)
   │           └── func->bind(config, func)  ← 功能驱动分配端点/描述符
   └── 更新设备描述符
        ↓
   composite_os_desc_req_prepare()         ← 若启用 OS 描述符
        ↓
   INFO: "%s ready"
```

### 4.2 composite_setup() — Setup 请求路由

> `drivers/usb/gadget/composite.c:1754`

这是 Gadget 框架最核心的函数，处理所有来自主机的 Setup 事务：

```
UDC 中断 → gadget_driver->setup(gadget, ctrl)
                    ↓
        composite_setup(gadget, ctrl)
                    │
        ┌───────────▼───��────────────────────────────────┐
        │  标准请求 (USB_TYPE_STANDARD)                   │
        │  switch(ctrl->bRequest):                        │
        │                                                 │
        │  GET_DESCRIPTOR:                                │
        │  ├── USB_DT_DEVICE   → 返回设备描述符           │
        │  ├── USB_DT_CONFIG   → config_desc(cdev,w_val) │
        │  ├── USB_DT_STRING   → get_string(cdev,...)    │
        │  ├── USB_DT_BOS      → bos_desc(cdev)          │
        │  └── USB_DT_OTG     → OTG 描述符               │
        │                                                 │
        │  SET_CONFIGURATION   → set_config(cdev,...)    │
        │  ├── 遍历 cdev->configs 找到目标配置            │
        │  ├── disable 当前所有函数                       │
        │  └── reset_config → 逐函数调用 set_alt(0,0)    │
        │                                                 │
        │  SET_INTERFACE       → 按接口号查 config->interface[] │
        │                        → func->set_alt()        │
        │  GET_INTERFACE       → func->get_alt()          │
        │  SET_FEATURE/CLR_FEAT→ 处理 REMOTE_WAKEUP 等   │
        └─────────────────────────────────────────────────┘
                    │
        ┌───────────▼──────────────────────────────┐
        │  unknown: 类/厂商请求                     │
        │  1. 遍历 config->functions                │
        │  2. 调用 func->req_match() 找到处理函数   │
        │  3. 调用 func->setup()                   │
        │  4. 若均不匹配，config->setup()           │
        └──────────────────────────────────────────┘
                    │
        ┌───────────▼────────────┐
        │  value >= 0: 入队响应  │
        │  usb_ep_queue(ep0, req)│
        └────────────────────────┘
```

### 4.3 usb_add_function() — 功能注册

> `drivers/usb/gadget/composite.c:310`

```c
int usb_add_function(struct usb_configuration *config,
                     struct usb_function *function)
{
    function->config = config;
    list_add_tail(&function->list, &config->functions);

    if (function->bind_deactivated)
        usb_function_deactivate(function);  /* 阻止过早枚举 */

    if (function->bind)
        value = function->bind(config, function);

    /* 根据提供的描述符更新配置速度支持位图 */
    if (function->fs_descriptors)  config->fullspeed = true;
    if (function->hs_descriptors)  config->highspeed = true;
    if (function->ss_descriptors)  config->superspeed = true;
    if (function->ssp_descriptors) config->superspeed_plus = true;
}
```

---

## 5. UDC Core（udc/core.c）

### 5.1 端点 I/O API

UDC Core 将 `usb_ep_ops` 包装为带 tracing 和合法性检查的公共接口：

| API | 功能 | 对应 ops 调用 |
|-----|------|--------------|
| `usb_ep_enable(ep)` | 使能端点，应用描述符 | `ops->enable(ep, desc)` |
| `usb_ep_disable(ep)` | 禁用端点 | `ops->disable(ep)` |
| `usb_ep_alloc_request(ep, gfp)` | 分配 I/O 请求 | `ops->alloc_request()` |
| `usb_ep_free_request(ep, req)` | 释放请求 | `ops->free_request()` |
| `usb_ep_queue(ep, req, gfp)` | 提交请求到硬件队列 | `ops->queue()` |
| `usb_ep_dequeue(ep, req)` | 取消已提交请求 | `ops->dequeue()` |
| `usb_ep_set_halt(ep)` | 设置 STALL | `ops->set_halt(ep, 1)` |
| `usb_ep_clear_halt(ep)` | 清除 STALL | `ops->set_halt(ep, 0)` |
| `usb_ep_set_wedge(ep)` | 永久 STALL（忽略 CLEAR_HALT） | `ops->set_wedge()` |
| `usb_ep_fifo_flush(ep)` | 清空 FIFO | `ops->fifo_flush()` |

### 5.2 端点自动分配（epautoconf.c）

功能驱动在 `bind()` 阶段通过 `usb_ep_autoconfig()` 申请端点，无需关心硬件端点编号：

```c
/* epautoconf.c:149 */
struct usb_ep *usb_ep_autoconfig(struct usb_gadget *gadget,
                                  struct usb_endpoint_descriptor *desc)
```

`usb_ep_autoconfig()` 根据描述符中的 `bmAttributes`（端点类型）和 `wMaxPacketSize` 在 `gadget->ep_list` 中搜索第一个满足条件且未被占用（`!ep->claimed`）的端点，填入硬件端点地址并标记 `ep->claimed = true`。

SuperSpeed 版本为 `usb_ep_autoconfig_ss()`，额外匹配 `usb_ss_ep_comp_descriptor`。

`usb_ep_autoconfig_reset(gadget)` 在重新配置前释放所有端点占用。

### 5.3 UDC 注册/注销

```
UDC 驱动 probe():
    usb_initialize_gadget(parent, gadget, release)   /* 初始化 gadget */
    usb_add_gadget(gadget)                            /* 加入全局 udc_list */
    → 若已有 gadget_driver 等待 udc_name 匹配，立即调用 udc_start

UDC 驱动 remove():
    usb_del_gadget(gadget)
    → 若已绑定，先调用 gadget_driver->unbind，再从 udc_list 移除
```

---

## 6. 功能驱动框架

### 6.1 功能驱动列表

Linux 6.19.6 `drivers/usb/gadget/function/` 提供 22 个功能驱动：

| 文件 | 功能名 | 描述 |
|------|--------|------|
| `f_acm.c` | acm | CDC-ACM 串口（AT 命令等） |
| `f_ecm.c` | ecm | CDC-ECM 以太网（标准） |
| `f_eem.c` | eem | CDC-EEM 以太网（嵌入式）|
| `f_ncm.c` | ncm | CDC-NCM 以太网（高性能）|
| `f_rndis.c` | rndis | RNDIS 以太网（Windows 兼容）|
| `f_subset.c` | geth | 简化以太网（CDC Subset）|
| `f_fs.c` | ffs | FunctionFS（用户空间实现功能）|
| `f_hid.c` | hid | HID 设备（键盘/鼠标/自定义）|
| `f_mass_storage.c` | mass_storage | 大容量存储（BOT/UAS）|
| `f_tcm.c` | tcm | USB 附加存储（UAS + SCSI Target）|
| `f_uac1.c` | uac1 | USB Audio Class 1.0 |
| `f_uac1_legacy.c` | uac1_legacy | UAC1 旧式实现 |
| `f_uac2.c` | uac2 | USB Audio Class 2.0 |
| `f_midi.c` | midi | USB MIDI 1.0 |
| `f_midi2.c` | midi2 | USB MIDI 2.0 |
| `f_uvc.c` | uvc | USB Video Class（摄像头）|
| `f_printer.c` | printer | USB 打印机 |
| `f_serial.c` | gser | 通用串口 |
| `f_obex.c` | obex | OBEX 对象交换 |
| `f_phonet.c` | phonet | PhoneNet 协议（Nokia） |
| `f_loopback.c` | loopback | 回环测试 |
| `f_sourcesink.c` | sourcesink | 源/汇测试（性能测试）|

### 6.2 功能驱动开发模式

以 `f_hid.c` 为典型参考，一个功能驱动的最小实现：

```c
/* 1. 定义多速度描述符 */
static struct usb_interface_descriptor hid_interface_desc = { ... };
static struct usb_descriptor_header *hid_fs_descriptors[] = {
    (struct usb_descriptor_header *) &hid_interface_desc,
    (struct usb_descriptor_header *) &hid_desc,
    (struct usb_descriptor_header *) &hid_fs_in_ep_desc,
    NULL,
};
/* 类似定义 hid_hs_descriptors、hid_ss_descriptors */

/* 2. 实现 bind：分配端点和字符串 */
static int hid_bind(struct usb_configuration *c, struct usb_function *f)
{
    struct f_hid *hidg = func_to_hid(f);

    /* 分配 IN 端点（hs 描述符指针） */
    hidg->in_ep = usb_ep_autoconfig(c->cdev->gadget, &hid_fs_in_ep_desc);
    /* 分配字符串 ID */
    ret = usb_interface_id(c, f);
    /* 填写描述符 */
    f->fs_descriptors = hid_fs_descriptors;
    f->hs_descriptors = hid_hs_descriptors;
    f->ss_descriptors = hid_ss_descriptors;
    return 0;
}

/* 3. 实现 set_alt：使能端点 */
static int hid_set_alt(struct usb_function *f, unsigned intf, unsigned alt)
{
    usb_ep_enable(hidg->in_ep);
    usb_ep_queue(hidg->in_ep, req, GFP_ATOMIC);
    return 0;
}

/* 4. 实现 disable：禁用端点 */
static void hid_disable(struct usb_function *f)
{
    usb_ep_disable(hidg->in_ep);
}

/* 5. 注册 usb_function_driver */
static struct usb_function_driver hidg_func_driver = {
    .name       = "hid",
    .alloc_inst = hidg_alloc_inst,  /* 创建 ConfigFS 实例 */
    .alloc_func = hidg_alloc_func,  /* 创建 usb_function */
};
module_usb_composite_driver(hidg_func_driver);
```

### 6.3 FunctionFS（f_fs.c）

FunctionFS 允许用户空间程序通过 `/dev/usb-ffs/xxx/ep0` 等文件直接实现 USB 功能，无需编写内核驱动：

```
内核 FunctionFS 层 (f_fs.c)
        ↑
   /dev/usb-ffs/adb/
   ├── ep0   ← 写描述符、读 Setup 请求
   ├── ep1   ← IN 数据端点
   └── ep2   ← OUT 数据端点
        ↑
用户空间守护进程 (如 adbd)
```

工作流程：
1. 打开 `ep0`，写入功能描述符（接口/端点描述符数组）
2. 写入字符串描述符
3. 内核 FunctionFS 驱动将此功能注册到 Composite 框架
4. 读取 `ep0` 获取 Setup 请求并写回响应
5. 直接读写 `ep1`/`ep2` 进行数据传输

---

## 7. 多速度描述符机制

USB Gadget 框架通过四套描述符数组支持不同总线速度，在枚举时根据实际协商速度选择对应集合：

```
┌────────────────┬──────────────┬───────────────────────────────────────────┐
│ 描述符字段     │ 对应速度     │ 说明                                       │
├────────────────┼──────────────┼───────────────────────────────────────────┤
│ fs_descriptors │ Full Speed   │ USB 1.1，12 Mbps，MPS: Bulk=64B           │
│ hs_descriptors │ High Speed   │ USB 2.0，480 Mbps，MPS: Bulk=512B         │
│ ss_descriptors │ Super Speed  │ USB 3.0，5 Gbps，MPS: Bulk=1024B          │
│ ssp_descriptors│ SS+ (Gen2)   │ USB 3.1，10/20 Gbps，MPS: Bulk=1024B     │
└────────────────┴──────────────┴───────────────────────────────────────────┘
```

**描述符选择时机：**

```c
/* composite.c 中 config_desc() 按当前速度选择描述符 */
switch (gadget->speed) {
case USB_SPEED_SUPER_PLUS:  use ssp_descriptors if available, else ss
case USB_SPEED_SUPER:       use ss_descriptors if available, else hs
case USB_SPEED_HIGH:        use hs_descriptors
default:                    use fs_descriptors
}
```

`usb_add_function()` 在注册时扫描功能提供的描述符集合，自动标记配置的速度支持位（`config->highspeed`、`config->superspeed` 等），主机后续请求 `GET_DESCRIPTOR(OTHER_SPEED_CONFIG)` 时框架自动切换描述符集。

**SS 端点配合描述符（`usb_ss_ep_comp_descriptor`）：**

SuperSpeed 每个端点描述符后必须跟随一个 SS 端点配合描述符，携带 `bMaxBurst`（最大突发包数，0~15）和 `bmAttributes`（Streams 数目、ISO Mult 等），`ss_descriptors` 数组中需同时包含两者。

---

## 8. 关键调用链

### 8.1 Gadget 启动链（ConfigFS 模式）

```
用户: echo "musb-hdrc.0" > /sys/kernel/config/usb_gadget/g1/UDC
        ↓
gadget_dev_desc_UDC_store()                  [configfs.c:271]
        ↓
usb_gadget_register_driver(&gi->composite.gadget_driver)
        ↓
UDC Core: 遍历 udc_list，找到名为 "musb-hdrc.0" 的 UDC
        ↓
udc_bind_to_driver()
  → gadget->ops->udc_start(gadget, driver)   ← 启动 UDC 硬件
  → driver->bind(gadget, driver)             ← 进入 composite_bind()
        ↓
composite_bind()                             [composite.c:2527]
  → composite_dev_prepare()                  ← 分配 ep0 缓冲
  → configfs_composite_bind(cdev)            ← ConfigFS 配置绑定
      ↓
      usb_add_config(cdev, &cfg->c, config_bind)
          ↓
          config_bind(config)
              ↓
              list_for_each_entry(fi, &gi->available_func)
                  usb_add_function(config, fi->f)
                      ↓
                      func->bind(config, func)   ← 各功能驱动分配资源
        ↓
gadget->ops->pullup(gadget, 1)              ← 接通 D+ 上拉，主机可见
        ↓
主机开始枚举（GET_DESCRIPTOR / SET_ADDRESS / SET_CONFIGURATION …）
```

### 8.2 数据发送链（功能驱动 → USB 总线）

```
功能驱动:
    req = usb_ep_alloc_request(ep, GFP_ATOMIC)
    req->buf    = data_buffer
    req->length = data_len
    req->complete = my_complete_handler
    usb_ep_queue(ep, req, GFP_ATOMIC)
        ↓
UDC Core: ep->ops->queue(ep, req, gfp)      ← 调用 UDC 硬件驱动
        ↓
UDC 硬件驱动（如 DWC3）:
    将 req 转换为 TRB（Transfer Request Block）
    写门铃寄存器触发 DMA 传输
        ↓
USB 总线 → 主机接收
        ↓
DMA 完成中断 → UDC 驱动调用 gadget->ops 的完成回调
    → usb_gadget_giveback_request(ep, req)   ← UDC Core 包装
    → req->complete(ep, req)                 ← 功能驱动回调
```

### 8.3 数据接收链（USB 总线 → 功能驱动）

```
功能驱动预先提交 OUT 请求:
    usb_ep_queue(out_ep, recv_req, GFP_ATOMIC)
        ↓
主机发送数据 → UDC 硬件接收
        ↓
DMA 完成 → UDC 驱动调用完成回调
    → recv_req->complete(out_ep, recv_req)
    → recv_req->actual = 实际接收字节数
    → 功能驱动处理数据，重新提交 recv_req
```

### 8.4 Setup 请求处理链

```
主机发送 Setup 包（控制传输阶段0）
        ↓
UDC 硬件中断
        ↓
UDC 驱动调用 gadget_driver->setup(gadget, &ctrl)
        ↓
composite_setup(gadget, ctrl)              [composite.c:1754]
        ↓
┌───────────────────────────────────────────────┐
│  标准请求？                                    │
│  GET_DESCRIPTOR → 汇总描述符，usb_ep_queue    │
│  SET_CONFIGURATION → 切换配置                 │
│  SET_INTERFACE → func->set_alt()              │
├───────────────────────────────────────────────┤
│  类/厂商请求？                                │
│  遍历 functions → func->req_match()?          │
│  → func->setup()                             │
│  或 → config->setup()                        │
└───────────────────────────────────────────────┘
        ↓
value >= 0: usb_ep_queue(cdev->gadget->ep0, cdev->req, GFP_ATOMIC)
        ↓
控制传输数据阶段 / 状态阶段完成
```

### 8.5 Gadget 停止链

```
用户: echo "" > UDC
        ↓
unregister_gadget(gi)
        ↓
usb_gadget_unregister_driver(gdrv)
        ↓
gadget_driver->disconnect(gadget)
        ↓
gadget_driver->unbind(gadget)                ← composite_unbind
    → usb_remove_function(config, func)      ← 每个功能
        func->disable(func)
        func->unbind(config, func)
    → composite_dev_cleanup(cdev)
        ↓
gadget->ops->udc_stop(gadget)
gadget->ops->pullup(gadget, 0)               ← 断开 D+ 上拉
```

---

## 9. OTG 支持

OTG（On-The-Go）允许同一设备既能做主机又能做从设备，通过 ID 引脚和 VBUS 状态动态切换。

### 9.1 角色切换机制

```
Mini-B connector (ID = Hi)  → 作为 B-Peripheral（从设备模式）
Mini-A connector (ID = Lo)  → 作为 A-Host（主机模式）
```

HNP（Host Negotiation Protocol）允许连接后动态协商角色：
- B 设备请求成为主机：发送 `SetFeature(B_HNP_ENABLE)` 给 A 主机
- A 主机同意后挂起总线，B 设备开始驱动总线

SRP（Session Request Protocol）允许 B 设备在总线空闲时请求 A 设备上电：
- B 设备通过 D+ 数据脉冲发出会话请求

### 9.2 Gadget 框架中的 OTG 字段

```c
/* struct usb_gadget 的 OTG 相关字段 */
unsigned is_otg:1;             /* 硬件支持 OTG */
unsigned is_a_peripheral:1;    /* 当前作为 A-Peripheral（HNP 后） */
unsigned b_hnp_enable:1;       /* A 主机已使能 HNP */
unsigned a_hnp_support:1;      /* A 主机在此端口支持 HNP */
unsigned a_alt_hnp_support:1;  /* A 主机在其他端口支持 HNP */
unsigned hnp_polling_support:1;/* 支持 HNP 轮询 */
unsigned host_request_flag:1;  /* 请求成为主机 */
```

功能驱动在 OTG 设备上需提供 OTG 描述符：

```c
static struct usb_otg_descriptor otg_descriptor = {
    .bLength         = sizeof(otg_descriptor),
    .bDescriptorType = USB_DT_OTG,
    .bmAttributes    = USB_OTG_SRP | USB_OTG_HNP,
    .bcdOTG          = cpu_to_le16(0x0200),
};
```

---

## 10. 关键源文件索引

| 文件 | 行数 | 职责 |
|------|------|------|
| `include/linux/usb/gadget.h` | ~900行 | 公共 API：usb_gadget / usb_ep / usb_request / usb_gadget_driver |
| `include/linux/usb/composite.h` | ~600行 | Composite API：usb_function / usb_configuration / usb_composite_driver |
| `drivers/usb/gadget/composite.c` | ~2700行 | Composite 框架核心实现 |
| `drivers/usb/gadget/configfs.c` | ~1900行 | ConfigFS 动态配置 |
| `drivers/usb/gadget/epautoconf.c` | ~230行 | 端点自动分配 |
| `drivers/usb/gadget/udc/core.c` | ~2100行 | UDC Core：端点 I/O 包装、驱动注册 |
| `drivers/usb/gadget/function/f_fs.c` | ~3200行 | FunctionFS（用户空间功能驱动） |
| `drivers/usb/gadget/function/f_mass_storage.c` | ~3300行 | USB 大容量存储（BOT 协议） |
| `drivers/usb/gadget/function/f_hid.c` | ~1000行 | USB HID 功能驱动 |
| `drivers/usb/gadget/function/f_ncm.c` | ~2200行 | CDC-NCM 网络功能驱动 |
| `drivers/usb/gadget/function/f_uac2.c` | ~1700行 | USB Audio Class 2.0 |
| `drivers/usb/gadget/function/f_uvc.c` | ~1400行 | USB Video Class |
| `drivers/usb/gadget/legacy/zero.c` | ~500行 | 参考实现（零设备）|

---

## 11. 设计要点总结

### 分层隔离

- **UDC 驱动**只需实现 `struct usb_gadget_ops` 和 `struct usb_ep_ops`，完全不感知 Composite/Function 层的逻辑
- **Composite 框架**提供多配置/多功能组合、Setup 路由、描述符汇总，不与硬件直接交互
- **功能驱动**只操作 `usb_ep` / `usb_request` 接口，与具体 UDC 硬件解耦

### ConfigFS 的灵活性

传统 Gadget 驱动通过 Kconfig 静态编译固定功能组合；ConfigFS 模式下，功能组合在运行时通过文件系统操作完成，同一内核可动态切换设备角色（如切换 MTP → ADB → RNDIS），Android 和嵌入式系统广泛使用这一机制。

### 多速度描述符的正确性

每个功能驱动必须为每种支持速度提供独立的描述符数组，因为不同速度下端点最大包大小不同（Full Speed Bulk = 64B，High Speed Bulk = 512B，Super Speed Bulk = 1024B），错误配置会导致枚举失败或数据传输异常。

### ep0 缓冲区的使用约束

`cdev->req` 是唯一的 ep0 控制缓冲，所有 Setup 响应共用。`delayed_status` 计数机制允许功能驱动延迟状态阶段响应（典型场景：SET_CONFIGURATION 后需要时间初始化硬件）。
