# EXT4 文件系统架构文档
## 基于 Linux 6.19.6 内核源码 (`fs/ext4/`) 深度分析

---

## 一、总体架构概览

ext4 是 Linux 内核中最成熟的日志文件系统，由 ext3 演进而来，在设计上通过**分层架构**将 VFS 接口、元数据管理、块分配、日志等多个子系统紧密协作，提供高性能、高可靠的文件存储服务。

```
┌─────────────────────────────────────────────────────────────────┐
│                    应用层 (User Space)                            │
└────────────────────────────┬────────────────────────────────────┘
                             │ syscall
┌────────────────────────────▼────────────────────────────────────┐
│              VFS 虚拟文件系统层 (Kernel VFS Layer)                │
│     file_operations / inode_operations / address_space_ops       │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                     EXT4 核心层                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │  inode.c     │  │  namei.c     │  │     file.c / dir.c   │   │
│  │  (读写路径)  │  │  (目录/查找) │  │     (文件/目录ops)    │   │
│  └──────┬───────┘  └──────┬───────┘  └──────────────────────┘   │
│         │                 │                                       │
│  ┌──────▼───────────────▼──────────────────────────────────┐    │
│  │              Extent 子系统 (extents.c)                   │    │
│  │  B树结构 | 路径遍历 | 插入/删除/分裂/合并                  │    │
│  └──────────────────────────┬───────────────────────────────┘   │
│                             │                                     │
│  ┌──────────────────────────▼───────────────────────────────┐   │
│  │         Extent 状态树 (extents_status.c)                  │   │
│  │  红黑树 | WRITTEN/UNWRITTEN/DELAYED/HOLE 状态追踪          │   │
│  └──────────────────────────┬───────────────────────────────┘   │
│                             │                                     │
│  ┌──────────────────────────▼───────────────────────────────┐   │
│  │          多块分配器 (mballoc.c / balloc.c)                 │   │
│  │  Buddy System | 预分配策略 | 本地性组 | 块位图              │   │
│  └──────────────────────────┬───────────────────────────────┘   │
│                             │                                     │
│  ┌──────────────────────────▼───────────────────────────────┐   │
│  │           日志子系统 (ext4_jbd2.c / fast_commit.c)         │   │
│  │  JBD2 集成 | 快速提交 | 事务管理 | Orphan 恢复              │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                      块设备层 (Block Device)                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、源文件目录（共 45 个文件）

### 2.1 头文件（10个）

| 文件 | 大小 | 核心作用 |
|------|------|---------|
| `ext4.h` | 138KB | **主头文件**：所有核心结构体、宏、特性标志 |
| `ext4_extents.h` | 8.4KB | Extent 树磁盘结构（`ext4_extent`、`ext4_extent_idx`、`ext4_extent_header`） |
| `ext4_jbd2.h` | 16KB | JBD2 日志集成接口，事务信用计算宏 |
| `extents_status.h` | 8.0KB | Extent 状态枚举、ES 树节点结构 |
| `mballoc.h` | 7.1KB | 多块分配器数据结构（Buddy、预分配空间） |
| `xattr.h` | 8.2KB | 扩展属性结构与命名空间索引 |
| `acl.h` | 1.6KB | POSIX ACL 磁盘格式结构 |
| `fast_commit.h` | 4.2KB | 快速提交标签定义、FC 状态结构 |
| `fsmap.h` | 2.0KB | 文件系统块映射接口 |
| `truncate.h` | 1.4KB | 截断辅助函数 |

### 2.2 核心源文件（35个）

| 文件 | 大小 | 职责 |
|------|------|------|
| `super.c` | 210KB | 超级块管理、挂载/卸载、分配器调优参数 |
| `inode.c` | 195KB | Inode 读写、块映射、延迟分配写路径 |
| `extents.c` | 172KB | Extent B 树管理（插入、删除、分裂、合并） |
| `mballoc.c` | 203KB | 多块分配器（Buddy 系统、预分配策略） |
| `namei.c` | 111KB | 文件/目录创建、链接、重命名、名字查找 |
| `xattr.c` | 85KB | 扩展属性的增删改查 |
| `fast_commit.c` | 68KB | 快速提交日志记录与恢复 |
| `extents_status.c` | 65KB | Extent 状态红黑树 |
| `resize.c` | 64KB | 在线文件系统扩容 |
| `ialloc.c` | 45KB | Inode 分配与回收 |
| `ioctl.c` | 54KB | ioctl 命令处理（碎片整理、特性设置等） |
| `indirect.c` | 43KB | 传统间接块映射（legacy 兼容） |
| `inline.c` | 50KB | 内联数据功能 |
| `balloc.c` | 30KB | 基础块分配、块组元数据 |
| `file.c` | 27KB | 文件读写操作、DIO 支持 |
| `dir.c` | 19KB | 目录遍历、条目校验 |
| `move_extent.c` | 19KB | Extent 迁移（碎片整理） |
| `orphan.c` | 19KB | 孤儿 inode 管理与崩溃恢复 |
| `page-io.c` | 17KB | Page I/O、Buffer 提交 |
| `migrate.c` | 17KB | 间接块格式 → Extent 格式迁移 |
| `fsmap.c` | 22KB | 文件系统块映射查询 |
| `sysfs.c` | 18KB | sysfs 接口（调优参数暴露） |
| `mmp.c` | 11KB | 多挂载保护（MMP） |
| `ext4_jbd2.c` | 11KB | JBD2 集成实现 |
| `verity.c` | 11KB | fs-verity 完整性验证 |
| `readpage.c` | 11KB | 页面读取优化 |
| `inode-test.c` | — | inode 单元测试 |
| `mballoc-test.c` | — | 多块分配器单元测试 |
| `crypto.c` | 6KB | fscrypt 加密集成 |
| `fsync.c` | 4.8KB | fsync 实现 |
| `block_validity.c` | 9.5KB | 块地址合法性检验 |
| `hash.c` | 7.4KB | 目录哈希算法 |
| `bitmap.c` | 2.7KB | 位图操作工具函数 |
| `acl.c` | 6.7KB | POSIX ACL 实现 |
| `symlink.c` | 3.2KB | 符号链接操作 |

---

## 三、磁盘格式（On-Disk Layout）

### 3.1 超级块 `struct ext4_super_block`（1024 字节，位于偏移 1024）

```
关键字段：
  s_inodes_count          - 总 inode 数
  s_blocks_count_lo/hi    - 总块数（64-bit 拆分）
  s_r_blocks_count_lo     - 保留块数（root 用户）
  s_free_blocks_count_lo  - 空闲块数
  s_log_block_size        - 块大小 = 1024 << s_log_block_size
  s_blocks_per_group      - 每块组的块数
  s_inodes_per_group      - 每块组的 inode 数
  s_first_data_block      - 第一个数据块号（1 KB块时为1，4 KB时为0）
  s_inode_size            - inode 大小（默认256字节）
  s_feature_compat        - 兼容特性标志
  s_feature_incompat      - 不兼容特性标志
  s_feature_ro_compat     - 只读兼容特性标志
  s_encrypt_pw_salt[16]   - 加密密码盐
  s_checksum_seed         - CRC32C 校验和种子
