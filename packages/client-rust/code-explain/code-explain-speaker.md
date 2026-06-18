# 音箱控制详解 (services/speaker.rs)

> 本文档详解 SpeakerManager，它是远程控制小米智能音箱的核心模块。

## 目录

- [模块概述](#模块概述)
- [远程执行机制](#远程执行机制)
- [方法详解](#方法详解)
  - [设备信息](#设备信息)
  - [播放控制](#播放控制)
  - [麦克风控制](#麦克风控制)
  - [小爱交互](#小爱交互)
- [调用链路图](#调用链路图)

---

## 模块概述

**文件：** `services/speaker.rs`

### 结构体：SpeakerManager

```rust
pub struct SpeakerManager;  // 单元结构体，无字段
```

所有方法都是"关联函数"（类似静态方法），无需实例化。

**核心特点：所有命令都通过 RPC 远程执行，不在本地运行。**

```
客户端程序 ──RPC──▶ 服务器 ──shell──▶ 音箱设备
```

---

## 远程执行机制

### 私有辅助函数：run_shell

```rust
async fn run_shell(script: &str) -> Result<CommandResult, AppError> {
    let response = RPC::instance()
        .call_remote("run_shell", Some(json!({"script": script})), None)
        .await?;
    // 反序列化响应中的 CommandResult
}
```

**调用链：**
```
SpeakerManager::任意方法()
  └─ run_shell("shell 命令")
       └─ RPC::call_remote("run_shell", payload, timeout)
            └─ WS 发送 Request{id, command:"run_shell", payload:{script:"..."}}
                 └─ 等待服务器返回 Response
                      └─ 反序列化为 CommandResult{stdout, stderr, exit_code}
```

---

## 方法详解

### 设备信息

#### `get_boot()` → String

```
shell 命令：fw_env -g boot_part
用途：获取当前启动分区（A/B 分区系统）
返回示例："boot_a" 或 "boot_b"
```

#### `set_boot(boot_part)` → bool

```
shell 命令：fw_env -s boot_part {boot_part}
用途：设置下次启动的分区
```

#### `get_device_model()` → String

```
shell 命令：micocfg_model
用途：获取设备型号
返回示例："LX04"（小爱音箱 Pro）
```

#### `get_device_sn()` → String

```
shell 命令：micocfg_sn
用途：获取设备序列号
```

### 播放控制

#### `get_play_status()` → String

```
shell 命令：mphelper mute_stat
用途：获取播放状态
返回："playing" / "paused" / "idle"
```

#### `play()` → bool

```
shell 命令：mphelper play
用途：恢复播放
```

#### `pause()` → bool

```
shell 命令：mphelper pause
用途：暂停播放
```

#### `play_text(text)` → bool

```
shell 命令：/usr/sbin/tts_play.sh {text}
用途：TTS 语音合成播放（文字转语音）
```

#### `play_url(url)` → bool

```
shell 命令：ubus call mediaplayer player_play_url '{"url":"{url}","type":1}'
用途：播放指定 URL 的音频
```

### 麦克风控制

#### `get_mic_status()` → String

```
shell 命令：cat /tmp/mipns/mute
用途：检查麦克风静音状态
返回："0"（开启）/ "1"（静音）
```

#### `mic_on()` → bool

```
shell 命令：ubus call pnshelper event_notify '{"event":7}'
用途：开启麦克风（取消静音）
```

#### `mic_off()` → bool

```
shell 命令：ubus call pnshelper event_notify '{"event":8}'
用途：关闭麦克风（静音）
```

### 小爱交互

#### `ask_xiaoai(text)` → bool

```
shell 命令：ubus call mibrain ai_service '{"nlp_text":"{text}","nlp":1,"nlp_execute":1,"tts":1,"tts_play":1}'
用途：向小爱发送语音指令（文本形式）
效果：小爱会像听到语音一样处理这个文本指令
```

#### `abort_xiaoai()` → bool

```
shell 命令：/etc/init.d/mico_aivs_lab restart
用途：重启小爱 AI 服务（用于中断当前对话）
```

#### `wake_up(flag)` → bool

```
当 flag = true:
  shell 命令：ubus call pnshelper event_notify '{"event":0}'
  用途：触发唤醒事件（模拟唤醒词）

当 flag = false:
  执行：mic_on() → mic_off()
  用途：通过麦克风开关切换来触发唤醒
```

---

## 调用链路图

### 远程 shell 执行完整链路

```
SpeakerManager::play_text("你好")
  │
  ▼
run_shell("/usr/sbin/tts_play.sh 你好")
  │
  ▼
RPC::call_remote("run_shell", Some({"script":"/usr/sbin/tts_play.sh 你好"}), None)
  │
  ├─ 生成 Request{id:"uuid-xxx", command:"run_shell", payload:{"script":"..."}}
  │
  ├─ send_request(request)
  │    └─ MessageManager::send(Message::Text(json))
  │         └─ WebSocket 发送
  │              └─ ═══════════════════════════════ ══▶ 服务器
  │
  ├─ pending_requests["uuid-xxx"] = oneshot_sender
  │
  └─ oneshot_receiver.await (阻塞，等待服务器响应)
       │
       │  ◀═══════════════════════════════════════ ══ 服务器
       │  Response{id:"uuid-xxx", code:0, msg:"success", data:{stdout:"", stderr:"", exit_code:0}}
       │
       ▼
  MessageManager::on_text()
    └─ MessageHandler::on_response()
         └─ RPC::on_response()
              └─ pending_requests["uuid-xxx"].send(response)
                   └─ oneshot_receiver 收到值
                        └─ call_remote 返回 Response
                             └─ 反序列化为 CommandResult
                                  └─ SpeakerManager::play_text 返回 Ok(true)
```

### ubus 命令说明

音箱设备上的 `ubus` 是 OpenWrt 的微总线系统，用于进程间通信。常用命令：

| ubus 命令 | 用途 |
|-----------|------|
| `ubus call pnshelper event_notify '{"event":0}'` | 触发唤醒 |
| `ubus call pnshelper event_notify '{"event":7}'` | 开启麦克风 |
| `ubus call pnshelper event_notify '{"event":8}'` | 关闭麦克风 |
| `ubus call mediaplayer player_play_url` | 播放音频 URL |
| `ubus call mibrain ai_service` | 调用小爱 AI 服务 |

### mphelper 命令说明

| mphelper 命令 | 用途 |
|---------------|------|
| `mphelper mute_stat` | 获取播放/静音状态 |
| `mphelper play` | 恢复播放 |
| `mphelper pause` | 暂停播放 |

### 其他系统命令

| 命令 | 用途 |
|------|------|
| `fw_env -g boot_part` | 读取 U-Boot 环境变量（启动分区） |
| `fw_env -s boot_part X` | 设置 U-Boot 环境变量 |
| `micocfg_model` | 读取设备型号配置 |
| `micocfg_sn` | 读取设备序列号 |
| `/usr/sbin/tts_play.sh text` | TTS 语音合成播放 |
| `/etc/init.d/mico_aivs_lab restart` | 重启小爱 AI 服务 |
