IDE／Spring 启动时报 **“No beans of ‘RainDataTask’ type found”**，也就是 ApplicationContext 里根本没找到这个 Bean。

# 1. 原因分析

- **Spring Boot 的组件扫描范围**  
    默认情况下，Spring Boot 会从主启动类所在的包（`@SpringBootApplication` 标注的包）及其子包去扫描 `@Component`、`@Service`、`@Configuration` 等注解，并注册成 Bean。
- **测试类包路径不在扫描范围内**  
    你把测试类放在了 `package java;`，这个包并不在主应用的根包（假设是 `com.hslt`）下，自然 Spring 容器启动时不会去扫描 `RainDataTask` 所在的 `com.hslt.quartz.task` 包。
- **Bean 未被注册**  
    虽然在 `RainDataTask` 上加了 `@Component("calculateRainData")`，但如果包根本没被扫描到，Spring 也不会注册它，`@Autowired` 自然取不到。

# 2. 解决方法

- **把测试类移到被扫描的包下**

	- 将测试类放到 `src/test/java/com/hslt/...` 目录，并在文件头声明对应包名 `package com.hslt.quartz;`。
	- 此时 `@SpringBootTest` 会自动扫描到 `com.hslt.quartz.task.RainDataTask`。

- **显式指定扫描路径或启动类**

```
@SpringBootTest(classes = ReservoirManageApplication.class)
@ComponentScan("com.hslt.quartz.task")
```
```
@SpringBootTest(classes = ReservoirManageApplication.class)
```

- **按名称注入（可选）**

    - 如果你给了 `@Component("calculateRainData")` 自定义名称，也可写：
        `@Autowired @Qualifier("calculateRainData") private RainDataTask rainDataTask;`
    - 但前提仍是 Bean 已被扫描并注册。

# 3. 原理小结

- **Component Scan**：Spring Boot 通过 `@ComponentScan`（隐含在 `@SpringBootApplication` 中）来树形扫描包，把符合条件的类注册到 IoC 容器。
- **Bean 注册**：只有被扫描到并标注了 `@Component`/`@Service`/`@Configuration` 等注解的类，才会变成可注入的 Bean。
- **Autowiring**：`@Autowired` 按类型（或按名称配合 `@Qualifier`）从容器里找 Bean，找不到就报 “No beans of type” 错误。
- **测试隔离**：`@SpringBootTest` 默认使用主启动类的包路径做扫描根，如果测试类或 `classes` 属性没正确配置，就可能漏掉某些 Bean。

只要保证 **测试时 Spring 容器能扫描到 `RainDataTask`**，`@Autowired` 就能正常注入，错误自然消失。