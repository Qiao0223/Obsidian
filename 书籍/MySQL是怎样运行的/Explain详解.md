# 1. 什么是执行计划？

- MySQL 优化器对查询进行各种优化（基于规则、成本估算等）之后，会生成一个 **执行计划（Execution Plan）**。
- 执行计划展示了 MySQL **如何实际执行你的查询**，例如：
    - 表的访问顺序
    - 是否使用了索引
    - 多表连接的方式（嵌套循环 / 索引连接等）

# 2. EXPLAIN 的字段
| 列名              | 作用                                                                            |
| --------------- | ----------------------------------------------------------------------------- |
| `id`            | 当前 SELECT 查询的编号。嵌套子查询/派生表的层级结构用 id 区分。                                        |
| `select_type`   | 查询的类型（如 SIMPLE、PRIMARY、SUBQUERY、DERIVED 等）。                                   |
| `table`         | 当前正在访问的表名或派生表名。                                                               |
| `partitions`    | 查询中匹配的分区信息（用于分区表）。                                                            |
| `type`          | 表访问方式（性能重要字段，后面详解）。如 `ALL`, `index`, `ref`, `const` 等。                        |
| `possible_keys` | 查询时可能用到的索引列表。                                                                 |
| `key`           | 实际选用的索引。                                                                      |
| `key_len`       | 实际使用索引的长度（单位是字节），越长通常匹配度越高。                                                   |
| `ref`           | 与索引列进行等值比较的对象（如常量、列名等）。                                                       |
| `rows`          | 预估扫描的记录行数（影响查询开销预估）。                                                          |
| `filtered`      | 过滤后剩余记录比例（%），结合 `rows` 估算最终行数。                                                |
| `Extra`         | 额外信息提示，如 `Using where`, `Using index`, `Using temporary`, `Using filesort` 等。 |
# 3. table

- `table` 字段用于**标识当前这条执行计划记录对应的是哪个表**。
- 每条记录描述的是 **对某个单表的访问方式**，哪怕是一个多表连接查询，EXPLAIN 也会将其拆解为对多个表的单独访问，并逐条输出。

# 4. id

- **每出现一个 `SELECT` 关键字**，MySQL 就会为其分配一个唯一的 `id` 值。
- **同一个 id 值表示属于同一个查询块（SELECT 层级）**。
- `id` 是 EXPLAIN 输出中用于表示查询层次结构的最重要字段。

|情况|表现|
|---|---|
|单表查询|一个 id，通常为 1|
|多表连接|所有表共用一个 id，表示一个查询块|
|子查询|每个子查询分配一个新 id|
|子查询被改写连接|所有表 id 相同，表示已被重写|
|UNION|每个查询分配 id，UNION RESULT 行 id 为 NULL|
|UNION ALL|没有 UNION RESULT，id 不会出现 NULL|
# 5. select_type

- `select_type` 表示当前这条记录所属的 **查询块** 在整个查询语句中的角色或作用。
- 每个 `SELECT` 对应一个 `select_type`。
- 不同类型反映了优化器对查询结构的理解与执行方式（如是否物化、是否依赖外部查询等）。

| 名称                     | 说明                            |
| ---------------------- | ----------------------------- |
| `SIMPLE`               | 不含子查询或 UNION 的简单查询            |
| `PRIMARY`              | 最外层查询（主查询）                    |
| `UNION`                | UNION 或 UNION ALL 中第二个及以后的子查询 |
| `UNION RESULT`         | UNION 操作的结果集（临时表）             |
| `SUBQUERY`             | 非相关子查询（独立执行一次，可物化）            |
| `DEPENDENT SUBQUERY`   | 相关子查询（依赖外层查询字段，可能会被多次执行）      |
| `DEPENDENT UNION`      | 在相关子查询中，UNION 的第二个及后续部分       |
| `DERIVED`              | 派生表（子查询出现在 FROM 子句中，可能被物化）    |
| `MATERIALIZED`         | 子查询被物化后作为临时表与主查询连接            |
| `UNCACHEABLE SUBQUERY` | 子查询每行都必须重新执行，不能缓存结果（不常见）      |
| `UNCACHEABLE UNION`    | 用于不能缓存结果的 UNION（不常见）          |
# 6. type

