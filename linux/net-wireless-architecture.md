# Linux net/wireless 子系统架构深度分析

> 基于 Linux 6.19.6 内核 `net/wireless/` 源码
> 面向内核驱动开发的工程参考文档

---

## 1. 整体架构

### 1.1 五层分层图

```
┌──────────────────────────────────────────────────────────────────┐
│                     用户空间（User Space）                        │
│  NetworkManager · wpa_supplicant · hostapd · iw · wlanconfig     │
└───────────────────────────┬──────────────────────────────────────┘
                            │ AF_NETLINK（Generic Netlink）
                            │ SIOCGIWXXX（遗留 Wireless Extensions）
┌───────────────────────────▼──────────────────────────────────────┐
│               nl80211 Netlink 接口层（net/wireless/nl80211.c）    │
│  21911 行  ·  380+ genl_ops  ·  6 个多播组                       │
│  命令解析 → 权限检查 → cfg80211 API 调用 → 事件上报              │
└───────────────────────────┬──────────────────────────────────────┘
                            │
┌───────────────────────────▼──────────────────────────────────────┐
│             cfg80211 无线配置管理框架（net/wireless/）            │
│                                                                    │
│  ┌──────────┬──────────┬──────────┬──────────┬─────────────────┐ │
│  │ core.c   │ scan.c   │ mlme.c   │ sme.c    │ reg.c           │ │
│  │ 1920行   │ 4013行   │ 1419行   │ 1622行   │ 4372行          │ │
│  │ 设备注册 │ 扫描/BSS │ MLME事件 │ 软件SME  │ 频谱管制        │ │
│  └──────────┴──────────┴──────────┴──────────┴─────────────────┘ │
│                                                                    │
│  ┌──────────┬──────────┬──────────┬──────────┬─────────────────┐ │
│  │ chan.c   │ mesh.c   │ ibss.c   │ ap.c     │ util.c          │ │
│  │ 1609行   │ 289行    │ 496行    │ 74行     │ 2988行          │ │
│  │ 信道管理 │ Mesh网络 │ Ad-hoc   │ AP模式   │ 工具/IE解析     │ │
│  └──────────┴──────────┴──────────┴──────────┴─────────────────┘ │
│                                                                    │
│  ┌──────────┬──────────┬──────────┬──────────┬─────────────────┐ │
│  │ pmsr.c   │ ocb.c    │ wext-*   │ sysfs.c  │ debugfs.c       │ │
│  │ 668行    │ 65行     │ 3362行   │ 181行    │ 304行           │ │
│  │ 对等测量 │ 车联网   │ 遗留兼容 │ sysfs    │ 调试接口        │ │
│  └──────────┴──────────┴──────────┴──────────┴─────────────────┘ │
└───────────────────────────┬──────────────────────────────────────┘
                            │ struct cfg80211_ops（150+ 回调）
┌───────────────────────────▼──────────────────────────────────────┐
│              mac80211（net/mac80211/）或厂商驱动                  │
│  ath9k / ath11k / iwlwifi / mt76 / rtw89 / brcmfmac / ...       │
└───────────────────────────┬──────────────────────────────────────┘
                            │ MMIO / PCIe / USB / SDIO
┌───────────────────────────▼──────────────────────────────────────┐
│                      无线硬件（PHY/MAC）                          │
└──────────────────────────────────────────────────────────────────┘
```

### 1.2 `net/wireless/` 目录文件总览

| 文件 | 行数 | 职责 |
|------|------|------|
| `nl80211.c` | 21,911 | Netlink 用户接口：380+ 命令处理、事件上报 |
| `reg.c` | 4,372 | 频谱管制：CRDA 交互、频道合规、DFS |
| `scan.c` | 4,013 | 扫描管理：BSS 缓存（RB 树）、隐藏 SSID、MBSSID |
| `util.c` | 2,988 | 工具函数：IE 解析、加密套件、比特率处理 |
| `chan.c` | 1,609 | 信道定义（chandef）、DFS 验证、带宽操作 |
| `core.c` | 1,920 | 设备注册/注销、wiphy 生命周期、rfkill |
| `sme.c` | 1,622 | 软件状态机：扫描→认证→关联完整流程 |
| `mlme.c` | 1,419 | MLME 事件处理：来自驱动的管理帧结果 |
| `wext-compat.c` | 1,520 | Wireless Extensions 兼容层（遗留 API） |
| `wext-core.c` | 1,219 | WExt 核心（SIOCGIWXXX ioctl 处理） |
| `pmsr.c` | 668 | 对等测量（FTM/RTT 测距） |
| `core.h` | 637 | 内部数据结构（不对外暴露） |
| `ibss.c` | 496 | Ad-hoc 网络（IBSS 模式） |
| `radiotap.c` | 370 | Radiotap 头解析（Monitor 模式） |
| `debugfs.c` | 304 | debugfs 调试节点 |
| `mesh.c` | 289 | 802.11s Mesh 网络 |
| `sysfs.c` | 181 | sysfs 属性节点（`/sys/class/net/<dev>/wireless`）|
| `ap.c` | 74 | AP/GO 模式操作辅助 |
| `ocb.c` | 65 | OCB（Outside Context of BSS，车联网 DSRC） |
| `of.c` | 138 | 设备树（DT）属性解析 |
| `trace.h` | 4,231 | ftrace 追踪点定义（覆盖全部子系统） |
| `rdev-ops.h` | 1,587 | 驱动操作封装宏（含锁保护和 trace） |

---

## 2. 核心数据结构

### 2.1 `struct cfg80211_registered_device`（`core.h:31`）

系统中每个无线物理设备（wiphy）在内核的**完整注册对象**，是 cfg80211 框架的中枢：

