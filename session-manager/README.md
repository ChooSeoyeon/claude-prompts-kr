# Claude 세션 관리 명령어

Claude Code 세션을 검색하고, 이어가고, 다른 프로젝트로 가져오는 커스텀 명령어 3개.

- `/session-info` — 현재 세션 ID 확인 + 삭제 명령어
- `/search-session` — 현재 프로젝트의 세션 내용 검색 + resume 명령어
- `/import-session` — 다른 프로젝트의 세션을 현재 프로젝트로 복사

---

## 세팅

[prompt.md](./prompt.md) 내용을 Claude Code에 붙여넣으면 세 파일이 자동으로 생성됩니다.

---

## 사용법

### 현재 세션 확인

입력:
```
/session-info
```

결과:
```
세션 ID: 69d9529b-df37-4745-a92c-5dc384e59576

다른 프로젝트에서 이 세션 이어가려면 (다른 프로젝트로 이동 후 claude 안에서 실행):
/import-session 69d9529b-df37-4745-a92c-5dc384e59576

이 세션 삭제하려면 (Ctrl+C로 Claude 종료 후 터미널에서 실행):
rm ~/.claude/projects/-Users-yourname-myrepo/69d9529b-df37-4745-a92c-5dc384e59576.jsonl
```

### 세션 내용 검색

입력:
```
/search-session
→ http 서버 / 최근7일
```

결과:
```
1. 69d9529b | 2026-03-26 17:41
   미리보기: "...http 서버 띄우는 방법 알아보다가..."
   이어가려면 (터미널에서 실행):
   claude --resume 69d9529b-df37-4745-a92c-5dc384e59576
```

### 다른 프로젝트 세션 가져오기

입력:
```
/import-session 69d9529b-df37-4745-a92c-5dc384e59576
```

결과:
```
✓ 세션 복사 완료
from: /Users/yourname/other-project
to:   ~/.claude/projects/-Users-yourname-myrepo/

이어가려면 (터미널에서 실행):
claude --resume 69d9529b-df37-4745-a92c-5dc384e59576
```
