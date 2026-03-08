# Linux DMA Engine 子系统架构深度分析

> 基于 Linux 6.19.6 内核 `drivers/dma/` 源码
> 面向内核驱动开发的工程参考文档

---

## 1. 整体架构

### 1.1 四层分层图

```
┌──────────────────────────────────────────────────────────┐
│                   消费者层（DMA 使用方）                   │
│  RAID / crypto / audio (ASoC) / SPI / UART / 网络驱动     │
│  async_tx API (lib/raid6/xor) · dmatest · 用户自定义驱动  │
└───────────────────────────┬──────────────────────────────┘
                            │  dmaengine_get/put · dma_request_channel
                            │  dmaengine_prep_* · dmaengine_submit
                            │  dma_async_issue_pending
┌───────────────────────────▼──────────────────────────────┐
│               DMA Engine Core（核心层）                    │
│  dmaengine.c · of-dma.c · acpi-dma.c · virt-dma.c        │
│  设备注册/注销 · 通道分配 · per-CPU 负载均衡 · sysfs/debugfs│
└───────────────────────────┬──────────────────────────────┘
                            │  struct dma_device / hc_driver ops
┌───────────────────────────▼──────────────────────────────┐
│              DMA 控制器驱动（Provider 层）                 │
│  pl330 · dw · dw-axi-dmac · dw-edma · idxd · ioat · ...  │
│  实现 device_prep_* / device_issue_pending / device_tx_status │
└───────────────────────────┬──────────────────────────────┘
                            │  MMIO / PCIe / 中断
┌───────────────────────────▼──────────────────────────────┐
│                  硬件 DMA 控制器                           │
│  ARM PL330 · DesignWare · Intel IOAT · Intel DSA (idxd)   │
└──────────────────────────────────────────────────────────┘
```

### 1.2 `drivers/dma/` 目录模块总览

| 文件/目录 | 代码规模 | 职责 |
|-----------|----------|------|
| `dmaengine.c` | 1635 行 | Core：注册/注销、通道分配、负载均衡、sysfs |
| `virt-dma.c/h` | ~250 行 | 虚拟通道辅助层，大多数驱动依赖 |
| `of-dma.c` | 371 行 | DT（Device Tree）通道查找绑定 |
| `acpi-dma.c` | 462 行 | ACPI 通道查找绑定 |
| `dmatest.c` | 1376 行 | 内核 DMA 测试框架（debugfs 触发） |
| `pl330.c` | 3274 行 | ARM CoreLink DMA-330 驱动（最复杂嵌入式 DMA） |
| `dw/` | ~3000 行 | Synopsys DesignWare DMA（广泛嵌入式应用） |
| `dw-axi-dmac/` | ~2000 行 | DesignWare AXI DMAC（AXI 总线版本） |
| `dw-edma/` | ~2500 行 | DesignWare eDMA（PCIe 内嵌 DMA） |
| `idxd/` | ~8000 行 | Intel Data Streaming Accelerator（DSA/IAX） |
| `ioat/` | ~4000 行 | Intel I/OAT DMA（服务器内存拷贝加速） |
| `stm32/` | ~6000 行 | STM32 系列 DMA/MDMA/BDMA |
| `ti/` | ~5000 行 | TI K3 系列 DMA（UDMA/BCDMA/PKTDMA） |
| `qcom/` | ~3000 行 | Qualcomm BAM/GPI/hidma |

---

## 2. 核心数据结构

### 2.1 `struct dma_device`（`include/linux/dmaengine.h:868`）

DMA 控制器在内核中的完整表示，是 Provider 驱动注册的根对象：

```c
struct dma_device {
    struct kref ref;              /* 引用计数，最后一次 put 时调用 device_release */
    unsigned int chancnt;         /* 通道总数 */
    unsigned int privatecnt;      /* 通过 dma_request_channel 独占的通道数 */
    struct list_head channels;    /* 通道链表（注册后不再修改，无锁遍历） */
    struct list_head global_node; /* 挂入 dma_device_list 全局链表 */
    struct dma_filter filter;     /* 设备/从设备过滤函数映射 */
    dma_cap_mask_t cap_mask;      /* 能力位图（DECLARE_BITMAP，DMA_TX_TYPE_END 位） */

    int dev_id;                   /* 全局唯一设备 ID（ida_alloc 分配） */
    struct device *dev;           /* 对应的底层硬件设备 */
    struct module *owner;         /* 所属模块（自动从 dev->driver->owner 提取） */
    struct ida chan_ida;           /* 通道 ID 分配器 */

    /* 传输能力参数 */
    u32 src_addr_widths;          /* 支持的源地址宽度位图（BIT(1)|BIT(2)|BIT(4)...） */
    u32 dst_addr_widths;          /* 支持的目标地址宽度位图 */
    u32 directions;               /* 支持的传输方向位图 */
    u32 min_burst;                /* 最小 burst 大小 */
    u32 max_burst;                /* 最大 burst 大小 */
    u32 max_sg_burst;             /* 单次 burst 最多 SG 条目（0=无限） */
    bool descriptor_reuse;        /* 是否支持描述符复用（DMA_CTRL_REUSE） */
    enum dma_residue_granularity residue_granularity; /* 剩余量上报粒度 */

    /* === 操作接口（Provider 必须实现的函数指针） === */
    int  (*device_alloc_chan_resources)(struct dma_chan *chan);
    void (*device_free_chan_resources)(struct dma_chan *chan);

    /* 传输准备（返回描述符，失败返回 NULL） */
    struct dma_async_tx_descriptor *(*device_prep_dma_memcpy)(...);
    struct dma_async_tx_descriptor *(*device_prep_dma_xor)(...);
    struct dma_async_tx_descriptor *(*device_prep_dma_pq)(...);
    struct dma_async_tx_descriptor *(*device_prep_dma_memset)(...);
    struct dma_async_tx_descriptor *(*device_prep_slave_sg)(...);
    struct dma_async_tx_descriptor *(*device_prep_dma_cyclic)(...);
    struct dma_async_tx_descriptor *(*device_prep_interleaved_dma)(...);
    struct dma_async_tx_descriptor *(*device_prep_peripheral_dma_vec)(...);

    /* 通道控制 */
    int  (*device_config)(struct dma_chan *, struct dma_slave_config *);
    int  (*device_pause)(struct dma_chan *);
    int  (*device_resume)(struct dma_chan *);
    int  (*device_terminate_all)(struct dma_chan *);
    void (*device_synchronize)(struct dma_chan *);   /* 等待回调执行完成 */

    /* 状态查询与触发 */
    enum dma_status (*device_tx_status)(struct dma_chan *, dma_cookie_t,
                                        struct dma_tx_state *);
    void (*device_issue_pending)(struct dma_chan *); /* 将 pending 队列推入硬件 */
    void (*device_release)(struct dma_device *);

    /* debugfs 支持 */
    void (*dbg_summary_show)(struct seq_file *, struct dma_device *);
    struct dentry *dbg_dev_root;
};
```

