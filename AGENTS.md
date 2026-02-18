# AGENTS.md — gfile-asr-speaches-skill

ASR pipeline: Google Drive → ffmpeg silence-detection → speaches API (faster-whisper GPU) → SRT.

## Usage

When the user sends a Google Drive link and asks for transcription (轉逐字稿, 轉文字, transcribe, ASR, etc.):

1. Ensure speaches Docker container is running: `docker ps --filter name=speaches`
2. Run: `python3 scripts/transcribe_smart.py <file_or_url> --lang zh`
3. Output: SRT/TXT/JSON files in `/home/kino/asr/`

## Key Features

- Adaptive silence detection (auto-adjusts threshold based on audio volume)
- Speech-segment extraction for high-silence audio (avoids Whisper hallucinations)
- Hallucination filtering (repeated text, subtitle watermarks)
- Smart chunking at silence boundaries (never cuts mid-sentence)

## CRITICAL

- Run the entire pipeline without stopping — do NOT wait for user input between steps
- Proactively deliver the SRT file via Telegram after completion