```

### 3.2 块组描述符 `struct ext4_group_desc`（32/64 字节）

```
  bg_block_bitmap_lo/hi   - 块位图所在块号
  bg_inode_bitmap_lo/hi   - inode 位图所在块号
  bg_inode_table_lo/hi    - inode 表起始块号
  bg_free_blocks_count    - 组内空闲块数
  bg_free_inodes_count    - 组内空闲 inode 数
  bg_itable_unused        - 未使用的 inode 数（加速扫描）
  bg_checksum             - 描述符 CRC16 校验
```

### 3.3 Inode `struct ext4_inode`（128~256+ 字节）

```
  i_mode                  - 文件类型 + 权限
  i_uid_lo / i_gid_lo     - 所有者 UID/GID
  i_size_lo / i_size_high - 文件大小（支持64-bit）
  i_atime/i_ctime/i_mtime - 时间戳（精度1秒）
  i_atime_extra/...       - 时间戳纳秒扩展
  i_crtime / i_crtime_extra - 文件创建时间（ext4新增）
  i_flags                 - inode 标志（EXTENTS_FL, INLINE_DATA_FL等）
  i_block[15]             - 60字节：extent树根 或 间接块指针 或 内联数据
  i_file_acl_lo/hi        - EA块指针
  i_blocks_lo/hi          - 已分配块数
  i_extra_isize           - 额外 inode 字段大小
  i_checksum_lo/hi        - inode CRC32C 校验
