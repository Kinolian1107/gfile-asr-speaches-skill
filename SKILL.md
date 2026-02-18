---
name: gfile-asr-speaches
description: >
  Download audio/video files from Google Drive and transcribe them locally using
  ffmpeg silence-detection + speaches Docker (faster-whisper with GPU).
  Outputs SRT subtitles, plain text, or JSON.
  Triggers on keywords: 轉逐字稿, 轉文字, transcribe, transcript, 語音轉文字, ASR, 字幕, subtitle.
metadata:
  openclaw:
    emoji: "🎙️"
    requires:
      bins: ["ffmpeg", "curl", "gdown", "python3"]
    os: ["linux"]
---

# Google Drive ASR — Speaches Mode

Transcribe audio/video from Google Drive using **ffmpeg silence-detection + speaches Docker API** (faster-whisper, GPU-accelerated).

## How It Works

1. **ffmpeg silencedetect** finds silence boundaries with adaptive threshold (based on mean volume)
2. Extracts speech segments and groups them into smart chunks (max 5 min each)
3. Sends each chunk to **speaches Docker API** (faster-whisper large-v3-turbo, int8, CUDA)
4. Filters hallucinated segments (repeated text, subtitle watermarks, etc.)
5. Combines results with corrected timestamps into SRT

This eliminates the hallucination problem in silence-heavy audio (e.g., phone calls with 70%+ silence).

## Trigger Conditions

Activate when ANY of the following are true:

1. User sends a **Google Drive link** + mentions: 轉逐字稿, 轉文字, transcribe, transcript, 語音轉文字, ASR, 字幕, subtitle, 摘要, summary, 分析
2. User provides a **local file path** to audio/video and asks for transcription
3. User says "transcribe" or "轉逐字稿" referencing a previously downloaded file

## Prerequisites

Verify speaches Docker container is running:

```bash
docker ps --filter name=speaches --format '{{.Status}}'
```

If not running:

```bash
sudo docker compose -f /opt/docker/docker-compose.yml up -d speaches
```

## Workflow

**CRITICAL: Run ALL steps in sequence without stopping. Do NOT wait for user prompts between steps. The entire pipeline must complete autonomously. Proactively report results via Telegram when done.**

### Step 1: Download from Google Drive

```bash
gdown "https://drive.google.com/uc?id={FILE_ID}" -O /home/kino/asr/{filename}
```

### Step 2: Run Smart Transcription

```bash
python3 "${SKILL_DIR}/scripts/transcribe_smart.py" \
    /home/kino/asr/{filename} --lang zh --format srt
```

The script handles everything automatically:
- Detects file type (audio/video)
- Converts to WAV if needed (ffmpeg)
- Uses ffmpeg silencedetect with adaptive threshold to find speech segments
- Groups speech segments into smart chunks (shorter for high-silence audio)
- Sends chunks to speaches API
- Filters hallucinated segments
- Merges results into SRT with correct timestamps

### Step 3: Report Results & Deliver via Telegram

**IMPORTANT: After transcription completes, you MUST proactively notify the user.**

1. Copy the SRT file to workspace:
   ```bash
   cp /home/kino/asr/{basename}.srt /home/kino/.openclaw/workspace/{basename}.srt
   ```

2. Send via Telegram using the `message` tool:
   ```
   action: send
   channel: telegram
   message: "轉寫完成！{basename}.srt（{duration}s，{n_segments} 條字幕）"
   filePath: /home/kino/.openclaw/workspace/{basename}.srt
   ```

3. Display summary in conversation:
   ```
   轉寫完成！
   📁 {basename}.srt
   ⏱️ {duration}s
   🔤 語言：{language}
   📨 已透過 Telegram 傳送
   ```

4. If user also asked for 摘要/分析, proceed to analyze the transcript.

## Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `SPEACHES_URL` | `http://localhost:18996` | speaches API endpoint |
| `ASR_MODEL` | `deepdml/faster-whisper-large-v3-turbo-ct2` | Whisper model |
| `ASR_DIR` | `/home/kino/asr` | Working directory |
| `--max-chunk` | `300` | Max chunk seconds (auto-reduces for high-silence audio) |
| `--lang` | `zh` | Language code (zh, en, ja, auto) |

## Supported Input

- **Audio**: MP3, WAV, M4A, FLAC, OGG, AAC, WMA
- **Video**: MP4, MKV, AVI, MOV, WebM, FLV
- **Sources**: Google Drive links, local file paths

## Known Issues & Solutions

| Issue | Solution |
|-------|----------|
| VAD too aggressive on silence-heavy audio | Adaptive silence detection + speech-segment extraction |
| Whisper hallucinations | Hallucination pattern filter + `temperature=0` + `condition_on_previous_text=false` |
| Language detection errors | Always specify `--lang zh` for Chinese content |
| Docker needs sudo | Some environments require `sudo docker` |

## References

- [speaches (faster-whisper-server)](https://github.com/speaches-ai/speaches)
- [faster-whisper](https://github.com/SYSTRAN/faster-whisper)
