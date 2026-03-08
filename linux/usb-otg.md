# Linux USB OTG 框架深度分析

> 基于 Linux 6.19.6 内核源码
> 关键路径：`include/linux/usb/otg.h`、`include/linux/usb/otg-fsm.h`、`include/linux/usb/phy.h`、`include/linux/usb/role.h`、`drivers/usb/common/usb-otg-fsm.c`、`drivers/usb/chipidea/`

---

## 1. OTG 概述

USB On-The-Go（OTG）是 USB 规范的扩展，允许同一设备在不同场景下扮演**主机（Host）**或**从设备（Peripheral）**角色，从根本上打破了 USB "必须有一端是 PC" 的限制。

### 1.1 规范版本演进

| 版本 | 年份 | 关键特性 |
|------|------|----------|
| OTG 1.0 | 2001 | Mini-AB 接口、SRP、HNP |
| OTG 1.3 | 2006 | ADP（附着检测协议） |
| OTG 2.0 / EH 2.0 | 2011 | Embedded Host、HNP 轮询、高速 SRP |
| USB Type-C | 2014 | CC 引脚替代 ID 引脚，角色由 PD 协议协商 |

### 1.2 三大核心协议

```
┌─────────────────────────────────────────────────────────────┐
│  SRP（Session Request Protocol）会话请求协议                 │
│  B 设备向 A 设备请求上电（VBUS），用于节省功耗              │
│  触发方式：数据线脉冲（Data Pulsing）                        │
├─────────────────────────────────────────────────────────────┤
│  HNP（Host Negotiation Protocol）主机协商协议                │
│  A 设备与 B 设备在连接后动态交换主机/从设备角色             │
│  触发方式：挂起总线后，原从设备拉高 D+ 驱动总线             │
├─────────────────────────────────────────────────────────────┤
│  ADP（Attach Detection Protocol）附着检测协议                │
│  VBUS 未上电时检测线缆是否连接（电容探测）                   │
│  OTG 1.3 引入，用于节省电池设备功耗                         │
└─────────────────────────────────────────────────────────────┘
```

### 1.3 A/B 设备定义

```
Mini-A 接口（白色）: ID 引脚接地（ID=0）→ 默认为 A 设备 → 初始主机角色
Mini-B 接口（黑色）: ID 引脚悬空（ID=1）→ 默认为 B 设备 → 初始从设备角色
Mini-AB 接口（灰色）: 可接 Mini-A 或 Mini-B，通过 ID 引脚区分角色

DFP (Downstream Facing Port): 主机端口（向下提供 VBUS）
UFP (Upstream Facing Port):  从设备端口（从 VBUS 取电）
DRP (Dual Role Port):        双角色端口（USB Type-C）
```

---

## 2. 整体框架架构

```
┌───────────────────────────────────────────────────────────────┐
│                     用户空间                                   │
│  /sys/class/udc/<udc>/   /sys/bus/platform/devices/<dev>/     │
│       inputs/{a_bus_req, a_bus_drop, b_bus_req, a_clr_err}   │
└───────────────────────────┬───────────────────────────────────┘
                            │  sysfs / extcon / USB Type-C PD
┌───────────────────────────▼───────────────────────────────────┐
│           USB Role Switch Framework    (role.h)               │
│   usb_role_switch_register / usb_role_switch_set_role         │
│   GPIO 检测: usb-conn-gpio.c (ID/VBUS GPIO → 角色)           │
└───────────────────────────┬───────────────────────────────────┘
                            │
┌───────────────────────────▼───────────────────────────────────┐
│           OTG FSM Framework   (usb-otg-fsm.c)                 │
│   otg_statemachine() — 核心状态机                             │
│   struct otg_fsm — 输入/输出/状态变量                         │
│   struct otg_fsm_ops — 硬件操作钩子                           │
└───────────────────────────┬───────────────────────────────────┘
                            │
┌───────────────────────────▼───────────────────────────────────┐
│           USB PHY Layer   (phy.h / usb_phy)                   │
│   struct usb_phy — PHY 抽象（UTMI/ULPI/HSIC）                 │
│   struct usb_otg — OTG 控制器绑定（host + gadget）            │
│   usb_phy_events 通知链（VBUS/ID 事件）                       │
└───────────────────────────┬───────────────────────────────────┘
                            │
┌───────────────────────────▼───────────────────────────────────┐
│           具体控制器 OTG 实现                                  │
│   ChipIdea (ci_hdrc): chipidea/otg.c + otg_fsm.c             │
│   MUSB: musb/musb_core.c + omap2430.c                        │
│   DWC3: dwc3/drd.c                                           │
│   NXP i.MX: phy-mxs-usb.c                                   │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. 核心数据结构

### 3.1 enum usb_otg_state — OTG 状态机枚举

> `include/linux/usb/phy.h:43`

```c
enum usb_otg_state {
    OTG_STATE_UNDEFINED = 0,   /* 初始化前 */

    /* B 设备（ID=Hi）状态链 */
    OTG_STATE_B_IDLE,          /* B-空闲，等待 VBUS 或发起 SRP */
    OTG_STATE_B_SRP_INIT,      /* B-发起 SRP 会话请求 */
    OTG_STATE_B_PERIPHERAL,    /* B-从设备模式（常规外设工作状态） */
    OTG_STATE_B_WAIT_ACON,     /* B-等待 A 连接（HNP 中间态） */
    OTG_STATE_B_HOST,          /* B-主机模式（HNP 后 B 设备成为主机）*/

    /* A 设备（ID=Lo）状态链 */
    OTG_STATE_A_IDLE,          /* A-空闲 */
    OTG_STATE_A_WAIT_VRISE,    /* A-等待 VBUS 上升 */
    OTG_STATE_A_WAIT_BCON,     /* A-等待 B 连接 */
    OTG_STATE_A_HOST,          /* A-主机模式（正常工作状态）*/
    OTG_STATE_A_SUSPEND,       /* A-挂起（HNP 前挂起总线）*/
    OTG_STATE_A_PERIPHERAL,    /* A-从设备模式（HNP 后 A 设备成为从设备）*/
    OTG_STATE_A_WAIT_VFALL,    /* A-等待 VBUS 下降 */
    OTG_STATE_A_VBUS_ERR,      /* A-VBUS 错误（过流保护）*/
};
```

共 14 个状态，覆盖从上电到正常工作、HNP 角色交换、错误处理的全生命周期。

### 3.2 struct usb_otg — OTG 控制器抽象

> `include/linux/usb/otg.h:16`

```c
struct usb_otg {
    u8                default_a;    /* TRUE = 默认 A 设备（ID=Lo）*/