```

### 3.4 Extent 树磁盘结构

```
ext4_extent_header (12 字节，位于 i_block 前12字节或内部节点块头部):
  eh_magic    = 0xF30A
  eh_entries  - 有效条目数
  eh_max      - 最大容量
  eh_depth    - 树深度（0=叶节点）
  eh_generation - 树版本

ext4_extent (12 字节，叶节点):
  ee_block    - 逻辑块起始号
  ee_len      - 块数（bit[15]=1 表示 unwritten）
                最大初始化长度: 32768 块 = 128MB (4KB块)
                最大未写入长度: 32767 块
  ee_start_hi - 物理块号高16位
  ee_start_lo - 物理块号低32位

ext4_extent_idx (12 字节，索引节点):
  ei_block    - 覆盖的逻辑块起始
  ei_leaf_lo  - 子节点物理块低32位
  ei_leaf_hi  - 子节点物理块高16位
```

**Extent 树布局图：**
```
inode.i_block[0..11]:
┌─────────────────────────────────────────────────────┐
│ ext4_extent_header (12B)                             │
│ ext4_extent[0] (12B) ... ext4_extent[3] (12B)       │  ← 根最多4条叶子
└─────────────────────────────────────────────────────┘
       若 eh_depth > 0, 则存索引节点，指向子块：
       ┌────────────────────────────────┐
       │ ext4_extent_header             │
       │ ext4_extent_idx[0..N]          │  ← 中间节点
       └────────────────────────────────┘
                     └─→ ...叶子块...
```

---

## 四、核心数据结构（In-Memory）

### 4.1 `struct ext4_inode_info`（每 inode 内存结构，嵌入 `struct inode`）

```c
struct ext4_inode_info {
    __le32 i_data[15];           // Extent树根 / 间接块指针 / 内联数据
    __u32  i_dtime;              // 删除时间
    ext4_fsblk_t i_file_acl;    // EA块物理地址
    __u32  i_block_group;        // inode所在块组

    /* 延迟分配 */
    __u32  i_reserved_data_blocks;  // 为延迟分配预留的块数

    /* 预分配 */
    struct rb_root i_prealloc_node;  // 预分配空间红黑树
    spinlock_t i_prealloc_lock;

    /* Extent 状态树 */
    struct ext4_es_tree i_es_tree;   // 逻辑块→状态的红黑树缓存
    rwlock_t i_es_lock;

    /* 快速提交 */
    ext4_lblk_t i_fc_lblk_start;
    ext4_lblk_t i_fc_lblk_len;
    struct list_head i_fc_list;

    /* 数据保护 */
    struct rw_semaphore i_data_sem;  // 保护 extent 树和 ES 树
};
```

### 4.2 `struct ext4_sb_info`（超级块内存镜像）

```c
struct ext4_sb_info {
    /* 块组 */
    ext4_group_t s_groups_count;
    unsigned long s_blocks_per_group;
    struct ext4_group_info ** __rcu *s_group_info;

    /* 日志 */
    struct journal_s *s_journal;

    /* 多块分配器 */
    struct inode *s_buddy_cache;             // Buddy缓存 inode
    struct xarray *s_mb_avg_fragment_size;   // 各阶平均碎片大小
    struct xarray *s_mb_largest_free_orders; // 各组最大空闲阶
    struct ext4_locality_group __percpu *s_locality_groups;

    /* 统计计数器 */
    struct percpu_counter s_freeclusters_counter;
    struct percpu_counter s_freeinodes_counter;
    struct percpu_counter s_dirtyclusters_counter;

    /* 分配调优 */
    unsigned int s_mb_stream_request;  // 流式分配阈值（默认16块）
    unsigned int s_mb_max_to_scan;     // 最大扫描次数
    unsigned int s_mb_group_prealloc;  // 组预分配大小（默认512块）