### 2.2 `struct dma_chan`（`include/linux/dmaengine.h:338`）

DMA 通道，消费者申请和使用的基本单位：

```c
struct dma_chan {
    struct dma_device *device;          /* 所属控制器 */
    struct device *slave;               /* 绑定的从设备（如 SPI 控制器） */
    dma_cookie_t cookie;                /* 最后分配的 cookie（单调递增） */
    dma_cookie_t completed_cookie;      /* 最后完成的 cookie */

    /* sysfs 节点 /sys/class/dma/dma<N>chan<M>/ */
    int chan_id;
    struct dma_chan_dev *dev;
    const char *name;
    const char *dbg_client_name;        /* debugfs 中的名称，如 "2b00000.mcasp:tx" */

    struct list_head device_node;       /* 挂入 dma_device.channels */
    struct dma_chan_percpu __percpu *local; /* per-CPU 统计（memcpy_count/bytes_transferred） */
    int client_count;                   /* 当前持有引用的消费者数量 */
    int table_count;                    /* 在负载均衡表中的分配次数 */

    struct dma_router *router;          /* DMA 路由器（用于 MUX 场景） */
    void *route_data;                   /* 路由器私有数据 */
    void *private;                      /* 特定消费者-通道关联的私有数据 */
};
```

### 2.3 `struct dma_async_tx_descriptor`（`include/linux/dmaengine.h:614`）

异步传输描述符，`device_prep_*` 函数的返回值：

```c
struct dma_async_tx_descriptor {
    dma_cookie_t cookie;                /* 该传输的唯一标识 */
    enum dma_ctrl_flags flags;          /* DMA_PREP_INTERRUPT / DMA_CTRL_REUSE 等 */
    dma_addr_t phys;                    /* 描述符本身的物理地址（可选） */
    struct dma_chan *chan;               /* 所属通道 */

    dma_cookie_t (*tx_submit)(struct dma_async_tx_descriptor *); /* 提交到 pending 队列 */
    int (*desc_free)(struct dma_async_tx_descriptor *);          /* 释放描述符 */

    dma_async_tx_callback callback;            /* 旧式回调：void (*)(void *) */
    dma_async_tx_callback_result callback_result; /* 新式回调（含结果） */
    void *callback_param;                      /* 回调参数 */

    struct dmaengine_unmap_data *unmap;        /* DMA unmap 信息 */
    enum dma_desc_metadata_mode desc_metadata_mode;
    struct dma_descriptor_metadata_ops *metadata_ops;
};
```

### 2.4 `struct dma_slave_config`（`include/linux/dmaengine.h:447`）

从设备通道运行时配置，通过 `dmaengine_slave_config()` 推送给驱动：

```c
struct dma_slave_config {
    enum dma_transfer_direction direction; /* 方向（已废弃，建议用 prep 函数参数） */
    phys_addr_t src_addr;        /* RX：设备数据寄存器物理地址 */
    phys_addr_t dst_addr;        /* TX：设备数据寄存器物理地址 */
    enum dma_slave_buswidth src_addr_width; /* 源端寄存器宽度（1/2/4/8 字节） */
    enum dma_slave_buswidth dst_addr_width; /* 目标端寄存器宽度 */
    u32 src_maxburst;    /* 单次 burst 最大字（以 addr_width 为单位）*/
    u32 dst_maxburst;
    u32 src_port_window_size;  /* 设备侧寄存器窗口大小（循环写入场景） */
    u32 dst_port_window_size;
    bool device_fc;      /* 是否由外设做流控（flow controller） */
    void *peripheral_config;   /* 外设专用配置（驱动自定义） */
    size_t peripheral_size;
};
```

### 2.5 传输类型枚举（`enum dma_transaction_type`）

