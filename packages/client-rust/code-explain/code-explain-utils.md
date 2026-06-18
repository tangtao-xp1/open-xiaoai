# 工具模块详解 (utils/)

> 本文档详解四个工具模块：Shell 执行、任务管理、事件总线、随机选择。

## 目录

- [模块结构](#模块结构)
- [1. shell.rs — Shell 命令执行](#1-shellrs--shell-命令执行)
- [2. task.rs — 异步任务管理](#2-taskrs--异步任务管理)
- [3. event.rs — 事件总线](#3-eventrs--事件总线)
- [4. rand.rs — 随机选择](#4-randrs--随机选择)

---

## 模块结构

```
utils/
├── shell.rs   // 本地 shell 命令执行
├── task.rs    // 异步任务的注册、管理和清理
├── event.rs   // 发布/订阅事件总线（当前未使用）
└── rand.rs    // 随机选择工具
```

---

## 1. shell.rs — Shell 命令执行

**文件：** `utils/shell.rs`

### 结构体：CommandResult

```rust
pub struct CommandResult {
    pub stdout: String,    // 标准输出
    pub stderr: String,    // 标准错误
    pub exit_code: i32,    // 退出码
}
```

### 函数：run_shell(script)

```
参数：script — 要执行的 shell 命令字符串
返回：Result<CommandResult, AppError>
执行逻辑：
  1. tokio::process::Command::new("/bin/sh")
       .arg("-c")
       .arg(script)
       .stdin(Stdio::null())
       .output()
       .await
  2. 解析 stdout、stderr 为 UTF-8 字符串
  3. 提取 exit_code
  4. 返回 CommandResult
```

**用途：** 在客户端本地执行 shell 命令。注意这与 `SpeakerManager::run_shell` 不同：
- `utils::shell::run_shell` — **本地执行**
- `SpeakerManager::run_shell` — 通过 RPC **远程执行**

**使用场景：**
- `PlayingMonitor` 调用 `mphelper mute_stat` 获取播放状态
- `bin/monitor.rs` 调用 `tts_play.sh` 播放语音

---

## 2. task.rs — 异步任务管理

**文件：** `utils/task.rs`

### 结构体：TaskManager

```rust
pub struct TaskManager {
    tasks: Arc<Mutex<HashMap<String, Vec<JoinHandle<()>>>>>,
    // tag → [任务句柄列表]
}
```

**单例：** `static INSTANCE: LazyLock<TaskManager>`

**设计目的：** 集中管理所有 `tokio::spawn` 创建的异步任务，便于统一清理。

### 核心方法

#### `add(tag, handle)` — 注册任务

```
参数：
  - tag: 任务标签（如 "MessageManager"、"EventBus-kws"）
  - handle: tokio 任务句柄
执行逻辑：
  1. 移除同 tag 下已完成的任务（遍历 Vec，retain 未完成的）
  2. 将新 handle 添加到 Vec 中
```

#### `dispose(tag)` — 清理指定标签的所有任务

```
参数：tag — 要清理的任务标签
执行逻辑：
  1. 从 HashMap 中移除该 tag 的所有 handle
  2. 对每个 handle 调用 abort() 强制终止
```

#### `run_async(future)` — 静态辅助方法

```
参数：future — 要执行的异步任务
执行逻辑：
  1. tokio::spawn(future)
  2. add("async", handle)
```

#### `dispose_async()` — 静态辅助方法

```
执行逻辑：dispose("async")  // 清理所有 "async" 标签的任务
```

### 使用场景

| 标签 | 使用位置 | 用途 |
|------|----------|------|
| `"MessageManager"` | message.rs::run_concurrently | 管理并发请求处理任务 |
| `"EventBus-{event}"` | event.rs::publish_async | 管理事件回调任务 |

---

## 3. event.rs — 事件总线

**文件：** `utils/event.rs`

### 类型定义

```rust
type EventCallback = Arc<dyn Fn(Event) -> BoxFuture<'static, Result<(), AppError>> + Send + Sync>;
// 异步回调函数类型
```

### 结构体：EventBus

```rust
pub struct EventBus {
    subscribers: Arc<Mutex<HashMap<String, Vec<EventCallback>>>>,
    // event_name → [回调函数列表]
}
```

**单例：** `static INSTANCE: LazyLock<EventBus>`

**注意：此模块已实现但当前未被使用。** 客户端使用直接回调闭包代替。

### 核心方法

#### `subscribe(event, callback)` — 订阅事件

```
参数：
  - event: 事件名称
  - callback: 回调函数
执行逻辑：将 callback 添加到 subscribers[event] 列表中
```

#### `unsubscribe(event)` — 取消订阅

```
参数：event — 事件名称
执行逻辑：
  1. 移除 subscribers[event] 的所有回调
  2. TaskManager::dispose("EventBus-{event}") 清理相关任务
```

#### `publish(event)` — 同步发布

```
参数：event — Event 对象
执行逻辑：遍历 subscribers[event.event]，依次调用每个回调（同步等待）
```

#### `publish_async(event)` — 异步发布

```
参数：event — Event 对象
执行逻辑：
  遍历 subscribers[event.event]：
    对每个 callback：
      tokio::spawn(async { callback(event).await })
      TaskManager::add("EventBus-{event_name}", handle)
```

---

## 4. rand.rs — 随机选择

**文件：** `utils/rand.rs`

### 线程局部静态

```rust
thread_local! {
    static RNG: RefCell<rand::rngs::ThreadRng> = RefCell::new(rand::rng());
}
```

每个线程独立的随机数生成器。

### 函数：pick_one(items)

```
参数：items — 任意类型的切片 &[T]
返回：&T — 随机选中的元素引用
执行逻辑：
  items.choose(&mut *RNG.borrow_mut()).unwrap()
```

**使用场景：** `bin/monitor.rs` 中随机选择唤醒回复音效。

---

## 工具模块依赖关系

```
base.rs (AppError)
  │
  ├── utils/shell.rs    // run_shell → tokio::process::Command
  │     │
  │     └── services/monitor/playing.rs  // 使用 run_shell 获取播放状态
  │
  ├── utils/task.rs     // TaskManager → 管理 JoinHandle
  │     │
  │     ├── services/connect/message.rs  // 注册并发任务
  │     └── utils/event.rs              // 注册事件回调任务
  │
  ├── utils/event.rs    // EventBus（未使用）
  │     │
  │     ├── services/connect/data.rs    // 使用 Event 类型
  │     └── utils/task.rs              // 依赖 TaskManager
  │
  └── utils/rand.rs     // pick_one → rand::choose
        │
        └── bin/monitor.rs              // 随机选择唤醒音效
```