    /* 孤儿文件 */
    struct ext4_orphan_info s_orphan_info;
};
```

---

## 五、核心子系统详解

### 5.1 Extent 树管理（`extents.c`）

#### 核心操作函数

| 函数 | 行号 | 功能 |
|------|------|------|
| `ext4_find_extent()` | ~870 | 查找给定逻辑块对应的 extent，返回路径 |
| `ext4_ext_insert_extent()` | ~2000 | 插入新 extent（含合并和分裂逻辑） |
| `ext4_ext_remove_space()` | ~3300 | 删除逻辑块范围（截断用） |
| `ext4_ext_map_blocks()` | ~4200 | 核心映射函数，整合查找+分配 |
| `ext4_ext_split()` | ~1400 | 节点满时分裂节点 |
| `ext4_ext_grow_indepth()` | ~1550 | 树层数增加 |
| `ext4_ext_search_right()` | ~1536 | 查找右侧相邻 extent（用于合并） |

#### Extent 操作流程

```
查找：ext4_find_extent(inode, lblk, path)
  ├─ 从 i_block 读取根 header
  ├─ 若 depth>0：二分查找索引节点，循环向下
  └─ 返回叶节点路径 (struct ext4_ext_path[])

插入：ext4_ext_insert_extent(handle, inode, path, newext)
  ├─ 尝试与前/后 extent 合并
  ├─ 叶节点有空位：直接插入
  ├─ 叶节点满：ext4_ext_split() 分裂节点
  │   ├─ 分配新块
  │   ├─ 移动半数条目
  │   └─ 更新父索引
  └─ 树满：ext4_ext_grow_indepth() 增加层数
```

### 5.2 延迟分配机制（Delayed Allocation）

这是 ext4 性能的核心优化。写入时**不立即分配物理块**，推迟到 writeback 阶段批量分配。

```
写路径（延迟分配）：
  write() → ext4_da_write_begin()
    └─ ext4_da_map_blocks()
        ├─ 查 ES 树，若已有 DELAYED extent：直接返回
        └─ 否则：在 ES 树中插入 DELAYED extent（无物理块）
                记录 i_reserved_data_blocks++

  writeback触发 → ext4_writepages()
    └─ ext4_map_blocks(flags=CREATE)
        ├─ 调 ext4_mb_new_blocks()：实际分配物理块
        ├─ 调 ext4_ext_insert_extent()：写入磁盘 extent 树
        └─ 更新 ES 树为 WRITTEN 状态

优势：
  - 多个小写合并为一次大分配，减少碎片
  - 降低日志事务数量
  - 获得更好的空间局部性
```

### 5.3 多块分配器（`mballoc.c`）

#### Buddy 分配系统

ext4 为每个块组维护一个 Buddy 位图，支持 O(log N) 时间的连续块查找：

```
ext4_buddy {
  bd_bitmap  - 块组原始位图
  bd_buddy   - Buddy 位图（伙伴系统标志）
  bd_info    - ext4_group_info（空闲块计数、各阶统计）
}

ext4_group_info {
  bb_counters[]  - 各 2^n 阶的空闲块组数
  bb_free        - 总空闲块数
  bb_first_free  - 第一个空闲块偏移
}
```

#### 分配标准（优先级递减）

```c
enum criteria {
  CR_POWER2_ALIGNED,  // 2^N 对齐分配（最快，最优局部性）
  CR_GOAL_LEN_FAST,   // 从目标位置快速搜索
  CR_GOAL_LEN_SLOW,   // 全组慢速扫描
  CR_BEST_AVAIL_LEN,  // 最优可用长度
  CR_PHYS_LINEAR_GOAL,// 物理线性目标
  CR_PHYS_LINEAR,     // 物理线性扫描
  CR_ANY_FREE,        // 任意空闲块（最后手段）
};
```

#### 预分配策略

```
1. Inode 预分配（MB_INODE_PA）：
   - 大文件（>16 块）
   - 预分配量基于文件大小：16KB→32KB→...→8MB
   - 保存在 ext4_prealloc_space 红黑树

