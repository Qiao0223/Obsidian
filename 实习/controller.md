# 1. AuthController

- 该类主要用于处理用户认证流程，包括登录、注册、社交账号绑定、退出登录等功能。
- 使用了 `SaToken`（权限管理工具）进行用户认证，利用 `SocialLogin` 进行社交平台的登录和绑定。
- 提供了 API 接口，使得前端可以进行用户登录、注册、切换租户等操作。
## 1.1. **login(@RequestBody String body)**

- **功能**：处理用户的登录请求。
    
- **具体操作**：
    - 解析请求中的 `LoginBody`，验证其有效性。
    - 验证客户端 `clientId` 和 `grantType` 是否正确。
    - 校验租户是否有效。
    - 根据不同的认证策略（如密码、验证码等）调用登录方法。
    - 登录成功后，向登录用户推送一条欢迎信息。

## 1.2. **authBinding(@PathVariable("source") String source, @RequestParam String tenantId, @RequestParam String domain)**

- **功能**：获取社交平台绑定的授权跳转 URL。
    
- **具体操作**：
    - 根据来源 `source` 获取对应的社交平台配置。
    - 调用 `SocialUtils` 获取授权请求，并生成跳转 URL，供用户跳转到社交平台进行授权。

## 1.3. **socialCallback(@RequestBody SocialLoginBody loginBody)**

- **功能**：处理前端回调的社交登录信息，并绑定社交平台账号。
    
- **具体操作**：
    - 校验用户是否已登录。
    - 获取社交平台返回的登录信息。
    - 注册新用户或绑定社交平台账号到现有账户。

## 1.4. **unlockSocial(@PathVariable Long socialId)**

- **功能**：取消社交平台账号的授权绑定。
    
- **具体操作**：
    - 校验用户是否已登录。
    - 删除社交平台的授权记录。

## 1.5. **logout()**

- **功能**：处理用户退出登录请求。
    
- **具体操作**：
    - 调用 `loginService.logout()` 完成退出操作。

## 1.6. **register(@Validated @RequestBody RegisterBody user)**

- **功能**：处理用户注册请求。
    
- **具体操作**：
    
    - 校验是否开启注册功能。
    - 调用 `registerService.register()` 完成注册。

## 1.7. **tenantList(HttpServletRequest request)**

- **功能**：获取当前登录用户所在租户列表。
    
- **具体操作**：
    - 返回是否启用了租户管理功能。
    - 如果启用，返回当前域名下的租户列表，支持超级管理员查看所有租户，普通用户只能查看与当前域名匹配的租户。

# 2. CaptchaController

`CaptchaController` 类是一个处理验证码相关操作的控制器，主要负责生成和发送短信验证码、邮箱验证码，以及图片验证码。它使用了 `RateLimiter` 注解来限制验证码请求的频率，确保系统不被滥用。

## 2.1. **smsCode(@NotBlank String phonenumber)**

- **功能**：发送短信验证码。

- **具体操作**：

    - 验证 `phonenumber` 是否为空。
    - 随机生成一个 4 位数字验证码
    - 将验证码存储到 Redis 中，设置过期时间为 `Constants.CAPTCHA_EXPIRATION`（通常为 5 分钟）。
    - 使用 `SmsBlend` 发送短信验证码，模板 ID 可以根据需求自定义。
    - 如果短信发送失败，记录日志并返回失败响应。
    - 返回操作成功响应。
 
## 2.2. **emailCode(@NotBlank String email)**

- **功能**：发送邮箱验证码。
    
- **具体操作**：
    - 检查是否启用邮箱功能 (`mailProperties.getEnabled()`)。
    - 验证 `email` 是否为空。
    - 随机生成一个 4 位数字验证码。
    - 将验证码存储到 Redis 中，设置过期时间。
    - 使用 `MailUtils` 发送带有验证码的邮件。
    - 如果邮件发送失败，记录错误日志并返回失败响应。
    - 返回操作成功响应。   

## 2.3. **getCode()**

- **功能**：生成图形验证码。
    
- **具体操作**：
    
    - 检查是否启用验证码功能 (`captchaProperties.getEnable()`)，如果禁用则直接返回验证码禁用状态。
    - 使用 `IdUtil.simpleUUID()` 生成一个唯一的 `uuid` 作为验证码标识。
    - 根据配置生成验证码类型（字符验证码或数学验证码）。
    - 使用 `CaptchaType` 设置验证码生成器，并调用 `AbstractCaptcha` 来生成验证码图像。
    - 如果是数学验证码，使用 Spring Expression Language (SpEL) 计算验证码的结果。
    - 将验证码存储到 Redis 中，设置过期时间。
    - 返回包含验证码图像和 UUID 的响应对象。

# 3. IndexController

`IndexController` 类是一个处理访问首页请求的控制器，主要用于返回系统的基础信息和欢迎语。它定义了一个简单的接口，当用户访问根路径 (`/`) 时，返回一段包含系统名称和版本号的欢迎信息。

## 3.1. **index()**

- **功能**：当访问根路径 (`/`) 时，返回系统的欢迎信息。
    
- **具体操作**：
    
    - 调用 `ruoyiConfig.getName()` 获取系统的名称。
    - 调用 `ruoyiConfig.getVersion()` 获取系统的版本号。
    - 使用 `StringUtils.format()` 将系统名称和版本号格式化到欢迎信息中，生成最终的返回内容。
