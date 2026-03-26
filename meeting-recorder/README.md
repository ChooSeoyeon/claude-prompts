# Meeting Recorder + Transcript + Translation

Records system audio + microphone simultaneously and saves as .mp3. Automatically transcribes to text, then translates with a single command.

- **No OBS or separate apps needed** — record lightly with a single terminal command
- **Auto-detects headphone connection** — records correctly even with AirPods connected
- **No external paid tools** — runs Whisper locally for free transcription
- **Minimal Claude tokens** — only once for initial setup, then small usage for translation only

---

## Setup

Paste the contents of [prompt.md](./prompt.md) into Claude Code and follow the instructions.

---

## Usage

**Terminal**
```
Before meeting: record-meeting
When done:      Ctrl+C
```

**Claude Code**
```
View translation: /meeting-translate ~/Meetings/txt/meeting_filename.txt tr
View original:    /meeting-translate ~/Meetings/txt/meeting_filename.txt en
Change settings:  /meeting-config
```

---

## Troubleshooting

**Ctrl+C doesn't stop the recording**

Run in a new terminal tab:
```
pkill -f record.py
```

---

**Zoom audio not being recorded**

Zoom has its own audio output setting that can ignore system output changes.

Zoom → Settings → Audio → Speaker → change to **Same as System**.

---

## How It Works

You can start using it right after setup. This section is for those curious about the internals.

1. Run `record-meeting` in terminal — records system audio + mic simultaneously → saves as `.mp3`
2. After stopping (Ctrl+C), Whisper runs automatically → transcribes to `.txt` (in the meeting language)
3. Run `/meeting-translate file.txt tr` in Claude Code → translates to your configured language
4. Use `/meeting-config` to change meeting language / translation language

**Runs locally (free):**
- Audio recording (BlackHole + Python)
- Transcription (mlx-whisper, uses Apple Silicon GPU)

**Token usage (minimal):**
- Translation step only (approx. $0.03-0.05 per hour of meeting)
