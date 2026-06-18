# 客户端主程序详解 (bin/)

> 本文档详解两个二进制程序：client（主客户端）和 monitor（唤醒词监控）。

## 目录

- [1. client.rs — 主客户端](#1-clientrs--主客户端)
  - [AppClient 结构体](#appclient-结构体)
  - [启动流程详解](#启动流程详解)
  - [初始化详解 (init)](#初始化详解-init)
  - [资源清理详解 (dispose)](#资源清理详解-dispose)
  - [RPC 命令处理器](#rpc-命令处理器)
  - [事件和流处理器](#事件和流处理器)
- [2. monitor.rs — 唤醒词监控程序](#2-monitorrs--唤醒词监控程序)
  - [程序入口](#程序入口)
  - [事件处理函数](#事件处理函数)

---

## 1. client.rs — 主客户端

**文件：** `bin/client.rs`

### AppClient 结构体

```rust
pub struct AppClient {
    kws_monitor: KwsMonitor,              // 唤醒词监控器
    instruction_monitor: InstructionMonitor, // 指令日志监控器
    playing_monitor: PlayingMonitor,       // 播放状态监控器
}
```

**职责：** 整个客户端的顶层协调者，管理所有组件的生命周期。

### 启动流程详解

#### `main()` — 程序入口

```rust
#[tokio::main]  // tokio 异步运行时入口
async fn main() {
    let mut client = AppClient::new();
    client.run().await;
}
```

`#[tokio::main]` 宏会：
1. 创建 tokio 多线程异步运行时
2. 将 `main` 函数包装为异步执行

#### `run()` — 主循环

```
执行逻辑：
  1. 读取命令行第一个参数作为服务器 URL
     url = std::env::args().nth(1)
     如果没有参数 → 打印用法提示并退出

  2. 连接循环：
     loop {
       match connect(url).await {
         Ok(ws_stream) => {
           // 连接成功
           init(ws_stream).await           // 初始化所有组件
           MessageManager::process_messages().await  // 进入消息循环（阻塞）
           dispose().await                  // 连接断开，清理资源
         }
         Err(e) => {
           // 连接失败
           println!("连接失败: {}", e)
           tokio::time::sleep(1s).await     // 等待 1 秒后重试
         }
       }
     }
```

**关键设计：**
- 连接失败会自动重试（1 秒间隔）
- 连接断开后会重新进入连接循环
- 程序永远不会退出（除非被 kill）

#### `connect(url)` — 建立 WebSocket 连接

```
参数：url — WebSocket 服务器地址，如 "ws://192.168.1.100:8080"
执行逻辑：
  1. tokio_tungstenite::connect_async(url).await
  2. 包装为 WsStream::Client(ws_stream)
  3. 返回 WsStream
```

---

### 初始化详解 (init)

```
async fn init(&mut self, ws_stream: WsStream) {

  // ===== 1. 初始化消息管理器 =====
  MessageManager::instance().init(ws_stream).await
  // 拆分 WS 为读写端，初始化 RPC 的发送函数

  // ===== 2. 设置 Event 处理器 =====
  MessageHandler::<Event>::instance().set_handler(on_event).await
  // on_event: 收到服务器事件时的处理函数

  // ===== 3. 设置 Stream 处理器 =====
  MessageHandler::<Stream>::instance().set_handler(on_stream).await
  // on_stream: 收到服务器数据流时的处理函数

  // ===== 4. 注册 RPC 命令 =====
  RPC::instance().add_command("get_version", get_version).await
  RPC::instance().add_command("run_shell", run_shell).await
  RPC::instance().add_command("start_play", start_play).await
  RPC::instance().add_command("stop_play", stop_play).await
  RPC::instance().add_command("start_recording", start_recording).await
  RPC::instance().add_command("stop_recording", stop_recording).await

  // ===== 5. 启动唤醒词监控 =====
  self.kws_monitor.start(|event| async {
    MessageManager::instance().send_event("kws", Some(json!(event))).await
  }).await

  // ===== 6. 启动指令日志监控 =====
  self.instruction_monitor.start(|event| async {
    MessageManager::instance().send_event("instruction", Some(json!(event))).await
  }).await

  // ===== 7. 启动播放状态监控 =====
  self.playing_monitor.start(|event| async {
    MessageManager::instance().send_event("playing", Some(json!(event))).await
  }).await
}
```

**三个监控器的回调模式相同：** 将本地监控到的事件通过 `send_event` 上报给服务器。

---

### 资源清理详解 (dispose)

```
async fn dispose(&mut self) {
  MessageManager::instance().dispose().await
  // 清空 WS 读写端，清理 RPC 状态，终止所有 MessageManager 任务

  AudioPlayer::instance().stop().await.ok()
  // 停止 aplay 进程，清理 channel

  AudioRecorder::instance().stop_recording().await.ok()
  // 停止 arecord 进程

  self.kws_monitor.stop().await
  self.instruction_monitor.stop().await
  self.playing_monitor.stop().await
  // 停止三个监控器的任务
}
```

`.ok()` 的作用：忽略 `Result` 中的错误（dispose 时的错误不重要）。

---

### RPC 命令处理器

以下 6 个函数是服务器可以远程调用的命令：

#### `get_version(_: Request)` → Response

```
功能：返回客户端版本号
参数：忽略
返回：Response::from_data(json!({"version": "1.0.0"}))
```

#### `start_play(request: Request)` → Response

```
功能：开始音频播放
参数：request.payload 可选 AudioConfig JSON
执行逻辑：
  1. 解析 payload 为 Option<AudioConfig>
     - 有 payload → serde_json::from_value(config)
     - 无 payload → None（使用默认配置）
  2. AudioPlayer::instance().start(config).await
  3. 返回 Response::success()
```

#### `stop_play(_: Request)` → Response

```
功能：停止音频播放
参数：忽略
执行逻辑：
  1. AudioPlayer::instance().stop().await
  2. 返回 Response::success()
```

#### `start_recording(request: Request)` → Response

```
功能：开始音频录制
参数：request.payload 可选 AudioConfig JSON
执行逻辑：
  1. 解析 payload 为 Option<AudioConfig>
  2. AudioRecorder::instance().start_recording(
       on_stream: |bytes| {
         MessageManager::instance().send_stream("record", bytes, None).await
         // 录制到的音频数据通过 WS 发送给服务器
       },
       config
     ).await
  3. 返回 Response::success()
```

**关键：** `on_stream` 闭包是录制数据的输出通道，每次录制到一个 buffer 的数据就通过 WS 发送。

#### `stop_recording(_: Request)` → Response

```
功能：停止音频录制
参数：忽略
执行逻辑：
  1. AudioRecorder::instance().stop_recording().await
  2. 返回 Response::success()
```

#### `run_shell(request: Request)` → Response

```
功能：在客户端本地执行 shell 命令
参数：request.payload 为 shell 命令字符串
执行逻辑：
  1. 解析 payload 为 String（脚本内容）
  2. utils::shell::run_shell(&script).await
     // 注意：这是本地执行，不是远程执行
  3. 返回 Response::from_data(json!(CommandResult))
```

**安全注意：** 这个命令允许服务器在客户端设备上执行任意 shell 命令，权限非常大。

---

### 事件和流处理器

#### `on_event(event: Event)` → Result

```
功能：处理从服务器收到的事件
执行逻辑：println!("收到事件: {:?}", event)
当前状态：仅打印日志，不做进一步处理
```

#### `on_stream(stream: Stream)` → Result

```
功能：处理从服务器收到的数据流
执行逻辑：
  if stream.tag == "play" {
    AudioPlayer::instance().play(stream.bytes).await
    // 将音频数据送入播放器
  }
  // 其他 tag 忽略
```

**数据流：** 服务器 → WS Binary → Stream{tag:"play", bytes} → AudioPlayer → aplay → 扬声器

---

## 2. monitor.rs — 唤醒词监控程序

**文件：** `bin/monitor.rs`

### 常量

```rust
static KWS_REPLY_FILE_PATH: &str = "/data/open-xiaoai/kws/reply.txt";
// 自定义唤醒回复文本文件路径
```

### 程序入口

```
#[tokio::main]
async fn main() {
  // 1. 如果 KWS 文件存在，播放欢迎音
  if Path::new(KWS_FILE_PATH).exists() {
    on_started().await
  }

  // 2. 创建并启动 KwsMonitor
  let mut kws_monitor = KwsMonitor::new();
  kws_monitor.start(|event| async {
    match event {
      KwsMonitorEvent::Started => on_started().await,
      KwsMonitorEvent::Keyword(keyword) => on_keyword(keyword).await,
    }
  }).await;

  // 3. 永久运行
  loop {
    tokio::time::sleep(Duration::from_secs(1)).await;
  }
}
```

**注意：** 这是一个独立程序，不连接 WebSocket 服务器，只在本地处理唤醒词事件。

---

### 事件处理函数

#### `on_started()` — 引擎启动处理

```
执行逻辑：
  1. 读取命令行参数作为欢迎消息（默认："自定义唤醒词已开启"）
  2. tokio::process::Command::new("/usr/sbin/tts_play.sh")
       .arg(message)
       .spawn()
  // 播放 TTS 欢迎语音
```

#### `on_keyword(keyword)` — 唤醒词检测处理

```
参数：keyword — 检测到的唤醒词文本
执行逻辑：
  1. 打印检测到的唤醒词

  2. 加载唤醒回复列表：
     if KWS_REPLY_FILE_PATH 存在 {
       读取文件，按行分割为回复列表
     } else {
       使用默认列表：
       [
         "/usr/share/mico/res/hello_xiaomi.wav",
         "/usr/share/mico/res/hello_xiaoai.wav"
       ]
     }

  3. 随机选择一个回复：
     reply = pick_one(&replies)

  4. 播放回复：
     if reply 以 "http://" 或 "https://" 开头 {
       // 当作音频 URL 播放
       Command::new("miplayer").arg(reply).spawn()
     } else {
       // 当作文本 TTS 播放
       Command::new("/usr/sbin/tts_play.sh").arg(reply).spawn()
     }

  5. 触发唤醒事件：
     Command::new("ubus")
       .args(["call", "pnshelper", "event_notify", '{"event":0}'])
       .spawn()
     // 通知系统有唤醒事件发生
```

**唤醒回复文件格式 (`/data/open-xiaoai/kws/reply.txt`)：**
- 每行一个回复
- 以 `http://` 或 `https://` 开头 → 当作音频 URL
- 其他 → 当作 TTS 文本

**示例文件内容：**
```
我在呢
你好呀
https://example.com/hello.wav
有什么可以帮你的
```

---

## 两个程序的关系

```
┌─────────────────────────────────────────────────────────┐
│                    client (主客户端)                       │
│                                                          │
│  功能：                                                   │
│  - 连接 WebSocket 服务器                                  │
│  - 处理服务器的 RPC 请求（播放/录制/执行命令）               │
│  - 上报设备状态事件（唤醒词/指令/播放状态）                  │
│  - 转发音频数据流                                          │
│                                                          │
│  监控器：KwsMonitor + InstructionMonitor + PlayingMonitor │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                 monitor (唤醒词监控)                       │
│                                                          │
│  功能：                                                   │
│  - 独立运行，不连接服务器                                   │
│  - 监控唤醒词日志文件                                      │
│  - 检测到唤醒词时播放回复音效                               │
│  - 触发 ubus 唤醒事件通知系统                               │
│                                                          │
│  监控器：仅 KwsMonitor                                    │
└─────────────────────────────────────────────────────────┘
```

**可以同时运行：** client 负责与服务器通信，monitor 负责本地唤醒词响应。
