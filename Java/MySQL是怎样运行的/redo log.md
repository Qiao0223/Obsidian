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
特点：
- mtr 是事务内部的最小原子操作单元；
- 每个 mtr 会产出一组 redo 日志；
- mtr 保证这些 redo 日志要么 **全部生效**，要么 **全部无效**；
- 崩溃恢复时以 mtr 为单位进行重做（redo）或放弃（skip）；

## 6.1. mtr 与 redo 日志的关系结构
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

## 6.2. redo 日志分组写入的机制
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

## 6.3. 为何需要 redo 日志“组”？

为了解决系统崩溃重启时的**数据页一致性**问题：
- redo 的作用是重放崩溃前已提交但未刷盘的操作；
- 如果一组 redo 执行一半，另一半丢失，就可能形成**逻辑错误的状态**（如 B+ 树不完整）；
- 所以 mtr 要**以组为单位全-or-nothing 执行 redo**。
# 7. redo日志的写入过程

## 7.1. redo log block

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
## 7.2. redo log 的写入流程

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

## 8.1. redo 日志的刷盘时机

InnoDB 中 redo 日志先写入内存中的 `log buffer`，在以下**五种主要情况**下才会同步到磁盘（即日志文件）：

|情况|说明|
|---|---|
|① log buffer 空间不足|`log buffer` 已使用接近一半，触发提前刷盘|
|② 事务提交|**保证持久性（Durability）**，必须将 redo 写入磁盘（WAL原则）|
|③ 后台线程周期性刷盘|每秒自动 flush 一次，提升容错能力（可调）|
|④ 数据库正常关闭|为确保数据完整性，刷新全部日志|
|⑤ 做 checkpoint 操作|结合 LSN 点保证数据页与日志的一致性（后续章节会详细解释）|

redo 日志刷盘与数据页刷盘是**解耦的**。提交事务只要求 redo 落盘，不要求脏页落盘。

## 8.2. redo 日志文件组（文件组织结构）
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
# 9. LSN（Log Sequence Number）
## 9.1. 什么是 LSN

**LSN 是 redo 日志的“全局字节序号”，表示系统总共写入了多少字节的 redo 日志。**
- 类似于日志的“进度计数器”，一旦增加不会减少；
- 每写入一段 redo 日志（含 block header/trailer），LSN 就按写入字节数增加；
- **LSN 单位：字节**。
初始值为 `8704`，因为前四个 block（2048 字节）被保留为 redo 文件头和 checkpoint 信息。
## 9.2. LSN 的核心变量和含义
|变量名|含义|
|---|---|
|`lsn`（当前 LSN）|redo 日志已写入的总字节数（包括 log buffer 中未刷盘部分）|
|`flushed_to_disk_lsn`|redo 日志**已刷到磁盘的部分**的最大 LSN|
|`buf_free`|log buffer 中的写入指针（LSN 增长点）|
|`write_lsn`（可选区分）|已写入 OS 缓冲区但未 `fsync` 的日志范围（非真正持久）|
`lsn` - `flushed_to_disk_lsn` 的差值表示：尚未持久化的 redo 日志大小
## 9.3. LSN 与 redo 刷盘过程
### 日志写入过程：
1. mtr 生成 redo 日志；
2. 写入 `log buffer`，`lsn` 增加；
3. 满足以下任一条件触发刷盘：
    - 事务提交；
    - log buffer 半满；
    - 定时后台线程；
    - checkpoint；
4. 将日志刷新到 redo 文件中（如 `ib_logfile0`）；
5. `flushed_to_disk_lsn` 也随之更新。
## 9.4. ## LSN 与 redo 文件偏移量关系

- 文件起始位置偏移为 2048（前 4 个 block 保留）；
- redo 文件偏移量 = `lsn` - 8704；
- 实现快速映射某条日志在哪个文件、哪个位置。

## 9.5. flush 链表与 LSN 的关系（脏页管理）

InnoDB Buffer Pool 中的页面一旦被修改，就变成**脏页**，加入到 **flush 链表** 中。
### 每个页控制块中包含：

