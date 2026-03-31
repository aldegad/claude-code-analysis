# 기술 스택 및 의존성 분석 보고서

**분석 대상**: nirholas/claude-code
**분석일**: 2026-03-31
**분석자**: 루미 (분석팀장)

---

## 1. 코어 기술 스택

| 카테고리 | 기술 | 버전/사양 | 비고 |
|----------|------|-----------|------|
| **런타임** | Bun | >= 1.1.0 | Node.js 대체, JSX 네이티브 지원, 빌드타임 피처 플래그 |
| **언어** | TypeScript | ^5.7.0 | strict 모드, ES modules |
| **터미널 UI** | React + Ink | React ^19.0.0 | CLI용 React 렌더러 |
| **CLI 파서** | Commander.js | ^13.1.0 (`@commander-js/extra-typings`) | 타입 안전 CLI 파싱 |
| **스키마 검증** | Zod | ^3.24.0 (v4 사용) | 런타임 입력 검증 |
| **린터/포매터** | Biome | ^1.9.0 | ESLint + Prettier 대체 |
| **API 클라이언트** | Anthropic SDK | ^0.39.0 | Claude API 호출 |
| **프로토콜** | MCP SDK | ^1.12.1 | Model Context Protocol |
| **텔레메트리** | OpenTelemetry | 다수 (api, core, logs, metrics, trace) | 관찰 가능성 |
| **피처 플래그** | GrowthBook | ^1.4.0 | A/B 테스팅 + 피처 게이팅 |

---

## 2. 의존성 상세 분석

### 2.1 프로덕션 의존성 (dependencies: 30개)

#### AI/프로토콜
| 패키지 | 버전 | 용도 |
|--------|------|------|
| `@anthropic-ai/sdk` | ^0.39.0 | Claude API 클라이언트 |
| `@modelcontextprotocol/sdk` | ^1.12.1 | MCP 서버/클라이언트 |

#### CLI/UI
| 패키지 | 버전 | 용도 |
|--------|------|------|
| `@commander-js/extra-typings` | ^13.1.0 | 타입 안전 CLI 파싱 |
| `react` | ^19.0.0 | UI 프레임워크 (Ink 기반 터미널 렌더링) |
| `chalk` | ^5.4.0 | 터미널 색상 스타일링 |
| `cli-boxes` | ^3.0.0 | 터미널 박스 그리기 |
| `figures` | ^6.1.0 | 유니코드 심볼 |
| `highlight.js` | ^11.11.0 | 코드 구문 강조 |
| `marked` | ^15.0.0 | 마크다운 파싱/렌더링 |
| `wrap-ansi` | ^9.0.0 | ANSI 문자열 래핑 |
| `strip-ansi` | ^7.1.0 | ANSI 이스케이프 제거 |
| `code-excerpt` | ^4.0.0 | 코드 발췌 표시 |
| `supports-hyperlinks` | ^3.1.0 | 터미널 하이퍼링크 지원 감지 |

#### 텔레메트리/분석
| 패키지 | 버전 | 용도 |
|--------|------|------|
| `@opentelemetry/api` | ^1.9.0 | OpenTelemetry API |
| `@opentelemetry/api-logs` | ^0.57.0 | 로그 API |
| `@opentelemetry/core` | ^1.30.0 | OpenTelemetry 코어 |
| `@opentelemetry/sdk-logs` | ^0.57.0 | 로그 SDK |
| `@opentelemetry/sdk-metrics` | ^1.30.0 | 메트릭 SDK |
| `@opentelemetry/sdk-trace-base` | ^1.30.0 | 트레이스 SDK |
| `@growthbook/growthbook` | ^1.4.0 | 피처 플래그 + A/B 테스팅 |

