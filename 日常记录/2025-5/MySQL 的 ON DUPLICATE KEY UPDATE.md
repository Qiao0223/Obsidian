MySQL 的 `ON DUPLICATE KEY UPDATE` 是一种“插入或更新”二合一的语法糖，用来保证同一主键／唯一索引冲突时，不报错而是执行更新，从而轻松实现幂等性。

# 1. 基本语法

```
INSERT INTO table_name (col1, col2, …)
VALUES (val1, val2, …)
ON DUPLICATE KEY UPDATE
  col1 = VALUES(col1),
  col2 = VALUES(col2), 
  …;
```
- **INSERT 部分**：尝试插入新行。
- **ON DUPLICATE KEY UPDATE 部分**：如果因 **主键** 或 **唯一索引** 冲突，改为执行后面的 `UPDATE` 子句。

# 2. 工作流程

1. **无冲突** → 正常插入一行。
2. **冲突** → 不插入新行，改用 `UPDATE … SET …` 更新已有行。

整个操作在一个原子事务里完成，对用户来说就像一次“幂等写入”：不管执行多少次，表里每个主键（或唯一索引）只有一条记录，内容总是最新的。

# 3. 示例

```
CREATE TABLE user_profile (
  user_id   INT        PRIMARY KEY,
  name      VARCHAR(50),
  last_login DATETIME,
  login_count INT      DEFAULT 0
);

-- 第一次：user_id=1 不存在，插入新行
INSERT INTO user_profile (user_id, name, last_login, login_count)
VALUES (1, 'Alice', NOW(), 1)
ON DUPLICATE KEY UPDATE
  last_login  = VALUES(last_login),
  login_count = login_count + 1;
  
-- 第二次：user_id=1 已存在，触发更新
INSERT INTO user_profile (user_id, name, last_login, login_count)
VALUES (1, 'Alice', NOW(), 1)
ON DUPLICATE KEY UPDATE
  last_login  = VALUES(last_login),
  login_count = login_count + 1;
```