|属性名|含义|
|---|---|
|`oldest_modification`|页第一次被修改时的 mtr 的起始 LSN|
|`newest_modification`|页最后一次被修改时的 mtr 的结束 LSN|
### flush 链表行为规则：
- 页首次被修改：插入 flush 链表头部，设置 `oldest_modification = mtr.lsn_start`
- 页再次被修改：不重复插入，只更新 `newest_modification = mtr.lsn_end`
- flush 链表**按 oldest_modification 递减排序**：新脏页在前，旧脏页在后
Flush策略会优先刷最早被修改的页面（LSN 最小），以推进 checkpoint。
## 9.6. LSN 的作用总结

|用途|功能说明|
|---|---|
|redo 日志唯一标识|每条 redo 日志都有起止 LSN|
|刷盘进度追踪|`flushed_to_disk_lsn` 判断是否持久化完成|
|崩溃恢复起点|checkpoint LSN 告诉系统从哪里开始 redo|
|判断脏页持久状态|页面 `newest_modification` < `flushed_to_disk_lsn` 则安全|
|控制 redo 空间重用|checkpoint 必须保证 redo 不会覆盖未 flush 的日志|
# 10. Checkpoint
## 10.1. 为什么需要 Checkpoint？
### redo 日志是循环使用的！

- InnoDB 的 redo 日志文件是**固定大小**，组成一个**循环使用的日志文件组**；
- 日志写入不断增加，最终会 **“追尾”**：新的 redo 要覆盖旧的 redo。
### 关键问题：
我们不能覆盖的日志 = **还没用完的日志**
- 日志内容**关联着脏页**
- 脏页尚未写入磁盘，崩溃时还得依赖这些 redo 日志恢复
 **解决方法**：  
**只有脏页写入磁盘后，它对应的 redo 日志才可被覆盖**
## 10.2. Checkpoint 的作用

checkpoint 是标志“**哪些 redo 日志可以被重用**”的机制
### 核心变量：`checkpoint_lsn`
- 表示**当前及之前的 redo 日志都可以安全覆盖**
- 即该 LSN 之前的所有 redo 日志对应的脏页都已落盘
### 执行 checkpoint 的作用：
- 释放 redo 空间，避免追尾；
- 缩短崩溃恢复时间（恢复从 checkpoint_lsn 开始）；
- 触发脏页刷新，使数据持久化推进。
## 10.3. Checkpoint 的执行步骤
### 步骤一：找到最早尚未刷盘的页面

- 遍历 flush 链表，找到尾部节点（最早被修改的脏页）
- 读取其控制块中的 `oldest_modification` 值，设为新的 `checkpoint_lsn`

`checkpoint_lsn = min(flush链表中所有页面的 oldest_modification)`
### 步骤二：写入 checkpoint 信息到 redo 文件头
- 包括：
    - `checkpoint_lsn`
    - 该 lsn 对应的 redo 文件偏移量（LSN - 8704）
    - checkpoint 编号：`checkpoint_no`（每次+1）
- 存入文件头中**前4个 block**中的 checkpoint 记录块（第1块或第3块）

**写入哪一块？**
- `checkpoint_no` 是偶数 → 写入 checkpoint1（block 1）
- `checkpoint_no` 是奇数 → 写入 checkpoint2（block 3）
## 10.4. 什么时候触发 checkpoint？
|触发场景|原因说明|
|---|---|
|redo 空间不足|避免日志“追尾”|
|用户事务提交频繁|redo 增长过快，空间压力大|
|定期后台任务（默认每秒）|保证系统耐久性和恢复速度|
|用户线程主动刷页|后台刷不过来了，必须抢救空间|
## 10.5. flush 链表与 checkpoint 的协同机制
### flush链表中的页按 `oldest_modification` 降序排列
- 前面是新修改的页，后面是旧的页；
- checkpoint 只看**最后一个**脏页（链表尾部）
```
flush链表（头 → 尾）:
[页d, o_m=9948] → [页b, o_m=8916] → [页c, o_m=8916]
                         ↑
                checkpoint 以此页的 o_m 为界限
```
- 页被刷盘后，从 flush 链表中移除，checkpoint_lsn 可向前推进
# 11. InnoDB 崩溃恢复机制（Crash Recovery）
**目的：** 系统崩溃后，使用 redo 日志恢复所有未持久化的页面，使数据库处于事务一致性状态（满足持久性 D）。
## 11.1. 确定恢复起点（checkpoint_lsn）

