WebSocket 是一种网络通信协议，旨在通过单个 TCP 连接实现客户端与服务器之间的全双工通信。该协议于 2011 年被 IETF 标准化为 RFC 6455，后由 RFC 7936 补充规范。WebSocket 使得客户端和服务器之间的数据交换更加高效，允许服务器主动向客户端推送数据，适用于对实时性要求较高的应用场景，如在线聊天、实时游戏、股票行情等 。

# 1. WebSocket 的工作原理

WebSocket 的通信过程主要包括以下几个步骤：[apifox+1perfect.org+1](https://apifox.com/apiskills/websocket-principle-explained/?utm_source=chatgpt.com)

1. **握手阶段**：客户端通过发送带有特殊头部的 HTTP 请求（如 `Upgrade: websocket` 和 `Connection: Upgrade`）来请求升级协议。服务器收到请求后，若支持 WebSocket，会返回状态码 101（Switching Protocols），并确认升级。
2. **建立连接**：握手成功后，客户端和服务器之间建立一个持久的 TCP 连接，双方可以在该连接上进行双向通信。
3. **数据传输**：在连接期间，客户端和服务器可以随时互相发送数据。WebSocket 协议定义了数据帧的格式，支持文本和二进制数据。
4. **关闭连接**：任一方可以随时发送关闭帧来终止连接，另一方收到后也应发送关闭帧作为回应。

# 2. WebSocket 的优势

- **实时双向通信**：服务器可以主动向客户端推送数据，无需客户端轮询，提高了实时性。
- **减少网络开销**：相比传统的 HTTP 请求，WebSocket 的数据帧头部较小，减少了传输的冗余数据。
- **保持连接状态**：一旦建立连接，客户端和服务器之间的通信无需重复建立和关闭连接，降低了延迟。
- **支持二进制数据**：除了文本数据，WebSocket 还支持二进制数据的传输，适用于多种应用场景。

# 3. WebSocket 与 HTTP 的区别
| 特性      | HTTP       | WebSocket       |
| ------- | ---------- | --------------- |
| 通信模式    | 请求-响应（单向）  | 全双工（双向）         |
| 连接方式    | 每次请求建立新连接  | 一次握手后保持连接       |
| 服务器推送能力 | 不支持        | 支持              |
| 数据传输效率  | 较低，头部信息冗余  | 高效，头部信息精简       |
| 应用场景    | 静态网页、表单提交等 | 实时聊天、在线游戏、股票行情等 |

# 4. 使用Spring 的 `WebSocketHandler`实现
## 4.1. 配置 WebSocket 映射路径

通过 `WebSocketConfigurer` 注册 Handler 和路由路径。
```
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {
    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry
            .addHandler(new MyHandler(), "/websocket") // 路由绑定
            .setAllowedOrigins("*"); // 允许跨域
    }
}
```

### 作用

- **建立 URI 与后端逻辑的映射**：把浏览器访问的 `ws://localhost:8080/websocket` 路径绑定到后端的 `WebSocketHandler` 实现上。
- **相当于 DispatcherServlet 的 URL 映射功能**，但用于 WebSocket 协议（`ws://`），而非传统 HTTP。

### 为什么需要

- 客户端连接 WebSocket 时必须提供连接地址，这一步就是告诉 Spring Boot：`/websocket` 请求该由谁处理（即 MyHandler）。

## 4.2. 实现 WebSocketHandler 逻辑

通过继承 `TextWebSocketHandler` 实现消息处理逻辑。
```
public class MyHandler extends TextWebSocketHandler {

    // 连接建立时触发
    @Override
    public void afterConnectionEstablished(WebSocketSession session) throws Exception {
        System.out.println("客户端连接：" + session.getId());
    }

    // 收到客户端消息时触发
    @Override
    public void handleTextMessage(WebSocketSession session, TextMessage message) throws Exception {
        System.out.println("收到消息：" + message.getPayload());

        // 回复消息给客户端
        session.sendMessage(new TextMessage("后端已收到：" + message.getPayload()));
    }

    // 连接关闭时触发
    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) throws Exception {
        System.out.println("连接关闭：" + session.getId());
    }

    // 异常处理
    @Override
    public void handleTransportError(WebSocketSession session, Throwable exception) throws Exception {
        exception.printStackTrace();
    }
}
```

### 作用

- **处理实际的 WebSocket 消息交互**：
    
    - `onOpen`：连接建立时调用
    - `handleTextMessage`：客户端发消息时调用
    - `onClose`：连接断开时调用
    - `onError`：发生异常时调用

### 为什么需要

- WebSocket 是双向通信协议，客户端可以随时发消息，后端必须定义如何处理这些“事件”。这一层就是事件处理器。

## 4.3. 会话管理（可选但常用）

如果你要广播消息或针对特定用户推送消息，就需要维护一个 `session` 映射：
```
private static final Map<String, WebSocketSession> sessions = new ConcurrentHashMap<>();

@Override
public void afterConnectionEstablished(WebSocketSession session) throws Exception {
    sessions.put(session.getId(), session);
}

@Override
public void afterConnectionClosed(WebSocketSession session, CloseStatus status) throws Exception {
    sessions.remove(session.getId());
}
```
广播所有连接：
```
public void broadcast(String msg) {
    sessions.values().forEach(session -> {
        try {
            session.sendMessage(new TextMessage(msg));
        } catch (IOException e) {
            e.printStackTrace();
        }
    });
}
```

### 作用：

- **保存所有客户端的连接对象**（`WebSocketSession`）
    
    - 这样你可以广播消息给所有用户
    - 或者点对点推送给某个特定的用户

### 为什么需要

- 默认情况下，WebSocket 是“被动”的。要实现“主动推送”消息，必须保留会话引用。
- 没有这一步，你就不能在其他地方调用 `session.sendMessage()`。

## 4.4. 发送消息的接口（可选）

可以提供 REST 接口调用后端主动推送消息：
```
@RestController
@RequestMapping("/api/message")
public class MessageController {
    @PostMapping("/broadcast")
    public String broadcast(@RequestParam String msg) {
        MyHandler.broadcast(msg); // 调用 handler 中的广播逻辑
        return "OK";
    }
}
```

### 作用

- 提供一个传统的 HTTP 接口，可以在系统中通过 HTTP 请求向 WebSocket 客户端发送消息。
- 实现“从后端主动触发”消息，而不是依赖前端发起请求。

### 为什么需要

- 这是将 WebSocket 融入系统业务逻辑的关键桥梁。
- 例如，你监听数据库变更或告警事件时，就可以通过调用这个接口通知前端。

# 5. 实际流程

访问一个前端页面时（比如某个 HTML 页面或 Vue/React 页面），确实**可能触发多个 HTTP 请求或控制器逻辑**，但**WebSocket 的建立不是由某个 controller 自动发起的**，而是由 **前端 JavaScript 主动调用 `new WebSocket(url)` 发起的连接请求**。

## 5.1. 用户访问一个页面

- 浏览器向后端请求 `GET /chat/index`（这是一个普通的 HTTP 请求），后端可能进入某个 `@Controller` 渲染页面或返回前端页面框架。
- 举个例子
```
@GetMapping("/chat/index")
public String chatPage() {
    return "chat.html"; // 或者前端 Vue 项目
}
```

## 5.2. 页面中的 JS 脚本发起 WebSocket 连接

页面加载完成后，JS 中写了如下代码
```
const socket = new WebSocket("ws://localhost:8080/websocket");
```
这一句才是触发 WebSocket 建立连接的**关键行为**，它相当于向后端发送了一个升级请求（Upgrade: websocket）。
这时就会匹配到 `WebSocketConfig` 中注册的 `/websocket` 路径，并交由你写的 `MyHandler` 处理。

## 5.3. 后端的 WebSocketHandler 接收连接并建立会话

由 Spring 的 `WebSocketHandler` 触发连接事件（`afterConnectionEstablished`）；
然后才开始消息收发。

# 6. 原生 WebSocket 与 STOMP 的区别

## 6.1. 详细对比
|功能点|不用 STOMP（原生 `WebSocketHandler`）|用 STOMP（Spring + `@MessageMapping` + `SimpMessagingTemplate`）|
|---|---|---|
|**通信协议**|原始 WebSocket|WebSocket + STOMP（类 HTTP 协议）|
|**连接入口**|`/websocket`（你注册的路径）|`/ws`（由 STOMP config 注册）|
|**消息结构**|任意文本或二进制，格式自定义|STOMP 帧格式（有命令头、destination 等字段）|
|**路由机制**|手动实现判断消息类型、路由|自动映射 `/app/xxx → @MessageMapping("/xxx")`|
|**广播**|手动遍历所有 session 调用 `sendMessage()`|`@SendTo("/topic/xxx")` 或 `convertAndSend()` 一行代码|
|**点对点发送**|需自己维护用户-会话映射表|支持 `convertAndSendToUser()`（自动映射用户）|
|**订阅机制**|你要自己写“谁订阅了什么”的逻辑|客户端调用 `subscribe('/topic/xxx')`，由框架自动管理|
|**会话管理**|你维护 `Map<String, WebSocketSession>`|Spring 框架管理，无需手动|
|**异常处理、断线重连**|需手动管理|SockJS + STOMP 支持自动重连、错误处理|
|**前端协议兼容性**|仅浏览器支持 WebSocket 的才能用|SockJS 支持回退为 AJAX 等|
|**调试与可读性**|比较底层、调试困难|STOMP 命令语义清晰（CONNECT, SUBSCRIBE, SEND）|
## 6.2. 使用体验层面对比

### 不使用 STOMP（原始 WebSocket）：

- 优点：

    - 更轻量，没有额外协议负担。
    - 自由度高，适合简单场景（如聊天、心跳、游戏控制等）。

- 缺点：
    
    - 没有消息语义，自己处理订阅、频道分发。
    - 不支持用户维度、组播、权限拦截等功能。
    - 增加复杂度，容易出 bug。

### 使用 STOMP：

- 优点：
    
    - 协议层支持【订阅/广播/用户消息】机制。
    - 后端逻辑清晰（类似 HTTP controller）。
    - 一套消息系统架构，易于扩展 RabbitMQ / Kafka 等消息代理。
    - 前后端一致性更强。

- 缺点：
    
    - 增加依赖和学习成本（STOMP + SockJS）。
    - 对于极轻量用途略显“重”。