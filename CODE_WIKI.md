# StreamCap Code Wiki

## 目录

1. [项目概述](#1-项目概述)
2. [整体架构](#2-整体架构)
3. [核心模块详解](#3-核心模块详解)
4. [关键类与函数](#4-关键类与函数)
5. [数据模型](#5-数据模型)
6. [依赖关系](#6-依赖关系)
7. [配置管理](#7-配置管理)
8. [运行方式](#8-运行方式)
9. [开发指南](#9-开发指南)

---

## 1. 项目概述

### 1.1 项目简介

**StreamCap** 是一个基于 FFmpeg 和 StreamGet 的多平台直播流录制客户端，支持 40+ 国内外主流直播平台，提供批量录制、循环监控、定时监控和自动转码等功能。

- **项目名称**: StreamCap
- **版本**: 1.0.3
- **许可证**: Apache-2.0
- **开发语言**: Python 3.10+
- **UI 框架**: Flet 0.85.3
- **核心依赖**: FFmpeg, streamget

### 1.2 主要功能

- **多端支持**: Windows / macOS / Linux / Web
- **循环监控**: 实时监控直播间状态，开播即录
- **定时任务**: 按设定时间范围检查直播间状态
- **多种输出格式**: ts、flv、mkv、mov、mp4、mp3、m4a 等
- **自动转码**: 录制完成后自动转码为 mp4 格式
- **消息推送**: 支持直播状态推送（钉钉、微信、飞书、Bark、Telegram 等）

### 1.3 支持平台

**国内平台（30+）**: 抖音、快手、虎牙、斗鱼、B站、小红书、YY、映客、Acfun 等

**海外平台（10+）**: TikTok、Twitch、PandTV、Soop、Youtube、Bigo 等

---

## 2. 整体架构

### 2.1 架构分层

```
┌─────────────────────────────────────────────────────────┐
│                     UI 层 (Flet)                         │
│  ┌─────────┐  ┌──────────┐  ┌────────┐  ┌──────────┐   │
│  │  Home   │  │ Recordings│  │Settings│  │ Storage  │   │
│  └─────────┘  └──────────┘  └────────┘  └──────────┘   │
└─────────────────────────┬───────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────┐
│                   应用管理层 (App)                        │
│  AppManager / TrayManager / Lifecycle / Auth            │
└─────────────────────────┬───────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────┐
│                  后端服务层 (BackendServices)            │
│  RecordingManager / ConfigManager / ProcessManager      │
│  LanguageManager / SettingsConfig                       │
└─────────────────────────┬───────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────┐
│                    核心业务层                              │
│  ┌──────────────┐  ┌────────────┐  ┌─────────────────┐ │
│  │ 直播流录制    │  │ 平台适配    │  │ 消息推送服务     │ │
│  │ LiveStream   │  │ Platform   │  │ MessagePusher   │ │
│  │ Recorder     │  │ Handlers   │  │                 │ │
│  └──────────────┘  └────────────┘  └─────────────────┘ │
│  ┌──────────────┐  ┌────────────┐                       │
│  │ FFmpeg 构建器 │  │ 直接下载器  │                       │
│  │ FFmpegBuilder│  │ Direct     │                       │
│  │              │  │ Downloader │                       │
│  └──────────────┘  └────────────┘                       │
└─────────────────────────────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────┐
│                   数据持久化层                             │
│  JSON 配置文件 / 录制任务存储 / 日志文件                  │
└─────────────────────────────────────────────────────────┘
```

### 2.2 目录结构

```
/workspace/
├── app/                          # 应用主代码
│   ├── api/                      # API 服务
│   │   └── video_stream_service.py
│   ├── auth/                     # 认证管理
│   │   └── auth_manager.py
│   ├── core/                     # 核心业务逻辑
│   │   ├── config/               # 配置管理
│   │   │   ├── config_manager.py
│   │   │   ├── language_manager.py
│   │   │   └── settings_config.py
│   │   ├── media/                # 媒体处理
│   │   │   ├── ffmpeg_builders/  # FFmpeg 命令构建器
│   │   │   │   ├── audio/        # 音频格式构建器
│   │   │   │   └── video/        # 视频格式构建器
│   │   │   └── direct_downloader.py
│   │   ├── platforms/            # 平台适配器
│   │   │   └── platform_handlers/
│   │   ├── recording/            # 录制管理
│   │   │   ├── record_manager.py
│   │   │   └── stream_manager.py
│   │   ├── runtime/              # 运行时服务
│   │   │   ├── backend_services.py
│   │   │   ├── bundled_env.py
│   │   │   ├── paths.py
│   │   │   └── process_manager.py
│   │   └── update/               # 更新检查
│   ├── initialization/           # 初始化管理
│   ├── lifecycle/                # 生命周期管理
│   │   ├── app_close_handler.py
│   │   └── tray_manager.py
│   ├── messages/                 # 消息推送
│   │   ├── desktop_notify.py
│   │   ├── message_pusher.py
│   │   └── notification_service.py
│   ├── models/                   # 数据模型
│   │   ├── media/
│   │   └── recording/
│   ├── scripts/                  # 安装脚本
│   ├── ui/                       # 用户界面
│   │   ├── components/           # UI 组件
│   │   ├── filters/              # 过滤器
│   │   ├── layout/               # 布局
│   │   ├── navigation/           # 导航
│   │   ├── themes/               # 主题
│   │   └── views/                # 页面视图
│   ├── utils/                    # 工具函数
│   │   ├── logger.py
│   │   └── utils.py
│   └── app_manager.py            # 应用主管理器
├── assets/                       # 静态资源
├── config/                       # 默认配置文件
├── docs/                         # 文档
├── locales/                      # 国际化文件
├── scripts/                      # 构建脚本
├── main.py                       # 应用入口
├── pyproject.toml                # 项目配置
└── Dockerfile                    # Docker 配置
```

---

## 3. 核心模块详解

### 3.1 应用入口模块

**文件**: [main.py](file:///workspace/main.py)

应用的启动入口，负责：
- 解析命令行参数（`--web`, `--host`, `--port`）
- 加载环境变量（`.env` 文件）
- 初始化后端服务 `BackendServices`
- 根据运行模式（桌面/Web）启动 Flet 应用
- 设置窗口属性、路由、事件处理器

**关键函数**:

| 函数名 | 说明 |
|--------|------|
| `main(page: ft.Page)` | Flet 应用主入口函数，初始化 App 实例并加载 UI |
| `setup_window(page, app)` | 设置窗口大小、位置、图标等 |
| `handle_route_change(page, app)` | 路由变化处理器 |
| `handle_window_event(page, app, save_progress_overlay)` | 窗口事件处理器（关闭等） |

### 3.2 应用管理器 (App)

**文件**: [app/app_manager.py](file:///workspace/app/app_manager.py)

`App` 类是整个应用的核心管理类，每个 Flet session 对应一个 App 实例。

**主要职责**:
- 管理所有 UI 页面（home, recordings, settings, storage, about）
- 协调后端服务与 UI 的交互
- 管理录制卡片的显示与更新
- 处理页面切换和路由
- 应用更新检查

**核心属性**:

| 属性 | 类型 | 说明 |
|------|------|------|
| `page` | `ft.Page` | Flet 页面对象 |
| `services` | `BackendServices` | 后端服务单例 |
| `config_manager` | `ConfigManager` | 配置管理器 |
| `record_manager` | `RecordingManager` | 录制管理器 |
| `pages` | `dict` | 页面对象字典 |
| `record_card_manager` | `RecordingCardManager` | 录制卡片管理器 |

**核心方法**:

| 方法 | 说明 |
|------|------|
| `switch_page(page_name)` | 切换到指定页面 |
| `start_periodic_tasks()` | 启动周期性直播检查任务 |
| `schedule_card_update(recording)` | 调度卡片更新（线程安全） |
| `schedule_snack(text, **kw)` | 调度提示条显示 |

### 3.3 后端服务 (BackendServices)

**文件**: [app/core/runtime/backend_services.py](file:///workspace/app/core/runtime/backend_services.py)

`BackendServices` 是全局单例，提供跨 session 的后端服务。

**设计模式**: 单例模式 + 桥接模式

**主要职责**:
- 管理全局配置、录制、进程等服务
- 作为后端线程与 UI 线程之间的桥梁
- 维护后台事件循环（Web 模式）
- 广播 UI 更新事件到所有连接的 session

**核心属性**:

| 属性 | 类型 | 说明 |
|------|------|------|
| `config_manager` | `ConfigManager` | 配置管理器 |
| `settings_config` | `SettingsConfig` | 设置配置 |
| `language_manager` | `LanguageManager` | 语言管理器 |
| `recording_manager` | `RecordingManager` | 录制管理器 |
| `process_manager` | `AsyncProcessManager` | 进程管理器 |
| `recording_enabled` | `bool` | 录制功能是否启用 |

**核心方法**:

| 方法 | 说明 |
|------|------|
| `bootstrap(run_path)` | 初始化后端服务单例 |
| `start_background_loop()` | 启动后台异步事件循环 |
| `run_coro(coro)` | 在后台循环中运行协程 |
| `register_ui_bridge(bridge)` | 注册 UI 桥接 |
| `broadcast_card_update(recording)` | 广播卡片更新到所有 UI |
| `broadcast_snack(text, **kw)` | 广播提示条 |

**UI 桥接协议 (UIBridge)**:

后端服务通过 `UIBridge` 协议与 UI 通信，所有 UI session 都实现此接口：

- `schedule_card_update(recording)` - 更新卡片
- `schedule_card_remove(recordings)` - 移除卡片
- `schedule_snack(text, **kw)` - 显示提示
- `schedule_pubsub(topic, payload)` - 发布订阅消息

### 3.4 录制管理器 (RecordingManager)

**文件**: [app/core/recording/record_manager.py](file:///workspace/app/core/recording/record_manager.py)

负责管理所有录制任务的核心类。

**主要职责**:
- 管理录制任务的增删改查
- 控制录制的启动/停止
- 周期性检查直播状态
- 磁盘空间检查
- 批量操作支持

**核心属性**:

| 属性 | 类型 | 说明 |
|------|------|------|
| `recordings` | `list[Recording]` | 全局录制列表（共享状态） |
| `active_recorders` | `dict` | 活跃的录制器字典 |
| `platform_semaphores` | `defaultdict` | 各平台并发控制信号量 |
| `loop_time_seconds` | `int` | 循环检查间隔（秒） |

**核心方法**:

| 方法 | 说明 |
|------|------|
| `add_recording(recording)` | 添加录制任务 |
| `remove_recording(recording)` | 移除录制任务 |
| `start_monitor_recording(recording)` | 开始监控单个直播 |
| `stop_monitor_recording(recording)` | 停止监控单个直播 |
| `check_if_live(recording)` | 检查直播是否在线 |
| `check_all_live_status()` | 检查所有监控中的直播状态 |
| `setup_periodic_live_check(interval)` | 设置周期性直播检查 |
| `check_free_space(output_dir)` | 检查磁盘剩余空间 |

**全局状态 (GlobalRecordingState)**:

使用 `GlobalRecordingState` 类维护全局共享的录制列表，通过线程锁保证并发安全。

### 3.5 直播流录制器 (LiveStreamRecorder)

**文件**: [app/core/recording/stream_manager.py](file:///workspace/app/core/recording/stream_manager.py)

负责单个直播流的录制工作。

**主要职责**:
- 获取直播流信息
- 构建 FFmpeg 命令或使用直接下载器
- 管理录制进程的生命周期
- 处理录制完成后的转码和脚本执行

**核心属性**:

| 属性 | 类型 | 说明 |
|------|------|------|
| `recording` | `Recording` | 关联的录制任务对象 |
| `recording_info` | `dict` | 录制配置信息 |
| `should_stop` | `bool` | 是否请求停止 |
| `direct_downloader` | `DirectStreamDownloader` | 直接下载器（FLV 用） |

**核心方法**:

| 方法 | 说明 |
|------|------|
| `fetch_stream()` | 获取直播流信息 |
| `start_recording(stream_info)` | 开始录制 |
| `start_ffmpeg(...)` | 使用 FFmpeg 进行录制 |
| `start_direct_download(...)` | 使用直接下载器录制 |
| `converts_mp4(file_path, delete_original)` | 转码为 MP4 |
| `custom_script_execute(...)` | 执行自定义脚本 |
| `request_stop()` | 请求停止录制 |

**录制流程**:

```
1. fetch_stream() → 获取流信息
2. 判断格式 → FFmpeg 或 直接下载
3. 构建命令 / 初始化下载器
4. 启动录制进程
5. 监控进程状态
6. 录制结束 → 转码/脚本执行
7. 重新检查直播状态
```

### 3.6 平台适配器 (Platform Handlers)

**文件**: [app/core/platforms/platform_handlers/](file:///workspace/app/core/platforms/platform_handlers/)

使用**注册模式**和**单例模式**管理各直播平台的适配器。

**基类**: `PlatformHandler` ([base.py](file:///workspace/app/core/platforms/platform_handlers/base.py))

**核心机制**:

```python
@classmethod
def register(cls, *patterns: str):
    """用 URL 正则模式注册处理器类"""

@classmethod
def get_handler_instance(cls, live_url, ...):
    """根据 URL 获取或创建处理器实例"""
```

**已支持平台（50+）**:

| 平台类 | 平台名称 | 键值 |
|--------|----------|------|
| `DouyinHandler` | 抖音直播 | douyin |
| `TikTokHandler` | TikTok | tiktok |
| `KuaishouHandler` | 快手直播 | kuaishou |
| `HuyaHandler` | 虎牙直播 | huya |
| `DouyuHandler` | 斗鱼直播 | douyu |
| `BilibiliHandler` | B站直播 | bilibili |
| `YoutubeHandler` | Youtube | youtube |
| `TwitchHandler` | TwitchTV | twitch |
| `CustomHandler` | 自定义流 | custom |
| ... | ... | ... |

**入口函数**:

- `get_platform_handler(live_url, ...)` - 获取平台处理器实例
- `get_platform_info(record_url)` - 获取平台名称和键值

### 3.7 FFmpeg 命令构建器

**文件**: [app/core/media/ffmpeg_builders/](file:///workspace/app/core/media/ffmpeg_builders/)

使用**建造者模式**为不同媒体格式构建 FFmpeg 命令。

**基类**: `FFmpegCommandBuilder` ([base.py](file:///workspace/app/core/media/ffmpeg_builders/base.py))

**支持的视频格式**:
- MP4 (`MP4CommandBuilder`)
- MKV (`MKVCommandBuilder`)
- TS (`TSCommandBuilder`)
- FLV (`FLVCommandBuilder`)
- MOV (`MOVCommandBuilder`)
- NUT (`NUTCommandBuilder`)

**支持的音频格式**:
- MP3 (`MP3CommandBuilder`)
- M4A (`M4ACommandBuilder`)
- WAV (`WAVCommandBuilder`)
- AAC (`AACCommandBuilder`)
- WMA (`WMACommandBuilder`)

**工厂函数**: `create_builder(format_type, *args, **kwargs)`

**基础 FFmpeg 参数配置**:

| 参数 | 国内默认值 | 海外默认值 | 说明 |
|------|----------|----------|------|
| `rw_timeout` | 15000000 | 50000000 | 读写超时（微秒） |
| `analyzeduration` | 20000000 | 40000000 | 分析时长（微秒） |
| `probesize` | 10000000 | 20000000 | 探测大小 |
| `bufsize` | 8000k | 15000k | 缓冲区大小 |

### 3.8 直接流下载器 (DirectStreamDownloader)

**文件**: [app/core/media/direct_downloader.py](file:///workspace/app/core/media/direct_downloader.py)

使用 HTTP 流式下载直接保存直播流，用于 FFmpeg 无法正常处理的 FLV 流。

**核心特性**:
- 基于 httpx 异步 HTTP 客户端
- 流式下载，16KB 分块写入
- 支持代理和自定义请求头
- 可中断的停止机制

**核心方法**:
- `start_download()` - 开始下载
- `stop_download()` - 停止下载

### 3.9 配置管理器 (ConfigManager)

**文件**: [app/core/config/config_manager.py](file:///workspace/app/core/config/config_manager.py)

负责所有配置文件的读写管理。

**配置文件列表**:

| 配置文件 | 路径 | 说明 |
|----------|------|------|
| 默认设置 | `config/default_settings.json` | 默认配置 |
| 用户设置 | `config/user_settings.json` | 用户自定义配置 |
| Cookies | `config/cookies.json` | 各平台 cookies |
| 账号 | `config/accounts.json` | 各平台账号信息 |
| 录制任务 | `config/recordings.json` | 录制任务列表 |
| 语言配置 | `config/language.json` | 语言选项 |
| 版本信息 | `config/version.json` | 版本信息 |
| Web 认证 | `config/web_auth.json` | Web 模式认证 |

**核心方法**:

| 方法 | 说明 |
|------|------|
| `init()` | 初始化所有配置文件 |
| `load_*_config()` | 加载各类配置 |
| `save_*_config(config)` | 异步保存配置（线程安全） |

**写入特性**:
- 原子写入（临时文件 + rename）
- 线程安全（`_write_lock`）
- 异步写入（`asyncio.to_thread`）

### 3.10 语言管理器 (LanguageManager)

**文件**: [app/core/config/language_manager.py](file:///workspace/app/core/config/language_manager.py)

国际化（i18n）管理。

**主要功能**:
- 加载语言文件（`locales/*.json`）
- 观察者模式通知语言变更
- 支持无头模式（headless）创建

**语言文件**:
- `locales/zh_CN.json` - 简体中文
- `locales/en.json` - 英语

### 3.11 消息推送 (MessagePusher)

**文件**: [app/messages/message_pusher.py](file:///workspace/app/messages/message_pusher.py)

直播状态消息推送服务。

**支持的推送渠道**:

| 渠道 | 配置键 | 说明 |
|------|--------|------|
| 钉钉 | `dingtalk_enabled` | 钉钉机器人 Webhook |
| 微信 | `wechat_enabled` | 企业微信 Webhook |
| 飞书 | `feishu_enabled` | 飞书机器人 Webhook |
| Bark | `bark_enabled` | iOS Bark 推送 |
| Ntfy | `ntfy_enabled` | Ntfy 通知服务 |
| Telegram | `telegram_enabled` | Telegram Bot |
| Email | `email_enabled` | SMTP 邮件 |
| Server酱 | `serverchan_enabled` | Server 酱 Turbo |

**核心方法**:
- `push_messages(msg_title, push_content)` - 推送到所有已启用渠道
- `should_push_message(settings, recording, ...)` - 判断是否应该推送

### 3.12 进程管理器

**文件**: [app/core/runtime/process_manager.py](file:///workspace/app/core/runtime/process_manager.py)

包含两个类：

**BackgroundService** - 后台任务服务
- 单例模式
- 基于线程的任务队列
- 用于应用关闭后的转码等后台任务

**AsyncProcessManager** - 异步进程管理器
- 管理所有 FFmpeg 子进程
- `cleanup()` - 清理所有进程（优雅关闭）

### 3.13 UI 层

**文件目录**: [app/ui/](file:///workspace/app/ui/)

基于 Flet 框架构建的响应式 UI。

**页面视图 (Views)**:

| 页面 | 文件 | 说明 |
|------|------|------|
| Home | [home_view.py](file:///workspace/app/ui/views/home_view.py) | 首页/录制列表 |
| Recordings | [recordings_view.py](file:///workspace/app/ui/views/recordings_view.py) | 录制管理 |
| Settings | [settings_view.py](file:///workspace/app/ui/views/settings_view.py) | 设置页面 |
| Storage | [storage_view.py](file:///workspace/app/ui/views/storage_view.py) | 存储管理 |
| About | [about_view.py](file:///workspace/app/ui/views/about_view.py) | 关于页面 |
| Login | [login_view.py](file:///workspace/app/ui/views/login_view.py) | 登录页面（Web） |

**核心组件**:

| 组件 | 说明 |
|------|------|
| `RecordingCard` | 录制卡片组件 |
| `RecordingDialog` | 录制对话框 |
| `VideoPlayer` | 视频播放器 |
| `LeftNavigationMenu` | 左侧导航菜单 |
| `NavigationSidebar` | 导航侧边栏 |
| `ShowSnackBar` | 提示条组件 |
| `SaveProgressOverlay` | 保存进度遮罩 |

---

## 4. 关键类与函数

### 4.1 App 类

**位置**: [app/app_manager.py](file:///workspace/app/app_manager.py#L22-L220)

应用主类，每个 Flet session 一个实例。

```python
class App:
    def __init__(self, page: ft.Page, services: BackendServices | None = None)
```

**关键方法详解**:

**`switch_page(page_name)`**
- 异步切换页面
- 防止并发加载（`_loading_page` 锁）
- 切换前检查设置变更

**`schedule_card_update(recording)`**
- 从后端线程安全地调度 UI 更新
- 使用 `asyncio.run_coroutine_threadsafe`
- 自动处理 session 已断开的情况

### 4.2 BackendServices 类

**位置**: [app/core/runtime/backend_services.py](file:///workspace/app/core/runtime/backend_services.py#L32-L193)

全局后端服务单例。

```python
class BackendServices:
    _instance: BackendServices | None = None

    @classmethod
    def bootstrap(cls, run_path: str) -> BackendServices
```

**关键方法详解**:

**`run_coro(coro)`**
- 优先投递到后台事件循环（Web 模式）
- 否则在当前循环中创建任务
- 自动处理无循环的情况

**`broadcast_*(...)`**
- 遍历所有 UI 桥接
- 每个桥接独立调用，异常不影响其他
- 用于多 session 场景下的状态同步

### 4.3 RecordingManager 类

**位置**: [app/core/recording/record_manager.py](file:///workspace/app/core/recording/record_manager.py#L21-L530)

录制任务管理核心。

**关键方法详解**:

**`check_if_live(recording)`**
```
流程:
1. 检查录制状态（是否正在录制/停止中）
2. 检查监控状态
3. 处理定时录制时间范围
4. 获取平台处理器
5. 获取流信息
6. 根据直播状态更新UI
7. 如果在线且需要录制 → 启动录制
8. 如果离线 → 更新监控状态
```

**`setup_periodic_live_check(interval)`**
- 全局单例检查任务（类级别 `_periodic_task_running`）
- 周期性检查磁盘空间和直播状态
- 可配置启动时是否立即检查

### 4.4 LiveStreamRecorder 类

**位置**: [app/core/recording/stream_manager.py](file:///workspace/app/core/recording/stream_manager.py#L24-L795)

单个直播流录制器。

**关键方法详解**:

**`start_ffmpeg(...)`**
```
流程:
1. 创建 FFmpeg 子进程
2. 注册到进程管理器
3. 进入监控循环
4. 检测停止信号 → 优雅停止（q 命令 / SIGINT）
5. 超时强制终止
6. 处理返回码，判断是否成功
7. 自动转码
8. 执行自定义脚本
9. 重新检查直播状态
```

**`_select_source_url(stream_info)`**
- 抖音/TikTok 优先使用 FLV 源
- H265 编码回退到 HLS
- 可配置强制使用 HLS

### 4.5 PlatformHandler 基类

**位置**: [app/core/platforms/platform_handlers/base.py](file:///workspace/app/core/platforms/platform_handlers/base.py#L13-L124)

平台处理器抽象基类。

**设计模式**: 注册模式 + 单例模式

```python
@PlatformHandler.register(r"douyin\.com/")
class DouyinHandler(PlatformHandler):
    async def get_stream_info(self, live_url: str) -> StreamData:
        ...
```

**实例缓存**:
- 以 (proxy, cookies, quality, platform, username, password, account_type) 为键
- 相同配置复用同一实例
- 线程安全的创建机制

### 4.6 Recording 数据类

**位置**: [app/models/recording/recording_model.py](file:///workspace/app/models/recording/recording_model.py#L4-L147)

录制任务数据模型。

**持久化字段** (`to_dict()` 包含):
- `rec_id` - 唯一标识
- `url` - 直播地址
- `streamer_name` - 主播名称
- `record_format` - 录制格式
- `quality` - 画质
- `segment_record` - 是否分段
- `segment_time` - 分段时长
- `monitor_status` - 监控状态
- `scheduled_recording` - 是否定时
- `scheduled_start_time` - 定时开始时间
- `monitor_hours` - 监控时长
- `recording_dir` - 保存目录
- `enabled_message_push` - 消息推送
- `only_notify_no_record` - 仅通知不录制
- `flv_use_direct_download` - FLV 直接下载
- `platform` - 平台名称
- `platform_key` - 平台键

**运行时字段** (不持久化):
- `is_live`, `is_recording`, `is_checking` - 状态标志
- `start_time` - 录制开始时间
- `cumulative_duration` - 累计录制时长
- `status_info` - 状态信息
- `preview_url` - 预览地址
- 等等...

---

## 5. 数据模型

### 5.1 录制模型

#### RecordingStatus

**文件**: [app/models/recording/recording_status_model.py](file:///workspace/app/models/recording/recording_status_model.py)

录制状态常量类:

| 状态 | 说明 |
|------|------|
| `STOPPED_MONITORING` | 已停止监控 |
| `MONITORING` | 监控中 |
| `RECORDING` | 录制中 |
| `NOT_RECORDING` | 未录制 |
| `STATUS_CHECKING` | 检查中 |
| `NOT_IN_SCHEDULED_CHECK` | 不在定时范围内 |
| `PREPARING_RECORDING` | 准备录制 |
| `RECORDING_ERROR` | 录制错误 |
| `NOT_RECORDING_SPACE` | 空间不足 |
| `LIVE_STATUS_CHECK_ERROR` | 直播状态检查错误 |
| `LIVE_BROADCASTING` | 直播中（仅通知） |

#### CardStateType

卡片状态枚举（UI 显示用）:
- `RECORDING` - 录制中
- `ERROR` - 错误
- `LIVE` - 在线
- `OFFLINE` - 离线
- `STOPPED` - 已停止
- `CHECKING` - 检查中
- `UNKNOWN` - 未知

### 5.2 媒体模型

#### VideoQuality

**文件**: [app/models/media/video_quality_model.py](file:///workspace/app/models/media/video_quality_model.py)

视频画质枚举。

#### AudioFormatModel / VideoFormatModel

媒体格式模型。

---

## 6. 依赖关系

### 6.1 核心依赖

| 依赖包 | 版本 | 用途 |
|--------|------|------|
| `flet` | 0.85.3 | UI 框架（桌面/Web） |
| `flet-video` | 0.85.3 | 视频播放组件 |
| `streamget` | >=4.0.10 | 直播流获取核心库 |
| `httpx` | >=0.28.1 | 异步 HTTP 客户端 |
| `aiofiles` | >=25.1.0 | 异步文件操作 |
| `python-dotenv` | >=1.2.2 | 环境变量加载 |
| `cachetools` | >=7.1.4 | 缓存工具 |
| `pystray` | ~0.19.5 | 系统托盘（桌面端） |
| `plyer` | ~2.1.0 | 桌面通知 |
| `screeninfo` | ~0.8.1 | 屏幕信息 |
| `deprecated` | >=1.3.1 | 弃用装饰器 |
| `loguru` | - | 日志记录 |
| `execjs` | - | JavaScript 执行 |

### 6.2 外部依赖

| 依赖 | 用途 |
|------|------|
| FFmpeg | 流媒体录制和转码 |
| Node.js | 部分平台 JS 解密执行 |

### 6.3 模块依赖图

```
main.py
  └── App (app_manager.py)
        ├── BackendServices (runtime/backend_services.py)
        │     ├── ConfigManager (config/config_manager.py)
        │     ├── SettingsConfig (config/settings_config.py)
        │     ├── LanguageManager (config/language_manager.py)
        │     ├── RecordingManager (recording/record_manager.py)
        │     │     ├── LiveStreamRecorder (recording/stream_manager.py)
        │     │     │     ├── Platform Handlers (platforms/)
        │     │     │     ├── FFmpeg Builders (media/ffmpeg_builders/)
        │     │     │     └── DirectStreamDownloader (media/)
        │     │     └── MessagePusher (messages/)
        │     └── AsyncProcessManager (runtime/process_manager.py)
        ├── UI Views (ui/views/)
        ├── UI Components (ui/components/)
        └── TrayManager (lifecycle/)
```

---

## 7. 配置管理

### 7.1 配置文件位置

所有配置文件存储在用户数据目录下的 `config/` 文件夹中。

用户数据目录位置:
- **Windows**: `%APPDATA%/StreamCap/`
- **macOS**: `~/Library/Application Support/StreamCap/`
- **Linux/源码**: 可执行文件所在目录

### 7.2 主要配置项

参考 [config/default_settings.json](file:///workspace/config/default_settings.json)

#### 基础设置

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `language` | "Chinese" | 界面语言 |
| `live_save_path` | "" | 录制保存路径 |
| `theme_mode` | "light" | 主题模式 |
| `is_grid_view` | true | 是否网格视图 |

#### 录制设置

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `video_format` | "TS" | 录制格式 |
| `record_quality` | "OD" | 录制画质 |
| `loop_time_seconds` | "180" | 循环检查间隔（秒） |
| `segmented_recording_enabled` | true | 是否分段录制 |
| `video_segment_time` | "1800" | 分段时长（秒） |
| `convert_to_mp4` | true | 自动转 MP4 |
| `delete_original` | false | 转码后删除原文件 |
| `recording_space_threshold` | "2.0" | 磁盘空间阈值（GB） |

#### 网络设置

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `enable_proxy` | true | 是否启用代理 |
| `proxy_address` | "" | 代理地址 |
| `force_https_recording` | true | 强制 HTTPS 录制 |
| `default_live_source` | "FLV" | 默认直播源 |
| `platform_max_concurrent_requests` | "3" | 单平台最大并发 |

#### 通知设置

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `system_notification_enabled` | true | 系统通知 |
| `stream_start_notification_enabled` | false | 开播通知 |
| `stream_end_notification_enabled` | false | 下播通知 |
| `dingtalk_enabled` | false | 钉钉推送 |
| `wechat_enabled` | false | 微信推送 |
| `feishu_enabled` | false | 飞书推送 |
| `bark_enabled` | false | Bark 推送 |
| `telegram_enabled` | false | Telegram 推送 |
| `email_enabled` | false | 邮件推送 |

### 7.3 环境变量 (.env)

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `PLATFORM` | - | 运行平台（web/桌面） |
| `HOST` | "127.0.0.1" | Web 模式监听地址 |
| `PORT` | "6006" | Web 模式监听端口 |
| `TZ` | "Asia/Shanghai" | 时区（Docker） |

---

## 8. 运行方式

### 8.1 从源码运行

#### 前置条件
- Python 3.10+
- FFmpeg（已配置环境变量）
- Node.js（部分平台需要）

#### 步骤

```bash
# 1. 克隆项目
git clone https://github.com/ihmily/StreamCap.git
cd StreamCap

# 2. 安装核心依赖
pip install streamget

# 3. 安装桌面端依赖
pip install -r requirements.txt

# 4. 复制环境变量配置
cp .env.example .env

# 5. 运行（桌面模式）
python main.py

# 6. 运行（Web模式）
python main.py --web
# 访问 http://127.0.0.1:6006
```

### 8.2 Docker 运行

```bash
# 使用 docker compose
docker compose up -d

# 或手动构建
docker build -t streamcap .
docker run -p 6006:6006 -v ./data:/app/config streamcap
```

### 8.3 预构建版本

从 [Releases 页面](https://github.com/ihmily/StreamCap/releases) 下载:
- **Windows**: `StreamCap.zip` → 解压运行 `StreamCap.exe`
- **macOS**: `StreamCap.dmg` → 安装后运行

### 8.4 命令行参数

```bash
python main.py [options]

选项:
  --web          Web 模式运行
  --host HOST    监听地址 (默认: 127.0.0.1)
  --port PORT    监听端口 (默认: 6006)
```

---

## 9. 开发指南

### 9.1 项目结构约定

```
app/
├── core/        # 核心业务逻辑（无 UI 依赖）
├── ui/          # UI 层（仅依赖 core）
├── models/      # 数据模型（纯数据，无业务逻辑）
├── utils/       # 工具函数（无业务逻辑）
├── messages/    # 消息推送服务
└── lifecycle/   # 应用生命周期管理
```

### 9.2 添加新平台支持

1. 在 `app/core/platforms/platform_handlers/handlers.py` 中创建新处理器类
2. 继承 `PlatformHandler` 基类
3. 使用 `@PlatformHandler.register(pattern)` 装饰器注册 URL 模式
4. 实现 `async get_stream_info(self, live_url: str) -> StreamData` 方法
5. 在 `__init__.py` 中导出新类

示例:
```python
from .base import PlatformHandler, StreamData
from streamget import ...

@PlatformHandler.register(r"example\.com/")
class ExampleHandler(PlatformHandler):
    async def get_stream_info(self, live_url: str) -> StreamData:
        # 实现获取直播流信息的逻辑
        ...
```

### 9.3 添加新的媒体格式

1. 在 `app/core/media/ffmpeg_builders/video/` 或 `audio/` 下创建新构建器
2. 继承 `FFmpegCommandBuilder` 基类
3. 实现 `build_command() -> list[str]` 方法
4. 在 `__init__.py` 的 `create_builder()` 函数中注册格式映射

### 9.4 添加新的推送渠道

1. 在 `app/messages/notification_service.py` 中添加发送方法
2. 在 `MessagePusher.push_messages()` 中添加调用逻辑
3. 在 `_get_push_channels()` 中添加配置键
4. 在默认配置文件中添加相关配置项

### 9.5 线程模型

```
UI 线程 (Flet session)
  ├── 处理用户交互
  ├── 更新 UI 组件
  └── 通过 schedule_* 方法接收后端更新

后端线程 (BackendServicesLoop) - Web 模式
  ├── 周期性直播检查
  ├── 磁盘空间检查
  └── 通过 run_coro() 执行后台任务

FFmpeg 子进程
  └── 每个录制一个独立进程

BackgroundService 线程
  └── 应用关闭后的转码/脚本任务
```

**线程安全注意事项**:
- 全局状态使用 `threading.Lock` 保护
- UI 更新必须通过 `schedule_*` 方法调度到 UI 线程
- 后端协程通过 `run_coro()` 投递到后台循环

### 9.6 日志系统

**日志库**: loguru

**日志文件**:
- `logs/streamget.log` - 主日志（DEBUG 级别，滚动 3MB，保留 3 份）
- `logs/play_url.log` - 播放地址日志（STREAM 级别）

**自定义日志级别**:
- `STREAM` (no=22) - 流地址日志

使用示例:
```python
from app.utils.logger import logger

logger.info("信息日志")
logger.error("错误日志")
logger.log("STREAM", "流地址: " + url)
```

### 9.7 代码规范

- **Linter**: Ruff
- **配置文件**: `.ruff.toml`
- **Python 版本**: >=3.10, <4.0
- **类型注解**: 推荐使用

运行 lint:
```bash
pip install ruff
ruff check .
```

---

## 附录

### A. 参考文件索引

| 类别 | 文件路径 |
|------|----------|
| 入口 | [main.py](file:///workspace/main.py) |
| 应用管理 | [app/app_manager.py](file:///workspace/app/app_manager.py) |
| 后端服务 | [app/core/runtime/backend_services.py](file:///workspace/app/core/runtime/backend_services.py) |
| 录制管理 | [app/core/recording/record_manager.py](file:///workspace/app/core/recording/record_manager.py) |
| 流录制 | [app/core/recording/stream_manager.py](file:///workspace/app/core/recording/stream_manager.py) |
| 配置管理 | [app/core/config/config_manager.py](file:///workspace/app/core/config/config_manager.py) |
| 平台适配 | [app/core/platforms/platform_handlers/](file:///workspace/app/core/platforms/platform_handlers/) |
| FFmpeg构建 | [app/core/media/ffmpeg_builders/](file:///workspace/app/core/media/ffmpeg_builders/) |
| 消息推送 | [app/messages/message_pusher.py](file:///workspace/app/messages/message_pusher.py) |
| 数据模型 | [app/models/](file:///workspace/app/models/) |
| 工具函数 | [app/utils/utils.py](file:///workspace/app/utils/utils.py) |
| 默认配置 | [config/default_settings.json](file:///workspace/config/default_settings.json) |

### B. 相关文档

- [README.md](file:///workspace/README.md) - 项目说明
- [Dockerfile](file:///workspace/Dockerfile) - Docker 构建
- [docker-compose.yml](file:///workspace/docker-compose.yml) - Docker Compose 配置
- [docs/packaging.md](file:///workspace/docs/packaging.md) - 打包指南