```c
struct cfg80211_registered_device {
    const struct cfg80211_ops *ops;   /* 驱动操作表指针（不变，注册时绑定） */
    struct list_head list;            /* 挂入全局 cfg80211_rdev_list（RTNL 保护） */

    /* rfkill（硬件/软件飞行模式） */
    struct rfkill_ops rfkill_ops;
    struct work_struct rfkill_block;

    /* 管制域信息 */
    char country_ie_alpha2[2];        /* 最近收到的 Country IE 国家码 */
    const struct ieee80211_regdomain *requested_regd; /* 驱动自管制域 */
    enum environment_cap env;         /* 室内/室外/通用 */

    /* 设备标识 */
    int wiphy_idx;                    /* 全局唯一索引（ida 分配） */
    int devlist_generation;           /* 版本号，用于检测 wdev 列表变化 */
    int opencount;                    /* 已打开接口计数 */

    /* BSS 缓存（扫描结果存储） */
    spinlock_t bss_lock;              /* 保护 bss_list 和 bss_tree */
    struct list_head bss_list;        /* 线性链表（按时间顺序，用于过期清理） */
    struct rb_root bss_tree;          /* 红黑树（按 BSSID+频道索引，O(log n) 查找）*/
    u32 bss_generation;               /* BSS 缓存版本号（dump 时检测变化） */
    u32 bss_entries;                  /* 当前 BSS 条目数 */

    /* 扫描状态 */
    struct cfg80211_scan_request_int *scan_req;  /* 当前活跃扫描请求 */
    struct cfg80211_scan_request_int *int_scan_req; /* SME 内部触发的扫描 */
    struct sk_buff *scan_msg;         /* 预建的扫描完成 skb */
    struct list_head sched_scan_req_list; /* 调度扫描请求链表 */
    struct wiphy_work scan_done_wk;   /* 扫描完成工作项（wiphy_work 机制）*/

    /* 连接事件处理 */
    struct work_struct conn_work;     /* SME 连接状态机工作队列 */
    struct work_struct event_work;    /* wdev 事件队列处理 */

    /* DFS（动态频率选择）*/
    struct delayed_work dfs_update_channels_wk; /* CAC 完成后更新频道状态 */
    struct wireless_dev *background_radar_wdev;  /* 后台雷达监测接口 */
    struct cfg80211_chan_def background_radar_chandef;
    struct delayed_work background_cac_done_wk;
    struct work_struct background_cac_abort_wk;
    struct cfg80211_chan_def radar_chandef;
    struct work_struct propagate_radar_detect_wk;

    /* 管理帧注册 */
    struct list_head beacon_registrations;    /* 信标帧监听者列表 */
    spinlock_t beacon_registrations_lock;
    struct work_struct mgmt_registrations_update_wk;
    spinlock_t mgmt_registrations_lock;

    /* wiphy_work 调度机制（轻量级工作队列）*/
    struct work_struct wiphy_work;
    struct list_head wiphy_work_list;
    spinlock_t wiphy_work_lock;
    bool suspended;

    /* 其他 */
    u64 cookie_counter;              /* 单调递增 cookie（mgmt_tx/pmsr 标识）*/
    struct cfg80211_coalesce *coalesce; /* 数据包合并配置 */

    /* 必须最后（wiphy_priv 指针运算依赖位置） */
    struct wiphy wiphy __aligned(NETDEV_ALIGN);
};
```

### 2.2 `struct wiphy`（`include/net/cfg80211.h`）

物理无线设备的**能力与配置描述**，对驱动和用户空间均可见：

```c
struct wiphy {
    struct mutex mtx;                /* 保护 wdev_list 和各类配置（主锁） */
    u8 perm_addr[ETH_ALEN];          /* 永久 MAC 地址 */
    u8 addr_mask[ETH_ALEN];          /* 多接口 MAC 地址掩码 */

    /* 接口能力 */
    u16 n_addresses;
    struct ieee80211_txrx_stypes mgmt_stypes; /* 每接口类型可注册的管理帧子类型 */
    const struct ieee80211_iface_combination *iface_combinations;
    int n_iface_combinations;         /* 支持的接口组合（AP+STA 共存等）*/
    u16 software_iftypes;             /* 纯软件接口类型位图 */
    u16 interface_modes;              /* 支持的接口模式（STATION/AP/MONITOR/MESH 等）*/

    /* 频段与频道 */
    struct ieee80211_supported_band *bands[NUM_NL80211_BANDS];
    /* bands[0]: 2.4GHz, bands[1]: 5GHz, bands[2]: 60GHz, bands[3]: 6GHz */

    /* 链接（wdev）管理 */
    struct list_head wdev_list;       /* 虚拟接口链表（struct wireless_dev）*/
    int n_priv_action;                /* vendor 私有命令数量 */

    /* 密码学能力 */
    const u32 *cipher_suites;         /* 支持的加密套件（CCMP/TKIP/GCMP 等）*/
    int n_cipher_suites;

    /* 功率管理 */
    const struct wiphy_wowlan_support *wowlan; /* Wake-on-WLAN 能力描述 */
    struct cfg80211_wowlan *wowlan_config;     /* 当前 WoWLAN 配置 */

    /* 信号类型 */
    enum cfg80211_signal_type signal_type;     /* MBM（毫 dBm）/ UNSPEC / NONE */

    /* 管制域 */
    u32 regulatory_flags;             /* REGULATORY_CUSTOM_REG 等标志 */
    u8 regd_src;                      /* 当前管制域来源 */

    /* 特性标志（两个超长位图）*/
    u32 features;                     /* NL80211_FEATURE_* 基础特性 */
    u8 ext_features[DIV_ROUND_UP(NUM_NL80211_EXT_FEATURES, 8)]; /* 扩展特性 */

    /* 扫描能力 */
    u8 max_scan_ssids;                /* 一次扫描最多探测的 SSID 数 */
    u16 max_scan_ie_len;              /* 探针请求 IE 最大长度 */
    u8 max_sched_scan_reqs;           /* 最多并发调度扫描请求数 */

    /* 其他 */
    const char *fw_version;           /* 固件版本（sysfs 暴露）*/
    const struct wiphy_iftype_akm_suites *iftype_akm_suites; /* AKM 套件 */
    void *priv __aligned(NETDEV_ALIGN); /* 驱动私有数据（紧随 wiphy 末尾）*/
};
```

关键设计：`wiphy_priv(wiphy)` 返回 `wiphy->priv`，驱动在此存储硬件私有数据（如 `struct ath10k *`）。

### 2.3 `struct wireless_dev`（`include/net/cfg80211.h`）

虚拟无线接口，对应一个逻辑网络接口（`wlan0`、`p2p0` 等）：

```c
struct wireless_dev {
    struct wiphy *wiphy;              /* 所属物理设备 */
    enum nl80211_iftype iftype;       /* STATION / AP / MONITOR / MESH_POINT / P2P_GO / NAN 等 */
    struct list_head list;            /* 挂入 wiphy->wdev_list */
    struct net_device *netdev;        /* 关联的网络设备（NULL 表示纯虚拟接口）*/
    u64 identifier;                   /* wdev 全局唯一 ID（wdev_id() 函数使用）*/

    /* 连接状态（STA 模式）*/
    struct list_head event_list;      /* 待处理事件队列（cfg80211_event）*/
    spinlock_t event_lock;
    struct cfg80211_conn *conn;       /* SME 连接状态机实例 */
    struct cfg80211_cached_keys *connect_keys; /* 关联时缓存的 WEP 密钥 */

    /* 管理帧注册 */
    struct list_head mgmt_registrations; /* 管理帧子类型监听者链表 */
    u8 mgmt_registrations_need_update:1;

    /* MLO（Multi-Link Operation，802.11be）*/
    struct {
        struct cfg80211_internal_bss *bss;  /* 关联的 BSS */
        struct ieee80211_channel *pub_chan;
        /* ... 其他 per-link 状态 */
    } links[IEEE80211_MLD_MAX_NUM_LINKS];   /* 多链接数组（最多 15 条） */
    u16 valid_links;                  /* 有效链接位图 */
    u8 addr[ETH_ALEN];               /* 接口 MAC 地址 */

    /* 信道信息 */
    struct cfg80211_chan_def preset_chandef; /* 预设信道定义（AP 启动前设置）*/

    /* CQM（连接质量监控）*/
    struct cfg80211_cqm_config __rcu *cqm_config;
    struct wiphy_work cqm_rssi_work;

    /* P2P */
    struct cfg80211_p2p_ps p2p_ps;

    /* 状态标志 */
    unsigned long unprot_beacon_reported; /* 上次未保护信标报告时间 */
};
```