2. 本地性组预分配（MB_GROUP_PA）：
   - 小文件（≤16 块）
   - per-CPU 结构，减少锁竞争
   - 复用已有空间给同CPU上的新分配

分配流程（ext4_mb_new_blocks）：
  1. 查 inode 预分配列表
  2. 查本地性组预分配
  3. ext4_mb_normalize_request()：规范化请求大小
  4. ext4_mb_find_by_goal()：从目标块号搜索
  5. Buddy扫描：从CR_POWER2_ALIGNED 到 CR_ANY_FREE 降级
  6. 标记位图、返回分配结果
```

### 5.4 Extent 状态树（`extents_status.c`）

每个 inode 维护一棵内存红黑树，缓存逻辑块到物理块的映射状态，**避免频繁查磁盘 extent 树**：

```
struct extent_status {
  rb_node          - 红黑树节点
  es_lblk          - 逻辑块起始
  es_len           - 长度
  es_pblk          - 物理块（高5位编码状态）
}

状态类型（存储在 es_pblk 高位）：
  EXTENT_STATUS_WRITTEN   - 已分配且已写入数据
  EXTENT_STATUS_UNWRITTEN - 已分配但未写入（预分配/异步IO）
  EXTENT_STATUS_DELAYED   - 延迟分配（无物理块）
  EXTENT_STATUS_HOLE      - 文件空洞
  EXTENT_STATUS_REFERENCED- 近期被访问（避免过早淘汰）

关键操作：O(log n)
  ext4_es_lookup_extent()  - 查找
  ext4_es_insert_extent()  - 插入（可能触发分裂）
  ext4_es_remove_extent()  - 删除（截断时）
```

### 5.5 日志子系统（`ext4_jbd2.c`）

#### JBD2 集成

ext4 使用 JBD2（Journaled Block Device v2）实现崩溃一致性：

```
三种日志模式：
  data=journal   - 数据和元数据都记入日志（最安全，最慢）
  data=ordered   - 数据先写磁盘，再写元数据日志（默认，平衡）
  data=writeback - 仅元数据日志，数据不保证顺序（最快）

事务信用（Credit）成本：
  单数据块写入（extent模式）= 20个日志块
    ├─ 1个 inode 块
    ├─ 1个块位图
    ├─ 最多5级 extent 树节点
    ├─ 块组摘要更新
    └─ 超级块

日志操作 API：
  ext4_journal_start()              - 开始事务（指定credit数）
  ext4_journal_get_write_access()   - 获取缓冲区修改权
  ext4_mark_iloc_dirty()            - 标记 inode 脏
  ext4_journal_stop()               - 结束事务（触发可能的提交）
```

#### 快速提交（`fast_commit.c`）

对传统完整日志的优化，只记录**增量变更**而非完整块内容：

```
Fast Commit TLV 标签格式：
  ADD_RANGE  (0x0001) - extent 添加记录
  DEL_RANGE  (0x0002) - extent 删除记录
  CREAT      (0x0003) - 文件/目录创建 + 目录条目
  LINK       (0x0004) - 目录条目链接
  UNLINK     (0x0005) - 目录条目删除
  INODE      (0x0006) - inode 元数据快照
  PAD        (0x0007) - 填充对齐
  TAIL       (0x0008) - 提交尾部（含CRC，保证原子性）
  HEAD       (0x0009) - 提交头部（特性信息）

提交流程（8步）：
  1. 标记 inode EXT4_STATE_FC_FLUSHING_DATA
  2. 刷新数据缓冲区到磁盘
  3. 锁定日志（阻止新 handle）
  4. 标记 inode EXT4_STATE_FC_COMMITTING
  5. 解锁日志（其他 inode 可继续）
  6. 写入目录变更 CREAT/LINK/UNLINK 标签
  7. 写入 inode 元数据 INODE 标签
  8. 写入 TAIL 标签（含 CRC，触发持久化）

恢复流程（3阶段）：
  SCAN   - 扫描 FC 区域，建立分配范围图
  REPLAY - 执行各 FC 操作，更新 inode
  INODES - 验证 inode 引用计数，修复不一致

