# translate-video-dubbing 架构草图（本地离线版）

## 总体概览

translate-video-dubbing 平台面向“上传视频 → 本地转写/翻译 → 本地合成语音”的离线桌面场景，所有计算和数据存储均在单台 macOS 设备（M1 Pro 芯片）上完成，保证隐私与离线可用。系统由四个核心子系统构成：

1. **本地前端应用**：基于 Next.js（或 Vite + React）打包成桌面应用（Tauri/Electron），负责文件导入、处理进度展示与结果导出。
2. **本地后端服务层**：基于 FastAPI（可替换为 Flask）运行于本机，提供 REST 接口、任务管理与权限控制。
3. **任务队列与工作节点**：使用 Celery（或 RQ）结合本地 Redis/SQLite 消息后端，实现异步任务调度，避免阻塞前端交互。
4. **推理节点（CPU/Metal 加速）**：封装 ASM 语音识别与 indexTTS2 推理服务，针对 Apple Silicon 使用 PyTorch Metal 后端或 CoreML Delegate，无需 NVIDIA GPU。

服务组件通过本地回环网络通信，可使用 Docker Desktop（Apple Silicon 原生镜像）或原生 Python 虚拟环境部署。整个系统在离线状态下即可运行，不依赖外部云存储与计算。

## 服务组件与职责

| 组件 | 技术栈 | 主要职责 |
| --- | --- | --- |
| 桌面前端 | Next.js / Vite + React + Tauri（或 Electron）| 提供文件导入、进度条、结果预览与导出界面，集成本地身份认证 UI |
| API 服务 | FastAPI + Uvicorn | 处理上传、任务创建、鉴权、速率限制、向队列派发任务、提供结果查询接口 |
| 任务队列 | Celery + Redis（本地单实例）或 RQ + SQLite | 管理长耗时推理任务，维持任务状态，支持重试与计划任务 |
| 推理服务 | Python + PyTorch（Metal 后端）/ CoreML + gRPC/REST | 运行 ASM 与 indexTTS2 模型推理，利用 M1 Pro GPU/ANE 加速或高性能 CPU 线程 |
| 持久化存储 | SQLite（元数据）、本地文件系统（APFS） | 存储任务元数据、用户信息、结果缓存与上传文件 |
| 缓存层 | Redis（同一进程可嵌入）或本地文件缓存 | 缓存转写结果、短期状态数据、会话信息 |

## 模型运行环境约定

- **硬件**：
  - Apple MacBook Pro（M1 Pro 芯片，16GB+ 统一内存）。
  - 利用 Apple Metal Performance Shaders (MPS) 或 CoreML Backend 作为加速方案。
- **基础依赖**：
  - 操作系统：macOS 13 Ventura 或更高版本。
  - Python 3.10+，PyTorch >= 2.1，启用 `PYTORCH_ENABLE_MPS_FALLBACK=1` 支持 Metal。
  - 安装 `ffmpeg`（可通过 `brew install ffmpeg`）。
  - 可选：`coremltools`、`onnxruntime-metal` 以提升推理性能。
- **模型资源路径**：
  - ASM 语音识别权重存放于 `~/Library/Application Support/translate-video-dubbing/models/asm/weights.bin`。
  - indexTTS2 合成模型权重存放于 `~/Library/Application Support/translate-video-dubbing/models/index_tts2/`。
  - 公共配置（tokenizer、声码器等）放置于 `~/Library/Application Support/translate-video-dubbing/models/shared/`。
- **运行模式**：
  - 默认在 Python 虚拟环境内运行，可通过 `conda` 或 `uv` 管理依赖。
  - 若使用容器，可采用 `colima` + `docker`，选择 arm64 基础镜像并绑定本地模型目录。

## 存储与数据生命周期

1. **上传视频临时存储**：
   - 前端通过 IPC/HTTP 将文件路径或分片发送至 API，后端将文件复制到本地工作目录 `~/Library/Application Support/translate-video-dubbing/uploads/raw/{task_id}/`。
   - 支持按配置自动清理，默认在处理完成后 3 天删除。

2. **音频提取与中间产物**：
   - 由任务队列在推理前提取音频，产物存储在 `.../uploads/audio/{task_id}/`，任务完成后立即删除（可通过后台守护进程清理）。

