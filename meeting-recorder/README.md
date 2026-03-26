# 미팅 자동 녹음 + 텍스트 변환 + 번역

시스템 오디오 + 마이크를 동시 녹음해 .mp3 파일로 저장. 자동으로 텍스트 변환 후 명령어 하나로 번역까지.

---

## 동작 방식

1. `record-meeting` — 시스템 오디오 + 마이크 동시 녹음 → `.mp3` 저장
2. 녹음 종료(Ctrl+C) 후 Whisper 자동 실행 → `.txt` 텍스트 변환
3. (영어 미팅이라면) Claude Code에서 `/translate-meeting file.txt ko` 입력 → Claude가 한국어로 번역

**로컬에서 실행 (무료):**
- 오디오 녹음 (BlackHole + Python)
- 텍스트 변환 (mlx-whisper, Apple Silicon GPU 활용)

**토큰 사용 (소량):**
- 번역 단계만 (1시간 미팅 기준 약 $0.03-0.05)

---

## 세팅

[prompt.md](./prompt.md) 내용을 Claude Code에 붙여넣은 후 시키는 대로 하면 됩니다.

---

## 사용법

```
미팅 시작 전:        record-meeting
미팅 끝나면:         Ctrl+C
번역 (Claude Code): /translate-meeting ~/meetings/meeting_파일명.txt ko
```
