# 07. 미공개/예정 기능 심층 분석 보고서

> **분석팀장 루미** | 2026-03-31  
> 대상: Claude Code CLI (유출 소스코드, ~52만 줄)  
> 발견된 빌드타임 피처 플래그: **88개** (`bun:bundle`의 `feature()`)  
> GrowthBook 런타임 게이트: **60개 이상** (`tengu_*` 네이밍)

---

## 1. 피처 플래그 시스템 개요

Claude Code는 **2중 피처 게이팅 구조**를 사용한다.

| 계층 | 메커니즘 | 동작 방식 |
|------|---------|----------|
| **빌드타임** | `feature('FLAG_NAME')` from `bun:bundle` | 번들러가 false인 플래그의 코드를 **완전 제거** (Dead Code Elimination) |
| **런타임** | GrowthBook `getFeatureValue_CACHED_MAY_BE_STALE('tengu_*')` | 서버에서 동적으로 on/off, A/B 테스트, 점진적 롤아웃 |
| **빌드 상수** | `process.env.USER_TYPE === 'ant'` | Anthropic 직원 빌드 vs 외부 빌드 분리 (빌드타임 상수) |

외부 빌드(npm 배포판)에서는 `feature()` false 플래그의 코드가 물리적으로 존재하지 않는다. 이 보고서는 소스코드에만 존재하는 숨겨진 기능들을 분석한다.

---

## 2. 최상위 발견: 미공개 기능 총괄표

### Tier 1 -- 대규모 신기능 (혁신적)

| # | 기능명 | 피처 플래그 | 설명 | 관련 파일 수 | 상태 |
|---|--------|-----------|------|------------|------|
| 1 | **KAIROS (어시스턴트 모드)** | `KAIROS`, `KAIROS_BRIEF`, `KAIROS_CHANNELS`, `KAIROS_DREAM`, `KAIROS_PUSH_NOTIFICATION`, `KAIROS_GITHUB_WEBHOOKS` | 상시 실행 AI 어시스턴트. 채널 알림, 푸시 노티, PR 구독, 일일 로그, 메모리 통합(dream) | 50+ | ant-only, 핵심 개발 중 |
| 2 | **ULTRAPLAN (원격 멀티에이전트 계획)** | `ULTRAPLAN` | CCR(Cloud) 환경에서 30분 타임아웃 멀티에이전트 탐색/계획. Opus 4.6 모델 사용, 원격 세션 텔레포트 | 10+ | ant-only, 실험 단계 |
| 3 | **COORDINATOR_MODE (코디네이터 모드)** | `COORDINATOR_MODE` | 에이전트를 "코디네이터"로 전환. 직접 코딩 대신 워커 에이전트들에게 작업 위임. 제한된 도구셋만 사용 | 20+ | ant-only, 활성 개발 |
| 4 | **VOICE_MODE (음성 입력)** | `VOICE_MODE` | Push-to-talk 음성 입력. Anthropic voice_stream WebSocket STT 엔드포인트 연결, OAuth 인증 | 5+ | ant-only |
| 5 | **BUDDY (가상 동반자)** | `BUDDY` | 터미널에 ASCII 스프라이트 캐릭터 표시. 감정 반응, 말풍선, `/buddy pet` 명령, 하트 애니메이션. 희귀도 시스템 존재 | 8+ | ant-only, 실험적 |
| 6 | **BRIDGE_MODE (원격 제어)** | `BRIDGE_MODE` | 로컬 머신을 원격 환경의 브릿지로 사용. `claude remote-control` 명령. WebSocket 기반 양방향 동기화 | 15+ | ant-only, GrowthBook 게이트 |
| 7 | **DAEMON (백그라운드 데몬)** | `DAEMON` | 장기 실행 supervisor 프로세스. `claude daemon` 서브커맨드, 워커 레지스트리 | 3+ | ant-only |
| 8 | **CHICAGO_MCP (컴퓨터 사용)** | `CHICAGO_MCP` | Computer Use MCP 서버 내장. macOS 전용, 스크린 캡처/마우스/키보드 제어 | 10+ | ant-only |

### Tier 2 -- 중요 신규 도구/기능

