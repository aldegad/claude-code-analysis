# 14. 과금/결제 시스템 분석 보고서

**분석자**: 부리 (분석팀 시장 분석가)
**분석일**: 2026-03-31
**대상**: Claude Code CLI (52만줄, Anthropic 유출 소스)

---

## 1. 전체 과금 아키텍처 요약

Claude Code는 **3중 과금 체계**를 운영한다:

| 계층 | 대상 | 과금 방식 |
|------|------|-----------|
| **구독 기반** | Claude.ai 사용자 (Pro/Max/Team/Enterprise) | 월정액 + 사용량 한도 |
| **종량제 (PAYG)** | API 키 사용자 (Console, Bedrock, Vertex, Foundry) | 토큰당 과금 |
| **x402 프로토콜** | MCP 외부 서비스 결제 | USDC 암호화폐 자동 결제 |

---

## 2. 구독 플랜 구조

### 2.1 코드에서 확인된 플랜 (SubscriptionType)

코드에 정의된 구독 유형은 정확히 **4개**:

```
'pro' | 'max' | 'team' | 'enterprise'
```

**Free 플랜은 코드에 별도 타입으로 존재하지 않는다.** 대신 `subscriptionType === null`로 처리되며, 이는 API 키 사용자와 동일하게 취급된다.

### 2.2 Max 플랜 세분화 (Rate Limit Tier)

Max 플랜은 내부적으로 **최소 2개의 하위 티어**로 분리된다:

| 티어 코드 | 설명 | 특징 |
|-----------|------|------|
| `default_claude_max_5x` | Team Premium과 동일 등급 | Opus 기본 모델 |
| `default_claude_max_20x` | **최상위 Max** | 업그레이드 불필요, Plan Mode 에이전트 3개 |

### 2.3 Team Premium

`isTeamPremiumSubscriber()` -- Team 구독 + `default_claude_max_5x` 티어.
Max와 동일하게 Opus가 기본 모델이 된다. 이는 공식 가격 페이지의 독립 상품명이라기보다, 코드와 레이트 리밋 티어 조합으로 식별되는 내부 구분이다.

### 2.4 조직 유형 매핑 (OAuth)

서버에서 내려오는 `organization_type`:
```
claude_max → 'max'
claude_pro → 'pro'
claude_enterprise → 'enterprise'
claude_team → 'team'
```

---

## 3. 플랜별 기능 비교표

| 기능 | Free/API(PAYG) | Pro | Max | Team | Team Premium | Enterprise |
|------|:-:|:-:|:-:|:-:|:-:|:-:|
| **기본 모델** | Sonnet 4.6 | Sonnet 4.6 | Opus 4.6 | Sonnet 4.6 | Opus 4.6 | Sonnet 4.6 |
| **Opus 접근** | O (과금) | O | O | O | O | O |
| **Haiku 접근** | O (과금) | O | O | O | O | O |
| **1M 컨텍스트** | O | Extra Usage 필요 | Extra Usage 필요 | Extra Usage 필요 | Extra Usage 필요 | Extra Usage 필요 |
| **Fast Mode (Opus 4.6)** | X (유료 필요) | O | O | O | O | O |
| **Computer Use** | X | O | O | X | X | X |
| **Voice Mode** | OAuth 필수 | O | O | O | O | O |
| **Plan Mode 에이전트 수** | 1 | 1 | 3(20x) / 1 | 3 | 3 | 3 |
| **사용량 표시** | 비용($) 표시 | 사용률(%) 표시 | 사용률(%) 표시 | 사용률(%) 표시 | 사용률(%) 표시 | 사용률(%) 표시 |
| **Extra Usage** | 해당없음 | O | O | 관리자 관리 | 관리자 관리 | 관리자 관리 |
| **/upgrade 명령** | 해당없음 | Max로 업그레이드 | 20x 아니면 업그레이드 | X | X | X |
| **Policy Limits** | X | X | X | O | O | O |
| **Billing 관리 접근** | Console 역할 기반 | 항상 O | 항상 O | 관리자/청구 역할 | 관리자/청구 역할 | 관리자/청구 역할 |
| **비용 경고** | PAYG만 | X | X | X | X | X |