### 2.4 `struct cfg80211_internal_bss`（`core.h:186`）

BSS 缓存条目，扫描结果的内核内部表示：

```c
struct cfg80211_internal_bss {
    struct list_head list;        /* 挂入 rdev->bss_list（时间顺序） */
    struct list_head hidden_list; /* 隐藏 SSID 相同 BSSID 的链表 */
    struct rb_node rbn;           /* RB 树节点（BSSID+频道索引）*/
    unsigned long ts;             /* 最后更新时间戳（jiffies）*/
    unsigned long refcount;       /* 软件引用计数 */
    atomic_t hold;                /* 硬引用（防止当前关联的 BSS 被回收）*/
    u64 parent_tsf;               /* 相对于父 BSS 的 TSF 时间 */
    u8 parent_bssid[ETH_ALEN];
    enum bss_source_type bss_source; /* DIRECT / MBSSID / STA_PROFILE */

    /* 必须最后（驱动私有数据紧随其后）*/
    struct cfg80211_bss pub;      /* 公共 BSS 信息（对外暴露给驱动/用户空间）*/
};
```

`pub` 内含：

```c
struct cfg80211_bss {
    struct ieee80211_channel *channel;
    u8 bssid[ETH_ALEN];
    s32 signal;                    /* 信号强度（单位由 wiphy->signal_type 决定）*/
    struct cfg80211_bss_ies __rcu *ies;          /* 最新 IE（信标或探针响应）*/
    struct cfg80211_bss_ies __rcu *beacon_ies;   /* 信标 IE */
    struct cfg80211_bss_ies __rcu *proberesp_ies;/* 探针响应 IE */
    struct cfg80211_bss *hidden_beacon_bss;       /* 指向隐藏 SSID 信标 BSS */
    struct cfg80211_bss *transmitted_bss;         /* MBSSID 中的发射 BSS */
    u64 ts_boottime;              /* 接收时的启动时间（CLOCK_BOOTTIME）*/
    u32 use_for;                  /* NL80211_BSS_USE_FOR_* 标志 */
    u8 priv[] __aligned(sizeof(void *)); /* 驱动私有数据 */
};
```

### 2.5 `struct cfg80211_ops`（`include/net/cfg80211.h`，驱动必须实现）

驱动与 cfg80211 的**唯一接口**，150+ 函数指针：

```c
struct cfg80211_ops {
    /* === 接口管理 === */
    int (*add_virtual_intf)(wiphy, name, net, type, params, wdev*);
    int (*del_virtual_intf)(wiphy, wdev);
    int (*change_virtual_intf)(wiphy, dev, type, params);

    /* === 密钥管理 === */
    int (*add_key)(wiphy, dev, link_id, key_idx, pairwise, mac, params);
    int (*del_key)(wiphy, dev, link_id, key_idx, pairwise, mac);
    int (*set_default_key)(wiphy, dev, link_id, key_idx, unicast, mcast);
    int (*set_default_mgmt_key)(wiphy, dev, link_id, key_idx);

    /* === 扫描 === */
    int  (*scan)(wiphy, request);
    void (*abort_scan)(wiphy, wdev);
    int  (*sched_scan_start)(wiphy, dev, request);
    int  (*sched_scan_stop)(wiphy, dev, req_id);

    /* === 认证/关联（MLME 模式，mac80211 驱动用）=== */
    int (*auth)(wiphy, dev, req);
    int (*assoc)(wiphy, dev, req, extack);
    int (*deauth)(wiphy, dev, req);
    int (*disassoc)(wiphy, dev, req);

    /* === 连接（SME 模式，cfg80211 接管状态机）=== */
    int (*connect)(wiphy, dev, params);
    int (*update_connect_params)(wiphy, dev, params, flags);
    int (*disconnect)(wiphy, dev, reason_code);

    /* === AP 模式 === */
    int  (*start_ap)(wiphy, dev, settings);
    int  (*change_beacon)(wiphy, dev, info);
    int  (*stop_ap)(wiphy, dev, link_id);
    int  (*add_station)(wiphy, dev, mac, params);
    int  (*del_station)(wiphy, dev, params);
    int  (*change_station)(wiphy, dev, mac, params);
    int  (*get_station)(wiphy, dev, mac, sinfo);
    int  (*dump_station)(wiphy, dev, idx, mac, sinfo);

    /* === 管理帧 === */
    int  (*mgmt_tx)(wiphy, wdev, params, cookie);
    void (*mgmt_tx_cancel_wait)(wiphy, wdev, cookie);
    int  (*update_mgmt_frame_registrations)(wiphy, wdev, upd);
    void (*frame_wait_cancel)(wiphy, wdev, cookie);

    /* === IBSS / OCB / Mesh === */
    int (*join_ibss)(wiphy, dev, params);
    int (*leave_ibss)(wiphy, dev);
    int (*join_ocb)(wiphy, dev, setup);
    int (*leave_ocb)(wiphy, dev);
    int (*join_mesh)(wiphy, dev, conf, setup);
    int (*leave_mesh)(wiphy, dev);

    /* === 电源管理 === */
    int (*set_power_mgmt)(wiphy, dev, enabled, timeout);
    int (*set_wakeup)(wiphy, enabled);
    int (*suspend)(wiphy, wow);
    int (*resume)(wiphy);

    /* === DFS / 雷达 === */
    int  (*start_radar_detection)(wiphy, dev, chandef, cac_time_ms, link_id);
    void (*end_cac)(wiphy, dev, link_id);
    int  (*set_radar_background)(wiphy, chandef);

    /* === 调度器 / QoS === */
    int (*set_noack_map)(wiphy, dev, noack_map);
    int (*set_txq_params)(wiphy, dev, params);
    int (*get_txq_stats)(wiphy, wdev, stats);

    /* === 能力查询 === */
    int (*get_channel)(wiphy, wdev, link_id, chandef);
    int (*get_antenna)(wiphy, tx_ant, rx_ant);
    int (*set_antenna)(wiphy, tx_ant, rx_ant);

    /* === P2P / NAN === */
    int (*start_p2p_device)(wiphy, wdev);
    void (*stop_p2p_device)(wiphy, wdev);
    int (*start_nan)(wiphy, wdev, conf);
    void (*stop_nan)(wiphy, wdev);

    /* === MLO（802.11be 多链接）=== */
    int (*assoc_ml_reconf)(wiphy, dev, req);

    /* === 其他 === */
    int (*set_wiphy_params)(wiphy, changed);  /* WIPHY_PARAM_FRAG_THRESHOLD 等 */
    int (*get_survey)(wiphy, idx, survey);
    int (*dump_survey)(wiphy, dev, idx, survey);
    int (*set_pmksa)(wiphy, dev, pmksa);
    int (*del_pmksa)(wiphy, dev, pmksa);
    int (*flush_pmksa)(wiphy, dev);
    int (*set_cqm_rssi_config)(wiphy, dev, rssi_thold, rssi_hyst);
    int (*start_pmsr)(wiphy, wdev, req);
    void (*abort_pmsr)(wiphy, wdev, req);
    void (*channel_switch)(wiphy, dev, params);
    /* ... 共 150+ 回调 */
};
```

