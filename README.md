# gfile-asr-speaches-skill

> **⚠️ DEPRECATED** — This repo has been merged into [openclaw-local-asr](https://github.com/Kinolian1107/openclaw-local-asr).
> Please use the unified repo for all future updates. This repo is kept for reference only.

[English](#english) | [中文](#中文)

---

## English

### What is this?

An AI-agent skill that transcribes audio/video files from Google Drive using a local **speaches** Docker container (faster-whisper + NVIDIA GPU). Features intelligent silence-based chunking to eliminate Whisper hallucinations.

### Features

- **Smart silence detection**: Uses ffmpeg `silencedetect` with adaptive threshold (auto-calibrates to audio volume)
- **Speech-segment extraction**: For silence-heavy audio (phone calls, interviews), only processes segments with actual speech
- **Hallucination filtering**: Detects and removes common Whisper hallucination patterns
- **GPU-accelerated**: faster-whisper large-v3-turbo with int8 quantization on NVIDIA GPUs
- **Multiple output formats**: SRT subtitles, plain text, JSON

### Prerequisites

| Component | Version | Purpose |
|-----------|---------|---------|
| Docker + Docker Compose | Latest | Run speaches container |
| NVIDIA GPU | Any with CUDA support | GPU acceleration |
| NVIDIA Container Toolkit | v1.18+ | GPU passthrough to Docker |
| ffmpeg | v5+ | Audio extraction, silence detection |
| Python | 3.10+ | Smart transcription script |
| gdown | Latest | Download from Google Drive |

### Installation

#### 1. Install NVIDIA Container Toolkit

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt update && sudo apt install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

#### 2. Deploy speaches Docker container

Add to your `docker-compose.yml`:

```yaml
services:
  speaches:
    image: ghcr.io/speaches-ai/speaches:latest-cuda-12.6.3
    container_name: speaches
    restart: unless-stopped
    ports:
      - "18996:8000"
    environment:
      - WHISPER__INFERENCE_DEVICE=cuda
      - WHISPER__COMPUTE_TYPE=int8
      - STT_MODEL_TTL=-1
      - LOG_LEVEL=info
      - PRELOAD_MODELS=["Systran/faster-whisper-large-v3-turbo"]
    volumes:
      - /path/to/huggingface-cache:/home/ubuntu/.cache/huggingface/hub
      - /path/to/asr-workdir:/asr
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
```

```bash
docker compose up -d speaches
```

#### 3. Install dependencies

```bash
# ffmpeg
sudo apt install -y ffmpeg
# or via Homebrew: brew install ffmpeg

# gdown (Google Drive downloader)
pip install gdown

# Verify
ffmpeg -version
gdown --version
curl http://localhost:18996/health
```

#### 4. Clone this skill

```bash
git clone https://github.com/Kinolian1107/gfile-asr-speaches-skill.git
```

### Usage

#### Command Line

```bash
# From Google Drive URL
python3 scripts/transcribe_smart.py "https://drive.google.com/file/d/xxx/view" --lang zh

# From local file
python3 scripts/transcribe_smart.py /path/to/audio.mp3 --lang zh

# English audio
python3 scripts/transcribe_smart.py /path/to/audio.mp3 --lang en

# Custom max chunk size
python3 scripts/transcribe_smart.py /path/to/audio.mp3 --lang zh --max-chunk 120
```

#### With AI Agents

**OpenClaw:**
```bash
ln -s /path/to/gfile-asr-speaches-skill ~/.openclaw/skills/gfile-asr-speaches
```
Then send a Google Drive link in Telegram with "轉逐字稿".

**Cursor / Claude Code / Gemini CLI / Codex:**
Clone this repo into your project. The agent will read `SKILL.md` / `CLAUDE.md` / `GEMINI.md` / `AGENTS.md` automatically.

### Architecture

```
Google Drive URL
    ↓ gdown
Local audio/video file
    ↓ ffmpeg (extract audio if video)
WAV (16kHz, mono)
    ↓ ffmpeg silencedetect (adaptive threshold)
Speech segments identified
    ↓ Group into smart chunks
    ↓ (shorter chunks for high-silence audio)
Speaches API (faster-whisper, GPU)
    ↓ Hallucination filter
SRT / TXT / JSON output
```

---

## 中文

### 這是什麼？

一個 AI 代理技能，可以從 Google Drive 下載音訊/影片檔案，使用本地的 **speaches** Docker 容器（faster-whisper + NVIDIA GPU）進行語音轉逐字稿。具有智慧靜音切段功能，消除 Whisper 幻覺問題。

### 特色

- **智慧靜音偵測**：使用 ffmpeg `silencedetect`，自動根據音訊音量校準門檻值
- **語音段落提取**：對高靜音比例的音訊（通話錄音、訪談），只處理有語音的段落
- **幻覺過濾**：偵測並移除 Whisper 常見的幻覺模式（重複文字、字幕浮水印等）
- **GPU 加速**：faster-whisper large-v3-turbo + int8 量化 + NVIDIA GPU
- **多種輸出格式**：SRT 字幕、純文字、JSON

### 前置需求

| 元件 | 版本 | 用途 |
|------|------|------|
| Docker + Docker Compose | 最新版 | 執行 speaches 容器 |
| NVIDIA GPU | 任何支援 CUDA 的顯卡 | GPU 加速 |
| NVIDIA Container Toolkit | v1.18+ | GPU 透傳到 Docker |
| ffmpeg | v5+ | 音訊提取、靜音偵測 |
| Python | 3.10+ | 智慧轉逐字稿腳本 |
| gdown | 最新版 | Google Drive 下載 |

### 安裝

#### 1. 安裝 NVIDIA Container Toolkit

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt update && sudo apt install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

#### 2. 部署 speaches Docker 容器

在 `docker-compose.yml` 中加入：

```yaml
services:
  speaches:
    image: ghcr.io/speaches-ai/speaches:latest-cuda-12.6.3
    container_name: speaches
    restart: unless-stopped
    ports:
      - "18996:8000"
    environment:
      - WHISPER__INFERENCE_DEVICE=cuda
      - WHISPER__COMPUTE_TYPE=int8
      - STT_MODEL_TTL=-1
      - LOG_LEVEL=info
      - PRELOAD_MODELS=["Systran/faster-whisper-large-v3-turbo"]
    volumes:
      - /path/to/huggingface-cache:/home/ubuntu/.cache/huggingface/hub
      - /path/to/asr-workdir:/asr
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
```

```bash
docker compose up -d speaches
```

#### 3. 安裝相依套件

```bash
# ffmpeg
sudo apt install -y ffmpeg

# gdown（Google Drive 下載工具）
pip install gdown

# 驗證
ffmpeg -version
gdown --version
curl http://localhost:18996/health
```

#### 4. Clone 此技能

```bash
git clone https://github.com/Kinolian1107/gfile-asr-speaches-skill.git
```

### 使用方式

#### 命令列

```bash
# 從 Google Drive URL
python3 scripts/transcribe_smart.py "https://drive.google.com/file/d/xxx/view" --lang zh

# 從本地檔案
python3 scripts/transcribe_smart.py /path/to/audio.mp3 --lang zh

# 英語音訊
python3 scripts/transcribe_smart.py /path/to/audio.mp3 --lang en
```

#### 搭配 AI 代理

**OpenClaw：**
```bash
ln -s /path/to/gfile-asr-speaches-skill ~/.openclaw/skills/gfile-asr-speaches
```
然後在 Telegram 中傳送 Google Drive 連結並說「轉逐字稿」。

**Cursor / Claude Code / Gemini CLI / Codex：**
Clone 此 repo 到你的專案目錄。代理會自動讀取對應的指示檔案。

### 架構

```
Google Drive URL
    ↓ gdown
本地音訊/影片檔案
    ↓ ffmpeg（影片時提取音軌）
WAV（16kHz, 單聲道）
    ↓ ffmpeg silencedetect（自適應門檻）
識別語音段落
    ↓ 分組成智慧切段
    ↓（高靜音音訊使用更短的切段）
Speaches API（faster-whisper, GPU）
    ↓ 幻覺過濾
SRT / TXT / JSON 輸出
```

## License

MIT