---

## 4. 사용량 제한 시스템

### 4.1 Rate Limit 유형 (RateLimitType)

코드에 정의된 5가지 사용량 제한:

| 유형 | 코드 | 표시명 | 설명 |
|------|------|--------|------|
| 5시간 세션 | `five_hour` | Session limit | 5시간 단위 사용량 제한 |
| 7일 주간 | `seven_day` | Weekly limit | 주간 전체 사용량 |
| 7일 Opus | `seven_day_opus` | Opus limit | Opus 모델 전용 주간 제한 |
| 7일 Sonnet | `seven_day_sonnet` | Sonnet limit | Sonnet 모델 전용 주간 제한 |
| 초과 사용 | `overage` | Extra usage limit | Extra Usage 지출 한도 |

### 4.2 제한 상태 (QuotaStatus)

```
'allowed'          -- 정상 사용 가능
'allowed_warning'  -- 경고 (한도 접근 중)
'rejected'         -- 사용 거부 (한도 도달)
```

### 4.3 조기 경고 임계값

**5시간 세션 제한:**
- 사용률 90% + 시간 72% 이하 경과 시 경고

**7일 주간 제한:**
- 사용률 75% + 시간 60% 이하 → 경고
- 사용률 50% + 시간 35% 이하 → 경고
- 사용률 25% + 시간 15% 이하 → 경고

### 4.4 사용량 추적 헤더 (API 응답)

```
anthropic-ratelimit-unified-status
anthropic-ratelimit-unified-reset
anthropic-ratelimit-unified-fallback
anthropic-ratelimit-unified-representative-claim
anthropic-ratelimit-unified-{5h|7d}-utilization
anthropic-ratelimit-unified-{5h|7d}-reset
anthropic-ratelimit-unified-{5h|7d}-surpassed-threshold
anthropic-ratelimit-unified-overage-status
anthropic-ratelimit-unified-overage-reset
anthropic-ratelimit-unified-overage-disabled-reason
```

---

## 5. 과금 단위 및 비용 구조

### 5.1 API (PAYG) 과금 -- 토큰 기반

모든 PAYG 사용자는 **토큰 단위**로 과금된다. 가격은 **백만 토큰(Mtok)** 단위:

| 모델 | Input | Output | Cache Write | Cache Read | Web Search |
|------|------:|-------:|------------:|-----------:|-----------:|
| **Haiku 3.5** | $0.80 | $4 | $1 | $0.08 | $0.01/req |
| **Haiku 4.5** | $1 | $5 | $1.25 | $0.10 | $0.01/req |
| **Sonnet 4/4.5/4.6** | $3 | $15 | $3.75 | $0.30 | $0.01/req |
| **Opus 4/4.1** | $15 | $75 | $18.75 | $1.50 | $0.01/req |
| **Opus 4.5/4.6 (일반)** | $5 | $25 | $6.25 | $0.50 | $0.01/req |
| **Opus 4.6 (Fast Mode)** | **$30** | **$150** | $37.50 | $3 | $0.01/req |

**주요 발견**: Opus 4.6 Fast Mode는 일반 가격 대비 **6배** 비싼 특수 가격이 적용된다.

### 5.2 Extra Usage (초과 사용)

구독자가 한도를 초과했을 때 추가 과금되는 영역:

- **Fast Mode**: 항상 Extra Usage로 과금
- **1M 컨텍스트 (Opus 4.6)**: `isOpus1mMerged`가 아닌 경우 Extra Usage
- **1M 컨텍스트 (Sonnet 4.6)**: Extra Usage로 과금

```typescript
// src/utils/extraUsage.ts
isBilledAsExtraUsage(model, isFastMode, isOpus1mMerged):
  - Fast Mode → 항상 Extra Usage
  - Opus 4.6 [1m] → Opus1m 합병 아니면 Extra Usage
  - Sonnet 4.6 [1m] → Extra Usage
```

### 5.3 비용 표시 분기

