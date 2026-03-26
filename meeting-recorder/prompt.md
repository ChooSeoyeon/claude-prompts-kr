# 미팅 녹음 + 한국어 번역 자동화 세팅
`v1.2.0`

아래 1~3단계는 직접 해야 해. 그 다음은 네가 다 자동으로 해줘.

---

## 1단계: BlackHole 설치 (터미널에서 직접 실행 후 맥 재시작)

```
brew install blackhole-2ch
```

설치 후 반드시 맥 재시작.

---

## 2단계: Audio MIDI Setup 설정 (수동 - GUI)

1. Spotlight(Cmd+Space) → "Audio MIDI Setup" 실행
2. 왼쪽 하단 `+` → "Create Multi-Output Device" 두 번 클릭해서 두 개 생성
3. 첫 번째 (이름: `Meeting-AirPods`):
   - ✅ AirPods
   - ✅ BlackHole 2ch
4. 두 번째 (이름: `Meeting-Speakers`):
   - ✅ MacBook Pro Speakers
   - ✅ BlackHole 2ch
5. 이름 변경은 디바이스 더블클릭해서 수정
6. 기본 출력은 평소에 그대로 써도 됨 (record-meeting 실행 시 자동 전환 + 종료 시 복구)

완료되면 아래를 진행해줘.

---

## 3단계: Zoom 사용자 설정 확인 (수동)

Zoom을 사용한다면 반드시 아래 설정을 확인해줘:

Zoom → Settings → Audio → Speaker → **Same as System** 으로 변경

이 설정이 되어 있지 않으면 Zoom 오디오가 BlackHole로 전달되지 않아 녹음되지 않음.

완료되면 아래를 진행해줘.

---

## 4단계: 자동 설치 및 설정 (Claude가 다 해줘)

### 4-1. 패키지 설치

- Python 버전 자동 감지 (python3 --version, brew python 등 확인해서 3.8 이상 사용)
- 감지한 Python으로 설치:
  - `mlx-whisper` (M칩 GPU 활용, 빠름)
  - `sounddevice`
- `brew install ffmpeg`
- `brew install switchaudio-osx`

### 4-2. ~/Meetings/ 폴더 생성

### 4-3. ~/Meetings/record.py 생성

아래 코드를 그대로 사용해서 파일 생성해줘. shebang 첫 줄은 감지한 Python 경로로 바꿔줘.

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
    raise RuntimeError("BlackHole을 찾을 수 없어요. 설치 및 재시작 확인해주세요.")

def get_mic_index():
    devices = sd.query_devices()
    # AirPods 연결된 경우 우선
    for i, d in enumerate(devices):
        if 'AirPods' in d['name'] and d['max_input_channels'] > 0:
            return i
    # MacBook Pro Microphone
    for i, d in enumerate(devices):
        if 'MacBook Pro Microphone' in d['name'] and d['max_input_channels'] > 0:
            return i
    # fallback: Zoom/BlackHole 제외한 첫 번째 input 디바이스
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

# 현재 출력 디바이스 저장 (녹음 후 복구용)
current_output = subprocess.run(["SwitchAudioSource", "-c"], capture_output=True, text=True).stdout.strip()

# 에어팟 연결 여부에 따라 Meeting 디바이스 선택
airpods_connected = any(
    'AirPods' in d['name'] and d['max_input_channels'] > 0
    for d in sd.query_devices()
)
meeting_device = "Meeting-AirPods" if airpods_connected else "Meeting-Speakers"
result = subprocess.run(["SwitchAudioSource", "-s", meeting_device], capture_output=True)
if result.returncode == 0:
    print(f"출력 디바이스: {meeting_device} (자동 설정)")
else:
    print(f"⚠️  {meeting_device} 전환 실패. Audio MIDI Setup에서 수동 설정 필요.")

print(f"녹음 시작...")
print(f"마이크: {mic_name}")
print(f"Ctrl+C로 중지\n")

with sd.InputStream(device=BLACKHOLE_INDEX, channels=BH_CHANNELS, samplerate=SAMPLE_RATE, callback=system_callback):
    with sd.InputStream(device=mic_index, channels=1, samplerate=SAMPLE_RATE, callback=mic_callback):
        try:
            while True:
                sd.sleep(1000)
        except KeyboardInterrupt:
            print("\n녹음 중지. 파일 생성 중...")

# 오디오 믹싱
system_audio = np.concatenate(system_frames) if system_frames else np.zeros((1, 2))
mic_audio = np.concatenate(mic_frames) if mic_frames else np.zeros((1, 1))

min_len = min(len(system_audio), len(mic_audio))
system_audio = system_audio[:min_len]
mic_audio = mic_audio[:min_len]

mic_stereo = np.column_stack([mic_audio[:, 0], mic_audio[:, 0]])
mixed = (system_audio + mic_stereo) / 2
mixed = np.clip(mixed, -1, 1)

# WAV 저장 후 MP3 변환
with wave.open(temp_wav, 'w') as f:
    f.setnchannels(2)
    f.setsampwidth(2)
    f.setframerate(SAMPLE_RATE)
    f.writeframes((mixed * 32767).astype(np.int16).tobytes())

subprocess.run(["ffmpeg", "-i", temp_wav, output_file, "-y", "-loglevel", "quiet"])
os.remove(temp_wav)

# 출력 디바이스 복구 (Whisper 돌리는 동안 소리 정상으로)
subprocess.run(["SwitchAudioSource", "-s", current_output], capture_output=True)
print(f"출력 디바이스: {current_output} (복구)")

# Whisper 자동 실행 (mlx-whisper: M칩 GPU 활용)
print(f"\nWhisper 전사 중...")
subprocess.run(["mlx_whisper", output_file, "--language", "English", "--output-format", "txt", "--output-dir", TXT_DIR])

txt_file = os.path.join(TXT_DIR, f"meeting_{timestamp}.txt")
print(f"\n저장 완료: {txt_file}")
print(f"\n── Claude Code에서 번역 ──")
print(f"한국어:")
print(f"/translate-meeting {txt_file} ko")
print(f"\n영어:")
print(f"/translate-meeting {txt_file} en")
```

### 4-4. ~/.claude/commands/translate-meeting.md 생성

아래 내용 그대로 파일 생성해줘:

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

### 4-5. ~/.zshrc에 alias 추가

감지한 Python 경로로 아래 alias 추가:
```
alias record-meeting="python3 ~/Meetings/record.py"
```

### 4-6. 완료 후 사용법 출력

```
── 사용법 ──
미팅 시작 전:  record-meeting
미팅 끝나면:   Ctrl+C → MP3 저장 → 출력 복구 → Whisper 자동 전사 → txt 저장
번역 (한국어): /translate-meeting ~/Meetings/meeting_파일명.txt ko
번역 (영어):   /translate-meeting ~/Meetings/meeting_파일명.txt en
```