3. **转写/翻译结果缓存**：
   - JSON 结果存储在 SQLite 数据库 `~/Library/Application Support/translate-video-dubbing/db.sqlite` 的 `transcripts` 表，字段包括 `task_id`、`language`、`status`、`payload`、`updated_at`。
   - Redis（若启用）仅在本地监听 `127.0.0.1`，缓存键形如 `transcript:{task_id}:{locale}`，TTL 24 小时；亦可使用本地文件缓存（如 `shelve`）替代。

4. **合成语音与字幕**：
   - 合成的音频、字幕文件（SRT/ASS）写入 `~/Library/Application Support/translate-video-dubbing/results/{task_id}/`，可由前端触发导出到用户指定目录。

5. **日志与审计**：
   - 结构化日志写入 `~/Library/Logs/translate-video-dubbing/app.log`，按文件大小轮转（例如 10MB * 5）。
   - 审计事件（导出、删除）记录在 SQLite `audit_events` 表。

## 服务间接口设计

### REST 接口（FastAPI，本地回环）

- `POST /api/v1/uploads`：接收视频或文件路径，返回 `task_id`。
- `POST /api/v1/tasks/{task_id}/submit`：提交处理配置（语言、翻译选项、TTS 角色）。
- `GET /api/v1/tasks/{task_id}`：查询任务状态与结果摘要。
- `GET /api/v1/tasks/{task_id}/artifacts`：返回本地文件路径或触发桌面应用下载。
- `POST /api/v1/auth/token`：本地身份认证（可结合系统账号或离线许可证）。

### gRPC / 内部 API

- `InferenceService.Transcribe(TaskSpec)`：调用 ASM 推理，返回中间 token 流或最终文本。
- `InferenceService.Synthesize(SynthesisSpec)`：调用 indexTTS2，返回音频文件路径或字节流。
- `HealthService.Check()`：返回推理服务健康状态、Metal 后端可用性、系统资源占用。

### 事件与任务流程

1. 用户在桌面前端选择视频 → API 复制文件并创建任务记录。
2. API 将 `transcribe` 任务推送到本地 Celery 队列（包含 `task_id`、文件路径、语言偏好）。
3. Worker 调用 `ffmpeg` 提取音频，随后通过 `InferenceService.Transcribe` 生成转写文本，将结果写入 SQLite 与缓存。
4. 如需翻译/TTS，Celery 继续派发 `translate`、`synthesize` 子任务；推理服务可批量调度但仅访问本地模型资源。
5. 子任务完成后，通过 WebSocket（本地）或前端轮询刷新状态；用户可导出字幕与音频到指定目录。

### 安全与隐私考量

- **上传限制**：
  - 单文件最大 8GB（受本地磁盘限制），分片大小可配置（默认 32MB），同时校验 MIME 与文件头。
  - 对本地 API 实施并发阈值与速率限制，避免资源争用。
- **权限控制**：
  - 支持本机多用户使用：通过 macOS 钥匙串或本地许可证文件验证。
  - 所有任务与资源均绑定 `owner_id`（对应系统用户），前端在切换用户时需重新认证。
  - 管理视图受限于管理员角色，可查看系统负载、手动重试任务。
- **数据安全**：
  - 本地工作目录默认存储于用户沙盒，可配置启用 FileVault 加密。
  - 导出文件前提示用户选择目标路径，避免误共享。
  - 日志中脱敏用户标识（如邮箱、访问令牌）。
- **推理节点安全**：
  - gRPC/REST 仅监听 `127.0.0.1`，通过本地 token 或 Unix 域套接字限制访问。
  - 定期更新 Python 依赖，执行安全扫描（如 `pip-audit`）。

## 运维与监控

- **监控指标**：
  - 使用 `Prometheus client + Grafana`（本地 Docker Desktop 或原生进程）监控 API 延迟、任务队列深度、Metal/MPS 利用率。
  - 对于纯离线用户，可使用 `Rich`/`Textual` 控制台展示实时指标。
- **告警**：
  - 队列堆积、内存占用过高、推理失败率超过阈值触发本地通知（macOS 通知中心）或邮件（离线暂存待联网发送）。
- **部署与更新**：
  - 使用 `Makefile`/`invoke` 自动化本地环境搭建（创建虚拟环境、安装依赖、下载模型）。
  - 桌面应用通过差分更新包（Sparkle/Tauri updater）分发，推理服务与模型可通过本地更新脚本同步。