```c
enum dma_transaction_type {
    DMA_MEMCPY,        /* 内存到内存拷贝 */
    DMA_XOR,           /* 多源 XOR 运算（RAID-5） */
    DMA_PQ,            /* P+Q 运算（RAID-6，Reed-Solomon） */
    DMA_XOR_VAL,       /* XOR 校验（验证 parity） */
    DMA_PQ_VAL,        /* P+Q 零和校验 */
    DMA_MEMSET,        /* 内存填充 */
    DMA_MEMSET_SG,     /* SG 内存填充 */
    DMA_INTERRUPT,     /* 纯中断（用于链式依赖） */
    DMA_PRIVATE,       /* 独占通道标志（不参与负载均衡表） */
    DMA_ASYNC_TX,      /* async_tx 层自动设置 */
    DMA_SLAVE,         /* 从设备传输（MEM↔DEV） */
    DMA_CYCLIC,        /* 循环传输（音频 DMA） */
    DMA_INTERLEAVE,    /* 交错传输（2D/stride 场景） */
    DMA_COMPLETION_NO_ORDER, /* 乱序完成 */
    DMA_REPEAT,        /* 自动重复传输 */
    DMA_LOAD_EOT,      /* 替换已有 repeat 传输 */
};
```

### 2.6 控制标志枚举（`enum dma_ctrl_flags`）

```c
enum dma_ctrl_flags {
    DMA_PREP_INTERRUPT  = BIT(0), /* 完成后触发中断/回调 */
    DMA_CTRL_ACK        = BIT(1), /* 消费者已确认，描述符可被复用 */
    DMA_PREP_FENCE      = BIT(5), /* 后续操作依赖此传输结果（fence） */
    DMA_CTRL_REUSE      = BIT(6), /* 描述符可被重复提交 */
    DMA_PREP_CMD        = BIT(7), /* 数据是命令格式（非普通 payload） */
    DMA_PREP_REPEAT     = BIT(8), /* 传输完成后自动重复（循环模式） */
    DMA_PREP_LOAD_EOT   = BIT(9), /* 替换当前 repeat 传输 */
};
```

---

## 3. Core 层实现（`drivers/dma/dmaengine.c`）

### 3.1 全局数据结构

```c
static DEFINE_MUTEX(dma_list_mutex);         /* 保护 dma_device_list 和引用计数 */
static DEFINE_IDA(dma_ida);                  /* 全局设备 ID 分配器 */
static LIST_HEAD(dma_device_list);           /* 已注册的所有 dma_device */
static long dmaengine_ref_count;             /* 持有 dmaengine_get() 的消费者计数 */

/* per-CPU 通道负载均衡表：channel_table[tx_type][cpu] → dma_chan */
static struct dma_chan_tbl_ent __percpu *channel_table[DMA_TX_TYPE_END];
```

### 3.2 设备注册流程

```
dma_async_device_register(device)           [dmaengine.c:1171]
  → 参数校验：device->dev 非 NULL
  → CHECK_CAP：声明的能力必须有对应 prep 函数
  → get_dma_id()：IDA 分配 dev_id
  → kref_init(&device->ref)
  → ida_init(&device->chan_ida)
  → list_add_tail_rcu(&device->global_node, &dma_device_list)
  → 遍历 device->channels：
      → __dma_async_device_channel_register(device, chan, NULL)
          → alloc_percpu(dma_chan_percpu) → chan->local
          → ida_alloc(&device->chan_ida) → chan->chan_id
          → device_register(&chan->dev->device)  [sysfs: dma<N>chan<M>]
          → device->chancnt++
  → dma_channel_rebalance()                 [重新分配 per-CPU 表]
  → dmaengine_debug_register(device)        [debugfs 目录]
```

### 3.3 通道分配机制

DMA Engine 提供两种通道获取方式：

#### 方式一：共享通道（mem-to-mem 操作）

```
dmaengine_get()                             [消费者调用，ref_count++]
  → dma_channel_rebalance()
      → 遍历所有 DMA 能力类型和在线 CPU
      → min_chan(cap, cpu)：找本 NUMA 节点上 table_count 最小的通道
      → per_cpu_ptr(channel_table[cap], cpu)->chan = chan

dma_find_channel(tx_type)                   [消费者在热路径调用]
  → return this_cpu_read(channel_table[tx_type]->chan)
  /* 完全无锁，per-CPU 读取 */
```

#### 方式二：独占通道（Slave DMA）

```
dma_request_channel(mask, filter_fn, filter_param)
  → 遍历 dma_device_list（持 dma_list_mutex）
  → dma_device_satisfies_mask()：能力位图匹配
  → filter_fn(chan, filter_param)：驱动自定义过滤
  → dma_chan_get(chan)：
      → try_module_get(owner)
      → kref_get_unless_zero(&device->ref)
      → device_alloc_chan_resources(chan)   [首次使用时分配 HW 资源]
      → chan->client_count++
  → 设置 DMA_PRIVATE 标志（防止进入负载均衡表）
```

### 3.4 Cookie 机制

Cookie 是通道级别的单调递增整数，用于追踪传输完成状态：