### 2.6 事件类型（`core.h:260`）

驱动异步上报给 cfg80211 的事件，由 `event_work` 工作队列串行处理：

```c
enum cfg80211_event_type {
    EVENT_CONNECT_RESULT,   /* 连接结果（成功/失败）*/
    EVENT_ROAMED,           /* 漫游完成 */
    EVENT_DISCONNECTED,     /* 断连（本地发起或 AP 发起）*/
    EVENT_IBSS_JOINED,      /* 加入 IBSS 成功 */
    EVENT_STOPPED,          /* 接口被停止 */
    EVENT_PORT_AUTHORIZED,  /* 802.1X 端口授权完成（4 次握手成功）*/
};

struct cfg80211_event {
    struct list_head list;        /* 挂入 wdev->event_list */
    enum cfg80211_event_type type;
    union {
        struct cfg80211_connect_resp_params cr; /* 连接响应参数 */
        struct cfg80211_roam_info rm;            /* 漫游信息 */
        struct { u16 reason; bool locally_generated; } dc; /* 断连 */
        struct { u8 bssid[ETH_ALEN]; struct ieee80211_channel *channel; } ij;
        struct { u8 peer_addr[ETH_ALEN]; /* td_bitmap */; } pa;
    };
};
```

---

## 3. nl80211 Netlink 接口层

### 3.1 Generic Netlink 注册

```c
/* nl80211.c 中的 genl_family 注册 */
static struct genl_family nl80211_fam = {
    .name       = NL80211_GENL_NAME,     /* "nl80211" */
    .version    = 1,
    .maxattr    = NUM_NL80211_ATTR - 1,
    .policy     = nl80211_policy,        /* NLA 策略数组（属性类型检查）*/
    .netnsok    = true,                  /* 支持网络命名空间 */
    .ops        = nl80211_ops,           /* 命令处���器数组 */
    .n_ops      = ARRAY_SIZE(nl80211_ops),
    .mcgrps     = nl80211_mcgrps,       /* 多播组 */
    .n_mcgrps   = ARRAY_SIZE(nl80211_mcgrps),
};
```

### 3.2 多播组（事件分发）

| 组名 | 用途 |
|------|------|
| `NL80211_MCGRP_CONFIG` | wiphy 配置变化、接口创建/删除 |
| `NL80211_MCGRP_SCAN` | 扫描启动/完成、调度扫描结果 |
| `NL80211_MCGRP_REGULATORY` | 管制域变化、频道可用性变化 |
| `NL80211_MCGRP_MLME` | 连接/断连/漫游/认证/管理帧 |
| `NL80211_MCGRP_VENDOR` | 厂商自定义事件 |
| `NL80211_MCGRP_NAN` | NAN（Neighbor Awareness Networking）事件 |

### 3.3 核心命令分类

```
设备/PHY 管理：
  NL80211_CMD_GET_WIPHY        → nl80211_get_wiphy()
  NL80211_CMD_SET_WIPHY        → nl80211_set_wiphy()
  NL80211_CMD_NEW_INTERFACE    → nl80211_new_interface()
  NL80211_CMD_DEL_INTERFACE    → nl80211_del_interface()
  NL80211_CMD_GET_INTERFACE    → nl80211_get_interface()

扫描：
  NL80211_CMD_TRIGGER_SCAN     → nl80211_trigger_scan()
  NL80211_CMD_ABORT_SCAN       → nl80211_abort_scan()
  NL80211_CMD_GET_SCAN         → nl80211_dump_scan()
  NL80211_CMD_START_SCHED_SCAN → nl80211_start_sched_scan()
  NL80211_CMD_STOP_SCHED_SCAN  → nl80211_stop_sched_scan()

认证/关联（MLME 低级接口）：
  NL80211_CMD_AUTHENTICATE     → nl80211_authenticate()
  NL80211_CMD_ASSOCIATE        → nl80211_associate()
  NL80211_CMD_DEAUTHENTICATE   → nl80211_deauthenticate()
  NL80211_CMD_DISASSOCIATE     → nl80211_disassociate()

连接（SME 高级接口）：
  NL80211_CMD_CONNECT          → nl80211_connect()
  NL80211_CMD_DISCONNECT       → nl80211_disconnect()
  NL80211_CMD_UPDATE_CONNECT_PARAMS → nl80211_update_connect_params()

AP 模式：
  NL80211_CMD_START_AP         → nl80211_start_ap()
  NL80211_CMD_SET_BEACON       → nl80211_set_beacon()
  NL80211_CMD_STOP_AP          → nl80211_stop_ap()

管理帧：
  NL80211_CMD_REGISTER_FRAME   → nl80211_register_mgmt()
  NL80211_CMD_FRAME            → nl80211_tx_mgmt()

Station 管理：
  NL80211_CMD_GET_STATION      → nl80211_get_station()
  NL80211_CMD_DUMP_STATION     → nl80211_dump_station()
  NL80211_CMD_NEW_STATION      → nl80211_new_station()
  NL80211_CMD_DEL_STATION      → nl80211_del_station()

密钥：
  NL80211_CMD_NEW_KEY          → nl80211_new_key()
  NL80211_CMD_DEL_KEY          → nl80211_del_key()
  NL80211_CMD_GET_KEY          → nl80211_get_key()

管制域：
  NL80211_CMD_REG_CHANGE       → （事件，由 reg.c 发出）
  NL80211_CMD_WIPHY_REG_CHANGE → （事件，per-wiphy 变化）

对等测量：
  NL80211_CMD_PEER_MEASUREMENT_START → nl80211_pmsr_start()
```

### 3.4 命令调用路径

```
用户空间 send(nl_sock, NL80211_CMD_TRIGGER_SCAN)
    ↓ Netlink → genl_rcv() → genl_rcv_msg()
nl80211_trigger_scan(skb, info)            [nl80211.c]
    ↓ 解析 NLA_POLICY 验证的 attrs
    ↓ 获取 rdev 和 wdev（wiphy_to_rdev + wdev_lookup）
    ↓ 构建 cfg80211_scan_request
    ↓ 调用 nl80211_check_scan_flags()
    ↓
cfg80211_scan(rdev)                        [core.c]
    ↓ rdev_scan(rdev, request)
    ↓
rdev->ops->scan(wiphy, request)            [驱动实现]
    （通过 rdev-ops.h 中的封装宏，含 trace 和锁检查）
```

### 3.5 `rdev-ops.h` 封装机制（`rdev-ops.h:1587 行`）

每个驱动操作都被包装成：

```c
static inline int rdev_scan(struct cfg80211_registered_device *rdev,
                             struct cfg80211_scan_request *request)
{
    struct wiphy *wiphy = &rdev->wiphy;
    int ret;
    trace_rdev_scan(wiphy, request);          /* ftrace 入口追踪 */
    ret = rdev->ops->scan(&rdev->wiphy, request);
    trace_rdev_return_int(wiphy, ret);        /* ftrace 返回追踪 */
    return ret;
}
```

所有 150+ ops 均有对应的 `rdev_*` 封装，确保：
1. 每次驱动调用可被 ftrace 追踪（`trace.h` 中定义）
2. 调用前可插入 `lockdep_assert_held(&rdev->wiphy.mtx)` 等检查

