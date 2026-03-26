# Meeting Recorder + Transcript + Translation Setup
`v1.4.0`

Steps 1–3 are performed by the user. Claude handles the rest automatically.

---

## Step 1: Install BlackHole (run in terminal, then restart Mac)

```
brew install blackhole-2ch
```

Restart your Mac after installation.

---

## Step 2: Audio MIDI Setup (manual - GUI)

1. Spotlight (Cmd+Space) → open "Audio MIDI Setup"
2. Click `+` at bottom left → "Create Multi-Output Device" twice to create two devices
3. First device (name: `Meeting-AirPods`):
   - ✅ AirPods
   - ✅ BlackHole 2ch
4. Second device (name: `Meeting-Speakers`):
   - ✅ MacBook Pro Speakers
   - ✅ BlackHole 2ch
5. Rename by double-clicking each device
6. Leave the default output as-is (record-meeting switches it automatically and restores on exit)

Proceed below when done.

---

## Step 3: Zoom Settings (manual)

If you use Zoom, make sure this setting is configured:

Zoom → Settings → Audio → Speaker → **Same as System**

Without this, Zoom audio won't be routed through BlackHole and won't be recorded.

Proceed below when done.

---

## Step 3.5: Set Default Translation Language

Ask the user for their preferred translation language:

> Please choose your translation language. (e.g. en English, ko Korean, ja Japanese, de German, tr Turkish, ru Russian)

Use the selected language code as `{DEFAULT_LANG}` in the steps below.

---

## Step 4: Automatic Installation (Claude handles this)

### 4-1. Install packages

- Auto-detect Python version (check python3 --version, brew python, etc. — use 3.8+)
- Install with detected Python:
  - `mlx-whisper` (uses M-chip GPU, fast)
  - `sounddevice`
- `brew install ffmpeg`
- `brew install switchaudio-osx`

### 4-2. Create ~/Meetings/ folder

### 4-3. Create ~/Meetings/record.py

Use the code below as-is. Replace the shebang line with the detected Python path, replace `{DEFAULT_LANG}` with the chosen language code, and `{DEFAULT_LANG_NAME}` with the language name (e.g. tr → Turkish, en → English, ko → Korean).

```python
#!/usr/bin/env python3
import sounddevice as sd
import numpy as np
import subprocess
import wave
import datetime
import os

BASE_DIR = os.path.expanduser("~/Meetings")
MP3_DIR = os.path.join(BASE_DIR, "mp3")
TXT_DIR = os.path.join(BASE_DIR, "txt")

def get_blackhole_index():
    for i, d in enumerate(sd.query_devices()):
        if 'BlackHole' in d['name'] and d['max_input_channels'] > 0:
            return i, int(d['default_samplerate']), d['max_input_channels']
    raise RuntimeError("BlackHole not found. Please check installation and restart.")

def get_mic_index():
    devices = sd.query_devices()
    # Prefer AirPods if connected
    for i, d in enumerate(devices):
        if 'AirPods' in d['name'] and d['max_input_channels'] > 0:
            return i
    # MacBook Pro Microphone
    for i, d in enumerate(devices):
        if 'MacBook Pro Microphone' in d['name'] and d['max_input_channels'] > 0:
            return i
    # fallback: first input device excluding Zoom/BlackHole
    for i, d in enumerate(devices):
        if d['max_input_channels'] > 0 and 'BlackHole' not in d['name'] and 'Zoom' not in d['name']:
            return i

os.makedirs(MP3_DIR, exist_ok=True)
os.makedirs(TXT_DIR, exist_ok=True)
timestamp = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
output_file = os.path.join(MP3_DIR, f"meeting_{timestamp}.mp3")
temp_wav = f"/tmp/meeting_{timestamp}.wav"

system_frames = []
mic_frames = []

def system_callback(indata, frames_count, time, status):
    system_frames.append(indata.copy())

def mic_callback(indata, frames_count, time, status):
    mic_frames.append(indata.copy())

BLACKHOLE_INDEX, SAMPLE_RATE, BH_CHANNELS = get_blackhole_index()
mic_index = get_mic_index()
mic_name = sd.query_devices(mic_index)['name']

# Save current output device (to restore after recording)
current_output = subprocess.run(["SwitchAudioSource", "-c"], capture_output=True, text=True).stdout.strip()

# Select Meeting device based on AirPods connection
airpods_connected = any(
    'AirPods' in d['name'] and d['max_input_channels'] > 0
    for d in sd.query_devices()
)
meeting_device = "Meeting-AirPods" if airpods_connected else "Meeting-Speakers"
result = subprocess.run(["SwitchAudioSource", "-s", meeting_device], capture_output=True)
if result.returncode == 0:
    print(f"Output device: {meeting_device} (auto-set)")
else:
    print(f"⚠️  Failed to switch to {meeting_device}. Set manually in Audio MIDI Setup.")

print(f"Recording started...")
print(f"Mic: {mic_name}")
print(f"Press Ctrl+C to stop\n")

with sd.InputStream(device=BLACKHOLE_INDEX, channels=BH_CHANNELS, samplerate=SAMPLE_RATE, callback=system_callback):
    with sd.InputStream(device=mic_index, channels=1, samplerate=SAMPLE_RATE, callback=mic_callback):
        try:
            while True:
                sd.sleep(1000)
        except KeyboardInterrupt:
            print("\nRecording stopped. Saving file...")

# Mix audio
system_audio = np.concatenate(system_frames) if system_frames else np.zeros((1, 2))
mic_audio = np.concatenate(mic_frames) if mic_frames else np.zeros((1, 1))

min_len = min(len(system_audio), len(mic_audio))
system_audio = system_audio[:min_len]
mic_audio = mic_audio[:min_len]

mic_stereo = np.column_stack([mic_audio[:, 0], mic_audio[:, 0]])
mixed = (system_audio + mic_stereo) / 2
mixed = np.clip(mixed, -1, 1)

# Save WAV then convert to MP3
with wave.open(temp_wav, 'w') as f:
    f.setnchannels(2)
    f.setsampwidth(2)
    f.setframerate(SAMPLE_RATE)
    f.writeframes((mixed * 32767).astype(np.int16).tobytes())

subprocess.run(["ffmpeg", "-i", temp_wav, output_file, "-y", "-loglevel", "quiet"])
os.remove(temp_wav)

# Restore output device
subprocess.run(["SwitchAudioSource", "-s", current_output], capture_output=True)
print(f"Output device: {current_output} (restored)")

# Run Whisper (mlx-whisper: uses M-chip GPU)
print(f"\nTranscribing with Whisper...")
subprocess.run(["mlx_whisper", output_file, "--language", "English", "--output-format", "txt", "--output-dir", TXT_DIR])

txt_file = os.path.join(TXT_DIR, f"meeting_{timestamp}.txt")
print(f"\nSaved: {txt_file}")
print(f"\n── Translate in Claude Code ──")
print(f"{DEFAULT_LANG_NAME} translation")
print(f"/meeting-translate {txt_file} {DEFAULT_LANG}")
print(f"\nView original")
print(f"/meeting-translate {txt_file} en")
```

