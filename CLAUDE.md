# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Pixelle-Video 是一个 AI 驱动的视频自动生成平台。用户输入主题或脚本，系统自动完成文案创作、图像/视频生成、语音合成和最终视频合成。

## Common Commands

```bash
# 安装依赖（使用 UV 包管理器）
uv sync

# 启动 Web UI（Streamlit，端口 8501）
uv run streamlit run web/app.py

# 启动 API 服务（FastAPI，端口 8000）
uv run python api/app.py

# 运行测试
uv run pytest -v

# 代码检查和格式化
uv run ruff check pixelle_video/ api/ web/
uv run ruff format pixelle_video/ api/ web/

# Docker 启动
docker-compose up -d
```

## Architecture

### 整体分层

```
Streamlit Web UI (web/)
    ↓ 直接调用 或 HTTP 请求
FastAPI Backend (api/)
    ↓
PixelleVideoCore (pixelle_video/service.py)  ← 核心服务单例
    ↓
Services Layer (pixelle_video/services/)
    ↓
External APIs: ComfyUI / RunningHub / LLM / TTS
```

### 核心入口：`pixelle_video/service.py`

`PixelleVideoCore` 是整个系统的服务编排器，提供统一访问入口：
- `pixelle_video.llm` — LLM 服务（OpenAI SDK 兼容）
- `pixelle_video.tts` — TTS 语音合成
- `pixelle_video.media` — 图像/视频生成
- `pixelle_video.video` — 视频合成（基于 FFmpeg）
- `pixelle_video.persistence` — 任务历史存储
- `pixelle_video.pipelines` — 已注册的视频生成流水线字典

### 视频生成流水线（`pixelle_video/pipelines/`）

- **StandardPipeline**：主流程，输入主题 → AI 生成文案 → 逐帧生成图像+TTS → 合成视频片段 → 拼接+BGM
- **CustomPipeline**：用户自定义工作流模板
- **AssetBasedPipeline**：基于已有素材（图片/视频）生成
- **LinearPipeline**（`linear.py`）：模板方法基类，其他流水线继承它

### 配置系统（`pixelle_video/config/`）

- `schema.py` — Pydantic 模型定义（所有配置项的数据结构）
- `loader.py` — YAML 配置文件读取
- `manager.py` — 单例 ConfigManager，启动时加载 `config.yaml`
- 运行时配置文件为 `config.yaml`（从 `config.example.yaml` 复制后修改）

### ComfyUI 集成

通过 `comfykit` 库驱动工作流。工作流 JSON 存放在 `workflows/` 目录：
- `workflows/selfhost/` — 本地 ComfyUI 服务器工作流
- `workflows/runninghub/` — 云端 RunningHub 工作流

`pixelle_video/services/comfy_base_service.py` 是 ComfyUI 服务基类。

### Web 前端（`web/`）

Streamlit 多页面应用：
- `web/app.py` — 主入口
- `web/pages/` — 页面（主生成界面、历史记录页）
- `web/components/` — 可复用 UI 组件（设置面板、输入表单、样式配置等）
- `web/state/session.py` — Streamlit session state 管理
- `web/pipelines/` — Web 层流水线实现（对接后端流水线）

### API 后端（`api/`）

FastAPI 应用，支持同步和异步视频生成：
- `api/app.py` — 主应用
- `api/routers/` — 路由模块（llm、tts、image、video、tasks、files 等）
- `api/tasks/` — 异步任务管理队列
- `api/schemas/` — Pydantic 请求/响应模型

## Key Configuration

`config.yaml` 中的关键配置区块：
- `llm` — API 密钥、base_url、模型名称（支持 OpenAI/Qwen/DeepSeek/Claude/Ollama）
- `comfyui` — 本地 ComfyUI 地址、RunningHub API 密钥、TTS/图像/视频默认工作流路径
- `template` — 默认视频帧 HTML 模板路径（存放在 `templates/` 目录）

LLM provider 预设定义在 `pixelle_video/llm_presets.py`，TTS 语音选项在 `pixelle_video/tts_voices.py`。

## Templates

HTML 模板用于渲染视频帧（通过 html2image 转图片），按分辨率组织：
- `templates/1080x1920/` — 竖版
- `templates/1080x1080/` — 方形
- `templates/1920x1080/` — 横版

## Tech Stack

- **Python 3.11+**，UV 包管理
- `fastapi` + `streamlit` — 后端 API 与前端 UI
- `openai` SDK — LLM 调用（兼容多厂商）
- `comfykit` — ComfyUI 工作流集成
- `edge-tts` — TTS 合成
- `ffmpeg-python` + `moviepy` — 视频处理
- `html2image` — HTML 帧渲染
- `pydantic` — 配置与数据校验
- `ruff` — linting 和格式化
- `pytest` + `pytest-asyncio` — 测试框架
