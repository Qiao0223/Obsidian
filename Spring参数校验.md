# 1. 概述

## 1.1. Bean Validation 支持
Spring MVC 原生支持 Java Bean Validation，当方法参数使用 `@Valid`（Jakarta Bean Validation 注解）或 Spring 自身的 `@Validated` 时，框架会在绑定后、调用控制器方法前执行校验，违例时抛出 `MethodArgumentNotValidException` 或 `ConstraintViolationException` 等异常，便于统一处理。 [Home](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-validation.html?utm_source=chatgpt.com)[Baeldung](https://www.baeldung.com/spring-boot-bean-validation?utm_source=chatgpt.com)

## 1.2. 引入依赖

在 Spring Boot 中，只需在 `pom.xml`中加入：
```
<dependency>   
<groupId>org.springframework.boot</groupId>   
<artifactId>spring-boot-starter-validation</artifactId> 
</dependency>
```
即可自动引入 Hibernate Validator，省去手动配置。

# 2. 常用校验注解
## 2.1. 空值与空白检查

### `@NotNull`

- **作用**：被注解元素必须不为 `null`，但可以是空字符串或空集合。
- **使用场景**：适用于任何需要确保值存在的场合，如数据库外键、业务必填字段等。 

```
public class UserDto {   
	@NotNull(message = "用户 ID 不能为空")   
	private Long id; 
}
```

### `@NotEmpty`

- **作用**：被注解的集合、数组、Map 或字符串必须不为 `null` 且长度/元素数 > 0。
- **使用场景**：常用于列表、数组或字符串型的必填且至少一个元素/字符的校验。 

```
public class OrderDto {   
	@NotEmpty(message = "商品列表不能为空")   
	private List<String> items; 
}
```

### `@NotBlank`

- **作用**：被注解的字符串必须不为 `null`、且去除空白后长度 > 0。
- **使用场景**：用于文本输入，既排除 `null` 又排除全空白字符串。 

```
public class AccountDto {   
	@NotBlank(message = "用户名不能为空")   
	private String username; 
}
```

## 2.2. 大小与范围检查

### `@Size`

- **作用**：适用于字符串、集合、数组或 Map，检查其元素个数或字符串长度在指定范围内。
- **使用场景**：如用户名长度、列表最大/最小条目数等校验。

```
public class ProfileDto {   
	@Size(min = 2, max = 30, message = "昵称长度必须在2到30之间")   
	private String nickname; }
```

### `@Min` / `@Max`

- **作用**：被注解的数值必须大于等于 `@Min` 或小于等于 `@Max`。
- **使用场景**：整数类型的年龄、数量、分页页码校验等。    

```
public class EmployeeDto {   
	@Min(value = 18, message = "年龄不能小于18")   
	@Max(value = 65, message = "年龄不能大于65")   
	private Integer age; 
}
```

### `@DecimalMin` / `@DecimalMax`

- **作用**：适用于 `BigDecimal`、`String`（数字字符串）等，校验其数值范围（支持小数）。
- **使用场景**：价格、折扣、精确度要求的金额范围校验。 

```
public class ProductDto {   
	@DecimalMin(value = "0.00", inclusive = false, message = "价格必须大于0")   
	private BigDecimal price; 
}
```

### `@Digits`

- **作用**：校验数值的整数位和小数位最大长度。
- **使用场景**：对金额、坐标等数字格式有严格位数要求时。

```
public class InvoiceDto {   
	@Digits(integer = 5, fraction = 2, message = "金额格式错误，整数最多5位，小数最多2位")   
	private BigDecimal amount; }
```

## 2.3. 格式检查

### `@Pattern`

- **作用**：被注解字符串必须匹配给定正则表达式。
- **使用场景**：如校验手机号、用户名格式、自定义复杂规则等。

```
public class ContactDto {
  @Pattern(regexp = "\\d{11}", message = "手机号格式不正确")
  private String phone;
}
```

### `@Email`

- **作用**：校验字符串是否为符合 RFC 标准的电子邮箱格式。
- **使用场景**：邮箱字段的必备校验。

```
public class RegistrationDto {
  @Email(message = "邮箱格式不正确")
  private String email;
}
```

### `@URL`

- **作用**：校验字符串是否为合法的 URL。
- **使用场景**：如校验用户提交的链接、图片地址等。

```
public class LinkDto {
  @URL(message = "链接地址不合法")
  private String url;
}
```

### `@CreditCardNumber`

- **作用**：校验字符串是否为合法的信用卡号（Luhn 算法）。
- **使用场景**：支付场景下卡号校验。 

```
public class PaymentDto {
  @CreditCardNumber(message = "信用卡号无效")
  private String cardNumber;
}
```

## 2.4. 日期与时间检查

### `@Past` / `@PastOrPresent`

- **作用**：校验日期/时间必须早于当前时间（或早于等于）。
- **使用场景**：如出生日期、历史事件时间等。

```
public class EventDto {
  @Past(message = "生日必须是过去的日期")
  private LocalDate birthday;
}
```

### `@Future` / `@FutureOrPresent`

- **作用**：校验日期/时间必须晚于当前时间（或晚于等于）。
- **使用场景**：如预约时间、有效期截止时间等。 

```
public class BookingDto {
  @Future(message = "预约时间必须在未来")
  private LocalDateTime appointment;
}
```

## 2.5. 逻辑检查

### `@AssertTrue` / `@AssertFalse`

- **作用**：校验 `boolean` 或 `Boolean` 字段必须为 `true`/`false`。
- **使用场景**：如勾选同意协议 (`@AssertTrue`)。 

```
public class ConsentDto {
  @AssertTrue(message = "您必须同意用户协议")
  private Boolean termsAccepted;
}
```

# 3. 校验分组与组合

在 Java Bean Validation（JSR-303/JSR-380）规范中，校验注解如 `@NotNull`、`@Size` 等提供了 `groups` 属性，用于指定该约束所属的分组。通过在校验注解中设置 `groups`，可以将不同的校验规则归类。

在使用 `@Validated` 注解时，可以指定要应用的校验分组，框架将只执行该分组中定义的校验规则。

## 3.1. 使用分组校验的步骤

### 1. 定义分组接口

创建空接口作为分组标识：
```
public interface AddGroup {} 
public interface UpdateGroup {}
```
### 2. 在实体类字段上添加校验注解，并指定分组

在需要校验的字段上添加注解，并通过 `groups` 属性指定所属分组

```
public class UserDTO {

    @NotNull(message = "ID不能为空", groups = UpdateGroup.class)
    private Long id;

    @NotBlank(message = "用户名不能为空", groups = {AddGroup.class, UpdateGroup.class})
    private String username;

    @Size(min = 6, max = 20, message = "密码长度应在6到20之间", groups = AddGroup.class)
    private String password;

    // 其他字段和方法...
}
```

在上述示例中：

- `id` 字段仅在更新操作时进行非空校验。
- `username` 字段在新增和更新操作时都进行非空校验。
- `password` 字段仅在新增操作时进行长度校验。

### 3. 在 Controller 方法中指定校验分组

在处理请求的方法参数上使用 `@Validated` 注解，并指定对应的分组：

```
@RestController
@RequestMapping("/users")
public class UserController {

    @PostMapping("/add")
    public String addUser(@Validated(AddGroup.class) @RequestBody UserDTO userDTO) {
        // 处理新增逻辑
        return "用户新增成功";
    }

    @PostMapping("/update")
    public String updateUser(@Validated(UpdateGroup.class) @RequestBody UserDTO userDTO) {
        // 处理更新逻辑
        return "用户更新成功";
    }
}
```

在上述示例中，`addUser` 方法将只执行 `AddGroup` 分组中的校验规则，而 `updateUser` 方法将只执行 `UpdateGroup` 分组中的校验规则。

## 3.2. 注意事项

- **未指定分组的校验注解**：如果校验注解未指定 `groups` 属性，默认属于 `Default` 分组。
- **使用 `@Validated` 指定分组时**：仅会执行指定分组中的校验规则，`Default` 分组的校验规则不会被执行，除非显式地将 `Default.class` 包含在内。
- **使用 `@Valid` 注解时**：只会执行 `Default` 分组中的校验规则，不支持分组校验。
- **分组继承**：可以通过接口继承的方式创建分组序列，实现多个分组的组合校验。

## 3.3. 分组继承

通过接口继承定义分组序列：

```
public interface CreateGroup {}
public interface UpdateGroup {}

@GroupSequence({CreateGroup.class, UpdateGroup.class})
public interface ValidationSequence {}
```

在 Controller 方法中使用分组序列：

```
@PostMapping("/save")
public String saveUser(@Validated(ValidationSequence.class) @RequestBody UserDTO userDTO) {
    // 处理保存逻辑
    return "用户保存成功";
}
```
在上述示例中，`saveUser` 方法将按照 `ValidationSequence` 中定义的顺序，先执行 `CreateGroup` 分组的校验规则，再执行 `UpdateGroup` 分组的校验规则。

通过合理使用分组校验，可以根据不同的业务场景灵活地应用不同的校验规则，提高代码的可维护性和复用性。





















Spring Bean Validation 提供了一系列内置约束注解，常见的包括：

- `@NotNull`：字段不能为空 [reflectoring.io](https://reflectoring.io/bean-validation-with-spring-boot/?utm_source=chatgpt.com)
    
- `@NotBlank`：字符串必须至少包含一个非空白字符 [reflectoring.io](https://reflectoring.io/bean-validation-with-spring-boot/?utm_source=chatgpt.com)
    
- `@Min` / `@Max`：数值范围校验 [reflectoring.io](https://reflectoring.io/bean-validation-with-spring-boot/?utm_source=chatgpt.com)
    
- `@Email`：邮件格式校验 [Baeldung](https://www.baeldung.com/java-validation?utm_source=chatgpt.com)
    
- `@Pattern`：正则表达式校验 [Baeldung](https://www.baeldung.com/java-validation?utm_source=chatgpt.com)
    

## 3.4. 在 Controller 中使用参数校验

### 1. `@Valid` 与 `@Validated`

- `@Valid`：Jakarta Bean Validation 标准注解，可用于方法入参或字段级嵌套校验。 [Medium](https://medium.com/%40himani.prasad016/validations-in-spring-boot-e9948aa6286b?utm_source=chatgpt.com)
    
- `@Validated`：Spring 专属注解，支持校验分组（Groups）功能，通常用于类或方法级别。 [Baeldung](https://www.baeldung.com/spring-valid-vs-validated?utm_source=chatgpt.com)[Stack Overflow](https://stackoverflow.com/questions/36173332/difference-between-valid-and-validated-in-spring/36173922?utm_source=chatgpt.com)
    

### 2. `BindingResult` 用法

当在控制器方法中紧跟 `@Valid` 参数后声明 `BindingResult`，可以手动检查校验结果：

java

复制编辑

`@PostMapping("/create") public ResponseEntity<?> createUser(@RequestBody @Valid UserDto dto, BindingResult result) {     if (result.hasErrors()) {         // 处理错误：result.getFieldErrors()...     }     // 执行业务逻辑 }`

`BindingResult` 提供了 `hasErrors()`、`getFieldErrors()` 等方法，便于在业务层对错误进行细粒度控制。 [Medium](https://medium.com/%40ByteCodeBlogger/bindingresult-an-interface-in-spring-framewr-a-feature-many-developers-dont-use-38d6232292f9?utm_source=chatgpt.com)

## 3.5. 全局异常处理

通过 `@ControllerAdvice` + `@ExceptionHandler`，可统一捕获校验异常并定制返回格式：

java

复制编辑

`@ControllerAdvice public class GlobalExceptionHandler {   @ExceptionHandler(MethodArgumentNotValidException.class)   public ResponseEntity<?> handleValidation(MethodArgumentNotValidException ex) {     // 提取错误信息并返回统一响应   } }`

这样既可避免在每个控制器中重复校验逻辑，也能为前端提供规范化的错误结构。 [Home](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-validation.html?utm_source=chatgpt.com)

## 3.6. 校验分组（Validation Groups）

使用 `@Validated(Group.class)` 可针对不同场景执行差异化校验，例如新增（AddGroup）和编辑（EditGroup）时对同一字段施加不同约束：

java

复制编辑

`public class UserDto {   @NotNull(groups = EditGroup.class)   private Long id;      @NotBlank(groups = {AddGroup.class, EditGroup.class})   private String name; }`

控制器上使用：

java

复制编辑

`@PostMapping("/edit") public void edit(@Validated(EditGroup.class) @RequestBody UserDto dto) { … }`

[reflectoring.io](https://reflectoring.io/bean-validation-with-spring-boot/?utm_source=chatgpt.com)

## 3.7. 自定义校验

### 1. 实现 `Validator` 接口

Spring 提供 `org.springframework.validation.Validator` 接口，可编写自定义校验器：

java

复制编辑

`public class PersonValidator implements Validator {   @Override   public boolean supports(Class<?> clazz) {     return Person.class.equals(clazz);   }      @Override   public void validate(Object target, Errors errors) {     Person p = (Person) target;     if (p.getAge() < 0) {       errors.rejectValue("age", "Negative.person.age");     }   } }`

[Home](https://docs.spring.io/spring-framework/reference/core/validation/validator.html?utm_source=chatgpt.com)

### 2. 注册与调用

- 在控制器方法中使用 `@InitBinder` 注册：
    
    java
    
    复制编辑
    
    `@InitBinder protected void initBinder(WebDataBinder binder) {     binder.addValidators(new PersonValidator()); }`
    
- 或者通过 `DataBinder` 编程方式调用：
    
    java
    
    复制编辑
    
    `DataBinder binder = new DataBinder(person); binder.setValidator(new PersonValidator()); binder.bind(...); binder.validate(); BindingResult results = binder.getBindingResult();`
    

[Home](https://docs.spring.io/spring-framework/reference/core/validation/validator.html?utm_source=chatgpt.com)

## 3.8. 方法级参数校验

借助 AOP，Spring 也支持对 Service 层方法参数或返回值进行校验，只需在配置类或主类上添加 `@Validated` 并在方法参数上使用约束注解：

java

复制编辑

`@Service @Validated public class OrderService {   public void process(@Min(1) Long id, @NotBlank String status) { … } }`

非 HTTP 请求场景亦能享受统一校验机制。 [dev-blog.zymen.net](https://dev-blog.zymen.net/how-to-validate-request-parameters/?utm_source=chatgpt.com)

---

通过以上机制，Spring 的参数校验覆盖了从前端请求入口到后端服务方法的全流程，既提供了开箱即用的标准校验，也支持自定义逻辑与灵活分组，助力开发者构建健壮、可维护的后端应用。