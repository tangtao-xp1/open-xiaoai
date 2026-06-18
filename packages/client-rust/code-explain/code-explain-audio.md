# 音频模块详解 (services/audio/)

> 本文档详解音频播放、录制和配置。

## 目录

- [模块结构](#模块结构)
- [1. config.rs — 音频配置](#1-configrs--音频配置)
- [2. play.rs — 音频播放](#2-playrs--音频播放)
- [3. record.rs — 音频录制](#3-recordrs--音频录制)
- [数据流图](#数据流图)

---

## 模块结构

```
audio/
├── config.rs   // 音频参数配置（采样率、位深、通道数等）
├── play.rs     // 音频播放器：通过 aplay 进程播放 PCM 数据
└── record.rs   // 音频录制器：通过 arecord 进程录制 PCM 数据
```

---

## 1. config.rs — 音频配置

**文件：** `services/audio/config.rs`

### 结构体：AudioConfig

```rust
pub struct AudioConfig {
    pub pcm: String,          // PCM 设备名，如 "hw:0,0"、"noop"
    pub channels: u16,        // 声道数：1=单声道, 2=立体声
    pub bits_per_sample: u16, // 位深：16 或 32
    pub sample_rate: u32,     // 采样率：如 16000 (16kHz)
    pub period_size: u32,     // 周期大小（帧数）
    pub buffer_size: u32,     // 缓冲区大小（帧数）
}
```

**用途：** 描述音频硬件参数，用于配置 `aplay`/`arecord` 的命令行参数。

### 默认配置：AUDIO_CONFIG

```rust
pub static AUDIO_CONFIG: LazyLock<AudioConfig> = LazyLock::new(|| AudioConfig {
    pcm: "noop".to_string(),
    channels: 1,
    bits_per_sample: 16,
    sample_rate: 16000,
    period_size: 160,
    buffer_size: 480,
});
```

- `pcm: "noop"` — 默认使用空设备（实际使用时由请求参数覆盖）
- `16kHz / 16bit / 单声道` — 语音助手的标准音频格式
- `period_size: 160` — 每次读写 160 帧 = 10ms @ 16kHz
- `buffer_size: 480` — 缓冲区 480 帧 = 30ms @ 16kHz

### ALSA 参数映射

AudioConfig 字段如何转换为 `aplay`/`arecord` 的命令行参数：

| AudioConfig 字段 | aplay/arecord 参数 | 示例值 |
|-------------------|-------------------|--------|
| `pcm` | 直接作为设备名 | `hw:0,0` |
| `bits_per_sample` | `-f S{bits}_LE` | `-f S16_LE` |
| `sample_rate` | `-r {rate}` | `-r 16000` |
| `channels` | `-c {channels}` | `-c 1` |
| `period_size` | `--period-size={size}` | `--period-size=160` |
| `buffer_size` | `--buffer-size={size}` | `--buffer-size=480` |

---

## 2. play.rs — 音频播放

**文件：** `services/audio/play.rs`

### 结构体：AudioPlayer

```rust
pub struct AudioPlayer {
    aplay_thread: Arc<Mutex<Option<Child>>>,
    // aplay 子进程句柄

    write_thread: Arc<Mutex<Option<ChildStdin>>>,
    // aplay 的 stdin 管道（用于写入音频数据）

    sender: Arc<Mutex<Option<mpsc::Sender<Vec<u8>>>>>,
    // mpsc 通道的发送端（play() 方法通过此发送音频数据）

    player_task: Arc<Mutex<Option<JoinHandle<()>>>>,
    // tokio 异步任务句柄（从 channel 读取数据写入 aplay stdin）
}
```

**单例：** `static INSTANCE: LazyLock<AudioPlayer>`

### 核心方法

#### `start(config)` — 启动播放器

```
调用时机：收到服务器 "start_play" RPC 请求时
参数：config — 可选的 AudioConfig，为 None 时使用默认配置
执行逻辑：
  1. stop()  // 如果已有播放器在运行，先停止
  2. 构造 aplay 命令行参数：
       aplay -t raw -f S{bits}_LE -r {rate} -c {channels}
             --period-size={period} --buffer-size={buffer} {pcm}
  3. tokio::process::Command 启动 aplay 子进程
     - stdin(Stdio::piped())  — 捕获 stdin 用于写入
     - stdout(Stdio::null())  — 丢弃 stdout
     - stderr(Stdio::null())  — 丢弃 stderr
  4. 创建 mpsc channel，容量 50
     - sender 存入 self.sender（供 play() 使用）
     - receiver 用于 player_task
  5. tokio::spawn 启动 player_task：
     loop {
       receiver.recv().await  // 等待音频数据
       └─ Some(bytes) → 写入 aplay stdin（100ms 超时）
       └─ None → break（sender 被 drop，通道关闭）
     }
  6. 存储 aplay 进程和 player_task 到 Mutex 中
```

#### `play(bytes)` — 播放音频数据

```
调用时机：
  - 收到服务器 "play" 标签的 Stream 消息时（bin/client.rs::on_stream）
参数：bytes — PCM 原始音频数据
执行逻辑：
  1. 从 self.sender 取出 mpsc::Sender
  2. sender.send(bytes).await  // 发送到 channel
  3. player_task 会自动读取并写入 aplay stdin
```

**为什么用 channel 而不是直接写 stdin？**
- `play()` 可能被多个异步任务并发调用
- channel 提供了缓冲（容量 50）和平滑写入
- player_task 以串行方式写入 aplay stdin，避免并发写入问题

#### `stop()` — 停止播放器

```
调用时机：
  - 收到服务器 "stop_play" RPC 请求时
  - start() 开始前清理旧实例
  - dispose() 资源清理时
执行逻辑：
  1. drop(sender)  // 关闭 channel sender，player_task 会因 recv() 返回 None 而退出
  2. abort(player_task)  // 强制终止 player_task
  3. 关闭 aplay stdin（100ms 超时）
     - write_thread.take() → drop(stdin)
     - 如果 100ms 内未完成 → 超时放弃
  4. aplay_thread.take() → kill() 终止 aplay 进程
```

### 播放数据流

```
服务器 WS Binary 消息
  └─ MessageManager::on_bytes()
       └─ MessageHandler::<Stream>::on(stream)
            └─ on_stream(stream)  [bin/client.rs]
                 └─ if stream.tag == "play":
                      AudioPlayer::instance().play(bytes)
                        └─ sender.send(bytes)
                             └─ mpsc channel (容量50, 缓冲)
                                  └─ player_task 读取
                                       └─ write(aplay stdin)  [100ms超时]
                                            └─ aplay 进程播放音频
```

---

## 3. record.rs — 音频录制

**文件：** `services/audio/record.rs`

### 私有枚举：State

```rust
enum State {
    Idle,       // 空闲状态
    Recording,  // 正在录制
}
```

### 常量

```rust
const A113_CAPTURE_BITS_PER_SAMPLE: u16 = 32;
// Amlogic A113 芯片的 PDM 麦克风硬件固定输出 32-bit 采样
```

### 结构体：AudioRecorder

```rust
pub struct AudioRecorder {
    state: Arc<Mutex<State>>,
    // 当前状态（Idle / Recording）

    arecord_thread: Arc<Mutex<Option<Child>>>,
    // arecord 子进程句柄

    read_thread: Arc<Mutex<Option<JoinHandle<()>>>>,
    // tokio 异步任务句柄（从 arecord stdout 读取音频数据）
}
```

**单例：** `static INSTANCE: LazyLock<AudioRecorder>`

### 核心方法

#### `start_recording(on_stream, config)` — 开始录制

```
调用时机：收到服务器 "start_recording" RPC 请求时
参数：
  - on_stream: 回调函数，录制到数据后调用（用于发送给服务器）
  - config: 可选的 AudioConfig
执行逻辑：
  1. 如果 state == Recording → 直接返回 Ok（防止重复启动）
  2. 设置 state = Recording
  3. capture_config_for_recording(config)
     - 如果请求的 bits_per_sample == 16 → 升级为 32（A113 硬件要求）
     - 其他参数保持不变
  4. spawn_arecord(capture_config)
     - 构造 arecord 命令行：
       arecord -t raw -f S{bits}_LE -r {rate} -c {channels}
               --period-size={period} --buffer-size={buffer} {pcm}
     - stdin(Stdio::null())   — 不需要 stdin
     - stdout(Stdio::piped()) — 捕获 stdout 用于读取录音数据
     - stderr(Stdio::null())  — 丢弃 stderr
  5. 计算缓冲参数：
     - bytes_per_frame = channels * (bits_per_sample / 8)
     - period_bytes = period_size * bytes_per_frame   // 每次读取量
     - buffer_bytes = buffer_size * bytes_per_frame    // 每次回调量
  6. tokio::spawn 启动 read_thread：
     loop {
       match timeout(500ms, arecord.stdout.read(buf)).await {
         Ok(0) | Err(_) → stop_recording(); break  // 无数据或超时 → 停止
         Ok(n) → {
           accumulator.extend_from_slice(&buf[..n])
           while accumulator.len() >= buffer_bytes {
             chunk = accumulator[..buffer_bytes]
             chunk = transform_stream_chunk(chunk, requested, capture)
                     // 如果需要 S32→S16 转换，在此执行
             on_stream(chunk).await  // 通过回调发送
             accumulator.drain(..buffer_bytes)
           }
         }
       }
     }
```

#### `stop_recording()` — 停止录制

```
调用时机：
  - 收到服务器 "stop_recording" RPC 请求时
  - 录制过程中 read 超时或返回 0 字节时（自动停止）
执行逻辑：
  1. 如果 state == Idle → 直接返回 Ok
  2. 设置 state = Idle
  3. abort(read_thread)  // 终止读取任务
  4. arecord_thread.take() → kill() 终止 arecord 进程
```

### 辅助函数

#### `capture_config_for_recording(requested)`

```
用途：根据硬件要求调整录制配置
逻辑：
  - 如果 requested.bits_per_sample == 16
    → 返回 bits_per_sample = 32 的配置（A113 硬件要求）
  - 否则 → 原样返回
```

#### `transform_stream_chunk(chunk, requested, capture)`

```
用途：将录制的原始数据转换为请求的格式
逻辑：
  - 如果 requested.bits_per_sample == 16 且 capture.bits_per_sample == 32
    → 调用 convert_a113_s32_to_s16(chunk)
  - 否则 → 原样返回
```

#### `convert_a113_s32_to_s16(chunk)`

```
用途：将 A113 芯片的 S32_LE PCM 转换为 S16_LE
背景：
  - A113 的 PDM 麦克风输出 32-bit 采样
  - 但有效数据只在低 24 位（PDM 数据特性）
  - 需要右移 8 位并截断为 16-bit
处理逻辑（每个采样）：
  1. 读取 4 字节 little-endian i32
  2. value >> 8  （右移 8 位，因为有效数据在低 24 位）
  3. clamp(i16::MIN, i16::MAX)  （防止溢出）
  4. 写入 2 字节 little-endian i16
```

### 录制数据流

```
服务器发送 Request("start_recording")
  └─ start_recording(on_stream, config)
       └─ arecord 进程 (麦克风硬件 → stdout)
            └─ read_thread 读取 stdout
                 ├─ 累积到 buffer_bytes 大小
                 ├─ transform_stream_chunk() [可能 S32→S16]
                 └─ on_stream(chunk) 回调
                      └─ MessageManager::send_stream("record", bytes)
                           └─ WS Binary 帧发送给服务器
```

---

## 数据流图

### 播放数据流

```
                    ┌──────────────────────────────────┐
                    │         WebSocket 服务器           │
                    └───────────────┬──────────────────┘
                                    │ Binary (PCM 音频)
                                    ▼
┌───────────────────────────────────────────────────────────────┐
│                      MessageManager                            │
│  process_messages() → on_bytes() → Stream{tag:"play", bytes}  │
└───────────────────────────────┬───────────────────────────────┘
                                │
                                ▼
┌───────────────────────────────────────────────────────────────┐
│                       AudioPlayer                              │
│                                                                │
│  play(bytes) ──▶ mpsc channel(cap=50) ──▶ player_task          │
│                                               │                │
│                                               ▼                │
│                                        aplay stdin             │
│                                               │                │
│                                               ▼                │
│                                         🔊 扬声器播放           │
└───────────────────────────────────────────────────────────────┘
```

### 录制数据流

```
┌───────────────────────────────────────────────────────────────┐
│                      AudioRecorder                             │
│                                                                │
│  🎤 麦克风 ──▶ arecord 进程 ──▶ stdout                         │
│                                    │                           │
│                                    ▼                           │
│                              read_thread                       │
│                                    │                           │
│                           ┌───────┴───────┐                   │
│                           │ 累积到 buffer  │                   │
│                           │ S32→S16 转换   │                   │
│                           └───────┬───────┘                   │
│                                   │                           │
│                                   ▼                           │
│                             on_stream(chunk) 回调               │
└───────────────────────────────┬───────────────────────────────┘
                                │
                                ▼
┌───────────────────────────────────────────────────────────────┐
│                      MessageManager                            │
│  send_stream("record", bytes) → WS Binary 帧发送               │
└───────────────────────────────┬───────────────────────────────┘
                                │
                                ▼
                    ┌──────────────────────────────────┐
                    │         WebSocket 服务器           │
                    └──────────────────────────────────┘
```
