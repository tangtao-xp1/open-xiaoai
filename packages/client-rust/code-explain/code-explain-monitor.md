# 监控模块详解 (services/monitor/)

> 本文档详解四个监控器：文件监控、指令监控、唤醒词监控、播放状态监控。

## 目录

- [模块结构](#模块结构)
- [1. file.rs — 通用文件监控器](#1-filers--通用文件监控器)
- [2. instruction.rs — 指令日志监控](#2-instructionrs--指令日志监控)
- [3. kws.rs — 唤醒词监控](#3-kwsrs--唤醒词监控)
- [4. playing.rs — 播放状态监控](#4-playingrs--播放状态监控)
- [监控器工作模式对比](#监控器工作模式对比)

---

## 模块结构

```
monitor/
├── file.rs           // 通用文件 tail 监控（其他监控器的基础）
├── instruction.rs    // 监控小爱指令日志文件
├── kws.rs            // 监控唤醒词检测日志
└── playing.rs        // 轮询播放状态（mphelper mute_stat）
```

---

## 1. file.rs — 通用文件监控器

**文件：** `services/monitor/file.rs`

### 枚举：FileMonitorEvent

```rust
pub enum FileMonitorEvent {
    NewFile,              // 文件被截断/重建（从头开始读）
    NewLine(String),      // 文件新增了一行内容
}
```

### 结构体：FileMonitor

```rust
pub struct FileMonitor {
    task_holder: Option<JoinHandle<()>>,  // 监控任务的句柄
}
```

**这是一个通用的 "tail -f" 实现，** 其他监控器（Instruction、Kws）都基于它构建。

### 核心方法

#### `start(file_path, on_update)` — 启动监控

```
参数：
  - file_path: 要监控的文件路径
  - on_update: 回调函数，收到 FileMonitorEvent 时调用
执行逻辑：
  1. 如果已有任务在运行 → abort() 终止
  2. tokio::spawn 启动 start_monitor 任务
```

#### `stop()` — 停止监控

```
执行逻辑：abort(task_holder) 终止监控任务
```

#### `start_monitor(file_path, on_update)` — 监控主循环

```
执行逻辑：
  // === 初始化阶段 ===
  1. loop {
       if 文件存在 → break
       sleep(10ms)  // 等待文件创建
     }
  2. 打开文件
  3. seek 到文件末尾（position = 文件当前大小）
     // 意味着：启动时只监控新增内容，不处理历史数据

  // === 监控循环 ===
  4. loop {
       // 截断检测
       current_size = 文件当前大小
       if current_size < position {
         // 文件被截断了（比如日志轮转）
         position = 0
         seek(0)
         on_update(FileMonitorEvent::NewFile)
       }

       // 读取新增内容
       while let Some(line) = read_line() {
         if !line.is_empty() {
           on_update(FileMonitorEvent::NewLine(line))
         }
       }
       position = 当前文件位置

       sleep(10ms)  // 轮询间隔
     }
```

**关键设计：**
- 启动时 seek 到文件末尾，只监控新增内容
- 检测文件截断（日志轮转场景），自动重置到文件开头
- 10ms 轮询间隔，平衡实时性和 CPU 消耗

---

## 2. instruction.rs — 指令日志监控

**文件：** `services/monitor/instruction.rs`

### 常量

```rust
static INSTRUCTION_FILE_PATH: &str = "/tmp/mico_aivs_lab/instruction.log";
// 小爱 AI 服务的指令日志文件
```

### 结构体：InstructionMonitor

```rust
pub struct InstructionMonitor {
    file_monitor: FileMonitor,  // 内部使用通用文件监控器
}
```

### 核心方法

#### `start(on_update)` — 启动监控

```
执行逻辑：file_monitor.start(INSTRUCTION_FILE_PATH, on_update)
```

#### `stop()` — 停止监控

```
执行逻辑：file_monitor.stop()
```

### 数据模型

日志文件每行是一个 JSON 对象，反序列化为 `LogMessage`：

```rust
pub struct LogMessage {
    pub header: Header,      // 消息头
    pub payload: Payload,    // 消息体（多种类型）
}

pub struct Header {
    pub dialog_id: String,   // 对话 ID
    pub id: String,          // 消息 ID
    pub name: String,        // 消息名称
    pub namespace: String,   // 命名空间
}
```

### Payload 变体

```rust
pub enum Payload {
    // ASR 语音识别结果
    RecognizeResultPayload {
        is_final: bool,           // 是否最终结果
        is_vad_begin: bool,       // VAD 检测开始
        results: Vec<RecognizeResult>,  // 识别结果列表
    }

    // 停止采集
    StopCapturePayload {
        stop_time: u64,
    }

    // TTS 语音合成响应
    SpeakPayload {
        text: String,             // 要朗读的文本
        emotion: Option<Emotion>, // 情感信息
    }

    // 播放指令
    PlayPayload {
        audio_items: Vec<AudioItem>,  // 音频项列表
        audio_type: String,
        loadmore_token: String,
        needs_loadmore: bool,
        origin_id: String,
        play_behavior: String,
    }

    // 属性设置
    SetPropertyPayload {
        name: String,
        value: String,
    }

    // 指令控制
    InstructionControlPayload {
        behavior: String,
    }

    // 空消息
    EmptyPayload {},
}
```

### RecognizeResult（ASR 结果）

```rust
pub struct RecognizeResult {
    pub confidence: f64,                // 置信度
    pub text: String,                   // 识别文本
    pub asr_binary_offset: Option<u64>, // ASR 二进制偏移
    pub begin_offset: Option<u64>,      // 开始时间偏移
    pub end_offset: Option<u64>,        // 结束时间偏移
    pub is_nlp_request: Option<bool>,   // 是否 NLP 请求
    pub is_stop: Option<bool>,          // 是否停止
    pub origin_text: Option<String>,    // 原始文本
}
```

### 播放指令相关结构

```rust
pub struct AudioItem {
    pub item_id: ItemId,
    pub log: Log,
    pub stream: Stream,  // 注意：这不是 connect::data::Stream
}

pub struct ItemId {
    pub audio_id: String,
    pub cp: Cp,           // 内容提供商
}

pub struct Cp {
    pub id: String,
    pub name: String,
}

pub struct Stream {
    pub authentication: bool,    // 是否需要认证
    pub duration_in_ms: u64,     // 时长（毫秒）
    pub offset_in_ms: u64,       // 偏移（毫秒）
    pub url: String,             // 音频 URL
}
```

---

## 3. kws.rs — 唤醒词监控

**文件：** `services/monitor/kws.rs`

### 常量

```rust
pub static KWS_FILE_PATH: &str = "/tmp/open-xiaoai/kws.log";
// 唤醒词检测日志文件路径
```

### 枚举：KwsMonitorEvent

```rust
pub enum KwsMonitorEvent {
    Started,              // KWS 引擎已启动
    Keyword(String),      // 检测到唤醒词
}
```

### 结构体：KwsMonitor

```rust
pub struct KwsMonitor {
    file_monitor: FileMonitor,  // 内部使用通用文件监控器
}
```

### 核心方法

#### `start(on_update)` — 启动监控

```
执行逻辑：
  1. file_monitor.start(KWS_FILE_PATH, callback)
  2. callback 中的处理逻辑：
     match event {
       FileMonitorEvent::NewFile → {
         // 文件重建，重置去重计数器
       }
       FileMonitorEvent::NewLine(line) → {
         // 解析格式：timestamp@keyword
         parts = line.split("@")
         timestamp = parts[0] (u64)
         keyword = parts[1]

         // 去重：通过 AtomicU64 记录上次处理的时间戳
         if timestamp <= last_timestamp → 跳过（已处理过）

         if keyword == "__STARTED__" {
           on_update(KwsMonitorEvent::Started)
         } else {
           on_update(KwsMonitorEvent::Keyword(keyword))
         }
       }
     }
```

**日志格式：** `{timestamp}@{keyword}`
- `1234567890@__STARTED__` — 引擎启动
- `1234567891@小爱同学` — 检测到唤醒词"小爱同学"

**去重机制：** 使用 `AtomicU64` 记录最后处理的时间戳，防止同一事件被重复处理。

---

## 4. playing.rs — 播放状态监控

**文件：** `services/monitor/playing.rs`

### 枚举：PlayingMonitorEvent

```rust
pub enum PlayingMonitorEvent {
    Playing,  // 正在播放（mute_stat 包含 "1"）
    Paused,   // 已暂停（mute_stat 包含 "2"）
    Idle,     // 空闲（默认状态）
}
```

### 结构体：PlayingMonitor

```rust
pub struct PlayingMonitor {
    task_holder: Option<JoinHandle<()>>,  // 轮询任务句柄
}
```

**与其他监控器不同：** 不基于 FileMonitor，而是直接轮询 shell 命令。

### 核心方法

#### `start(on_update)` — 启动监控

```
执行逻辑：
  1. 如果已有任务 → abort()
  2. tokio::spawn 启动 start_monitor 任务
```

#### `start_monitor(on_update)` — 监控主循环

```
执行逻辑：
  last_status = None  // 上次状态

  loop {
    // 执行 shell 命令获取播放状态
    result = run_shell("mphelper mute_stat")

    // 解析状态
    status = match result.stdout {
      包含 "1" → PlayingMonitorEvent::Playing,
      包含 "2" → PlayingMonitorEvent::Paused,
      _        → PlayingMonitorEvent::Idle,
    }

    // 只在状态变化时触发回调
    if Some(&status) != last_status.as_ref() {
      on_update(status.clone()).await
      last_status = Some(status)
    }

    sleep(10ms)  // 轮询间隔
  }
```

**关键设计：** 只在状态**变化**时触发回调，避免重复通知。

---

## 监控器工作模式对比

| 监控器 | 监控方式 | 数据源 | 事件类型 |
|--------|----------|--------|----------|
| FileMonitor | 文件 tail（轮询 10ms） | 任意文件 | NewFile / NewLine |
| InstructionMonitor | FileMonitor | `/tmp/mico_aivs_lab/instruction.log` | LogMessage（ASR/播放/TTS 指令） |
| KwsMonitor | FileMonitor + 去重 | `/tmp/open-xiaoai/kws.log` | Started / Keyword(word) |
| PlayingMonitor | Shell 轮询（10ms） | `mphelper mute_stat` | Playing / Paused / Idle |

### 监控器在 client 中的使用

```
AppClient::init() {
  // 启动三个监控器，每个都有回调函数
  kws_monitor.start(callback)           // → MessageManager::send_event("kws", ...)
  instruction_monitor.start(callback)   // → MessageManager::send_event("instruction", ...)
  playing_monitor.start(callback)       // → MessageManager::send_event("playing", ...)
}

// 回调中的统一模式：
callback(event) {
  MessageManager::instance().send_event("事件名", Some(json!(event)))
  // 将监控到的本地事件通过 WS 上报给服务器
}
```

### 监控器在 monitor 程序中的使用

```
// monitor.rs 是独立程序，不连 WS
KwsMonitor::start(callback) {
  callback(KwsMonitorEvent::Started) → on_started()    // 播放欢迎音
  callback(KwsMonitorEvent::Keyword(k)) → on_keyword(k) // 播放唤醒回复 + 触发 ubus
}
```
