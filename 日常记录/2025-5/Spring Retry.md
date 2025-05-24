# 1. 核心功能概览

- **声明式重试**：通过 `@Retryable` 注解，指定方法在遇到特定异常时进行自动重试。
- **重试策略配置**：支持设置最大重试次数、重试间隔、指数退避等策略。
- **恢复机制**：使用 `@Recover` 注解定义重试失败后的回调方法，进行异常处理或提供默认返回值。
- **编程式重试**：通过 `RetryTemplate` 提供更灵活的编程方式，适用于复杂的重试场景。

# 2. 快速上手

## 2.1. 添加依赖

在 Maven 项目中引入以下依赖
```
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
    <version>1.3.1</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-aspects</artifactId>
    <version>5.3.9</version>
</dependency>
```

## 2.2. 启用重试功能

在 Spring Boot 应用的启动类或配置类上添加 `@EnableRetry` 注解
```
@SpringBootApplication
@EnableRetry
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

## 2.3. 使用 `@Retryable` 注解

在需要重试的方法上添加 `@Retryable` 注解，配置重试策略
```
@Service
public class MyService {

    @Retryable(
        value = { RemoteAccessException.class },
        maxAttempts = 3,
        backoff = @Backoff(delay = 2000, multiplier = 1.5)
    )
    public void performOperation() {
        // 可能抛出 RemoteAccessException 的操作
    }
}
```

## 2.4. 定义恢复方法

当重试次数耗尽后，可以使用 `@Recover` 注解定义恢复方法
```
@Recover
public void recover(RemoteAccessException e) {
    // 处理重试失败的情况
}
```

# 3. 进阶配置

### 重试策略（RetryPolicy）

Spring Retry 提供了多种重试策略，包括：

- **SimpleRetryPolicy**：固定次数重试策略，默认最大重试次数为 3 次。
- **TimeoutRetryPolicy**：在指定的超时时间内进行重试。
- **CircuitBreakerRetryPolicy**：实现断路器模式，当失败次数达到阈值时，停止重试一段时间。
- **CompositeRetryPolicy**：组合多个重试策略，支持乐观和悲观组合方式。

### 退避策略（BackOffPolicy）

用于控制重试之间的等待时间，常见的退避策略有：

- **FixedBackOffPolicy**：固定间隔时间进行重试。
- **ExponentialBackOffPolicy**：指数递增的重试间隔时间。
- **UniformRandomBackOffPolicy**：在指定范围内随机选择重试间隔时间。

# 4. 注意事项

- **异常类型**：`@Retryable` 默认对所有异常进行重试，但可以通过 `value` 属性指定特定的异常类型。
- **方法签名**：`@Recover` 方法的返回类型应与被重试的方法一致，且第一个参数为异常类型。
- **幂等性**：确保被重试的方法是幂等的，以避免因重复执行导致的数据不一致问题。