| # | 기능명 | 피처 플래그 | 설명 | 상태 |
|---|--------|-----------|------|------|
| 9 | **AGENT_TRIGGERS (스케줄링)** | `AGENT_TRIGGERS`, `AGENT_TRIGGERS_REMOTE` | Cron 기반 에이전트 자동 실행. CronCreate/Delete/List 도구. 원격 트리거 지원 | ant-only |
| 10 | **BG_SESSIONS (백그라운드 세션)** | `BG_SESSIONS` | `claude --bg`로 백그라운드 실행. `ps`, `logs`, `attach`, `kill` 서브커맨드 | ant-only |
| 11 | **WEB_BROWSER_TOOL (웹 브라우저)** | `WEB_BROWSER_TOOL` | 내장 웹 브라우저 도구. Bun WebView 활용, 브라우저 패널 UI | ant-only |
| 12 | **TERMINAL_PANEL (터미널 패널)** | `TERMINAL_PANEL` | 분할 터미널 패널. `meta+j` 단축키, TerminalCaptureTool | ant-only |
| 13 | **MONITOR_TOOL (모니터링)** | `MONITOR_TOOL` | 백그라운드 프로세스/MCP 서버 모니터링 도구. 자동 감시 태스크 생성 | ant-only |
| 14 | **WORKFLOW_SCRIPTS (워크플로우)** | `WORKFLOW_SCRIPTS` | 사용자 정의 워크플로우 스크립트 실행. 번들된 워크플로우 + WorkflowTool | ant-only |
| 15 | **UDS_INBOX (피어 메시징)** | `UDS_INBOX` | Unix Domain Socket 기반 Claude 인스턴스 간 메시징. ListPeersTool, `/peers` 명령 | ant-only |
| 16 | **PROACTIVE (능동 모드)** | `PROACTIVE` | 에이전트가 먼저 행동 개시. SleepTool로 대기 후 자동 실행, 파일 시스템 감시 | ant-only |
| 17 | **FORK_SUBAGENT (포크 서브에이전트)** | `FORK_SUBAGENT` | 대화 포크로 서브에이전트 분기. `/fork` 명령을 독립 서브에이전트 타입으로 변환 | ant-only |

### Tier 3 -- 컨텍스트/메모리 최적화

| # | 기능명 | 피처 플래그 | 설명 | 상태 |
|---|--------|-----------|------|------|
| 18 | **CONTEXT_COLLAPSE (컨텍스트 축소)** | `CONTEXT_COLLAPSE` | 413 에러 시 지능적 컨텍스트 축소. CtxInspectTool로 컨텍스트 검사 | ant-only |
| 19 | **HISTORY_SNIP (히스토리 스니핑)** | `HISTORY_SNIP` | 대화 히스토리에서 불필요한 부분을 "스니핑"(잘라내기). SnipTool 제공 | ant-only |
| 20 | **REACTIVE_COMPACT (반응형 압축)** | `REACTIVE_COMPACT` | 토큰 임계치 도달 시 자동 압축 트리거. 선제적 컨텍스트 관리 | ant-only |
| 21 | **CACHED_MICROCOMPACT (캐시 마이크로압축)** | `CACHED_MICROCOMPACT` | 캐시 친화적 마이크로 압축. 프롬프트 캐시 효율 극대화 | ant-only |
| 22 | **TOKEN_BUDGET (토큰 예산)** | `TOKEN_BUDGET` | 사용자가 토큰 예산 설정 가능. 예산 소진 시 자동 중단 | ant-only |
| 23 | **PROMPT_CACHE_BREAK_DETECTION** | `PROMPT_CACHE_BREAK_DETECTION` | 프롬프트 캐시 깨짐 감지. 캐시 효율 모니터링 | ant-only |

### Tier 4 -- 메모리/학습 시스템

| # | 기능명 | 피처 플래그 | 설명 | 상태 |
|---|--------|-----------|------|------|
| 24 | **EXTRACT_MEMORIES (메모리 추출)** | `EXTRACT_MEMORIES` | 대화에서 자동으로 기억할 사항 추출. 턴 종료 후 실행 | ant-only |
| 25 | **TEAMMEM (팀 메모리)** | `TEAMMEM` | 팀 간 공유 메모리. 동기화, 비밀 필터링, 워처 시스템 | ant-only, 대규모 |
| 26 | **MEMORY_SHAPE_TELEMETRY** | `MEMORY_SHAPE_TELEMETRY` | 메모리 형태 분석 텔레메트리 | ant-only |
| 27 | **AGENT_MEMORY_SNAPSHOT** | `AGENT_MEMORY_SNAPSHOT` | 커스텀 에이전트의 메모리 스냅샷 자동 업데이트 | ant-only |

