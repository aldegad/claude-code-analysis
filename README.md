# Claude Code CLI 유출 소스 분석

2026년 3월 유출된 Claude Code CLI 소스코드(약 52만 줄)를 Claude Opus와 OpenAI Codex 교차검증으로 분석한 18개 리포트입니다.

온라인 뷰어: https://aldegad.github.io/claude-code-analysis/

---

## 리포트 목록

| # | 파일 | 제목 |
|---|------|------|
| 01 | [01-project-overview.md](01-project-overview.md) | 프로젝트 개요 분석 |
| 02 | [02-architecture.md](02-architecture.md) | 아키텍처 및 디렉토리 구조 분석 |
| 03 | [03-tech-stack.md](03-tech-stack.md) | 기술 스택 및 의존성 분석 |
| 04 | [04-code-quality.md](04-code-quality.md) | 코드 품질 및 패턴 분석 |
| 05 | [05-core-modules.md](05-core-modules.md) | 핵심 모듈 상세 분석 |
| 06 | [06-security-and-testing.md](06-security-and-testing.md) | 보안 및 테스트 분석 |
| 07 | [07-hidden-features.md](07-hidden-features.md) | 미공개/예정 기능 심층 분석 |
| 08 | [08-structural-defects.md](08-structural-defects.md) | 구조적 결함 및 잠재적 버그 심층 분석 |
| 09 | [09-content-strategy.md](09-content-strategy.md) | 콘텐츠 전략 분석 |
| 10 | [10-third-party-comparison.md](10-third-party-comparison.md) | 서드파티 오픈소스 비교 분석 |
| 11 | [11-undercover-ethics.md](11-undercover-ethics.md) | Undercover Mode 윤리적 심층 분석 |
| 12 | [12-undercover-public-reaction.md](12-undercover-public-reaction.md) | Undercover Mode 공개 반응 및 논란 분석 |
| 13 | [13-telemetry.md](13-telemetry.md) | 텔레메트리/데이터 수집 심층 분석 |
| 14 | [14-billing.md](14-billing.md) | 과금/결제 시스템 분석 |
| 15 | [15-system-prompt.md](15-system-prompt.md) | 시스템 프롬프트 전문 분석 |
| 16 | [16-mcp-protocol.md](16-mcp-protocol.md) | MCP (Model Context Protocol) 구현 분석 |
| 17 | [17-performance.md](17-performance.md) | 성능 최적화 패턴 분석 |
| 18 | [18-easter-eggs.md](18-easter-eggs.md) | 이스터에그와 만우절 기능 분석 |

---

## 분석 방법론

Claude Opus 15개 세션과 OpenAI Codex 7개 세션으로 소스코드를 대조하며 팩트체크를 수행했습니다. 두 모델의 분석 결과를 교차검증하여 단일 모델의 오류나 환각을 최소화했습니다.

---

## 주의사항

이 분석은 유출된 소스코드를 기반으로 합니다. 코드 자체는 재배포하지 않으며, 분석 리포트만 포함되어 있습니다.
