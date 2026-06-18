# WebSocket 连接层详解 (services/connect/)

> 本文档详解 `connect` 模块，它是整个客户端的通信核心。

## 目录

- [模块结构](#模块结构)
- [1. data.rs — 消息协议定义](#1-datars--消息协议定义)
- [2. message.rs — WebSocket 消息管理](#2-messagers--websocket-消息管理)
- [3. handler.rs — 消息分发处理](#3-handlerrs--消息分发处理)
- [4. rpc.rs — 远程过程调用](#4-rpcrs--远程过程调用)
- [模块协作流程图](#模块协作流程图)

---

## 模块结构

```
connect/
├── data.rs       // 定义消息数据结构（Request/Response/Event/Stream）
├── message.rs    // WebSocket 连接管理，消息收发
├── handler.rs    // 消息类型分发，路由到对应处理器
└── rpc.rs        // RPC 机制：注册命令、发起远程调用、等待响应
```

---

## 1. data.rs — 消息协议定义

**文件：** `services/connect/data.rs`

### 枚举：AppMessage

所有 WebSocket 消息的顶层类型：

```rust
pub enum AppMessage {
    Request(Request),     // RPC 请求
    Response(Response),   // RPC 响应
    Event(Event),         // 单向事件通知
    Stream(Stream),       // 二进制数据流
}
```

JSON 序列化后通过 WebSocket 传输。Text 帧传 Request/Response/Event，Binary 帧传 Stream。

### 结构体：Request

```rust
pub struct Request {
    pub id: String,           // 请求唯一 ID（UUID v4）
    pub command: String,      // 命令名称，如 "start_play"、"run_shell"
    pub payload: Option<Value>, // 参数（serde_json::Value，任意 JSON）
}
```

**用途：** 服务器发给客户端的 RPC 请求，或客户端发给服务器的远程调用。

### 结构体：Response

```rust
pub struct Response {
    pub id: String,           // 对应 Request 的 id（用于配对）
    pub code: Option<i32>,    // 状态码：0=成功，-1=失败
    pub msg: Option<String>,  // 消息描述
    pub data: Option<Value>,  // 返回数据
}
```

**工厂方法：**

| 方法 | 用途 | 示例场景 |
|------|------|----------|
| `Response::success()` | 创建成功响应 | RPC 命令执行成功 |
| `Response::from_data(data)` | 创建带数据的响应 | `get_version` 返回版本号 |
| `Response::from_error(id, e)` | 创建错误响应 | 命令不存在或执行失败 |

### 结构体：Event

```rust
pub struct Event {
    pub id: String,           // 事件唯一 ID（UUID v4）
    pub event: String,        // 事件名称，如 "kws"、"playing"、"instruction"
    pub data: Option<Value>,  // 事件数据
}
```

**用途：** 单向通知，不需要响应。客户端用于上报监控到的设备状态变化。

### 结构体：Stream

```rust
pub struct Stream {
    pub id: String,           // 流唯一 ID（UUID v4）
    pub tag: String,          // 标签，如 "play"（播放流）、"record"（录音流）
    pub bytes: Vec<u8>,       // 原始二进制数据（PCM 音频）
    #[serde(skip_serializing_if = "Option::is_none")]
    pub data: Option<Value>,  // 可选元数据
}
```

**用途：** 传输音频数据。`tag` 字段区分数据用途：
- `"play"` — 服务器下发的待播放音频
- `"record"` — 客户端上传的录音数据

---

## 2. message.rs — WebSocket 消息管理

**文件：** `services/connect/message.rs`

### 枚举：WsStream / WsReader / WsWriter

```rust
pub enum WsStream {
    Server(WebSocketStream<TcpStream>),                    // 服务端连接
    Client(WebSocketStream<MaybeTlsStream<TcpStream>>),   // 客户端连接（可能 TLS）
}
```

使用枚举包装是为了同时支持作为服务端和客户端两种连接模式。`WsReader`、`WsWriter` 是对应的读/写端枚举。

### 结构体：MessageManager

```rust
pub struct MessageManager {
    semaphore: Arc<Semaphore>,           // 并发控制信号量，容量 32
    reader: Arc<Mutex<Option<WsReader>>>, // WS 读取端
    writer: Arc<Mutex<Option<WsWriter>>>, // WS 写入端
}
```

**单例：** `static INSTANCE: LazyLock<MessageManager>`

### 核心方法

#### `init(ws_stream)`

```
调用时机：AppClient::init() 中
参数：WsStream（已建立的 WebSocket 连接）
执行逻辑：
  1. 根据 Server/Client 变体拆分 ws_stream 为 reader + writer
  2. 存储 reader 和 writer 到 Mutex 中
  3. 初始化 RPC 模块，传入发送函数：
     - 发送函数将 Request 序列化为 JSON → 包装为 AppMessage::Request → 文本帧发送
```

#### `dispose()`

```
调用时机：连接断开时
执行逻辑：
  1. 清空 reader 和 writer（Option = None）
  2. RPC::dispose() — 清理所有 RPC 状态
  3. TaskManager::dispose("MessageManager") — 终止所有相关异步任务
```

#### `process_messages()` — 主消息循环

```
调用时机：init 完成后
执行逻辑：
  loop {
    reader.read().await  // 读取下一条 WS 消息
    match 消息类型:
      Text(text)   → on_text(text)    // JSON 文本消息
      Binary(bytes) → on_bytes(bytes) // 二进制消息
      None | Close → break            // 连接关闭，退出循环
  }
```

**这是整个客户端的"心跳"，阻塞在此循环中直到连接断开。**

#### `on_text(text)` — 文本消息分发

```
执行逻辑：
  1. serde_json::from_str::<AppMessage>(text)  // 反序列化
  2. match AppMessage:
       Request(req)   → run_concurrently(|| MessageHandler::on_request(req))
                         // 并发处理请求（受信号量限制，最多 32 个并发）
       Response(resp)  → MessageHandler::on_response(resp)
                         // 直接处理响应（需要快速唤醒等待的 oneshot channel）
       Event(event)    → MessageHandler::on(event)
                         // 直接处理事件
```

#### `on_bytes(bytes)` — 二进制消息分发

```
执行逻辑：
  1. serde_json::from_slice::<Stream>(bytes)  // 反序列化为 Stream
  2. MessageHandler::<Stream>::instance().on(data)
```

#### `send_event(event, data)` — 发送事件

```
执行逻辑：
  1. Event::new(event, data)  // 创建 Event，自动生成 UUID
  2. 序列化为 AppMessage::Event JSON
  3. Message::Text(json) → send()
```

#### `send_stream(tag, bytes, data)` — 发送数据流

```
执行逻辑：
  1. Stream::new(tag, bytes, data)  // 创建 Stream，自动生成 UUID
  2. 序列化为 JSON
  3. Message::Binary(json_bytes) → send()
```

#### `run_concurrently(f)` — 并发任务执行

```
执行逻辑：
  1. semaphore.acquire().await  // 获取许可（最多 32 个并发）
  2. tokio::spawn(async move { f().await })  // 启动异步任务
  3. TaskManager::add("MessageManager", handle)  // 注册任务句柄
```

**设计意图：** 限制同时处理的请求数量，防止服务器并发发送大量请求压垮客户端。

---

## 3. handler.rs — 消息分发处理

**文件：** `services/connect/handler.rs`

### 类型定义

```rust
type Handler<T> = Arc<dyn Fn(T) -> BoxFuture<'static, Result<(), AppError>> + Send + Sync>;
```

这是一个异步回调函数的类型。`T` 是消息类型（Request/Response/Event/Stream）。

### 结构体：MessageHandler\<T\>

```rust
pub struct MessageHandler<T> {
    handler: Arc<Mutex<Option<Handler<T>>>>,  // 存储的回调函数
}
```

**泛型设计：** 为每种消息类型创建独立的单例：
- `MessageHandler<Request>` — 处理 RPC 请求
- `MessageHandler<Response>` — 处理 RPC 响应
- `MessageHandler<Event>` — 处理事件通知
- `MessageHandler<Stream>` — 处理数据流

### 核心方法

#### `set_handler(handler)` — 注册回调

```
调用时机：AppClient::init() 中
用途：设置 Event 和 Stream 的处理回调
```

#### `on(data)` — 触发回调

```
执行逻辑：
  1. 从 Mutex 中取出 handler
  2. 如果 Some(handler) → 调用 handler(data).await
  3. 如果 None → 忽略（没有注册处理器）
```

#### `on_request(request)` — Request 专用处理

```
调用时机：MessageManager::on_text() 中收到 Request 时
执行逻辑：
  1. 打印请求信息（命令名和 payload）
  2. RPC::instance().on_request(request)  // 查找并执行命令处理器
  3. 将返回的 Response 序列化为 AppMessage::Response JSON
  4. MessageManager::instance().send(Message::Text(json))  // 发送响应
```

**注意：** Request 处理是"接收请求 → 执行 → 返回响应"的完整流程。

#### `on_response(response)` — Response 专用处理

```
调用时机：MessageManager::on_text() 中收到 Response 时
执行逻辑：
  RPC::instance().on_response(response)  // 唤醒等待该响应的 oneshot channel
```

---

## 4. rpc.rs — 远程过程调用

**文件：** `services/connect/rpc.rs`

### 类型定义

```rust
// 发送请求的函数签名
type SendRequestFn = Arc<dyn Fn(Request) -> BoxFuture<'static, Result<(), AppError>> + Send + Sync>;

// 请求处理器的函数签名
type RequestHandler = Arc<dyn Fn(Request) -> BoxFuture<'static, Result<Response, AppError>> + Send + Sync>;
```

### 结构体：RPC

```rust
pub struct RPC {
    send_request: Arc<RwLock<Option<SendRequestFn>>>,
    // 发送请求的函数（由 MessageManager::init 注入）

    request_handlers: Arc<RwLock<HashMap<String, RequestHandler>>>,
    // 本地注册的命令处理器：command_name → handler_fn

    pending_requests: Arc<Mutex<HashMap<String, oneshot::Sender<Response>>>>,
    // 等待响应的请求：request_id → oneshot sender
}
```

**单例：** `static INSTANCE: LazyLock<RPC>`

### 核心方法

#### `init(send_request)` — 初始化

```
调用时机：MessageManager::init() 中
执行逻辑：存储发送函数，后续 call_remote 使用此函数发送请求
```

#### `add_command(command, handler)` — 注册命令

```
调用时机：AppClient::init() 中，注册 6 个命令
执行逻辑：将 (command_name, handler_fn) 存入 request_handlers HashMap
```

**已注册的命令：**

| 命令名 | 处理函数 | 功能 |
|--------|----------|------|
| `get_version` | `get_version()` | 返回客户端版本号 |
| `run_shell` | `run_shell()` | 执行 shell 命令 |
| `start_play` | `start_play()` | 开始音频播放 |
| `stop_play` | `stop_play()` | 停止音频播放 |
| `start_recording` | `start_recording()` | 开始音频录制 |
| `stop_recording` | `stop_recording()` | 停止音频录制 |

#### `on_request(request)` — 处理收到的请求

```
调用时机：MessageHandler::on_request() 中
参数：request — 收到的 RPC 请求
执行逻辑：
  1. request_handlers.read() 获取处理器映射
  2. 查找 request.command 对应的 handler
  3. 找到 → handler(request).await → 返回 Response
  4. 未找到 → 返回 Response::from_error("command not found")
```

#### `on_response(response)` — 处理收到的响应

```
调用时机：MessageHandler::on_response() 中
执行逻辑：
  1. pending_requests.lock() 获取等待映射
  2. 查找 response.id 对应的 oneshot::Sender
  3. 找到 → sender.send(response) 唤醒等待的 call_remote
  4. 未找到 → 忽略（可能已超时）
```

#### `call_remote(command, payload, timeout)` — 发起远程调用

```
调用时机：SpeakerManager 的各种方法中
参数：
  - command: 要调用的远程命令名
  - payload: 命令参数（JSON）
  - timeout: 超时毫秒数（默认 10000ms）
执行逻辑：
  1. 生成 UUID 作为 request.id
  2. 构造 Request { id, command, payload }
  3. 调用 send_request(request) 通过 WS 发送
  4. 创建 oneshot channel
  5. pending_requests[id] = sender
  6. tokio::time::timeout(timeout, receiver).await
  7. 超时 → 清理 pending_requests，返回超时错误
  8. 收到响应 → 返回 Response
```

**这是客户端向服务器发起 RPC 调用的核心方法。** 调用方（如 SpeakerManager）会一直阻塞等待，直到服务器返回响应或超时。

#### `dispose()` — 清理

```
执行逻辑：
  1. 清空 send_request（设为 None）
  2. 清空 request_handlers
  3. 清空 pending_requests（所有等待中的 call_remote 会因 channel 关闭而收到错误）
```

---

## 模块协作流程图

### 服务器 → 客户端：RPC 请求流程

```
服务器发送 JSON: {"Request":{"id":"abc","command":"start_play","payload":{...}}}
     │
     ▼
MessageManager::process_messages()
  └─ on_text(text)
       └─ serde_json 反序列化
            └─ AppMessage::Request(Request{id:"abc", command:"start_play", ...})
                 └─ run_concurrently (获取信号量许可)
                      └─ MessageHandler::<Request>::on_request(request)
                           └─ RPC::on_request(request)
                                ├─ 查找 "start_play" handler
                                └─ start_play(request).await
                                     └─ 返回 Response{code:0, msg:"success"}
                                          └─ 序列化 → WS 发送 Response
```

### 客户端 → 服务器：远程调用流程

```
SpeakerManager::run_shell("ls")
     └─ RPC::call_remote("run_shell", Some(json!({"script":"ls"})), None)
          ├─ 生成 Request{id:"uuid-xxx", command:"run_shell", payload:{...}}
          ├─ send_request(request) → WS 发送
          ├─ pending_requests["uuid-xxx"] = oneshot_sender
          └─ oneshot_receiver.await (阻塞等待)
               │
               │  [服务器执行 ls，返回 Response]
               │
               ▼
          MessageManager::on_text()
            └─ MessageHandler::<Response>::on_response(response)
                 └─ RPC::on_response(response)
                      └─ pending_requests["uuid-xxx"].send(response)
                           └─ call_remote 收到 Response → 返回给 SpeakerManager
```