`type` 表示 **MySQL 优化器决定如何访问某个表**，是执行计划中的核心字段之一。
 **性能从好到差排序：**  
 `system` > `const` > `eq_ref` > `ref` > `ref_or_null` > `index_merge` > `range` > `index` > `ALL`

| 访问方式 (`type`)     | 描述                                     | 是否用索引 | 示例说明                                             |
| ----------------- | -------------------------------------- | ----- | ------------------------------------------------ |
| `system`          | 表只有一行，且存储引擎统计信息精准（常用于 MyISAM、Memory）   | ✅     | 小表查询（仅 1 条记录）                                    |
| `const`           | 主键/唯一索引 与 常量比较，优化器判断结果最多一行             | ✅     | `WHERE id = 5`                                   |
| `eq_ref`          | 被驱动表通过主键/唯一索引，**多表连接**中使用              | ✅     | `s1.id = s2.id` 且 `s2.id` 为主键                    |
| `ref`             | 普通索引列等值查询                              | ✅     | `WHERE key1 = 'a'`                               |
| `ref_or_null`     | `ref` 的变体，支持 `OR IS NULL` 查询           | ✅     | `WHERE key1 = 'a' OR key1 IS NULL`               |
| `index_merge`     | 使用多个索引联合（并集、交集、排序并集）                   | ✅多个   | `WHERE key1 = 'a' OR key3 = 'a'`                 |
| `unique_subquery` | 子查询优化为 `EXISTS`，并通过主键做等值匹配             | ✅     | `IN (SELECT id FROM s2 WHERE s1.key1 = s2.key1)` |
| `index_subquery`  | 与 `unique_subquery` 类似，但使用普通索引         | ✅     | `IN (SELECT key3 FROM s2 ...)`                   |
| `range`           | 使用索引范围扫描（`BETWEEN`、`> <`、`IN (...)` 等） | ✅     | `WHERE key1 > 'a' AND key1 < 'b'`                |
| `index`           | **扫描整个索引树**（不读表数据，只读索引，常用于覆盖索引）        | ✅     | 只查询索引字段，无 WHERE                                  |
| `ALL`             | **全表扫描**，MySQL 需要访问表中所有数据              | ❌     | 没有索引/未命中索引                                       |
## 调优建议

|type 值|优化建议|
|---|---|
|`ALL`|添加合适索引（WHERE、JOIN 字段）|
|`index`|虽然使用了索引，但仍是全索引扫描，考虑优化 WHERE 条件或减少字段|
|`range`|通常较好，注意索引列是否满足选择性要求|
|`ref_or_null`|尽量避免使用 `IS NULL`，或者加索引提高效率|
|`index_merge`|考虑是否能通过联合索引代替多个单列索引合并|
|`unique_subquery`, `index_subquery`|可以说明优化器成功转换了 IN 子查询，可进一步分析优化方式|
# 7. possible_keys 与 key

|字段名|含义|
|---|---|
|`possible_keys`|**优化器认为可能使用的索引集合**，即通过分析 WHERE 或 JOIN 条件推断出的可选索引。|
|`key`|**最终实际选用的索引**，即优化器计算成本后真正决定使用的索引。若为 `NULL`，表示未使用任何索引。|
## 理解背后的优化器行为
###  1. `possible_keys` 是基于 **查询条件分析** 得出的：
- 例如 `WHERE key1 = 'x'` 会命中 `idx_key1`
- 多个字段同时出现，会列出多个可选索引
- 但 **并不意味着都会参与执行**，只是“候选集合”
###  2. `key` 是优化器最终决策：
- 会基于：
    - **索引选择性**
    - **成本估算（rows、IO）**
    - **统计信息准确性**