### Tier 5 -- 분류기/자동화

| # | 기능명 | 피처 플래그 | 설명 | 상태 |
|---|--------|-----------|------|------|
| 28 | **TRANSCRIPT_CLASSIFIER (자동 모드)** | `TRANSCRIPT_CLASSIFIER` | 대화 분류기 기반 자동 권한 결정. "auto" 모드 활성화 | 부분 공개 |
| 29 | **BASH_CLASSIFIER (배시 분류기)** | `BASH_CLASSIFIER` | Bash 명령 위험도 ML 분류. 병렬 분류/실행으로 지연 최소화 | ant-only |
| 30 | **TEMPLATES (작업 분류기)** | `TEMPLATES` | 작업 유형 자동 분류. 턴 종료 시 작업 분류 실행 | ant-only |
| 31 | **VERIFICATION_AGENT (검증 에이전트)** | `VERIFICATION_AGENT` | 3개 이상 태스크 완료 시 자동 검증 에이전트 생성 권고 | ant-only |

### Tier 6 -- UI/UX 개선

| # | 기능명 | 피처 플래그 | 설명 | 상태 |
|---|--------|-----------|------|------|
| 32 | **AWAY_SUMMARY (부재 요약)** | `AWAY_SUMMARY` | 5분 이상 부재 후 복귀 시 "부재 중 요약" 메시지 자동 생성 | ant-only |
| 33 | **MESSAGE_ACTIONS (메시지 액션)** | `MESSAGE_ACTIONS` | 메시지에 대한 인라인 액션 (복사, 편집 등). 커서 기반 | ant-only |
| 34 | **QUICK_SEARCH (빠른 검색)** | `QUICK_SEARCH` | 프롬프트 입력 중 빠른 검색 오버레이 | ant-only |
| 35 | **HISTORY_PICKER (히스토리 피커)** | `HISTORY_PICKER` | 모달 다이얼로그 기반 대화 히스토리 탐색. Ctrl+R 대체 | ant-only |
| 36 | **AUTO_THEME (자동 테마)** | `AUTO_THEME` | 터미널 환경에 맞는 자동 테마 선택 | ant-only |
| 37 | **STREAMLINED_OUTPUT (간소화 출력)** | `STREAMLINED_OUTPUT` | stream-json 모드에서 출력 간소화 | ant-only |
| 38 | **LODESTONE (딥링크/프로토콜)** | `LODESTONE` | `cc://` 프로토콜 핸들러 등록. 터미널 선호도 설정 | ant-only |

### Tier 7 -- 인프라/보안/내부

| # | 기능명 | 피처 플래그 | 설명 | 상태 |
|---|--------|-----------|------|------|
| 39 | **NATIVE_CLIENT_ATTESTATION** | `NATIVE_CLIENT_ATTESTATION` | 네이티브 클라이언트 증명 헤더 (`cch=`) 추가 | ant-only |
| 40 | **ANTI_DISTILLATION_CC** | `ANTI_DISTILLATION_CC` | 모델 증류 방지 메커니즘 | ant-only |
| 41 | **CONNECTOR_TEXT** | `CONNECTOR_TEXT` | 커넥터 텍스트 블록 처리. 요약 베타 헤더 | ant-only |
| 42 | **COMMIT_ATTRIBUTION (커밋 귀속)** | `COMMIT_ATTRIBUTION` | AI 생성 커밋의 귀속 정보 추적. 내부 모델 리포 전용 | ant-only |
| 43 | **UNATTENDED_RETRY (무인 재시도)** | `UNATTENDED_RETRY` | 무인 세션에서 429/529 에러 자동 재시도 | ant-only |
| 44 | **FILE_PERSISTENCE (파일 지속성)** | `FILE_PERSISTENCE` | 턴 시작/종료 시점 파일 상태 추적 | ant-only |
| 45 | **SETTINGS_SYNC** | `UPLOAD_USER_SETTINGS`, `DOWNLOAD_USER_SETTINGS` | 사용자 설정 클라우드 동기화 (업로드/다운로드) | ant-only |

### Tier 8 -- 원격 실행/배포

