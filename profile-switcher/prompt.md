# Claude Code 프로파일 스위처 세팅
`v1.0.0`

Claude Code를 여러 환경(회사, 팀, 개인 등)에서 서로 다른 설정으로 사용할 수 있도록 프로파일 전환 시스템을 구축한다.

아래 1단계는 사용자가 직접 수행한다. 이후 단계는 Claude가 자동으로 진행한다.

---

## 1단계: 프로파일 정의 (사용자가 직접)

사용할 프로파일 목록과 각 프로파일의 환경변수를 Claude에게 알려준다.

예시:
- `work`: ANTHROPIC_BASE_URL=https://my-proxy.example.com, ANTHROPIC_AUTH_TOKEN=sk-xxx
- `personal`: ANTHROPIC_AUTH_TOKEN=sk-yyy

완료 후 아래를 진행한다.

---

## 2단계: 자동 설치 및 설정 (Claude가 처리)

### 2-1. ~/.claude/profiles/ 폴더 생성

### 2-2. ~/.claude/profiles/base.json 생성

공통 설정 파일. 모든 프로파일에 적용된다. 아래 내용으로 생성한다:

```json
{}
```

> hooks, plugins 등 모든 프로파일에 공통으로 적용할 설정이 있으면 이 파일에 추가하면 된다.

### 2-3. 프로파일 JSON 파일 생성

1단계에서 사용자가 정의한 각 프로파일에 대해 `~/.claude/profiles/<이름>.json` 파일을 생성한다.

형식:
```json
{
  "env": {
    "KEY": "VALUE"
  }
}
```

### 2-4. ~/.claude/profiles/switch.sh 생성

아래 내용으로 생성 후 실행 권한(`chmod +x`) 부여:

```bash
#!/bin/bash

PROFILES_DIR="$HOME/.claude/profiles"
SETTINGS="$HOME/.claude/settings.json"

PROFILE="$1"

if [[ -z "$PROFILE" ]]; then
  echo "Usage: switch.sh <profile>"
  echo "Available profiles: $(ls $PROFILES_DIR/*.json 2>/dev/null | xargs -n1 basename | sed 's/\.json//' | grep -v base | tr '\n' ', ' | sed 's/,$//')"
  exit 1
fi

PROFILE_FILE="$PROFILES_DIR/$PROFILE.json"

if [[ ! -f "$PROFILE_FILE" ]]; then
  echo "Profile '$PROFILE' not found at $PROFILE_FILE"
  exit 1
fi

jq -s '.[0] * .[1]' "$PROFILES_DIR/base.json" "$PROFILE_FILE" > "$SETTINGS"

echo "Switched to '$PROFILE' profile"
```

### 2-5. ~/.zshrc에 alias 추가

1단계에서 정의한 각 프로파일에 대해 아래 형식으로 alias를 추가한다:

```
# Claude profile switcher
alias claude-<프로파일명>='~/.claude/profiles/switch.sh <프로파일명> && claude'
```

### 2-6. 완료 후 안내 출력

```
── 설정 완료 ──
프로파일 전환: 터미널에서 source ~/.zshrc 실행 후 사용

사용법:
  claude-<프로파일명>   settings.json을 해당 프로파일로 업데이트 후 Claude 실행
  claude               현재 settings.json 그대로 Claude 실행

공통 설정 수정: ~/.claude/profiles/base.json
프로파일 설정 수정: ~/.claude/profiles/<이름>.json
새 프로파일 추가: JSON 파일 생성 후 alias 추가
```