- 只会显示一个被选中的索引（使用 `index_merge` 时例外）
###  3. 如果 `key` 为 `NULL`：
- 说明没有使用任何索引，极可能是全表扫描（`type = ALL`）
- 也可能是由于：
    - 没有合适索引
    - 条件表达式不能命中索引（例如对索引字段使用了函数）

# 8. key_len

- `key_len` 表示 **MySQL 实际用于查询的索引字节数**。
- 它是评估 **联合索引使用了几个字段** 的核心指标。
- **不是实际占用空间，而是优化器用于判断可匹配索引的长度信息**。
- `key_len` 是 **判断 MySQL 实际使用了联合索引的几个字段** 的核心字段，越接近联合索引总长度，说明利用得越充分。

## 当某字段被用于索引时，其 `key_len` 的计算由以下几部分组成：

|类型|说明|
|---|---|
|固定长度类型|如 `INT` 占用 4 字节|
|字符类型（变长）|`VARCHAR(n)` 使用 `n × 字符集最大字节数`，例如 UTF-8 为 `n × 3`|
|可空字段|可为 NULL 时，**额外增加 1 字节** 用于记录 NULL 值存在性|
|可变长度字段|一律加 **2 字节** 用于记录字段的实际长度（MySQL server 层统一策略）|
## 示例

### 示例 1：INT 主键字段
`EXPLAIN SELECT * FROM s1 WHERE id = 5;`
- `id` 是 `INT NOT NULL`
- 所以 `key_len = 4`
### 示例 2：INT 可空字段
`EXPLAIN SELECT * FROM s1 WHERE key2 = 5;`
- `key2` 是 `INT NULL`
- 所以 `key_len = 4（INT）+ 1（NULL 标记） = 5`
### 示例 3：变长字符字段
`EXPLAIN SELECT * FROM s1 WHERE key1 = 'a';`
- `key1` 是 `VARCHAR(100)`，字符集为 UTF-8（每字符最大 3 字节）
- 可空 ➜ 加 1 字节
- 可变长度 ➜ 加 2 字节
- `key_len = 100 × 3 + 1 + 2 = 303`
### 示例 4：联合索引只用一列
`EXPLAIN SELECT * FROM s1 WHERE key_part1 = 'a';`
- `idx_key_part (key_part1, key_part2, key_part3)`
- 查询只用到 `key_part1` ➜ `key_len = 303`
### 示例 5：联合索引使用两列
`EXPLAIN SELECT * FROM s1 WHERE key_part1 = 'a' AND key_part2 = 'b';`
- 两个字段都是 `VARCHAR(100)`，UTF-8，可变、可空
- 每个字段占 `303` 字节
- `key_len = 303 + 303 = 606`

# 9. ref

- `ref` 表示 **用于与索引列做等值匹配的对象**，即：**匹配条件右边是谁**。
- 仅在以下 `type` 类型中出现：  
    `const`、`eq_ref`、`ref`、`ref_or_null`、`index_subquery`、`unique_subquery`
- 如果该字段为 `NULL`，说明没有等值匹配条件。
## 示例
### 示例 1：与常量匹配
`EXPLAIN SELECT * FROM s1 WHERE key1 = 'a';`
输出节选：
`key: idx_key1 ref: const`
`ref = const`：说明是和一个 **常数值** 进行等值比较。
### 示例 2：表间列匹配（连接查询）
`EXPLAIN SELECT * FROM s1 INNER JOIN s2 ON s1.id = s2.id;`
输出节选：
`table: s2 key: PRIMARY ref: xiaohaizi.s1.id`
`ref = xiaohaizi.s1.id`：说明 **s2 的主键 id** 是与 `s1.id` 进行等值匹配的（跨表连接匹配）。
`ref` 中可能包含数据库名，这是 EXPLAIN 的完整输出风格。
## 示例 3：使用函数作为匹配条件
`EXPLAIN SELECT * FROM s1 INNER JOIN s2 ON s2.key1 = UPPER(s1.key1);`
输出节选：
`key: idx_key1 ref: func`
`ref = func`：说明用的是一个 **函数结果** 与索引列比较，如 `UPPER(s1.key1)`，这通常会**影响索引效率**。

