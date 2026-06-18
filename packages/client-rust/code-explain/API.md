# client-rust WebSocket 接口文档

> 本文档面向 **Python Server 开发者**，描述如何通过 WebSocket 与运行在小爱音箱上的 client-rust 客户端通信。

---

## 目录

- [1. 连接方式](#1-连接方式)
- [2. 消息格式总览](#2-消息格式总览)
- [3. 四种消息类型详解](#3-四种消息类型详解)
  - [3.1 Request — RPC 请求](#31-request--rpc-请求)
  - [3.2 Response — RPC 响应](#32-response--rpc-响应)
  - [3.3 Event — 事件通知](#33-event--事件通知)
  - [3.4 Stream — 音频数据流](#34-stream--音频数据流)
- [4. Server → Client：RPC 命令](#4-server--clientrpc-命令)
  - [4.1 start_play — 开始播放](#41-start_play--开始播放)
  - [4.2 stop_play — 停止播放](#42-stop_play--停止播放)
  - [4.3 start_recording — 开始录音](#43-start_recording--开始录音)
  - [4.4 stop_recording — 停止录音](#44-stop_recording--停止录音)
  - [4.5 get_version — 获取版本](#45-get_version--获取版本)
  - [4.6 run_shell — 执行 shell 命令](#46-run_shell--执行-shell-命令)
- [5. Server → Client：发送音频流](#5-server--client发送音频流)
- [6. Client → Server：接收事件](#6-client--server接收事件)
  - [6.1 instruction 事件](#61-instruction-事件)
  - [6.2 playing 事件](#62-playing-事件)
  - [6.3 kws 事件](#63-kws-事件)
- [7. Client → Server：接收音频流](#7-client--server接收音频流)
- [8. 典型业务流程](#8-典型业务流程)
  - [8.1 语音对话流程](#81-语音对话流程)
  - [8.2 音乐播放流程](#82-音乐播放流程)
- [9. Python Server 示例代码](#9-python-server-示例代码)
- [10. AudioConfig 参数参考](#10-audioconfig-参数参考)

---

## 1. 连接方式

**角色：** Server 监听等待，Client 主动连接。

```
Server (你的 Python 程序)          Client (音箱上的 Rust 程序)
        │                                    │
        │  监听 0.0.0.0:4399                  │
        │                                    │
        │  ◀──── TCP 连接 ────────────────────│
        │  ◀──── WebSocket 升级 ──────────────│
        │                                    │
        │     双向 WebSocket 通信              │
        │  ◀─────────────────────────────────▶│
```

**Python Server 监听示例：**

```python
import asyncio
import websockets

async def handle_client(websocket):
    """处理一个客户端连接"""
    async for message in websocket:
        # 处理消息...
        pass

async def main():
    async with websockets.serve(handle_client, "0.0.0.0", 4399):
        await asyncio.Future()  # 永久运行

asyncio.run(main())
```

---

## 2. 消息格式总览

所有消息都是 **JSON 文本**，通过 WebSocket 帧传输。

| 消息类型 | WebSocket 帧类型 | 方向 | 用途 |
|----------|-----------------|------|------|
| Request | Text 帧 | 双向 | RPC 请求（Server→Client 为主） |
| Response | Text 帧 | 双向 | RPC 响应 |
| Event | Text 帧 | Client→Server | 设备状态事件通知 |
| Stream | **Binary 帧** | 双向 | 音频数据传输 |

**顶层 JSON 结构（serde 外部标签模式）：**

```json
{"Request":  {"id": "...", "command": "...", "payload": ...}}
{"Response": {"id": "...", "code": 0, "msg": "...", "data": ...}}
{"Event":    {"id": "...", "event": "...", "data": ...}}
```

> **注意：** Stream 类型使用 Binary 帧而非 Text 帧，内部仍是 JSON 格式（bytes 字段为 base64 编码）。

---

## 3. 四种消息类型详解

### 3.1 Request — RPC 请求

用于调用对端注册的命令。

```json
{
  "Request": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "command": "start_play",
    "payload": {
      "pcm": "noop",
      "channels": 1,
      "bits_per_sample": 16,
      "sample_rate": 24000,
      "period_size": 360,
      "buffer_size": 1440
    }
  }
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | 是 | 请求唯一 ID（UUID v4），用于匹配 Response |
| `command` | string | 是 | 命令名称 |
| `payload` | any/null | 否 | 命令参数，结构取决于 command |

### 3.2 Response — RPC 响应

对 Request 的应答，通过 `id` 字段配对。

**成功响应：**
```json
{
  "Response": {
    "id": "0",
    "code": 0,
    "msg": "success"
  }
}
```

**带数据的响应：**
```json
{
  "Response": {
    "id": "0",
    "data": "1.0.0"
  }
}
```

**错误响应：**
```json
{
  "Response": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "code": -1,
    "msg": "command not found: unknown_cmd"
  }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | 对应 Request 的 id |
| `code` | int/null | 状态码：0=成功，-1=失败 |
| `msg` | string/null | 消息描述 |
| `data` | any/null | 返回数据 |

### 3.3 Event — 事件通知

单向通知，不需要响应。

```json
{
  "Event": {
    "id": "660e8400-e29b-41d4-a716-446655440001",
    "event": "playing",
    "data": "Playing"
  }
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | 是 | 事件唯一 ID |
| `event` | string | 是 | 事件类型：`"instruction"` / `"playing"` / `"kws"` |
| `data` | any/null | 否 | 事件数据，结构取决于 event 类型 |

### 3.4 Stream — 音频数据流

通过 WebSocket **Binary 帧**传输，内部是 JSON（bytes 字段为 base64 编码）。

```json
{
  "Stream": {
    "id": "770e8400-e29b-41d4-a716-446655440002",
    "tag": "record",
    "bytes": "UklGRkQAAABXQVZFZm10IBAAAAABAAEA..."
  }
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | 是 | 流唯一 ID |
| `tag` | string | 是 | 数据标签：`"play"`=播放音频，`"record"`=录音数据 |
| `bytes` | base64 string | 是 | 原始音频 PCM 数据的 base64 编码 |
| `data` | any/null | 否 | 可选元数据（一般不用） |

---

## 4. Server → Client：RPC 命令

Server 通过发送 Request 消息调用 Client 注册的命令。Client 处理后返回 Response。

### 4.1 start_play — 开始播放

启动 aplay 进程，准备接收音频数据。

**发送：**
```json
{
  "Request": {
    "id": "uuid-001",
    "command": "start_play",
    "payload": {
      "pcm": "noop",
      "channels": 1,
      "bits_per_sample": 16,
      "sample_rate": 24000,
      "period_size": 360,
      "buffer_size": 1440
    }
  }
}
```

**接收：**
```json
{
  "Response": {
    "id": "0",
    "code": 0,
    "msg": "success"
  }
}
```

**说明：**
- `payload` 可选，不传则使用默认配置（16kHz/16bit/单声道）
- 调用后需要通过 Stream 消息持续发送音频数据
- 如果之前已在播放，会自动停止旧的播放再启动新的

### 4.2 stop_play — 停止播放

停止当前音频播放，清理 aplay 进程。

**发送：**
```json
{
  "Request": {
    "id": "uuid-002",
    "command": "stop_play",
    "payload": null
  }
}
```

**接收：**
```json
{
  "Response": {
    "id": "0",
    "code": 0,
    "msg": "success"
  }
}
```

### 4.3 start_recording — 开始录音

启动 arecord 进程，Client 开始录制麦克风音频并通过 Stream 消息发送给 Server。

**发送：**
```json
{
  "Request": {
    "id": "uuid-003",
    "command": "start_recording",
    "payload": {
      "pcm": "noop",
      "channels": 1,
      "bits_per_sample": 16,
      "sample_rate": 16000,
      "period_size": 360,
      "buffer_size": 1440
    }
  }
}
```

**接收：**
```json
{
  "Response": {
    "id": "0",
    "code": 0,
    "msg": "success"
  }
}
```

**说明：**
- `payload` 可选，不传则使用默认配置
- 调用后 Client 会持续发送 `tag="record"` 的 Stream 消息
- 内部可能将 16-bit 请求升级为 32-bit 录制（A113 硬件要求），然后自动转换回 16-bit 发送

### 4.4 stop_recording — 停止录音

停止麦克风录音。

**发送：**
```json
{
  "Request": {
    "id": "uuid-004",
    "command": "stop_recording",
    "payload": null
  }
}
```

**接收：**
```json
{
  "Response": {
    "id": "0",
    "code": 0,
    "msg": "success"
  }
}
```

### 4.5 get_version — 获取版本

获取 Client 的版本号。

**发送：**
```json
{
  "Request": {
    "id": "uuid-005",
    "command": "get_version",
    "payload": null
  }
}
```

**接收：**
```json
{
  "Response": {
    "id": "0",
    "data": "1.0.0"
  }
}
```

### 4.6 run_shell — 执行 shell 命令

在音箱设备上执行 shell 命令，返回执行结果。

**发送：**
```json
{
  "Request": {
    "id": "uuid-006",
    "command": "run_shell",
    "payload": "echo $(micocfg_model) $(micocfg_sn)"
  }
}
```

**接收：**
```json
{
  "Response": {
    "id": "0",
    "data": {
      "stdout": "LX04 123456789\n",
      "stderr": "",
      "exit_code": 0
    }
  }
}
```

**payload 格式：** 纯字符串，就是要执行的 shell 命令。

**返回 data 结构：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `stdout` | string | 标准输出 |
| `stderr` | string | 标准错误 |
| `exit_code` | int | 退出码（0=成功） |

**常用 shell 命令参考：**

| 命令 | 功能 |
|------|------|
| `tts_play.sh '你好'` | TTS 语音播放（阻塞） |
| `miplayer -f 'http://...'` | 播放音频 URL（阻塞） |
| `ubus call mibrain text_to_speech '{"text":"你好","save":0}'` | TTS 播放（非阻塞） |
| `ubus call mediaplayer player_play_url '{"url":"http://...","type":1}'` | URL 播放（非阻塞） |
| `ubus call mibrain ai_service '{"nlp":1,"nlp_text":"今天天气","tts":1}'` | 让小爱执行指令 |
| `ubus call pnshelper event_notify '{"event":0}'` | 触发唤醒 |
| `ubus call pnshelper event_notify '{"event":7}'` | 开启麦克风 |
| `ubus call pnshelper event_notify '{"event":8}'` | 关闭麦克风 |
| `mphelper mute_stat` | 获取播放状态 |
| `mphelper play` / `mphelper pause` | 播放/暂停 |
| `micocfg_model` | 获取设备型号 |
| `micocfg_sn` | 获取设备序列号 |
| `/etc/init.d/mico_aivs_lab restart` | 重启小爱服务 |

---

## 5. Server → Client：发送音频流

Server 通过 Binary 帧发送音频数据给 Client 播放。

**前提：** 必须先发送 `start_play` RPC 命令启动播放器。

**Binary 帧内容（JSON，bytes 为 base64 编码）：**
```json
{
  "Stream": {
    "id": "770e8400-e29b-41d4-a716-446655440002",
    "tag": "play",
    "bytes": "UklGRkQAAABXQVZFZm10IBAAAAABAAEA..."
  }
}
```

**Python 发送示例：**

```python
import json
import base64
import uuid

async def send_audio(websocket, pcm_bytes: bytes):
    """发送音频数据给客户端播放"""
    stream = {
        "Stream": {
            "id": str(uuid.uuid4()),
            "tag": "play",
            "bytes": base64.b64encode(pcm_bytes).decode("ascii")
        }
    }
    await websocket.send(json.dumps(stream).encode("utf-8"))
```

**音频格式要求：**
- 格式：原始 PCM（raw PCM）
- 字节序：Little-Endian
- 与 `start_play` 中配置的 AudioConfig 一致
- 常用配置：16-bit signed, 24000Hz, 单声道

---

## 6. Client → Server：接收事件

Client 通过 Text 帧发送 Event 消息，上报设备状态变化。

### 6.1 instruction 事件

小爱内置语音识别结果。当用户对音箱说话时，小爱的 ASR 系统会输出识别结果。

**消息格式：**
```json
{
  "Event": {
    "id": "...",
    "event": "instruction",
    "data": {
      "NewLine": "{\"header\":{\"dialog_id\":\"...\",\"id\":\"...\",\"name\":\"RecognizeResult\",\"namespace\":\"SpeechRecognizer\"},\"payload\":{\"is_final\":true,\"is_vad_begin\":false,\"results\":[{\"confidence\":0.95,\"text\":\"今天天气怎么样\"}]}}"
    }
  }
}
```

> **注意：** `data.NewLine` 是一个 **JSON 字符串**，需要二次解析。

**解析后结构：**

```json
{
  "header": {
    "dialog_id": "xxx",
    "id": "xxx",
    "name": "RecognizeResult",
    "namespace": "SpeechRecognizer"
  },
  "payload": {
    "is_final": true,
    "is_vad_begin": false,
    "results": [
      {
        "confidence": 0.95,
        "text": "今天天气怎么样"
      }
    ]
  }
}
```

**关键字段：**

| 字段 | 说明 |
|------|------|
| `header.namespace` | `"SpeechRecognizer"` 表示语音识别结果 |
| `header.name` | `"RecognizeResult"` 表示识别结果 |
| `payload.is_final` | `true` = 最终结果，`false` = 中间结果 |
| `payload.is_vad_begin` | `true` = VAD 检测到说话开始（唤醒） |
| `payload.results[0].text` | 识别出的文本 |

**典型场景：**
- `is_vad_begin=true, text=""` → 用户唤醒了小爱（还没说话）
- `is_final=true, text="今天天气"` → 用户说完了，最终识别结果

**Python 解析示例：**

```python
def handle_instruction(event_data: dict):
    new_line = event_data.get("data", {}).get("NewLine")
    if not new_line:
        return
    line = json.loads(new_line)
    header = line.get("header", {})
    if header.get("namespace") != "SpeechRecognizer":
        return
    if header.get("name") != "RecognizeResult":
        return
    payload = line.get("payload", {})
    if payload.get("is_vad_begin") and not payload.get("results"):
        print("用户唤醒了小爱")
    elif payload.get("is_final") and payload.get("results"):
        text = payload["results"][0].get("text", "")
        print(f"用户说了: {text}")
```

### 6.2 playing 事件

音乐播放状态变化。

**消息格式：**
```json
{
  "Event": {
    "id": "...",
    "event": "playing",
    "data": "Playing"
  }
}
```

`data` 取值：

| 值 | 含义 |
|------|------|
| `"Playing"` | 正在播放 |
| `"Paused"` | 已暂停 |
| `"Idle"` | 空闲（无播放） |

### 6.3 kws 事件

唤醒词检测事件（来自自定义 KWS 模块）。

**消息格式：**
```json
{
  "Event": {
    "id": "...",
    "event": "kws",
    "data": "Keyword"
  }
}
```

或引擎启动事件：
```json
{
  "Event": {
    "id": "...",
    "event": "kws",
    "data": "Started"
  }
}
```

`data` 取值：

| 值 | 含义 |
|------|------|
| `"Started"` | KWS 引擎已启动 |
| `"Keyword"` | 检测到唤醒词（具体词由设备配置决定） |

---

## 7. Client → Server：接收音频流

Client 通过 Binary 帧发送录音数据给 Server。

**前提：** Server 必须先发送 `start_recording` RPC 命令。

**Binary 帧内容：**
```json
{
  "Stream": {
    "id": "...",
    "tag": "record",
    "bytes": "UklGRkQAAABXQVZFZm10IBAAAAABAAEA..."
  }
}
```

**Python 接收示例：**

```python
import json
import base64

async def handle_message(websocket, message):
    if isinstance(message, bytes):
        # Binary 帧 → Stream（音频数据）
        stream = json.loads(message)
        tag = stream["Stream"]["tag"]
        audio_bytes = base64.b64decode(stream["Stream"]["bytes"])
        if tag == "record":
            # audio_bytes 是原始 PCM 数据
            # 与 start_recording 中配置的 AudioConfig 一致
            process_audio(audio_bytes)
    else:
        # Text 帧 → Request/Response/Event
        data = json.loads(message)
        if "Event" in data:
            handle_event(data["Event"])
        elif "Request" in data:
            handle_request(data["Request"])
        elif "Response" in data:
            handle_response(data["Response"])
```

---

## 8. 典型业务流程

### 8.1 语音对话流程

```
Server                                 Client (音箱)
  │                                       │
  │  ◀── WebSocket 连接 ──────────────────│
  │                                       │
  │  Request("start_recording") ────────▶ │  开启麦克风
  │  ◀── Response(success) ──────────────│
  │                                       │
  │  ◀── Stream(tag:"record", PCM) ──────│  持续发送录音
  │  ◀── Stream(tag:"record", PCM) ──────│
  │  ◀── Stream(tag:"record", PCM) ──────│
  │                                       │
  │  [Server 端做 VAD + ASR，检测到说完]    │
  │                                       │
  │  Request("stop_recording") ─────────▶ │  关闭麦克风
  │  ◀── Response(success) ──────────────│
  │                                       │
  │  [Server 端调用 LLM 生成回复]           │
  │  [Server 端 TTS 合成音频]              │
  │                                       │
  │  Request("start_play") ─────────────▶ │  启动播放器
  │  ◀── Response(success) ──────────────│
  │                                       │
  │  Stream(tag:"play", PCM) ───────────▶ │  发送音频数据
  │  Stream(tag:"play", PCM) ───────────▶ │
  │  Stream(tag:"play", PCM) ───────────▶ │
  │                                       │
  │  Request("stop_play") ──────────────▶ │  停止播放
  │  ◀── Response(success) ──────────────│
```

### 8.2 音乐播放流程

```
Server                                 Client (音箱)
  │                                       │
  │  ◀── Event("instruction", ASR文本) ──│  用户说了"播放音乐"
  │                                       │
  │  run_shell("ubus call mediaplayer     │
  │    player_play_url ...") ────────────▶│  让小爱播放音乐
  │  ◀── Response(stdout) ──────────────│
  │                                       │
  │  ◀── Event("playing", "Playing") ────│  播放状态变化
  │                                       │
  │  ... 用户说"暂停" ...                  │
  │  ◀── Event("instruction", ASR文本) ──│
  │                                       │
  │  run_shell("mphelper pause") ────────▶│  暂停播放
  │  ◀── Response(stdout) ──────────────│
  │                                       │
  │  ◀── Event("playing", "Paused") ─────│  状态变化
```

---

## 9. Python Server 示例代码

```python
import asyncio
import json
import base64
import uuid
from typing import Optional

import websockets


class XiaoAIServer:
    def __init__(self, host="0.0.0.0", port=4399):
        self.host = host
        self.port = port
        self.websocket: Optional[websockets.WebSocketServerProtocol] = None
        self.pending_responses: dict[str, asyncio.Future] = {}

    async def start(self):
        """启动服务器"""
        async with websockets.serve(self._handle_client, self.host, self.port):
            print(f"Server listening on {self.host}:{self.port}")
            await asyncio.Future()

    async def _handle_client(self, websocket):
        """处理客户端连接"""
        self.websocket = websocket
        print(f"Client connected: {websocket.remote_address}")
        try:
            async for message in websocket:
                await self._on_message(message)
        except websockets.ConnectionClosed:
            print("Client disconnected")
        finally:
            self.websocket = None

    async def _on_message(self, message):
        """分发消息"""
        if isinstance(message, bytes):
            # Binary 帧 → Stream
            stream = json.loads(message)
            tag = stream["Stream"]["tag"]
            audio_bytes = base64.b64decode(stream["Stream"]["bytes"])
            await self._on_stream(tag, audio_bytes)
        else:
            # Text 帧 → Request / Response / Event
            data = json.loads(message)
            if "Event" in data:
                await self._on_event(data["Event"])
            elif "Response" in data:
                resp = data["Response"]
                req_id = resp.get("id")
                if req_id and req_id in self.pending_responses:
                    self.pending_responses[req_id].set_result(resp)

    # ========== 接收事件 ==========

    async def _on_event(self, event: dict):
        """处理 Client 上报的事件"""
        event_type = event.get("event")
        event_data = event.get("data")

        if event_type == "instruction":
            await self._on_instruction(event_data)
        elif event_type == "playing":
            print(f"Playing status: {event_data}")
        elif event_type == "kws":
            print(f"KWS event: {event_data}")

    async def _on_instruction(self, data: dict):
        """处理语音识别结果"""
        new_line = data.get("NewLine")
        if not new_line:
            return
        line = json.loads(new_line)
        header = line.get("header", {})
        if header.get("namespace") != "SpeechRecognizer":
            return
        payload = line.get("payload", {})
        if payload.get("is_vad_begin"):
            print("Wakeup detected")
        elif payload.get("is_final") and payload.get("results"):
            text = payload["results"][0].get("text", "")
            print(f"Recognized: {text}")

    async def _on_stream(self, tag: str, audio_bytes: bytes):
        """处理 Client 发送的音频流"""
        if tag == "record":
            # 处理录音数据（PCM 原始格式）
            # AudioConfig: 16kHz, 16-bit, mono
            pass

    # ========== 发送 RPC 命令 ==========

    async def call_remote(self, command: str, payload=None, timeout=10.0) -> dict:
        """发送 RPC 请求并等待响应"""
        req_id = str(uuid.uuid4())
        future = asyncio.get_event_loop().create_future()
        self.pending_responses[req_id] = future

        request = {
            "Request": {
                "id": req_id,
                "command": command,
                "payload": payload
            }
        }
        await self.websocket.send(json.dumps(request))

        try:
            response = await asyncio.wait_for(future, timeout=timeout)
            return response
        except asyncio.TimeoutError:
            raise TimeoutError(f"RPC {command} timed out")
        finally:
            self.pending_responses.pop(req_id, None)

    async def start_play(self, sample_rate=24000):
        """开始播放音频"""
        config = {
            "pcm": "noop",
            "channels": 1,
            "bits_per_sample": 16,
            "sample_rate": sample_rate,
            "period_size": sample_rate * 60 // 1000,
            "buffer_size": sample_rate * 60 // 1000 * 4
        }
        return await self.call_remote("start_play", config)

    async def stop_play(self):
        """停止播放"""
        return await self.call_remote("stop_play")

    async def start_recording(self, sample_rate=16000):
        """开始录音"""
        config = {
            "pcm": "noop",
            "channels": 1,
            "bits_per_sample": 16,
            "sample_rate": sample_rate,
            "period_size": sample_rate * 60 // 1000,
            "buffer_size": sample_rate * 60 // 1000 * 4
        }
        return await self.call_remote("start_recording", config)

    async def stop_recording(self):
        """停止录音"""
        return await self.call_remote("stop_recording")

    async def run_shell(self, script: str, timeout=10.0) -> dict:
        """执行 shell 命令"""
        resp = await self.call_remote("run_shell", script, timeout)
        return resp.get("data", {})

    async def get_version(self) -> str:
        """获取客户端版本"""
        resp = await self.call_remote("get_version")
        return resp.get("data", "")

    # ========== 发送音频流 ==========

    async def send_audio(self, pcm_bytes: bytes):
        """发送音频数据给客户端播放"""
        stream = {
            "Stream": {
                "id": str(uuid.uuid4()),
                "tag": "play",
                "bytes": base64.b64encode(pcm_bytes).decode("ascii")
            }
        }
        await self.websocket.send(json.dumps(stream).encode("utf-8"))


# ========== 使用示例 ==========

async def main():
    server = XiaoAIServer()

    # 在另一个协程中启动服务器
    asyncio.create_task(server.start())

    # 等待客户端连接...
    while not server.websocket:
        await asyncio.sleep(0.1)

    # 获取版本
    version = await server.get_version()
    print(f"Client version: {version}")

    # 执行 shell 命令
    result = await server.run_shell("micocfg_model")
    print(f"Device model: {result['stdout'].strip()}")

    # TTS 播放
    await server.run_shell("/usr/sbin/tts_play.sh '你好，我是小爱'")

    # 让小爱回答问题
    await server.run_shell(
        "ubus call mibrain ai_service "
        "'{\"nlp\":1,\"nlp_text\":\"今天天气怎么样\",\"tts\":1}'"
    )

    await asyncio.Future()


if __name__ == "__main__":
    asyncio.run(main())
```

---

## 10. AudioConfig 参数参考

### 结构

```json
{
  "pcm": "noop",
  "channels": 1,
  "bits_per_sample": 16,
  "sample_rate": 16000,
  "period_size": 160,
  "buffer_size": 480
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `pcm` | string | ALSA 设备名，使用流式传输时设为 `"noop"` |
| `channels` | int | 声道数：1=单声道，2=立体声 |
| `bits_per_sample` | int | 位深：16 或 32 |
| `sample_rate` | int | 采样率（Hz） |
| `period_size` | int | 每周期帧数（影响延迟） |
| `buffer_size` | int | 缓冲区帧数（影响稳定性） |

### 常用配置

| 场景 | sample_rate | period_size | buffer_size | 说明 |
|------|-------------|-------------|-------------|------|
| 录音（语音识别） | 16000 | 360 | 1440 | 16kHz, 22.5ms/period |
| 播放（TTS） | 24000 | 360 | 1440 | 24kHz, 15ms/period |
| 播放（音乐） | 48000 | 960 | 3840 | 48kHz, 20ms/period |
| 默认 | 16000 | 160 | 480 | 10ms/period |

**计算公式：**
- 每帧字节数 = `channels * (bits_per_sample / 8)`
- period 时长(ms) = `period_size / sample_rate * 1000`
- 每次发送的 bytes 长度 = `buffer_size * 每帧字节数`

**示例：** 16kHz/16bit/单声道录音
- 每帧 = 1 * 2 = 2 字节
- buffer_size=1440 → 每次发送 1440 * 2 = 2880 字节
- 时长 = 1440 / 16000 = 90ms 的音频
