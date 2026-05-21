# HarBeat 项目交接文档

## 项目概览

HarBeat 是一个面向街舞 Cypher 场景的 AI DJ 系统，包含三个核心组件：
- **手机 App**：Flutter 跨平台应用，负责用户界面和控制命令
- **Jetson 后端**：负责音乐分析、MixPlan 生成、用户管理
- **RK3588 现场盒**：负责现场音频播放、混音、加花

---

## 手机 App（负责人 C）已完成工作

### ✅ 已实现功能

#### 1. 项目基础设施
- Flutter 3.22.x 项目初始化
- 完整的项目目录结构
- GitHub Actions 自动构建配置（可成功生成 APK）

#### 2. API 客户端层
- **JetsonClient**：与 Jetson 后端通信的客户端
  - 登录认证
  - 音乐库管理
  - MixPlan 生成（支持 SSE 流式返回）
  - AssetManifest 获取
  - 响应包络解析（`code`/`message`/`data` 格式）

- **RKClient**：与 RK3588 现场盒通信的客户端
  - 播放/暂停/继续/切歌
  - 能量等级切换（低/中/高）
  - 风格切换（Hiphop/Breaking）
  - 混音切换（平滑/能量提升/硬切）
  - 一键加花（6个音效）
  - WebSocket 实时状态订阅

#### 3. 状态管理（Riverpod）
- `authProvider`：用户认证状态
- `deviceProvider`：设备连接状态
- `playbackProvider`：播放状态
- `liveProvider`：现场控制状态
- **状态保持机制**：所有关键 Provider 使用 `ref.keepAlive()` 避免页面切换时丢失状态

#### 4. 核心页面
- **登录页** (`login_page.dart`)：支持 Mock 模式离线登录
- **设备连接页** (`device_connection_page.dart`)：局域网设备发现、手动添加、配对流程
- **现场控制页** (`live_page.dart`)：
  - 播放/暂停/下一首按钮
  - 能量等级切换（3个按钮）
  - 风格切换（2个按钮）
  - 混音过渡（3个按钮）
  - 6个一键加花按钮
  - 实时播放状态显示
  - 自动配对码填充

#### 5. 协议模型
- P1 `SongStatus`：歌曲分析状态
- P2 `MixPlan`：混音计划
- P3 `AssetManifest`：资产清单
- P5 `PlaybackState`：播放状态
- P8 `DeviceInfo`：设备信息

#### 6. 开发工具
- **Mock 服务器** (`mock_rk3588_server.py`)：
  - 模拟 RK3588 所有 API 接口
  - 支持 WebSocket 实时状态推送
  - 支持配对流程
  - 模拟播放状态变化

---

### 📂 项目结构

```
lib/
├── core/
│   ├── api/
│   │   ├── jetson_client.dart      # Jetson 后端客户端 ✅
│   │   └── rk_client.dart          # RK3588 客户端 ✅
│   ├── config/
│   │   ├── api_config.dart         # API 配置
│   │   └── theme_config.dart       # 主题配置
│   ├── network/
│   │   └── api_client.dart         # Dio 网络层
│   ├── services/
│   │   ├── audio_player_service.dart
│   │   ├── bluetooth_service.dart
│   │   ├── hardware_service.dart   # 硬件服务（设备发现）✅
│   │   └── session_service.dart
│   └── utils/
│       ├── helpers.dart
│       └── logger.dart
├── data/
│   ├── models/
│   └── services/
│       ├── auth_service.dart
│       ├── data_services.dart
│       └── song_service.dart
├── models/
│   ├── models.dart
│   ├── p1_song_status.dart         # P1 模型 ✅
│   ├── p2_mix_plan.dart            # P2 模型 ✅
│   ├── p3_asset_manifest.dart      # P3 模型 ✅
│   ├── p5_playback_state.dart      # P5 模型 ✅
│   └── p8_device_info.dart         # P8 模型 ✅
├── pages/
│   ├── live_page.dart              # 现场页 ✅
│   ├── login_page.dart             # 登录页 ✅
│   ├── main_page.dart
│   ├── prep_page.dart
│   └── replay_page.dart
├── presentation/
│   ├── pages/
│   │   ├── device_connection_page.dart  # 设备连接页 ✅
│   │   ├── home_page.dart
│   │   └── ...
│   └── widgets/
│       └── song_card.dart
├── routing/
│   └── app_router.dart             # 路由配置
├── state/
│   └── providers.dart              # Riverpod 状态管理 ✅
├── widgets/
│   ├── nine_key_grid.dart
│   └── transport_bar.dart
└── main.dart                       # 应用入口 ✅

scripts/
└── mock_rk3588_server.py           # RK3588 模拟服务器 ✅

.github/
└── workflows/
    └── build.yml                   # GitHub Actions 自动构建 ✅
```