不适用场景（回退到完整提交）：
  - xattr 修改
  - 目录重命名
  - fallocate 操作
  - 加密文件名
  - 日志数据模式
```

### 5.6 孤儿 Inode 管理（`orphan.c`）

处理"已 unlink 但仍被打开"的 inode，防止崩溃后泄漏：

```
传统方式：orphan 链表（超级块中 s_last_orphan 链表）
现代方式：orphan 文件（COMPAT_ORPHAN_FILE 特性）
  - 专用 inode 存储 orphan 记录
  - 避免链表操作的竞争

崩溃恢复：
  挂载时扫描 orphan 链表/文件
  → 对每个 orphan inode：
    - 若 nlink=0：截断并释放
    - 若 nlink>0：仅截断到正确大小
```

---

## 六、目录子系统

### 6.1 目录条目格式

```c
// 现代格式（rev > 0）
struct ext4_dir_entry_2 {
    __le32  inode;          // Inode 号
    __le16  rec_len;        // 记录长度（4字节对齐）
    __u8    name_len;       // 文件名长度
    __u8    file_type;      // 文件类型（普通/目录/符号链接/块设备/字符设备/FIFO/Socket）
    char    name[];         // 变长文件名
};

// 块尾校验（metadata_csum 特性）
struct ext4_dir_entry_tail {
    __le32  det_reserved_zero1;
    __le16  det_rec_len;
    __u8    det_reserved_zero2;
    __u8    det_reserved_ft;
    __le32  det_checksum;   // CRC32C 校验
};
```

### 6.2 Hash 树索引（HTree）

大目录使用 B树索引加速查找：

```
两级结构：
  Level 0：索引块（hash → 叶块指针）
  Level 1：叶块（实际目录条目）

查找流程：
  1. 计算文件名 hash（多种算法：legacy, half-md4, tea, siphash）
  2. 在索引块二分查找 hash
  3. 定位叶块，线性扫描匹配条目

支持：
  - CASEFOLD：大小写不敏感（UTF-8 NFC）
  - LARGEDIR：3级 htree（>2GB 或超大目录）
  - ENCRYPTED：加密文件名（hash 存储在 dirent 中）
```

---

## 七、高级特性

### 7.1 扩展属性（`xattr.c`）

```
双存储策略：

1. Inode 内部 EA（ibody）：
   位于 128字节基础 inode 之后的 extra 空间
   由 ext4_xattr_ibody_header 管理

2. 外部 EA 块：
   单独块，由 i_file_acl 指向
   多 inode 可共享相同 EA 块（引用计数）
   通过 mbcache 缓存

命名空间（10种）：
  USER(1), POSIX_ACL_ACCESS(2), POSIX_ACL_DEFAULT(3),
  TRUSTED(4), SECURITY(6), SYSTEM(7), ENCRYPTION(9) ...

大型 EA（EA_INODE 特性）：
  值 > 16KB 时，存入独立 inode
  通过 e_value_inum 字段引用
```

### 7.2 内联数据（`inline.c`）

```
布局：
  i_block[15]（60字节）= 文件前60字节数据
  系统 xattr "system.data" = 超出60字节的数据

触发条件：
  - EXT4_FEATURE_INCOMPAT_INLINE_DATA 已开启
  - 文件小于 max_inline_size（通常 ~4092 字节）

转换为 Extent：
  当文件增大超出内联阈值时自动转换：
  1. 分配数据块
  2. 拷贝内联数据
  3. 清除 INLINE_DATA_FL，设置 EXTENTS_FL
```

### 7.3 加密（`crypto.c` + fscrypt）

```
架构：
  ext4 只负责：
  - 存取加密上下文（作为 xattr: EXT4_XATTR_INDEX_ENCRYPTION）
  - 提供钩子函数（fscrypt_operations）
  实际加密由通用 fscrypt 层完成

加密粒度：
  - 目录级：整个目录树继承策略
  - 文件名：磁盘存储为密文，读时解密

约束：
  - 与 DAX 不兼容
  - 内联数据自动转换为 extent
  - 根目录不可加密
```

### 7.4 FS-Verity（`verity.c`）

```
磁盘布局：
  |<---- 文件数据（i_size）--->|[间隙]|[Merkle树]|[描述符]|[描述符大小(4B)]|
                               ↑64K对齐          ↑4K对齐

