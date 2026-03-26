Do the following automatically. Steps 1 and 2 have already been completed manually (BlackHole installed, Audio MIDI Setup configured with Meeting-AirPods and Meeting-Speakers devices).

## 1. Install packages

- Auto-detect Python version (3.8+ required)
- Install with detected Python:
  - `mlx-whisper`
  - `sounddevice`
- `brew install ffmpeg`
- `brew install switchaudio-osx`

## 2. Create ~/meetings/ folder

## 3. Create ~/meetings/record.py

Create this file exactly. Replace the shebang line with the detected Python path.

```python
#!/usr/bin/env python3
import sounddevice as sd
import numpy as np
import subprocess
import wave
import datetime
import os

OUTPUT_DIR = os.path.expanduser("~/meetings")

def get_blackhole_index():
    for i, d in enumerate(sd.query_devices()):
        if 'BlackHole' in d['name'] and d['max_input_channels'] > 0:
            return i, int(d['default_samplerate']), d['max_input_channels']
    raise RuntimeError("BlackHole not found. Please install and restart.")

def get_mic_index():
    devices = sd.query_devices()
    for i, d in enumerate(devices):
        if 'AirPods' in d['name'] and d['max_input_channels'] > 0:
            return i
    for i, d in enumerate(devices):
        if 'MacBook Pro Microphone' in d['name'] and d['max_input_channels'] > 0:
            return i
    for i, d in enumerate(devices):
        if d['max_input_channels'] > 0 and 'BlackHole' not in d['name'] and 'Zoom' not in d['name']:
            return i

os.makedirs(OUTPUT_DIR, exist_ok=True)
timestamp = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
output_file = os.path.join(OUTPUT_DIR, f"meeting_{timestamp}.mp3")
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

# Save current output device to restore later
current_output = subprocess.run(["SwitchAudioSource", "-c"], capture_output=True, text=True).stdout.strip()

# Auto-select Meeting device based on AirPods connection
airpods_connected = any(
    'AirPods' in d['name'] and d['max_input_channels'] > 0
    for d in sd.query_devices()
)
meeting_device = "Meeting-AirPods" if airpods_connected else "Meeting-Speakers"
result = subprocess.run(["SwitchAudioSource", "-s", meeting_device], capture_output=True)
if result.returncode == 0:
    print(f"Output: {meeting_device} (auto)")
else:
    print(f"⚠️  Failed to switch to {meeting_device}. Set manually in Audio MIDI Setup.")

print(f"Recording...")
print(f"Mic: {mic_name}")
print(f"Press Ctrl+C to stop\n")

with sd.InputStream(device=BLACKHOLE_INDEX, channels=BH_CHANNELS, samplerate=SAMPLE_RATE, callback=system_callback):
    with sd.InputStream(device=mic_index, channels=1, samplerate=SAMPLE_RATE, callback=mic_callback):
        try:
            while True:
                sd.sleep(1000)
        except KeyboardInterrupt:
            print("\nStopped. Saving...")

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

# Restore original output device
subprocess.run(["SwitchAudioSource", "-s", current_output], capture_output=True)
print(f"Output restored: {current_output}")

# Run Whisper automatically
print(f"\nTranscribing with Whisper...")
subprocess.run(["mlx_whisper", output_file, "--language", "English", "--output-format", "txt", "--output-dir", OUTPUT_DIR])

txt_file = output_file.replace(".mp3", ".txt")
print(f"\nSaved: {txt_file}")
print(f"\n── Translate in Claude Code ──")
print(f"Korean:  /translate-meeting {txt_file} ko")
print(f"English: /translate-meeting {txt_file} en")
```

## 4. Create ~/.claude/commands/translate-meeting.md

```
The user will provide a .txt file path and optionally a language flag as arguments.

Usage:
- `/translate-meeting file.txt` → translate to Korean (default)
- `/translate-meeting file.txt ko` → Korean
- `/translate-meeting file.txt en` → English transcript only

Steps:

1. Parse $ARGUMENTS: first token is the txt file path, second token (if exists) is the language flag (ko/en). Default is ko.

2. Read the txt file directly.

3. If language is `en`: show only the content as-is.

4. If language is `ko` (or default): translate the full transcript to Korean word-for-word. Do not summarize. Show only the Korean translation.
```

> Customize step 4 for your own language if needed (e.g. "translate to Japanese").

## 5. Add alias to ~/.zshrc

```
alias record-meeting="python3 ~/meetings/record.py"
```

## 6. Print usage summary

```
── Usage ──
Before meeting:  record-meeting
After meeting:   Ctrl+C → MP3 saved → output restored → Whisper transcribes → txt saved
Translate (KO):  /translate-meeting ~/meetings/meeting_FILENAME.txt ko
Translate (EN):  /translate-meeting ~/meetings/meeting_FILENAME.txt en
```