---

## 4. 扫描子系统（`scan.c`）

### 4.1 数据结构关系

```
rdev
├─ bss_list ─────────────────────────────── 线性链表（时间顺序）
│                                              用于过期清理（cfg80211_bss_expire）
└─ bss_tree ─────────────────────────────── RB 树
        按 (channel + bssid) 索引
        O(log n) 查找特定 BSS

    cfg80211_internal_bss
    ├─ pub.bssid
    ├─ pub.channel
    ├─ pub.signal
    ├─ pub.ies ──── RCU 保护
    ├─ pub.beacon_ies
    ├─ pub.proberesp_ies
    ├─ hidden_list ── 同 bssid 的隐藏 SSID 信标 BSS 链
    └─ pub.transmitted_bss ── MBSSID 发射 BSS 指针

    隐藏 SSID 设计：
    ├─ 隐藏信标（SSID len=0）→ hidden_beacon_bss
    └─ 探针响应（SSID 可见）→ pub BSS，hidden_list 链接到信标
```

### 4.2 BSS 更新流程

```
驱动接收信标/探针响应
    ↓
cfg80211_inform_bss_frame_data(wiphy, data, mgmt, len, gfp)
    ↓
cfg80211_inform_single_bss_frame_data()
    ↓ 解析 IE，提取 SSID、支持速率、RSN 等信息
cfg80211_bss_update(rdev, tmp, signal_valid, ts)
    ↓
    spin_lock_bh(&rdev->bss_lock)
    ├─ rb_find_bss()          /* 在 RB 树中查找已有条目 */
    ├─ 若已存在：更新 ies、信号强度、时间戳
    │   ↓ cfg80211_update_known_bss()
    │       ├─ RCU 替换 ies 指针
    │       └─ 更新隐藏 SSID 链接
    └─ 若不存在：分配新 cfg80211_internal_bss
        ├─ rb_insert_bss()    /* 插入 RB 树 */
        └─ list_add_tail()    /* 加入线性链表 */
    spin_unlock_bh
```

### 4.3 BSS 生命周期

```
创建  →  cfg80211_bss_update()
引用  →  cfg80211_ref_bss() / cfg80211_hold_bss()
使用  →  驱动/SME 通过 pub 指针访问 IE 和信号强度
释放  →  cfg80211_put_bss() → 引用计数 -1
过期  →  cfg80211_bss_expire()（30 秒定时触发）
         ├─ hold == 0 且 ts 超过 30 秒 → 从树和链表中删除
         └─ bss_free()：RCU kfree_rcu(ies)，kfree(bss)
```

**MBSSID 支持**：一个物理 AP 可广播多个虚拟 SSID（MBSSID IE），驱动调用 `cfg80211_inform_bss_frame_data()` 时，框架自动解析 MBSSID 并为每个非发射 BSS 创建独立条目，`bss_source = BSS_SOURCE_MBSSID`。

**MLO/STA Profile 支持**：802.11be 多链接设备，每个链接的 BSS 由 `BSS_SOURCE_STA_PROFILE` 标记。

### 4.4 调度扫描（Scheduled Scan）

```
用户空间配置计划任务（如每 30 秒扫描一次）：
  nl80211_start_sched_scan()
    ↓
  cfg80211_sched_scan_req_possible()   /* 检查并发限制 */
    ↓
  rdev->ops->sched_scan_start()        /* 驱动启动周期扫描 */

驱动发现新结果时：
  cfg80211_sched_scan_results(wiphy, reqid)
    ↓ schedule_work(&rdev->sched_scan_res_wk)
    ↓ cfg80211_sched_scan_results_wk()
        → nl80211_send_sched_scan(req, NL80211_CMD_SCHED_SCAN_RESULTS)
```

---

## 5. MLME 与 SME 子系统

### 5.1 两种驱动接入模式

```
模式一：MLME 低级接口（mac80211 驱动使用）
  用户空间直接发送认证/关联命令
  驱动（mac80211）自己维护 802.11 状态机
  cfg80211 仅做参数验证和事件转发

  用户空间 NL80211_CMD_AUTHENTICATE
    ↓ nl80211_authenticate()
    ↓ cfg80211_mlme_auth(rdev, dev, req)   [mlme.c]
    ↓ rdev_auth(rdev, dev, req)
    ↓ ops->auth(wiphy, netdev, req)         [驱动执行]

模式二：SME 高级接口（简单驱动使用）
  用户空间发送 NL80211_CMD_CONNECT
  cfg80211 内部 SME 状态机完成扫描→认证→关联全流程
  驱动只需实现 ops->connect() 一个函数

  用户空间 NL80211_CMD_CONNECT
    ↓ nl80211_connect()
    ↓ cfg80211_connect(rdev, dev, params, ...)   [sme.c]
    ↓ 若驱动有 ops->connect：直接调用
      否则：cfg80211_sme_connect()  启动内部状态机
```

### 5.2 SME 状态机（`sme.c`）

```c
enum cfg80211_conn_state {
    CFG80211_CONN_SCANNING,           /* 触发扫描，寻找目标 AP */
    CFG80211_CONN_SCAN_AGAIN,         /* 扫描无结果，再次扫描 */
    CFG80211_CONN_AUTHENTICATE_NEXT,  /* 即将发送认证请求 */
    CFG80211_CONN_AUTHENTICATING,     /* 等待认证响应 */
    CFG80211_CONN_AUTH_FAILED_TIMEOUT,/* 认证超时 */
    CFG80211_CONN_ASSOCIATE_NEXT,     /* 即将发送关联请求 */
    CFG80211_CONN_ASSOCIATING,        /* 等待关联响应 */
    CFG80211_CONN_ASSOC_FAILED,       /* 关联失败 */
    CFG80211_CONN_ASSOC_FAILED_TIMEOUT,
    CFG80211_CONN_DEAUTH,             /* 发送去认证 */
    CFG80211_CONN_ABANDON,            /* 放弃连接 */
    CFG80211_CONN_CONNECTED,          /* 连接成功 */
};
```

**状态迁移（`conn_work` 工作队列驱动）**：

```
cfg80211_sme_connect()
  → 初始化 conn->state = SCANNING
  → schedule_work(&rdev->conn_work)

cfg80211_conn_work()                           [sme.c, conn_work 处理函数]
  → 遍历所有 wdev（持 wiphy->mtx）
  → cfg80211_conn_do_work(wdev, &quit_now)
      switch (conn->state):
      SCAN_AGAIN:
        → cfg80211_conn_scan(wdev)             触发内部扫描
        → state = SCANNING
      AUTHENTICATE_NEXT:
        → cfg80211_mlme_auth(rdev, ...)        发送认证帧
        → state = AUTHENTICATING
      ASSOCIATE_NEXT:
        → cfg80211_mlme_assoc(rdev, ...)       发送关联帧
        → state = ASSOCIATING
      DEAUTH:
        → cfg80211_mlme_deauth(rdev, ...)      发送去认证
      ABANDON:
        → __cfg80211_connect_result(status=ASSOC_FAILED)
        → 清理 conn，通知用户空间

扫描完成 → cfg80211_sme_scan_done(dev)
  → 查找最佳匹配 BSS
  → conn->state = AUTHENTICATE_NEXT
  → schedule_work(&rdev->conn_work)

认证响应 → cfg80211_sme_rx_auth(wdev, buf, len)
  → 解析 Authentication frame
  → 若成功：state = ASSOCIATE_NEXT
  → 若失败：state = DEAUTH 或 SCAN_AGAIN（换算法重试）
  → schedule_work(&rdev->conn_work)

关联响应 → cfg80211_sme_rx_assoc_resp(wdev, status)
  → 若 success：调用 __cfg80211_connect_result(CONNECTED)
  → 若 failure：state = ASSOC_FAILED
```

