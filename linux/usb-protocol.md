# USB 协议详细文档

> 覆盖 USB 1.1 / 2.0 / 3.x 规范
> 结合 Linux 6.19.6 内核源码（`include/uapi/linux/usb/ch9.h`）交叉验证

---

## 目录

1. [协议版本演进](#1-协议版本演进)
2. [总线拓扑与寻址](#2-总线拓扑与寻址)
3. [物理层](#3-物理层)
4. [USB 2.0 包结构](#4-usb-20-包结构)
5. [事务模型](#5-事务模型)
6. [四种传输类型](#6-四种传输类型)
7. [描述符体系](#7-描述符体系)
8. [标准设备请求（Chapter 9）](#8-标准设备请求chapter-9)
9. [设备枚举流程](#9-设备枚举流程)
10. [电源管理](#10-电源管理)
11. [USB 3.x 超速协议](#11-usb-3x-超速协议)
12. [USB Type-C](#12-usb-type-c)

---

## 1. 协议版本演进

| 版本 | 发布年份 | 速率 | 信号 | 线缆 | 代表控制器 |
|------|----------|------|------|------|-----------|
| USB 1.0 | 1996 | LS 1.5 Mbps / FS 12 Mbps | 半双工差分 | Type-A/B | — |
| USB 1.1 | 1998 | LS / FS | 半双工差分 | Type-A/B | UHCI / OHCI |
| USB 2.0 | 2000 | HS 480 Mbps（+兼容 LS/FS） | 半双工差分 | Type-A/B/Mini | EHCI |
| USB 2.0 LPM | 2007 | HS | 增加 L1 链路状态 | — | — |
| USB 3.0 | 2008 | SS 5 Gbps | 全双工 SuperSpeed | Type-A/B/Micro-B | xHCI |
| USB 3.1 Gen2 | 2013 | SS+ 10 Gbps | 全双工 SuperSpeed+ | Type-C | xHCI |
| USB 3.2 | 2017 | 20 Gbps（双通道） | 2×10 Gbps | Type-C only | xHCI |
| USB4 v1 | 2019 | 40 Gbps | Thunderbolt 3 兼容 | Type-C only | — |
| USB4 v2 | 2022 | 80 Gbps | — | Type-C only | — |

**Linux 内核速度枚举**（`include/uapi/linux/usb/ch9.h:1201`）：

```c
enum usb_device_speed {
    USB_SPEED_UNKNOWN = 0,
    USB_SPEED_LOW,        /* 1.5 Mbps，USB 1.1 */
    USB_SPEED_FULL,       /* 12  Mbps，USB 1.1 */
    USB_SPEED_HIGH,       /* 480 Mbps，USB 2.0 */
    USB_SPEED_WIRELESS,   /* 480 Mbps，Wireless USB 2.5 */
    USB_SPEED_SUPER,      /* 5   Gbps，USB 3.0 */
    USB_SPEED_SUPER_PLUS, /* 10  Gbps，USB 3.1 */
};
```

---

## 2. 总线拓扑与寻址

### 2.1 星型分层拓扑

USB 采用单主机（Host）控制的星型分层总线拓扑：

```
Host Controller
    │
    ├── Root Hub（根 Hub，集成在控制器中）
    │     ├── Port 1 ── USB Device（叶节点）
    │     ├── Port 2 ── External Hub
    │     │               ├── Port 1 ── USB Device
    │     │               └── Port 2 ── USB Device
    │     └── Port 3 ── USB Device
    └──（更多根 Hub 端口）
```

- **最大层级**：7 层（包含根 Hub 层，即外部 Hub 最多嵌套 5 级）
- **最大设备数**：127（7 位地址，地址 0 保留用于枚举）
- **Hub 的作用**：信号再生、速率转换（TT 事务转换器）、端口管理

### 2.2 设备寻址

每个设备在枚举时被分配一个 **7 位设备地址**（1–127）：

```
设备地址（7 bit）+ 端点号（4 bit）= 唯一端点标识
```

- 地址 0：保留，所有未枚举设备默认使用地址 0 响应 `SET_ADDRESS`
- 端点 0（EP0）：每个设备必须实现，用于控制传输
- 主机通过 `(设备地址, 端点号, 方向)` 三元组唯一定位一个数据管道

### 2.3 端点（Endpoint）

端点是设备侧的数据缓冲节点，是传输的最终目标：

| 属性 | 说明 |
|------|------|
| 方向 | IN（设备→主机）或 OUT（主机→设备），对 EP0 双向 |
| 编号 | 0–15，每个方向各 15 个（EP0 双向共用） |
| 类型 | Control / Bulk / Interrupt / Isochronous |
| 最大包大小 | 由端点描述符的 `wMaxPacketSize` 定义 |

---

## 3. 物理层

### 3.1 USB 2.0 物理层（D+ / D−）

USB 2.0 使用一对差分信号线传输数据：

```
连接器引脚：
  Pin 1: VBUS (+5V)
  Pin 2: D−
  Pin 3: D+
  Pin 4: GND
```

**差分信号电平**：

| 状态 | D+ | D− | 含义 |
|------|----|----|------|
| J 状态（FS/HS）| 高 | 低 | 空闲（FS）/ 数据 1 |
| K 状态（FS/HS）| 低 | 高 | 数据 0（FS）/ SOF |
| SE0（Single-Ended Zero）| 低 | 低 | 复位信号 / EOP |
| SE1 | 高 | 高 | 错误状态 |

- **FS/LS**：使用 3.3V 信号，NRZI 编码 + 位填充（bit stuffing）
- **HS（480 Mbps）**：切换到电流模式（17.78 mA），去除上拉电阻

### 3.2 NRZI 编码与位填充

USB 使用 **NRZI（Non-Return-to-Zero Inverted）** 编码：

```
数据位 1 → 电平不翻转
数据位 0 → 电平翻转
```

**位填充（Bit Stuffing）**：连续 6 个 1 之后强制插入 1 个 0，接收方检测后删除，确保信号跳变频率（时钟同步）。

### 3.3 帧与微帧

| 速度 | 帧/微帧 | 周期 | 说明 |
|------|---------|------|------|
| FS/LS | 帧（Frame） | 1 ms | SOF（Start of Frame）标记 |
| HS | 微帧（Microframe） | 125 μs | 每帧 8 个微帧 |

主机每帧/微帧发送一个 SOF 令牌包，用于同步和帧号递增。

### 3.4 USB 3.x 物理层（SuperSpeed）

USB 3.x 增加一对独立的差分对（TX/RX 各一对），实现全双工传输：

```
连接器引脚（Type-A SuperSpeed）：
  StdA_SSRX−, StdA_SSRX+  (接收差分对)
  GND_DRAIN
  StdA_SSTX−, StdA_SSTX+  (发送差分对)
```

- **编码**：USB 3.0 使用 **8b/10b** 编码（开销 20%，有效 5 Gbps × 80% = 4 Gbps）
- **USB 3.1 Gen2**：改用 **128b/132b** 编码（开销 3%，有效效率更高）
- **极性翻转**：支持 TX/RX 极性翻转，允许 PCB 走线交叉

---

## 4. USB 2.0 包结构

### 4.1 包的基本格式

每个 USB 包由以下字段组成：

```
┌──────┬────────┬──────────────────────────┬───────┐
│ SYNC │  PID   │        Payload           │  EOP  │
│ 8bit │  8bit  │     (可变长度)            │       │
└──────┴────────┴──────────────────────────┴───────┘
```

- **SYNC**：同步序列，FS/LS 为 `00000001`（7个0+1个1），HS 为 32 bit
- **PID**：包标识符（4 bit PID + 4 bit PID 补码校验）
- **EOP**（End of Packet）：FS/LS 为 SE0 持续 2 bit；HS 为特殊序列

### 4.2 PID 类型完整表

PID 字段低 4 位定义类型，高 4 位为低 4 位的按位取反：

| PID 类型 | 名称 | 值（低4位） | 用途 |
|----------|------|-------------|------|
| **Token** | OUT | 0001 | 主机→设备 OUT 事务 |
| | IN | 1001 | 主机请求设备发送 |
| | SOF | 0101 | 帧起始 |
| | SETUP | 1101 | 控制传输 Setup 阶段 |
| **Data** | DATA0 | 0011 | 数据包（DATA toggle = 0）|
| | DATA1 | 1011 | 数据包（DATA toggle = 1）|
| | DATA2 | 0111 | HS ISO 高带宽用 |
| | MDATA | 1111 | HS Split 事务 |
| **Handshake** | ACK | 0010 | 接收成功 |
| | NAK | 1010 | 接收方未就绪 |
| | STALL | 1110 | 端点暂停（错误） |
| | NYET | 0110 | HS 专用，未完成（用于 PING） |
| **Special** | PRE | 1100 | 前导（LS 设备访问） |
| | ERR | 1100 | HS Split 事务错误 |
| | SPLIT | 1000 | HS Split 事务 |
| | PING | 0100 | HS 流量控制 |
| | Reserved | 0000 | — |

### 4.3 Token 包格式

```
┌──────┬────────┬──────────┬───────────┬───────┐
│ SYNC │  PID   │  ADDR    │  ENDP     │  CRC5 │
│ 8bit │  8bit  │  7 bit   │  4 bit    │ 5 bit │
└──────┴────────┴──────────┴───────────┴───────┘
        共 24 bit payload（不含 SYNC/EOP）
```

- **ADDR**：7 位设备地址（0–127）
- **ENDP**：4 位端点号（0–15）
- **CRC5**：对 ADDR + ENDP 的 5 位 CRC 校验

**SOF 包**（特殊 Token）：
```
┌──────┬────────┬───────────────┬───────┐
│ SYNC │  PID   │  Frame Number │  CRC5 │
│ 8bit │  8bit  │    11 bit     │ 5 bit │
└──────┴────────┴───────────────┴───────┘
```
Frame Number 范围 0–2047，每帧递增，溢出归零。

### 4.4 Data 包格式

```
┌──────┬────────┬───────────────────────┬────────┐
│ SYNC │  PID   │  Data Payload         │ CRC16  │
│ 8bit │  8bit  │  0 ~ MaxPacketSize    │ 16 bit │
└──────┴────────┴───────────────────────┴────────┘
```

- **CRC16**：CCITT 多项式 G(x) = x¹⁶ + x¹⁵ + x² + 1，覆盖所有数据字节
- **最大包大小（wMaxPacketSize）**：

| 传输类型 | LS | FS | HS |
|----------|----|----|-----|
| Control EP0 | 8 | 8/16/32/64 | 64 |
| Bulk | — | 8/16/32/64 | 512 |
| Interrupt | ≤8 | ≤64 | ≤1024 |
| Isochronous | — | ≤1023 | ≤1024 |

### 4.5 Handshake 包格式

Handshake 包只有 PID，无 Payload：

```
┌──────┬────────┐
│ SYNC │  PID   │
│ 8bit │  8bit  │
└──────┴────────┘
```

- **ACK**：接收方成功接收，无 CRC 错误，缓冲区就绪
- **NAK**：设备未就绪（IN：无数据；OUT：缓冲区满）→ 主机重试
- **STALL**：端点暂停（错误状态），需要主机发送 CLEAR_FEATURE(ENDPOINT_HALT) 清除
- **NYET**（HS 专用）：设备接收了数据但未就绪接受更多，需要 PING 探测

### 4.6 DATA Toggle 机制

用于检测重复包和丢包，保证数据顺序：

```
主机发 DATA0 → 设备 ACK → 主机发 DATA1 → 设备 ACK → 主机发 DATA0 ...
若主机未收到 ACK，重发 DATA0 → 设备检测到重复（上次已 ACK DATA0）→ 丢弃并重发 ACK
```

- Toggle 初始值：端点使能时置 DATA0
- SETUP 事务后强制重置为 DATA1（数据阶段）

---

## 5. 事务模型

### 5.1 三阶段事务结构

USB 2.0 的基本通信单元是**事务（Transaction）**，由 1~3 个包组成：

```
OUT 事务：
  主机 →[Token: OUT]→ 设备
  主机 →[Data: DATA0/1]→ 设备
  设备 →[Handshake: ACK/NAK/STALL]→ 主机

IN 事务：
  主机 →[Token: IN]→ 设备
  设备 →[Data: DATA0/1]→ 主机    （或 NAK/STALL）
  主机 →[Handshake: ACK]→ 设备

SETUP 事务：
  主机 →[Token: SETUP]→ 设备
  主机 →[Data: DATA0（Setup 数据）]→ 设备
  设备 →[Handshake: ACK]→ 主机
```

### 5.2 HS PING 机制

高速设备引入 PING 令牌，避免主机盲目发送大数据包：

```
主机 →[Token: PING]→ 设备
设备 →[ACK]→ 主机    （缓冲区可用，主机可发送 OUT）
设备 →[NAK]→ 主机    （缓冲区满，继续等待）
```

### 5.3 Split 事务（TT，事务转换器）

当 HS Hub 下挂 FS/LS 设备时，EHCI 使用 Split 事务桥接两种速度域：

**完整 Split 事务流程（OUT 方向）**：

```
HS 总线（EHCI ↔ Hub）:
  EHCI →[SSPLIT(OUT)]→ Hub     （Start Split）
  Hub  →[ACK]→ EHCI

  Hub 内部转 FS 总线完成 FS OUT 事务:
  Hub  →[OUT]→ FS Device
  FS Device →[ACK]→ Hub

HS 总线（完成查询）:
  EHCI →[CSPLIT(OUT)]→ Hub     （Complete Split）
  Hub  →[ACK/NAK/STALL]→ EHCI
```

FS 和 LS 设备的实际传输被包含在 HS 微帧内，对 EHCI 透明。

---

## 6. 四种传输类型

### 6.1 控制传输（Control Transfer）

**用途**：设备配置、枚举请求、通用命令通道。每个设备必须在 EP0 支持控制传输。

**三阶段结构**：

```
阶段1 - Setup Stage：
  SETUP 事务（8字节 Setup 数据，固定 DATA0）

阶段2 - Data Stage（可选）：
  若 wLength > 0，则进行 IN 或 OUT 数据事务
  数据以 wMaxPacketSize 分包，DATA1 开始，逐包交替 Toggle

阶段3 - Status Stage：
  方向与 Data Stage 相反的零长度数据包（ZLP）
  确认整个控制传输完成
```

**时序要求**：
- Setup 到第一个 Data 事务：≤5 ms
- 相邻 Data 事务之间：≤5 ms
- 最后 Data 到 Status：≤5 ms

**带宽保证**：控制传输不保证带宽，但规范要求每帧至少保留 10% 给控制传输。

### 6.2 批量传输（Bulk Transfer）

**用途**：大数据量、非实时传输（U 盘、打印机、扫描仪）。

**特点**：
- 无周期性，尽力而为，空余带宽时传输
- 全速：最大包 8/16/32/64 字节；高速：最大包 512 字节
- 支持流控（NAK 无限次重试，直到成功）
- 提供错误检测（CRC16）和重传
- USB 3.0 引入 Stream（流），支持单端点多 Stream ID 并行传输

**传输终止条件**：
1. 短包（Small Packet，< wMaxPacketSize）→ 正常结束
2. 零长包（ZLP）→ 当数据恰好是 wMaxPacketSize 整数倍时，发送 ZLP 标记结束
3. 主机请求数据量满足

### 6.3 中断传输（Interrupt Transfer）

**用途**：低延迟、小数据量的周期性轮询（键盘、鼠标、HID 设备）。

**特点**：
- 主机按固定间隔（Interval）发送 IN 轮询（由 `bInterval` 定义）
- 若无数据，设备回 NAK；有数据则发送
- **保证延迟上限**，不保证固定带宽
- FS/LS：`bInterval` 单位为帧（1 ms），范围 1–255
- HS：`bInterval` 以 2^(bInterval-1) 微帧为单位（125 μs ~ 4096 ms）
- USB 3.0：与 HS 相同，但增加 Notification 类型端点

**带宽分配**：在设置接口时（SET_INTERFACE）从周期性带宽池分配，FS 最多 90%，HS 最多 80%。

### 6.4 同步传输（Isochronous Transfer）

**用途**：实时连续流（音频、视频、麦克风），允许丢包，不允许重传。

**特点**：
- 每帧/微帧保证带宽和时隙
- **无握手包**，无 NAK/ACK（丢包直接丢弃，不重传）
- **无 Data Toggle**（数据不计序列号）
- 每个 ISO 数据包用 `usb_iso_packet_descriptor` 单独记录状态和长度
- FS：每帧 1 包，最大 1023 字节
- HS：每微帧 1~3 包（高带宽 ISO），最大 1024 × 3 = 3072 字节/微帧

**同步类型**（`bmAttributes[3:2]`）：

| 值 | 名称 | 含义 |
|----|------|------|
| 00 | None | 无同步（异步数据源） |
| 01 | Async | 异步（自带时钟，与 SOF 无关） |
| 10 | Adaptive | 自适应（根据 SOF 调整采样率） |
| 11 | Sync | 同步（与 SOF 锁定） |

**四种传输类型综合对比**：

| 特性 | Control | Bulk | Interrupt | Isochronous |
|------|---------|------|-----------|-------------|
| 带宽保证 | 无（最低 10%） | 无 | 有（延迟上限） | 有（固定时隙） |
| 延迟保证 | 无 | 无 | 有（bInterval） | 有（每帧/微帧）|
| 错误重传 | 有 | 有 | 有 | **无** |
| DATA Toggle | 有 | 有 | 有 | **无** |
| 握手包 | 有 | 有 | 有 | **无** |
| EP0 | 必须 | 可选 | 可选 | 可选 |
| 典型应用 | 枚举/配置 | U盘/打印机 | 键鼠/HID | 音视频 |

---

## 7. 描述符体系

USB 使用层级化描述符传递设备能力和配置信息。所有描述符均以 `bLength + bDescriptorType` 开头：

```c
struct usb_descriptor_header {
    __u8 bLength;          /* 描述符总字节数 */
    __u8 bDescriptorType;  /* 描述符类型 ID */
};
```

### 7.1 描述符层次结构

```
Device Descriptor（1个）
  └── Configuration Descriptor（1个或多个）
        ├── Interface Association Descriptor（可选）
        └── Interface Descriptor（1个或多个）
              ├── Class-Specific Descriptor（可选，类定义）
              └── Endpoint Descriptor（0个或多个，EP0除外）
                    └── SS Endpoint Companion Descriptor（USB 3.0）

BOS Descriptor（USB 2.0 LPM / USB 3.x）
  ├── USB 2.0 Extension Capability
  ├── SuperSpeed USB Capability
  ├── Container ID Capability
  └── Platform Capability
```

### 7.2 设备描述符（`USB_DT_DEVICE = 0x01`）

定义于 `include/uapi/linux/usb/ch9.h:292`，大小固定 **18 字节**：

```c
struct usb_device_descriptor {
    __u8  bLength;            /* = 18 */
    __u8  bDescriptorType;    /* = 0x01 */
    __le16 bcdUSB;            /* USB 规范版本，如 0x0200 = USB 2.0 */
    __u8  bDeviceClass;       /* 设备类（0 = 由接口定义，0xFF = 厂商自定义） */
    __u8  bDeviceSubClass;    /* 设备子类 */
    __u8  bDeviceProtocol;    /* 设备协议 */
    __u8  bMaxPacketSize0;    /* EP0 最大包大小（8/16/32/64，SS 为 2^n） */
    __le16 idVendor;          /* 厂商 ID（VID，USB-IF 分配） */
    __le16 idProduct;         /* 产品 ID（PID，厂商自定义） */
    __le16 bcdDevice;         /* 设备版本（厂商定义） */
    __u8  iManufacturer;      /* 制造商字符串索引（0 = 无） */
    __u8  iProduct;           /* 产品字符串索引 */
    __u8  iSerialNumber;      /* 序列号字符串索引 */
    __u8  bNumConfigurations; /* 配置数量 */
};
```

**设备类编码**（`bDeviceClass`）：

| 值 | 类 | 说明 |
|----|----|------|
| 0x00 | 由接口定义 | 每个接口独立声明类 |
| 0x01 | Audio | USB 音频 |
| 0x02 | CDC | 通信设备（串口、以太网） |
| 0x03 | HID | 人机接口设备 |
| 0x06 | Still Image | 数码相机 |
| 0x07 | Printer | 打印机 |
| 0x08 | Mass Storage | U 盘、移动硬盘 |
| 0x09 | Hub | USB Hub |
| 0x0A | CDC-Data | CDC 数据接口 |
| 0x0E | Video | USB 视频 |
| 0xEF | Misc | 复合设备（IAD） |
| 0xFE | Application Specific | DFU、测试测量 |
| 0xFF | Vendor Specific | 厂商自定义 |

### 7.3 配置描述符（`USB_DT_CONFIG = 0x02`）

大小 **9 字节**（后跟接口和端点描述符）：

```c
struct usb_config_descriptor {
    __u8  bLength;             /* = 9 */
    __u8  bDescriptorType;     /* = 0x02 */
    __le16 wTotalLength;       /* 该配置所有描述符的总字节数 */
    __u8  bNumInterfaces;      /* 该配置包含的接口数 */
    __u8  bConfigurationValue; /* SET_CONFIGURATION 的参数值 */
    __u8  iConfiguration;      /* 配置字符串索引 */
    __u8  bmAttributes;        /* 属性位图 */
    __u8  bMaxPower;           /* 最大总线电流（单位 2 mA，HS）/ 8 mA（SS） */
};
```

**`bmAttributes` 位定义**：
- bit7：必须置 1（历史遗留）
- bit6：Self-powered（设备自带电源，不依赖总线）
- bit5：Remote Wakeup（设备可从挂起状态唤醒主机）
- bit4：Battery-powered（无线 USB）

主机读取配置描述符时，先请求 9 字节获取 `wTotalLength`，再请求完整长度的所有描述符（一次传输）。

### 7.4 接口描述符（`USB_DT_INTERFACE = 0x04`）

大小 **9 字节**：

```c
struct usb_interface_descriptor {
    __u8 bLength;            /* = 9 */
    __u8 bDescriptorType;    /* = 0x04 */
    __u8 bInterfaceNumber;   /* 接口编号（0 起始） */
    __u8 bAlternateSetting;  /* 备用设置编号（0 为默认） */
    __u8 bNumEndpoints;      /* 该接口使用的端点数（不含 EP0） */
    __u8 bInterfaceClass;    /* 接口类（同设备类编码） */
    __u8 bInterfaceSubClass; /* 接口子类 */
    __u8 bInterfaceProtocol; /* 接口协议 */
    __u8 iInterface;         /* 接口字符串索引 */
};
```

**备用设置（Alternate Setting）**：同一接口可定义多组端点配置（带宽不同），通过 `SET_INTERFACE` 切换，常用于 ISO 音频设备（0 = 静默，1 = 传输）。

### 7.5 端点描述符（`USB_DT_ENDPOINT = 0x05`）

大小 **7 字节**（音频端点 9 字节）：

```c
struct usb_endpoint_descriptor {
    __u8  bLength;           /* = 7（或 9，Audio） */
    __u8  bDescriptorType;   /* = 0x05 */
    __u8  bEndpointAddress;  /* [7]: 方向 0=OUT/1=IN; [3:0]: 端点号 */
    __u8  bmAttributes;      /* [1:0]: 传输类型; [3:2]: 同步类型（ISO）; [5:4]: 用途 */
    __le16 wMaxPacketSize;   /* [10:0]: 最大包大小; [12:11]: 高带宽乘数（HS ISO/INT） */
    __u8  bInterval;         /* 轮询间隔（INT/ISO），单位由速度决定 */
};
```

**`bEndpointAddress` 位域**：
```
bit7 = 方向（1=IN，0=OUT）
bit6-4 = 保留
bit3-0 = 端点号（0-15）
```

**`bmAttributes` 位域**：
```
bit1:0 = 传输类型（00=Control, 01=ISO, 10=Bulk, 11=INT）
bit3:2 = 同步类型（ISO 专用：00=None, 01=Async, 10=Adaptive, 11=Sync）
bit5:4 = 用途（ISO 专用：00=Data, 01=Feedback, 10=Implicit Feedback）
```

**`wMaxPacketSize` 位域**：
```
bit10:0 = 最大包字节数
bit12:11 = 附加事务数（HS 高带宽：00=×1, 01=×2, 10=×3）
```

### 7.6 字符串描述符（`USB_DT_STRING = 0x03`）

```c
struct usb_string_descriptor {
    __u8  bLength;
    __u8  bDescriptorType; /* = 0x03 */
    __le16 wData[];        /* UTF-16LE 编码字符串 */
};
```

- **索引 0**（特殊）：不是字符串，而是支持的语言 ID 列表（`wData[]` 为 LANGID 数组）
- 常用 LANGID：0x0409 = 英语（美国），0x0804 = 中文（简体）
- 最大 126 个字符（`USB_MAX_STRING_LEN`）
- 读取字符串前需先读索引 0 获取支持的语言

### 7.7 BOS 描述符（`USB_DT_BOS = 0x0F`）

Binary Object Store（USB 3.0 / USB 2.0 LPM 引入），聚合设备能力描述：

```c
struct usb_bos_descriptor {
    __u8  bLength;          /* = 5 */
    __u8  bDescriptorType;  /* = 0x0F */
    __le16 wTotalLength;    /* BOS + 所有能力描述符总长度 */
    __u8  bNumDeviceCaps;   /* 能力描述符数量 */
};
```

**常见能力描述符**（跟在 BOS 之后）：

| 类型 | 值 | 结构体 | 包含信息 |
|------|----|----|------|
| USB 2.0 Extension | 0x02 | `usb_ext_cap_descriptor` | LPM 支持、BESL 值 |
| SuperSpeed Capability | 0x03 | `usb_ss_cap_descriptor` | 支持的速度、U1/U2 退出延迟 |
| Container ID | 0x04 | `usb_ss_container_id_descriptor` | 128 bit UUID，跨接口标识 |
| Platform | 0x05 | — | 平台特定能力（MSFT/WebUSB 等） |

### 7.8 SuperSpeed 端点伴随描述符（`USB_DT_SS_ENDPOINT_COMP = 0x30`）

USB 3.0 中每个非控制端点描述符后必须跟随该描述符：

```c
struct usb_ss_ep_comp_descriptor {
    __u8  bLength;           /* = 6 */
    __u8  bDescriptorType;   /* = 0x30 */
    __u8  bMaxBurst;         /* 突发传输中最大包数 - 1（0~15） */
    __u8  bmAttributes;      /* Bulk: 低5位=最大Stream数; ISO: 低2位=Mult */
    __le16 wBytesPerInterval;/* ISO/INT: 每服务间隔传输字节数 */
};
```

---

## 8. 标准设备请求（Chapter 9）

### 8.1 Setup 数据包格式

所有控制传输的 Setup 阶段传输固定 **8 字节**（`struct usb_ctrlrequest`）：

```c
struct usb_ctrlrequest {
    __u8  bRequestType; /* 请求方向 + 类型 + 接收者 */
    __u8  bRequest;     /* 具体请求命令 */
    __le16 wValue;      /* 请求参数1（与 bRequest 相关） */
    __le16 wIndex;      /* 请求参数2（接口/端点号） */
    __le16 wLength;     /* Data Stage 数据字节数 */
};
```

**`bRequestType` 位域**：

```
bit7     = 方向（0 = 主机→设备 OUT, 1 = 设备→主机 IN）
bit6:5   = 类型（00 = Standard, 01 = Class, 10 = Vendor, 11 = Reserved）
bit4:0   = 接收者（00000 = Device, 00001 = Interface, 00010 = Endpoint）
```

### 8.2 标准请求完整表

定义于 `include/uapi/linux/usb/ch9.h:78`：

| `bRequest` | 值 | 方向 | 接收者 | `wValue` | `wIndex` | `wLength` | 功能 |
|-----------|-----|------|--------|---------|---------|----------|------|
| `GET_STATUS` | 0x00 | IN | Dev/Intf/EP | 0 | 0/接口号/端点 | 2 | 读取状态（2字节）|
| `CLEAR_FEATURE` | 0x01 | OUT | Dev/Intf/EP | Feature选择器 | 0/接口/端点 | 0 | 清除特性 |
| `SET_FEATURE` | 0x03 | OUT | Dev/Intf/EP | Feature选择器 | 0/接口/端点 | 0 | 设置特性 |
| `SET_ADDRESS` | 0x05 | OUT | Device | 新设备地址（1-127） | 0 | 0 | 设置设备地址 |
| `GET_DESCRIPTOR` | 0x06 | IN | Device | 描述符类型+索引 | 语言ID | 描述符长度 | 读取描述符 |
| `SET_DESCRIPTOR` | 0x07 | OUT | Device | 描述符类型+索引 | 语言ID | 描述符长度 | 写入描述符（可选）|
| `GET_CONFIGURATION` | 0x08 | IN | Device | 0 | 0 | 1 | 读取当前配置值 |
| `SET_CONFIGURATION` | 0x09 | OUT | Device | 配置值（0=unconfigure）| 0 | 0 | 激活配置 |
| `GET_INTERFACE` | 0x0A | IN | Interface | 0 | 接口号 | 1 | 读取当前 altsetting |
| `SET_INTERFACE` | 0x0B | OUT | Interface | Alternate Setting | 接口号 | 0 | 切换 altsetting |
| `SYNCH_FRAME` | 0x0C | IN | Endpoint | 0 | 端点号 | 2 | ISO 帧同步 |
| `SET_SEL` | 0x30 | OUT | Device | 0 | 0 | 6 | 设置 U1/U2 退出延迟（USB 3.0）|
| `SET_ISOCH_DELAY` | 0x31 | OUT | Device | 延迟（ns） | 0 | 0 | 设置 ISO 延迟（USB 3.0）|

### 8.3 GET_STATUS 响应格式

**设备状态**（2字节）：
```
bit0 = Self-Powered（1 = 自供电）
bit1 = Remote Wakeup（1 = 已启用远程唤醒）
bit2 = U1 Enable（USB 3.0）
bit3 = U2 Enable（USB 3.0）
bit4 = LTM Enable（USB 3.0）
```

**端点状态**（2字节）：
```
bit0 = HALT（端点被 STALL）
```

### 8.4 CLEAR_FEATURE 常用特性

| Feature 选择器 | 值 | 接收者 | 功能 |
|---------------|-----|-------|------|
| `ENDPOINT_HALT` | 0 | Endpoint | 清除 STALL，重置 Toggle |
| `DEVICE_REMOTE_WAKEUP` | 1 | Device | 关闭远程唤醒 |
| `TEST_MODE` | 2 | Device | 测试模式（HS 专用） |
| `U1_ENABLE` | 48 | Device | 关闭 U1（USB 3.0）|
| `U2_ENABLE` | 49 | Device | 关闭 U2（USB 3.0）|

---

## 9. 设备枚举流程

枚举是主机识别、配置新接入设备的完整过程，通常在设备插入时由 Hub 驱动触发。

### 9.1 完整枚举序列

```
步骤 1: 检测连接
  Hub 检测到 D+ 或 D− 上拉（由设备速度决定上拉哪根线）
  FS/LS 设备：上拉至 3.3V
  Hub 的端口状态寄存器设置 PORT_CONNECTION 位，产生中断

步骤 2: 速度检测
  Hub 报告端口连接变化给主机
  主机读取 Hub 端口状态寄存器（GET_PORT_STATUS）
  确定设备速度（LS/FS/HS 通过 chirp 协议区分 HS）

步骤 3: 复位
  主机发 SET_PORT_FEATURE(PORT_RESET) 给 Hub
  Hub 对该端口施加 SE0 超过 10 ms（复位）
  HS 设备在复位中进行 Chirp 协商：
    设备发 Chirp K → Hub 收到后发 Chirp K Chirp J × 3 交替
    设备检测到 3 组 KJ 序列 → 进入 HS 模式
  复位结束后端口处于 Enabled 状态

步骤 4: 读设备描述符（部分）
  主机向地址 0 发送 GET_DESCRIPTOR(Device, 0, 8)
  读取前 8 字节，获取 bMaxPacketSize0（EP0 最大包大小）

步骤 5: 第二次复位（某些情况需要）
  确保设备状态干净（部分控制器要求）

步骤 6: 分配地址
  主机从地址位图选择空闲地址
  发送 SET_ADDRESS(new_addr)
  设备 ACK 后等待 ≥2 ms 切换到新地址
  此后所有通信使用新地址

步骤 7: 读完整设备描述符
  GET_DESCRIPTOR(Device, 0, 18)
  获取 VID/PID/Class/NumConfigs 等

步骤 8: 读 BOS 描述符（USB 3.x 或支持 LPM 的 USB 2.0 设备）
  GET_DESCRIPTOR(BOS, 0, 5)  → 获取 wTotalLength
  GET_DESCRIPTOR(BOS, 0, wTotalLength)

步骤 9: 读字符串描述符（可选）
  GET_DESCRIPTOR(String, 0, 255)  → 读语言 ID 列表
  GET_DESCRIPTOR(String, idx, 255, langid)  → 读具体字符串

步骤 10: 读配置描述符
  GET_DESCRIPTOR(Config, 0, 9)     → 获取 wTotalLength
  GET_DESCRIPTOR(Config, 0, wTotalLength)  → 读所有接口/端点描述符

步骤 11: 选择配置
  主机分析所有配置，选择合适的
  SET_CONFIGURATION(bConfigurationValue)
  设备激活端点，进入已配置状态

步骤 12: 驱动匹配
  内核在设备/接口层面匹配 usb_driver（按 VID/PID/Class/SubClass/Protocol）
  调用 driver->probe()，驱动开始工作
```

### 9.2 HS Chirp 协商时序

```
复位开始（SE0 持续）
    ↓ 约 3 ms 后
设备发 Chirp K（约 1-7 ms）
    ↓
Hub 检测到 Chirp K
Hub 发 Chirp KJKJKJ（每个约 100-500 μs）
    ↓
设备计数到 3 组 KJ → 进入 HS（切换到电流模式）
    ↓
Hub 也切换到 HS 模式
复位结束
```

若未检测到 Chirp（FS 设备不响应 Chirp），设备以 FS 运行。

---

## 10. 电源管理

### 10.1 总线电源规格

| 配置 | VBUS 电压 | 最大电流 | 总功率 |
|------|-----------|---------|-------|
| USB 2.0 低功耗（枚举前） | 4.75~5.25V | 100 mA | 500 mW |
| USB 2.0 高功耗（配置后） | 4.75~5.25V | 500 mA | 2.5 W |
| USB 3.0 低功耗 | 4.45~5.25V | 150 mA | 0.75 W |
| USB 3.0 高功耗 | 4.45~5.25V | 900 mA | 4.5 W |
| USB Battery Charging 1.2 | — | 1500 mA | — |
| USB PD（Type-C） | 5/9/12/15/20V | 最大 5A | 最大 100 W |

配置描述符中 `bMaxPower` 以 **2 mA** 为单位（USB 2.0）或 **8 mA**（USB 3.x）声明需求。

### 10.2 USB 2.0 挂起（Suspend）

**全局挂起**：总线空闲超过 3 ms（无 SOF、无数据），设备进入挂起状态：
- 设备电流降至 ≤500 μA（低功耗）或 ≤2.5 mA（自供电）
- VBUS 维持，设备保持地址和配置

**选择性挂起（Selective Suspend）**：主机驱动可单独挂起某个设备或接口，不影响其他设备。

**远程唤醒（Remote Wakeup）**：
- 设备需在配置描述符中声明支持（`bmAttributes[5]`）
- 主机通过 `SET_FEATURE(DEVICE_REMOTE_WAKEUP)` 授权
- 设备挂起后可通过总线信号唤醒主机：发送 K 状态持续 1-15 ms

### 10.3 USB 2.0 LPM（Link Power Management）

LPM 引入 **L1 状态**（介于 L0 运行态和 L2/L3 断电态之间），退出延迟更短：

| 状态 | 名称 | 描述 | 退出延迟 |
|------|------|------|---------|
| L0 | U0（运行） | 正常传输 | — |
| L1 | U1（LPM 待机） | 快速待机，VBUS 保持 | ≤50 μs |
| L2 | U2（挂起） | 传统挂起 | ≤10 ms |
| L3 | U3（断开） | VBUS 断开 | 复位 |

L1 由 **EXT Token Packet（LPM Token）** 触发，主机发 EXT 令牌 + LPM 载荷，设备回 ACK/NAK/STALL/NYET。

### 10.4 USB 3.x 链路电源状态（U States）

USB 3.x 定义 4 个链路功率状态：

| 状态 | 名称 | 描述 | U1 退出延迟 | U2 退出延迟 |
|------|------|------|------------|------------|
| U0 | 活动（Active） | 正常传输 | — | — |
| U1 | 空闲（Idle） | 短暂空闲，快速恢复 | ≤10 μs（设备侧） | — |
| U2 | 深度空闲 | 更低功耗，LFPS 唤醒 | — | ≤57 μs |
| U3 | 挂起（Suspend） | 最低功耗，等效 USB 2.0 Suspend | — | ≤20 ms |

**LTM（Latency Tolerance Message）**：设备通过 LTM 告知主机自己能容忍的最大延迟，主机据此决定是否进入 U1/U2。

---

## 11. USB 3.x 超速协议

### 11.1 架构差异

USB 3.x 在 USB 2.0 的半双工差分总线之上，**额外增加**一对全双工 SuperSpeed 差分对，两套总线完全独立运行：

```
USB 3.x 连接器（Type-A）:
  USB 2.0 部分：VBUS / D- / D+ / GND（向下兼容）
  USB 3.x 部分：RX- / RX+ / GND / TX- / TX+
```

### 11.2 数据链路层

USB 3.x 引入真正的数据链路层概念，使用 **Header Packet** 和 **Data Packet** 分离：

**Header Packet（20 字节）**：
```
┌──────────────────────────────────────────┐
│ Link Header（4B）: Type + DPH + Seq + CRC │
│ Transaction Packet Header（12B）          │
│ Link Control Word（4B）                   │
└──────────────────────────────────────────┘
```

包类型（Transaction Packet Type）：

| 类型 | 值 | 描述 |
|------|-----|------|
| DP_INFO | 0x4 | 数据包信息头 |
| DP_ACK | 0x1 | 数据包确认 |
| ACK | 0x1 | 事务确认 |
| NRDY | 0x6 | 端点未就绪 |
| ERDY | 0x3 | 端点就绪（替代 NAK 轮询）|
| STATUS | 0x2 | 控制传输状态 |
| STALL | 0x5 | 端点暂停 |
| DEV_NOTIF | 0x8 | 设备通知 |
| PING | 0xC | 链路 Ping |
| PING_RESP | 0xD | Ping 响应 |

### 11.3 ERDY 取代 NAK 轮询

USB 2.0 中主机必须持续轮询端点（发 IN 令牌收到 NAK 则等待重试），浪费带宽。

USB 3.x 引入 **ERDY（Endpoint Ready）** 机制：

```
主机发 IN TP（事务包）→ 设备无数据 → 设备回 NRDY
主机停止轮询，等待
（当设备有数据时）
设备主动发 ERDY TP → 主机收到后立即发送 IN TP
设备发送数据
```

这消除了无效轮询，大幅提升高速设备的带宽利用率。

### 11.4 突发传输（Burst Transfer）

USB 3.x Bulk 端点支持突发传输，连续发送多个数据包而无需逐包握手：

```
MaxBurst = bMaxBurst + 1    （端点伴随描述符）
最大突发包数：1-16（MaxBurst=0-15）
单次突发总字节：MaxPacketSize × MaxBurst
```

突发流程：
```
主机发 OUT TP（NumP=MaxBurst）
设备接收 MaxBurst 个数据包
设备发 DP ACK（确认整个突发）
```

### 11.5 流（Streams）

USB 3.0 Bulk 端点支持多 Stream，在一个端点内实现多个逻辑通道（用于 UAS 等协议）：

- `bMaxBurst` 的 Stream 位（`bmAttributes[4:0]`）：0 = 不支持，n = 最多 2^n 个 Stream
- 每个 Stream 独立流控（独立的 NRDY/ERDY）
- 主机通过 `ALLOC_STREAMS` xHCI 命令为端点分配流

### 11.6 LTSSM（链路训练状态机）

Link Training and Status State Machine，USB 3.x 物理层连接建立过程：

```
Rx.Detect → Polling → U0（运行）
                ↓ 训练：
            Polling.LFPS → Polling.Active → Polling.Config → Polling.Idle → U0
```

**LFPS（Low-Frequency Periodic Signal）**：用于低功耗状态下的唤醒和协商，频率 10–50 MHz，与 SuperSpeed 信号独立。

```
U0 → U1 入口：
  发送方发 LFPS Ping → 接收方响应 LFPS Ping_Resp → 进入 U1

U3 唤醒：
  任一端发 LFPS Warm Reset / U3 LFPS → 双方协商回到 U0
```

### 11.7 USB 3.x 编码方案

| 版本 | 编码 | 效率 | 实际有效带宽 |
|------|------|------|-------------|
| USB 3.0（5 Gbps） | 8b/10b | 80% | ~4 Gbps |
| USB 3.1 Gen2（10 Gbps） | 128b/132b | 97% | ~9.7 Gbps |
| USB 3.2（20 Gbps） | 128b/132b × 2 lanes | 97% | ~19.5 Gbps |

**8b/10b**：每 8 位数据编为 10 位，保证 DC 平衡，最多 5 个连续相同位（可自时钟）
**128b/132b**：每 128 位数据加 4 位控制头，开销仅 3%，保留频繁的时钟嵌入

---

## 12. USB Type-C

### 12.1 物理接口

Type-C 连接器有 24 个引脚，支持正反插（引脚完全对称）：

```
引脚分配（CC1 侧）：
  GND, TX1+, TX1-, VBUS, CC1, D+, D-, SBU1, VBUS, RX2-, RX2+, GND

引脚分配（CC2 侧，镜像对称）：
  GND, RX1+, RX1-, VBUS, SBU2, D+, D-, CC2, VBUS, TX2-, TX2+, GND
```

**CC（Configuration Channel）引脚**：
- 检测连接方向（CC1 vs CC2 哪个被拉低）
- 区分 DFP（Host）和 UFP（Device）角色
- 传输 USB PD 消息
- 提供 VCONN 给有源线缆的 E-marker 芯片

### 12.2 角色检测

| 设备类型 | CC1/CC2 处理 |
|---------|-------------|
| UFP（Device） | 通过 Rd（5.1 kΩ）下拉 CC |
| DFP（Host） | 通过 Rp 上拉 CC（56 kΩ/22 kΩ/10 kΩ 对应不同电流能力）|
| DRP（Dual Role） | 交替切换 Rp/Rd 探测 |

Rp 值对应电流能力：
- 56 kΩ → 默认 USB 电流（500 mA/900 mA）
- 22 kΩ → 1.5 A
- 10 kΩ → 3.0 A

### 12.3 USB Power Delivery（PD）

USB PD 通过 CC 引脚传输 BMC 编码的 PD 消息，实现动态功率协商：

**PD 消息结构**：
```
Preamble（64 bits）+ SOP（4 symbols）+ Header（16 bits）+ Data Objects（可选）+ CRC（32 bits）+ EOP
```

**消息类型**：
- **Control Messages**：GoodCRC / GotoMin / Accept / Reject / PS_RDY 等
- **Data Messages**：Source_Capabilities / Request / BIST / Vendor_Defined 等
- **Extended Messages**：Security_Request / FW_Update 等

**PD 状态机（协商流程）**：
```
Source 发 Source_Capabilities（含所有 PDO）
Sink 分析 PDO，发 Request（指定 Request Data Object）
Source 回 Accept
Source 切换电压/电流
Source 稳定后发 PS_RDY
传输开始（Sink 以新功率工作）
```

**PDO（Power Data Object）类型**：
- Fixed PDO：固定电压（5V/9V/12V/15V/20V），固定电流
- Variable PDO：电压范围可调
- Battery PDO：以瓦特为单位
- Augmented PDO（PPS）：程控电源，电压可 20 mV 精度调节，电流可 50 mA 精度调节

### 12.4 Alternate Mode

Type-C 引脚复用允许在 USB 之外传输其他协议（通过 PD VDM 消息协商）：

| Alternate Mode | 用途 |
|---------------|------|
| DisplayPort | 视频输出（DP 1.4）|
| Thunderbolt 3/4 | Intel 高速通道（含 PCIe / DP）|
| USB4 | 基于 Thunderbolt 3 |
| HDMI | 直接输出 HDMI 信号 |
| MHL | 移动高清链路 |

进入 Alternate Mode 流程：
```
DFP 发 VDM Discover Identity → UFP 响应自身 ID
DFP 发 VDM Discover SVIDs → UFP 响应支持的 SVID
DFP 发 VDM Enter Mode（指定 SVID）→ UFP 响应 ACK
进入 Alt Mode，SuperSpeed 引脚重新映射
```

---

## 附录：关键常量速查

### 最大包大小

| 传输类型 | LS | FS | HS | SS（USB 3.0） |
|----------|----|----|-----|--------------|
| Control（EP0） | 8 | 8/16/32/64 | 64 | 512（2^n，实为512）|
| Bulk | — | 8/16/32/64 | 512 | 1024 |
| Interrupt | 8 | 64 | 1024 | 1024 |
| Isochronous | — | 1023 | 1024 | 1024 |

### 传输时序约束

| 约束 | 值 |
|------|----|
| 控制传输 Setup 到 Data/Status | ≤5 ms |
| SET_ADDRESS 后设备切换延迟 | ≥2 ms |
| USB 2.0 挂起超时 | 3 ms 无活动 |
| USB 2.0 复位持续时间 | 10–20 ms |
| HS Chirp K 持续时间 | 1–7 ms |
| SOF 间隔（FS） | 1 ms ±1% |
| 微帧间隔（HS） | 125 μs ±0.05% |

### 标准描述符类型 ID

| ID | 描述符 | 定义 |
|----|--------|------|
| 0x01 | Device | `USB_DT_DEVICE` |
| 0x02 | Configuration | `USB_DT_CONFIG` |
| 0x03 | String | `USB_DT_STRING` |
| 0x04 | Interface | `USB_DT_INTERFACE` |
| 0x05 | Endpoint | `USB_DT_ENDPOINT` |
| 0x06 | Device Qualifier | `USB_DT_DEVICE_QUALIFIER` |
| 0x07 | Other Speed Config | `USB_DT_OTHER_SPEED_CONFIG` |
| 0x0B | Interface Association | `USB_DT_INTERFACE_ASSOCIATION` |
| 0x0F | BOS | `USB_DT_BOS` |
| 0x10 | Device Capability | `USB_DT_DEVICE_CAPABILITY` |
| 0x30 | SS Endpoint Companion | `USB_DT_SS_ENDPOINT_COMP` |
| 0x31 | SSP Isoc EP Companion | `USB_DT_SSP_ISOC_ENDPOINT_COMP` |

### 内核源码对照

| 本文内容 | 内核文件 | 行号 |
|---------|---------|------|
| Setup 请求结构体 | `include/uapi/linux/usb/ch9.h` | 210 |
| 设备描述符 | `include/uapi/linux/usb/ch9.h` | 292 |
| 配置描述符 | `include/uapi/linux/usb/ch9.h` | 354 |
| 接口描述符 | `include/uapi/linux/usb/ch9.h` | 397 |
| 端点描述符 | `include/uapi/linux/usb/ch9.h` | 415 |
| SS 端点伴随描述符 | `include/uapi/linux/usb/ch9.h` | 709 |
| BOS 描述符 | `include/uapi/linux/usb/ch9.h` | 866 |
| 设备状态枚举 | `include/uapi/linux/usb/ch9.h` | 1211 |
| 速度枚举 | `include/uapi/linux/usb/ch9.h` | 1201 |
| 标准请求宏 | `include/uapi/linux/usb/ch9.h` | 78–111 |
| 描述符类型宏 | `include/uapi/linux/usb/ch9.h` | 237–270 |
| 端点类型/方向宏 | `include/uapi/linux/usb/ch9.h` | 437–467 |
| 设备类宏 | `include/uapi/linux/usb/ch9.h` | 318–342 |
