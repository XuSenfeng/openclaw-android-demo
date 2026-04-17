# OpenClaw Python 虚拟平台集成指南

本项目用于把 Python 虚拟消息平台接入 OpenClaw，并支持 Android 客户端进行实时聊天和语音交互。

## 开源仓库映射

本项目拆分为以下 4 个仓库：
1. OpenClaw Fork（插件改动）：https://github.com/XuSenfeng/openclaw
2. Python 服务端仓库：https://github.com/XuSenfeng/open-claw-python-channel-server
3. Android 客户端仓库：https://github.com/XuSenfeng/openclaw-channel-android-client
4. 整合说明仓库（当前仓库）：https://github.com/XuSenfeng/openclaw-android-demo

各仓库职责：
1. `openclaw`：承载 `python-platform` 插件及与官方 OpenClaw 的差异改动。
2. `python-virtual-platform`：承载 Python 虚拟消息平台服务与 CLI 模拟客户端。
3. `android-client`：承载 Android App 代码、构建配置和移动端交互逻辑。
4. 当前整合仓库：承载文档、接入说明、部署说明与多仓协作指引。

## AI 生成声明

本项目文档与部分代码在开发过程中使用了 AI 辅助生成与修改（包括但不限于方案草拟、代码补全、重构建议与文档整理）。
所有 AI 产出内容均没有经过人工审查、调试与验证后再提交。

核心代码位置：
1. Python 虚拟平台服务端：[python-virtual-platform/server.py](python-virtual-platform/server.py)
2. Python 命令行模拟客户端：[python-virtual-platform/simulate.py](python-virtual-platform/simulate.py)
3. OpenClaw 平台插件：[openclaw/extensions/python-platform/index.ts](openclaw/extensions/python-platform/index.ts)
4. 插件元信息（插件 ID、配置项）：[openclaw/extensions/python-platform/openclaw.plugin.json](openclaw/extensions/python-platform/openclaw.plugin.json)
5. Android 客户端工程：[android-client/app/build.gradle.kts](android-client/app/build.gradle.kts)

## 1. 新手先决条件

建议在 macOS 上按以下版本准备环境：
1. Node.js 22.16+（建议 24）
2. pnpm（用于 OpenClaw 源码运行）
3. Python 3.10+
4. Android Studio（SDK 34、JDK 17）
5. adb（可选，用于命令行安装 APK）

快速检查：
```bash
node -v
pnpm -v
python3 -V
adb version
```

## 2. 一图理解架构

```text
Android App / simulate.py
  |
  | WebSocket (8765)
  v
python-virtual-platform/server.py
  |
  | WebSocket (8765)
  v
OpenClaw python-platform plugin
  |
  | 内部调度
  v
OpenClaw Gateway (18888)
  |
  v
LLM Provider
```

## 3. 第一次启动（本机开发）

请按顺序开 3 个终端。

### 3.1 启动 Python 虚拟平台服务

```bash
cd python-virtual-platform

# 推荐使用虚拟环境
python3 -m venv .venv
source .venv/bin/activate

pip install -r requirements.txt
python server.py
```

服务默认监听 `0.0.0.0:8765`。

### 3.2 安装并配置 OpenClaw（含插件）

```bash
cd openclaw
pnpm install
```

#### A. 先确认 OpenClaw 配置文件位置

```bash
pnpm openclaw config file
```

#### B. 配置模型（必须）

你可以用向导：
```bash
pnpm openclaw config --section model
```

或者按你自己的 provider 方式用 `config set` 配置。

#### C. 安装 python-platform 插件

当前仓库里插件目录是 [openclaw/extensions/python-platform](openclaw/extensions/python-platform)。

```bash
cd openclaw
pnpm openclaw plugins install ./extensions/python-platform --force
pnpm openclaw plugins enable python-platform
pnpm openclaw plugins list --enabled
```

#### D. 配置插件连接地址

插件配置项来自 [openclaw/extensions/python-platform/openclaw.plugin.json](openclaw/extensions/python-platform/openclaw.plugin.json)。

```bash
cd openclaw
pnpm openclaw config set channels.python-platform.wsUrl "ws://127.0.0.1:8765"
pnpm openclaw config set channels.python-platform.enabled true --strict-json
pnpm openclaw config validate
```

