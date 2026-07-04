# StreamCap Code Wiki

> 本文档为 [StreamCap](https://github.com/ihmily/StreamCap) 仓库的结构化代码 wiki，覆盖项目整体架构、模块职责、关键类与函数、依赖关系以及运行方式等核心信息。
>
> 适用版本：StreamCap `1.0.3`（核心 `streamget>=4.0.10`，UI 框架 `flet==0.85.3`）。

---

## 目录

- [1. 项目概述](#1-项目概述)
- [2. 整体架构](#2-整体架构)
- [3. 目录结构](#3-目录结构)
- [4. 主要模块职责](#4-主要模块职责)
- [5. 关键类与函数说明](#5-关键类与函数说明)
- [6. 依赖关系](#6-依赖关系)
- [7. 项目运行方式](#7-项目运行方式)
- [8. 配置文件说明](#8-配置文件说明)
- [9. 部署（Docker）](#9-部署docker)
- [10. 录制核心流程详解](#10-录制核心流程详解)

---

## 1. 项目概述

**StreamCap** 是一个基于 [FFmpeg](https://ffmpeg.org) 与 [streamget](https://github.com/ihmily/streamget) 的多平台直播流录制客户端，覆盖 40+ 国内外主流直播平台（抖音、TikTok、B站、斗鱼、虎牙、快手、Twitch、YouTube 等）。它支持：

- **多端运行**：Windows / macOS 桌面应用、Linux/Web 浏览器访问。
- **循环监控**：实时监控直播间状态，开播即录。
- **定时任务**：按时间窗口检查并录制直播。
- **多种输出格式**：视频（`ts`/`flv`/`mkv`/`mov`/`mp4`/`nut`）与音频（`mp3`/`m4a`/`wav`/`aac`/`wma`）。
- **自动转码**：录制完成后自动 `ts → mp4` 转码。
- **消息推送**：钉钉、企业微信、飞书、Bark、ntfy、Telegram、Server 酱、邮件。
- **更新检查**：GitHub Release / 自定义 API 双通道。

技术栈：Python `>=3.10`，[Flet](https://flet.dev)（Material 风格跨端 UI 框架）、`httpx`、`aiofiles`、`loguru`、`pystray`、`plyer`，以及 `streamget` 作为各直播平台解析库。构建工具 Poetry / PyInstaller；容器化 Docker / Docker Compose。

---

## 2. 整体架构

StreamCap 采用**分层 + 容器注入**架构，整体可分为 6 层：

```
┌──────────────────────────────────────────────────────────────┐
│  入口层  main.py                                             │
│  - 解析 CLI 参数 / .env                                       │
│  - 启动 BackendServices（后台单例）                            │
│  - 桌面端：setup_bundled_flet_view + ft.run                  │
│  - Web 端：services.start_background_loop + ft.run(web)      │
└─────────────┬─────────────────────────────┬──────────────────┘
              │                             │
   ┌──────────▼──────────┐     ┌────────────▼───────────────┐
   │  UI 层 (app/ui)     │     │  后端服务 BackendServices   │
   │  - views: 首页/录制 │     │  (process-wide singleton)   │
   │   列表/设置/存储/关于│◄───►│  - ConfigManager            │
   │   /登录              │     │  - SettingsConfig           │
   │  - components: 卡片/ │     │  - LanguageManager          │
   │   对话框/播放器       │     │  - AsyncProcessManager      │
   │  - navigation/themes │     │  - RecordingManager (lazy)  │
   │  - layout: 响应式     │     │  - background asyncio loop  │
   └──────────┬──────────┘     │  - UIBridge 注册表          │
              │                └────────────┬───────────────┘
              │                             │
   ┌──────────▼─────────────────────────────▼───────────────┐
   │  录制核心层 (app/core/recording)                       │
   │  - RecordingManager: 全局录制列表、循环监控、调度         │
   │  - LiveStreamRecorder: 单个录制任务、ffmpeg 子进程        │
   └──────────┬─────────────────────────────┬──────────────┘
              │                             │
   ┌──────────▼──────────┐     ┌────────────▼───────────────┐
   │  平台层 (platforms) │     │  媒体层 (media)             │
   │  - PlatformHandler  │     │  - FFmpegCommandBuilder    │
   │    抽象基类 + 注册表  │     │    (audio/video builders)  │
   │  - 50+ 平台 handler │     │  - DirectStreamDownloader  │
   │  - streamget 包装    │     │    (httpx FLV 直下)        │
   └─────────────────────┘     └────────────────────────────┘

   ┌──────────────────────────────────────────────────────────┐
   │  横切关注点                                              │
   │  - config: JSON 持久化 (user/cookies/accounts/auth/...) │
   │  - models: Recording / RecordingStatus / Format / Quality │
   │  - messages: MessagePusher + NotificationService + 桌面   │
   │  - lifecycle: 关闭确认 + 系统托盘                         │
   │  - initialization: FFmpeg / Node 自动安装                 │
   │  - update: 版本检查 (GitHub / custom)                     │
   │  - auth: web 模式登录会话                                 │
   │  - api: 独立 FastAPI 视频流服务 (端口 6007)               │
   │  - utils: logger / utils / delay                          │
   └──────────────────────────────────────────────────────────┘
```

### 关键设计原则

1. **单例后端 + 多会话 UI**：`BackendServices` 为进程级单例，持有 `RecordingManager`、后台 asyncio 循环与配置；多个 UI 会话（web 多客户端、桌面单实例）通过实现 `UIBridge` 协议注册到后端，后端用 `broadcast_*` 把录制状态变更扇出到每个会话。
2. **UI 与录制解耦**：录制逻辑跑在后端 `asyncio` 循环线程里，UI 通过 `schedule_*` 方法接收更新；web 模式下断开会话不会停止录制。
3. **观察者 + i18n**：`LanguageManager` 维护观察者列表，页面/管理器注册后在语言切换时被回调重新渲染。
4. **配置原子写入**：所有配置通过 `ConfigManager._save_config` 在线程锁 + `tempfile.mkstemp` + `os.replace` 下原子落盘。
5. **可扩展的平台 handler**：新平台只需继承 `PlatformHandler` 并用 `@Handler.register(*patterns)` 注册 URL 正则。

---

## 3. 目录结构

```
StreamCap/
├── main.py                       # 程序入口，CLI 解析 + Flet 启动
├── pyproject.toml                # Poetry 配置 & 依赖
├── requirements.txt              # 桌面端依赖
├── requirements-web.txt          # Web 端依赖
├── poetry.toml / .ruff.toml      # 构建与 lint 配置
├── Dockerfile / docker-compose.yml
├── .env.example                  # 环境变量模板
├── config/                       # 内置配置（首次运行时复制到工作目录）
│   ├── default_settings.json     # 用户设置默认值
│   ├── language.json             # 显示名 → i18n 代码映射
│   └── version.json              # 版本信息 & 历史更新
├── locales/                      # i18n 文案
│   ├── en.json
│   └── zh_CN.json
├── assets/                       # 图标、字体、图片
└── app/
    ├── __init__.py               # 暴露 execute_dir
    ├── app_manager.py            # App 类（UI 与服务的胶水层）
    ├── api/
    │   └── video_stream_service.py  # FastAPI 视频点播服务
    ├── auth/auth_manager.py      # Web 模式登录鉴权
    ├── core/
    │   ├── config/               # ConfigManager / SettingsConfig / LanguageManager
    │   ├── media/
    │   │   ├── direct_downloader.py
    │   │   └── ffmpeg_builders/
    │   │       ├── base.py
    │   │       ├── audio/{aac,m4a,mp3,wav,wma}.py
    │   │       └── video/{flv,mkv,mov,mp4,nut,ts}.py
    │   ├── platforms/platform_handlers/{base,handlers}.py
    │   ├── recording/{record_manager,stream_manager}.py
    │   ├── runtime/{backend_services,process_manager,bundled_env}.py
    │   └── update/update_checker.py
    ├── initialization/installation_manager.py
    ├── lifecycle/{app_close_handler,tray_manager}.py
    ├── messages/{message_pusher,notification_service,desktop_notify}.py
    ├── models/
    │   ├── media/{audio_format,video_format,video_quality}_model.py
    │   └── recording/{recording_model,recording_status_model}.py
    ├── scripts/{ffmpeg_install,node_install}.py
    ├── ui/
    │   ├── base_page.py
    │   ├── views/{home,recordings,settings,storage,about,login}_view.py
    │   ├── components/
    │   │   ├── business/{recording_card,recording_dialog,video_player}.py
    │   │   ├── common/{save_progress_overlay,show_snackbar}.py
    │   │   ├── dialogs/{card_dialog,help_dialog,search_dialog}.py
    │   │   └── state/recording_card_state.py
    │   ├── filters/recording_filters.py
    │   ├── layout/responsive_layout.py
    │   ├── navigation/sidebar.py
    │   └── themes/{theme,theme_manager}.py
    └── utils/{logger,utils,delay}.py
```

---

## 4. 主要模块职责

### 4.1 入口与胶水层

| 模块 | 职责 |
|---|---|
| [`main.py`](file:///workspace/main.py) | 解析 `.env` 与 CLI 参数（`--web --host --port`），启动 `BackendServices`，桌面端调用 `setup_bundled_flet_view()` 后 `ft.run`；Web 端启动后台循环并以 `WEB_BROWSER` 视图运行。同时绑定路由切换、窗口关闭、断线、resize 等事件回调。 |
| [`app/app_manager.py`](file:///workspace/app/app_manager.py) | `App` 类聚合 UI 与服务：持有所有页面（home/recordings/settings/storage/about）、`RecordingsPage` 设置对象、`RecordingCardManager`、`InstallationManager`、`UpdateChecker`，并实现 `UIBridge` 协议（`schedule_card_update/remove`、`schedule_snack`、`schedule_pubsub`）把后端事件转发到当前 Flet 会话循环。 |
| [`app/__init__.py`](file:///workspace/app/__init__.py) | 计算 `execute_dir`（脚本所在目录），用作相对路径基准。 |

### 4.2 后端服务（runtime）

| 模块 | 职责 |
|---|---|
| [`core/runtime/backend_services.py`](file:///workspace/app/core/runtime/backend_services.py) | 进程级单例 `BackendServices`：组合 `ConfigManager` / `SettingsConfig` / `LanguageManager` / `AsyncProcessManager`，懒加载 `RecordingManager`；启动独立后台 `asyncio` 循环线程；提供 `UIBridge` 协议与 `broadcast_card_update/remove/snack/pubsub` 扇出。 |
| [`core/runtime/process_manager.py`](file:///workspace/app/core/runtime/process_manager.py) | `AsyncProcessManager`：登记所有 ffmpeg 子进程，关闭时优雅退出（Windows 写 `q` 到 stdin，Unix `SIGINT`，5 秒后 `kill`）；`BackgroundService`：FIFO 队列 + 工作线程，用于一次性后台任务（推送、转码）。 |
| [`core/runtime/bundled_env.py`](file:///workspace/app/core/runtime/bundled_env.py) | `setup_bundled_flet_view()`：当被 PyInstaller 冻结时，定位 `flet_desktop/app/flet` 并设置 `FLET_VIEW_PATH`，使打包版本可找到内置 Flet 视图。 |

### 4.3 录制核心（recording）

| 模块 | 职责 |
|---|---|
| [`core/recording/record_manager.py`](file:///workspace/app/core/recording/record_manager.py) | `RecordingManager`：维护 `GlobalRecordingState.recordings` 全局列表与锁；执行 `setup_periodic_live_check(interval)` 周期监控；`check_if_live` 为单条录制解析平台、获取 `StreamData`、加并发信号量、创建 `LiveStreamRecorder` 启动录制；负责录制增删改与持久化；触发桌面通知与多渠道推送。 |
| [`core/recording/stream_manager.py`](file:///workspace/app/core/recording/stream_manager.py) | `LiveStreamRecorder`：单条录制的执行器。`fetch_stream` 调用 `platform_handlers.get_platform_handler`；`start_recording` 决定格式/路径，调用 `ffmpeg_builders.create_builder().build_command()` 启动 ffmpeg，或构造 `DirectStreamDownloader` 直下 FLV；轮询子进程状态，处理 ts→mp4 转码、自定义脚本、错误/结束通知、`recheck_live_status`。 |

### 4.4 平台层（platforms）

| 模块 | 职责 |
|---|---|
| [`core/platforms/platform_handlers/base.py`](file:///workspace/app/core/platforms/platform_handlers/base.py) | `PlatformHandler` 抽象基类：维护 `_registry`（URL 正则 → handler 类）与 `_instances`（七元组键缓存实例）；`register(*patterns)` 装饰器；`get_handler_instance` 工厂方法。 |
| [`core/platforms/platform_handlers/handlers.py`](file:///workspace/app/core/platforms/platform_handlers/handlers.py) | 50+ 具体 handler：每个封装 `streamget.<Platform>LiveStream`，实现 `get_stream_info(live_url) -> StreamData`；底部按 URL 正则注册。 |
| [`core/platforms/platform_handlers/__init__.py`](file:///workspace/app/core/platforms/platform_handlers/__init__.py) | 暴露 `get_platform_handler`、`get_platform_info` 工厂函数及全部 handler 类。 |

### 4.5 媒体层（media）

| 模块 | 职责 |
|---|---|
| [`core/media/ffmpeg_builders/base.py`](file:///workspace/app/core/media/ffmpeg_builders/base.py) | `FFmpegCommandBuilder` 抽象基类：定义 `DEFAULT_CONFIG`（国内）/`OVERSEAS_CONFIG`（海外）调参；`_get_basic_ffmpeg_command()` 构建通用 ffmpeg 前缀（rw_timeout / analyzeduration / bufsize 等），子类只需实现 `build_command()`。 |
| [`core/media/ffmpeg_builders/__init__.py`](file:///workspace/app/core/media/ffmpeg_builders/__init__.py) | `create_builder(format_type, *args, **kwargs)` 工厂：根据格式字符串实例化对应 builder。 |
| [`core/media/ffmpeg_builders/video/*`](file:///workspace/app/core/media/ffmpeg_builders/video) | `TS/MP4/FLV/MKV/MOV/NUT` 视频命令构造器。 |
| [`core/media/ffmpeg_builders/audio/*`](file:///workspace/app/core/media/ffmpeg_builders/audio) | `MP3/M4A/WAV/AAC/WMA` 音频命令构造器。 |
| [`core/media/direct_downloader.py`](file:///workspace/app/core/media/direct_downloader.py) | `DirectStreamDownloader`：基于 `httpx` 流式下载 FLV 到 `aiofiles`，带 stop_event 控制与重定向支持，用于 Shopee 等 ffmpeg 难以处理的源。 |

### 4.6 UI 层（ui）

| 模块 | 职责 |
|---|---|
| [`ui/base_page.py`](file:///workspace/app/ui/base_page.py) | `PageBase` 抽象基类：定义 `app/page/content_area/_` 与抽象 `async load()`。 |
| [`ui/views/home_view.py`](file:///workspace/app/ui/views/home_view.py) | `HomePage`：登录后的欢迎页，含问候语、快捷操作、公告、录制统计、功能卡片。 |
| [`ui/views/recordings_view.py`](file:///workspace/app/ui/views/recordings_view.py) | `RecordingsPage`：录制管理主页（最复杂视图）。卡片网格/列表切换、状态/平台过滤、批量开始/停止/删除、添加/编辑对话框、搜索、键盘快捷键（Alt+H/N/B/P/D、Ctrl+F/R）、pubsub 跨客户端同步、自适应列数计算。 |
| [`ui/views/settings_view.py`](file:///workspace/app/ui/views/settings_view.py) | `SettingsPage`：5 个 Tab（录制、推送、Cookies、Accounts、安全），所有改动通过 `DelayedTaskExecutor` 防抖落盘，Ctrl+S 强制保存。 |
| [`ui/views/storage_view.py`](file:///workspace/app/ui/views/storage_view.py) | `StoragePage`：录制目录浏览器，使用线程池避免阻塞 UI，支持目录导航和视频预览（web 模式构造 API URL）。 |
| [`ui/views/about_view.py`](file:///workspace/app/ui/views/about_view.py) | `AboutPage`：项目介绍、版本、特性、版本更新日志、更新检查按钮。 |
| [`ui/views/login_view.py`](file:///workspace/app/ui/views/login_view.py) | `LoginPage`：Web 模式独立登录卡片，校验后写入 `page.shared_preferences["session_token"]`。 |
| [`ui/components/business/recording_card.py`](file:///workspace/app/ui/components/business/recording_card.py) | `RecordingCardManager`：每条录制一张卡片，绑定 7 个操作按钮（录制/打开目录/信息/预览/编辑/删除/监控）、每秒刷新时长标签、pubsub 订阅 `update`/`delete`。 |
| [`ui/components/business/recording_dialog.py`](file:///workspace/app/ui/components/business/recording_dialog.py) | `RecordingDialog`：新增/编辑录制对话框，支持单条与批量输入；`RecordingConfig` 提供取值优先级（初始值 → 用户配置 → 默认）。 |
| [`ui/components/business/video_player.py`](file:///workspace/app/ui/components/business/video_player.py) | `VideoPlayer`：基于 `flet_video` 的视频预览对话框，支持截图、打开直播间、复制 URL。 |
| [`ui/components/state/recording_card_state.py`](file:///workspace/app/ui/components/state/recording_card_state.py) | `RecordingCardState`：根据 `Recording` 状态推导 `CardStateType`、卡片边框色、状态标签文案/颜色。 |
| [`ui/filters/recording_filters.py`](file:///workspace/app/ui/filters/recording_filters.py) | `RecordingFilters`：状态/平台过滤谓词，供 `RecordingsPage` 判定卡片可见性。 |
| [`ui/navigation/sidebar.py`](file:///workspace/app/ui/navigation/sidebar.py) | `LeftNavigationMenu`/`NavigationSidebar`/`NavigationItem`：左侧导航 + 主题切换 + 14 色调色板。 |
| [`ui/layout/responsive_layout.py`](file:///workspace/app/ui/layout/responsive_layout.py) | `setup_responsive_layout`：宽度 <768 切换到移动端布局（底部 NavigationBar），否则桌面端（侧边栏 + 内容）。 |
| [`ui/themes/theme.py`](file:///workspace/app/ui/themes/theme.py) | `create_light_theme`/`create_dark_theme`：构造 `ft.Theme`；`PopupColorItem`：调色板项。 |
| [`ui/themes/theme_manager.py`](file:///workspace/app/ui/themes/theme_manager.py) | `ThemeManager`：注册阿里普惠体字体、应用初始主题、更新 Material 3 seed 色并持久化。 |

### 4.7 配置（config）

| 模块 | 职责 |
|---|---|
| [`core/config/config_manager.py`](file:///workspace/app/core/config/config_manager.py) | `ConfigManager`：管理 8 个 JSON 文件（user_settings / cookies / accounts / recordings / web_auth / version / language / default_settings）；提供加载、原子保存（线程锁 + 临时文件 + `os.replace`）、`get_config_value`。 |
| [`core/config/settings_config.py`](file:///workspace/app/core/config/settings_config.py) | `SettingsConfig`：把 `ConfigManager` 中的字典加载到内存，提供类型化访问；`get_video_save_path` 返回录制目录；`adopt_*` 在 UI 重建配置后替换内部引用。 |
| [`core/config/language_manager.py`](file:///workspace/app/core/config/language_manager.py) | `LanguageManager`：根据 `language_code` 加载 `locales/<code>.json`；观察者模式通知页面重渲染；`create_headless` 为无 UI 的 Web 后端构造。 |

### 4.8 横切关注点

| 模块 | 职责 |
|---|---|
| [`auth/auth_manager.py`](file:///workspace/app/auth/auth_manager.py) | `AuthManager`：Web 模式登录、会话 token（`secrets`）、SHA-256+盐密码哈希、改密。首次启动创建默认 `admin/admin`。 |
| [`api/video_stream_service.py`](file:///workspace/app/api/video_stream_service.py) | 独立 FastAPI 服务（默认 6007）：HTTP Range 流式播放录制文件、ETag/304、路径穿越防护、chunk 缓存。 |
| [`core/update/update_checker.py`](file:///workspace/app/core/update/update_checker.py) | `UpdateChecker`：按优先级并发查询 GitHub Release / 自定义 API；语义版本比较（含 `-alpha/-beta/-rc`）；弹出更新对话框。 |
| [`initialization/installation_manager.py`](file:///workspace/app/initialization/installation_manager.py) | `InstallationManager`：检测 FFmpeg/Node.js 缺失并弹出带进度条的安装对话框。 |
| [`scripts/ffmpeg_install.py`](file:///workspace/app/scripts/ffmpeg_install.py) | 跨平台 FFmpeg 自动安装（Win 蓝奏云、macOS Homebrew、Linux yum/apt）+ `PATH` 更新 + `check_ffmpeg_installed`。 |
| [`scripts/node_install.py`](file:///workspace/app/scripts/node_install.py) | 跨平台 Node.js 自动安装（Win npmmirror、CentOS/Ubuntu NodeSource、macOS Homebrew）。 |
| [`lifecycle/app_close_handler.py`](file:///workspace/app/lifecycle/app_close_handler.py) | 关闭确认对话框、最小化到托盘、等待录制完成、托盘销毁、`os._exit(0)`。 |
| [`lifecycle/tray_manager.py`](file:///workspace/app/lifecycle/tray_manager.py) | `TrayManager`：基于 `pystray` 的系统托盘图标，提供 Restore/Exit 菜单。 |
| [`messages/message_pusher.py`](file:///workspace/app/messages/message_pusher.py) | `MessagePusher`：根据设置把开播/结束通知扇出到所有启用的渠道；`should_push_message` 综合判断推送条件。 |
| [`messages/notification_service.py`](file:///workspace/app/messages/notification_service.py) | `NotificationService`：钉钉、企业微信、飞书、Bark、ntfy、Server 酱、Telegram、邮件（SMTP）的具体发送实现。 |
| [`messages/desktop_notify.py`](file:///workspace/app/messages/desktop_notify.py) | `send_notification`：基于 `plyer` 的系统通知；`should_push_notification` 仅在窗口隐藏时触发。 |
| [`utils/logger.py`](file:///workspace/app/utils/logger.py) | `loguru` 配置：`logs/streamget.log`（DEBUG，3MB 轮转）+ 自定义 `STREAM` 级别到 `logs/play_url.log`。 |
| [`utils/utils.py`](file:///workspace/app/utils/utils.py) | 大杂烩工具：`is_web_session_alive`、`run_task_safe`、`trace_error_decorator`、emoji/URL/cookie/时间/文件/网络辅助、`get_startup_info`（隐藏 Windows 控制台）、`open_folder` 等。 |
| [`utils/delay.py`](file:///workspace/app/utils/delay.py) | `DelayedTaskExecutor`：防抖保存配置（默认 3 秒）。 |

### 4.9 模型层（models）

| 模块 | 职责 |
|---|---|
| [`models/recording/recording_model.py`](file:///workspace/app/models/recording/recording_model.py) | `Recording`：核心领域对象，含 ~30 个字段（URL、流主、格式、质量、监控状态、定时、is_live/is_recording/manually_stopped/force_stop、累计时长、平台等）；`to_dict`/`from_dict` 持久化。 |
| [`models/recording/recording_status_model.py`](file:///workspace/app/models/recording/recording_status_model.py) | `CardStateType` 枚举 + `RecordingStatus` 字符串常量（监控中、录制中、错误、未在定时窗口等）。 |
| [`models/media/video_format_model.py`](file:///workspace/app/models/media/video_format_model.py) | `VideoFormat`：TS/MP4/FLV/MKV/MOV/NUT。 |
| [`models/media/audio_format_model.py`](file:///workspace/app/models/media/audio_format_model.py) | `AudioFormat`：WAV/MP3/WMA/M4A/AAC。 |
| [`models/media/video_quality_model.py`](file:///workspace/app/models/media/video_quality_model.py) | `VideoQuality`：OD(原画)/UHD/HD/SD/LD。 |

---

## 5. 关键类与函数说明

### 5.1 入口与编排

#### `main.main(page: ft.Page) -> None` ([main.py](file:///workspace/main.py))
Flet 主回调。流程：创建 `App` → 设置窗口 → 注册主题/路由/窗口事件 → Web 模式下走 `AuthManager` 鉴权（默认 `admin/admin`，可通过 `login_required` 配置关闭） → `load_app()` 加载页面、启动周期监控任务、恢复上次路由。

#### `class App` ([app/app_manager.py](file:///workspace/app/app_manager.py))
- `__init__(page, services)`：组装所有页面、侧边栏、`RecordingCardManager`、`InstallationManager`、`UpdateChecker`，调用 `services.register_ui_bridge(self)` 把自己注册为后端广播目标。
- `schedule_card_update/recording/remove/snack/pubsub`：实现 `UIBridge`，把后端事件投递到当前会话的 asyncio 循环（`asyncio.run_coroutine_threadsafe`）。
- `switch_page(name)`：路由切换，先 `clear_content_area` 再调用目标页 `load()`。
- `start_periodic_tasks()`：调用 `record_manager.setup_periodic_live_check(interval)`。

### 5.2 后端服务

#### `class BackendServices` ([backend_services.py](file:///workspace/app/core/runtime/backend_services.py))
- `bootstrap(run_path)`：幂等创建单例并懒加载 `RecordingManager`。
- `start_background_loop()`：启动守护线程跑专用 asyncio 循环，并启动 `check_free_space` + `setup_periodic_live_check`。
- `run_coro(coro)`：跨线程把协程投递到后台循环。
- `register_ui_bridge/unregister_ui_bridge/snapshot_bridges`：线程安全管理 UI 桥弱引用集合。
- `broadcast_card_update/broadcast_card_remove/broadcast_snack/broadcast_pubsub`：扇出到所有已注册 UI 桥。

#### `class UIBridge(Protocol)` ([backend_services.py](file:///workspace/app/core/runtime/backend_services.py))
结构化协议，定义 `schedule_card_update/schedule_card_remove/schedule_snack/schedule_pubsub` 四个方法，由桌面 `App`（及潜在的多 web 会话）实现。

#### `class AsyncProcessManager` ([process_manager.py](file:///workspace/app/core/runtime/process_manager.py))
- `add_process(process)`：登记 ffmpeg 子进程。
- `async cleanup()`：优雅停止所有子进程（Windows 写 `q`，Unix `SIGINT`，5s 超时后 `kill`）。

### 5.3 录制核心

#### `class RecordingManager` ([record_manager.py](file:///workspace/app/core/recording/record_manager.py))
- `recordings` 属性：只读访问 `GlobalRecordingState.recordings`，写入必须用 `add_recording/remove_recording/clear_all_recordings`。
- `setup_periodic_live_check(interval=180)`：后台任务循环调用 `check_all_live_status` + `check_free_space`，类级 `_periodic_task_running` 标志保证全局唯一。
- `check_if_live(recording)`：核心调度——评估定时窗口 → 解析平台 → 加信号量 → 创建 `LiveStreamRecorder` → `fetch_stream` → `is_live` 则更新状态、发桌面通知 + 多渠道推送、`recorder.start_recording`。
- `start_monitor_recording/stop_recording/delete_recording_cards`：单条操作入口。
- `check_free_space(output_dir)`：低于阈值时关闭 `recording_enabled` 并长驻 snack 提示。

#### `class LiveStreamRecorder` ([stream_manager.py](file:///workspace/app/core/recording/stream_manager.py))
- `fetch_stream()`：调 `get_platform_handler` 取 `StreamData`。
- `start_recording(stream_info)`：决定格式、文件名、目录、路径；`ffmpeg_builders.create_builder(fmt).build_command()` → `start_ffmpeg(cmd)`，或 `DirectStreamDownloader` → `start_direct_download`。
- `start_ffmpeg(cmd)`：`asyncio.create_subprocess_exec` 启动，注册到 `process_manager`，轮询直到 `should_stop/force_stop/returncode`，处理结束/错误状态、ts→mp4 转码、自定义脚本、`recheck_live_status`。
- `converts_mp4(path, is_original_delete)`：`ffmpeg -i ... -c:v copy -c:a copy -f mp4 ...`，成功后删除原文件或移到 `original/` 子目录。
- `request_stop()`：设置 `should_stop=True` 让轮询循环退出。
- `recheck_live_status()`：录制时长 > `min_valid_recording_duration`(25s) 且非 FLV 偏好平台时重新 `check_if_live`，否则标记 `RECORDING_ERROR`。

### 5.4 平台与媒体

#### `class PlatformHandler(abc.ABC)` ([base.py](file:///workspace/app/core/platforms/platform_handlers/base.py))
- `register(*patterns)`：类装饰器，把 URL 正则加入 `_registry`。
- `get_handler_class(live_url)`：遍历 `_registry` 找到首个匹配的 handler 类。
- `get_handler_instance(live_url, ...)`：解析类 → 计算 `InstanceKey` → 双检锁缓存实例（用 `inspect.signature` 过滤多余 kwargs，避免子类 `__init__` 签名不同报错）。
- `get_stream_info(live_url) -> StreamData`：抽象方法，由各平台子类实现。

#### `get_platform_handler(live_url, ...)` / `get_platform_info(record_url)` ([__init__.py](file:///workspace/app/core/platforms/platform_handlers/__init__.py))
工厂函数；后者通过硬编码 `platform_map` 返回 `(显示名, 平台 key)`。

#### `class FFmpegCommandBuilder(abc.ABC)` ([base.py](file:///workspace/app/core/media/ffmpeg_builders/base.py))
- `build_command() -> list[str]`：抽象。
- `_get_basic_ffmpeg_command()`：构建通用 ffmpeg 前缀（rw_timeout/analyzeduration/probesize/bufsize/max_muxing_queue_size 等），按 `is_overseas` 选 `DEFAULT_CONFIG` 或 `OVERSEAS_CONFIG`，按需插入 `-headers` 与 `-http_proxy`。

#### `create_builder(format_type, *args, **kwargs)` ([__init__.py](file:///workspace/app/core/media/ffmpeg_builders/__init__.py))
格式字符串 → builder 类的映射工厂，支持 11 种格式。

#### `class DirectStreamDownloader` ([direct_downloader.py](file:///workspace/app/core/media/direct_downloader.py))
`httpx.AsyncClient` 流式 GET → `aiofiles` 分块写盘；`stop_download()` 设置 `stop_event` 并等待任务结束。

### 5.5 配置与 i18n

#### `class ConfigManager` ([config_manager.py](file:///workspace/app/core/config/config_manager.py))
- `init()` / `init_*_config()`：首次运行时创建缺失的 JSON 文件（user 配置从 `default_settings.json` 复制）。
- `_save_config(path, config, ...)`：`threading.Lock` + `tempfile.mkstemp` + `os.replace`，在 `asyncio.to_thread` 中执行，保证原子且不阻塞事件循环。
- `save_user_config/save_cookies_config/save_accounts_config/save_recordings_config/save_web_auth_config`：异步保存各配置。

#### `class SettingsConfig` ([settings_config.py](file:///workspace/app/core/config/settings_config.py))
- `get_config_value(key, default)`：user → default 兜底。
- `get_video_save_path()`：返回 `live_save_path` 或 `<run_path>/downloads`。
- `adopt_user_config/adopt_cookies_config/adopt_accounts_config`：UI 重建配置字典后替换内部引用。

#### `class LanguageManager` ([language_manager.py](file:///workspace/app/core/config/language_manager.py))
- `create_headless(services)`：无 `app` 的构造路径（Web 后端）。
- `add_observer/remove_observer/notify_observers`：观察者模式，语言切换时通知所有页面重渲染。

### 5.6 UI 关键类

#### `class PageBase` ([base_page.py](file:///workspace/app/ui/base_page.py))
所有路由页面基类，定义 `app/page/content_area/_` 与抽象 `async load()`。

#### `class RecordingsPage(PageBase)` ([recordings_view.py](file:///workspace/app/ui/views/recordings_view.py))
- `apply_filter()`：用 `RecordingFilters.should_show_recording` 设置每张卡片可见性。
- `add_recording(recordings_info)`：构造 `Recording` 实例 → `get_platform_info` 探测 → `record_manager.add_recording` → `record_card_manager.create_card` → pubsub 广播 `add`。
- `on_keyboard(e)`：Alt+H/N/B/P/D、Ctrl+F/R 等快捷键。
- `recalculate_grid_columns()`：根据页面宽度减去侧边栏宽度计算列数。

#### `class RecordingCardManager` ([recording_card.py](file:///workspace/app/ui/components/business/recording_card.py))
- `create_card(recording, subscribe_add_cards=False)`：可选触发 `check_if_live`，构造 7 个操作按钮，启动每秒刷新时长的后台任务。
- `update_card(recording)`：刷新标题、状态标签、时长、按钮图标、卡片颜色。
- `edit_recording_callback(recording_list)`：应用编辑并广播 `update`。
- `preview_video_button_on_click`：Web 用 `recording.preview_url`；桌面遍历 `recording_dir` 取最新有效视频。

#### `class RecordingCardState` ([recording_card_state.py](file:///workspace/app/ui/components/state/recording_card_state.py))
`get_card_state(recording) -> CardStateType` 状态机；`get_border_color` / `get_status_label_config` 提供视觉配置。

#### `class RecordingFilters` ([recording_filters.py](file:///workspace/app/ui/filters/recording_filters.py))
`STATUS_FILTER_MAP`（all/recording/living/error/offline/stopped）→ 谓词；`should_show_recording` 组合状态 + 平台谓词。

#### `class SettingsPage(PageBase)` ([settings_view.py](file:///workspace/app/ui/views/settings_view.py))
- 5 个 Tab 构造方法 `create_*_settings_tab`。
- `on_change/on_cookies_change/on_accounts_change`：内存更新 + 副作用 + 防抖保存。
- `is_changed()`：Ctrl+S 强制保存所有脏配置。

#### `class ThemeManager` ([theme_manager.py](file:///workspace/app/ui/themes/theme_manager.py))
`init_fonts()` 注册阿里普惠体；`apply_initial_theme()` 应用主题；`update_theme_color(color)` 改 Material 3 seed 色并持久化。

### 5.7 消息推送

#### `class MessagePusher` ([message_pusher.py](file:///workspace/app/messages/message_pusher.py))
- `should_push_message(settings, recording, check_manually_stopped, message_type)`：综合判断（启用推送、是否仅通知不录制、开播/结束通知开关、是否有可用渠道、手动停止抑制）。
- `push_messages(title, content)`：遍历 `_get_push_channels()` 调对应 `NotificationService.send_to_*`。
- `push_messages_sync(title, content)`：在新事件循环里同步执行，供 `BackgroundService` 调用。

#### `class NotificationService` ([notification_service.py](file:///workspace/app/messages/notification_service.py))
8 种渠道的具体发送：`send_to_dingtalk/wechat/feishu/bark/ntfy/serverchan/telegram/email`。

### 5.8 认证、更新、安装

#### `class AuthManager` ([auth_manager.py](file:///workspace/app/auth/auth_manager.py))
- `initialize()`：无用户则创建 `admin/admin`（SHA-256 + 随机盐）。
- `authenticate(username, password)` → 返回 token 并登记会话。
- `validate_session(token)` / `logout(token)` / `change_password(...)`。

#### `class UpdateChecker` ([update_checker.py](file:///workspace/app/core/update/update_checker.py))
- `check_for_updates()`：按优先级并发查询 GitHub / 自定义 API，`asyncio.as_completed` 取首个成功。
- `_compare_versions(v1, v2)`：语义版本比较，处理 `-alpha/-beta/-rc` 预发布标签。
- `show_update_dialog(update_info)`：Flet `AlertDialog`，`page.launch_url` 打开对应平台下载链接。

#### `class InstallationManager` ([installation_manager.py](file:///workspace/app/initialization/installation_manager.py))
`check_env()` 入口；`get_install_components()` 检测 FFmpeg/Node.js；`install_components()` 调度安装脚本并更新进度环。

### 5.9 视频点播 API

#### `video_stream_service.py` ([api/video_stream_service.py](file:///workspace/app/api/video_stream_service.py))
独立 FastAPI 应用：
- `validate_filename(filename)`：拒绝路径分隔符，防穿越。
- `GET /api/videos/{filename}` 与 `GET /api/videos/{filename}/{subfolder}`：支持 `If-None-Match`/`If-Modified-Since`（304）、ETag、HTTP Range（206）、完整流式。
- `file_sender` / `file_sender_range`：`aiofiles` 64KB 分块；小 chunk（<1MB）进 `CHUNK_CACHE`。

### 5.10 工具函数精选 ([utils/utils.py](file:///workspace/app/utils/utils.py))
- `is_web_session_alive(page)`：检查 flet session/connection/loop。
- `run_task_safe(page, handler, *args, ui_only=False, **kwargs)`：安全调度协程。
- `trace_error_decorator(func)`：捕获 `execjs.ProgramError`（Node.js 报错）与通用异常，记录行号。
- `check_md5` / `get_file_paths` / `check_disk_capacity` / `is_valid_video_file`：文件工具。
- `clean_name` / `remove_emojis` / `dict_to_cookie_str` / `jsonp_to_json`：字符串/cookie 工具。
- `is_current_time_within_range` / `add_hours_to_time` / `is_time_interval_exceeded`：时间工具。
- `get_startup_info`：Windows 子进程隐藏控制台；`open_folder`：跨平台打开目录。

---

## 6. 依赖关系

### 6.1 运行时第三方依赖（[pyproject.toml](file:///workspace/pyproject.toml)）

| 包 | 版本 | 用途 |
|---|---|---|
| `flet[desktop,cli]` | `==0.85.3` | 跨端 UI 框架（桌面端），CLI 打包工具 |
| `flet[web,cli]` | `==0.85.3` | Web 端变体（`requirements-web.txt`） |
| `flet-video` | `==0.85.3` | 视频预览控件 |
| `httpx[http2]` | `>=0.28.1` | HTTP 客户端（推送、平台请求、视频 API） |
| `screeninfo` | `>=0.8.1` | 获取屏幕分辨率以设置窗口大小 |
| `aiofiles` | `>=25.1.0` | 异步文件 IO（直下、视频流式） |
| `streamget` | `>=4.0.10` | 各直播平台解析库（核心） |
| `python-dotenv` | `>=1.2.1` | 加载 `.env` |
| `cachetools` | `>=7.1.4` | 视频 API 元数据/chunk 缓存（仅 Web） |
| `pystray` | `>=0.19.5` | 系统托盘 |
| `plyer` | `>=2.1.0` | 跨平台桌面通知 |
| `deprecated` | `>=1.3.1` | `@deprecated` 装饰器 |
| `loguru` | （隐式） | 日志 |
| `fastapi` / `uvicorn` | （隐式，Web 模式） | `app/api/video_stream_service.py` |
| `distro` / `execjs` | （隐式） | Node 安装 / 平台 handler |

### 6.2 内部模块依赖图（关键路径）

```
main.py
  → app.app_manager.App
      → core.runtime.backend_services.BackendServices (singleton)
          → core.config.config_manager.ConfigManager
          → core.config.settings_config.SettingsConfig
          → core.config.language_manager.LanguageManager
          → core.runtime.process_manager.AsyncProcessManager
          → core.recording.record_manager.RecordingManager (lazy)
                → core.recording.stream_manager.LiveStreamRecorder
                      → core.platforms.platform_handlers (get_platform_handler)
                            → streamget (third-party)
                      → core.media.ffmpeg_builders.create_builder
                            → core.media.ffmpeg_builders.base.FFmpegCommandBuilder
                      → core.media.direct_downloader.DirectStreamDownloader
                      → messages.message_pusher.MessagePusher
                            → messages.notification_service.NotificationService
                      → messages.desktop_notify
      → ui.views.* (各页面)
      → ui.components.business.recording_card.RecordingCardManager
      → ui.navigation.sidebar.LeftNavigationMenu
      → initialization.installation_manager.InstallationManager
            → scripts.ffmpeg_install / scripts.node_install
      → core.update.update_checker.UpdateChecker
      → auth.auth_manager.AuthManager (web only)
      → lifecycle.tray_manager.TrayManager (desktop only)
```

### 6.3 数据/控制流依赖

- **配置流**：`default_settings.json` → `ConfigManager.init_user_config` → `user_settings.json` → `SettingsConfig` 内存字典 → 各模块读取；UI 改动经 `DelayedTaskExecutor` 防抖 → `ConfigManager._save_config` 原子写回。
- **录制状态流**：`RecordingManager.check_if_live` → `LiveStreamRecorder.start_recording` → ffmpeg 子进程 → 状态变更 → `services.broadcast_card_update` → 各 `UIBridge.schedule_card_update` → UI 卡片刷新。
- **跨客户端同步**：web 多会话通过 `page.pubsub.send_others_on_topic` 广播 `add`/`update`/`delete`/`delete_all` 主题。
- **持久化**：`recordings.json`（录制列表）、`user_settings.json`、`cookies.json`、`accounts.json`、`web_auth.json`，均由 `ConfigManager` 管理。

### 6.4 外部系统依赖

- **FFmpeg**：必需，所有录制都通过 ffmpeg；缺失时 `InstallationManager` 提示安装。
- **Node.js**：部分平台 handler（依赖 `streamget` 中的 JS 执行）需要；`execjs` 调用。
- **GitHub API**：`UpdateChecker._check_github_update` 查询 Release。
- **直播平台 Web API**：通过 `streamget` 各 `LiveStream` 客户端访问。
- **通知服务**：钉钉/企业微信/飞书/Bark/ntfy/Telegram/ServerChan/SMTP。

---

## 7. 项目运行方式

### 7.1 环境要求

- **Python**：`>=3.10, <4.0`（推荐 3.12）
- **FFmpeg**：必需（程序会在缺失时引导安装）
- **Node.js**：部分平台解析需要
- 操作系统：Windows / macOS / Linux

### 7.2 从源码运行

```bash
# 1. 克隆
git clone https://github.com/ihmily/StreamCap.git
cd StreamCap

# 2. 安装 streamget 核心
pip install -i https://pypi.org/simple streamget

# 3. 安装依赖
pip install -r requirements.txt        # 桌面端
pip install -r requirements-web.txt    # Web 端（二选一）

# 4. 配置环境变量
cp .env.example .env
# 按需修改 .env（PLATFORM / HOST / PORT / VIDEO_API_PORT 等）

# 5. 运行
python main.py                  # 桌面端（Windows/macOS 默认）
python main.py --web            # Web 端（Linux 推荐）
python main.py --web --host 0.0.0.0 --port 6006   # 自定义监听
```

启动 Web 端后访问 `http://127.0.0.1:6006`，默认登录 `admin/admin`（可在 设置 → 安全 修改或关闭登录要求）。

### 7.3 桌面端 vs Web 端差异

| 维度 | 桌面端 | Web 端 |
|---|---|---|
| 启动命令 | `python main.py` | `python main.py --web` |
| Flet 视图 | `FLET_APP_HIDDEN` | `WEB_BROWSER`（CanvasKit） |
| 后台循环 | 不启动 `BackendServicesLoop` | 启动独立 asyncio 守护线程 |
| 系统托盘 | 有（`TrayManager`） | 无 |
| 鉴权 | 无 | `AuthManager` 可选登录 |
| 视频预览 | 直接打开本地文件路径 | 走 `VIDEO_API_EXTERNAL_URL` 或本地 6007 API |
| 多会话 | 单实例 | 多客户端通过 pubsub 同步 |
| 资源 | `flet[desktop]` + `pystray` + `plyer` | `flet[web]` + `cachetools` |

### 7.4 CLI 参数

```
python main.py [--web] [--host HOST] [--port PORT]
```
- `--web`：以 Web 模式运行（也可在 `.env` 设 `PLATFORM=web`）。
- `--host`：默认读 `HOST` 环境变量，再默认 `127.0.0.1`。
- `--port`：默认读 `PORT` 环境变量，再默认 `6006`。

### 7.5 打包为桌面应用

通过 Flet CLI / PyInstaller：

```bash
flet pack .               # 生成可执行文件
```
`pyproject.toml` 中的 `[tool.flet]` 配置：`org=io.github.ihmily.streamcap`、`product=StreamCap`，`compile.app=false`。打包后由 `bundled_env.setup_bundled_flet_view` 自动定位内置 Flet 视图。

### 7.6 开发调试

- **日志**：`logs/streamget.log`（DEBUG，3MB 轮转，保留 3 份）；`logs/play_url.log`（仅 `STREAM` 级别，500KB 轮转）。
- **Lint**：`ruff`（`[tool.poetry.group.lint]`），配置见 `.ruff.toml`。
- **测试**：CI 见 `.github/workflows/test.yml`；lint 见 `python-lint.yml`。
- **国际化**：编辑 `locales/zh_CN.json` / `locales/en.json`；语言映射在 `config/language.json`。

---

## 8. 配置文件说明

### 8.1 `.env`（[.env.example](file:///workspace/.env.example))

| 变量 | 默认 | 说明 |
|---|---|---|
| `PLATFORM` | `desktop` | 运行模式 `desktop` / `web` |
| `TZ` | `Asia/Shanghai` | 容器时区 |
| `HOST` | `127.0.0.1` | Web 监听地址 |
| `PORT` | `6006` | Web 服务端口 |
| `VIDEO_API_PORT` | `6007` | 视频点播 API 端口 |
| `CUSTOM_VIDEO_ROOT_DIR` | 空 | 视频根目录（默认 `<run_path>/downloads`） |
| `VIDEO_API_EXTERNAL_URL` | 空 | Web 端视频预览的外部访问 URL |
| `AUTO_CHECK_UPDATE` | `false` | 启动自动检查更新 |
| `UPDATE_SOURCE` | `both` | `github` / `custom` / `both` |
| `GITHUB_REPO` | `ihmily/StreamCap` | GitHub 仓库 |
| `CUSTOM_UPDATE_API` | 空 | 自定义更新 API |
| `UPDATE_CHECK_INTERVAL` | `86400` | 检查间隔（秒） |

### 8.2 `config/default_settings.json` ([file](file:///workspace/config/default_settings.json))

首次运行被复制为 `user_settings.json`，包含全部用户可调项，按功能分组：

- **录制**：`video_format`(TS)、`record_quality`(OD)、`loop_time_seconds`(180)、`segmented_recording_enabled`、`video_segment_time`(1800)、`convert_to_mp4`、`delete_original`、`generate_time_subtitle_file`、`execute_custom_script` + `custom_script_command`、`default_live_source`(FLV)、`flv_use_direct_download`、`force_https_recording`、`recording_space_threshold`(2.0 GB)。
- **文件命名**：`live_save_path`、`filename_includes_title`、`remove_emojis`、`folder_name_{platform,author,time,title}`。
- **网络**：`enable_proxy`、`proxy_address`、`default_platform_with_proxy`（逗号分隔的需代理平台 key 列表）。
- **通知**：`system_notification_enabled`、`stream_start/end_notification_enabled`、`only_notify_no_record`、`custom_notification_title/content`，以及 8 个渠道的 `*_enabled` 与具体配置（webhook、API key、SMTP 等）。
- **UI**：`language`(Chinese)、`theme_color`(blue)、`is_grid_view`(true)、`theme_mode`(light)、`last_route`(/home)、`check_live_on_browser_refresh`、`platform_max_concurrent_requests`(3)、`remember_window_size`。

### 8.3 其他配置

- `config/version.json`：版本元数据（介绍、许可证、`version_updates` 数组，含 `version`/`kernel_version`/`release_date`/多语言更新说明/公告）。`UpdateChecker` 与 `AboutPage` 读取。
- `config/language.json`：`{"Chinese": "zh_CN", "English": "en"}`。
- `cookies.json`：~46 个平台的 cookie 字符串（设置 → Cookies Tab 编辑）。
- `accounts.json`：sooplive/flextv/popkontv/twitcasting 等需要登录的平台账号（`key_sub` 形式）。
- `recordings.json`：录制任务列表（`Recording.to_dict()` 序列化）。
- `web_auth.json`：Web 模式用户与密码哈希。
- `locales/{zh_CN,en}.json`：i18n 文案，按命名空间（`home_page`/`recordings_page`/`settings_page`/`recording_card`/`base` 等）组织。

---

## 9. 部署（Docker）

### 9.1 Dockerfile ([file](file:///workspace/Dockerfile))

多阶段构建，最终镜像基于 `python:3.12-slim`：
- **Builder 阶段**：装 `build-essential`，`pip install -r requirements-web.txt`，拷贝源码，创建 `/app/logs` 和 `/app/downloads`。
- **Final 阶段**：装 `ffmpeg`、`tzdata`、`curl`、`gnupg`，通过 NodeSource 安装 Node.js 20；设 `TZ=Asia/Shanghai` 并链接时区；从 builder 拷贝 site-packages、bin、源码。
- **CMD**：`python main.py --web --host 0.0.0.0`。

### 9.2 docker-compose.yml ([file](file:///workspace/docker-compose.yml))

定义两个服务，共享 `streamcap-network` 桥接网络：

| 服务 | 镜像 | 端口 | 命令 | 卷 |
|---|---|---|---|---|
| `streamcap` | `ihmily/streamcap` | `${PORT:-6006}` | Dockerfile CMD | `./logs`、`./config`、`./downloads`、`./.env` |
| `video_api` | `ihmily/streamcap` | `${VIDEO_API_PORT:-6007}` | `python -m app.api.video_stream_service` | `./downloads`、`./.env` |

`video_api` 依赖 `streamcap`，两者共享 `downloads` 卷：主应用录制文件，视频 API 提供点播。

### 9.3 容器使用

```bash
# 准备 .env
cp .env.example .env

# 启动（前台）
docker compose up

# 后台
docker compose up -d

# 停止
docker compose stop

# 自定义构建
docker build -t streamcap .
```

健康检查：`streamcap` 服务通过 `curl http://localhost:6006/about` 探活。

---

## 10. 录制核心流程详解

### 10.1 添加录制任务

1. 用户在 `RecordingsPage` 点击 +，`RecordingDialog.show_dialog()` 收集 URL/格式/质量/定时等参数。
2. `on_confirm` 调 `get_platform_info(url)` 探测平台（未知平台给出警告但允许保存为 `custom`）。
3. `RecordingsPage.add_recording(recordings_info)`：构造 `Recording` → `record_manager.add_recording` → `record_card_manager.create_card` → pubsub 广播 `add`（多客户端同步）。

### 10.2 周期监控循环

`RecordingManager.setup_periodic_live_check(interval=180)` 启动后台任务：

```
while not stop:
    await check_free_space()
    await check_all_live_status()
    await asyncio.sleep(interval)
```

`check_all_live_status` 遍历所有 `monitor_status=True` 且未在录制的录制，若距上次检查超过 `loop_time_seconds` 则触发 `check_if_live`。

### 10.3 单条录制生命周期

`check_if_live(recording)`：

1. 跳过条件检查：`stopping_in_progress` / `is_recording` / `is_checking`。
2. 评估定时窗口 `get_scheduled_time_range`：未在窗口内则置 `NOT_IN_SCHEDULED_CHECK`。
3. `get_platform_info` 解析平台；构造 `recording_info`。
4. 获取平台并发信号量（`Semaphore(platform_max_concurrent_requests)`，默认 3）。
5. 创建 `LiveStreamRecorder(services, recording, recording_info)`，`await recorder.fetch_stream()` 得到 `StreamData`。
6. 若 `is_live`：
   - 更新 `Recording` 状态、`start_time`、`is_recording=True`、广播 `update`。
   - `should_push_message` 判定后调 `MessagePusher` 推送开播通知 + `desktop_notify` 桌面通知。
   - `only_notify_no_record` 为真则仅通知不录制；否则 `await recorder.start_recording(stream_info)`。
7. 若不在线：置 `NOT_RECORDING`，重置 `notified_live_end`。

`LiveStreamRecorder.start_recording(stream_info)`：

1. `_get_record_format`：shopee / `flv_use_direct_download` 强制 `flv` + 直下；FLV 源为 h265 时降级 `ts`。
2. `_get_filename` / `_get_output_dir` / `_get_save_path`：按 `folder_name_*` 配置组装路径；分段录制追加 `_%03d.<ext>`。
3. 停掉同 `rec_id` 的旧 recorder。
4. 注册到 `record_manager.active_recorders[rec_id]`。
5. 选择路径：
   - **ffmpeg**：`create_builder(fmt, ...).build_command()` → `start_ffmpeg(cmd)` → `asyncio.create_subprocess_exec` → `process_manager.add_process` → 轮询直到 `should_stop/force_stop/returncode` → 处理结束/错误。
   - **直下**：`DirectStreamDownloader(record_url, save_path, headers, proxy).start_download()` → 轮询直到 stop 或下载完成。
6. 结束后处理：
   - `convert_to_mp4` 为真则 `converts_mp4`（`ffmpeg -c:v copy -c:a copy -f mp4`）。
   - `execute_custom_script` 为真则 `custom_script_execute`（Python / shell 脚本，参数形态不同）。
   - `recheck_live_status`：时长 > 25s 且非 FLV 偏好平台则重新 `check_if_live`，否则标 `RECORDING_ERROR`。
7. `_handle_recording_finished/error` → `end_message_push` + 桌面通知（非手动停止时）。

### 10.4 停止录制

`record_manager.stop_recording(recording, manually_stopped=True)`：

- 若 `active_recorders` 中存在：`recorder.request_stop()`（设 `should_stop=True`，让 ffmpeg 轮询循环退出）。
- 否则置 `force_stop=True`。
- 更新 `cumulative_duration` 与 `last_duration`，重置 `is_recording`、`stopping_in_progress`（由 `_reset_stopping_flag` 异步清零）。

### 10.5 关闭应用

`handle_app_close(page, app, save_progress_overlay)`（[app_close_handler.py](file:///workspace/app/lifecycle/app_close_handler.py)）：

1. 弹出确认对话框：取消 / 最小化到托盘 / 退出。
2. 退出分支：保存 `last_route`，检查 `process_manager.ffmpeg_processes` 是否有活跃录制。
3. 显示 `SaveProgressOverlay`，在守护线程中按录制数量加权等待，再 `tray_manager.stop()` + `os._exit(0)`。
4. Web 模式下 `page.on_disconnect` 触发 `handle_disconnect`：保存路由、`services.unregister_ui_bridge(app)`，但**不停止后端录制**。

---

## 附录：常用快捷键

| 快捷键 | 作用 | 视图 |
|---|---|---|
| `Alt+H` | 打开帮助对话框 | 录制列表/设置/关于 |
| `Ctrl+F` | 搜索录制 | 录制列表 |
| `Ctrl+R` | 刷新卡片 | 录制列表 |
| `Ctrl+S` | 立即保存设置 | 设置 |
| `Alt+N` | 新增录制 | 录制列表 |
| `Alt+B` | 批量开始监控 | 录制列表 |
| `Alt+P` | 批量停止监控 | 录制列表 |
| `Alt+D` | 批量删除 | 录制列表 |

---

## 附录：扩展开发指引

### 新增直播平台

1. 在 [`handlers.py`](file:///workspace/app/core/platforms/platform_handlers/handlers.py) 添加 `XxxHandler(PlatformHandler)`，设置 `platform` 类属性，`__init__` 调 `super().__init__(...)` 传所需参数，`get_stream_info` 用 `@trace_error_decorator` 装饰并调 `streamget.XxxLiveStream`。
2. 在文件底部 `XxxHandler.register(r"https://...pattern.../")` 注册 URL 正则。
3. 在 [`platform_handlers/__init__.py`](file:///workspace/app/core/platforms/platform_handlers/__init__.py) 的 import 列表与 `__all__` 中加入；并在 `get_platform_info` 的 `platform_map` 中加映射。
4. 在 `locales/*.json` 添加平台显示名（可选）。

### 新增输出格式

1. 在 [`ffmpeg_builders/<audio|video>/`](file:///workspace/app/core/media/ffmpeg_builders) 新增 `xxx.py` 实现 `XXXCommandBuilder(FFmpegCommandBuilder).build_command()`。
2. 在对应 `__init__.py` 导出。
3. 在 [`ffmpeg_builders/__init__.py`](file:///workspace/app/core/media/ffmpeg_builders/__init__.py) 的 `format_to_class` 注册。
4. 在 [`models/media/`](file:///workspace/app/models/media) 对应模型类加常量。

### 新增通知渠道

1. 在 [`notification_service.py`](file:///workspace/app/messages/notification_service.py) 加 `send_to_xxx(...)`。
2. 在 [`message_pusher.py`](file:///workspace/app/messages/message_pusher.py) 的 `_get_push_channels()` 加 `xxx_enabled`。
3. 在 [`default_settings.json`](file:///workspace/config/default_settings.json) 加 `xxx_enabled` 与渠道配置键。
4. 在 [`settings_view.py`](file:///workspace/app/ui/views/settings_view.py) 的 `create_push_settings_tab` 添加 UI。
5. 在 `locales/*.json` 加文案。

---

*本 wiki 基于仓库当前主分支代码生成，如发现与实际实现不符请以源码为准。*