| # | 기능명 | 피처 플래그 | 설명 | 상태 |
|---|--------|-----------|------|------|
| 46 | **DIRECT_CONNECT** | `DIRECT_CONNECT` | `cc://` URL로 직접 원격 연결 | ant-only |
| 47 | **SSH_REMOTE** | `SSH_REMOTE` | SSH 기반 원격 환경 접속 | ant-only |
| 48 | **CCR_AUTO_CONNECT** | `CCR_AUTO_CONNECT` | Cloud 환경 자동 연결 | ant-only |
| 49 | **CCR_MIRROR** | `CCR_MIRROR` | Cloud 환경 미러링 (아웃바운드 전용) | ant-only |
| 50 | **CCR_REMOTE_SETUP** | `CCR_REMOTE_SETUP` | 원격 환경 설정 (`/web` 명령) | ant-only |
| 51 | **SELF_HOSTED_RUNNER** | `SELF_HOSTED_RUNNER` | 자체 호스팅 러너 모드 | ant-only |
| 52 | **BYOC_ENVIRONMENT_RUNNER** | `BYOC_ENVIRONMENT_RUNNER` | Bring Your Own Compute 환경 러너 | ant-only |

### Tier 9 -- 기타/실험적

| # | 기능명 | 피처 플래그 | 설명 | 상태 |
|---|--------|-----------|------|------|
| 53 | **EXPERIMENTAL_SKILL_SEARCH** | `EXPERIMENTAL_SKILL_SEARCH` | 원격 스킬 검색/매칭. 스킬 인덱스 캐시 | ant-only |
| 54 | **MCP_SKILLS** | `MCP_SKILLS` | MCP 서버에서 스킬 자동 발견/등록 | ant-only |
| 55 | **REVIEW_ARTIFACT (코드 리뷰)** | `REVIEW_ARTIFACT` | Hunter 스킬 - 아티팩트 리뷰 자동화 | ant-only |
| 56 | **BUILDING_CLAUDE_APPS** | `BUILDING_CLAUDE_APPS` | Claude API 앱 빌드 가이드 스킬 | ant-only |
| 57 | **RUN_SKILL_GENERATOR** | `RUN_SKILL_GENERATOR` | 스킬 자동 생성기 | ant-only |
| 58 | **TORCH** | `TORCH` | `/torch` 명령 (용도 미상, 내부 디버그 추정) | ant-only |
| 59 | **ABLATION_BASELINE** | `ABLATION_BASELINE` | A/B 테스트용 기능 제거 기준선. 모든 고급 기능 비활성화 | ant-only |
| 60 | **TREE_SITTER_BASH** | `TREE_SITTER_BASH`, `TREE_SITTER_BASH_SHADOW` | Tree-sitter 기반 Bash 명령 파싱. 섀도우 모드로 기존 대비 검증 | ant-only |
| 61 | **ULTRATHINK** | `ULTRATHINK` | 강화된 사고 모드 (thinking 확장) | ant-only |
| 62 | **BUILTIN_EXPLORE_PLAN_AGENTS** | `BUILTIN_EXPLORE_PLAN_AGENTS` | 내장 탐색/계획 에이전트 | ant-only |
| 63 | **COMPACTION_REMINDERS** | `COMPACTION_REMINDERS` | 압축 필요 시 알림 | ant-only |
| 64 | **NEW_INIT** | `NEW_INIT` | 개선된 `/init` 명령. 스킬/훅 설정 포함 | ant-only |
| 65 | **POWERSHELL_AUTO_MODE** | `POWERSHELL_AUTO_MODE` | PowerShell 자동 모드 권한 | ant-only |
| 66 | **SKILL_IMPROVEMENT** | `SKILL_IMPROVEMENT` | 스킬 자동 개선 훅 | ant-only |
| 67 | **HOOK_PROMPTS** | `HOOK_PROMPTS` | 훅에 프롬프트 전달 | ant-only |
| 68 | **SHOT_STATS** | `SHOT_STATS` | 샷(시도) 분포 통계 | ant-only |
| 69 | **PERFETTO_TRACING** | `PERFETTO_TRACING` | Perfetto 형식 성능 트레이싱 | ant-only |
| 70 | **HARD_FAIL** | `HARD_FAIL` | 에러 시 즉시 중단 모드 (디버그용) | ant-only |
| 71 | **MCP_RICH_OUTPUT** | `MCP_RICH_OUTPUT` | MCP 도구 출력 리치 텍스트 렌더링 | ant-only |
| 72 | **NATIVE_CLIPBOARD_IMAGE** | `NATIVE_CLIPBOARD_IMAGE` | 네이티브 클립보드 이미지 붙여넣기 | ant-only |