### 5.3 MLME 事件处理（`mlme.c`）

驱动主动上报的事件（无论哪种接入模式）：

```
cfg80211_rx_assoc_resp(dev, resp_data)     [驱动调用]
  → 解析 AssocResp 帧 IE
  → cfg80211_sme_rx_assoc_resp(wdev, status)    [若 SME 模式]
  → 加入 wdev->event_list
  → schedule_work(&rdev->event_work)
  → cfg80211_process_wdev_events(wdev)
      → __cfg80211_connect_result(dev, params, wextev=true)
          → 更新 wdev->links[link].bss
          → nl80211_send_connect_result(rdev, dev, params, GFP_KERNEL)
          → cfg80211_upload_connect_keys(wdev)    上传缓存密钥到驱动

cfg80211_disconnected(dev, reason, ie, len, local)  [驱动调用]
  → 加入 wdev->event_list
  → schedule_work(&rdev->event_work)
      → __cfg80211_disconnected(dev, ie, len, reason, from_ap)
          → 清理 wdev 的连接状态
          → nl80211_send_disconnected(rdev, dev, reason, ie, len, from_ap)

cfg80211_roamed(dev, roam_info)              [驱动调用，漫游后]
  → 加入 event_list
  → schedule_work(&rdev->event_work)
      → __cfg80211_roamed(wdev, info)
          → 更新 wdev->links[link].bss
          → nl80211_send_roamed(rdev, dev, info, GFP_KERNEL)
```

---

## 6. 频谱管制子系统（`reg.c`）

### 6.1 核心数据结构

```c
struct ieee80211_regdomain {
    u32 n_reg_rules;
    char alpha2[3];                      /* ISO 3166 国家码 */
    enum nl80211_dfs_regions dfs_region; /* ETSI / FCC / JP / KR 等 */
    struct ieee80211_reg_rule reg_rules[]; /* 频率规则数组 */
};

struct ieee80211_reg_rule {
    struct ieee80211_freq_range freq_range; /* 频率范围（kHz） */
    struct ieee80211_power_rule power_rule; /* 功率限制（mBm） */
    u32 flags;    /* NL80211_RRF_NO_OFDM / NO_IR / DFS / AUTO_BW 等 */
    u32 dfs_cac_ms; /* DFS CAC 等待时间（毫秒）*/
};

struct ieee80211_freq_range {
    u32 start_freq_khz;
    u32 end_freq_khz;
    u32 max_bandwidth_khz;
};

struct ieee80211_power_rule {
    u32 max_antenna_gain;  /* 最大天线增益（mBi）*/
    u32 max_eirp;          /* 最大等效全向辐射功率（mBm）*/
};
```

### 6.2 管制域来源优先级

```
优先级（从低到高）：
  NL80211_REGDOM_SET_BY_CORE          内核内置 world regulatory domain
  NL80211_REGDOM_SET_BY_USER          用户手动设置（iw reg set CN）
  NL80211_REGDOM_SET_BY_DRIVER        驱动调用 regulatory_hint()
  NL80211_REGDOM_SET_BY_COUNTRY_IE   关联后收到 Country IE
```

**交集计算**：当多个来源存在时，取最严格的规则（最小功率 + 最小带宽 + 最多禁止标志），确保合规。

### 6.3 DFS（动态频率选择）状态机

```
频道状态枚举：
  NL80211_DFS_USABLE     → 可用（未使用，无 DFS 要求）
  NL80211_DFS_UNAVAILABLE → 不可用（雷达检测到）
  NL80211_DFS_AVAILABLE  → 已完成 CAC，当前可用

                CAC 开始
USABLE ──────────────────────────────► AVAILABLE
  ▲         CAC 期间（60-600 秒）            │
  │                                           │
  │    雷达检测到           30 分钟后恢复     │
  └──────────────────── UNAVAILABLE ◄─────────┘

触发流程：
  rdev->ops->start_radar_detection(wiphy, wdev, chandef, cac_time_ms)
    ↓ （CAC 期间禁止 TX）
  cfg80211_cac_event(wdev, NL80211_RADAR_CAC_FINISHED)
    ↓
  cfg80211_set_dfs_state(AVAILABLE)
  nl80211_radar_notify(NL80211_RADAR_CAC_FINISHED)

雷达检测到（AP 运行中）：
  cfg80211_radar_event(wiphy, chandef, gfp)
    ↓
  cfg80211_set_dfs_state(UNAVAILABLE)
  nl80211_radar_notify(NL80211_RADAR_DETECTED)
  schedule_work(propagate_radar_detect_wk)
    ↓ 通知所有在该信道的 wdev 停止 AP
```

### 6.4 后台 CAC（Background CAC）

允许 AP 在当前信道工作的同时，在后台对候选信道执行 CAC：

```
cfg80211_start_background_radar_detection(rdev, wdev, chandef)
  → ops->set_radar_background(wiphy, chandef)
  → 记录 background_radar_wdev / background_radar_chandef

后台 CAC 完成：
  cfg80211_background_cac_done_wk()
    → cfg80211_set_dfs_state(chandef, AVAILABLE)
    → nl80211_radar_notify(NL80211_RADAR_CAC_FINISHED)
```

---

## 7. 信道管理（`chan.c`）

### 7.1 `cfg80211_chan_def` 结构

```c
struct cfg80211_chan_def {
    struct ieee80211_channel *chan;  /* 主信道（20 MHz） */
    enum nl80211_chan_width width;   /* 20/40/80/80+80/160/320 MHz */
    u32 center_freq1;               /* 中心频率1（MHz） */
    u32 center_freq2;               /* 中心频率2（仅 80+80 MHz 使用）*/
    u16 punctured;                  /* EHT 打孔位图（802.11be）*/
};
```

常见 chandef 示例：

```
2.4GHz HT40+：
  chan = channel(2437 MHz)
  width = NL80211_CHAN_WIDTH_40
  center_freq1 = 2447 MHz    （主 20MHz + 次 20MHz）

5GHz 80MHz：
  chan = channel(5180 MHz)   （channel 36）
  width = NL80211_CHAN_WIDTH_80
  center_freq1 = 5210 MHz    （36-48 的中心）

5GHz 160MHz：
  chan = channel(5180 MHz)
  width = NL80211_CHAN_WIDTH_160
  center_freq1 = 5250 MHz    （36-64 的中心）
```

### 7.2 信道可用性验证

`_cfg80211_chandef_usable(wiphy, chandef, prohibited_flags, permitting_flags)`：

