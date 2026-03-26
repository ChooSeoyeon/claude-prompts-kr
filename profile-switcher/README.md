# Claude Code 프로파일 스위처

Claude Code를 여러 환경(회사 LLM 프록시, 팀별 설정 등)에서 서로 다른 설정으로 사용할 수 있는 프로파일 전환 시스템.

- **환경별 API 엔드포인트/토큰 분리** — 회사 프록시, 개인 계정 등 별도 관리
- **공통 설정 한 곳에서 관리** — hooks, plugins 등은 base.json 하나로
- **명령어 하나로 전환** — `claude-work`, `claude-personal` 등 alias로 즉시 전환

---

## 세팅

[prompt.md](./prompt.md) 내용을 Claude Code에 붙여넣은 후 시키는 대로 하면 됩니다.

---

## 사용법

```
claude-<프로파일명>   settings.json을 해당 프로파일로 업데이트 후 Claude 실행
claude               현재 settings.json 그대로 Claude 실행
```

---

## 구조

```
~/.claude/profiles/
├── base.json          # 공통 설정 (hooks, plugins 등)
├── <프로파일명>.json  # 프로파일별 env 설정
└── switch.sh          # 전환 스크립트
```

`switch.sh`는 `base.json + <프로파일>.json`을 머지해서 `~/.claude/settings.json`으로 저장.
