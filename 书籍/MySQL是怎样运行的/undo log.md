# 1. 为什么需要回滚机制？

事务必须遵循 **ACID 的原子性**，即：**要么全做，要么什么都不做**。
## 1.1. 可能导致事务中断的情况
- 事务执行过程中抛出异常（语法、约束、死锁等）；
- 操作系统崩溃（如断电）；
- 用户主动执行 `ROLLBACK`；
- 数据库崩溃重启。
## 1.2. 如何实现“假装没发生过”？
必须在每次修改数据时 **记录一份“撤销信息”**，以便回滚时恢复原状。  
这就是：
> **Undo 日志（undo log）**：记录所有 DML 操作的**反向操作信息**

|操作类型|Undo 内容|
|---|---|
|INSERT|记录“删除该记录”的操作|
|DELETE|记录“还原该记录的内容”的信息|
|UPDATE|记录“修改前的字段值”|
## 1.3. Undo 日志的特征
- Undo log 本质是构造**回滚用的反向操作链**；
- 只有修改操作（INSERT、UPDATE、DELETE）才会生成 undo log；
- SELECT 查询等不会生成 undo log；
- Undo log 也为 MVCC 提供“旧版本数据”。
# 2. 事务 ID（trx_id）
## 2.1. 事务 ID是干啥的？
InnoDB 每次对记录执行写操作时（INSERT/UPDATE/DELETE），都会把当前事务 ID 写入这条记录的 `trx_id` 隐藏列中。
### 作用
- 标识**是谁最后修改了这条记录**；
- 在 MVCC 一致性读时，用来判断**某个事务能不能“看到”这条记录**；
- rollback 或崩溃恢复时辅助判断哪些操作属于哪个事务。
## 2.2. 事务 ID 的分配机制
InnoDB 使用一个**全局自增变量**来分配事务 ID：
### 分配规则如下

|类型|是否分配 trx_id|
|---|---|
|只读事务|只有在修改临时表时才分配|
|读写事务|只有首次修改任意表时才分配|
|查询事务|永远不分配|
|DDL 语句事务|分配（隐式事务）|
###  内存变量管理方式
- 初始值为内存中保存的某个值；
- 每分配一个事务 ID，该值 `+1`；
- 每当事务 ID 达到 256 的倍数时，会同步持久化到系统表空间第 5 页中的 `Max Trx ID` 字段中（用于崩溃恢复）；
- 重启时，从磁盘中读取 `Max Trx ID + 256` 来初始化。
# 3. Undo 日志格式
- 同一事务中，Undo 日志从 `undo no = 0` 开始递增。
- Undo 日志存储在 `FIL_PAGE_UNDO_LOG` 类型（十六进制 0x0002）的页中。
- 页面来源可以是系统表空间，也可以是独立的 undo tablespace。
## 3.1. INSERT 操作对应的 Undo 日志

### 场景描述
- 插入一条记录需要写 Undo 日志，以便回滚时能将其删除。
- 仅记录**主键信息**（用于识别记录并删除）。
### 日志结构要点
- 包含字段：`undo no`、主键各列的 `<len, value>`。
- 主键为单列（如 INT）时，只需记录 4 字节长度和真实值。
- 每个事务插入一条记录，会生成一条 `TRX_UNDO_INSERT_REC`。
```
BEGIN; 
INSERT INTO undo_demo(id, key1, col) VALUES (1, 'AWM', '狙击枪'), (2, 'M416', '步枪');
```
- 会生成两条 Undo 日志：
    - undo no = 0，主键值 = 1
    - undo no = 1，主键值 = 2
## 3.2. DELETE 操作对应的 Undo 日志
### DELETE 的两个阶段
- **阶段一（delete mark）**：设置 `delete_mask = 1`，记录进入“中间状态”（事务未提交前）。
- **阶段二（purge）**：事务提交后，后台线程将其从正常链表移出，加入**垃圾链表**，回收空间。
### 日志结构要点
- 类型为 `TRX_UNDO_DEL_MARK_REC`
- 包含字段：
    - `old_trx_id`、`old_roll_pointer`：可追溯上一个版本日志
    - 所有涉及索引的列的 `<pos, len, value>` 信息
- 构成版本链：如一个事务先插入后删除，Undo 日志会串成链，便于版本回溯
### 存储细节
- 删除记录形成“垃圾链表”，头节点由 `PAGE_FREE` 指向。
- 页面中空闲空间由 `PAGE_GARBAGE` 统计。
- 插入时优先复用 `PAGE_FREE` 指向的已删除记录空间。
- 空间不足或碎片太多时，可触发页重组（性能较低）。
## 3.3. UPDATE操作对应的undo日志
### UPDATE 操作的分类
#### 1. 不更新主键
进一步细分为两种子情形：
- **（1）就地更新（in-place update）**：所有被更新列在更新前后**占用存储空间大小相同**。
- **（2）先删再插**：任意一列的空间大小发生变化（变大或变小）→ 旧记录需从页中删除，再插入新记录。
#### 2. 更新主键
- 主键更新会导致聚簇索引中记录位置发生变化 → 必须将旧记录做 **delete mark**，插入一条新记录。
### UPDATE 不更新主键
#### 1. 就地更新（in-place update）
- 适用条件：**每个被更新列**在更新前后存储空间大小不变。
- 记录内容：直接修改原记录的字段值。
- Undo 类型：`TRX_UNDO_UPD_EXIST_REC`
- Undo 内容包含
    - `n_updated`: 更新列数量
    - 每列的 `<pos, old_len, old_value>`
    - 如果更新了索引列，还会附加 `<索引列各列信息>`（<pos, len, value>）