---

## 还需要其他负责人完成的工作

---

### Jetson 后端（负责人 A）需要做的

#### 1. 启动 Jetson 服务
- 确认 Jetson 已连接到 Tailscale
- 启动 `harbeat.service` 服务
- 确认 8000 端口可访问
- 提供测试账号信息

#### 2. 确认 API 接口实现
- ✅ `/api/users/login` - 登录
- ✅ `/api/library/songs` - 歌曲列表
- ✅ `/api/library/songs/{id}/status` - 歌曲分析状态
- ✅ `/api/playlists/{id}/dj-mix-stream` - SSE 流式生成 MixPlan
- ✅ `/api/playlists/{id}/manifest` - 资产清单
- ✅ `/health` - 健康检查

#### 3. 响应格式
所有接口必须返回统一格式：
```json
{
  "code": 0,
  "message": "ok",
  "data": {...}
}
```

---

### RK3588 现场盒（负责人 B）需要做的

#### 1. 启动 RK3588 服务
- 确认 RK3588 已连接到同一局域网（如手机热点）
- 启动 `edge-agent.service`（端口 9000）
- 确认服务可访问
- 提供 RK3588 的 IP 地址

#### 2. 实现完整 API 接口
参考文档 `jetson-handoff-for-rk3588.md` 和 `team-rk3588-edge.md`，需要实现：

**基础接口：**
- ✅ `GET /api/edge/info` - 设备信息
- ✅ `GET /api/edge/status` - 播放状态
- ✅ `GET /api/edge/pair/start` - 开始配对
- ✅ `POST /api/edge/pair/confirm` - 确认配对

**播放控制：**
- ✅ `POST /play` - 播放
- ✅ `POST /pause` - 暂停
- ✅ `POST /resume` - 继续
- ✅ `POST /next` - 下一首
- ✅ `POST /seek` - 跳转

**现场控制：**
- ✅ `POST /energy` - 能量切换（`level: low|medium|high`）
- ✅ `POST /style` - 风格切换（`style: hiphop|breaking`）
- ✅ `POST /mix` - 混音过渡（`transition: smooth|energy|cut`）
- ✅ `POST /trigger` - 加花（`key: 1-6`）
- ✅ `POST /loop` - 循环控制（`enabled: bool`）

**WebSocket：**
- ✅ `ws://<ip>:9000/ws/control?token=...` - 实时状态推送
- 推送格式与文档 P5 `PlaybackState` 一致

#### 3. 验证清单
- [ ] App 能成功发现并连接 RK3588
- [ ] 配对流程正常工作
- [ ] 播放/暂停/下一首有响应
- [ ] 能量/风格/混音按钮有响应
- [ ] 加花按钮有响应
- [ ] WebSocket 状态能正常推送到 App
- [ ] 服务器终端有对应的日志输出

---

## 测试和验证

### 1. 使用 Mock 服务器测试（本地）

如果你暂时没有真实的硬件，可以使用模拟服务器测试：

```bash
# 启动模拟 RK3588 服务器
python scripts/mock_rk3588_server.py

# 在 App 中手动添加设备
# 地址：http://localhost:9000 (或 http://<你的电脑IP>:9000)
# 配对码：123456
```

### 2. 连接真实硬件

确认以下服务都在运行：
- Jetson（Tailscale 可访问）
- RK3588（局域网可访问）

然后：
1. App 输入真实账号登录
2. 连接 RK3588 设备
3. 测试所有功能

---

## 相关文档

请参考以下文档了解完整需求：
- `team-jetson-backend.md` - Jetson 后端开发文档
- `team-rk3588-edge.md` - RK3588 现场盒开发文档
- `team-mobile-app.md` - App 开发文档
- `cypher-feature-flows.md` - 功能流程文档
- `jetson-handoff-for-rk3588.md` - RK3588 接口文档
- `jetson-handoff-for-app.md` - Jetson 接口文档

---

## 联系信息

- **项目仓库**：https://github.com/MOONFISH2233/harbeat
- **问题反馈**：提交 Issue 到 GitHub

---

*交接日期：2026-05-21*