```c
/* dmaengine.h */
static inline dma_cookie_t dma_cookie_assign(struct dma_async_tx_descriptor *tx)
{
    cookie = chan->cookie + 1;
    if (cookie < DMA_MIN_COOKIE)  /* DMA_MIN_COOKIE = 1，防止 0 被误判 */
        cookie = DMA_MIN_COOKIE;
    tx->cookie = chan->cookie = cookie;
    return cookie;
}

static inline enum dma_status dma_cookie_status(struct dma_chan *chan,
    dma_cookie_t cookie, struct dma_tx_state *state)
{
    used = chan->cookie;
    complete = chan->completed_cookie;
    barrier();  /* 防止编译器乱序 */
    return dma_async_is_complete(cookie, complete, used);
    /* 判断逻辑：cookie <= complete → COMPLETE
     *           complete < cookie <= used → IN_PROGRESS
     *           otherwise → ERROR  */
}
```

Cookie 溢出处理：`dma_async_is_complete()` 使用模运算比较，支持 `s32` 范围内的翻转。

### 3.5 负载均衡策略

`dma_channel_rebalance()`（在设备注册/注销时触发）：

1. 清空 `channel_table[cap][cpu]`
2. 对每种能力 × 每个在线 CPU，调用 `min_chan(cap, cpu)`
3. `min_chan` 优先选**同 NUMA 节点**的通道，再按 `table_count`（引用次数）选最小值
4. NUMA 亲和性通过 `dev_to_node(chan->device->dev)` 获取

```
NUMA 节点 0：CPU 0-3     → 优先分配 node0 上的 DMA 通道
NUMA 节点 1：CPU 4-7     → 优先分配 node1 上的 DMA 通道
无本地通道时 → 退化为全局最小引用数通道
```

### 3.6 DT/ACPI 通道查找（`of-dma.c` / `acpi-dma.c`）

#### DT 路径（`of-dma.c`）

```
of_dma_request_slave_channel(np, name)
  → of_property_read_u32_index("dmas", ...)  /* 解析 phandle + 参数 */
  → of_dma_find_controller()                 /* 在 of_dma_list 中找对应 Provider */
  → ofdma->of_dma_xlate(dma_spec, ofdma)     /* Provider 实现的通道转换函数 */
  → 返回 dma_chan *
```

Provider 通过 `of_dma_controller_register(np, xlate_fn, data)` 注册，`xlate_fn` 将 DT 参数（通道号、FIFO 地址等）转换为具体 `dma_chan`。

#### DT 节点示例

```dts
/* SPI 控制器使用 DMA */
&spi0 {
    dmas = <&dma0 1>, <&dma0 2>;   /* TX: 通道1, RX: 通道2 */
    dma-names = "tx", "rx";
};

/* DMA 控制器节点 */
dma0: dma-controller@e0000000 {
    compatible = "arm,pl330";
    #dma-cells = <1>;              /* 一个参数：通道请求线号 */
};
```

---

## 4. 虚拟通道层（`virt-dma.c/h`）

### 4.1 设计动机

大多数硬件 DMA 控制器的通道数有限（4～16 个），但上层软件可能需要更多逻辑通道。`virt-dma` 层提供了一套通用的**描述符状态机**和**任务let 完成回调**机制，让 Provider 驱动无需重复实现队列管理。

### 4.2 核心结构

```c
struct virt_dma_desc {
    struct dma_async_tx_descriptor tx;    /* 内嵌基类 */
    struct dmaengine_result tx_result;    /* 完成结果（result code + residue） */
    struct list_head node;                /* 挂入 vc 的某个队列 */
};

struct virt_dma_chan {
    struct dma_chan chan;                  /* 内嵌基类（对外暴露的接口） */
    struct tasklet_struct task;           /* 完成回调的 tasklet */
    void (*desc_free)(struct virt_dma_desc *); /* 驱动提供的描述符释放函数 */
    spinlock_t lock;                      /* 保护以下所有队列 */

    /* 描述符五状态队列 */
    struct list_head desc_allocated;   /* 已分配未提交 */
    struct list_head desc_submitted;   /* 已提交未 issue */
    struct list_head desc_issued;      /* 已 issue 给硬件 */
    struct list_head desc_completed;   /* 硬件完成，等待回调 */
    struct list_head desc_terminated;  /* 已终止，等待释放 */

    struct virt_dma_desc *cyclic;      /* 当前活跃的 cyclic 描述符 */
};
```

### 4.3 描述符状态机

```
driver: vd = kzalloc(...)
                ↓
vchan_tx_prep(vc, vd, flags)
  → list_add_tail(desc_allocated)    ← ALLOCATED
                ↓
tx->tx_submit(tx)  →  vchan_tx_submit()
  → list_move_tail(desc_submitted)   ← SUBMITTED
                ↓
driver: device_issue_pending(chan)
  → vchan_issue_pending(vc)
      → list_splice_tail(submitted → issued)  ← ISSUED
  → 驱动将 desc_issued 链表头推入硬件
                ↓
硬件完成 → 中断 → 驱动 IRQ handler
  → vchan_cookie_complete(vd)
      → dma_cookie_complete(&vd->tx)
      → list_move_tail(desc_completed)       ← COMPLETED
      → tasklet_schedule(&vc->task)
                ↓
tasklet: vchan_vdesc_fini(vd) / callback 调用
  → 若 DMA_CTRL_REUSE：move back to desc_allocated
  → 否则：vc->desc_free(vd)                  ← FREED
```

### 4.4 关键 API