---

## 3. 핵심 기능 상세 분석

### 3.1 KAIROS -- 상시 AI 어시스턴트

KAIROS는 Claude Code를 **상시 실행 어시스턴트**로 전환하는 가장 대규모 미공개 기능이다.

**구성 요소:**
- `KAIROS` (메인): 어시스턴트 모드 활성화, 세션 지속, 일일 로그
- `KAIROS_BRIEF`: 요약 모드 - SendUserMessage로 체크포인트 전달
- `KAIROS_CHANNELS`: MCP 채널 기반 알림 시스템
- `KAIROS_PUSH_NOTIFICATION`: 모바일 푸시 알림
- `KAIROS_GITHUB_WEBHOOKS`: PR 구독 (SubscribePRTool)
- `KAIROS_DREAM`: 메모리 통합 - 수면 중 기억 정리

**핵심 코드 위치:**
- `src/main.tsx` (80-81행): assistant 모듈 조건 임포트
- `src/assistant/` 디렉토리: 게이트 체크, 모드 전환
- `src/tools/SendUserFileTool/`: 사용자에게 파일 전송
- `src/tools/PushNotificationTool/`: 푸시 알림 전송
- `src/tools/SubscribePRTool/`: GitHub PR 웹훅 구독
- `src/tools/BriefTool/BriefTool.ts`: 5분 주기 요약 갱신
- `src/services/mcp/channelNotification.ts`: 채널 알림
- `src/memdir/memdir.ts` (319-432행): 일일 로그 프롬프트

**추정 출시**: 2026 H2 (가장 활발한 개발 중, 파일 50개 이상 관련)

### 3.2 ULTRAPLAN -- 원격 멀티에이전트 계획

**작동 방식:**
1. 사용자가 `/ultraplan` 슬래시 커맨드 또는 프롬프트에서 트리거
2. CCR(Cloud) 환경에 원격 세션 생성 (30분 타임아웃)
3. Opus 4.6 모델로 멀티에이전트 탐색 실행
4. 승인된 계획을 로컬로 텔레포트
5. 로컬에서 계획 실행

**핵심 코드:**
- `src/commands/ultraplan.tsx`: 메인 명령 구현
- `src/utils/ultraplan/ccrSession.ts`: CCR 세션 관리, 텔레포트
- `src/constants/xml.ts`: `ULTRAPLAN_TAG` XML 태그
- GrowthBook: `tengu_ultraplan_model`로 모델 동적 선택

### 3.3 BUDDY -- 가상 동반자 (가장 이색적)

터미널에 **ASCII 스프라이트 캐릭터**를 표시하는 기능.

**특징:**
- 아이들 시퀀스 애니메이션 (500ms 틱)
- 감정 반응 (`CompanionSprite.tsx`)
- 말풍선 (10초 표시, 3초 페이드)
- `/buddy pet` 명령: 하트 파티클 애니메이션 (2.5초)
- **희귀도 시스템** (`RARITY_COLORS` 상수)
- 풀스크린 모드 대응 (좁은 레이아웃/와이드 레이아웃)

**코드:** `src/buddy/` 디렉토리 전체

> 상세 분석은 [18-easter-eggs.md](18-easter-eggs.md) 참조.

### 3.4 Undercover Mode -- Anthropic 직원 위장 모드

`process.env.USER_TYPE === 'ant'` 전용. 외부/오픈소스 저장소 작업 시:

- 특정 attribution과 커밋/PR 프롬프트 지시사항 조정
- Co-Authored-By 라인 제거
- 모델 코드네임 (Capybara, Tengu 등) 노출 방지
- 자동 감지: 내부 리포 allowlist와 비교하여 자동 ON/OFF
- `CLAUDE_CODE_UNDERCOVER=1`로 강제 활성화
- 비활성화 옵션 없음 (안전 우선)

**코드:** `src/utils/undercover.ts`

---

## 4. GrowthBook 런타임 게이트 분석