启用流程：
  1. 加入 orphan 列表（崩溃保护）
  2. 构建 Merkle 树，写入 i_size 之后
  3. 写入 verity 描述符
  4. 原子设置 EXT4_INODE_VERITY 标志
  5. 从 orphan 移除

崩溃恢复：
  若挂载时发现 orphan 中有 verity-in-progress 的 inode，
  则截断其元数据，允许重试。
```

---

## 八、数据完整性机制

### 8.1 元数据校验和（CRC32C）

| 被保护的结构 | 校验字段 |
|------------|---------|
| 超级块 | `s_checksum` |
| 块组描述符 | `bg_checksum` |
| Inode | `i_checksum_lo/hi` |
| Extent 块 | 块尾部 `ext4_extent_tail.et_checksum` |
| 目录块 | `ext4_dir_entry_tail.det_checksum` |
| 块/inode 位图 | 独立校验字段 |

### 8.2 块地址合法性（`block_validity.c`）

```
通过红黑树记录所有"系统区域"块（超级块副本、块组描述符、
inode表、位图块），写入前验证目标块不在系统区域内，
防止元数据损坏导致的数据覆写。
```

### 8.3 多挂载保护（`mmp.c`）

```
通过专用块的心跳机制检测多节点同时挂载同一分区，
防止分布式环境下的元数据损坏。
```

---

## 九、特性标志（Feature Flags）

### 兼容特性（COMPAT）—— 老内核可读写

| 标志 | 功能 |
|------|------|
| `HAS_JOURNAL` | 支持日志 |
| `EXT_ATTR` | 扩展属性 |
| `RESIZE_INODE` | 在线扩容 |
| `DIR_INDEX` | Hash 树目录索引 |
| `SPARSE_SUPER2` | 稀疏超级块副本 |
| `FAST_COMMIT` | 快速提交 |
| `ORPHAN_FILE` | 孤儿文件 |
| `STABLE_INODES` | 稳定 inode 号 |

### 只读兼容特性（RO_COMPAT）—— 老内核可只读挂载

| 标志 | 功能 |
|------|------|
| `LARGE_FILE` | 支持 >2GB 文件 |
| `HUGE_FILE` | 支持超大文件（块数>2^32） |
| `GDT_CSUM` | 块组描述符校验和 |
| `EXTRA_ISIZE` | 扩展 inode 空间 |
| `METADATA_CSUM` | CRC32C 元数据校验 |
| `BIGALLOC` | 簇分配（多块作为单位） |
| `QUOTA` | 配额支持 |
| `PROJECT` | 项目配额 |
| `VERITY` | fs-verity 支持 |

### 不兼容特性（INCOMPAT）—— 老内核不可挂载

| 标志 | 功能 |
|------|------|
| `FILETYPE` | dirent 包含文件类型 |
| `EXTENTS` | Extent 树映射 |
| `64BIT` | 64位地址支持 |
| `MMP` | 多挂载保护 |
| `FLEX_BG` | 弹性块组 |
| `EA_INODE` | EA值存储于独立 inode |
| `INLINE_DATA` | 内联数据 |
| `ENCRYPT` | 加密 |
| `CASEFOLD` | 大小写不敏感目录 |
| `CSUM_SEED` | 超级块存校验种子 |
| `LARGEDIR` | 大目录（3级htree） |

---

## 十、锁机制与并发

### 锁获取顺序（防死锁）

```
1. i_rwsem          - inode VFS 读写锁（最外层）
2. i_data_sem       - 数据树保护（extent树+ES树修改）
3. i_es_lock        - ES 树读写锁
4. blockgroup_lock  - 块组位图锁（per-group）
5. 日志句柄锁       - JBD2 内部锁
```

### 关键锁说明

| 锁 | 类型 | 保护对象 |
|----|------|---------|
| `i_data_sem` | `rw_semaphore` | extent 树修改、延迟分配计数 |
| `i_es_lock` | `rwlock_t` | ES 红黑树（读多写少） |
| `i_prealloc_lock` | `spinlock_t` | inode 预分配列表 |
| `s_fc_lock` | `spinlock_t` | 快速提交队列 |
| `xattr_sem` | `rw_semaphore` | EA 块读写 |
| `i_fc_lock` | `spinlock_t` | 快速提交 inode 状态 |

---

## 十一、完整 I/O 数据流

### 11.1 写入路径（延迟分配模式）

```
应用 write()
    │
    ▼ ext4_da_write_begin() [inode.c ~3112]
    │
    ▼ ext4_da_map_blocks()
    │   ├─ 查 ES 树：有 DELAYED extent → 直接返回
    │   └─ 无命中 → 插入 DELAYED extent，i_reserved_data_blocks++
    │
    ▼ 页缓存标脏（mark_page_buffers_dirty）
    │
    ▼ [writeback 触发] ext4_writepages()
    │
    ▼ ext4_map_blocks(CREATE) [inode.c ~693]
    │   ├─ ext4_mb_new_blocks() - Buddy 分配器分配物理块
    │   │   ├─ ext4_mb_normalize_request() - 规范化大小
    │   │   ├─ 查预分配 → 查目标位置 → Buddy扫描
    │   │   └─ 返回 ext4_fsblk_t 物理块
    │   ├─ ext4_ext_insert_extent() - 写入磁盘 extent 树
    │   └─ 更新 ES 树为 WRITTEN
    │
    ▼ jbd2_journal_dirty_metadata() - 提交到日志
    │
    ▼ 事务提交（完整 or 快速提交）
    │
    ▼ 块设备写入
