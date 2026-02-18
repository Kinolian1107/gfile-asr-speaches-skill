# Google Drive ASR (faster-whisper) — GitHub Copilot Instructions

## What This Skill Does

Transcribes audio/video files from Google Drive links (or local paths) using a local faster-whisper server (speaches Docker container) with NVIDIA GPU acceleration. Outputs SRT subtitles with timestamps.

## When to Activate

When the user sends a Google Drive link or local file path and mentions any of:
轉逐字稿, 轉文字, transcribe, transcript, 語音轉文字, ASR, 字幕, subtitle, 摘要, summary, 分析

## Workflow

### 1. Ensure speaches container is running

```bash
docker ps --filter name=speaches --format '{{.Status}}'
# If not running:
sudo docker compose -f /opt/docker/docker-compose.yml up -d speaches
```

### 2. Download from Google Drive (if URL provided)

```bash
gdown "https://drive.google.com/uc?id={FILE_ID}" -O /home/kino/asr/{filename}
```

### 3. Extract audio (if video)

```bash
ffmpeg -i /home/kino/asr/{input} -vn -acodec pcm_s16le -ar 16000 -ac 1 /home/kino/asr/{basename}.wav -y
```

### 4. Transcribe with chunked processing

**CRITICAL: Always split audio into 15-second chunks before transcribing.** Whisper's VAD aggressively filters speech with natural pauses, causing missing text.

```bash
# Split into chunks
ffmpeg -y -i audio.wav -f segment -segment_time 15 -ar 16000 -ac 1 -acodec pcm_s16le /home/kino/asr/chunks/chunk_%03d.wav

# Transcribe each chunk
curl -s -X POST http://localhost:18996/v1/audio/transcriptions \
  -F "file=@chunk.wav" \
  -F "model=deepdml/faster-whisper-large-v3-turbo-ct2" \
  -F "response_format=verbose_json" \
  -F "language=zh" \
  -F "condition_on_previous_text=false" \
  -F "temperature=0"
```

### 5. Combine chunks into SRT

Parse each chunk's verbose_json response, adjust timestamps by chunk offset (chunk_index x 15 seconds), and build a combined SRT file.

### 6. Or use the helper script

```bash
bash scripts/transcribe.sh "/home/kino/asr/{filename}" zh srt
```

## Key Parameters

- Model: `deepdml/faster-whisper-large-v3-turbo-ct2`
- API: `http://localhost:18996/v1/audio/transcriptions`
- Chunk size: 15 seconds
- Default language: zh
- Default output: SRT