    struct phy       *phy;          /* 通用 PHY 框架接口（新式）*/
    struct usb_phy   *usb_phy;      /* USB PHY 接口（旧式）*/
    struct usb_bus   *host;         /* 绑定的主机控制器（HCD）*/
    struct usb_gadget *gadget;      /* 绑定的从设备控制器（UDC）*/

    enum usb_otg_state state;       /* 当前 OTG FSM 状态 */

    /* 绑定/解绑主机控制器 */
    int (*set_host)(struct usb_otg *otg, struct usb_bus *host);

    /* 绑定/解绑从设备控制器 */
    int (*set_peripheral)(struct usb_otg *otg, struct usb_gadget *gadget);

    /* A 设备控制 VBUS（B 设备忽略此接口）*/
    int (*set_vbus)(struct usb_otg *otg, bool enabled);

    /* B 设备发起 SRP */
    int (*start_srp)(struct usb_otg *otg);

    /* 启动 HNP 角色切换 */
    int (*start_hnp)(struct usb_otg *otg);
};
```

`usb_otg` 是 OTG 控制器的核心抽象，同时持有主机侧（`usb_bus`）和从设备侧（`usb_gadget`）的引用，通过状态机协调两者的使能/禁用。

### 3.3 struct usb_otg_caps — OTG 能力描述

> `include/linux/usb/otg.h:53`

```c
struct usb_otg_caps {
    u16  otg_rev;       /* OTG 规范版本，BCD 格式（0x0200 = OTG 2.0）*/
    bool hnp_support;   /* 支持 HNP（主机协商协议）*/
    bool srp_support;   /* 支持 SRP（会话请求协议）*/
    bool adp_support;   /* 支持 ADP（附着检测协议，OTG 1.3+）*/
};
```

此结构存入 `struct usb_gadget.otg_caps`，Gadget 框架在生成 OTG 描述符时读取其中字段。

### 3.4 struct usb_phy — USB PHY 抽象

> `include/linux/usb/phy.h:88`

```c
struct usb_phy {
    struct device            *dev;
    const char               *label;
    unsigned int              flags;

    enum usb_phy_type         type;        /* USB2/USB3 */
    enum usb_phy_events       last_event;  /* 最近事件 */
    struct usb_otg           *otg;         /* 关联的 OTG 控制器 */

    struct device            *io_dev;
    struct usb_phy_io_ops    *io_ops;      /* 寄存器读写（ULPI 接口）*/
    void __iomem             *io_priv;

    /* extcon 事件通知（连接器框架）*/
    struct extcon_dev        *edev;        /* VBUS extcon */
    struct extcon_dev        *id_edev;     /* ID  extcon */
    struct notifier_block     vbus_nb;
    struct notifier_block     id_nb;
    struct notifier_block     type_nb;

    /* USB 充电检测 */
    enum usb_charger_type     chg_type;    /* SDP/DCP/CDP/ACA */
    enum usb_charger_state    chg_state;
    struct usb_charger_current chg_cur;    /* SDP/DCP/CDP 电流范围 */
    struct work_struct         chg_work;

    /* 事件通知链（HOST/PERIPHERAL/CHARGER 等）*/
    struct atomic_notifier_head notifier;

    /* OTG 根 Hub 端口状态（传递给 HCD）*/
    u16  port_status;
    u16  port_change;

    /* PHY 操作接口 */
    int  (*init)(struct usb_phy *x);
    void (*shutdown)(struct usb_phy *x);
    int  (*set_vbus)(struct usb_phy *x, int on);
    int  (*set_power)(struct usb_phy *x, unsigned mA);
    int  (*set_suspend)(struct usb_phy *x, int suspend);
    int  (*set_wakeup)(struct usb_phy *x, bool enabled);
    int  (*notify_connect)(struct usb_phy *x, enum usb_device_speed speed);
    int  (*notify_disconnect)(struct usb_phy *x, enum usb_device_speed speed);
    enum usb_charger_type (*charger_detect)(struct usb_phy *x);
};
```

`usb_phy` 是旧式 PHY 接口，新驱动应使用通用 PHY 框架（`struct phy`）。`otg` 字段建立了 PHY 与 OTG 控制器的关联。

### 3.5 struct otg_fsm — OTG 状态机实例

> `include/linux/usb/otg-fsm.h:124`

```c
struct otg_fsm {
    /* === 输入变量 === */

    /* 通用输入 */
    int id;             /* ID 引脚：1=B设备, 0=A设备 */
    int adp_change;     /* ADP 测量值发生变化 */
    int power_up;       /* 设备上电，触发 ADP 测量 */

    /* A 设备输入 */
    int a_srp_det;      /* A 设备检测到 B 发起的 SRP */
    int a_vbus_vld;     /* VBUS 电压在有效范围内 */
    int b_conn;         /* A 检测到 B 设备已连接 */
    int a_bus_resume;   /* B 检测到 A 发出恢复信号（K态）*/

    /* B 设备输入 */
    int a_bus_suspend;  /* B 检测到 A 已挂起总线 */
    int a_conn;         /* B 检测到 A 设备已连接 */
    int b_se0_srp;      /* 总线处于 SE0 时间超过 SRP 最小值 */
    int b_ssend_srp;    /* VBUS 低于 VOTG_SESS_VLD 超过最小值 */
    int b_sess_vld;     /* B 检测到 VBUS 高于 VOTG_SESS_VLD */
    int test_device;    /* B-Host 检测到 OTG 测试设备 */

    /* 应用层输入（A 设备）*/
    int a_bus_drop;     /* A 应用请求断开（关闭 VBUS）*/
    int a_bus_req;      /* A 应用请求使用总线 */

    /* 应用层输入（B 设备）*/
    int b_bus_req;      /* B 应用请求使用总线 */