### 4-4. Create ~/.claude/commands/meeting-translate.md

Create the file with the following content:

```
The user will provide a .txt file path and optionally a language code as arguments.

Usage:
- `/meeting-translate file.txt tr` → Turkish
- `/meeting-translate file.txt en` → English
- `/meeting-translate file.txt ko` → Korean
- Other language codes like ja, de, ru, el, etc. are also supported
- `/meeting-translate file.txt` (no language code) → use default language (ko)

Steps:

1. Parse $ARGUMENTS: first token is the txt file path, second token (if exists) is the language code. Default is ko.

2. Read the txt file directly.

3. Translate the full transcript to the target language word-for-word. Do not omit any part. Do not summarize. Show only the translation.

4. If the target language matches the source language (e.g. en→en), show the content as-is.
```

### 4-5. Create ~/.claude/commands/meeting-config.md

Create the file with the following content:

```
View and change meeting recording settings.

Steps:

1. Read `~/Meetings/record.py`.

2. Extract current settings:
   - Meeting language: `--language` value (e.g. English)
   - Translation language: language code from the translate-meeting output line (e.g. tr)

3. Show current settings:
   ── Current Meeting Settings ──
   Meeting language (Whisper): English
   Translation language: tr (Turkish)

4. Ask the user:
   > What would you like to change?
   > 1. Meeting language (language spoken in the meeting)
   > 2. Translation language (language Claude translates to)
   > 3. Both
   > 4. No changes

5. Based on the user's choice, ask for the new value(s) and update `~/Meetings/record.py` accordingly:
   - Meeting language: replace `--language` value + also update the language code in the "View original" command (e.g. English→en, Korean→ko, Japanese→ja, German→de, Turkish→tr, Russian→ru)
   - Translation language: replace the language code + label in the "translation" output line

6. Confirm the change:
   ── Settings Updated ──
   Meeting language: Korean
   Translation language: de (German)
```

### 4-7. Add alias to ~/.zshrc

Add the following alias using the detected Python path:
```
alias record-meeting="python3 ~/Meetings/record.py"
```

### 4-8. Print usage instructions

```
── Usage ──
Before meeting:  record-meeting
When done:       Ctrl+C → saves MP3 → restores audio → Whisper transcribes → saves txt
Translate:       /meeting-translate ~/Meetings/txt/meeting_filename.txt {DEFAULT_LANG}
Change settings: /meeting-config
```

Replace `{DEFAULT_LANG}` with the language code chosen in Step 3.5.