#### 유틸리티
| 패키지 | 버전 | 용도 |
|--------|------|------|
| `axios` | ^1.7.0 | HTTP 클라이언트 |
| `undici` | ^7.3.0 | HTTP 클라이언트 (Node.js fetch 구현체) |
| `ws` | ^8.18.0 | WebSocket 클라이언트/서버 |
| `lodash-es` | ^4.17.21 | 유틸리티 함수 (ES 모듈 버전) |
| `diff` | ^7.0.0 | 텍스트 diff 생성 |
| `execa` | ^9.5.0 | 자식 프로세스 실행 |
| `chokidar` | ^4.0.0 | 파일 시스템 감시 |
| `picomatch` | ^4.0.0 | glob 패턴 매칭 |
| `semver` | ^7.6.0 | 시맨틱 버전 관리 |
| `zod` | ^3.24.0 | 스키마 검증 |
| `yaml` | ^2.6.0 | YAML 파싱 |
| `p-map` | ^7.0.0 | 병렬 Promise 매핑 |
| `proper-lockfile` | ^4.1.2 | 파일 잠금 |
| `tree-kill` | ^1.2.2 | 프로세스 트리 종료 |
| `type-fest` | ^4.30.0 | TypeScript 타입 유틸리티 |
| `stack-utils` | ^2.0.6 | 스택 트레이스 파싱 |

### 2.2 개발 의존성 (devDependencies: 11개)

| 패키지 | 버전 | 용도 |
|--------|------|------|
| `@biomejs/biome` | ^1.9.0 | 린터 + 포매터 |
| `typescript` | ^5.7.0 | TypeScript 컴파일러 |
| `@types/react` | ^19.0.0 | React 타입 |
| `@types/node` | ^22.10.0 | Node.js 타입 |
| `@types/diff` | ^7.0.0 | diff 타입 |
| `@types/lodash-es` | ^4.17.12 | lodash 타입 |
| `@types/picomatch` | ^3.0.0 | picomatch 타입 |
| `@types/proper-lockfile` | ^4.1.4 | proper-lockfile 타입 |
| `@types/semver` | ^7.5.8 | semver 타입 |
| `@types/stack-utils` | ^2.0.3 | stack-utils 타입 |
| `@types/ws` | ^8.5.0 | WebSocket 타입 |

---

## 3. 빌드 시스템 및 설정