    /* === 超时指示 === */
    int a_wait_vrise_tmout;    /* VBUS 上升超时 */
    int a_wait_vfall_tmout;    /* VBUS 下降超时 */
    int a_wait_bcon_tmout;     /* 等待 B 连接超时 */
    int a_aidl_bdis_tmout;     /* A 挂起等待 B 断开超时 */
    int b_ase0_brst_tmout;     /* B 等待 A SE0 后恢复超时 */
    int a_bidl_adis_tmout;     /* A 从设备等待 B 挂起超时 */

    /* === 输出变量（只读，由 FSM 更新）=== */
    int drv_vbus;       /* 驱动 VBUS 上电 */
    int loc_conn;       /* 本地设备已连接（D+ 上拉/信号）*/
    int loc_sof;        /* 本地设备产生 SOF（HOST 工作）*/
    int adp_prb;        /* ADP 探测进行中 */
    int adp_sns;        /* ADP 感应进行中（B 设备）*/
    int data_pulse;     /* SRP 数据线脉冲进行中 */

    /* === 内部变量 === */
    int a_set_b_hnp_en; /* 已成功设置 B 设备的 b_hnp_enable 特性 */
    int b_srp_done;     /* B 设备已完成 SRP 发起 */
    int b_hnp_enable;   /* B 设备接受了 SetFeature(b_hnp_enable)*/
    int a_clr_err;      /* 清除 VBUS 过流错误 */

    struct otg_fsm_ops *ops;    /* 硬件操作接口 */
    struct usb_otg     *otg;    /* 关联的 OTG 控制器 */

    int protocol;       /* 当前运行协议：0=未定义/1=主机/2=从设备 */
    struct mutex lock;

    /* HNP 轮询（向连接的设备发送 GET_STATUS 轮询 host_request_flag）*/
    u8                  *host_req_flag;
    struct delayed_work  hnp_polling_work;
    bool                 hnp_work_inited;
    bool                 state_changed;  /* 本次 statemachine() 是否发生状态转换 */
};
```

`otg_fsm` 是状态机的完整上下文，所有输入事件（ID 变化、VBUS 变化、超时、应用请求）都以位标志形式写入，然后调用 `otg_statemachine()` 统一判断状态转换。

### 3.6 struct otg_fsm_ops — 硬件操作接口

> `include/linux/usb/otg-fsm.h:190`

```c
struct otg_fsm_ops {
    /* 对 VBUS 充电（B 设备 SRP 充电脉冲）*/
    void (*chrg_vbus)(struct otg_fsm *fsm, int on);
    /* 驱动 VBUS（A 设备上电）*/
    void (*drv_vbus)(struct otg_fsm *fsm, int on);
    /* 控制本地连接信号（D+ 上拉）*/
    void (*loc_conn)(struct otg_fsm *fsm, int on);
    /* 控制本地 SOF 生成（启停主机传输）*/
    void (*loc_sof)(struct otg_fsm *fsm, int on);
    /* 启动 SRP 数据线脉冲 */
    void (*start_pulse)(struct otg_fsm *fsm);
    /* 启动 ADP 探测 */
    void (*start_adp_prb)(struct otg_fsm *fsm);
    /* 启动 ADP 感应 */
    void (*start_adp_sns)(struct otg_fsm *fsm);
    /* 添加/删除 FSM 定时器 */
    void (*add_timer)(struct otg_fsm *fsm, enum otg_fsm_timer timer);
    void (*del_timer)(struct otg_fsm *fsm, enum otg_fsm_timer timer);
    /* 启停主机控制器 */
    int  (*start_host)(struct otg_fsm *fsm, int on);
    /* 启停从设备控制器 */
    int  (*start_gadget)(struct otg_fsm *fsm, int on);
};
```

`start_host(fsm, 1)` 和 `start_gadget(fsm, 1)` 是最关键的两个接口，它们触发实际的主机/从设备角色切换（加载 HCD 或 UDC）。

---

## 4. OTG 状态机（usb-otg-fsm.c）

### 4.1 OTG 状态机全图

```
                              ┌─ power_up/adp_change/a_bus_req ──┐
                              │                                   ↓
                           A_IDLE ─────── id=1 ──────────────→ B_IDLE
                              │                                   │
               a_vbus_vld     ↓  a_bus_req|a_srp_det             │ b_sess_vld
                    ┌── A_WAIT_VRISE                              ↓
                    │         │                            B_PERIPHERAL
                    │    a_vbus_vld                               │
                    ↓         ↓                      b_bus_req & b_hnp_enable
              A_VBUS_ERR  A_WAIT_BCON                 & a_bus_suspend
                    ↑         │ b_conn                            ↓
              !a_vbus_vld     ↓                            B_WAIT_ACON
                    │      A_HOST ──→ !a_bus_req & b_hnp → A_SUSPEND
                    │         │                                   │ a_conn
                    │         │ !b_conn                           ↓
                    │    A_WAIT_BCON              B_HOST (B 成为主机)
                    │                                    !b_bus_req ↓
                    │    A_SUSPEND ── !b_conn & b_hnp → A_PERIPHERAL
                    │         │
                    │    A_WAIT_VFALL ── a_wait_vfall_tmout → A_IDLE
                    └─────────┘