#### 2. 删除旧记录再插入新记录
- 适用条件：任意列更新前后占用空间大小发生变化。
- 执行步骤：
    1. 从聚簇索引页中真正删除原记录（非 delete mark）。
    2. 同一线程同步插入新记录。
    3. 空间管理机制与垃圾链表复用逻辑类似 `DELETE`。
- Undo 日志：
    - 删除旧记录 → 无需写 `TRX_UNDO_DEL_MARK_REC`，**直接删除**
    - 插入新记录 → 生成 `TRX_UNDO_INSERT_REC`
    - 整体也会记录 `TRX_UNDO_UPD_EXIST_REC` 以便恢复
### UPDATE 更新主键
#### 操作步骤
1. **delete mark 旧记录**
    - 不做真正删除，以保留对并发事务的可见性（MVCC 支持）
    - 写入一条 `TRX_UNDO_DEL_MARK_REC` undo 日志
2. **插入新记录**
    - 创建包含新主键值的新记录并插入
    - 写入一条 `TRX_UNDO_INSERT_REC` undo 日志
#### 特别注意
- 由于主键变化导致记录位置在聚簇索引中需重新定位
- 形成 **2 条 undo 日志**：`DEL_MARK` + `INSERT`
# 4. Undo 页面结构（FIL_PAGE_UNDO_LOG）
## 4.1. Undo 页面结构（FIL_PAGE_UNDO_LOG）

### 页面分类（16KB页）
- 用于存储 Undo 日志的专用页
- 页面类型标记为：`FIL_PAGE_UNDO_LOG`

### 页面结构

|区域|说明|
|---|---|
|`File Header`|通用页头（页号、LSN等）|
|`Undo Page Header`|**Undo 特有字段**，包括页内日志写入偏移和链表连接信息|
|`File Trailer`|页尾校验信息|

### Undo Page Header 字段

| 字段名                   | 说明                                                                                                                                           |
| --------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `TRX_UNDO_PAGE_TYPE`  | 页内 Undo 日志的大类类型（INSERT or UPDATE）  <br>- `1` = TRX_UNDO_INSERT（只放 INSERT 类型）  <br>- `2` = TRX_UNDO_UPDATE（DELETE、UPDATE 等）  <br>**不同类型不能混写** |
| `TRX_UNDO_PAGE_START` | 第一条 Undo 日志起始偏移位置                                                                                                                            |
| `TRX_UNDO_PAGE_FREE`  | 最后一条 Undo 日志结束偏移位置（即下次写入位置）                                                                                                                  |
| `TRX_UNDO_PAGE_NODE`  | 当前页在链表中的 `List Node` 节点（用于串页）                                                                                                                |
## 4.2. Undo 页面链表组织方式

### 单个事务内的 Undo 页面链表
- 一个事务可执行多个语句，每条语句又可能修改多条记录 → 可能产生很多条 Undo 日志。
- **一个 Undo 页放不下** → 多个页组成链表，用 `TRX_UNDO_PAGE_NODE` 串联。

### 链表分类
由于：
1. 同一页中不能混放不同类型的 Undo 日志（Insert vs Update）
2. 普通表与临时表需分开记录（原因后续解释）
因此一个事务**最多会创建 4 类 Undo 页链表**：

|Undo链表类型|触发条件|
|---|---|
|普通表的 insert undo链表|INSERT 语句 / 更新主键|
|普通表的 update undo链表|DELETE / 非主键 UPDATE|
|临时表的 insert undo链表|向临时表插入 / 更新主键|
|临时表的 update undo链表|删除临时表记录 / 非主键 UPDATE|
**分配策略**：按需分配（懒加载）
- 刚开启事务时：**不分配任何链表**
- 哪种操作触发了对应类型的 Undo 日志，就创建相应链表。

### 多个事务下的 Undo 页面链表

- 为提升并发效率，**不同事务使用不同 Undo 页面链表**
- 例如：
    - 事务 trx1 修改普通表和临时表 → 分配3条链表
    - 事务 trx2 只修改普通表 → 分配2条链表
- 所以，多个事务并发执行时，会创建多个链表实例，每个链表独立维护。
好处
- 提高日志写入并发能力（链表并行）
- 避免事务间 undo 空间干扰
- 便于事务提交/回滚时只处理自己链表中的日志
# 5. undo日志具体写入过程
## 5.1. Undo Log Segment Header
每个 Undo 页链表对应一个独立的段（Segment）
- 称为 **Undo Log Segment**
- 位于该链表的第一个页面（`first undo page`）的特殊区域，称为 `Undo Log Segment Header`
- 包含以下字段：

|字段名|含义|
|---|---|
|`TRX_UNDO_STATE`|当前段状态（活跃、待清理、待回收、已缓存、PREPARED）|
|`TRX_UNDO_LAST_LOG`|本链表中最后一个 Undo Log Header 的位置|
|`TRX_UNDO_FSEG_HEADER`|段头信息，即前面提到的 Segment Header（10字节）|
|`TRX_UNDO_PAGE_LIST`|该段内 Undo 页链表的基节点（List Base Node）|
### 状态枚举（`TRX_UNDO_STATE`）：

|状态枚举值|含义|
|---|---|
|`TRX_UNDO_ACTIVE`|当前事务活跃，正在写 Undo 日志|
|`TRX_UNDO_CACHED`|Undo 页面链表被缓存，可供重用|
|`TRX_UNDO_TO_FREE`|INSERT undo 页面链表已不可重用，等待释放|
|`TRX_UNDO_TO_PURGE`|UPDATE undo 页面链表需保留供 MVCC，等待清除|
|`TRX_UNDO_PREPARED`|事务处于分布式 PREPARE 状态（略）|