### 3.1 TypeScript 설정 (tsconfig.json)

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "isolatedModules": true,
    "noEmit": true,
    "allowImportingTsExtensions": true
  }
}
```

핵심 포인트:
- **ESNext 타겟**: 최신 JS 기능 전부 사용
- **bundler 모듈 해석**: Bun의 번들러 모드에 맞춤
- **react-jsx**: React 17+ JSX Transform
- **noEmit**: TypeScript는 타입 체크만, 빌드는 Bun이 담당
- **bun:bundle 경로 매핑**: `"bun:bundle": ["./src/types/bun-bundle.d.ts"]`

### 3.2 Biome 설정 (biome.json)

| 항목 | 설정 |
|------|------|
| 들여쓰기 | 탭 (tab), 2칸 너비 |
| 줄 너비 | 100자 |
| 따옴표 | 작은따옴표 (single quote) |
| 세미콜론 | 필요할 때만 (asNeeded) |
| 인지 복잡도 | 경고 (noExcessiveCognitiveComplexity: warn) |
| 미사용 임포트/변수 | 경고 |
| non-null assertion | 허용 (off) |
| 명시적 any | 허용 (off) |
| JSON 들여쓰기 | 스페이스, 2칸 |

### 3.3 npm 스크립트

| 스크립트 | 명령 |
|----------|------|
| `typecheck` | `tsc --noEmit` |
| `lint` | `biome check src/` |
| `lint:fix` | `biome check --write src/` |
| `format` | `biome format --write src/` |
| `check` | `biome check src/ && tsc --noEmit` |

**주목**: 빌드 스크립트가 없음. 원본 빌드 도구가 유출에 포함되지 않았기 때문.

### 3.4 Dockerfile

```dockerfile
FROM oven/bun:1-alpine AS base
RUN apk add --no-cache git ripgrep
```

- **기반 이미지**: `oven/bun:1-alpine` (경량 Bun 런타임)
- **런타임 의존**: git, ripgrep (GrepTool의 백엔드)
- **용도**: 소스 탐색용 개발 컨테이너 (실행 가능한 빌드는 생성하지 않음)

---

## 4. 런타임 외부 의존성

코드 내에서 참조되는 시스템 레벨 도구들:

| 도구 | 용도 | 참조 위치 |
|------|------|-----------|
| **ripgrep (rg)** | GrepTool의 백엔드 검색 엔진 | `tools/GrepTool/` |
| **git** | 버전 관리, 커밋, PR | `utils/git.ts`, 다수 커맨드 |
| **macOS Keychain** | 인증 정보 저장 | `utils/auth.ts` |
| **tree-sitter** | Bash AST 분석 | `utils/bash/treeSitterAnalysis.ts` |

---

## 5. 아키텍처 레벨 기술 선택의 특이점

### 5.1 Bun 런타임 채택

Node.js가 아닌 Bun을 런타임으로 선택. 주요 이유:
- **JSX 네이티브 지원**: 별도 트랜스파일러 없이 TSX 직접 실행
- **빌드타임 피처 플래그**: `bun:bundle`의 `feature()` 함수로 죽은 코드 제거
- **빠른 시작 시간**: CLI 도구에 중요한 부팅 속도 최적화
- **번들러 내장**: 별도 webpack/esbuild 불필요

### 5.2 React를 터미널 UI에 사용

웹이 아닌 터미널 UI에 React + Ink를 사용하는 독특한 선택:
- **컴포넌트 기반 UI**: 144개의 재사용 가능한 터미널 컴포넌트
- **React 훅**: 상태 관리, 사이드 이펙트를 웹 개발과 동일한 패턴으로
- **React Compiler**: 최적화된 리렌더링 (React 19)
- **디자인 시스템**: `components/design-system/`에 터미널용 디자인 시스템까지 구축

### 5.3 Biome 채택

ESLint + Prettier 조합 대신 Biome 단일 도구 사용:
- 린팅과 포매팅을 하나의 도구로
- Rust 기반으로 빠른 실행 속도
- `noExplicitAny: off` -- 51만줄 규모에서 any 허용은 실용적 결정

### 5.4 OpenTelemetry 전면 도입

관찰 가능성(Observability)을 위해 OpenTelemetry를 5개 패키지나 사용:
- API, Core, Logs, Metrics, Trace
- 지연 로딩으로 시작 시간 영향 최소화 (~400KB + gRPC ~700KB)

### 5.5 이중 HTTP 클라이언트

`axios`와 `undici`를 동시에 의존:
- `axios`: 범용 HTTP 클라이언트
- `undici`: Node.js의 fetch 구현체, 고성능 HTTP

### 5.6 Bash 파서 자체 구축

외부 라이브러리가 아닌 **자체 Bash 파서를 4,437줄 + AST 2,680줄 규모로 구축**. 보안 검증을 위해 Bash 명령을 파싱하고 분석하는 것이 핵심 목적. tree-sitter도 함께 사용.

---

## 6. 의존성 건강성 평가

### 장점
- **의존성 수가 적은 편**: 프로덕션 30개, 개발 11개 -- 51만줄 프로젝트 치고는 절제된 수준
- **모던 버전 사용**: React 19, TypeScript 5.7, Zod v4 등 최신 버전
- **ES 모듈 일관성**: `lodash-es` 사용, `"type": "module"` 설정
- **타입 안전성**: 모든 주요 의존성에 타입 정의 포함

### 주의점
- **bun:bundle 의존**: Bun 런타임에 강하게 결합. 다른 런타임으로 전환 어려움
- **빌드 도구 부재**: 원본 빌드 설정이 유출되지 않아 실행 가능한 빌드 불가능
- **MACRO.VERSION**: 빌드 타임 매크로 치환이 있지만 정의가 누락
- **내부 전용 코드**: `USER_TYPE === 'ant'` 가드, 내부 피처 플래그 등이 혼재

---

## 7. 코드 스타일 요약

| 항목 | 규칙 |
|------|------|
| 파일 명명 | PascalCase (export) 또는 kebab-case (commands) |
| 컴포넌트 | PascalCase |
| 타입 | PascalCase + Props/State/Context 접미사 |
| 훅 | `use` 접두사 |
| 상수 | SCREAMING_SNAKE_CASE |
| 임포트 | ES modules, `.js` 확장자 (Bun 규약) |
| 순환 의존성 | `const getModule = () => require('./heavy.js')` 패턴으로 해결 |