| 函数 | 调用方 | 用途 |
|------|--------|------|
| `vchan_tx_prep(vc, vd, flags)` | Provider | 初始化描述符并入 allocated 队列 |
| `vchan_tx_submit(tx)` | Core（via tx_submit 指针） | 移至 submitted 队列 |
| `vchan_issue_pending(vc)` | Provider（device_issue_pending） | submitted → issued |
| `vchan_cookie_complete(vd)` | Provider（IRQ handler） | 标记完成，调度 tasklet |
| `vchan_cyclic_callback(vd)` | Provider（IRQ handler） | cyclic 传输周期完成 |
| `vchan_terminate_vdesc(vd)` | Provider（device_terminate_all） | 移至 terminated 队列 |
| `vchan_synchronize(vc)` | Provider（device_synchronize） | 杀死 tasklet + 释放 terminated |
| `vchan_free_chan_resources(vc)` | Provider（device_free_chan_resources） | 释放所有队列中的描述符 |

---

## 5. 传输类型详解

### 5.1 Slave DMA（外设 ↔ 内存）

最常用的模式，用于 SPI/I2C/UART/I2S/SAI 等外设数据搬运：

```
/* 消费者（如 SPI 驱动）使用流程 */
chan = dma_request_chan(dev, "tx");          /* 按 DT dma-names 申请 */

config.direction = DMA_MEM_TO_DEV;
config.dst_addr = spi_tx_reg_phys;          /* SPI FIFO 物理地址 */
config.dst_addr_width = DMA_SLAVE_BUSWIDTH_1_BYTE;
config.dst_maxburst = 4;                    /* 4 字节 burst */
dmaengine_slave_config(chan, &config);      /* → device_config() */

tx = dmaengine_prep_slave_sg(chan, sgl, nents,
                              DMA_MEM_TO_DEV, DMA_PREP_INTERRUPT);
tx->callback = spi_dma_tx_done;
tx->callback_param = spi_dev;
cookie = dmaengine_submit(tx);              /* → tx_submit() */
dma_async_issue_pending(chan);              /* → device_issue_pending() */
```

### 5.2 Cyclic DMA（循环传输，音频）

用于 ASoC 音频场景，硬件持续在环形缓冲区中搬运数据，每完成一个 period 触发一次回调：

```
tx = dmaengine_prep_dma_cyclic(chan,
    buf_addr,       /* 环形缓冲区起始物理地址 */
    buf_len,        /* 总缓冲区大小 */
    period_len,     /* 每 period 大小（触发一次回调） */
    DMA_DEV_TO_MEM, /* 方向 */
    DMA_PREP_INTERRUPT);
tx->callback = audio_period_elapsed;
dmaengine_submit(tx);
dma_async_issue_pending(chan);
/* 硬件循环传输，每 period_len 字节触发一次 callback，直到 terminate_all */
```

### 5.3 Interleaved DMA（交错/2D 传输）

用于图像处理、矩阵转置等 stride 访问场景：

```
/* 2D 图像行拷贝：每行 width 字节，行间距 stride，共 height 行 */
struct dma_interleaved_template xt = {
    .src_start = src_buf,
    .dst_start = dst_buf,
    .dir       = DMA_MEM_TO_MEM,
    .src_inc   = true,
    .dst_inc   = true,
    .src_sgl   = true,           /* 源端有间隙（stride） */
    .dst_sgl   = false,          /* 目标端连续 */
    .numf      = height,         /* 帧数 = 行数 */
    .frame_size = 1,             /* 每帧一个 chunk */
    .sgl[0] = {
        .size    = width,        /* 有效数据宽度 */
        .src_icg = stride - width, /* 源端行间隙 */
    },
};
tx = dmaengine_prep_interleaved_dma(chan, &xt, DMA_PREP_INTERRUPT);
```

### 5.4 XOR / PQ（RAID 加速）

```
/* RAID-5：计算多块磁盘的 XOR parity */
tx = chan->device->device_prep_dma_xor(chan,
    dst_dma,          /* parity 目标地址 */
    src_dma_array,    /* 数据块物理地址数组 */
    src_count,        /* 数据块数量（最多 max_xor 个）*/
    len,              /* 每块大小 */
    DMA_PREP_INTERRUPT);

/* RAID-6：计算 P+Q（Reed-Solomon） */
tx = chan->device->device_prep_dma_pq(chan,
    dst_pq,           /* [P, Q] 两个目标地址 */
    src_dma_array,    /* 数据块地址 */
    src_count,
    scf,              /* Galois Field 系数数组 */
    len, flags);
```

---

## 6. 典型 Provider 驱动分析

### 6.1 ARM PL330（`pl330.c`，3274 行）

**PL330（DMA-330）** 是 ARM CoreLink 提供的可编程 DMA 控制器，具有独立的微代码处理器：

**核心特点**：
- 内置 **DMAC 微处理器**：执行存储在内存中的 DMA 程序（指令序列）
- 指令集包含：`MOV`（寄存器赋值）、`LD`/`ST`（内存访问）、`LOOP`（循环）、`WFE`（等待事件）、`SEV`（发送事件）
- 每个通道对应一个独立的**线程状态**（PC、DAR、SAR 等寄存器组）

**驱动架构**：

