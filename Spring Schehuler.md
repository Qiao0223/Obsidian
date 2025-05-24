Spring 的 Scheduler 是一个轻量级的任务调度框架，主要通过 `@Scheduled` 注解实现定时任务的功能。它基于 Java 的 `ScheduledExecutorService`，并集成了 Spring 的依赖注入和生命周期管理，适用于中小型应用的定时任务需求。

# 1. 基本使用方式

### 1. 启用调度功能

在 Spring Boot 项目中，只需在主类或配置类上添加 `@EnableScheduling` 注解，即可启用调度功能：
```
@SpringBootApplication
@EnableScheduling
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```
### 2. 创建定时任务

使用 `@Scheduled` 注解的方法必须满足以下条件：
- 方法无参数
- 返回类型为 `void
```
@Component
public class ScheduledTasks {

    @Scheduled(fixedRate = 5000)
    public void reportCurrentTime() {
        System.out.println("当前时间：" + new Date());
    }
}
```

# 2. `@Scheduled` 注解参数详解

`@Scheduled` 提供了多种方式来配置任务的执行时间：

- **`fixedRate`**：以固定频率执行任务，单位为毫秒。任务开始后，每隔指定时间再次执行，不考虑上一次任务是否完成。
```
  @Scheduled(fixedRate = 5000)
  public void task() {
      // 每5秒执行一次
  }
```

- **`fixedDelay`**：在上一次任务完成后，延迟指定时间再执行下一次任务，单位为毫秒。
```
  @Scheduled(fixedDelay = 5000)
  public void task() {
      // 上一次任务完成后，延迟5秒执行
  }
```

- **`initialDelay`**：首次执行任务前的延迟时间，单位为毫秒。可与 `fixedRate` 或 `fixedDelay` 结合使用。
```
  @Scheduled(initialDelay = 1000, fixedRate = 5000)
  public void task() {
      // 程序启动1秒后开始执行，每5秒执行一次
  }
```

- **`cron`**：使用 Cron 表达式定义任务的执行时间。
```
  @Scheduled(cron = "0 0 9 * * ?")
  public void task() {
      // 每天上午9点执行
  }
```

Cron 表达式由 6 或 7 个字段组成，依次表示：秒、分、时、日、月、星期（年为可选）。常用符号包括：

- `*`：表示任意值
- `?`：表示不指定值
- `-`：表示范围
- `,`：表示列出枚举值
- `/`：表示步长
例如，`0 0/5 9-17 * * MON-FRI` 表示在每周一至周五的 9 点至 17 点之间，每隔 5 分钟执行一次任务。