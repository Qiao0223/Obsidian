MyBatis-PageHelper 是一款开源的 MyBatis 分页插件，旨在简化分页查询的实现过程。它通过 MyBatis 的拦截器机制，在执行 SQL 查询前自动添加分页逻辑，支持多种数据库（如 MySQL、Oracle、PostgreSQL 等）

# 1. 核心特性

- **物理分页**：直接在 SQL 层添加 `LIMIT` 或 `ROWNUM` 等分页语句，避免内存分页带来的性能问题。
- **多数据库支持**：兼容 MySQL、Oracle、PostgreSQL、SQL Server、SQLite 等主流数据库。
- **多种调用方式**：支持 `PageHelper.startPage()`、`RowBounds`、Mapper 参数传递等多种分页调用方式。
- **分页信息封装**：通过 `PageInfo` 类封装分页结果，提供总记录数、总页数、当前页码等信息，方便前端展示。

# 2. 快速上手

## 2.1. 添加依赖
```
<dependency>
    <groupId>com.github.pagehelper</groupId>
    <artifactId>pagehelper-spring-boot-starter</artifactId>
    <version>1.4.6</version> <!-- 请根据项目需要选择合适的版本 -->
</dependency>
```

## 2.2. 配置参数
```
pagehelper:
  helperDialect: mysql
  reasonable: true
  supportMethodsArguments: true
  params: count=countSql
```

- `helperDialect`：指定数据库类型，PageHelper 会根据此参数生成对应的分页语句。
- `reasonable`：分页合理化参数，设置为 `true` 时，如果页码小于 1 会查询第一页，大于总页数会查询最后一页。
- `supportMethodsArguments`：支持通过 Mapper 接口参数传递分页参数。
- `params`：用于从请求中获取分页参数的映射关系。

## 2.3. 使用示例

Service 层
```
public PageInfo<User> getUsers(int pageNum, int pageSize) {
    PageHelper.startPage(pageNum, pageSize);
    List<User> users = userMapper.selectAll();
    return new PageInfo<>(users);
}
```

Controller 层
```
@GetMapping("/users")
public PageInfo<User> listUsers(@RequestParam int pageNum, @RequestParam int pageSize) {
    return userService.getUsers(pageNum, pageSize);
}
```