```

### 4.2 otg_statemachine() — 状态转换判断

> `drivers/usb/common/usb-otg-fsm.c:328`

`otg_statemachine()` 是状态机的核心，每当输入变量改变时调用，遍历当前状态的所有转换条件：

```c
int otg_statemachine(struct otg_fsm *fsm)
{
    enum usb_otg_state state;
    mutex_lock(&fsm->lock);
    state = fsm->otg->state;
    fsm->state_changed = 0;

    switch (state) {
    case OTG_STATE_UNDEFINED:
        /* 首次调用：根据 ID 引脚决定初始角色 */
        if (fsm->id)
            otg_set_state(fsm, OTG_STATE_B_IDLE);  /* ID=Hi → B 设备 */
        else
            otg_set_state(fsm, OTG_STATE_A_IDLE);  /* ID=Lo → A 设备 */
        break;

    case OTG_STATE_B_IDLE:
        if (!fsm->id)                         /* ID 变为 Lo → 成为 A 设备 */
            otg_set_state(fsm, OTG_STATE_A_IDLE);
        else if (fsm->b_sess_vld && fsm->otg->gadget)
            otg_set_state(fsm, OTG_STATE_B_PERIPHERAL); /* VBUS 有效 → 从设备 */
        else if ((fsm->b_bus_req || fsm->adp_change || fsm->power_up)
                 && fsm->b_ssend_srp && fsm->b_se0_srp)
            otg_set_state(fsm, OTG_STATE_B_SRP_INIT);   /* 发起 SRP */
        break;

    case OTG_STATE_B_PERIPHERAL:
        if (!fsm->id || !fsm->b_sess_vld)
            otg_set_state(fsm, OTG_STATE_B_IDLE);
        /* HNP：B 应用请求成为主机，且 A 已使能 HNP，且 A 已挂起 */
        else if (fsm->b_bus_req && fsm->otg->gadget->b_hnp_enable
                 && fsm->a_bus_suspend)
            otg_set_state(fsm, OTG_STATE_B_WAIT_ACON);
        break;

    case OTG_STATE_A_HOST:
        if (fsm->id || fsm->a_bus_drop)
            otg_set_state(fsm, OTG_STATE_A_WAIT_VFALL);
        /* HNP：A 应用不再需要总线，且 B 已接受 HNP 使能 */
        else if ((!fsm->a_bus_req || fsm->a_suspend_req_inf)
                 && fsm->otg->host->b_hnp_enable)
            otg_set_state(fsm, OTG_STATE_A_SUSPEND);
        else if (!fsm->b_conn)
            otg_set_state(fsm, OTG_STATE_A_WAIT_BCON);
        else if (!fsm->a_vbus_vld)
            otg_set_state(fsm, OTG_STATE_A_VBUS_ERR);
        break;

    case OTG_STATE_A_SUSPEND:
        /* B 断开连接 + B 已接受 HNP → A 进入从设备模式 */
        if (!fsm->b_conn && fsm->otg->host->b_hnp_enable)
            otg_set_state(fsm, OTG_STATE_A_PERIPHERAL);
        /* ... 其他条件 ... */
        break;

    /* ... 其余状态 ... */
    }

    mutex_unlock(&fsm->lock);
    return fsm->state_changed;
}
```

### 4.3 otg_set_state() — 状态进入动作

> `drivers/usb/common/usb-otg-fsm.c:206`

每次进入新状态时，`otg_set_state()` 先调用 `otg_leave_state()`（清理旧状态的定时器和变量），再执行新状态的入口动作：

| 新状态 | 主要入口动作 |
|--------|-------------|
| `B_IDLE` | 关闭 VBUS 驱动、关闭连接信号、启动 ADP 感应、启动 B_SE0_SRP 定时器 |
| `B_SRP_INIT` | 启动数据线脉冲（`start_pulse`）、启动 B_SRP_FAIL 定时器 |
| `B_PERIPHERAL` | 关闭充电、启动从设备协议（`start_gadget(1)`）、使能连接信号 |
| `B_WAIT_ACON` | 切换到主机协议（`start_gadget(0)` + `start_host(1)`）、启动 B_ASE0_BRST 定时器 |
| `B_HOST` | 启动枚举（`usb_bus_start_enum`）、启动 HNP 轮询 |
| `A_IDLE` | 关闭 VBUS、启动 ADP 探测、协议设为主机 |
| `A_WAIT_VRISE` | 驱动 VBUS（`drv_vbus(1)`）、启动 A_WAIT_VRISE 定时器 |
| `A_HOST` | 维持 VBUS、启动 SOF（`loc_sof(1)`）、启动 HNP 轮询 |
| `A_SUSPEND` | 停止 SOF（`loc_sof(0)`）、启动 A_AIDL_BDIS 定时器 |
| `A_PERIPHERAL` | 启动从设备协议（`start_gadget(1)`）、维持 VBUS 驱动 |
| `A_WAIT_VFALL` | 停止 VBUS 驱动（`drv_vbus(0)`）、启动 A_WAIT_VFALL 定时器 |
| `A_VBUS_ERR` | 停止 VBUS、停止所有信号、协议设为未定义 |

---

## 5. OTG 协议详解

### 5.1 SRP（Session Request Protocol）

SRP 允许 B 设备在 VBUS 空闲时请求 A 设备上电，从而节省系统功耗。

```
B-IDLE:
  等待 b_ssend_srp（VBUS 低于 VOTG_SESS_VLD 超过 TB_SSEND_SRP=1.5s）
  且 b_se0_srp（总线 SE0 超过 TB_SE0_SRP=1s）
        ↓
B-SRP_INIT:
  发起数据线脉冲（Data Pulsing）:
    1. B 设备将 D+ 驱动高电平，持续 TB_DATA_PLS=5~10ms
    2. A 设备 PHY 检测到脉冲，设置 a_srp_det=1
  同时启动 TB_SRP_FAIL=6s 超时
        ↓
  A 设备收到 SRP 信号（a_srp_det=1）:
    A_IDLE → A_WAIT_VRISE: 驱动 VBUS（drv_vbus=1）
        ↓
  VBUS 上升（a_vbus_vld=1）:
    A_WAIT_VRISE → A_WAIT_BCON
        ↓
  B 设备检测到 VBUS（b_sess_vld=1）:
    B_SRP_INIT → B_PERIPHERAL（SRP 成功）
```

SRP 失败（B_SRP_FAIL 超时）：B 设备退回 B_IDLE，可以重试。

**SRP 时序参数（chipidea/otg_fsm.h）：**

| 参数 | 值 | 含义 |
|------|-----|------|
| `TB_DATA_PLS` | 10ms | SRP 数据线脉冲时长 |
| `TB_SE0_SRP` | 1000ms | 发起 SRP 前最低 SE0 持续时间 |
| `TB_SSEND_SRP` | 1500ms | 发起 SRP 前 VBUS 最低低电平时间 |
| `TB_SRP_FAIL` | 6000ms | SRP 失败超时（B 未收到 VBUS 则放弃）|

### 5.2 HNP（Host Negotiation Protocol）

HNP 实现在已连接的 A/B 设备之间动态交换主机角色。

```
初始状态：A=主机，B=从设备