1. 检查主信道是否在 wiphy 支持的频段中
2. 验证宽度是否被 wiphy 的 `iface_combination` 允许
3. 检查每个 20MHz 子信道的 `flags` 是否包含 `prohibited_flags`（如 `NO_IR`、`RADAR`）
4. 对 DFS 信道检查 `dfs_state` 是否为 `AVAILABLE`
5. 验证带宽不超过 freq_range.max_bandwidth_khz

---

## 8. 接口组合与多接口共存

### 8.1 `ieee80211_iface_combination`

驱动声明支持的接口组合：

```c
/* 示例：支持 1 个 AP + 1 个 STA 同时工作 */
static const struct ieee80211_iface_limit limits[] = {
    { .max = 1, .types = BIT(NL80211_IFTYPE_AP) },
    { .max = 1, .types = BIT(NL80211_IFTYPE_STATION) },
};

static const struct ieee80211_iface_combination combinations[] = {
    {
        .limits = limits,
        .n_limits = ARRAY_SIZE(limits),
        .max_interfaces = 2,
        .num_different_channels = 1,   /* 必须在同一信道 */
        .beacon_int_infra_match = true,
    },
};
```

### 8.2 MLO（Multi-Link Operation，802.11be）

802.11be 引入多链接，一个 STA 可同时在多个频段上关联同一 AP：

```
wdev->valid_links  位图（最多 15 个链接）
wdev->links[0]     → 2.4 GHz 链接
wdev->links[1]     → 5 GHz 链接
wdev->links[2]     → 6 GHz 链接

每个 link：
  .bss     → 对应 cfg80211_bss（各链接有独立 BSS）
  .addr    → 该链接使用的 MAC 地址

cfg80211_assoc_ml_reconf()  → MLO 链接重配置（动态增加/移除链接）
```

---

## 9. 其他重要子系统

### 9.1 P2P（Wi-Fi Direct）

```
P2P 接口类型：
  NL80211_IFTYPE_P2P_CLIENT   → P2P 客户端（加入 GO 的群组）
  NL80211_IFTYPE_P2P_GO       → P2P 群主（充当 AP）
  NL80211_IFTYPE_P2P_DEVICE   → P2P 发现设备（不收发数据）

P2P Device（/dev/p2p）：
  - 不绑定 net_device（wdev->netdev = NULL）
  - 仅用于 P2P 发现（Probe Request/Response 交换）
  - 管理帧注册：PROBE_REQ → wpa_supplicant 处理
  - 找到对等方后：建立 GO（start_ap）或 GC（connect）连接
```

### 9.2 NAN（Neighbor Awareness Networking）

近场发现协议，用于 IoT 设备间低功耗发现：

```
NL80211_CMD_START_NAN          → rdev->ops->start_nan()
NL80211_CMD_ADD_NAN_FUNCTION   → rdev->ops->add_nan_func()
NL80211_CMD_RM_NAN_FUNCTION    → rdev->ops->rm_nan_func()
NL80211_CMD_CHANGE_NAN_CONFIG  → rdev->ops->nan_change_conf()

事件（通过 NL80211_MCGRP_NAN 多播组）：
  NL80211_CMD_NAN_MATCH  → 发现匹配的 NAN 服务
```

### 9.3 对等测量 PMSR（`pmsr.c`）

FTM（Fine Timing Measurement）距离测量：

```
用户空间：NL80211_CMD_PEER_MEASUREMENT_START
  → nl80211_pmsr_start()
      → 解析目标对等方列表和测量类型（FTM/LCI/CIVIC）
      → rdev->ops->start_pmsr(wiphy, wdev, request)
      → 驱动执行 RTT 测量

驱动完成 → cfg80211_pmsr_report(wdev, req, result, gfp)
  → nl80211_send_pmsr_result()   上报测量结果
```

### 9.4 Wireless Extensions 兼容层（`wext-*.c`）

遗留 `ioctl(SIOCGIWXXX)` 接口的兼容实现，允许旧工具（`iwconfig`、`iwlist`）继续工作：

```
wext-core.c     核心 ioctl 处理（wireless_process_ioctl）
wext-compat.c   将 WExt 命令翻译为 cfg80211/nl80211 操作
wext-sme.c      WExt 连接命令转换为 cfg80211_connect()
wext-priv.c     私有 ioctl 命令支持

典型翻译：
  SIOCSIWESSID（设置 SSID）→ cfg80211_mgd_wext_connect()
  SIOCGIWSCAN（扫描结果）→ 遍历 rdev->bss_list
  SIOCSIWFREQ（设置频率）→ cfg80211_set_monitor_channel()
```

---

## 10. 同步机制与并发模型

### 10.1 锁层次（从粗到细）

```
RTNL（rtnl_lock）
  └─ 保护：cfg80211_rdev_list、wiphy/wdev 注册/注销
     nl80211 命令通常在 RTNL 下执行
  └─ 子锁：

  wiphy->mtx（per-wiphy mutex）
    └─ 保护：wdev_list、scan_req、各子系统状态
    └─ 子锁：

    rdev->bss_lock（spinlock_bh）
      └─ 保护：bss_list、bss_tree

    rdev->beacon_registrations_lock（spinlock）
      └─ 保护：beacon_registrations

    rdev->mgmt_registrations_lock（spinlock）
      └─ 保护：wdev->mgmt_registrations

    wdev->event_lock（spinlock）
      └─ 保护：wdev->event_list
```

### 10.2 RCU 使用

BSS 的 IE 数据通过 RCU 保护，允许无锁读取扫描结果：

```c
/* 读取 IE（无锁）*/
rcu_read_lock();
ies = rcu_dereference(bss->pub.ies);
/* 使用 ies */
rcu_read_unlock();

/* 更新 IE（持 bss_lock）*/
spin_lock_bh(&rdev->bss_lock);
old = rcu_dereference_protected(bss->pub.ies, lockdep_is_held(&rdev->bss_lock));
rcu_assign_pointer(bss->pub.ies, new_ies);
kfree_rcu(old, rcu_head);
spin_unlock_bh(&rdev->bss_lock);
```

### 10.3 工作队列体系

| 工作队列项 | 触发时机 | 处理函数 |
|-----------|---------|---------|
| `conn_work` | SME 状态需推进 | `cfg80211_conn_work()` |
| `event_work` | 驱动事件入队 | `cfg80211_process_rdev_events()` |
| `scan_done_wk` | 扫描完成 | `__cfg80211_scan_done()` |
| `sched_scan_res_wk` | 调度扫描有结果 | `cfg80211_sched_scan_results_wk()` |
| `dfs_update_channels_wk` | CAC 完成 | `cfg80211_dfs_channels_update_work()` |
| `mgmt_registrations_update_wk` | 管理帧监听变化 | `cfg80211_mgmt_registrations_update_wk()` |
| `wiphy_work` | wiphy_work 机制 | `cfg80211_process_wiphy_works()` |
| `destroy_work` | 设备注销延迟 | 清理资源 |

**`wiphy_work` 机制**：新引入的轻量级工作队列，在 `wiphy->mtx` 内执行，比 `work_struct` 更安全（不需要内部再加锁）。用于 CQM RSSI 监控、扫描完成等场景。

---

## 11. 完整连接流程（端到端）

