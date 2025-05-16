# 1. 组件声明类注解

| 注解                | 所属          | 说明                                       |
| ----------------- | ----------- | ---------------------------------------- |
| `@Component`      | Spring      | 通用组件                                     |
| `@Service`        | Spring      | 业务逻辑组件                                   |
| `@Repository`     | Spring      | DAO 组件（带异常转换）                            |
| `@Controller`     | Spring      | MVC 控制器                                  |
| `@RestController` | Spring Boot | REST 控制器 (`@Controller + @ResponseBody`) |

---

# 2. 依赖注入 / 装配相关

| 注解           | 所属     | 说明                    |
| ------------ | ------ | --------------------- |
| `@Autowired` | Spring | 按类型自动注入               |
| `@Qualifier` | Spring | 配合 `@Autowired` 按名称注入 |
| `@Resource`  | Spring | JSR-250 标准注入（默认按名称）   |
| `@Value`     | Spring | 配置文件值注入               |

---

# 3. 配置 / Bean 声明

| 注解                | 所属     | 说明        |
| ----------------- | ------ | --------- |
| `@Configuration`  | Spring | 配置类声明     |
| `@Bean`           | Spring | 声明一个 Bean |
| `@Scope`          | Spring | Bean 的作用域 |
| `@PropertySource` | Spring | 引入外部配置文件  |

---

# 4. Spring Boot 核心注解

| 注解                         | 所属          | 说明            |
| -------------------------- | ----------- | ------------- |
| `@SpringBootApplication`   | Spring Boot | 启动主类注解，包含复合注解 |
| `@EnableAutoConfiguration` | Spring Boot | 启用自动配置        |
| `@ComponentScan`           | Spring      | 指定扫描组件包       |

---

# 5. Web 请求处理

| 注解                               | 所属          | 说明               |
| -------------------------------- | ----------- | ---------------- |
| `@RequestMapping`                | Spring      | 路由映射（类/方法都可以用）   |
| `@GetMapping` `@PostMapping` ... | Spring Boot | HTTP 指定方法映射简化版   |
| `@PathVariable`                  | Spring      | 获取路径参数           |
| `@RequestParam`                  | Spring      | 获取请求参数           |
| `@RequestBody`                   | Spring      | 接收请求体 JSON       |
| `@ResponseBody`                  | Spring      | 将返回值直接作为 HTTP 应答 |

---

# 6. 生命周期

| 注解               | 所属     | 说明          |
| ---------------- | ------ | ----------- |
| `@PostConstruct` | Spring | Bean 初始化后执行 |
| `@PreDestroy`    | Spring | Bean 销毁前执行  |

---

# 7. 定时 / 异步处理

| 注解                  | 所属     | 说明       |
| ------------------- | ------ | -------- |
| `@EnableScheduling` | Spring | 启用定时任务支持 |
| `@Scheduled`        | Spring | 声明定时方法   |
| `@EnableAsync`      | Spring | 启用异步方法支持 |
| `@Async`            | Spring | 展示方法异步执行 |

---

# 8. 事务管理

| 注解               | 所属     | 说明     |
| ---------------- | ------ | ------ |
| `@Transactional` | Spring | 展示事务操作 |
