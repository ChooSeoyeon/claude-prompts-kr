# 미팅 녹음 + 대본 추출 + 번역

시스템 오디오 + 마이크를 동시 녹음해 .mp3 파일로 저장. 자동으로 텍스트 변환 후 명령어 하나로 번역까지.

- **OBS 같은 별도 앱 불필요** — 터미널 명령어 하나로 가볍게 녹음
- **이어폰 연결 여부 자동 감지** — 에어팟이 연결되어 있어도 소리 누락 없이 정상 녹음
- **Clova 같은 외부 유료 툴 불필요** — Whisper 모델을 로컬에서 직접 실행해 무료로 텍스트 변환
- **Claude 토큰 거의 안씀** — 최초 세팅 시 1회, 이후엔 번역할 때만 소량 사용

---

## 세팅

[prompt.md](./prompt.md) 내용을 Claude Code에 붙여넣은 후 시키는 대로 하면 됩니다.

---

## 사용법

**터미널**
```
미팅 시작 전: record-meeting
미팅 끝나면:  Ctrl+C
```

**Claude Code**
```
번역본 보기: /translate-meeting ~/Meetings/txt/meeting_파일명.txt ko
원문 보기:   /translate-meeting ~/Meetings/txt/meeting_파일명.txt en
설정 변경:   /meeting-config
```

---

## 트러블슈팅

**Ctrl+C로 종료가 안 되는 경우**

새 터미널 탭에서 실행:
```
pkill -f record.py
```

---

**Zoom 회의 소리가 녹음되지 않는 경우**

Zoom은 자체 오디오 출력 설정이 있어 시스템 출력 전환을 무시할 수 있습니다.

Zoom → Settings → Audio → Speaker → **Same as System** 으로 변경 후 재시도하세요.

---

## 상세 동작 방식

세팅 후 바로 사용하실 수 있습니다. 내부 동작이 궁금하신 분만 참고하세요.

1. 터미널에서 `record-meeting` — 시스템 오디오 + 마이크 동시 녹음 → `.mp3` 저장
2. 녹음 종료(Ctrl+C) 후 Whisper 자동 실행 → `.txt` 텍스트 변환 (미팅 언어로 전사)
3. Claude Code에서 `/translate-meeting file.txt ko` 입력 → 설정한 언어로 번역
4. `/meeting-config` 로 미팅 언어 / 번역 언어 변경 가능

**로컬에서 실행 (무료):**
- 오디오 녹음 (BlackHole + Python)
- 텍스트 변환 (mlx-whisper, Apple Silicon GPU 활용)

**토큰 사용 (소량):**
- 번역 단계만 (1시간 미팅 기준 약 $0.03-0.05)