| 사용자 유형 | 표시 방식 |
|-------------|-----------|
| PAYG (Console) | 실제 비용 `$X.XXXX` 표시 |
| 구독자 | 비용 비표시 (사용률 % 표시) |
| 비인증 | 비용 미표시 |

---

## 6. 결제 게이트웨이

### 6.1 Stripe (주 결제 시스템)

코드에서 확인되는 `billingType` 값:

```
'stripe_subscription'            -- 일반 Stripe 구독
'stripe_subscription_contracted' -- 계약형 Stripe 구독
'apple_subscription'             -- Apple IAP
'google_play_subscription'       -- Google Play 결제
```

**Stripe가 주 결제 게이트웨이**이며, Apple/Google 모바일 결제도 지원한다.
위 4개 billingType만 Extra Usage(초과 과금) 프로비저닝이 허용된다.
AWS Marketplace 같은 다른 결제 유형은 Extra Usage가 불가하다.

### 6.2 x402 (암호화폐 결제 -- 실험적/부분 구현 단계)

**실험적/부분 구현 단계.** 모듈과 서명 로직은 존재하나, 기본값 disabled이고 /x402 커맨드는 미등록, WebFetchTool과 미연결 상태이다.

#### x402 프로토콜 동작 흐름

```
1. 클라이언트 → 서버: HTTP 요청
2. 서버 → 클라이언트: 402 Payment Required + X-Payment-Required 헤더
3. 클라이언트: 결제 요건 파싱 (금액, 수신 지갑, 네트워크)
4. 클라이언트: 지출 한도 검증 (건당, 세션)
5. 클라이언트: EIP-712 서명 (EIP-3009 TransferWithAuthorization)
6. 클라이언트 → 서버: 재요청 + X-Payment 헤더 (base64 인코딩)
7. 서버: Facilitator(x402.org)를 통해 결제 검증
8. 서버 → 클라이언트: 200 OK + 리소스
```

#### x402 핵심 설정값

```typescript
X402_DEFAULTS = {
  enabled: false,           // 기본 비활성화
  network: 'base',          // Base 체인이 기본
  maxPaymentPerRequestUSD: 0.10,  // 건당 최대 $0.10
  maxSessionSpendUSD: 5.00,       // 세션당 최대 $5.00
}
```

#### 지원 네트워크

