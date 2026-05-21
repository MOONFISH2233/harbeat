# HarBeat App 与 RK3588 接口规范文档

**版本**: v1.0  
**日期**: 2026-05-20  
**适用对象**: RK3588 硬件开发同学

---

## 一、概述

本文档定义了 HarBeat 移动端 App 与 RK3588 现场盒之间的通信协议和 API 接口规范。

### 1.1 技术栈
- **协议**: HTTP/1.1 + WebSocket
- **数据格式**: JSON
- **编码**: UTF-8
- **端口**: 默认 `9000`（可配置）

### 1.2 认证机制
- 使用 Bearer Token 认证
- Token 通过配对接口获取
- Token 有效期：3600 秒（1小时）

---

## 二、API 接口列表

### 2.1 设备信息接口

#### GET /api/edge/info

**功能**: 获取设备基本信息

**请求**: 无

**响应**:
```json
{
  "device_id": "rk3588-001",
  "model": "RK3588",
  "status": "connected",
  "battery": 100,
  "cpu_pct": 25.0,
  "mem_mb": 2048,
  "timestamp": "2026-05-20T12:00:00Z"
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| device_id | string | 设备唯一标识 |
| model | string | 设备型号 |
| status | string | 连接状态（connected/disconnected） |
| battery | int | 电量百分比 |
| cpu_pct | float | CPU 使用率 |
| mem_mb | int | 内存大小（MB） |
| timestamp | string | 响应时间戳 |

---

### 2.2 播放状态接口

#### GET /api/edge/status

**功能**: 获取当前播放状态

**请求**: 无

**响应**:
```json
{
  "type": "playback_state",
  "ts": 1716172800000,
  "playing": true,
  "paused": false,
  "current_song_id": 26,
  "position_sec": 45.5,
  "duration_sec": 180.0,
  "bpm": 120,
  "active_loops": []
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| type | string | 消息类型，固定为 "playback_state" |
| ts | int | 时间戳（毫秒） |
| playing | bool | 是否正在播放 |
| paused | bool | 是否暂停 |
| current_song_id | int | 当前歌曲 ID |
| position_sec | float | 当前播放位置（秒） |
| duration_sec | float | 歌曲时长（秒） |
| bpm | int | 当前歌曲 BPM |
| active_loops | array | 活跃的循环列表 |

---

### 2.3 配对接口

#### GET /api/edge/pair/start

**功能**: 开始配对，获取配对码

**请求**: 无

**响应**:
```json
{
  "device_id": "mock-device-001",
  "name": "RK3588-MOCK",
  "local_url": "http://192.168.1.100:9000",
  "pair_code": "123456",
  "expires_in_sec": 120,
  "is_connected": false,
  "last_connected_time": 0
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| device_id | string | 设备 ID |
| name | string | 设备名称 |
| local_url | string | 本地访问地址 |
| pair_code | string | 6位配对码 |
| expires_in_sec | int | 配对码有效期（秒） |
| is_connected | bool | 是否已连接 |
| last_connected_time | int | 上次连接时间戳 |

---

#### POST /api/edge/pair/confirm

**功能**: 确认配对，获取设备 Token

**请求体**:
```json
{
  "device_id": "mock-device-001",
  "pair_code": "123456",
  "client_name": "HarBeat Mobile",
  "client_type": "mobile"
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| device_id | string | 设备 ID |
| pair_code | string | 配对码 |
| client_name | string | 客户端名称 |
| client_type | string | 客户端类型（mobile/web） |

**成功响应**:
```json
{
  "device_token": "mock-device-token-12345",
  "expires_in": 3600,
  "success": true
}
```

**失败响应**:
```json
{
  "success": false,
  "message": "Invalid pair code"
}
```

---

### 2.4 播放控制接口

#### POST /play

**功能**: 播放指定歌曲

**请求体**:
```json
{
  "song_id": 26,
  "start_at_sec": 0.0
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| song_id | int | 歌曲 ID |
| start_at_sec | float | 开始播放位置（秒），可选，默认 0 |

**响应**:
```json
{
  "success": true,
  "message": "Playing song 26"
}
```

---

#### POST /pause

**功能**: 暂停播放

**请求体**: `{}`

**响应**:
```json
{
  "success": true,
  "message": "Paused"
}
```

---

#### POST /resume

**功能**: 继续播放

**请求体**: `{}`

**响应**:
```json
{
  "success": true,
  "message": "Resumed"
}
```

---

#### POST /next

**功能**: 播放下一首

**请求体**: `{}`

**响应**:
```json
{
  "success": true,
  "message": "Playing song 27"
}
```

---

#### POST /seek

**功能**: 跳转到指定位置

**请求体**:
```json
{
  "sec": 45.5
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| sec | float | 目标位置（秒） |

**响应**:
```json
{
  "success": true,
  "position_sec": 45.5
}
```

---

### 2.5 能量控制接口

#### POST /energy

**功能**: 设置能量等级

**请求体**:
```json
{
  "level": "high"
}
```

| 字段 | 类型 | 可选值 |
|------|------|--------|
| level | string | low / medium / high |

**响应**:
```json
{
  "success": true,
  "energy": "high"
}
```

---

### 2.6 风格切换接口

#### POST /style

**功能**: 设置音乐风格

**请求体**:
```json
{
  "style": "hiphop"
}
```

| 字段 | 类型 | 可选值 |
|------|------|--------|
| style | string | hiphop / breaking |

**响应**:
```json
{
  "success": true,
  "style": "hiphop"
}
```

---

### 2.7 混音过渡接口

#### POST /mix

**功能**: 执行混音过渡

**请求体**:
```json
{
  "transition": "smooth"
}
```

| 字段 | 类型 | 可选值 | 说明 |
|------|------|--------|------|
| transition | string | smooth | 平滑过渡 |
| | | energy | 能量提升 |
| | | cut | 硬切 |

**响应**:
```json
{
  "success": true,
  "transition": "smooth"
}
```

---

### 2.8 加花触发接口

#### POST /trigger

**功能**: 触发音效加花

**请求体**:
```json
{
  "key": 1
}
```

| 字段 | 类型 | 可选值 | 对应音效 |
|------|------|--------|----------|
| key | int | 1 | ha! |
| | | 2 | scratch |
| | | 3 | horn |
| | | 4 | drum |
| | | 5 | bass |
| | | 6 | hat |

**响应**:
```json
{
  "success": true,
  "key": 1,
  "latency_ms": 25
}
```

---

### 2.9 循环控制接口

#### POST /loop

**功能**: 设置循环模式

**请求体**:
```json
{
  "enabled": true
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| enabled | bool | 是否启用循环 |

**响应**:
```json
{
  "success": true,
  "loop_enabled": true
}
```

---

## 三、WebSocket 实时推送

### 3.1 连接端点

```
ws://<RK3588_IP>:9000/ws/control?token=<device_token>
```

### 3.2 消息格式

**播放状态推送**:
```json
{
  "type": "playback_state",
  "ts": 1716172800000,
  "playing": true,
  "paused": false,
  "current_song_id": 26,
  "position_sec": 45.5,
  "duration_sec": 180.0,
  "bpm": 120,
  "active_loops": []
}
```

**设备信息推送**:
```json
{
  "type": "device_info",
  "device_id": "rk3588-001",
  "model": "RK3588",
  "status": "connected",
  "battery": 100
}
```

### 3.3 消息类型

| type | 说明 |
|------|------|
| playback_state | 播放状态更新 |
| device_info | 设备信息更新 |
| sync_progress | 同步进度更新 |

---

## 四、错误处理

### 4.1 HTTP 状态码

| 状态码 | 说明 |
|--------|------|
| 200 | 请求成功 |
| 400 | 请求参数错误 |
| 401 | 未授权（Token 无效） |
| 404 | 接口不存在 |
| 500 | 服务器内部错误 |

### 4.2 错误响应格式

```json
{
  "success": false,
  "message": "Error description",
  "error_code": "ERROR_CODE"
}
```

---

## 五、模拟服务器使用说明

### 5.1 启动方式

```bash
# 安装依赖
pip install fastapi uvicorn websockets

# 启动服务器
python mock_rk3588_server.py
```

### 5.2 测试命令

```bash
# 测试设备信息
curl http://localhost:9000/api/edge/info

# 测试播放状态
curl http://localhost:9000/api/edge/status

# 测试暂停
curl -X POST http://localhost:9000/pause

# 测试能量切换
curl -X POST http://localhost:9000/energy -H "Content-Type: application/json" -d '{"level":"high"}'
```

---

## 六、对接检查清单

请确认以下接口已正确实现：

| 序号 | 接口 | 方法 | 状态 |
|------|------|------|------|
| 1 | `/api/edge/info` | GET | ☐ |
| 2 | `/api/edge/status` | GET | ☐ |
| 3 | `/api/edge/pair/start` | GET | ☐ |
| 4 | `/api/edge/pair/confirm` | POST | ☐ |
| 5 | `/play` | POST | ☐ |
| 6 | `/pause` | POST | ☐ |
| 7 | `/resume` | POST | ☐ |
| 8 | `/next` | POST | ☐ |
| 9 | `/seek` | POST | ☐ |
| 10 | `/energy` | POST | ☐ |
| 11 | `/style` | POST | ☐ |
| 12 | `/mix` | POST | ☐ |
| 13 | `/trigger` | POST | ☐ |
| 14 | `/loop` | POST | ☐ |
| 15 | `/ws/control` | WebSocket | ☐ |

---

## 七、相关文件

| 文件 | 路径 | 说明 |
|------|------|------|
| 模拟服务器 | `scripts/mock_rk3588_server.py` | API 参考实现 |
| App 客户端 | `lib/core/api/rk_client.dart` | App 端调用代码 |
| 硬件服务 | `lib/core/services/hardware_service.dart` | 连接管理 |

---

**文档版本**: v1.0  
**最后更新**: 2026-05-20  
**联系人**: HarBeat App 开发组