```c
struct pl330_dmac {
    void __iomem *base;              /* MMIO 基地址 */
    struct pl330_thread *channels;   /* 通道线程数组 */
    struct list_head req_done;       /* 完成请求链表 */
    spinlock_t lock;
};

struct pl330_thread {
    u8 id;                           /* 通道号 */
    int ev;                          /* 分配的事件号 */
    struct pl330_dmac *dmac;
    struct list_head req_work;       /* 待执行请求队列 */
    struct pl330_req *req[2];        /* 双缓冲请求（乒乓） */
    unsigned lstenq:1;               /* 当前活跃的 req buffer 索引 */
};
```

**微程序生成**（`_bursts()`、`_loop()`）：驱动根据传输参数动态生成 DMAC 微代码，写入 `mcode_buf`（内存中的指令缓冲区），然后通过 `DBGINSN`（调试指令槽）启动通道执行。

**双 buffer 乒乓调度**：每个通道同时持有 2 个 `pl330_req`，一个在执行，另一个可被软件准备，实现连续传输无空泡。

### 6.2 Synopsys DesignWare DMA（`dw/`）

**DW DMA** 是工业界应用最广泛的 DMA IP，广泛集成于 Intel Apollo Lake/Baytrail、NXP、Renesas、Altera SoC 等。

**驱动结构**：

```c
struct dw_dma {                      /* dw/ 主结构 */
    struct dma_device dma;           /* 内嵌基类 */
    void __iomem *regs;
    struct clk *clk;
    struct dw_dma_chan *chan;         /* 通道数组 */

    /* 调度工作队列 */
    struct tasklet_struct tasklet;
};

struct dw_dma_chan {
    struct virt_dma_chan vc;         /* 内嵌 virt_dma_chan */
    struct dw_dma *dw;
    void __iomem *ch_regs;           /* 通道寄存器基地址 */
    u8 mask;                         /* 通道掩码 */
    struct dw_desc *active_desc;     /* 当前活跃描述符 */
    struct list_head active_list;    /* LLI（链表项）链表 */
};

struct dw_desc {
    struct virt_dma_desc vd;         /* 内嵌 virt_dma_desc */
    struct list_head desc_list;      /* LLI 子描述符链表 */
    unsigned int descs_allocated;
    /* LLI（Linked List Item）硬件描述符 */
    struct dw_lli lli;
};
```

**LLI（Linked List Item）硬件描述符**：

```c
struct dw_lli {
    __le32 sar;     /* Source Address Register */
    __le32 dar;     /* Destination Address Register */
    __le32 llp;     /* 下一个 LLI 的物理地址（0 = 链表末端） */
    __le32 ctllo;   /* Control Register Low */
    __le32 ctlhi;   /* Control Register High（含块大小） */
    __le32 sstat;   /* Source Status（可选） */
    __le32 dstat;   /* Destination Status（可选） */
};
```

**多 master 支持**：DW DMA 支持多个 AHB master 口（`m_master`/`p_master`），内存通道走 m_master，外设通道走 p_master，通过 `ctllo.SMS`/`ctllo.DMS` 字段选择。

**块传输 vs. scatter-gather**：
- 单块：一个 LLI，`llp = 0`
- SG 传输：多个 LLI 通过 `llp` 链接，硬件自动遍历

### 6.3 Intel IOAT DMA（`ioat/`）

**I/OAT（I/O Acceleration Technology）** 是 Intel 服务器芯片组中的内存拷贝加速引擎，设计目标是卸载网络栈的 memcpy（SKB 拷贝）。

**驱动结构**：

```c
struct ioatdma_device {
    struct dma_device dma_dev;
    void __iomem *reg_base;
    struct pci_dev *pdev;
    struct ioatdma_chan **idx;     /* 通道索引数组 */
    struct dca_provider *dca;     /* Direct Cache Access */
    enum ioat_version version;    /* v1/v2/v3/v3.3/v3.4 */
};

struct ioatdma_chan {
    struct dma_chan dma_chan;
    void __iomem *reg_base;       /* 通道寄存器基地址 */
    struct ioat_ring_ent **ring;  /* 描述符环（软件侧指针数组） */
    u16 head;                     /* 生产者指针 */
    u16 tail;                     /* 消费者指针 */
    u16 issued;                   /* 已推送到硬件的数量 */
    struct timer_list timer;      /* 看门狗定时器 */
};
```

**描述符环设计**：
- 使用**环形队列**（ring buffer）而非链表，大幅减少 DMA 池操作
- `head` → 下一个可写位置，`tail` → 最旧的未完成描述符，`issued` → 已写入 `DMACOUNT` 寄存器的数量
- 硬件通过轮询 `completion_addr` 指针确认完成

**DCA（Direct Cache Access）**：IOAT 支持在 DMA 完成时直接预取数据到 CPU 缓存，降低网络收包延迟。

### 6.4 Intel DSA（idxd，`idxd/`，~8000 行）

**DSA（Data Streaming Accelerator）** 是 Intel Sapphire Rapids+ 引入的新一代数据移动加速器，同系列还包括 **IAX（In-Memory Analytics Accelerator）**。

**核心概念**：

| 概念 | 说明 |
|------|------|
| **WQ（Work Queue）** | 工作队列，可以是 Dedicated（单生产者）或 Shared（多生产者）模式 |
| **Descriptor（工作描述符）** | 64 字节，描述一次 DSA 操作 |
| **Completion Record** | 32 字节，硬件写入操作结果（用户指定地址） |
| **Portal** | MMIO 门户页，应用通过 `movdir64b` 指令提交描述符 |
| **Batch Descriptor** | 批量描述符，一次提交多个操作 |