| 네트워크 | Chain ID | USDC 주소 |
|----------|----------|-----------|
| Base | 8453 | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` |
| Base Sepolia | 84532 | `0x036CbD53842c5426634e7929541eC2318f3dCF7e` |
| Ethereum | 1 | `0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48` |
| Ethereum Sepolia | 11155111 | `0x1c7D4B196Cb0C7B01d743Fbc6116a902379C7238` |

#### x402 통합 지점

- **fetch 래퍼**: Anthropic SDK 클라이언트의 fetch를 가로채서 402 자동 처리
- **axios 인터셉터**: WebFetchTool의 axios에 결제 인터셉터 추가
- **커맨드**: `/x402` (별칭: `/wallet`, `/pay`) -- 지갑 설정/상태/활성화/비활성화/한도 설정/제거

#### x402 보안 메커니즘

- **개인키 저장**: `~/.claude/config.json`에 파일 권한 600으로 저장
- **건당 한도**: 요청 1건당 최대 지불 금액 (기본 $0.10)
- **세션 한도**: 세션당 최대 총 지출 (기본 $5.00)
- **네트워크 검증**: 설정된 네트워크와 요청된 네트워크 일치 확인
- **토큰 검증**: USDC 컨트랙트 주소 검증
- **Nonce**: 랜덤 32바이트로 재전송 공격 방지

---

## 7. 무료 사용자 제한

**Free 플랜**은 코드에서 명시적으로 정의되지 않으나, 다음 제한이 적용된다:

1. **Fast Mode 차단**: `disabled_reason: 'free'` → "Fast mode requires a paid subscription"
2. **Computer Use 차단**: `hasRequiredSubscription()`이 Max/Pro만 허용
3. **Voice Mode 차단**: OAuth 인증 필수 (API 키 불가)
4. **Plan Mode 에이전트**: 1개로 제한
5. **Extra Usage**: 사용 불가
6. **모델**: Sonnet이 기본, Opus 접근 가능하나 과금됨

---

## 8. 기업용(Team/Enterprise) 전용 기능

### 8.1 Policy Limits

Team/Enterprise만 사용 가능한 관리자 정책 제한 시스템:
```typescript
// src/services/policyLimits/index.ts
// Enterprise/Team OAuth 사용자만 eligible
tokens.subscriptionType !== 'enterprise' && tokens.subscriptionType !== 'team'
```
- `allow_remote_sessions` 등 관리자 설정 가능한 정책
- CLAUDE_AI_INFERENCE_SCOPE 필요

### 8.2 Extra Usage 관리 체계

| 역할 | 권한 |
|------|------|
| 관리자 (admin/billing/owner) | 직접 설정 변경 |
| 일반 멤버 | 관리자에게 요청 전송 (`/extra-usage`) |

Overage 비활성화 사유 (총 12가지):
```
overage_not_provisioned / org_level_disabled / org_level_disabled_until
out_of_credits / seat_tier_level_disabled / member_level_disabled
seat_tier_zero_credit_limit / group_zero_credit_limit / member_zero_credit_limit
org_service_level_disabled / org_service_zero_credit_limit / no_limits_configured
```

### 8.3 Unified Rate Limit Fallback

Enterprise 사용자에 대한 특수 처리:
- 429 에러 시 자동 재시도 (PAYG 방식이므로)
- `unifiedRateLimitFallbackAvailable` 헤더로 폴백 가능 여부 전달

### 8.4 관리자 요청 시스템

비관리자 멤버가 한도 증가를 요청하는 시스템:
```typescript
createAdminRequest({ request_type: 'limit_increase', details: null })
getMyAdminRequests('limit_increase', ['pending', 'dismissed'])
checkAdminRequestEligibility('limit_increase')
```

---

## 9. 주목할 과금 정책 및 내부 구현

### 9.1 Fast Mode 가격 (프리미엄 요율)

Opus 4.6 Fast Mode는 $30/$150 per Mtok으로 **일반 Opus 4.6($5/$25)의 6배** 가격이다. 이 가격은 코드의 `COST_TIER_30_150`에 하드코딩되어 있으며, `/fast` 다이얼로그에서 `$30/$150 per Mtok` 가격과 `Billed as extra usage at a premium rate` 경고가 표시된다. 공식 가격 페이지(platform.claude.com)에도 `6x standard rates`로 안내된다.

### 9.2 x402 결제 시스템 (실험적/부분 구현)

Base/Ethereum 체인의 USDC를 통한 기계간(Machine-to-Machine) 암호화폐 결제 모듈과 서명 로직이 코드에 존재하나, 실험적/부분 구현 단계이다. 기본값 disabled이고 /x402 커맨드는 미등록, WebFetchTool과 미연결 상태다. Coinbase의 x402 프로토콜(https://github.com/coinbase/x402)을 기반으로 하며.

### 9.3 Team Premium 티어 (내부 구분)

Team 구독 + Max 5x 레이트 리밋 = Team Premium. Max와 동일한 Opus 기본 모델 + Plan Mode 3 에이전트. 공식 가격 페이지의 독립 상품명으로 분리되어 있지는 않지만, 코드상 별도 티어 조합으로 식별된다.

### 9.4 Max 20x 티어

`default_claude_max_20x` -- 최고 등급 Max 플랜. `/upgrade` 시 "You are already on the highest Max subscription plan" 표시. Plan Mode 에이전트 3개.

### 9.5 Opus Plan Mode (opusplan)

`opusplan` -- 계획 모드에서만 Opus를 사용하고 구현은 Sonnet으로 하는 하이브리드 모드. 모델 선택기에서는 숨겨져 있으나 직접 설정 가능.

### 9.6 1M 컨텍스트 과금 분리

1M 컨텍스트 사용 시 Extra Usage로 별도 과금된다 (Opus 1M 합병 플래그가 없는 경우). 이는 `isBilledAsExtraUsage()`에서 제어된다.

### 9.7 GrowthBook 피처 플래그 기반 과금 제어

다수의 과금 관련 기능이 GrowthBook A/B 테스트 플래그로 제어된다:
- `tengu_penguins_off` -- Fast Mode 비활성화
- `tengu_amber_quartz_disabled` -- Voice Mode 비활성화
- `tengu_malort_pedway` -- Computer Use 설정
- `tengu_jade_anvil_4` -- Rate limit 옵션 UI 순서

---

## 10. 결제 흐름 요약 다이어그램

```
[사용자 인증]
    │
    ├── OAuth (claude.ai) ────────────────────────────────────┐
    │   ├── Pro ($20/mo?)    → Sonnet 기본, 5h + 7d 제한    │
    │   ├── Max ($100/mo?)   → Opus 기본, 5h + 7d 제한      │
    │   ├── Max 20x          → 최상위, 3 에이전트            │
    │   ├── Team             → Sonnet 기본, Policy Limits    │
    │   ├── Team Premium     → Opus 기본, Max급              │
    │   └── Enterprise       → Sonnet 기본, Policy Limits    │
    │       └── Extra Usage  → 초과 시 종량 과금 ────────────┤
    │                                                         │
    ├── API Key (console.anthropic.com) ──────────────────────┤
    │   └── 토큰당 과금 (PAYG)                                │
    │       └── Stripe / 클라우드 결제                        │
    │                                                         │
    ├── 3P (Bedrock/Vertex/Foundry) ──────────────────────────┤
    │   └── 해당 클라우드 결제 시스템                          │
    │                                                         │
    └── x402 (암호화폐) ──────────────────────────────────────┘
        └── USDC on Base/Ethereum → MCP 도구 결제
