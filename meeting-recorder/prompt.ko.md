아래를 자동으로 실행해줘. 1, 2단계는 이미 수동으로 완료됨 (BlackHole 설치, Audio MIDI Setup에서 Meeting-AirPods, Meeting-Speakers 디바이스 설정).

## 1. 패키지 설치

- Python 버전 자동 감지 (3.8 이상 필요)
- 감지된 Python으로 설치:
  - `mlx-whisper`
  - `sounddevice`
- `brew install ffmpeg`
- `brew install switchaudio-osx`

## 2. ~/meetings/ 폴더 생성

## 3. ~/meetings/record.py 생성

아래 내용 그대로 생성. 셔뱅(shebang) 줄은 감지된 Python 경로로 교체.

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

# 종료 후 원래 출력 디바이스 복구를 위해 현재 값 저장
current_output = subprocess.run(["SwitchAudioSource", "-c"], capture_output=True, text=True).stdout.strip()

# 에어팟 연결 여부에 따라 Meeting 디바이스 자동 선택
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

# 오디오 믹싱
system_audio = np.concatenate(system_frames) if system_frames else np.zeros((1, 2))
mic_audio = np.concatenate(mic_frames) if mic_frames else np.zeros((1, 1))

min_len = min(len(system_audio), len(mic_audio))
system_audio = system_audio[:min_len]
mic_audio = mic_audio[:min_len]

mic_stereo = np.column_stack([mic_audio[:, 0], mic_audio[:, 0]])
mixed = (system_audio + mic_stereo) / 2
mixed = np.clip(mixed, -1, 1)

# WAV 저장 후 MP3로 변환
with wave.open(temp_wav, 'w') as f:
    f.setnchannels(2)
    f.setsampwidth(2)
    f.setframerate(SAMPLE_RATE)
    f.writeframes((mixed * 32767).astype(np.int16).tobytes())

subprocess.run(["ffmpeg", "-i", temp_wav, output_file, "-y", "-loglevel", "quiet"])
os.remove(temp_wav)

# 원래 출력 디바이스로 복구
subprocess.run(["SwitchAudioSource", "-s", current_output], capture_output=True)
print(f"Output restored: {current_output}")

# Whisper 자동 실행
print(f"\nTranscribing with Whisper...")
subprocess.run(["mlx_whisper", output_file, "--language", "English", "--output-format", "txt", "--output-dir", OUTPUT_DIR])

txt_file = output_file.replace(".mp3", ".txt")
print(f"\nSaved: {txt_file}")
print(f"\n── Claude Code에서 번역 ──")
print(f"한국어: /translate-meeting {txt_file} ko")
print(f"영어:   /translate-meeting {txt_file} en")
```

## 4. ~/.claude/commands/translate-meeting.md 생성

```
The user will provide a .txt file path and optionally a language flag as arguments.

Usage:
- `/translate-meeting file.txt` → 한국어 번역 (default)
- `/translate-meeting file.txt ko` → 한국어 번역
- `/translate-meeting file.txt en` → 영어 트랜스크립트만

Steps:

1. Parse $ARGUMENTS: first token is the txt file path, second token (if exists) is the language flag (ko/en). Default is ko.

2. Read the txt file directly.

3. If language is `en`: show only the content as-is.

4. If language is `ko` (or default): translate the full transcript to Korean word-for-word (한 토씨도 빼지 말고 전부 번역). Do not summarize. Show only the Korean translation.
```

## 5. ~/.zshrc에 alias 추가

```
alias record-meeting="python3 ~/meetings/record.py"
```

## 6. 사용법 출력

```
── 사용법 ──
미팅 시작 전:  record-meeting
미팅 끝나면:   Ctrl+C → MP3 저장 → 출력 복구 → Whisper 전사 → txt 저장
번역 (한국어): /translate-meeting ~/meetings/meeting_파일명.txt ko
번역 (영어):   /translate-meeting ~/meetings/meeting_파일명.txt en
```