```c
struct idxd_device {
    struct dma_device dma;
    struct pci_dev *pdev;
    void __iomem *reg_base;
    struct idxd_wq **wqs;          /* 工作队列数组 */
    struct idxd_engine **engines;  /* 引擎数组 */
    struct idxd_group **groups;    /* 引擎组数组 */
    struct idxd_irq_entry *irq_entries; /* MSI-X 中断项 */
};

struct idxd_wq {
    struct dma_chan dma_chan;       /* 对外暴露为 DMA 通道 */
    struct idxd_device *idxd;
    void __iomem *portal;          /* MMIO 提交门户 */
    enum idxd_wq_type type;        /* KERNEL 或 USER */
    u32 wq_id;
    /* 字符设备接口（/dev/dsa/wq<N.M>） */
    struct cdev cdev;
};
```

**用户态直接提交（UACCE/SVA 模式）**：通过 `mmap` portal 页，用户进程可直接用 `MOVDIR64B` 提交工作描述符，完全绕过内核，延迟极低（< 1 μs）。

---

## 7. 传输完整生命周期

### 7.1 Slave DMA 端到端调用链

```
（消费者）dmaengine_prep_slave_sg(chan, sgl, nents, dir, flags)
    → chan->device->device_prep_slave_sg(...)
        [Provider 内部]
        → vd = kzalloc(sizeof(*vd), GFP_NOWAIT)   /* 分配描述符 */
        → 构建 LLI/TRB/微程序 等硬件描述符
        → vchan_tx_prep(&chan->vc, &vd->vd, flags)
            → dma_async_tx_descriptor_init(&vd->tx, ...)
            → list_add_tail(&vd->node, &vc->desc_allocated)
        → return &vd->vd.tx

（消费者）tx->callback = my_callback
（消费者）cookie = dmaengine_submit(tx)
    → tx->tx_submit(tx)
        → vchan_tx_submit(tx)
            → spin_lock_irqsave(&vc->lock)
            → dma_cookie_assign(tx)               /* cookie++ */
            → list_move_tail(&vd->node, &vc->desc_submitted)
            → spin_unlock_irqrestore

（消费者）dma_async_issue_pending(chan)
    → chan->device->device_issue_pending(chan)
        [Provider 内部]
        → spin_lock_irqsave(&vc->lock)
        → vchan_issue_pending(&chan->vc)
            → list_splice_tail_init(submitted → issued)
        → 取 desc_issued 链表头 → 写硬件寄存器触发传输
        → spin_unlock_irqrestore

（硬件）DMA 完成 → 中断
    → [Provider IRQ handler]
        → spin_lock(&vc->lock)
        → vchan_cookie_complete(vd)
            → dma_cookie_complete(&vd->tx)         /* completed_cookie 更新 */
            → list_move_tail(&vd->node, &vc->desc_completed)
            → tasklet_schedule(&vc->task)
        → 推送 desc_issued 下一个描述符到硬件
        → spin_unlock

（tasklet 上下文）vchan_task(data)
    → spin_lock_irqsave(&vc->lock)
    → 遍历 desc_completed 链表
    → spin_unlock_irqrestore
    → dmaengine_desc_get_callback_invoke(tx, &result)
        → tx->callback_result(param, result)  或
        → tx->callback(param)
    → vchan_vdesc_fini(vd)                       /* 释放或复用描述符 */
```

### 7.2 状态查询

```
dma_async_is_tx_complete(chan, cookie, &last, &used)
    → chan->device->device_tx_status(chan, cookie, &state)
        [Provider 实现，通常委托给]
        → dma_cookie_status(chan, cookie, state)
            → 读 chan->cookie（used）和 chan->completed_cookie（last）
            → 无锁，使用 barrier() 确保可见性
            → 返回 DMA_COMPLETE / DMA_IN_PROGRESS / DMA_ERROR

（阻塞等待，生产代码慎用）
dma_sync_wait(chan, cookie)
    → dma_async_issue_pending(chan)
    → 轮询 dma_async_is_tx_complete（最多 5 秒超时）
```

---

## 8. 同步原语与并发模型

### 8.1 锁结构总览

| 锁 | 类型 | 保护对象 |
|----|------|----------|
| `dma_list_mutex` | `mutex` | `dma_device_list`、`dmaengine_ref_count`、通道注册/注销 |
| `vc->lock`（virt-dma） | `spinlock`（irqsave） | 描述符五状态队列、`completed_cookie` |
| Provider 自定义 lock | `spinlock` | 硬件寄存器访问、LLI 链表操作 |
| `channel_table` per-CPU | per-CPU + RCU | 无锁读取（`this_cpu_read`） |

### 8.2 IRQ 上下文安全

```
中断上下文（硬中断）：
  → Provider irq_handler()：持 vc->lock（irqsave），修改描述符队列，调度 tasklet
  → 禁止在硬中断中调用 callback（可能睡眠），只入队 desc_completed

软中断上下文（tasklet）：
  → vchan_task()：不持 vc->lock，调用 callback
  → callback 可以做 DMA mapping、提交新 URB 等

进程上下文：
  → dmaengine_submit / dma_async_issue_pending：持 vc->lock（irqsave）
  → device_config / terminate_all：驱动自行保证与硬中断互斥
```

### 8.3 `device_synchronize` 语义

