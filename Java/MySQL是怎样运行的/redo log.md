# 1. 什么是 redo 日志（redo log）

**redo log（重做日志）**是 InnoDB 存储引擎为了**实现事务的持久性（Durability）**而引入的一种**物理日志**，它记录的是**“对页面的修改操作”**，而不是整个页面本身。

# 2. 为什么需要 redo 日志？

1. **事务必须满足持久性**：事务提交后即使系统崩溃，数据也不能丢失。
2. **缓存在 Buffer Pool 中的修改不是立即写入磁盘**：写入磁盘是**懒惰策略（deferred write）**。
3. **强制刷新整个页面开销太大**：
    - 改了一个字节也要刷整个 16KB 页。
    - 多个页面可能随机分布在磁盘上，导致大量**随机 IO**，速度慢。

# 3. redo 日志解决方案
### 思路：
- **只写变更记录，不写整页数据**。
- 记录：哪个表空间、哪个页、哪个偏移位置、被改成了什么值。
### 事务提交时，只做两件事：
1. 把 redo 日志 **顺序写入磁盘**（Write-Ahead Logging，WAL机制）；
2. 返回成功，**不立即刷新数据页**。

# 4. redo 日志的优点
| 优点    | 说明                             |
| ----- | ------------------------------ |
| 写入量小  | 记录的是页内变化的“小补丁”，不是整个页面（16KB）    |
| 顺序写磁盘 | 支持顺序 I/O，速度快，特别对机械硬盘友好         |
| 持久性强  | 事务提交时只要 redo log 写入成功，就算崩了也能恢复 |
# 5. redo 日志的基本结构

redo 日志记录了数据库页的物理或逻辑修改，其通用结构如下：

|字段|含义说明|
|---|---|
|`type`|redo 日志类型（标识是哪种操作）|
|`space_id`|表空间 ID，标识在哪个表空间内|
|`page_no`|页号，指向修改的数据页|
|`data`|具体的变更数据（或变更描述）|
# 6. Mini-Transaction（mtr）

**Mini-Transaction（mtr）** 是 InnoDB 中 **一次对一个或多个页面的原子性修改操作** 的最小单位。
### 特点：
- mtr 是事务内部的最小原子操作单元；
- 每个 mtr 会产出一组 redo 日志；
- mtr 保证这些 redo 日志要么 **全部生效**，要么 **全部无效**；
- 崩溃恢复时以 mtr 为单位进行重做（redo）或放弃（skip）；

## mtr 与 redo 日志的关系结构
```
一个事务
├── 多条语句（SQL）
│   ├── 多个 mtr（Mini-Transaction）
│   │   ├── 多条 redo 日志
```
例如，一条 INSERT 语句可能包含如下 mtr：
- 修改 Max Row ID → 一个 mtr（产生一条 redo）
- 插入聚簇索引 → 一个 mtr（可能产生多条 redo）
- 插入二级索引 → 一个 mtr（可能产生多条 redo）

## redo 日志分组写入的机制
|类型|redo 日志数量|是否需要分组|标识方式|
|---|---|---|---|
|简单 mtr（如 Max Row ID 更新）|1|✅ 需要|在 type 字节中标记首位为 1|
|复杂 mtr（如 B+ 树插入）|多条|✅ 需要|最后一条日志后追加 `MLOG_MULTI_REC_END`|

### 方案细节一：单条 redo 的 mtr

**例如**：只改了一个页面的 8 字节（如 Max Row ID）
- 只写 1 条 redo（如 `MLOG_8BYTE`）
- 利用 type 字节的**首位**标记：`1xxxxxxx` → 表示“我是一整个 mtr，且只有这 1 条日志”

这样就**节省了写 `MLOG_MULTI_REC_END` 的空间**。
### 方案细节二：多条 redo 的 mtr

**例如**：悲观插入导致页分裂、段扩展、父节点目录插入等
- 需要写很多 redo（如 `MLOG_COMP_REC_INSERT`, `MLOG_COMP_PAGE_CREATE`, ...）
- 在最后追加一条特殊的：
    `MLOG_MULTI_REC_END (type = 31)`
**恢复时的规则：**
- **必须成组读取且以 `MLOG_MULTI_REC_END` 结束，才进行 redo**
- 否则忽略本组，避免恢复出不一致状态（如残缺的 B+ 树）

## 为何需要 redo 日志“组”？