GrowthBook을 통해 **서버 사이드에서 동적으로 제어**되는 기능들. 모든 게이트는 `tengu_` 접두사를 사용한다.

| GrowthBook 게이트 | 기능 | 비고 |
|------------------|------|------|
| `tengu_scratch` | 코디네이터 모드 스크래치패드 | 파일시스템 권한 관련 |
| `tengu_onyx_plover` | 자동 dream (메모리 통합) | 임계치 설정 포함 |
| `tengu_cobalt_frost` | 음성 STT Nova 3 모델 | 음성 모드 모델 선택 |
| `tengu_harbor_permissions` | 채널 권한 | KAIROS 채널 권한 |
| `tengu_willow_mode` | 세션 복원 모드 | 'off', 기타 모드 |
| `tengu_sedge_lantern` | Away Summary 활성화 | 3P 기본 false |
| `tengu_thinkback` | Thinkback 명령 | 사고 과정 재생 |
| `tengu_slate_thimble` | 비대화형 메모리 추출 | 추출 모드 게이트 |
| `tengu_slate_prism` | 컴팩트 관련 | 기본 true |
| `tengu_passport_quail` | 메모리 추출 게이트 | ant-only 추가 체크 |
| `tengu_bramble_lintel` | 메모리 추출 빈도 | N턴마다 실행 |
| `tengu_moth_copse` | 관련 메모리 프리페치 | 메모리 검색 최적화 |
| `tengu_paper_halyard` | 메모리 형태 | 미상 |
| `tengu_ultraplan_model` | UltraPlan 모델 선택 | 기본: opus46 |
| `tengu_terminal_sidebar` | 터미널 사이드바 | 탭 상태 게이트 |
| `tengu_streaming_tool_execution2` | 스트리밍 도구 실행 v2 | 런타임 설정 |
| `tengu_otk_slot_v1` | 최대 토큰 에스컬레이션 | 토큰 제한 동적 조정 |
| `tengu_keybinding_customization_release` | 키바인딩 커스터마이징 공개 | ant-only |
| `tengu_bridge_poll_interval_config` | 브릿지 폴링 간격 | 원격 제어 |
| `tengu_ant_model_override` | 내부 모델 오버라이드 | ant-only |
| `tengu_max_version_config` | 최대 버전 설정 | 업데이트 제어 |
| `tengu_event_sampling_config` | 이벤트 샘플링 | 텔레메트리 |
| `tengu_1p_event_batch_config` | 1P 이벤트 배치 | 텔레메트리 |
| `tengu_frond_boric` | 싱크 킬스위치 | 텔레메트리 비상 정지 |
| `tengu_log_datadog_events` | Datadog 이벤트 로깅 | 모니터링 |
| `tengu_tool_pear` | 도구 관련 (미상) | 도구 실행 수정 |

---

## 5. Anthropic 직원 전용 기능 (`USER_TYPE === 'ant'`)

빌드타임 상수로 feature() 플래그와 별도로 게이팅된 기능들:

| 기능 | 설명 | 코드 위치 |
|------|------|----------|
| **REPLTool** | 인터랙티브 REPL 도구 | `src/tools/REPLTool/` |
| **SuggestBackgroundPRTool** | 백그라운드 PR 제안 도구 | `src/tools/SuggestBackgroundPRTool/` |
| **TungstenTool** | Tmux 기반 가상 터미널 (싱글톤) | `src/tools/TungstenTool/` |
| **VerifyPlanExecutionTool** | 계획 실행 검증 도구 | `src/tools/VerifyPlanExecutionTool/` |
| **Undercover Mode** | 오픈소스 기여 시 신원 위장 | `src/utils/undercover.ts` |
| **내부 모델 오버라이드** | GrowthBook으로 모델 강제 변경 | `src/services/analytics/growthbook.ts` |
| **시스템 프롬프트 덤프** | `--dump-system-prompt` 플래그 | `src/entrypoints/cli.tsx:53` |
| **1P 이벤트 로깅** | 자사 텔레메트리 엔드포인트 | `src/services/analytics/firstPartyEventLogger.ts` |
| **확장 에러 정보** | 5xx 에러 시 상세 디버그 정보 | `src/services/api/withRetry.ts` |
| **내부 베타 헤더** | `cli-internal-2026-02-09` | `src/constants/betas.ts:30` |
| **키바인딩 커스터마이징** | 외부 사용자는 기본 바인딩만 | `src/keybindings/loadUserBindings.ts` |
| **GrowthBook 오버라이드** | 로컬에서 피처 플래그 강제 설정 | `src/services/analytics/growthbook.ts:163-274` |
| **VSCode SDK MCP** | VSCode MCP 클라이언트 | `src/services/mcp/vscodeSdkMcp.ts` |
| **MagicDocs 확장** | 내부 문서 시스템 | `src/services/MagicDocs/magicDocs.ts:243` |