Step 1: B 设备应用请求成为主机（b_bus_req=1）
         B_PERIPHERAL → B_WAIT_ACON（条件：b_hnp_enable=1, a_bus_suspend=1）

Step 2: A 主机 HNP 轮询（hnp_polling_work）
    ┌── 每 T_HOST_REQ_POLL=1500ms 向 B 发送 GET_STATUS(OTG_STS_SELECTOR)
    └── 检查 B 返回的 host_request_flag（0x01=B 请求主机角色）

Step 3: A 检测到 B 请求主机角色
    A 发送 SET_FEATURE(USB_DEVICE_B_HNP_ENABLE) 给 B
    host->b_hnp_enable = 1
    a_bus_req = 0（A 放弃总线）
    A_HOST → A_SUSPEND

Step 4: A 挂起总线（停止 SOF，TA_AIDL_BDIS=5s）
    B 检测到 a_bus_suspend=1
    B 断开 D+ 上拉（loc_conn=0），驱动总线
    B_WAIT_ACON: 等待 A 作为设备重新连接

Step 5: A 检测到 B 断开（b_conn=0）
    A_SUSPEND → A_PERIPHERAL（若 b_hnp_enable=1）
    A 切换为从设备（start_gadget(1)，VBUS 仍维持）

Step 6: B 成为主机
    B_WAIT_ACON → B_HOST（检测到 A 作为设备连接）
    B 启动枚举，对 A 发送 RESET

Step 7: 角色恢复
    B 主机不再需要（b_bus_req=0）或 A 重新请求
    B_HOST → B_PERIPHERAL
    A_PERIPHERAL → A_WAIT_BCON → A_HOST
```

**HNP 时序参数：**

| 参数 | 值 | 含义 |
|------|-----|------|
| `T_HOST_REQ_POLL` | 1500ms | HNP 轮询间隔 |
| `TA_AIDL_BDIS` | 5000ms | A 挂起等待 B 断开最长时间 |
| `TB_ASE0_BRST` | 155ms | B 等待 A SE0 信号（复位信号）最短时间 |
| `TA_BIDL_ADIS` | 500ms | A 从设备等待 B 挂起最长时间 |

### 5.3 ADP（Attach Detection Protocol）

ADP 通过周期性测量连接器上的寄生电容来检测设备连接，无需上电 VBUS：

```
A_IDLE → 启动 ADP 探测（adp_prb=1）:
  周期性向 VBUS 施加小电流，测量充电时间
  时间越短 → 电容越大 → 有设备连接

B_IDLE → 启动 ADP 感应（adp_sns=1）:
  监测 VBUS 上的 ADP 探测脉冲

adp_change=1: 测量值变化超过阈值（CADP_THR）→ 触发状态机
```

ADP 目前在 Linux 内核中部分实现（chipidea OTG FSM TODO 列表中标注为待实现）。

---

## 6. HNP 轮询机制

> `drivers/usb/common/usb-otg-fsm.c:120`

HNP 轮询是 OTG 2.0 引入的机制，主机周期性向连接的 OTG 设备查询是否请求主机角色：

```c
static void otg_hnp_polling_work(struct work_struct *work)
{
    struct otg_fsm *fsm = container_of(to_delayed_work(work), ...);
    u8 flag;

    /* 向连接的 USB 设备发送标准 GET_STATUS 请求 */
    retval = usb_control_msg(udev,
        usb_rcvctrlpipe(udev, 0),
        USB_REQ_GET_STATUS,
        USB_DIR_IN | USB_RECIP_DEVICE,
        0,
        OTG_STS_SELECTOR,        /* wIndex = 0xF000 */
        fsm->host_req_flag,      /* 1 字节 */
        1,
        USB_CTRL_GET_TIMEOUT);

    flag = *fsm->host_req_flag;
    if (flag == HOST_REQUEST_FLAG) {    /* 0x01 */
        /* 设备请求成为主机：设置 SetFeature(B_HNP_ENABLE) */
        usb_control_msg(udev, ..., USB_REQ_SET_FEATURE, 0,
                        USB_DEVICE_B_HNP_ENABLE, ...);
        host->b_hnp_enable = 1;
        fsm->a_bus_req = 0;     /* A 放弃总线 */
        otg_statemachine(fsm);  /* 触发状态机 */
    } else {
        /* 继续轮询 */
        schedule_delayed_work(&fsm->hnp_polling_work,
                              msecs_to_jiffies(T_HOST_REQ_POLL));
    }
}
```

**OTG 状态选择器（OTG_STS_SELECTOR = 0xF000）：**

GET_STATUS 请求中 `wIndex=0xF000` 是 OTG 协议定义的特殊选择器，返回 1 字节的 OTG 状态字节，其 bit0（HOST_REQUEST_FLAG）表示从设备是否请求成为主机。

---

## 7. USB Role Switch 框架

> `include/linux/usb/role.h`

Role Switch 是 Linux 内核为 **USB Type-C** 场景提供的新式角色管理框架，比 OTG FSM 更简洁，适用于通过 CC 引脚协商角色的 Type-C 设备。

### 7.1 核心接口

```c
enum usb_role {
    USB_ROLE_NONE,      /* 无角色（断开）*/
    USB_ROLE_HOST,      /* 主机角色 */
    USB_ROLE_DEVICE,    /* 从设备角色 */
};

struct usb_role_switch_desc {
    struct fwnode_handle *fwnode;    /* DT/ACPI 节点 */
    struct device *usb2_port;        /* USB2 主机端口 */
    struct device *usb3_port;        /* USB3 主机端口 */
    struct device *udc;              /* UDC 设备 */
    usb_role_switch_set_t set;       /* 设置角色回调 */
    usb_role_switch_get_t get;       /* 获取当前角色回调 */
    bool allow_userspace_control;    /* 是否允许 sysfs 控制 */
    void *driver_data;
    const char *name;
};
```

### 7.2 注册与使用

```c
/* 控制器驱动注册 Role Switch */
struct usb_role_switch *sw = usb_role_switch_register(dev, &desc);