## 其他可能的 `ref` 取值汇总

|`ref` 取值|含义说明|
|---|---|
|`const`|与常量匹配（如 `WHERE key = 1`）|
|`<表名>.<列名>`|与另一个表字段匹配（如 JOIN）|
|`func`|与函数结果匹配（如 `UPPER(key)`）|
|`NULL`|没有等值匹配条件，或使用了 `index` / `ALL` 等非等值方式|
## 应用价值

- **诊断索引是否正确使用：**
    - 是否真的和字段匹配？
    - 是否使用了表达式导致索引无法命中？
- **辅助判断 type 的准确性：**
    - `ref` 为 `const` ➜ `type = const`
    - `ref` 为 `<表列>` ➜ 多用于 `eq_ref` / `ref`
    - `ref` 为 `func` ➜ 要小心，可能造成性能下降

# 10. filtered

- `filtered` 表示：**当前表中符合“剩余搜索条件”记录的比例（百分比）**
- 是 **MySQL 优化器对筛选条件的选择性估算结果**。
- 通常配合 `rows` 字段一起看，用于估算当前表**预计返回多少记录**。

# 11. Extra

### 索引相关优化信息

|Extra 内容|含义说明|
|---|---|
|**Using index**|使用了 **覆盖索引**，即无需回表，直接从索引中取出所有需要的列数据。|
|**Using index condition**|**索引条件下推（ICP）**，在使用索引扫描时部分 WHERE 条件可在索引层完成，减少回表次数。|
|**Using where**|查询中含有额外的过滤条件，这些条件未完全通过索引过滤，需对记录做进一步判断。|
### 连接相关信息

|Extra 内容|含义说明|
|---|---|
|**Using join buffer (Block Nested Loop)**|被驱动表未能使用索引，MySQL 会为其分配内存 buffer 进行块嵌套循环处理。|
|**Not exists**|外连接 + `IS NULL` 判断 + 目标列不可为 NULL，优化器可快速终止对被驱动表的扫描。|
### 索引合并执行策略

|Extra 内容|说明|
|---|---|
|**Using intersect(...)**|使用索引合并的 Intersect 策略。|
|**Using union(...)**|使用索引合并的 Union 策略。|
|**Using sort_union(...)**|使用索引合并的 Sort-Union 策略。|
### 排序 / 临时表

|Extra 内容|说明|
|---|---|
|**Using filesort**|无法使用索引进行排序，需在内存/磁盘中进行“文件排序”。|
|**Using temporary**|查询需要使用临时表（如：`DISTINCT`、`GROUP BY`、`UNION` 等）。|
|**Zero limit**|`LIMIT 0`，无需实际查询记录，直接短路。|
### 半连接 Semi-Join 优化策略相关

|Extra 内容|说明|
|---|---|
|**Start temporary**|半连接的 DuplicateWeedout 策略开始建临时表进行去重。|
|**End temporary**|半连接的 DuplicateWeedout 策略对驱动表执行结束。|
|**LooseScan**|半连接的 LooseScan 策略，使用索引去重扫描。|
|**FirstMatch(tbl_name)**|半连接的 FirstMatch 策略，匹配到第一个结果就停止处理被驱动表。|

### 建议

- `Using index` 是最优的提示（表示覆盖索引），而 `Using filesort` 和 `Using temporary` 往往是优化的重点对象。
- 可以通过以下方法优化这些额外信息带来的性能问题：
    - 尽量构建覆盖索引避免 `Using temporary`
    - 使用合理的索引顺序减少 `Using filesort`
    - 写 `WHERE` 条件时避免复杂函数导致 `Using where` 而不是索引命中
    - 使用 `ANALYZE TABLE` 保证统计信息新鲜，避免估算偏差