---

## 6. 숨겨진 명령/도구 요약

### 숨겨진 슬래시 명령

| 명령 | 피처 플래그 | 설명 |
|------|-----------|------|
| `/ultraplan` | `ULTRAPLAN` | 원격 멀티에이전트 계획 |
| `/torch` | `TORCH` | 내부 전용 명령 (용도 미상) |
| `/fork` | `FORK_SUBAGENT` | 대화 포크로 서브에이전트 분기 |
| `/buddy` | `BUDDY` | 가상 동반자 관리 |
| `/loop` | `AGENT_TRIGGERS` | 반복 실행 스킬 |
| `/schedule` | `AGENT_TRIGGERS_REMOTE` | 원격 에이전트 스케줄링 |
| `/dream` | `KAIROS`/`KAIROS_DREAM` | 메모리 통합 |
| `/peers` | `UDS_INBOX` | 피어 인스턴스 목록 |
| `/workflows` | `WORKFLOW_SCRIPTS` | 워크플로우 관리 |
| `/web` | `CCR_REMOTE_SETUP` | 원격 환경 설정 |
| `/subscribe-pr` | `KAIROS_GITHUB_WEBHOOKS` | PR 웹훅 구독 |
| `/thinkback` | (런타임 게이트) | 사고 과정 재생 |
| `/brief` | `KAIROS_BRIEF` | 요약 모드 토글 |

### 숨겨진 CLI 서브커맨드

| 명령 | 피처 플래그 | 설명 |
|------|-----------|------|
| `claude daemon` | `DAEMON` | 백그라운드 데몬 실행 |
| `claude remote-control` | `BRIDGE_MODE` | 원격 제어 서버 |
| `claude ps` | `BG_SESSIONS` | 백그라운드 세션 목록 |
| `claude logs` | `BG_SESSIONS` | 세션 로그 조회 |
| `claude attach` | `BG_SESSIONS` | 세션 연결 |
| `claude kill` | `BG_SESSIONS` | 세션 종료 |
| `claude --bg` | `BG_SESSIONS` | 백그라운드 실행 |
| `claude --computer-use-mcp` | `CHICAGO_MCP` | Computer Use MCP 서버 |
| `claude --daemon-worker` | `DAEMON` | 데몬 워커 프로세스 |
| `claude --dump-system-prompt` | `DUMP_SYSTEM_PROMPT` | 시스템 프롬프트 출력 |
| `claude environment-runner` | `BYOC_ENVIRONMENT_RUNNER` | BYOC 환경 러너 |
| `claude self-hosted-runner` | `SELF_HOSTED_RUNNER` | 자체 호스팅 러너 |

### 숨겨진 도구 (Tool)

| 도구 | 피처 플래그 | 설명 |
|------|-----------|------|
| SleepTool | `PROACTIVE`/`KAIROS` | 에이전트 대기 |
| SendUserFileTool | `KAIROS` | 사용자에게 파일 전송 |
| PushNotificationTool | `KAIROS`/`KAIROS_PUSH_NOTIFICATION` | 푸시 알림 |
| SubscribePRTool | `KAIROS_GITHUB_WEBHOOKS` | PR 웹훅 구독 |
| CronCreate/Delete/ListTool | `AGENT_TRIGGERS` | Cron 작업 관리 |
| RemoteTriggerTool | `AGENT_TRIGGERS_REMOTE` | 원격 트리거 |
| MonitorTool | `MONITOR_TOOL` | 프로세스 모니터링 |
| CtxInspectTool | `CONTEXT_COLLAPSE` | 컨텍스트 검사 |
| SnipTool | `HISTORY_SNIP` | 히스토리 스니핑 |
| TerminalCaptureTool | `TERMINAL_PANEL` | 터미널 캡처 |
| WebBrowserTool | `WEB_BROWSER_TOOL` | 웹 브라우저 |
| ListPeersTool | `UDS_INBOX` | 피어 목록 |
| WorkflowTool | `WORKFLOW_SCRIPTS` | 워크플로우 실행 |
| OverflowTestTool | `OVERFLOW_TEST_TOOL` | 오버플로우 테스트 |