/* 角色切换（由 Type-C 控制器驱动或用户空间触发）*/
usb_role_switch_set_role(sw, USB_ROLE_HOST);    /* 切换为主机 */
usb_role_switch_set_role(sw, USB_ROLE_DEVICE);  /* 切换为从设备 */

/* 用户空间通过 sysfs 查询和设置（若 allow_userspace_control=true）*/
/* /sys/class/usb_role/<switch-name>/role */
```

Role Switch 与 OTG FSM 的对比：

| 维度 | OTG FSM | Role Switch |
|------|---------|-------------|
| 适用场景 | USB 2.0 Mini/Micro OTG | USB Type-C (CC 引脚) |
| 角色协商 | ID 引脚 + HNP/SRP | CC 引脚 + USB PD |
| 状态机 | 14 个状态，复杂 | 3 个角色，简单 |
| 内核接口 | `otg_statemachine()` | `usb_role_switch_set_role()` |
| 用户空间 | sysfs inputs 属性 | sysfs role 属性 |

---

## 8. GPIO 连接器检测（usb-conn-gpio.c）

> `drivers/usb/common/usb-conn-gpio.c`

对于使用 GPIO 检测 ID/VBUS 的嵌入式平台，内核提供了通用的 GPIO 连接器驱动：

```c
struct usb_conn_info {
    struct device            *dev;
    struct usb_role_switch   *role_sw;   /* 关联的 Role Switch */
    struct power_supply      *charger;   /* USB 充电 PSY */

    struct gpio_desc *id_gpiod;    /* ID 引脚 GPIO（低=A设备）*/
    struct gpio_desc *vbus_gpiod;  /* VBUS 检测 GPIO */

    struct delayed_work dw_det;    /* 防抖延迟工作队列 */
};
```

检测逻辑：

```c
static void usb_conn_detect_cable(struct work_struct *work)
{
    /* 读取 GPIO 状态 */
    id   = id_gpiod   ? gpiod_get_value_cansleep(id_gpiod)   : 1;
    vbus = vbus_gpiod ? gpiod_get_value_cansleep(vbus_gpiod) : id;

    /* 确定角色 */
    if (!id)            role = USB_ROLE_HOST;   /* ID=Lo → 主机 */
    else if (vbus)      role = USB_ROLE_DEVICE; /* VBUS=Hi → 从设备 */
    else                role = USB_ROLE_NONE;   /* 无连接 */

    /* 通知 Role Switch */
    usb_role_switch_set_role(info->role_sw, role);
}
```

---

## 9. ChipIdea OTG 实现

ChipIdea（Freescale/NXP USB IP）是内核中最完整的 OTG FSM 实现，覆盖 OTGSC 寄存器管理、全套定时器和 sysfs 接口。

### 9.1 OTGSC 寄存器

ChipIdea 控制器有专用的 OTG 状态控制寄存器（OTGSC）：

| 位 | 名称 | 含义 |
|----|------|------|
| `OTGSC_ID` | ID | ID 引脚状态（1=B设备）|
| `OTGSC_BSV` | BSVIS | B-Session Valid（VBUS > VOTG_SESS_VLD）|
| `OTGSC_ASV` | A-Session Valid | VBUS > VA_SESS_VLD |
| `OTGSC_IDIS` | ID 中断状态 | ID 引脚变化 |
| `OTGSC_BSVIS` | VBUS 中断状态 | VBUS 变化 |
| `OTGSC_IDIE` | ID 中断使能 | |
| `OTGSC_BSVIE` | VBUS 中断使能 | |
| `OTGSC_DPIE` | 数据脉冲中断使能 | |
| `OTGSC_PP` | 端口电源（PORTSC） | 驱动 VBUS |

```c
/* chipidea/otg.c:29 */
u32 hw_read_otgsc(struct ci_hdrc *ci, u32 mask)
{
    u32 val = hw_read(ci, OP_OTGSC, mask);

    /* 若使用 extcon 框架，用外部通知覆盖寄存器值 */
    cable = &ci->platdata->vbus_extcon;
    if (!IS_ERR(cable->edev) || ci->role_switch) {
        if (cable->connected)  val |= OTGSC_BSV;
        else                   val &= ~OTGSC_BSV;
    }
    /* ID 引脚类似处理 */
    return val & mask;
}
```

`hw_read_otgsc()` 支持两种 ID/VBUS 信号源：
- **寄存器直读**：直接从硬件 OTGSC 读取
- **extcon 覆盖**：若使用外部连接器框架（Type-C 控制器），用 extcon 事件值覆盖寄存器值

### 9.2 角色切换流程

```c
/* chipidea/otg.c:122 */
enum ci_role ci_otg_role(struct ci_hdrc *ci)
{
    /* OTGSC_ID=1 → B设备（Gadget），OTGSC_ID=0 → A设备（Host）*/
    return hw_read_otgsc(ci, OTGSC_ID) ? CI_ROLE_GADGET : CI_ROLE_HOST;
}

/* chipidea/otg.c:171 */
void ci_handle_id_switch(struct ci_hdrc *ci)
{
    enum ci_role role = ci_otg_role(ci);

    if (role != ci->role) {
        ci_role_stop(ci);              /* 停止当前角色的 HCD 或 UDC */
        if (role == CI_ROLE_GADGET)
            hw_wait_vbus_lower_bsv(ci); /* 等待 VBUS 降低（5s 超时）*/
        ci_role_start(ci, role);       /* 启动新角色的 HCD 或 UDC */
        if (role == CI_ROLE_GADGET)
            ci_handle_vbus_change(ci);  /* 处理 VBUS 状态 */
    }
}
```

### 9.3 FSM 定时器实现

ChipIdea 使用 `hrtimer` 实现多定时器管理，避免为每个 OTG 定时器创建独立 timer：

```c
/* chipidea/otg_fsm.c:383 */
static enum hrtimer_restart ci_otg_hrtimer_func(struct hrtimer *t)
{
    /* 扫描所有已启用定时器，找出已超时的 */
    for_each_set_bit(cur_timer, &enabled_timer_bits, NUM_OTG_FSM_TIMERS) {
        if (ktime_compare(now, ci->hr_timeouts[cur_timer]) >= 0) {
            /* 超时：调用对应处理器（设置 fsm->xxx_tmout=1）*/
            otg_timer_handlers[cur_timer](ci);
        }
    }
    /* 重新编程 hrtimer 到下一个最近超时 */
    if (next_timer < NUM_OTG_FSM_TIMERS)
        hrtimer_start_range_ns(&ci->otg_fsm_hrtimer,
                               ci->hr_timeouts[next_timer], ...);

