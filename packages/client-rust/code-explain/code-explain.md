# client-rust 代码详解

> 本文档帮助非 Rust 背景的开发者快速理解 client-rust 的代码逻辑和架构。

## 目录

- [1. 项目概览](#1-项目概览)
- [2. 技术栈与依赖](#2-技术栈与依赖)
- [3. 架构总览](#3-架构总览)
- [4. 启动流程](#4-启动流程)
- [5. 核心消息协议](#5-核心消息协议)
- [6. 整体函数调用链](#6-整体函数调用链)
- [7. 现实场景详解：client 与 monitor 如何协作](#7-现实场景详解client-与-monitor-如何协作)
- [8. 单例模式说明](#8-单例模式说明)
- [9. 模块详细文档索引](#9-模块详细文档索引)
- [10. 关键 Rust 语法速查](#10-关键-rust-语法速查)

---

## 1. 项目概览

这是一个运行在 **小米智能音箱（ARM Linux）** 上的 Rust 客户端程序，通过 WebSocket 与远程服务器通信，实现：

- **音频播放**：接收服务器下发的音频数据，通过 `aplay` 播放
- **音频录制**：通过 `arecord` 录制麦克风音频，发送给服务器
- **设备监控**：监控小爱指令日志、播放状态、自定义唤醒词
- **远程控制**：服务器可通过 RPC 远程执行 shell 命令、控制设备

**两个二进制程序：**
- `client` — 主客户端，连接 WebSocket 服务器，处理所有业务
- `monitor` — 独立的唤醒词监控程序，不连服务器

---

## 2. 技术栈与依赖

| 依赖 | 用途 |
|------|------|
| `tokio` | 异步运行时（多线程），处理并发 IO |
| `tokio-tungstenite` | WebSocket 客户端/服务端库 |
| `serde` / `serde_json` | JSON 序列化/反序列化 |
| `futures` | 异步编程工具（`BoxFuture` 等） |
| `uuid` | 生成请求/事件的唯一 ID |
| `rand` | 随机选择唤醒音效 |

**编译目标：** `armv7-unknown-linux-gnueabihf`（32位 ARM，用于音箱硬件）

---

## 3. 架构总览

```
┌─────────────────────────────────────────────────────────────┐
│                        bin/client.rs                         │
│                      (主程序入口 AppClient)                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ KwsMonitor   │  │InstructionMon│  │PlayingMonitor│      │
│  │ 唤醒词监控    │  │ 指令日志监控  │  │ 播放状态监控  │      │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘      │
│         │                 │                 │               │
│         ▼                 ▼                 ▼               │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              MessageManager (WebSocket)               │    │
│  │         消息收发 · JSON 序列化 · 并发控制               │    │
│  └──────────────────────┬──────────────────────────────┘    │
│                         │                                   │
│         ┌───────────────┼───────────────┐                   │
│         ▼               ▼               ▼                   │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐            │
│  │   RPC      │  │  Handler   │  │   Event    │            │
│  │ 远程调用    │  │ 消息分发    │  │ 事件处理    │            │
│  └─────┬──────┘  └────────────┘  └────────────┘            │
│        │                                                    │
│        ▼                                                    │
│  ┌────────────┐  ┌────────────────┐                        │
│  │  Speaker   │  │ AudioPlayer /  │                        │
│  │  Manager   │  │ AudioRecorder  │                        │
│  │ 远程设备控制│  │ 音频播放/录制   │                        │
│  └────────────┘  └────────────────┘                        │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                       utils (工具层)                          │
│  TaskManager · EventBus · Shell · Rand                      │
└─────────────────────────────────────────────────────────────┘
```

**核心设计模式：全局单例 + 回调驱动**

几乎每个管理器都使用 `LazyLock<T>` 实现全局单例，通过 `::instance()` 访问。
模块间通过回调闭包（callback）传递事件，而非直接调用。

---

## 4. 启动流程

### client 主程序启动流程

```
main()
  └─ AppClient::new()          // 创建三个监控器实例
  └─ AppClient::run()          // 主循环
       ├─ 读取命令行参数获取服务器 URL
       ├─ connect(url)          // WebSocket 连接（失败重试，间隔1秒）
       └─ [连接成功]
            ├─ init(ws_stream)  // 初始化所有组件
            │    ├─ MessageManager::init()     // 拆分 WS 读写端
            │    ├─ 设置 MessageHandler 回调    // Event / Stream 处理
            │    ├─ RPC::add_command() × 6      // 注册 6 个 RPC 命令
            │    └─ 三个 Monitor::start()       // 启动监控，回调发事件
            │
            ├─ MessageManager::process_messages()  // 阻塞式消息循环
            │    [持续读取 WS 消息 → 分发处理]
            │
            └─ [连接断开]
                 └─ dispose()    // 清理所有资源
                      ├─ MessageManager::dispose()
                      ├─ AudioPlayer::stop()
                      ├─ AudioRecorder::stop_recording()
                      └─ 三个 Monitor::stop()
```

### monitor 程序启动流程

```
main()
  ├─ 检查 KWS_FILE_PATH 是否存在
  │    └─ 存在则 on_started()     // 播放欢迎语音
  ├─ KwsMonitor::new()
  ├─ KwsMonitor::start(callback)  // 监控唤醒词日志
  │    └─ callback 中处理:
  │         ├─ KwsMonitorEvent::Started  → 播放欢迎音
  │         └─ KwsMonitorEvent::Keyword  → 随机播放唤醒回复 + 触发 ubus 唤醒事件
  └─ loop { sleep(1s) }           // 永久运行
```

---

## 5. 核心消息协议

所有 WebSocket 消息都通过 `AppMessage` 枚举表示，JSON 格式传输：

```rust
enum AppMessage {
    Request(Request),    // RPC 请求：{id, command, payload}
    Response(Response),  // RPC 响应：{id, code, msg, data}
    Event(Event),        // 单向事件：{id, event, data}
    Stream(Stream),      // 二进制流：{id, tag, bytes, data}
}
```

**消息流向：**

```
服务器 ──Request──▶ 客户端    // 服务器调用客户端功能（如 start_play）
服务器 ◀──Response── 客户端    // 客户端返回执行结果
服务器 ──Event──▶ 客户端       // 服务器推送通知
服务器 ◀──Event── 客户端       // 客户端上报事件（监控到的变化）
服务器 ◀──Stream── 客户端      // 客户端发送音频流（录音数据）
服务器 ──Stream──▶ 客户端      // 服务器下发音频流（播放数据）
```

**Request/Response 配对机制：** 通过 `id` 字段配对。客户端发送 Request 时生成 UUID，创建 oneshot channel 等待对应的 Response 返回。

---

## 6. 整体函数调用链

### 6.1 接收服务器 RPC 请求并播放音频

```
[WS 收到 Text 消息]
MessageManager::process_messages()
  └─ on_text(text)
       └─ 反序列化为 AppMessage::Request
            └─ MessageHandler::on_request(request)
                 └─ RPC::on_request(request)
                      └─ 查找 "start_play" 命令处理器
                           └─ start_play(request)        // bin/client.rs
                                ├─ 反序列化 AudioConfig
                                └─ AudioPlayer::instance().start(config)
                                     ├─ stop() [清理旧进程]
                                     ├─ spawn("aplay", args)
                                     ├─ 创建 mpsc channel(capacity=50)
                                     └─ spawn tokio task: channel → aplay stdin
```

### 6.2 接收服务器音频流并播放

```
[WS 收到 Binary 消息]
MessageManager::process_messages()
  └─ on_bytes(bytes)
       └─ 反序列化为 AppMessage::Stream
            └─ MessageHandler::<Stream>::on(stream)
                 └─ on_stream(stream)                    // bin/client.rs
                      └─ AudioPlayer::instance().play(bytes)
                           └─ sender.send(bytes) → channel → aplay stdin
```

### 6.3 录制音频并发送给服务器

```
服务器发送 Request("start_recording")
  └─ start_recording(request)                            // bin/client.rs
       └─ AudioRecorder::instance().start_recording(on_stream, config)
            ├─ capture_config_for_recording(config)       // 可能升级到 32-bit
            ├─ spawn("arecord", args)
            └─ spawn tokio task:
                 loop {
                   arecord stdout.read(buf)               // 读取 PCM 数据
                   └─ on_stream(bytes) 回调
                        └─ MessageManager::send_stream("record", bytes)
                             └─ 序列化为 Binary 消息通过 WS 发送
                 }
```

### 6.4 监控到播放状态变化并上报

```
PlayingMonitor::start(callback)
  └─ loop {
       run_shell("mphelper mute_stat")
       └─ 解析状态 (Playing/Paused/Idle)
            └─ 状态变化时调用 callback
                 └─ MessageManager::send_event("playing", data)
                      └─ 序列化为 AppMessage::Event → WS 发送
    }
```

### 6.5 远程 shell 命令执行链

```
SpeakerManager::play_text("你好")
  └─ run_shell("/usr/sbin/tts_play.sh 你好")
       └─ RPC::call_remote("run_shell", payload, timeout)
            ├─ 生成 UUID Request
            ├─ send_request(request) → WS 发送
            ├─ 创建 oneshot channel
            ├─ pending_requests[id] = sender
            └─ receiver.await (等待 Response，超时 10s)

服务器收到 Request → 执行 → 返回 Response
  └─ MessageHandler::on_response(response)
       └─ RPC::on_response(response)
            └─ pending_requests[response.id].send(response)
                 └─ sender 发送，receiver 解除等待
```

---

## 7. 现实场景详解：client 与 monitor 如何协作

> 本节通过具体场景举例，说明 `client` 和 `monitor` 两个程序各自做了什么、调用了哪些函数、以及它们如何协作。

### 7.1 client 与 monitor 的关系

**两个独立编译、独立运行的常驻程序**，互不依赖，可以同时运行：

| | `client`（主客户端） | `monitor`（唤醒词监控） |
|---|---|---|
| 是否连服务器 | 是（WebSocket） | 否（纯本地） |
| 包含的监控器 | KwsMonitor + InstructionMonitor + PlayingMonitor | 仅 KwsMonitor |
| 检测到唤醒词后 | 上报服务器（`send_event("kws", ...)`） | 本地播放回复 + 触发 ubus 事件 |
| 服务器下发指令时 | 执行指令（播放/录制/执行命令） | 不参与 |
| 离线时能否工作 | 不能（连不上服务器，持续重试） | 能（纯本地，不依赖网络） |

**不会冲突：** 两个进程各自独立监控同一个日志文件 `/tmp/open-xiaoai/kws.log`，各自处理各自的事件，互不影响。

### 7.2 事件触发机制：谁写了 kws.log？

`/tmp/open-xiaoai/kws.log` 是唤醒词检测的日志文件，由小爱音箱系统底层的 KWS（Keyword Spotting）引擎写入。

日志格式：`{timestamp}@{keyword}`，例如：
```
1718000001@__STARTED__        ← 引擎启动
1718000002@小爱同学            ← 检测到唤醒词"小爱同学"
1718000005@小爱同学            ← 再次检测到唤醒词
```

`client` 和 `monitor` 中的 `KwsMonitor` 都基于 `FileMonitor`（类似 `tail -f`），以 10ms 间隔轮询这个文件，一旦有新行就触发回调。

### 7.3 场景一：用户说"小爱同学"（在线模式）

此时 `client` 和 `monitor` 都在运行，服务器在线。

```
时间线：
  T0  用户说"小爱同学"
  T1  系统 KWS 引擎写入日志：1718000002@小爱同学
  T2  monitor 检测到唤醒词 → 本地播放"我在呢" + 触发 ubus
  T3  client 检测到唤醒词 → 上报服务器
  T4  服务器收到唤醒事件 → 决定开始录音
  T5  服务器发送 RPC: start_recording → client 开始录音
  T6  用户说话，音频流实时传给服务器
  T7  服务器识别完毕 → 发送 RPC: stop_play → 播放回复语音
```

**涉及的函数调用链：**

**monitor 侧（T2）：**
```
FileMonitor::start_monitor()           // 检测到 kws.log 新增一行
  └─ 解析 "1718000002@小爱同学"
  └─ 回调: KwsMonitorEvent::Keyword("小爱同学")
       └─ on_keyword("小爱同学")         // bin/monitor.rs
            ├─ 读取 /data/open-xiaoai/kws/reply.txt（回复文本池）
            ├─ 随机选一行，如 "我在呢"
            ├─ 调用 tts_play.sh 播放 "我在呢"
            └─ 调用 ubus call pnshelper event_notify '{"event":0}'
                 // 通知小爱系统：有唤醒事件发生
```

**client 侧（T3 → T5）：**
```
FileMonitor::start_monitor()           // 检测到 kws.log 新增一行
  └─ 解析 "1718000002@小爱同学"
  └─ 回调: KwsMonitorEvent::Keyword("小爱同学")
       └─ MessageManager::send_event("kws", {"keyword":"小爱同学"})
            └─ 序列化为 AppMessage::Event → 通过 WebSocket 发送给服务器

服务器收到事件后，决定开始录音：
  └─ 发送 Request("start_recording") → WebSocket
       └─ client 收到 → RPC::on_request()
            └─ start_recording(request)       // bin/client.rs
                 └─ AudioRecorder::instance().start_recording(on_stream, config)
                      └─ 启动 arecord 进程 → 录音数据通过 on_stream 回调
                           └─ MessageManager::send_stream("record", bytes)
                                → 音频流实时发送给服务器
```

### 7.4 场景二：服务器下发语音回复

服务器识别完用户语音后，需要播放回复音频。

```
服务器发送 Request("start_play") → client
  └─ start_play(request)                     // bin/client.rs
       └─ AudioPlayer::instance().start(config)
            └─ 启动 aplay 进程，准备接收音频数据

服务器发送 Stream(tag:"play", bytes) → client
  └─ on_stream(stream)                       // bin/client.rs
       └─ AudioPlayer::instance().play(bytes)
            └─ bytes 通过 mpsc channel → aplay stdin → 扬声器播出
```

### 7.5 场景三：纯离线模式（只有 monitor）

服务器不可用，只有 `monitor` 在运行。

```
T0  用户说"小爱同学"
T1  系统写入 kws.log
T2  monitor 检测到 → 本地播放"我在呢" + 触发 ubus
T3  ubus 事件通知小爱系统 → 系统自身的语音流程被唤醒
    // 此时走的是小爱音箱原生的语音处理流程，不是 open-xiaoai 的服务器
```

离线时 `client` 会持续重试连接服务器（每秒一次），但不影响 `monitor` 的工作。

### 7.6 场景四：服务器远程执行命令

服务器通过 client 在音箱上执行 shell 命令。

```
服务器发送 Request("run_shell", payload="ls /data")
  └─ client 收到 → RPC::on_request()
       └─ run_shell(request)                  // bin/client.rs
            └─ utils::shell::run_shell("ls /data")
                 └─ 执行命令，返回结果
                      └─ Response { data: "open-xiaoai\n..." }
                           → 通过 WebSocket 返回给服务器
```

### 7.7 场景五：播放状态监控上报

client 持续监控音箱的播放状态变化，自动上报服务器。

```
PlayingMonitor::start_monitor()              // 10ms 轮询
  └─ run_shell("mphelper mute_stat")
  └─ 解析输出：包含 "1" → Playing，"2" → Paused，其他 → Idle
  └─ 状态变化时触发回调
       └─ MessageManager::send_event("playing", {"status":"Playing"})
            → 通过 WebSocket 上报服务器
```

### 7.8 总结：谁负责什么

```
┌─────────────────────────────────────────────────────────────┐
│                    小爱音箱系统底层                            │
│  KWS 引擎 → 写入 /tmp/open-xiaoai/kws.log                   │
│  指令系统 → 写入 /tmp/mico_aivs_lab/instruction.log          │
│  播放系统 → mphelper mute_stat 查询状态                       │
└──────────┬──────────────────────────────┬────────────────────┘
           │ 监控同一个文件                │ 监控同一个文件
           ▼                              ▼
┌─────────────────────┐    ┌──────────────────────────────────┐
│  monitor（离线响应）  │    │  client（在线响应）                │
│                     │    │                                  │
│  检测唤醒词          │    │  检测唤醒词 → 上报服务器            │
│  → 播放本地回复      │    │  检测指令日志 → 上报服务器          │
│  → 触发 ubus 事件    │    │  监控播放状态 → 上报服务器          │
│                     │    │                                  │
│  不需要服务器        │    │  服务器可远程：播放/录制/执行命令    │
└─────────────────────┘    └──────────────────────────────────┘
```

---

## 8. 单例模式说明

代码中大量使用 Rust 的 `LazyLock` 实现全局单例：

```rust
static INSTANCE: LazyLock<AudioPlayer> = LazyLock::new(AudioPlayer::new);

impl AudioPlayer {
    pub fn instance() -> &'static Self {
        &INSTANCE
    }
}
```

**所有单例列表：**

| 单例 | 文件 | 职责 |
|------|------|------|
| `AudioPlayer` | audio/play.rs | 音频播放管理 |
| `AudioRecorder` | audio/record.rs | 音频录制管理 |
| `MessageManager` | connect/message.rs | WebSocket 消息收发 |
| `RPC` | connect/rpc.rs | RPC 请求/响应管理 |
| `MessageHandler<Request>` | connect/handler.rs | Request 消息处理 |
| `MessageHandler<Response>` | connect/handler.rs | Response 消息处理 |
| `MessageHandler<Event>` | connect/handler.rs | Event 消息处理 |
| `MessageHandler<Stream>` | connect/handler.rs | Stream 消息处理 |
| `EventBus` | utils/event.rs | 事件总线（未使用） |
| `TaskManager` | utils/task.rs | 异步任务管理 |
| `AUDIO_CONFIG` | audio/config.rs | 默认音频配置 |

---

## 9. 模块详细文档索引

每个文档包含对应模块的：结构体定义、函数签名、参数说明、执行逻辑、与其他模块的交互。

| 文档 | 内容 |
|------|------|
| [code-explain-client.md](code-explain-client.md) | 客户端主程序 `bin/client.rs` + `bin/monitor.rs` |
| [code-explain-connect.md](code-explain-connect.md) | WebSocket 连接层：消息协议、RPC、消息分发 |
| [code-explain-audio.md](code-explain-audio.md) | 音频模块：播放、录制、配置 |
| [code-explain-monitor.md](code-explain-monitor.md) | 监控模块：文件监控、指令监控、唤醒词、播放状态 |
| [code-explain-speaker.md](code-explain-speaker.md) | 音箱控制：远程设备操作 |
| [code-explain-utils.md](code-explain-utils.md) | 工具层：Shell、Task、Event、Rand |

---

## 10. 关键 Rust 语法速查

对于非 Rust 开发者，以下是代码中常用的语法解释：

### `async fn` / `.await`
异步函数，类似 JS 的 `async/await`。调用时不会立即执行，需要 `.await` 才会实际运行。

### `Result<T, E>`
Rust 的错误处理类型，类似 Go 的 `(value, error)`。`Ok(T)` 表示成功，`Err(E)` 表示失败。`?` 操作符用于提前返回错误。

### `Arc<Mutex<T>>`
- `Arc` = 原子引用计数智能指针，允许多个线程共享所有权
- `Mutex` = 互斥锁，保证同一时间只有一个线程能访问内部数据
- 组合使用 = 线程安全的共享可变状态

### `Arc<RwLock<T>>`
读写锁，允许多个读者或一个写者。比 `Mutex` 更适合读多写少的场景。

### `LazyLock<T>`
延迟初始化的全局变量，首次访问时创建。用于实现单例模式。

### `Option<T>`
可选值类型，类似 Java/Kotlin 的 `nullable`。`Some(value)` 或 `None`。

### `Box<dyn Trait>`
动态分发的 trait 对象，类似 Java 的接口引用。`AppError = Box<dyn std::error::Error>` 表示任意错误类型。

### `impl Trait`（参数位置）
表示接受任何实现了该 trait 的类型。闭包参数常用此写法。

### `tokio::spawn`
在 tokio 异步运行时中启动一个新任务，类似 Go 的 `go` 关键字。

### `mpsc::channel`
多生产者单消费者通道，类似 Go 的 channel。用于异步任务间传递数据。

### `oneshot::channel`
单次使用的通道，发送一次值后关闭。用于 RPC 请求等待响应。

### `#[derive(...)]`
自动派生宏，让编译器自动生成 `Debug`、`Clone`、`Serialize`、`Deserialize` 等实现。

### `pub(crate)` / `pub`
可见性控制。`pub` = 公开，`pub(crate)` = 仅当前 crate 内可见，无修饰符 = 私有。

### 枚举（enum）中的变体
```rust
enum AppMessage {
    Request(Request),    // 每个变体可以携带不同类型的数据
    Response(Response),
}
```
类似 TypeScript 的联合类型或 Java 的 sealed class。
