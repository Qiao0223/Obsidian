IDE／Spring 启动时报 **“No beans of ‘RainDataTask’ type found”**，也就是 ApplicationContext 里根本没找到这个 Bean。

# 1. 原因分析

- **Spring Boot 的组件扫描范围**  
    默认情况下，Spring Boot 会从主启动类所在的包（`@SpringBootApplication` 标注的包）及其子包去扫描 `@Component`、`@Service`、`@Configuration` 等注解，并注册成 Bean。
- **测试类包路径不在扫描范围内**  
    你把测试类放在了 `package java;`，这个包并不在主应用的根包（假设是 `com.hslt`）下，自然 Spring 容器启动时不会去扫描 `RainDataTask` 所在的 `com.hslt.quartz.task` 包。
- **Bean 未被注册**  
    虽然在 `RainDataTask` 上加了 `@Component("calculateRainData")`，但如果包根本没被扫描到，Spring 也不会注册它，`@Autowired` 自然取不到。

# 2. 解决方法