    /* 触发工作队列重新运行状态机 */
    ci_otg_queue_work(ci);
}
```

### 9.4 sysfs 用户空间接口

ChipIdea OTG 在 `/sys/bus/platform/devices/<ci_hdrc>/inputs/` 下暴露 4 个可写属性：

| 属性 | 可写值 | 含义 |
|------|--------|------|
| `a_bus_req` | 0/1 | A 设备应用是否请求使用总线 |
| `a_bus_drop` | 0/1 | A 设备应用是否要求关闭 VBUS（设置后 a_bus_req 自动清零）|
| `b_bus_req` | 0/1 | B 设备应用是否请求成为主机（触发 HNP）|
| `a_clr_err` | 1 | 清除 A_VBUS_ERR 状态（过流恢复）|

---

## 10. OTG 完整状态转换表

| 当前状态 | 转换条件 | 目标状态 | 触发动作 |
|----------|----------|----------|---------|
| UNDEFINED | `id=1` | B_IDLE | 初始化 B 设备 |
| UNDEFINED | `id=0` | A_IDLE | 初始化 A 设备 |
| B_IDLE | `!id` | A_IDLE | 角色切换 |
| B_IDLE | `b_sess_vld` | B_PERIPHERAL | 启动 Gadget |
| B_IDLE | `b_bus_req & b_ssend_srp & b_se0_srp` | B_SRP_INIT | 发起 SRP |
| B_SRP_INIT | `!id \| b_srp_done` | B_IDLE | 停止 SRP |
| B_PERIPHERAL | `!id \| !b_sess_vld` | B_IDLE | 停止 Gadget |
| B_PERIPHERAL | `b_bus_req & b_hnp_enable & a_bus_suspend` | B_WAIT_ACON | 停止 Gadget，启动 Host |
| B_WAIT_ACON | `a_conn` | B_HOST | 完成角色切换 |
| B_WAIT_ACON | `a_bus_resume \| b_ase0_brst_tmout` | B_PERIPHERAL | 恢复从设备角色 |
| B_HOST | `!b_bus_req \| !a_conn \| test_device` | B_PERIPHERAL | 交还主机角色 |
| A_IDLE | `id` | B_IDLE | 角色切换 |
| A_IDLE | `!a_bus_drop & (a_bus_req\|a_srp_det\|...)` | A_WAIT_VRISE | 驱动 VBUS |
| A_WAIT_VRISE | `a_vbus_vld` | A_WAIT_BCON | VBUS 就绪 |
| A_WAIT_VRISE | `id \| a_bus_drop \| a_wait_vrise_tmout` | A_WAIT_VFALL | 关闭 VBUS |
| A_WAIT_BCON | `b_conn` | A_HOST | B 已连接 |
| A_WAIT_BCON | `!a_vbus_vld` | A_VBUS_ERR | 过流 |
| A_WAIT_BCON | `id \| a_bus_drop \| a_wait_bcon_tmout` | A_WAIT_VFALL | 超时 |
| A_HOST | `!a_bus_req & b_hnp_enable` | A_SUSPEND | 发起 HNP |
| A_HOST | `!b_conn` | A_WAIT_BCON | B 断开 |
| A_HOST | `!a_vbus_vld` | A_VBUS_ERR | 过流 |
| A_SUSPEND | `!b_conn & b_hnp_enable` | A_PERIPHERAL | HNP 完成 |
| A_SUSPEND | `!b_conn & !b_hnp_enable` | A_WAIT_BCON | B 断开无 HNP |
| A_SUSPEND | `a_bus_req \| b_bus_resume` | A_HOST | 恢复主机 |
| A_SUSPEND | `id \| a_bus_drop \| a_aidl_bdis_tmout` | A_WAIT_VFALL | 放弃 |
| A_PERIPHERAL | `id \| a_bus_drop` | A_WAIT_VFALL | ID 变化 |
| A_PERIPHERAL | `a_bidl_adis_tmout \| b_bus_suspend` | A_WAIT_BCON | 恢复主机 |
| A_WAIT_VFALL | `a_wait_vfall_tmout` | A_IDLE | VBUS 已落 |
| A_VBUS_ERR | `id \| a_bus_drop \| a_clr_err` | A_WAIT_VFALL | 错误恢复 |

---

## 11. OTG 定时器规格

所有定时器值源自 OTG 规范 Table 5-1（Section 5.5）：

| 定时器 | 枚举名 | 超时值（ms）| 含义 |
|--------|--------|-------------|------|
| `A_WAIT_VRISE` | `A_WAIT_VRISE` | 100 | 等待 VBUS 上升至有效电平 |
| `A_WAIT_VFALL` | `A_WAIT_VFALL` | 1000 | 等待 VBUS 下降至安全电平 |
| `A_WAIT_BCON` | `A_WAIT_BCON` | 10000 | 等待 B 设备连接（最大值）|
| `A_AIDL_BDIS` | `A_AIDL_BDIS` | 5000 | A 挂起状态等待 B 断开 |
| `B_ASE0_BRST` | `B_ASE0_BRST` | 155 | B 等待 A 复位信号最短时间 |
| `A_BIDL_ADIS` | `A_BIDL_ADIS` | 500 | A 从设备等待 B 挂起 |
| `B_AIDL_BDIS` | `B_AIDL_BDIS` | 20 | B 挂起到断开时延 |
| — | `B_SE0_SRP` | 1000 | SRP 前 SE0 最低持续时间 |
| — | `B_SRP_FAIL` | 6000 | SRP 会话请求失败超时 |
| — | `B_DATA_PLS` | 10 | SRP 数据线脉冲宽度 |
| — | `B_SSEND_SRP` | 1500 | SRP 前 VBUS 低电平最低持续时间 |

---

## 12. 关键调用链

### 12.1 A 设备从空闲到主机模式

```
ID 引脚接地（ID=Lo）
        ↓
