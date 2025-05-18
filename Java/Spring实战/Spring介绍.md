# 1. Spring 是什么？

- **Spring 是一个容器**：其核心是 _Spring 应用程序上下文_，用于创建、管理和组装应用中的组件（称为 _bean_）。
- **依赖注入（DI）机制**：组件之间的依赖关系由容器负责注入，而不是组件自己创建依赖。典型做法是通过构造函数或 setter 方法注入。

# 2. Spring 应用组件示例

- 以 `ProductService` 依赖 `InventoryService` 为例，演示 Spring 如何管理和注入 bean（组件）。
- 这些 bean 是通过 Spring 上下文统一管理的，组件之间不直接创建彼此，而是通过配置实现解耦。

# 3. Spring 配置方式的演变

1. **早期配置方式：XML 配置**
    - 示例：通过 `<bean>` 标签声明 `InventoryService` 和 `ProductService` 并设置构造函数注入。
2. **现代推荐方式：基于 Java 的配置**
    - 使用 `@Configuration` 和 `@Bean` 注解定义 bean。
    - 比 XML 更具类型安全、可重构性更强。

# 4. 自动装配与组件扫描

- **组件扫描**：Spring 自动识别和注册被 `@Component`、`@Service` 等注解标记的类。
- **自动装配**：Spring 自动将所需依赖注入进组件，无需手动指定。

# 5. Spring Boot 的引入与作用

- **Spring Boot 是 Spring 的增强版**，主打 “约定优于配置”，提升开发效率。
- **自动配置（Auto-Configuration）** 是其核心特性之一，可以根据类路径和环境自动注册 bean。
- **没有显式配置的代码**：配置看不到，但组件功能却自动启用 —— “如风一般”的自动配置。

# 6. Spring Boot 与传统 Spring 的关系

- Spring Boot 极大简化了 Spring 应用的开发流程。
- 本书将 Spring 和 Spring Boot 视为一体，优先使用 Spring Boot，仅在必要时编写显式配置。
- XML 配置已过时，主推 Java 配置。