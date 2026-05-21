# HarBeat App

面向街舞 Cypher 场景的 AI DJ 移动应用。

## 项目简介

HarBeat 是一个完整的 AI DJ 系统，包含三个核心组件：

- **手机 App**：Flutter 跨平台应用，负责用户界面和控制命令
- **Jetson 后端**：负责音乐分析、MixPlan 生成、用户管理
- **RK3588 现场盒**：负责现场音频播放、混音、加花

## 技术栈

- **框架**：Flutter 3.22.x
- **语言**：Dart 3.0+
- **状态管理**：Riverpod 2.x
- **路由**：GoRouter
- **网络**：Dio
- **实时通信**：WebSocket

## 功能特性

### ✅ 已实现

- 用户认证（支持 Mock 模式离线登录）
- 设备发现与连接
- RK3588 配对流程
- 现场 DJ 控制台
  - 播放/暂停/下一首
  - 能量等级切换（低/中/高）
  - 风格切换（Hiphop/Breaking）
  - 混音过渡（平滑/能量提升/硬切）
  - 6个一键加花按钮
- WebSocket 实时状态同步
- 状态保持机制（页面切换时不丢失连接状态）

### 📱 核心页面

- **登录页**：用户认证
- **设备连接页**：发现和连接 RK3588
- **现场控制页**：DJ 控制台（主要功能）

## 快速开始

### 前置条件

- Flutter SDK 3.22.x
- Android Studio / VS Code
- 真实设备或模拟器

### 运行项目

```bash
# 安装依赖
flutter pub get

# 运行
flutter run

# 构建 APK
flutter build apk --debug
```

### 开发与测试

#### 1. 使用 Mock 服务器

在没有真实硬件时，可以使用模拟服务器测试：

```bash
# 启动模拟 RK3588 服务器
python scripts/mock_rk3588_server.py
```

在 App 中手动添加设备：
- 地址：`http://localhost:9000` 或 `http://<你的电脑IP>:9000`
- 配对码：`123456`

#### 2. 连接真实硬件

确保以下服务在运行：
- Jetson（通过 Tailscale 访问）
- RK3588（同一局域网访问）

然后在 App 中：
1. 输入真实账号登录
2. 搜索或手动添加 RK3588
3. 完成配对流程
4. 开始使用

## 项目结构

```
lib/
├── core/
│   ├── api/
│   │   ├── jetson_client.dart      # Jetson 后端客户端
│   │   └── rk_client.dart          # RK3588 客户端
│   ├── services/
│   │   └── hardware_service.dart   # 硬件服务（设备发现）
│   └── ...
├── models/
│   ├── p1_song_status.dart         # P1 歌曲分析状态
│   ├── p2_mix_plan.dart            # P2 混音计划
│   ├── p3_asset_manifest.dart      # P3 资产清单
│   ├── p5_playback_state.dart      # P5 播放状态
│   └── p8_device_info.dart         # P8 设备信息
├── pages/
│   ├── live_page.dart              # 现场控制页
│   ├── login_page.dart             # 登录页
│   └── ...
├── presentation/
│   └── pages/
│       └── device_connection_page.dart  # 设备连接页
├── state/
│   └── providers.dart              # Riverpod 状态管理
└── main.dart                       # 应用入口

scripts/
└── mock_rk3588_server.py           # RK3588 模拟服务器

docs/                                # 项目文档
├── cypher-feature-flows.md
├── jetson-handoff-for-app.md
├── jetson-handoff-for-rk3588.md
├── team-jetson-backend.md
├── team-mobile-app.md
└── team-rk3588-edge.md
```

## 文档

详细文档请参考 `docs/` 文件夹：

- [HANDOVER.md](HANDOVER.md) - 项目交接文档
- [RK3588_API_SPEC.md](RK3588_API_SPEC.md) - RK3588 API 规范
- [docs/cypher-feature-flows.md](docs/cypher-feature-flows.md) - 功能流程
- [docs/jetson-handoff-for-app.md](docs/jetson-handoff-for-app.md) - Jetson 接口文档
- [docs/jetson-handoff-for-rk3588.md](docs/jetson-handoff-for-rk3588.md) - RK3588 接口文档
- [docs/team-jetson-backend.md](docs/team-jetson-backend.md) - Jetson 开发文档
- [docs/team-mobile-app.md](docs/team-mobile-app.md) - App 开发文档
- [docs/team-rk3588-edge.md](docs/team-rk3588-edge.md) - RK3588 开发文档

## CI/CD

项目使用 GitHub Actions 自动构建 Android APK。提交代码到 `main` 分支或手动触发工作流即可。

构建配置：`.github/workflows/build.yml`

## 团队

- **App 开发**：负责人 C
- **Jetson 后端**：负责人 A
- **RK3588 现场盒**：负责人 B

## 许可证

（待补充）