中断 → ci_otg_fsm_irq()
    检测到 OTGSC_IDIS（ID 变化）
    读 OTGSC_ID=0
    fsm->id = 0
        ↓
ci_otg_queue_work() → ci_otg_work() → ci_otg_fsm_work() → otg_statemachine()
        ↓
UNDEFINED → A_IDLE:
    otg_drv_vbus(fsm, 0)
    otg_start_adp_prb(fsm)
    otg_set_protocol(fsm, PROTO_HOST)
        ↓
a_bus_req=1（用户写入 sysfs 或自动）
otg_statemachine()
A_IDLE → A_WAIT_VRISE:
    otg_drv_vbus(fsm, 1)    ← 调用 ci_otg_drv_vbus → regulator_enable
    otg_add_timer(A_WAIT_VRISE)
        ↓
VBUS 上升，中断触发
    fsm->a_vbus_vld = 1
otg_statemachine()
A_WAIT_VRISE → A_WAIT_BCON:
    otg_drv_vbus(fsm, 1)
    otg_add_timer(A_WAIT_BCON)
        ↓
B 设备连接，D+ 拉高被 A 检测
    fsm->b_conn = 1
otg_statemachine()
A_WAIT_BCON → A_HOST:
    otg_drv_vbus(fsm, 1)
    otg_loc_sof(fsm, 1)     ← 主机开始发 SOF
    otg_set_protocol(fsm, PROTO_HOST) → otg_start_host(fsm, 1)
        ↓
ci_otg_fsm_ops.start_host → ci_role_start(CI_ROLE_HOST)
    → 加载 EHCI/XHCI HCD，开始枚举
        ↓
otg_start_hnp_polling() → 启动 HNP 轮询工作队列
```

### 12.2 HNP 角色切换完整链

```
A_HOST 状态下，HNP 轮询检测到 B 请求主机
        ↓
otg_hnp_polling_work():
    GET_STATUS(OTG_STS_SELECTOR) → flag = 0x01
    SET_FEATURE(USB_DEVICE_B_HNP_ENABLE)
    host->b_hnp_enable = 1
    fsm->a_bus_req = 0
    otg_statemachine()
        ↓
A_HOST → A_SUSPEND:
    otg_loc_sof(fsm, 0)     ← 停止 SOF（总线挂起）
    otg_set_protocol(HOST)  ← 保持主机协议（D+ 还在驱动）
    otg_add_timer(A_AIDL_BDIS)
        ↓
B 设备检测到 a_bus_suspend=1
B_PERIPHERAL → B_WAIT_ACON:
    otg_set_protocol(HOST)  ← start_host(1)（B 启动 HCD）
    otg_chrg_vbus(fsm, 0)   ← B 断开 D+
    otg_add_timer(B_ASE0_BRST)
        ↓
A 检测到 b_conn=0（B 断开）
A_SUSPEND → A_PERIPHERAL（b_hnp_enable=1）:
    otg_set_protocol(GADGET) ← start_gadget(1)（A 启动 UDC）
    otg_drv_vbus(fsm, 1)     ← A 保持 VBUS 供电
    otg_loc_conn(fsm, 1)     ← A 作为从设备连接
    otg_add_timer(A_BIDL_ADIS)
        ↓
B 检测到 A 重新连接（a_conn=1）
B_WAIT_ACON → B_HOST:
    otg_loc_sof(fsm, 1)     ← B 开始发 SOF
    usb_bus_start_enum(host, otg_port)  ← 枚举 A 设备
    otg_start_hnp_polling() ← B 也启动 HNP 轮询
```

---

## 13. USB Type-C 与 OTG 的关系

USB Type-C 接口用 **CC（Configuration Channel）引脚**替代了传统 OTG 的 **ID 引脚**，用 **USB PD（Power Delivery）协议**替代了 **SRP/HNP**：

| 功能 | 传统 OTG | USB Type-C |
|------|----------|-----------|
| 角色检测 | ID 引脚 (Hi/Lo) | CC 引脚电阻（Rd/Rp）|
| 电源协商 | VBUS 固定 5V | USB PD（最高 48V/5A）|
| 角色交换 | HNP 协议 | PD PR_SWAP/DR_SWAP 报文 |
| 角色通知 | extcon/OTGSC 中断 | USB PD 事件 → Role Switch |
| 内核接口 | `otg_statemachine()` | `usb_role_switch_set_role()` |

在 Type-C 场景中，TCPM（Type-C Port Manager，`drivers/usb/typec/tcpm/`）负责 CC 引脚检测和 PD 协商，最终通过 Role Switch 框架通知 USB 控制器切换角色，完全绕过传统的 OTG FSM。

---

## 14. 关键源文件索引

| 文件 | 行数 | 职责 |
|------|------|------|
| `include/linux/usb/otg.h` | 133行 | OTG 核心接口：usb_otg / usb_otg_caps / usb_dr_mode |
| `include/linux/usb/otg-fsm.h` | 312行 | FSM 接口：otg_fsm / otg_fsm_ops / otg_fsm_timer |
| `include/linux/usb/phy.h` | 356行 | PHY 接口：usb_phy / usb_otg_state / usb_phy_events |
| `include/linux/usb/role.h` | 126行 | Role Switch 接口：usb_role_switch / usb_role |
| `drivers/usb/common/usb-otg-fsm.c` | 454行 | OTG 通用状态机实现（otg_statemachine）|
| `drivers/usb/common/usb-conn-gpio.c` | 375行 | GPIO ID/VBUS 检测驱动 |
| `drivers/usb/chipidea/otg.c` | ~230行 | ChipIdea OTGSC 寄存器管理、ID/VBUS 切换 |
| `drivers/usb/chipidea/otg_fsm.c` | ~700行 | ChipIdea FSM 实现：hrtimer、sysfs inputs、ops 实现 |
| `drivers/usb/chipidea/otg_fsm.h` | 101行 | ChipIdea OTG 定时器参数 |
| `drivers/usb/musb/musb_core.c` | ~2800行 | MUSB 双角色控制器（含 OTG 支持）|
| `drivers/usb/phy/phy-generic.c` | ~350行 | 通用 USB PHY（VBUS/ID 上拉控制）|
