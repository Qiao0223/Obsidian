- 在测试类或方法上加了 `@Transactional`，Spring 会在每个测试方法结束时自动回滚事务。即便你的 Mapper 真地执行了 `INSERT … SELECT`，测试一结束，所有新增的数据都会被撤销。

- **去掉事务或设置不回滚后才能“看到”数据**  
    当你把 `@Transactional` 去掉，或者在测试上加 `@Rollback(false)`，Spring 就不会回滚这笔事务了，`INSERT` 的结果才会真正 **提交** 到数据库，外部查询才能查得到。

- **如何在测试里验证**

    - 如果想在同一个测试里就校验结果，不要依赖外部查看，而应该在方法末尾用 `JdbcTemplate` 或 MyBatis 再 SELECT 一次，Assert 出来。
    - 如果确实要外部查看，则必须去掉自动回滚，或使用 `@Rollback(false)`。