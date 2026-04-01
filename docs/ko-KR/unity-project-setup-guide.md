# Unity 프로젝트에 ECC 적용 가이드

> 모든 설정을 유저 레벨(`~/.claude/`)에 두고, dotfiles로 관리하여 여러 컴퓨터에서 동일한 환경을 유지하는 방법.

---

## 최종 디렉토리 구조

```
~/.claude/                           ← 유저 레벨 설정 (dotfiles로 관리)
├── settings.json                    ← 훅 설정
├── rules/                           ← Claude 행동 지침
│   ├── csharp/
│   ├── cpp/                         ← 네이티브 플러그인 작업 시
│   └── common/
├── agents/                          ← 서브에이전트
├── commands/                        ← 슬래시 커맨드
├── skills/                          ← 워크플로우 스킬
└── scripts/                         ← 훅이 참조하는 Node.js 스크립트

<Unity 프로젝트 루트>/
└── CLAUDE.md                        ← 프로젝트별 지침 (Claude가 가장 먼저 읽는 파일)
```

---

## 1단계 — CLAUDE.md 생성

각 Unity 프로젝트 루트에 `CLAUDE.md`를 만든다. Claude가 세션 시작 시 가장 먼저 읽는 파일이다.

```markdown
# CLAUDE.md

## 프로젝트 개요
<!-- Unity 프로젝트 이름, 목적, 주요 기술 스택 기술 -->

## 빌드 환경
- Unity 버전: (예: 2022.3.x LTS)
- 타겟 플랫폼: (예: Android, iOS, Windows)
- 네이티브 플러그인: C++ (있는 경우)

## 주요 경로
- Assets/Scripts/        ← C# 스크립트
- Assets/Plugins/        ← 네이티브 플러그인 (.dll, .so, .dylib)
- Assets/Tests/          ← 유니티 테스트

## 작업 규칙
- C# 코딩: ~/.claude/rules/csharp/ 참고
- 네이티브 플러그인: ~/.claude/rules/cpp/ 참고
```

---

## 2단계 — 구성요소 설치

저장소를 클론한 뒤 필요한 구성요소를 `~/.claude/`에 복사한다.

```bash
# 저장소 클론
git clone https://github.com/affaan-m/everything-claude-code.git
cd everything-claude-code
```
```bash
# Rules 복사 (common + C# + C++)
cp -r rules/common/* ~/.claude/rules/
cp -r rules/csharp/* ~/.claude/rules/
cp -r rules/cpp/* ~/.claude/rules/         # 네이티브 플러그인 사용 시

# Agents 복사
cp agents/*.md ~/.claude/agents/

# Commands 복사
cp commands/*.md ~/.claude/commands/

# Skills 복사
cp -r skills/* ~/.claude/skills/

# Scripts 복사 (훅 실행에 필요)
cp -r scripts/* ~/.claude/scripts/
```

필요한 것만 선택적으로 복사해도 된다. 예를 들어 에이전트를 골라서 넣으려면:

```bash
# 에이전트 선택 복사
cp agents/code-reviewer.md ~/.claude/agents/
cp agents/cpp-build-resolver.md ~/.claude/agents/
cp agents/cpp-reviewer.md ~/.claude/agents/
cp agents/tdd-guide.md ~/.claude/agents/
cp agents/planner.md ~/.claude/agents/
cp agents/build-error-resolver.md ~/.claude/agents/
```

---

## 3단계 — settings.json 작성

`~/.claude/settings.json`을 생성한다. ECC 레포의 `hooks/hooks.json` 내용을 기반으로 **필요한 훅만 선택**해서 넣는다.

