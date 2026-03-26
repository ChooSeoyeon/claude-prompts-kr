# 구글밋 자동 녹음 + 트랜스크립트 + 번역

구글밋 오디오를 자동 녹음하고 텍스트로 전사. 명령어 하나로 번역까지.

---

## 동작 방식

1. `record-meeting` — 시스템 오디오 + 마이크 동시 녹음
2. 종료(Ctrl+C) 후 Whisper 자동 실행 → `.txt` 트랜스크립트 저장
3. `/translate-meeting file.txt ko` — Claude가 한국어로 번역

**로컬에서 실행 (무료):**
- 오디오 녹음 (BlackHole + Python)
- 전사 (mlx-whisper, Apple Silicon GPU 활용)

**토큰 사용 (소량):**
- 번역 단계만 (~1시간 미팅 기준 $0.03~0.05)

---

## 사전 준비 (직접 해야 하는 단계)

### 1. BlackHole 설치 후 맥 재시작

```bash
brew install blackhole-2ch
```

**설치 후 반드시 맥 재시작.**

### 2. Audio MIDI Setup 설정 (GUI)

1. Spotlight(Cmd+Space) → "Audio MIDI Setup" 실행
2. 왼쪽 하단 `+` → "Create Multi-Output Device" 두 번 생성
3. 첫 번째 — 이름: `Meeting-AirPods`:
   - ✅ AirPods
   - ✅ BlackHole 2ch
4. 두 번째 — 이름: `Meeting-Speakers`:
   - ✅ MacBook Pro Speakers
   - ✅ BlackHole 2ch
5. 이름 변경은 디바이스 더블클릭
6. 기본 출력은 그대로 둬도 됨 (record-meeting 실행 시 자동 전환 + 종료 시 복구)

---

## 세팅

[prompt.md](./prompt.md) 내용을 Claude Code에 붙여넣으면 나머지는 Claude가 자동으로 처리.

---

## 사용법

```
미팅 시작 전:  record-meeting
미팅 끝나면:   Ctrl+C
번역 (한국어): /translate-meeting ~/meetings/meeting_파일명.txt ko
번역 (영어):   /translate-meeting ~/meetings/meeting_파일명.txt en
```