为了解决系统崩溃重启时的**数据页一致性**问题：
- redo 的作用是重放崩溃前已提交但未刷盘的操作；
- 如果一组 redo 执行一半，另一半丢失，就可能形成**逻辑错误的状态**（如 B+ 树不完整）；
- 所以 mtr 要**以组为单位全-or-nothing 执行 redo**。
# 7. redo日志的写入过程

## 7.1 redo log block

InnoDB 把 redo 日志写入单位定义为 **512 字节的 block**，每个 block 分为三部分：

|区域|大小|内容|
|---|---|---|
|`Header`|12B|日志管理信息|
|`Body`|496B|实际 redo 日志数据|
|`Trailer`|4B|校验和（checksum）|
**block != 数据页**，但设计类似。数据页是用于表空间存储，log block 是日志空间的页单位。
### Header 中重要字段

|字段名称|说明|
|---|---|
|`LOG_BLOCK_HDR_NO`|当前 block 的编号（递增）|
|`LOG_BLOCK_HDR_DATA_LEN`|block 中已经写入的数据长度（初始为12）|
|`LOG_BLOCK_FIRST_REC_GROUP`|当前 block 第一个 redo 组的起始偏移量|
|`LOG_BLOCK_CHECKPOINT_NO`|对应 checkpoint 编号（用于恢复）|
## 7.2 redo log 的写入流程

### 1. log buffer（日志缓冲区）
- 不是直接将 redo 写入磁盘，而是先写入内存中的 **log buffer**
- 大小由参数 `innodb_log_buffer_size` 决定（默认16MB）
- log buffer 被划分为连续的多个 **redo log blocks**（512B 一块）

**为什么不直接写磁盘？**  
为了避免频繁磁盘 IO，提升写入性能 —— 和 Buffer Pool 缓存数据页的思想一样。

### 2. 写入机制：以 mtr 为单位成组写入

|步骤|描述|
|---|---|
|①|每个 mtr 执行期间产生 redo 日志条目（多条）|
|②|暂存这组 redo 日志（还未写入 buffer）|
|③|mtr 执行完成，**一次性将该组日志写入 log buffer**|
|④|如果当前 block 剩余空间不足，则换下一个 block|
|⑤|使用 `buf_free` 指针跟踪当前写入位置|

redo 是 **顺序写入** log buffer 的（WAL机制，Write Ahead Logging）
# 8. redo日志文件

## 8.1 redo 日志的刷盘时机

InnoDB 中 redo 日志先写入内存中的 `log buffer`，在以下**五种主要情况**下才会同步到磁盘（即日志文件）：

|情况|说明|
|---|---|
|① log buffer 空间不足|`log buffer` 已使用接近一半，触发提前刷盘|
|② 事务提交|**保证持久性（Durability）**，必须将 redo 写入磁盘（WAL原则）|
|③ 后台线程周期性刷盘|每秒自动 flush 一次，提升容错能力（可调）|
|④ 数据库正常关闭|为确保数据完整性，刷新全部日志|
|⑤ 做 checkpoint 操作|结合 LSN 点保证数据页与日志的一致性（后续章节会详细解释）|

redo 日志刷盘与数据页刷盘是**解耦的**。提交事务只要求 redo 落盘，不要求脏页落盘。

## 8.2 redo 日志文件组（文件组织结构）
###  redo 日志默认文件

位于数据目录中：
`ib_logfile0 ib_logfile1`
### 可配置参数（常用于调优）

|参数|含义|
|---|---|
|`innodb_log_group_home_dir`|redo 文件的存储目录|
|`innodb_log_file_size`|每个 redo 文件的大小，默认 48MB（5.7.21）|
|`innodb_log_files_in_group`|redo 文件个数，默认 2，最大 100|
### 写入方式：**循环使用**
- 从 `ib_logfile0` 开始顺序写；
- 写满后切换到下一个文件；
- 全部写满后 **循环回到头部**；
- ⚠️ 如果 checkpoint 进度滞后，新日志可能覆盖旧日志，因此必须有 checkpoint 限制。
总 redo 容量 = `innodb_log_file_size` × `innodb_log_files_in_group`

### redo 日志文件的格式结构
```
[ Block 0 ]  --> log file header（基本信息）
[ Block 1 ]  --> checkpoint1（记录上次检查点）
[ Block 2 ]  --> 未使用（保留）
[ Block 3 ]  --> checkpoint2（同 checkpoint1 备用）
[ Block 4 ~ 末尾 ] --> redo log block（真正存 redo 日志）
```