`${CLAUDE_PLUGIN_ROOT}`는 환경변수로 자동 치환된다 (8단계 참고).

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{ "type": "command", "command": "npx block-no-verify@1.1.2" }],
        "description": "git hook 우회 플래그 차단"
      },
      {
        "matcher": "Bash",
        "hooks": [{ "type": "command", "command": "node \"${CLAUDE_PLUGIN_ROOT}/scripts/hooks/run-with-flags.js\" \"pre:bash:git-push-reminder\" \"scripts/hooks/pre-bash-git-push-reminder.js\" \"standard\"" }],
        "description": "push 전 검토 알림"
      },
      {
        "matcher": "Bash",
        "hooks": [{ "type": "command", "command": "node \"${CLAUDE_PLUGIN_ROOT}/scripts/hooks/run-with-flags.js\" \"pre:bash:commit-quality\" \"scripts/hooks/pre-bash-commit-quality.js\" \"standard\"" }],
        "description": "커밋 전 품질 검사"
      },
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [{ "type": "command", "command": "node \"${CLAUDE_PLUGIN_ROOT}/scripts/hooks/run-with-flags.js\" \"pre:config-protection\" \"scripts/hooks/config-protection.js\" \"standard\"" }],
        "description": "설정 파일 보호"
      }
    ],
    "SessionStart": [
      {
        "matcher": "*",
        "hooks": [{ "type": "command", "command": "node -e \"const fs=require('fs');const path=require('path');const {spawnSync}=require('child_process');const raw=fs.readFileSync(0,'utf8');const rel=path.join('scripts','hooks','run-with-flags.js');const hasRunnerRoot=candidate=>{const value=typeof candidate==='string'?candidate.trim():'';return value.length>0&&fs.existsSync(path.join(path.resolve(value),rel));};const root=(()=>{const envRoot=process.env.CLAUDE_PLUGIN_ROOT||'';if(hasRunnerRoot(envRoot))return path.resolve(envRoot.trim());const home=require('os').homedir();const claudeDir=path.join(home,'.claude');if(hasRunnerRoot(claudeDir))return claudeDir;return claudeDir;})();const script=path.join(root,rel);if(fs.existsSync(script)){const result=spawnSync(process.execPath,[script,'session:start','scripts/hooks/session-start.js','standard'],{input:raw,encoding:'utf8',env:process.env,cwd:process.cwd(),timeout:30000});const stdout=typeof result.stdout==='string'?result.stdout:'';if(stdout)process.stdout.write(stdout);else process.stdout.write(raw);if(result.stderr)process.stderr.write(result.stderr);process.exit(Number.isInteger(result.status)?result.status:0);}process.stdout.write(raw);\"" }],
        "description": "세션 시작 시 이전 컨텍스트 로드"
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write|MultiEdit",
        "hooks": [{ "type": "command", "command": "node \"${CLAUDE_PLUGIN_ROOT}/scripts/hooks/run-with-flags.js\" \"post:quality-gate\" \"scripts/hooks/quality-gate.js\" \"standard\"", "async": true, "timeout": 30 }],
        "description": "파일 편집 후 품질 게이트"
      }
    ],
    "Stop": [
      {
        "matcher": "*",
        "hooks": [{ "type": "command", "command": "node -e \"const fs=require('fs');const path=require('path');const {spawnSync}=require('child_process');const raw=fs.readFileSync(0,'utf8');const rel=path.join('scripts','hooks','run-with-flags.js');const hasRunnerRoot=candidate=>{const value=typeof candidate==='string'?candidate.trim():'';return value.length>0&&fs.existsSync(path.join(path.resolve(value),rel));};const root=(()=>{const envRoot=process.env.CLAUDE_PLUGIN_ROOT||'';if(hasRunnerRoot(envRoot))return path.resolve(envRoot.trim());const home=require('os').homedir();return path.join(home,'.claude');})();const script=path.join(root,rel);if(fs.existsSync(script)){const result=spawnSync(process.execPath,[script,'stop:session-end','scripts/hooks/session-end.js','standard'],{input:raw,encoding:'utf8',env:process.env,cwd:process.cwd(),timeout:30000});const stdout=typeof result.stdout==='string'?result.stdout:'';if(stdout)process.stdout.write(stdout);else process.stdout.write(raw);if(result.stderr)process.stderr.write(result.stderr);process.exit(Number.isInteger(result.status)?result.status:0);}process.stdout.write(raw);\"", "async": true, "timeout": 10 }],
        "description": "세션 종료 시 상태 저장"
      }
    ]
  }
}
```

#### 훅 선택 가이드

| 훅 ID | 설명 | 권장 |
|-------|------|------|
| `pre:bash:commit-quality` | 커밋 전 품질 검사 | 필수 |
| `pre:bash:git-push-reminder` | push 전 검토 알림 | 권장 |
| `pre:config-protection` | 설정 파일 보호 | 권장 |
| `post:quality-gate` | 파일 편집 후 품질 게이트 | 권장 |
| `session:start` | 이전 세션 컨텍스트 로드 | 권장 |
| `stop:session-end` | 세션 상태 저장 | 권장 |
| `stop:cost-tracker` | 토큰/비용 추적 | 선택 |
| `pre:bash:tmux-reminder` | tmux 알림 | **제외** (Windows) |
| `post:edit:typecheck` | TypeScript 검사 | **제외** (C#과 무관) |
| `post:edit:format` | JS/TS 포매터 | **제외** (C#과 무관) |

> 전체 훅 목록 및 설명: [원본 저장소 hooks/hooks.json](https://github.com/affaan-m/everything-claude-code/blob/main/hooks/hooks.json)

---

## 4단계 — 환경변수 설정

dotfiles에 다음 환경변수를 추가한다 (예: `.bashrc`, `.zshrc`, `.profile` 등).

```bash
# 훅 스크립트 경로 기준점
export CLAUDE_PLUGIN_ROOT="$HOME/.claude"

# 훅 프로파일: minimal | standard | strict (기본값: standard)
export ECC_HOOK_PROFILE=standard

# Unity 프로젝트에 불필요한 훅 비활성화
export ECC_DISABLED_HOOKS=pre:bash:tmux-reminder,post:edit:typecheck,post:edit:format

# 기본 브랜치 이름
export DEFAULT_BASE_BRANCH=main
```

Windows의 경우 시스템 환경변수에 등록하거나, Git Bash의 `~/.bashrc`에 추가한다.

---

## 5단계 — 검증

Claude Code를 프로젝트 폴더에서 열고 다음을 확인한다:

1. **CLAUDE.md 인식** — 세션 시작 시 프로젝트 지침이 로드되는지 확인
2. **슬래시 커맨드** — `/tdd`, `/code-review` 입력 시 자동완성에 표시되는지 확인
3. **Rules 적용** — C# 파일 편집 요청 시 `~/.claude/rules/csharp/` 규칙을 따르는지 확인
4. **훅 동작** — 커밋 명령 시 `pre:bash:commit-quality` 훅이 실행되는지 확인

---

## 요약

| 구성요소 | 위치 | 필수 여부 |
|----------|------|-----------|
| 프로젝트 지침 | `<프로젝트>/CLAUDE.md` | 필수 (프로젝트별) |
| 환경변수 | dotfiles (`.bashrc` 등) | 필수 |
| 훅 설정 | `~/.claude/settings.json` | 필수 |
| 훅 스크립트 | `~/.claude/scripts/` | 필수 |
| Rules | `~/.claude/rules/csharp/`, `cpp/`, `common/` | 필수 |
| Agents | `~/.claude/agents/` | 권장 |
| Commands | `~/.claude/commands/` | 권장 |
| Skills | `~/.claude/skills/` | 권장 |
| MCP 설정 | `~/.claude/settings.json`의 `mcpServers` 키 | 선택 |
