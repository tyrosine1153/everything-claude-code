# CLAUDE.md

> **원본 저장소:** [affaan-m/everything-claude-code](https://github.com/affaan-m/everything-claude-code)
> 이 브랜치(`for-unity`)는 원본 ECC 플러그인을 Unity/C# 프로젝트에 적용하기 위해 커스터마이징한 것이다.

## 프로젝트 개요

이 저장소는 **Everything Claude Code(ECC)** 플러그인의 Unity 특화 포크다. 원본은 범용 에이전트 하네스 성능 최적화 시스템이며, 여기서는 Unity/C# + C++ 네이티브 플러그인 개발에 필요한 구성요소만 선별한다.

적용 가이드: [docs/ko-KR/unity-project-setup-guide.md](docs/ko-KR/unity-project-setup-guide.md)

## 테스트 실행

```bash
# 전체 테스트 (36개 파일)
node tests/run-all.js

# 훅 테스트 개별 실행
node tests/hooks/hooks.test.js
node tests/hooks/pre-bash-commit-quality.test.js
node tests/hooks/config-protection.test.js

# 라이브러리 테스트 개별 실행
node tests/lib/utils.test.js
node tests/lib/package-manager.test.js
node tests/lib/session-manager.test.js
```

## 아키텍처

```
everything-claude-code/
├── agents/          ← 16개 서브에이전트
│   ├── code-reviewer.md         # 코드 품질/보안 리뷰
│   ├── cpp-build-resolver.md    # C++ 빌드 오류 해결
│   ├── cpp-reviewer.md          # C++ 코드 리뷰
│   ├── tdd-guide.md             # TDD 가이드
│   ├── planner.md               # 구현 계획
│   ├── architect.md             # 시스템 설계
│   ├── build-error-resolver.md  # 범용 빌드 오류 해결
│   ├── security-reviewer.md     # 보안 취약점 분석
│   ├── refactor-cleaner.md      # 사용하지 않는 코드 정리
│   ├── doc-updater.md           # 문서 동기화
│   ├── docs-lookup.md           # 문서 검색
│   ├── database-reviewer.md     # DB 리뷰
│   ├── chief-of-staff.md        # 프로젝트 오케스트레이션
│   ├── harness-optimizer.md     # 하네스 성능 최적화
│   ├── loop-operator.md         # 연속 루프 운영
│   └── performance-optimizer.md # 성능 최적화
│
├── commands/        ← 41개 슬래시 커맨드
│   ├── tdd.md, plan.md, code-review.md, build-fix.md     # 핵심 워크플로우
│   ├── cpp-build.md, cpp-review.md, cpp-test.md          # C++ 전용
│   ├── save-session.md, resume-session.md, sessions.md   # 세션 관리
│   ├── learn.md, evolve.md, skill-create.md              # 학습/스킬
│   ├── verify.md, quality-gate.md, test-coverage.md      # 검증
│   └── ...그 외 26개
│
├── skills/          ← 54개 워크플로우 스킬 (각 폴더에 SKILL.md)
│   ├── cpp-coding-standards/    # C++ 코딩 표준
│   ├── cpp-testing/             # C++ 테스트 패턴
│   ├── tdd-workflow/            # TDD 방법론
│   ├── git-workflow/            # Git 워크플로우
│   ├── security-review/         # 보안 체크리스트
│   ├── continuous-learning-v2/  # 직관 기반 학습 (agents/, hooks/, scripts/ 포함)
│   ├── codebase-onboarding/     # 온보딩
│   └── ...그 외 47개
│
├── rules/           ← 20개 규칙 파일
│   ├── common/      # 10개: coding-style, git-workflow, security, testing, patterns 등
│   ├── csharp/      # 5개: coding-style, patterns, testing, security, hooks
│   └── cpp/         # 5개: coding-style, patterns, testing, security, hooks
│
├── hooks/           ← 훅 설정
│   ├── hooks.json   # 전체 훅 정의 (PreToolUse, PostToolUse, SessionStart, Stop 등)
│   └── README.md    # 훅 설명
│
├── scripts/         ← Node.js 유틸리티
│   ├── hooks/       # 30개 훅 스크립트 (run-with-flags.js, session-start.js 등)
│   └── lib/         # 50개 라이브러리 (session-manager, utils, package-manager 등)
│       ├── state-store/       # SQLite 기반 상태 저장소
│       ├── skill-evolution/   # 스킬 진화 추적
│       ├── skill-improvement/ # 스킬 개선
│       ├── session-adapters/  # 세션 어댑터 (canonical, claude-history, dmux)
│       ├── install/           # 설치 파이프라인
│       └── install-targets/   # 설치 대상 (claude, codex, cursor 등)
│
├── tests/           ← 36개 테스트 파일
│   ├── hooks/       # 19개 훅 테스트
│   ├── lib/         # 16개 라이브러리 테스트
│   └── integration/ # 1개 통합 테스트
│
├── contexts/        ← 3개 동적 컨텍스트 (dev, research, review)
├── examples/        ← 8개 CLAUDE.md 예제 (django, go, rust, laravel 등)
├── mcp-configs/     ← MCP 서버 설정 (github, memory, context7 등 10개)
└── docs/            ← 문서 (ko-KR 포함)
```

## 커맨드

41개 슬래시 커맨드를 사용할 수 있다. 전체 목록은 [COMMANDS-QUICK-REF.md](COMMANDS-QUICK-REF.md) 참고.

## 훅 런타임 제어

훅은 `hooks/hooks.json`에 정의되며, 환경변수로 런타임 제어가 가능하다.

```bash
# 훅 엄격도 프로필 (기본값: standard)
# minimal — 세션 시작/종료만 | standard — 품질 검사 포함 | strict — 포매팅/타입체크까지
ECC_HOOK_PROFILE=standard

# 특정 훅 비활성화 (쉼표 구분)
ECC_DISABLED_HOOKS=pre:bash:tmux-reminder,post:edit:typecheck,post:edit:format

# 훅 스크립트 경로 기준점 — ~/.claude/ 절대경로
CLAUDE_PLUGIN_ROOT=C:/Users/<사용자명>/.claude
```

모든 훅 스크립트는 `scripts/hooks/run-with-flags.js` 래퍼를 통해 실행되며, 프로필과 비활성화 목록을 자동으로 반영한다.

## 개발 노트

- **런타임:** Node.js >=18 (트랜스파일 없음, CommonJS)
- **모듈 시스템:** CommonJS only — `require`/`module.exports` 사용, ESM(`import`/`export`) 금지 (`.mjs` 제외)
- **파일 명명:** 하이픈 소문자 (예: `cpp-reviewer.md`, `tdd-workflow.md`)
- **에이전트 형식:** YAML 프론트매터가 있는 마크다운 (`name`, `description`, `tools`, `model`)
- **스킬 형식:** 마크다운 — When to Use, How It Works, Examples 섹션
- **커맨드 형식:** 마크다운 — `description:` 프론트매터 필수
- **훅 형식:** JSON — `matcher` 조건 + `hooks` 배열 (`type`, `command`, `async`, `timeout`)
- **훅 스크립트:** 200줄 이하 유지, 헬퍼는 `scripts/lib/`로 분리, 파싱 오류 시 반드시 `exit 0`
- **테스트:** 새 `scripts/lib/` 스크립트에는 `tests/lib/`에 매칭 테스트 필요

## 원본 참고 자료

- [원본 README](https://github.com/affaan-m/everything-claude-code/blob/main/README.md) — 전체 구성요소 목록, 릴리스 노트
- [요약 가이드](https://github.com/affaan-m/everything-claude-code/blob/main/the-shortform-guide.md) — 설정 철학과 핵심 개념
- [상세 가이드](https://github.com/affaan-m/everything-claude-code/blob/main/the-longform-guide.md) — 토큰 최적화, 메모리 영속성, 병렬화
- [보안 가이드](https://github.com/affaan-m/everything-claude-code/blob/main/the-security-guide.md) — 에이전트 보안
- [훅 설명](https://github.com/affaan-m/everything-claude-code/blob/main/hooks/README.md) — 각 훅의 동작 방식
- [스킬 개발 가이드](https://github.com/affaan-m/everything-claude-code/blob/main/docs/SKILL-DEVELOPMENT-GUIDE.md) — 스킬 작성 패턴
- [스킬 배치 정책](https://github.com/affaan-m/everything-claude-code/blob/main/docs/SKILL-PLACEMENT-POLICY.md) — 스킬 저장 위치 규칙
- [기여 가이드](https://github.com/affaan-m/everything-claude-code/blob/main/CONTRIBUTING.md) — 에이전트/스킬/커맨드 작성 형식