### 原因：
- checkpoint_lsn 之前的 redo 对应脏页已经持久化；
- **只需从 checkpoint_lsn 之后开始恢复**。
### 查找过程：
1. 从第一个 redo 文件头的 checkpoint1、checkpoint2 block 中读取：
    - `checkpoint_no`（哪个大说明更新更晚）
    - 对应的 `checkpoint_lsn` 和 `checkpoint_offset`
2. 从 `checkpoint_offset` 开始扫描 redo 日志。
## 11.2. 确定恢复终点

### LOG_BLOCK_HDR_DATA_LEN 属性决定终点
- redo 是顺序写入的；
- 每个 block 的 header 中记录了实际使用的字节数；
- **最后一个未写满的 block（data_len < 512）为恢复终点**。
## 11.3. 执行恢复过程（redo replay）
### 恢复原则：
- 从 `checkpoint_lsn` 起开始扫描 redo 日志；
- 解析每条 redo 日志的操作（插入、修改、删除）；
- **将对应的页加载到内存中进行恢复重放**；
- 再写入磁盘，完成恢复。
# 12. 两阶段提交

MySQL 中的 **2PC（Two-Phase Commit，两阶段提交）机制** 是为了确保 **redo log 与 binlog 之间的一致性**，解决“事务提交一半崩溃”导致数据不一致的问题，尤其是数据库**崩溃恢复时如何判断事务状态”的核心机制**。
## 12.1. 为什么需要两阶段提交（2PC）？
假设你这样写事务逻辑：
```
BEGIN; 
UPDATE account SET balance = balance - 100 WHERE id = 1; 
INSERT INTO t_log ...; -- 写一条日志 
COMMIT;
```
MySQL 会写入两种日志：
- `redo log`：InnoDB 引擎层的物理数据变更日志
- `binlog`：Server 层的逻辑操作日志（供主从复制、恢复使用）
### 问题来了：**redo log 和 binlog 写入不一致怎么办？**

|情况|redo log|binlog|结果|
|---|---|---|---|
|崩溃前只写 redo|✅|❌|数据已修改，但 binlog 没同步，主从不一致 ❌|
|崩溃前只写 binlog|❌|✅|binlog 记录了提交，但数据没真正修改 ❌|

所以要保证：**两者要么都成功、要么都失败，不能一半一半。**
## 12.2. 两阶段提交过程
由 MySQL Server 层和 InnoDB 引擎层 **协调 redo log 和 binlog 的提交顺序**，保证崩溃后恢复结果一致。
### 2PC 执行流程：
### 阶段一：**prepare 阶段**
由 InnoDB 引擎执行：
1. 写入 **redo log 的 prepare 记录**（物理变更 + 状态标记为 prepare）
2. **刷盘（fsync）**，此时数据物理变更已落盘，但未标记提交状态
如果崩溃发生在 prepare 后、commit 前，此事务处于 “悬而未决” 状态。
### 阶段二：**commit 阶段**
由 MySQL Server 层主导：
1. 写入 **binlog 日志**
2. **fsync 刷盘**
3. 调用 InnoDB 的 `commit()`：
    - 将 redo log 标记为 **commit**
    - 并刷盘
至此，事务正式提交完成。
### 崩溃恢复时的核心判断逻辑
当系统崩溃并重启时，InnoDB 会检查 redo 日志：
- 若存在 **prepare 状态但无 commit 状态** 的事务：
    - 去 binlog 中检查是否写入了对应的事务记录
        - binlog 有 → redo 继续 commit（事务提交）
        - binlog 无 → rollback（事务回滚）
这就是通过 **“双日志交叉确认”** 达到一致性恢复的核心思想。
### 2PC 的关键日志状态对照表
| 阶段       | redo log 状态 | binlog 状态 | 崩溃恢复行为 |
| -------- | ----------- | --------- | ------ |
| 正常提交前    | 无           | 无         | 不处理    |
| prepare后 | prepare     | 无         | 回滚     |
| 写binlog后 | prepare     | 已写        | 提交     |
| 全部完成     | commit      | 已写        | 无需处理   |