```
dmaengine_synchronize(chan)
    → chan->device->device_synchronize(chan)
        → vchan_synchronize(vc)
            → tasklet_kill(&vc->task)     /* 等待当前 tasklet 完成，禁止再调度 */
            → 释放 desc_terminated 队列  /* terminate_all 之后清理残留描述符 */
```

**使用场景**：驱动卸载前调用 `terminate_all` + `synchronize`，确保所有回调执行完毕，防止 use-after-free。

---

## 9. sysfs / debugfs 接口

### 9.1 sysfs（`/sys/class/dma/`）

每个 DMA 通道暴露一个 class device：

```
/sys/class/dma/
  dma0chan0/
    ├── memcpy_count         # 该通道执行的 memcpy 次数（per-CPU 累加）
    ├── bytes_transferred    # 累计传输字节数
    └── in_use               # 当前引用计数（client_count）
```

### 9.2 debugfs（`/sys/kernel/debug/dmaengine/`）

```
/sys/kernel/debug/dmaengine/
  ├── summary                  # 所有设备/通道状态快照
  └── dma<N>/                  # 各控制器私有调试信息
       └── ...                 # 由 dbg_summary_show 回调提供
```

`summary` 文件格式示例：
```
dma0 (2b00000.pdma0): number of channels: 4
 dma0chan0    | 2b00000.mcasp:tx
 dma0chan1    | 2b00000.mcasp:rx

dma1 (e8000000.edma): number of channels: 8
 dma1chan3    | in-use
```

---

## 10. 测试框架（`dmatest.c`，1376 行）

`dmatest` 提供完整的 DMA 引擎正确性和性能测试，通过 debugfs 或模块参数控制：

### 10.1 配置接口

```
# 通过 debugfs 配置
echo dma0chan0 > /sys/kernel/debug/dmatest/channel   # 指定测试通道
echo 1         > /sys/kernel/debug/dmatest/run        # 启动测试
cat /sys/kernel/debug/dmatest/results                 # 查看结果
```

模块参数（`insmod dmatest channel=dma0chan0 iterations=100 verbose=1`）：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `channel` | - | 指定通道名（空表示所有通道） |
| `timeout` | 3000 | 单次传输超时（ms） |
| `iterations` | 1 | 测试迭代次数（0=无限） |
| `threads_per_chan` | 1 | 每通道并发线程数 |
| `max_channels` | 0 | 最多测试通道数（0=全部） |
| `xor_sources` | 3 | XOR 测试源块数 |
| `pq_sources` | 3 | PQ 测试源块数 |
| `alignment` | -1 | 地址对齐要求（-1=随机） |

### 10.2 测试流程

```
dmatest_thread(data)
  → 分配源/目标缓冲区（可测试不对齐地址）
  → 填充随机测试数据 + 毒药值（poison）
  → dmaengine_prep_dma_memcpy / prep_dma_xor / prep_slave_sg
  → dmaengine_submit + dma_async_issue_pending
  → 等待完成（dma_sync_wait 或 completion）
  → 逐字节比对结果
  → 统计：错误数 / 吞吐量 / 延迟
```

---

## 11. 驱动生态地图

### 11.1 按平台分类

| 平台 | 驱动 | 备注 |
|------|------|------|
| ARM 通用 | `pl330.c` | 三星/海思/Rockchip/Marvell 广泛集成 |
| ARM SoC（Synopsys IP）| `dw/` | Intel/NXP/Renesas/Altera |
| Intel x86 服务器 | `ioat/`、`idxd/` | IOAT（旧）→ DSA（新） |
| Raspberry Pi | `bcm2835-dma.c` | BCM2835/2711 |
| NXP i.MX | `imx-dma.c`、`imx-sdma.c` | SDMA：可编程微控制器 |
| STM32 | `stm32/` | DMA/MDMA/BDMA 三级 DMA 体系 |
| TI K3 | `ti/` | UDMA（数据包感知 DMA） |
| Qualcomm | `qcom/` | BAM/GPI/hidma |
| Apple Silicon | `apple-admac.c` | ADMAC（音频专用） |
| MediaTek | `mediatek/` | CQDMA/HSDMA |
| Xilinx/AMD | `xilinx/` | AXI CDMA/VDMA/ZDMA |

### 11.2 关键源文件索引

| 文件 | 行数 | 职责 |
|------|------|------|
| `include/linux/dmaengine.h` | ~1200 | 全量 API 定义（Consumer + Provider） |
| `drivers/dma/dmaengine.c` | 1635 | Core 实现 |
| `drivers/dma/virt-dma.h` | ~240 | 虚拟通道描述符状态机（头文件即实现）|
| `drivers/dma/virt-dma.c` | ~90 | vchan_task、vchan_find_desc |
| `drivers/dma/of-dma.c` | 371 | DT 通道查找 |
| `drivers/dma/acpi-dma.c` | 462 | ACPI 通道查找 |
| `drivers/dma/dmatest.c` | 1376 | 内核 DMA 测试框架 |
| `drivers/dma/pl330.c` | 3274 | ARM PL330（可编程微处理器架构） |
| `drivers/dma/dw/core.c` | 1406 | DesignWare DMA 核心 |
| `drivers/dma/idxd/device.c` | 1629 | Intel DSA 设备管理 |
| `drivers/dma/ioat/dma.c` | 1062 | Intel IOAT 主体 |
| `drivers/dma/imx-sdma.c` | ~2500 | NXP i.MX SDMA（微控制器架构）|