```

---

## 11. 경쟁 임팩트 분석

### x402의 전략적 의미

1. **MCP 생태계 수익화**: 제3자 MCP 서비스가 자체 과금 가능
2. **마이크로페이먼트**: 건당 $0.10 이하의 소액 결제 가능
3. **Coinbase 파트너십 시사**: x402.org facilitator 사용
4. **탈중앙화 결제**: Stripe 의존 없는 대안 결제 채널

### Fast Mode의 수익 구조

Fast Mode가 6배 프리미엄인 이유:
- Extra Usage로 과금 → 구독 한도 외 추가 수익
- 고성능(속도) 대가로 프리미엄 징수
- Rate limit 시 자동 쿨다운 후 일반 모드 폴백

---

## 12. 핵심 코드 파일 참조

| 파일 | 역할 |
|------|------|
| `src/services/x402/types.ts` | x402 프로토콜 타입, USDC 주소, 기본 설정 |
| `src/services/x402/client.ts` | EIP-712 서명, 결제 생성/검증 |
| `src/services/x402/config.ts` | 지갑 설정/개인키 관리 |
| `src/services/x402/paymentFetch.ts` | fetch/axios 402 인터셉터 |
| `src/services/x402/tracker.ts` | 세션 결제 추적 |
| `src/utils/modelCost.ts` | 모델별 토큰 가격표 |
| `src/utils/auth.ts` | 구독 유형/티어 판별 |
| `src/utils/billing.ts` | 과금 접근 권한 |
| `src/utils/extraUsage.ts` | Extra Usage 과금 판별 |
| `src/utils/fastMode.ts` | Fast Mode 상태/비활성화 로직 |
| `src/services/claudeAiLimits.ts` | 사용량 한도 추적/경고 |
| `src/services/rateLimitMessages.ts` | 한도 도달 메시지 생성 |
| `src/utils/model/modelOptions.ts` | 플랜별 모델 선택기 |
| `src/utils/model/check1mAccess.ts` | 1M 컨텍스트 접근 권한 |
| `src/commands/upgrade/upgrade.tsx` | 구독 업그레이드 흐름 |
| `src/commands/extra-usage/extra-usage-core.ts` | Extra Usage 관리 |

---

*부리 분석 완료. x402 암호화폐 결제 시스템은 실험적/부분 구현 단계의 주목할 만한 발견이며, Fast Mode 6배 프리미엄과 Team Premium 티어 구분도 코드 분석상 확인되는 핵심 정보이다.*
