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