---

## 7. 주석/TODO에서 발견된 향후 계획 힌트

| 위치 | 내용 | 해석 |
|------|------|------|
| `src/constants/apiLimits.ts:10` | "Future: See issue #13240 for dynamic limits fetching from server" | API 제한을 서버에서 동적 조회하는 기능 예정 |
| `src/commands/ultraplan.tsx:20` | "TODO(prod-hardening): OAuth token may go stale over the 30min poll" | UltraPlan 프로덕션 준비 중 |
| `src/skills/bundled/scheduleRemoteAgents.ts:31` | "TODO(public-ship): Before shipping publicly, the /v1/mcp_servers endpoint" | 원격 에이전트 스케줄링 공개 배포 준비 중 |
| `src/services/mcp/xaa.ts:229` | "TODO(xaa-ga): consult token_endpoint_auth_methods_supported" | OAuth 인증 GA 준비 |
| `src/services/lsp/LSPServerManager.ts:374` | "TODO: Integrate with compact - call closeFile() when compact removes files" | LSP와 컴팩트 통합 예정 |
| `src/services/compact/autoCompact.ts:191` | "out of external builds (REACTIVE_COMPACT is ant-only)" | 반응형 압축 외부 배포 가능성 |

---

## 8. 출시 시기 추정

| 기능 | 추정 시기 | 근거 |
|------|---------|------|
| TRANSCRIPT_CLASSIFIER (auto 모드) | **이미 부분 공개** | 외부 빌드에도 포함, GrowthBook 게이트로 롤아웃 중 |
| BG_SESSIONS (백그라운드 세션) | **2026 Q2-Q3** | CLI 서브커맨드 완성도 높음 |
| AGENT_TRIGGERS (스케줄링) | **2026 Q2-Q3** | `TODO(public-ship)` 주석 존재 |
| BRIDGE_MODE (원격 제어) | **2026 Q2** | 이미 ant 빌드에서 활성, GrowthBook 게이트 |
| KAIROS (어시스턴트) | **2026 H2** | 6개 하위 플래그, 가장 큰 규모 |
| COORDINATOR_MODE | **2026 Q3** | 코드 안정적, 런타임 게이트 존재 |
| ULTRAPLAN | **2026 Q3-Q4** | CCR 의존성, prod-hardening TODO |
| VOICE_MODE | **2026 Q3-Q4** | WebSocket STT 완성, OAuth 연동 |
| BUDDY | **불확실** | 이스터에그/실험적 성격 강함 |
| CHICAGO_MCP | **2026 H2** | macOS 전용, Computer Use 통합 |

---

## 9. 핵심 통찰

1. **Claude Code는 "에디터 도우미"가 아니라 "자율 에이전트 플랫폼"으로 진화 중이다.** KAIROS, COORDINATOR_MODE, AGENT_TRIGGERS, BG_SESSIONS, DAEMON이 결합되면 24/7 자율 실행 AI 시스템이 된다.

2. **88개 피처 플래그 중 약 85%가 ant-only이다.** 외부 사용자가 보는 Claude Code와 Anthropic 내부에서 쓰는 Claude Code는 완전히 다른 제품이다.

3. **내부 코드네임 체계:** 프로젝트 코드네임은 "Tengu"(텐구), GrowthBook 게이트는 모두 `tengu_` 접두사. 피처 플래그는 일본 신화/도시 이름 패턴 (KAIROS=그리스어 적시, CHICAGO, LODESTONE 등).

4. **보안 의식이 매우 높다.** Undercover Mode는 Anthropic 직원이 오픈소스에 기여할 때 특정 attribution과 커밋/PR 프롬프트 지시사항을 조정해 내부 코드네임 노출을 줄이려는 장치다. 비활성화 옵션이 아예 없다.

5. **BUDDY는 가장 이색적인 발견이다.** 터미널 CLI에 타마고치 같은 가상 동반자를 넣으려는 시도는 개발자 경험의 새로운 방향을 시사한다. 희귀도 시스템까지 있어 수집 요소를 갖추고 있다.
