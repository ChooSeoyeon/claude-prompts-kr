# Claude 세션 관리 명령어 설치
`v1.1.0`

아래 세 개의 커스텀 명령어 파일을 `~/.claude/commands/`에 생성한다.

---

## 1. `~/.claude/commands/session-info.md`

````
현재 세션 ID를 출력한다.

1. 현재 프로젝트의 ~/.claude/projects/ 폴더에서 가장 최근에 수정된 JSONL 파일을 찾는다:
   ```
   ls -t ~/.claude/projects/<encoded-pwd>/*.jsonl | head -1
   ```
   encoded-pwd: pwd 결과에서 `/`를 `-`로 변환

2. 파일명(확장자 제외)이 현재 세션 ID다. 아래 형식으로만 출력:
   ```
   세션 ID: <uuid>

   다른 프로젝트에서 이 세션 이어가려면 (다른 프로젝트로 이동 후 claude 안에서 실행):
   /session-import <uuid>

   이 세션 삭제하려면 (Ctrl+C로 Claude 종료 후 터미널에서 실행):
   rm ~/.claude/projects/<encoded-pwd>/<uuid>.jsonl
   ```
````

---

## 2. `~/.claude/commands/session-import.md`

````
다른 프로젝트의 세션 파일을 현재 프로젝트로 복사해온다.

인자: $ARGUMENTS (세션 ID 또는 없음)

## 인자가 없을 때

1. 모든 프로젝트 목록을 보여준다:
   ```
   ls ~/.claude/projects/
   ```
   각 폴더명의 `-`를 `/`로 바꿔서 사람이 읽을 수 있는 경로로 표시.

2. 사용자에게 소스 프로젝트를 선택하게 한다.

3. 선택된 프로젝트의 최근 세션 목록을 보여준다:
   ```
   ls -lt ~/.claude/projects/<선택된-프로젝트>/*.jsonl | head -10
   ```

4. 사용자에게 가져올 세션 ID를 선택하게 한다.

## 세션 ID가 주어졌을 때 (또는 선택 완료 후)

1. 현재 프로젝트 경로 계산:
   - `pwd` 결과에서 `/`를 `-`로 변환

2. 소스 파일 탐색 (모든 프로젝트에서 해당 세션 ID 검색):
   ```
   ls ~/.claude/projects/*/<session-id>.jsonl 2>/dev/null
   ```

3. 현재 프로젝트 폴더로 복사:
   ```
   cp <소스경로>/<session-id>.jsonl ~/.claude/projects/<현재-프로젝트>/
   ```

4. 결과 출력:
   ```
   ✓ 세션 복사 완료
   from: <소스 프로젝트>
   to:   ~/.claude/projects/<현재-프로젝트>/

   resume하려면: claude --resume
   ```
````

---

## 3. `~/.claude/commands/session-search.md`

````
Search through conversation files in the current project by content.

1. Ask the user in a single prompt:
   - 검색어 (필수)
   - 기간 (선택): "오늘", "최근 N일", "YYYY-MM-DD 이후" — 생략 시 전체 검색
   예: "검색어를 입력하세요 (예: 'http 서버' 또는 'http 서버 / 최근7일')"

2. Determine the current project folder:
   - encoded-pwd: current working directory with `/` replaced by `-`
   - project dir: `~/.claude/projects/<encoded-pwd>/`

3. Find matching `.jsonl` files in that project dir, filtered by date if specified:
   - If "오늘": `find ~/.claude/projects/<encoded-pwd>/ -name "*.jsonl" -mtime -1`
   - If "최근 N일": `find ~/.claude/projects/<encoded-pwd>/ -name "*.jsonl" -mtime -N`
   - If "YYYY-MM-DD 이후": on macOS use `touch -t YYYYMMDD0000 /tmp/ref_date` then `find ~/.claude/projects/<encoded-pwd>/ -name "*.jsonl" -newer /tmp/ref_date`
   - If omitted: all `.jsonl` files in the project dir

4. Search for the term **only in user messages**. Use python3 to parse JSONL and filter:
   ```
   python3 -c "
   import json, sys, re
   pattern = sys.argv[1]
   for line in open(sys.argv[2]):
       try:
           obj = json.loads(line)
           # Only check user-role messages (not assistant, not thinking, not tool results)
           msg = obj.get('message', {})
           if msg.get('role') != 'user':
               continue
           content = msg.get('content', '')
           if isinstance(content, list):
               text = ' '.join(c.get('text', '') for c in content if isinstance(c, dict) and c.get('type') == 'text')
           else:
               text = str(content)
           if re.search(pattern, text, re.IGNORECASE):
               print(text[:200])
       except: pass
   " "<검색어>" <file>
   ```
   Run this for each candidate file to find matches.

5. For each matching file, show a snippet of the matched user message (the text printed by the python3 command above).

6. Display results as a numbered list:
   ```
   1. [세션 ID] | [수정일시]
      미리보기: "...매칭된 내용..."
      이어가려면 (터미널에서 실행):
      claude --resume <uuid>
   ```

7. If no results: "검색 결과 없음".
````