### 11.1 STA 模式连接（SME 路径）

```
① 用户空间
   NetworkManager → wpa_supplicant
   NL80211_CMD_TRIGGER_SCAN（发现 AP）

② nl80211 层
   nl80211_trigger_scan()
     → 验证权限（CAP_NET_ADMIN）
     → 构建 cfg80211_scan_request
     → cfg80211_scan(rdev)

③ cfg80211 层 → 驱动
   rdev_scan(rdev, request)
     → ops->scan(wiphy, request)

④ 硬件扫描（异步）
   驱动接收信标/探针响应
     → cfg80211_inform_bss_frame_data()  更新 bss_tree
   扫描完成：
     → cfg80211_scan_done(wiphy, info)
         → schedule wiphy_work(scan_done_wk)
         → nl80211_send_scan_msg(NL80211_CMD_NEW_SCAN_RESULTS)

⑤ 用户空间收到扫描完成
   wpa_supplicant 选择最佳 AP
   NL80211_CMD_CONNECT（含 SSID、密码、AKM 套件）

⑥ nl80211 层
   nl80211_connect()
     → 构建 cfg80211_connect_params
     → cfg80211_connect(rdev, dev, params, connkeys, NULL)

⑦ cfg80211 SME
   cfg80211_sme_connect(rdev, wdev, params, connkeys)
     → 创建 wdev->conn（cfg80211_conn）
     → 查找缓存 BSS
     → 若未找到：触发内部扫描（SCANNING 状态）
     → 找到后：AUTHENTICATE_NEXT → AUTHENTICATING

⑧ 认证帧交换（通过 cfg80211_mlme_auth）
   → ops->auth(wiphy, dev, req)
   → 驱动发送 Auth Request 帧
   → 收到 Auth Response → cfg80211_sme_rx_auth()
   → ASSOCIATE_NEXT → ASSOCIATING

⑨ 关联帧交换（通过 cfg80211_mlme_assoc）
   → ops->assoc(wiphy, dev, req)
   → 驱动发送 Assoc Request 帧（含 RSN IE、HT/VHT/HE Cap）
   → 收到 Assoc Response → cfg80211_sme_rx_assoc_resp()

⑩ 连接成功
   __cfg80211_connect_result(dev, params, true)
     → wdev->links[0].bss = bss（保持 BSS 引用）
     → cfg80211_upload_connect_keys(wdev)  上传 WEP 密钥
     → nl80211_send_connect_result(NL80211_CMD_CONNECT, status=0)

⑪ 用户空间收到连接成功
    wpa_supplicant 执行 4-Way Handshake
    完成后：cfg80211_port_authorized()
          → nl80211_send_port_authorized(NL80211_CMD_PORT_AUTHORIZED)
    NetworkManager 配置 IP 地址，网络就绪
```

### 11.2 AP 模式启动流程

```
① hostapd 配置 AP 参数
   NL80211_CMD_NEW_INTERFACE（创建 AP 接口）
   NL80211_CMD_SET_WIPHY（设置信道/功率）
   NL80211_CMD_START_AP（含 beacon_interval、SSID、RSN IE 等）

② nl80211_start_ap()
   → 验证 chandef 是否合规（cfg80211_chandef_usable）
   → 验证 beacon_interval（cfg80211_validate_beacon_int）
   → 验证接口组合（cfg80211_can_use_iftype_chan）
   → rdev_start_ap(rdev, dev, settings)
       → ops->start_ap(wiphy, dev, settings)

③ 驱动启动信标
   → 硬件配置 beacon 帧内容
   → 开始周期性发送 Beacon

④ 客户端关联
   → 驱动收到 Assoc Request
   → ops 处理（nl80211 可通过 NL80211_CMD_NEW_STATION 添加 STA 记录）
   → cfg80211_new_sta(dev, mac, sinfo, gfp)
       → nl80211_send_new_sta()

⑤ 信道切换（CSA）
   NL80211_CMD_CHANNEL_SWITCH
   → ops->channel_switch(wiphy, dev, params)
   → cfg80211_ch_switch_started_notify()
   → 驱动发送 Channel Switch Announcement（CSA IE）
   → cfg80211_ch_switch_notify()
       → 更新 chandef
       → nl80211_send_ch_switch_notify()
```

---

## 12. debugfs / sysfs 接口

### 12.1 debugfs（`/sys/kernel/debug/ieee80211/`）

```
/sys/kernel/debug/ieee80211/
  phy0/
    ├── rdev/           （由 cfg80211 的 debugfs.c 创建）
    │   ├── devices
    │   └── reg
    ├── ...             （mac80211 子目录）
    └── stations/
```

`debugfs.c`（304 行）为每个 rdev 创建：
- `reg` 文件：显示当前管制域规则
- `devices` 文件：显示 wdev 列表

### 12.2 sysfs（`/sys/class/net/<dev>/wireless/`）

`sysfs.c`（181 行）暴露的属性：
- `wiphy/`：物理设备信息（索引、名称）
- 通过 `struct class ieee80211_class` 注册

### 12.3 tracing（`trace.h`，4231 行）

为每个关键函数定义 ftrace 追踪点，启用方式：

```bash
# 追踪所有 cfg80211 操作
echo 1 > /sys/kernel/debug/tracing/events/cfg80211/enable

# 查看驱动操作调用
cat /sys/kernel/debug/tracing/trace | grep rdev_scan
```

追踪点覆盖：扫描、连接、管理帧、信道操作、DFS 事件、PMK 操作等 100+ 函数。

---

## 13. 关键源文件索引

| 文件路径 | 行数 | 职责 |
|----------|------|------|
| `include/net/cfg80211.h` | ~6500 | 全量公共 API（Consumer + Provider 均依赖）|
| `include/uapi/linux/nl80211.h` | ~8000 | nl80211 协议定义（命令/属性/枚举）|
| `net/wireless/core.h` | 637 | 内部数据结构（rdev、internal_bss、event）|
| `net/wireless/nl80211.h` | 130 | nl80211 内部函数声明 |
| `net/wireless/rdev-ops.h` | 1587 | 驱动操作封装宏（含 trace）|
| `net/wireless/core.c` | 1920 | 设备注册/注销、rfkill、网络命名空间 |
| `net/wireless/nl80211.c` | 21911 | Netlink 接口实现 |
| `net/wireless/scan.c` | 4013 | BSS 缓存（RB 树）、扫描管理 |
| `net/wireless/reg.c` | 4372 | 频谱管制、DFS、CRDA 交互 |
| `net/wireless/sme.c` | 1622 | 软件状态机（SCANNING→CONNECTED）|
| `net/wireless/mlme.c` | 1419 | MLME 事件处理（来自驱动）|
| `net/wireless/chan.c` | 1609 | 信道定义与验证 |
| `net/wireless/util.c` | 2988 | IE 解析、加密套件、速率工具 |
| `net/wireless/mesh.c` | 289 | 802.11s Mesh 网络 |
| `net/wireless/ibss.c` | 496 | Ad-hoc（IBSS）网络 |
| `net/wireless/pmsr.c` | 668 | FTM/RTT 对等测量 |
| `net/wireless/trace.h` | 4231 | ftrace 追踪点定义 |
