# Claude Code 완전 정복 - 단축 가이드

![헤더: Anthropic 해커톤 우승 - Claude Code 팁 & 트릭](../../assets/images/shortform/00-header.png)

---

**2월 실험적 롤아웃 이후 Claude Code를 매일 사용해왔으며, [@DRodriguezFX](https://x.com/DRodriguezFX)와 함께 Claude Code만으로 [zenith.chat](https://zenith.chat)을 만들어 Anthropic x Forum Ventures 해커톤에서 우승했습니다.**

10개월간의 매일 사용 끝에 완성된 나의 전체 설정을 공개합니다: 스킬, 훅, 서브에이전트, MCP, 플러그인, 그리고 실제로 효과 있는 것들.

---

## 스킬과 커맨드

스킬은 특정 범위와 워크플로우에 제한된 규칙처럼 작동합니다. 특정 워크플로우를 실행해야 할 때 프롬프트를 단축해주는 역할을 합니다.

Opus 4.5로 긴 코딩 세션 후 죽은 코드와 불필요한 .md 파일을 정리하고 싶다면 `/refactor-clean`을 실행하세요. 테스팅이 필요하다면 `/tdd`, `/e2e`, `/test-coverage`를 사용하세요. 스킬에는 코드맵도 포함될 수 있는데, 이를 통해 Claude가 컨텍스트를 소모하지 않고 코드베이스를 빠르게 탐색할 수 있습니다.

![터미널에서 커맨드 체이닝](../../assets/images/shortform/02-chaining-commands.jpeg)
*커맨드 체이닝 예시*

커맨드는 슬래시 커맨드를 통해 실행되는 스킬입니다. 겹치는 부분이 있지만 저장 위치가 다릅니다:

- **스킬**: `~/.claude/skills/` - 더 넓은 워크플로우 정의
- **커맨드**: `~/.claude/commands/` - 빠르게 실행 가능한 프롬프트

```bash
# 스킬 구조 예시
~/.claude/skills/
  pmx-guidelines.md      # 프로젝트별 패턴
  coding-standards.md    # 언어 모범 사례
  tdd-workflow/          # README.md가 있는 다중 파일 스킬
  security-review/       # 체크리스트 기반 스킬
```

---

## 훅

훅은 특정 이벤트에 반응하는 트리거 기반 자동화입니다. 스킬과 달리 도구 호출 및 라이프사이클 이벤트에만 제한됩니다.

**훅 유형:**

1. **PreToolUse** - 도구 실행 전 (유효성 검사, 알림)
2. **PostToolUse** - 도구 완료 후 (포매팅, 피드백 루프)
3. **UserPromptSubmit** - 메시지 전송 시
4. **Stop** - Claude 응답 완료 시
5. **PreCompact** - 컨텍스트 압축 전
6. **Notification** - 권한 요청 시

**예시: 장시간 실행 커맨드 전 tmux 알림**

```json
{
  "PreToolUse": [
    {
      "matcher": "tool == \"Bash\" && tool_input.command matches \"(npm|pnpm|yarn|cargo|pytest)\"",
      "hooks": [
        {
          "type": "command",
          "command": "if [ -z \"$TMUX\" ]; then echo '[Hook] 세션 지속성을 위해 tmux 사용을 고려하세요' >&2; fi"
        }
      ]
    }
  ]
}
```

![PostToolUse 훅 피드백](../../assets/images/shortform/03-posttooluse-hook.png)
*PostToolUse 훅 실행 중 Claude Code에서 받는 피드백 예시*

**팁:** JSON을 직접 작성하는 대신 `hookify` 플러그인을 사용하면 대화 형식으로 훅을 만들 수 있습니다. `/hookify`를 실행하고 원하는 것을 설명하면 됩니다.

---

## 서브에이전트

서브에이전트는 오케스트레이터(메인 Claude)가 제한된 범위로 작업을 위임할 수 있는 프로세스입니다. 백그라운드 또는 포그라운드에서 실행되어 메인 에이전트의 컨텍스트를 확보해줍니다.

서브에이전트는 스킬과 잘 어우러집니다. 스킬의 일부를 실행할 수 있는 서브에이전트에게 작업을 위임하면 해당 스킬을 자율적으로 사용할 수 있습니다. 특정 도구 권한으로 샌드박스화할 수도 있습니다.

```bash
# 서브에이전트 구조 예시
~/.claude/agents/
  planner.md           # 기능 구현 계획
  architect.md         # 시스템 설계 결정
  tdd-guide.md         # 테스트 주도 개발
  code-reviewer.md     # 품질/보안 검토
  security-reviewer.md # 취약점 분석
  build-error-resolver.md
  e2e-runner.md
  refactor-cleaner.md
```

서브에이전트별로 허용 도구, MCP, 권한을 설정하여 적절한 범위를 지정하세요.

---

## 규칙과 메모리

`.rules` 폴더에는 Claude가 항상 따라야 할 모범 사례가 담긴 `.md` 파일들이 있습니다. 두 가지 접근 방식이 있습니다:

1. **단일 CLAUDE.md** - 모든 것을 한 파일에 (사용자 또는 프로젝트 레벨)
2. **규칙 폴더** - 관심사별로 모듈화된 `.md` 파일

```bash
~/.claude/rules/
  security.md      # 하드코딩된 비밀 없음, 입력 유효성 검사
  coding-style.md  # 불변성, 파일 구성
  testing.md       # TDD 워크플로우, 80% 커버리지
  git-workflow.md  # 커밋 형식, PR 프로세스
  agents.md        # 서브에이전트 위임 시기
  performance.md   # 모델 선택, 컨텍스트 관리
```

**규칙 예시:**

- 코드베이스에 이모지 사용 금지
- 프론트엔드에서 보라색 계열 사용 자제
- 배포 전 항상 코드 테스트
- 거대 파일보다 모듈식 코드 우선
- console.log 커밋 금지

---

## MCP (모델 컨텍스트 프로토콜)

MCP는 Claude를 외부 서비스에 직접 연결합니다. API의 대체가 아닙니다. 프롬프트 기반 래퍼로서 정보 탐색에 더 많은 유연성을 제공합니다.

**예시:** Supabase MCP를 사용하면 Claude가 복사-붙여넣기 없이 특정 데이터를 가져오고 SQL을 직접 실행할 수 있습니다. 데이터베이스, 배포 플랫폼 등에도 동일하게 적용됩니다.

![Supabase MCP 테이블 목록](../../assets/images/shortform/04-supabase-mcp.jpeg)
*Supabase MCP가 public 스키마 내 테이블을 나열하는 예시*

**Claude의 Chrome:** 내장 플러그인 MCP로, Claude가 브라우저를 자율적으로 조작할 수 있어 작동 방식을 직접 확인할 수 있습니다.

**중요: 컨텍스트 윈도우 관리**

MCP는 엄선해서 사용하세요. 모든 MCP를 사용자 설정에 보관하되 **사용하지 않는 것은 모두 비활성화**합니다. `/plugins`로 이동하여 스크롤하거나 `/mcp`를 실행하세요.

![/plugins 인터페이스](../../assets/images/shortform/05-plugins-interface.jpeg)
*/plugins를 사용하여 현재 설치된 MCP와 상태 확인*

너무 많은 도구가 활성화되면 200k 컨텍스트 윈도우가 압축 전 70k밖에 남지 않을 수 있습니다. 성능이 크게 저하됩니다.

**원칙:** 설정에 20-30개의 MCP를 두되, 활성화는 10개 이하 / 도구는 80개 이하로 유지.

```bash
# 활성화된 MCP 확인
/mcp

# ~/.claude.json의 projects.disabledMcpServers에서 미사용 항목 비활성화
```

---

## 플러그인

플러그인은 수동 설정 없이 도구를 쉽게 설치할 수 있도록 패키지화합니다. 플러그인은 스킬 + MCP의 조합이거나 훅/도구 묶음일 수 있습니다.

**플러그인 설치:**

```bash
# 마켓플레이스 추가
# @mixedbread-ai의 mgrep 플러그인
claude plugin marketplace add https://github.com/mixedbread-ai/mgrep

# Claude 열기, /plugins 실행, 새 마켓플레이스 찾기, 그곳에서 설치
```

![mgrep을 보여주는 마켓플레이스 탭](../../assets/images/shortform/06-marketplaces-mgrep.jpeg)
*새로 설치된 Mixedbread-Grep 마켓플레이스 표시*

**LSP 플러그인**은 편집기 외부에서 Claude Code를 자주 실행하는 경우 특히 유용합니다. 언어 서버 프로토콜을 통해 IDE 없이도 실시간 타입 검사, 정의로 이동, 지능형 완성 기능을 제공합니다.

```bash
# 활성화된 플러그인 예시
typescript-lsp@claude-plugins-official  # TypeScript 인텔리전스
pyright-lsp@claude-plugins-official     # Python 타입 검사
hookify@claude-plugins-official         # 대화형 훅 생성
mgrep@Mixedbread-Grep                   # ripgrep보다 향상된 검색
```

MCP와 동일한 주의사항 - 컨텍스트 윈도우를 주시하세요.

---

## 팁과 트릭

### 키보드 단축키

- `Ctrl+U` - 전체 줄 삭제 (백스페이스 연타보다 빠름)
- `!` - 빠른 bash 커맨드 접두사
- `@` - 파일 검색
- `/` - 슬래시 커맨드 시작
- `Shift+Enter` - 여러 줄 입력
- `Tab` - 사고 과정 표시 토글
- `Esc Esc` - Claude 중단 / 코드 복원

### 병렬 워크플로우

- **Fork** (`/fork`) - 대화를 분기하여 겹치지 않는 작업을 병렬로 수행 (메시지 큐 대기 없이)
- **Git Worktrees** - 충돌 없이 Claude 인스턴스를 병렬 실행. 각 워크트리는 독립적인 체크아웃

```bash
git worktree add ../feature-branch feature-branch
# 각 워크트리에서 별도의 Claude 인스턴스 실행
```

### 장시간 실행 커맨드를 위한 tmux

Claude가 실행하는 로그/bash 프로세스를 스트리밍하고 모니터링합니다:

```bash
tmux new -s dev
# Claude가 여기서 커맨드를 실행하면 분리했다가 다시 연결 가능
tmux attach -t dev
```

### mgrep > grep

`mgrep`은 ripgrep/grep보다 크게 개선되었습니다. 플러그인 마켓플레이스에서 설치 후 `/mgrep` 스킬을 사용하세요. 로컬 검색과 웹 검색 모두 지원합니다.

```bash
mgrep "function handleSubmit"  # 로컬 검색
mgrep --web "Next.js 15 app router changes"  # 웹 검색
```

### 기타 유용한 커맨드

- `/rewind` - 이전 상태로 돌아가기
- `/statusline` - 브랜치, 컨텍스트 %, 할 일 목록으로 커스터마이즈
- `/checkpoints` - 파일 레벨 실행 취소 지점
- `/compact` - 수동으로 컨텍스트 압축 트리거

### GitHub Actions CI/CD

GitHub Actions로 PR에 코드 리뷰를 설정하세요. 설정 완료 후 Claude가 자동으로 PR을 검토합니다.

![Claude 봇이 PR 승인](../../assets/images/shortform/08-github-pr-review.jpeg)
*Claude가 버그 수정 PR을 승인하는 모습*

### 샌드박싱

위험한 작업에는 샌드박스 모드를 사용하세요. Claude가 실제 시스템에 영향을 주지 않는 제한된 환경에서 실행됩니다.

---

## 편집기에 대하여

편집기 선택은 Claude Code 워크플로우에 상당한 영향을 미칩니다. Claude Code는 어떤 터미널에서도 작동하지만, 유능한 편집기와 함께 사용하면 실시간 파일 추적, 빠른 탐색, 통합 커맨드 실행이 가능해집니다.

### Zed (개인 선호)

저는 [Zed](https://zed.dev)를 사용합니다. Rust로 작성되어 진짜로 빠릅니다. 즉시 열리고, 거대한 코드베이스도 무리 없이 처리하며, 시스템 리소스를 거의 사용하지 않습니다.

**Zed + Claude Code가 좋은 조합인 이유:**

- **속도** - Rust 기반 성능으로 Claude가 빠르게 파일을 편집할 때 편집기가 따라갑니다
- **에이전트 패널 통합** - Zed의 Claude 통합으로 Claude가 편집할 때 파일 변경사항을 실시간으로 추적. 편집기를 벗어나지 않고 Claude가 참조하는 파일 간 이동 가능
- **CMD+Shift+R 커맨드 팔레트** - 모든 커스텀 슬래시 커맨드, 디버거, 빌드 스크립트에 검색 가능한 UI로 빠른 접근
- **최소한의 리소스 사용** - 무거운 작업 중 Claude와 RAM/CPU를 경쟁하지 않습니다. Opus 실행 시 중요합니다
- **Vim 모드** - 원하신다면 완전한 vim 키 바인딩 지원

![커스텀 커맨드가 있는 Zed 편집기](../../assets/images/shortform/09-zed-editor.jpeg)
*CMD+Shift+R로 커스텀 커맨드 드롭다운이 있는 Zed 편집기. 우측 하단 과녁 표시가 팔로잉 모드.*

**편집기 공통 팁:**

1. **화면 분할** - 한쪽에 Claude Code 터미널, 다른 쪽에 편집기
2. **Ctrl + G** - Zed에서 Claude가 현재 작업 중인 파일 빠르게 열기
3. **자동 저장** - Claude의 파일 읽기가 항상 최신 상태가 되도록 자동 저장 활성화
4. **Git 통합** - 편집기의 git 기능으로 커밋 전 Claude의 변경사항 검토
5. **파일 감시자** - 대부분의 편집기는 변경된 파일을 자동으로 다시 로드합니다. 활성화되어 있는지 확인하세요

### VSCode / Cursor

이것도 좋은 선택이며 Claude Code와 잘 작동합니다. `\ide`를 사용한 자동 동기화로 터미널 형식으로 사용하거나 (플러그인과 어느 정도 중복되지만 LSP 기능 포함), 편집기와 더 잘 통합되고 일치하는 UI를 가진 확장 프로그램을 선택할 수 있습니다.

![VS Code Claude Code 확장](../../assets/images/shortform/10-vscode-extension.jpeg)
*VS Code 확장은 IDE에 직접 통합된 네이티브 그래픽 인터페이스를 제공합니다.*

---

## 나의 설정

### 플러그인

**설치됨:** (보통 한 번에 4-5개만 활성화)

```markdown
ralph-wiggum@claude-code-plugins       # 루프 자동화
frontend-design@claude-code-plugins    # UI/UX 패턴
commit-commands@claude-code-plugins    # Git 워크플로우
security-guidance@claude-code-plugins  # 보안 검사
pr-review-toolkit@claude-code-plugins  # PR 자동화
typescript-lsp@claude-plugins-official # TS 인텔리전스
hookify@claude-plugins-official        # 훅 생성
code-simplifier@claude-plugins-official
feature-dev@claude-code-plugins
explanatory-output-style@claude-code-plugins
code-review@claude-code-plugins
context7@claude-plugins-official       # 실시간 문서
pyright-lsp@claude-plugins-official    # Python 타입
mgrep@Mixedbread-Grep                  # 향상된 검색
```

### MCP 서버

**설정됨 (사용자 레벨):**

```json
{
  "github": { "command": "npx", "args": ["-y", "@modelcontextprotocol/server-github"] },
  "firecrawl": { "command": "npx", "args": ["-y", "firecrawl-mcp"] },
  "supabase": {
    "command": "npx",
    "args": ["-y", "@supabase/mcp-server-supabase@latest", "--project-ref=YOUR_REF"]
  },
  "memory": { "command": "npx", "args": ["-y", "@modelcontextprotocol/server-memory"] },
  "sequential-thinking": {
    "command": "npx",
    "args": ["-y", "@modelcontextprotocol/server-sequential-thinking"]
  },
  "vercel": { "type": "http", "url": "https://mcp.vercel.com" },
  "railway": { "command": "npx", "args": ["-y", "@railway/mcp-server"] },
  "cloudflare-docs": { "type": "http", "url": "https://docs.mcp.cloudflare.com/mcp" },
  "cloudflare-workers-bindings": {
    "type": "http",
    "url": "https://bindings.mcp.cloudflare.com/mcp"
  },
  "clickhouse": { "type": "http", "url": "https://mcp.clickhouse.cloud/mcp" },
  "AbletonMCP": { "command": "uvx", "args": ["ableton-mcp"] },
  "magic": { "command": "npx", "args": ["-y", "@magicuidesign/mcp@latest"] }
}
```

핵심은 이것입니다 - 14개의 MCP를 설정했지만 프로젝트별로 5-6개만 활성화합니다. 컨텍스트 윈도우를 건강하게 유지하는 방법입니다.

### 주요 훅

```json
{
  "PreToolUse": [
    { "matcher": "npm|pnpm|yarn|cargo|pytest", "hooks": ["tmux 알림"] },
    { "matcher": "Write && .md 파일", "hooks": ["README/CLAUDE가 아니면 차단"] },
    { "matcher": "git push", "hooks": ["검토를 위해 편집기 열기"] }
  ],
  "PostToolUse": [
    { "matcher": "Edit && .ts/.tsx/.js/.jsx", "hooks": ["prettier --write"] },
    { "matcher": "Edit && .ts/.tsx", "hooks": ["tsc --noEmit"] },
    { "matcher": "Edit", "hooks": ["console.log 경고 grep"] }
  ],
  "Stop": [
    { "matcher": "*", "hooks": ["수정된 파일에서 console.log 확인"] }
  ]
}
```

### 커스텀 상태 표시줄

사용자, 디렉토리, dirty 인디케이터가 있는 git 브랜치, 남은 컨텍스트 %, 모델, 시간, 할 일 수를 표시합니다:

![커스텀 상태 표시줄](../../assets/images/shortform/11-statusline.jpeg)
*Mac 루트 디렉토리에서의 상태 표시줄 예시*

```
affoon:~ ctx:65% Opus 4.5 19:52
▌▌ plan mode on (shift+tab to cycle)
```

### 규칙 구조

```
~/.claude/rules/
  security.md      # 필수 보안 검사
  coding-style.md  # 불변성, 파일 크기 제한
  testing.md       # TDD, 80% 커버리지
  git-workflow.md  # 컨벤셔널 커밋
  agents.md        # 서브에이전트 위임 규칙
  patterns.md      # API 응답 형식
  performance.md   # 모델 선택 (Haiku vs Sonnet vs Opus)
  hooks.md         # 훅 문서
```

### 서브에이전트

```
~/.claude/agents/
  planner.md           # 기능 분해
  architect.md         # 시스템 설계
  tdd-guide.md         # 테스트 먼저 작성
  code-reviewer.md     # 품질 검토
  security-reviewer.md # 취약점 스캔
  build-error-resolver.md
  e2e-runner.md        # Playwright 테스트
  refactor-cleaner.md  # 죽은 코드 제거
  doc-updater.md       # 문서 동기화 유지
```

---

## 핵심 요약

1. **과도하게 복잡하게 만들지 마세요** - 설정을 아키텍처가 아닌 미세 조정처럼 취급하세요
2. **컨텍스트 윈도우는 소중합니다** - 미사용 MCP와 플러그인 비활성화
3. **병렬 실행** - 대화 분기, git 워크트리 사용
4. **반복 작업 자동화** - 포매팅, 린팅, 알림을 위한 훅
5. **서브에이전트 범위 지정** - 제한된 도구 = 집중된 실행

---

## 참고 자료

- [플러그인 참고](https://code.claude.com/docs/en/plugins-reference)
- [훅 문서](https://code.claude.com/docs/en/hooks)
- [체크포인팅](https://code.claude.com/docs/en/checkpointing)
- [인터랙티브 모드](https://code.claude.com/docs/en/interactive-mode)
- [메모리 시스템](https://code.claude.com/docs/en/memory)
- [서브에이전트](https://code.claude.com/docs/en/sub-agents)
- [MCP 개요](https://code.claude.com/docs/en/mcp-overview)

---

**참고:** 이것은 상세 내용의 일부입니다. 고급 패턴은 [롱폼 가이드](./the-longform-guide.md)를 참조하세요.

---

*[@DRodriguezFX](https://x.com/DRodriguezFX)와 함께 [zenith.chat](https://zenith.chat)을 만들어 뉴욕에서 열린 Anthropic x Forum Ventures 해커톤에서 우승했습니다*
