# 1. POM 的基本结构
```
<project xmlns="http://maven.apache.org/POM/4.0.0" …>
  <modelVersion>4.0.0</modelVersion>

  <!-- ① 坐标 -->
  <groupId>com.example</groupId>
  <artifactId>my-app</artifactId>
  <version>1.0.0</version>
  <packaging>jar</packaging>

  <!-- ② 父 POM -->
  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.5.9</version>
    <relativePath/>
  </parent>

  <!-- ③ 属性 (Properties) -->
  <properties>
    <java.version>1.8</java.version>
    <maven.compiler.plugin.version>3.8.1</maven.compiler.plugin.version>
  </properties>

  <!-- ④ 依赖管理 (Dependency Management) -->
  <dependencyManagement>
    <dependencies>
      <!-- BOM 或统一版本声明 -->
    </dependencies>
  </dependencyManagement>

  <!-- ⑤ 实际依赖 (Dependencies) -->
  <dependencies>
    <!-- 声明自己用的库 -->
  </dependencies>

  <!-- ⑥ 构建插件管理 (Plugin Management) -->
  <build>
    <pluginManagement>
      <plugins>
        <!-- 插件版本统一管理 -->
      </plugins>
    </pluginManagement>
    <plugins>
      <!-- 真正要执行的插件 -->
    </plugins>
  </build>

  <!-- ⑦ 模块列表（多模块项目） -->
  <modules>
    <module>module-a</module>
    …
  </modules>
</project>
```

# 2. `<dependencyManagement>` 与 `<dependencies>` 的区别

|元素|作用|是否自动引入到 classpath|
|---|---|---|
|`<dependencyManagement>`|**只管理版本**，不自动添加依赖；子模块如要使用，仍需在 `<dependencies>` 中声明（可省略 `<version>`）|否|
|`<dependencies>`|**真正引入**依赖，编译／运行时才可见；若父 POM 已在 `<dependencies>` 中声明，则子模块自动继承|是|

- **常见用法**

    - 父 POM `<dependencyManagement>` 中列出所有库的统一版本。
    - 各子模块在自身 `<dependencies>` 中写需要的依赖，无需再写 `<version>`。

```
<!-- 父 POM -->
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-databind</artifactId>
      <version>2.12.6</version>
    </dependency>
  </dependencies>
</dependencyManagement>

<!-- 子 POM -->
<dependencies>
  <dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <!-- 版本由父 POM 管理，无需再写 -->
  </dependency>
</dependencies>
```

# 3. 父 POM (`<parent>`) 的作用

- **继承依赖管理**  
    通过 `<parent>` 引入 `spring-boot-starter-parent`（或自定义的聚合 Parent POM），自动获得一整套 BOM 版本控制和插件管理。
- **插件版本管理**  
    `<pluginManagement>` 由父 POM 提供，子模块只需在 `<plugins>` 中声明要用的插件即可，无需再写 `<version>`。
- **统一属性**  
    `java.version`、`project.build.sourceEncoding` 等，所有子模块自动继承。

# 4. BOM（Bill Of Materials）与 `<scope>import</scope>`

- **BOM** 是一种特殊的 POM，用来集中管理一组相关依赖的版本。
- 在 `<dependencyManagement>` 中，使用：

```
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-dependencies</artifactId>
  <version>2.5.9</version>
  <type>pom</type>
  <scope>import</scope>
</dependency>
```

这时就可在子 POM 直接声明 Spring Boot 的各个 starter，而不再写版本号。

# 5. 依赖作用域（Scope）
|Scope|说明|
|:--|:--|
|**compile** (默认)|编译、运行、测试都可见|
|**provided**|编译可见；运行时由容器提供（如 Servlet API）|
|**runtime**|编译时不可见；运行时可见（如 JDBC 驱动）|
|**test**|仅测试代码可见；打包时不包含|
|**system**|与 `provided` 类似，需要手动提供 `<systemPath>`|
|**import**|仅用于 BOM 导入|
# 6. 传递依赖

- 当 A 依赖 B、B 又依赖 C，默认 A 可以使用 C（传递依赖）。
- 可通过 `<exclusions>` 排除不需要的传递依赖。
```
<dependency>
  <groupId>com.example</groupId>
  <artifactId>A</artifactId>
  <exclusions>
    <exclusion>
      <groupId>org.unwanted</groupId>
      <artifactId>bad-lib</artifactId>
    </exclusion>
  </exclusions>
</dependency>
```

# 7. 插件管理 vs 插件声明

- `<pluginManagement>`：集中管理插件的版本及配置，不自动执行。
- `<build><plugins>`：声明实际要运行的插件，若版本和配置在 `<pluginManagement>` 中已有，则此处可省略。

# 8. 多模块项目的依赖继承

1. **聚合模块 (Aggregator)**：顶层 POM，`<packaging>pom</packaging>`，列 `<modules>`。
2. **子模块 (Module)**：在自己的 POM 中用 `<parent>` 指向聚合模块。
3. **继承关系**：子模块自动继承父的 `<properties>`、`<dependencyManagement>`、`<pluginManagement>`，但不会自动继承父的 `<dependencies>`（除非显式写在父的 `<dependencies>` 中）。

# 9. 最佳实践

- **统一版本管理**：把所有第三方库版本放到顶层 POM 的 `<dependencyManagement>`。
- **按需引用**：子模块只在 `<dependencies>` 中声明自己实际用到的依赖。
- **使用 BOM**：对于一套相关的库（如 Spring Cloud、gRPC、Guava 生态），通过 `<scope>import</scope>` 引入 BOM。
- **隔离测试依赖**：将所有测试工具（JUnit、Mockito、Spring Boot Test）放在 `<dependencyManagement>` 中，由子模块在 test 作用域里声明。
- **插件集中管理**：把常用的 Maven 插件统一配置在父 POM 的 `<pluginManagement>`。