```

### 11.2 读取路径

```
应用 read()
    │
    ▼ ext4_readpage() / ext4_readahead()
    │
    ▼ ext4_map_blocks(READ) [inode.c]
    │   ├─ 查 ES 树（O(log n)）
    │   │   ├─ WRITTEN → 有物理块，直接提交 BIO
    │   │   ├─ UNWRITTEN → 返回零填充
    │   │   ├─ DELAYED → 理论上不应在读路径出现
    │   │   └─ HOLE → 零填充
    │   └─ ES 树缺失 → ext4_find_extent() 查磁盘树
    │
    ▼ bio_submit() → 块设备读
```

---

## 十二、配置与调优参数（sysfs）

通过 `/sys/fs/ext4/<device>/` 暴露：

| 参数 | 默认值 | 含义 |
|------|--------|------|
| `mb_stream_request` | 16块 | 小于此值走流式分配（局部性组） |
| `mb_max_to_scan` | 200 | Buddy 最大扫描次数 |
| `mb_min_to_scan` | 10 | Buddy 最小扫描次数 |
| `mb_group_prealloc` | 512块 | 组预分配大小 |
| `mb_prefetch` | 32 | 预取块组数 |
| `mb_optimize_scan` | 1 | 启用最优组扫描 |
| `max_writeback_mb_bump` | 128MB | writeback 最大合并大小 |

---

## 十三、关键文件路径索引

| 功能 | 文件 | 关键函数 |
|------|------|---------|
| 块映射核心 | `fs/ext4/inode.c:693` | `ext4_map_blocks()` |
| 延迟分配写 | `fs/ext4/inode.c:3112` | `ext4_da_write_begin()` |
| Extent查找 | `fs/ext4/extents.c:870` | `ext4_find_extent()` |
| Extent插入 | `fs/ext4/extents.c:2000` | `ext4_ext_insert_extent()` |
| 多块分配 | `fs/ext4/mballoc.c:6238` | `ext4_mb_new_blocks()` |
| 分配规范化 | `fs/ext4/mballoc.c:4521` | `ext4_mb_normalize_request()` |
| ES树查找 | `fs/ext4/extents_status.c` | `ext4_es_lookup_extent()` |
| 超级块挂载 | `fs/ext4/super.c` | `ext4_fill_super()` |
| 快速提交 | `fs/ext4/fast_commit.c` | `ext4_fc_commit()` |
| 目录查找 | `fs/ext4/namei.c` | `ext4_find_entry()` |

---

*基于 Linux 6.19.6 `fs/ext4/` 源码分析，共 45 个源文件，约 2MB 代码。*