说明：
1. `channels.python-platform.wsUrl` 配置优先级高于环境变量 `PYTHON_PLATFORM_WS_URL`。
2. 若你不写配置，也可只用环境变量临时覆盖。

#### E. 启动网关

```bash
cd openclaw
PYTHON_PLATFORM_WS_URL=ws://127.0.0.1:8765 pnpm openclaw gateway run --port 18888 --force --verbose
```

### 3.3 启动命令行模拟客户端

```bash
cd python-virtual-platform
source .venv/bin/activate
python simulate.py
```

输入文本后，服务端会把消息广播给 OpenClaw 插件并返回 AI 结果。

## 4. Android 手机端编译与安装

Android 配置要点见 [android-client/app/build.gradle.kts](android-client/app/build.gradle.kts)：
1. `compileSdk = 34`
2. `minSdk = 24`
3. Java/Kotlin 目标版本为 17

### 4.1 编译 Debug 包

```bash
cd android-client
./gradlew assembleDebug
```

输出 APK：
`android-client/app/build/outputs/apk/debug/app-debug.apk`

### 4.2 安装到手机（USB 调试）

```bash
adb install -r android-client/app/build/outputs/apk/debug/app-debug.apk
```

### 4.3 手机连接地址怎么填

1. 手机和电脑在同一局域网
2. App 里的 WS 地址填电脑局域网 IP，比如：`ws://192.168.1.20:8765`
3. 不要填 `127.0.0.1`（那会指向手机自身）

## 5. openclaw config / 插件配置常用命令

### 5.1 查看当前配置

```bash
cd openclaw
pnpm openclaw config get channels.python-platform
```

### 5.2 修改插件 wsUrl

```bash
cd openclaw
pnpm openclaw config set channels.python-platform.wsUrl "ws://10.0.0.5:8765"
pnpm openclaw config validate
```

### 5.3 查看插件状态

```bash
cd openclaw
pnpm openclaw plugins inspect python-platform
pnpm openclaw channels status
```

## 6. 服务器部署建议（生产/长期运行）

### 6.1 最简单方案：同机部署

在同一台 Linux 服务器跑两个长期进程：
1. `python server.py`（8765）
2. `openclaw gateway run`（18888）

建议：
1. 用 systemd/pm2/supervisor 托管进程
2. `8765` 仅内网开放
3. `18888` 如需公网，务必加鉴权和反向代理

### 6.2 OpenClaw Docker 化（可选）

仓库已提供 [openclaw/docker-compose.yml](openclaw/docker-compose.yml)，可用于网关容器化。

典型思路：
1. Python 虚拟平台服务独立运行（宿主机或单独容器）
2. OpenClaw Gateway 用 compose 运行
3. 在 OpenClaw 配置里把 `channels.python-platform.wsUrl` 指向 Python 服务内网地址

### 6.3 公网部署最小安全清单

1. 网关不要裸露在公网无认证访问
2. 反向代理层启用 HTTPS
3. 控制来源 IP（至少限制管理入口）
4. 使用强随机 token/password
5. 定期 `openclaw doctor` 和日志巡检

## 7. 常见问题排查

### 7.1 插件没连上 Python 服务

检查：
1. Python 服务是否在 8765 端口监听
2. `channels.python-platform.wsUrl` 是否正确
3. 网关日志是否有 `Connected to server` 或重连错误

快速命令：
```bash
lsof -i :8765
lsof -i :18888
```

### 7.2 手机连不上

1. 手机与电脑不在同一网段
2. 手机填了 `127.0.0.1`
3. 电脑防火墙未放行 8765

### 7.3 网关能启动但不回复

1. 模型 provider 未配置或凭据无效
2. 配置文件有错误（先跑 `openclaw config validate`）
3. 插件未启用（`plugins list --enabled` 查看）

## 8. 推荐启动顺序（避免踩坑）

1. 先启 Python 服务（8765）
2. 再启 OpenClaw Gateway（18888）
3. 最后启手机 App 或 `simulate.py`

这样最稳定，也最容易定位问题。
