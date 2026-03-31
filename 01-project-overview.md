# 프로젝트 개요 분석 보고서

**분석 대상**: nirholas/claude-code (GitHub)
**분석일**: 2026-03-31
**분석자**: 루미 (분석팀장)

---

## 1. 프로젝트 정체

이 레포지토리는 **Anthropic의 공식 CLI 도구인 "Claude Code"의 외부 공개 소스 코드로 알려진 자료**를 아카이브한 것이다.

- **유출일**: 2026-03-31
- **유출 경로**: npm 레지스트리에 퍼블리시된 패키지의 `.map` (소스맵) 파일이 원본 TypeScript 소스를 가리켰던 것으로 알려져 있다
- **발견자**: Chaofan Shou (@Fried_rice)가 X(트위터)에 관련 내용을 공개한 것으로 알려져 있다
- **원본 소유**: Anthropic (모든 코드의 저작권은 Anthropic에 귀속)

npm 패키지의 소스맵 파일이 Anthropic의 R2 스토리지 버킷에 있는 난독화되지 않은 원본 TypeScript 소스를 가리켰고, 이를 zip으로 내려받을 수 있었던 것으로 보고되었다.

---

## 2. Claude Code란 무엇인가

Claude Code는 **터미널에서 Claude AI와 직접 상호작용하여 소프트웨어 엔지니어링 작업을 수행하는 CLI 도구**이다.

### 핵심 기능

| 카테고리 | 기능 |
|----------|------|
| **파일 작업** | 파일 읽기, 쓰기, 편집 (부분 수정 포함) |
| **셸 명령** | Bash/PowerShell 명령 실행, 보안 검증 포함 |
| **코드 검색** | ripgrep 기반 콘텐츠 검색, glob 패턴 파일 탐색 |
| **Git 워크플로** | 커밋, PR, 코드 리뷰, diff, 브랜치 관리 |
| **웹 접근** | URL 페치, 웹 검색 |
| **다중 에이전트** | 서브에이전트 생성, 팀 에이전트, 코디네이터 |
| **IDE 통합** | VS Code, JetBrains와 양방향 브릿지 통신 |
| **MCP 지원** | Model Context Protocol 서버/클라이언트 |
| **플러그인 시스템** | 빌트인 + 서드파티 플러그인, 마켓플레이스 |
| **스킬 시스템** | 재사용 가능한 워크플로 정의 및 실행 |
| **음성 입력** | STT 기반 음성 인식 |
| **태스크 관리** | 에이전트 태스크 생성, 추적, 업데이트 |
| **메모리** | 영구 메모리 시스템 (CLAUDE.md, 프로젝트/사용자 메모리) |
| **예약 실행** | Cron 기반 스케줄드 트리거 |

### 슬래시 커맨드 (~50개)

사용자가 REPL에서 `/` 접두사로 호출하는 명령 체계:

- `/commit` -- git 커밋 생성
- `/review` -- 코드 리뷰
- `/compact` -- 컨텍스트 압축
- `/mcp` -- MCP 서버 관리
- `/config` -- 설정 관리
- `/doctor` -- 환경 진단
- `/memory` -- 영구 메모리 관리
- `/vim` -- Vim 모드 토글
- `/diff`, `/cost`, `/theme`, `/context`, `/pr_comments` 등

### 에이전트 도구 (~40개)

Claude가 호출할 수 있는 도구들:

- `BashTool` -- 셸 명령 실행 (보안 검증 포함)
- `FileReadTool` -- 파일/이미지/PDF/노트북 읽기
- `FileWriteTool` / `FileEditTool` -- 파일 생성/수정
- `GlobTool` / `GrepTool` -- 파일 검색
- `AgentTool` -- 서브에이전트 생성
- `WebFetchTool` / `WebSearchTool` -- 웹 접근
- `MCPTool` -- MCP 서버 도구 호출
- `LSPTool` -- Language Server Protocol 통합
- `SkillTool` -- 스킬 실행
- `TeamCreateTool` / `TeamDeleteTool` -- 팀 에이전트 관리
- `EnterWorktreeTool` / `ExitWorktreeTool` -- Git worktree 격리
- `ScheduleCronTool` -- 예약 트리거 생성
- 기타 다수

---

## 3. 프로젝트 규모

| 지표 | 수치 |
|------|------|
| **총 파일 수** | ~1,892개 (TS/TSX) |
| **총 코드 라인** | 515,894줄 |
| **src 루트 파일** | 18개 (핵심 엔트리 + 레지스트리) |
| **서브디렉토리** | 53개 |
| **슬래시 커맨드** | ~50개 (102개 항목, 디렉토리 포함) |
| **에이전트 도구** | ~40개 (43개 디렉토리) |
| **UI 컴포넌트** | ~144개 |
| **React 훅** | ~87개 |
| **서비스 모듈** | ~37개 |
| **유틸리티** | ~329개 항목 |

---

## 4. 이 레포의 부가 자산

### MCP 서버 (`mcp-server/`)

Claude Code 소스를 탐색할 수 있게 해주는 MCP 서버가 포함되어 있다. Claude 세션에서 직접 소스코드를 조회, 검색, 분석할 수 있는 도구와 프롬프트를 제공한다.

제공 도구:
- `list_tools`, `list_commands` -- 도구/커맨드 목록
- `get_tool_source`, `get_command_source` -- 소스 코드 읽기
- `read_source_file`, `search_source` -- 파일 읽기/검색
- `get_architecture` -- 아키텍처 개요

### Dockerfile

`oven/bun:1-alpine` 기반의 개발 컨테이너. 실행 가능한 빌드를 만들지는 않지만 (원본 빌드 도구가 유출에 포함되지 않음), 소스 탐색용 환경을 제공한다.

### GitPretty

파일별 커밋 메시지를 시각적으로 구분하기 위한 이모지 커밋 스크립트.

---

## 5. 핵심 인사이트

1. **이것은 Anthropic의 실제 프로덕션 코드**: 51만줄 규모의 완전한 TypeScript 소스로, 테스트 코드 없이 순수 프로덕션 코드만 유출됨
2. **소스맵을 통한 유출**: npm 패키지에 포함된 소스맵이 유출 경로였던 것으로 알려져 있다. 보안적으로 주목할 만한 사례
3. **빌드 도구 미포함**: `bun:bundle` 피처 플래그, `MACRO.VERSION` 등 빌드 타임 치환이 있지만, 실제 빌드 설정은 포함되지 않음
4. **내부 전용 기능 존재**: `process.env.USER_TYPE === 'ant'`로 게이팅된 Anthropic 내부 전용 기능, `feature('DUMP_SYSTEM_PROMPT')` 같은 내부 진단 도구 등이 코드에 남아있음
