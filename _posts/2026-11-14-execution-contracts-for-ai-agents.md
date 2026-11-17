---
layout: post
title: "Execution Contracts Over Prompt Rules: Designing Reliable Tool Boundaries for AI Agents"
subtitle: "JSON schemas validate syntax, not semantics. How to build typed execution contracts, blast-radius sandboxing, and structured recovery for production agents."
date: 2026-11-14 09:00:00 +0900
categories: [Architecture]
tags: [ai-agent, tool-calling, sandboxing, reliability, autonomous-systems]
---

Most engineering teams treat LLM tool calling as a schema-generation problem. Define a Pydantic model or JSON Schema, attach a clear docstring, and let the model emit arguments.

In production, that assumption breaks on day one.

JSON Schema tells you that `amount` is an integer and `target_env` is a string. It does **not** tell you whether a refund exceeds the original invoice, whether an agent is about to wipe a production table instead of staging, or whether a subtle parameter mismatch will silently corrupt downstream state.

When you rely on system-prompt admonitions like *"Never delete production data"* or *"Always verify IDs before mutating"*, you are asking a probabilistic text generator to enforce deterministic invariants. **Prompt rules are advisory. Execution boundaries must be contractual.**

Here are four patterns we use to turn brittle LLM tool calls into deterministic, auditable execution boundaries.

---

## 1. Separate Semantic Validation from Schema Validation

Standard function-calling APIs only check structural types. Between the LLM's output and the actual side-effecting function, insert an **Execution Contract Layer** that checks domain invariants before anything runs:

```text
[LLM Tool Call]
      │
      ▼
[1. Syntax Check]      ──► JSON Schema / Type Validation
      │
      ▼
[2. Contract Check]    ──► Preconditions, Ownership, Limits, State Locks
      │
      ▼
[3. Side-Effecting API]
```

Consider an agent handling customer refunds. A syntactic schema passes `{ "order_id": "ord_991", "amount_cents": 50000 }`. The execution contract enforces:
- Does `order_id` belong to the authenticated tenant in the current session?
- Is the order status currently `CAPTURED` or `SETTLED`?
- Is `amount_cents <= order.refundable_balance`?

If any invariant fails, the contract rejects the call **before** the payment gateway is ever touched, returning a structured domain error rather than a generic 400.

---

## 2. Return Corrective Affordances, Not Stack Traces

When a tool fails in traditional software, raising an exception or returning `"InvalidParameterError: status mismatch"` is enough for a developer reading logs. For an autonomous agent, raw stack traces trigger blind retry loops or hallucinated workarounds.

Design every contract rejection as a **machine-actionable affordance**:

```json
{
  "status": "rejected",
  "reason": "AMOUNT_EXCEEDS_REFUNDABLE_BALANCE",
  "details": {
    "requested_cents": 50000,
    "max_refundable_cents": 32000
  },
  "next_valid_actions": [
    "Retry refund_order with amount_cents <= 32000",
    "Call escalate_to_billing_team if partial refund is insufficient"
  ]
}
```

When the tool boundary explicitly tells the reasoning loop *why* an invariant failed and *which transitions remain valid*, agents self-correct on the next turn without burning tokens guessing.

---

## 3. Capability-Scoped Sandboxing & Blast-Radius Tiers

Never hand an autonomous agent a monolithic admin token and hope prompt instructions restrict its reach. Scope credentials and tool capabilities dynamically per workflow phase:

- **Tier 0 (Read-Only Discovery)**: Unlimited read tools, zero mutation capabilities. Even if prompt-injected via retrieved web content or external emails, the agent literally lacks the handles to alter state.
- **Tier 1 (Reversible / Bounded Mutations)**: Creating drafts, staging records, posting to sandbox environments, or mutating resources bounded by strict rate and budget quotas.
- **Tier 2 (Irreversible / External Side Effects)**: Sending external emails, executing financial transfers, merging to `main`, or deleting records. These tools are not directly callable to completion—invoking them emits a signed intent payload for policy evaluation or human approval.

By binding tool visibility and credential scopes to the current state machine node, **blast radius becomes a compile-time property of your orchestrator, not a wish in your system prompt.**

---

## 4. Dry-Run Simulation Before High-Impact Execution

For multi-step mutations—such as batch database updates, infrastructure changes, or multi-recipient notifications—separate **plan generation** from **plan commitment**.

Require the agent to invoke a `preview_*` or `dry_run=true` contract first. The boundary computes the exact diff against live state (e.g., *"14 rows matched, 2 conflicts detected, $142.50 total delta"*) and returns a short-lived `confirmation_hash`. The actual commit tool requires that exact `confirmation_hash` as a mandatory parameter.

This achieves two things simultaneously:
1. The model cannot skip verification because the commit tool is cryptographically locked behind the dry-run output.
2. If underlying state changes between preview and execution, the hash invalidates automatically, preventing TOCTOU (Time-of-Check to Time-of-Use) race conditions.


<!-- lang:ko -->

# 🇰🇷 한국어 완역본 (Full Korean Translation)

### 프롬프트 규칙을 넘어 실행 계약으로: AI 에이전트를 위한 신뢰할 수 있는 툴 경계 설계

대부분의 엔지니어링 팀은 LLM의 툴 호출(Tool Calling)을 단순한 스키마 생성 문제로 다룹니다. Pydantic 모델이나 JSON Schema를 정의하고, 명확한 docstring을 달아둔 뒤, 모델이 인자(Argument)를 생성하도록 맡깁니다.

하지만 프로덕션 환경에서는 바로 첫날부터 그 전제가 무너집니다.

JSON Schema는 `amount`가 정수형이고 `target_env`가 문자열이라는 사실은 검증해 줍니다. 하지만 환불 금액이 원결제 금액을 초과하는지, 에이전트가 스테이징이 아니라 프로덕션 테이블을 날리려 하는지, 혹은 미묘한 파라미터 불일치가 하위 시스템의 상태를 조용히 망가뜨릴지는 **검증해 주지 못합니다.**

*"절대 프로덕션 데이터를 삭제하지 마세요"* 혹은 *"변경 전 항상 ID를 확인하세요"* 같은 시스템 프롬프트의 당부에 의존하는 것은, 확률적인 텍스트 생성기에게 결정론적인 불변식(Invariant)을 강제해 달라고 부탁하는 셈입니다. **프롬프트 규칙은 권고 사항일 뿐입니다. 실행 경계는 반드시 계약(Contract)이어야 합니다.**

부서지기 쉬운 LLM 툴 호출을 결정론적이고 감사 가능한 실행 경계로 바꾸기 위해 우리가 사용하는 네 가지 설계 패턴을 소개합니다.

---

## 1. 스키마 검증과 의미론적 검증의 분리

표준 함수 호출 API는 구조적인 타입만 검사합니다. LLM의 출력과 실제 부수 효과(Side-effect)를 일으키는 함수 사이에, 실행 전 도메인 불변식을 검사하는 **실행 계약 계층(Execution Contract Layer)**을 두어야 합니다.

```text
[LLM Tool Call]
      │
      ▼
[1. 문법 검증]         ──► JSON Schema / 타입 유효성 검사
      │
      ▼
[2. 계약 검증]         ──► 사전 조건, 소유권, 한도, 상태 잠금 확인
      │
      ▼
[3. 실제 실행 API]
```

고객 환불을 처리하는 에이전트를 예로 들어 봅시다. 문법적 스키마 검증은 `{ "order_id": "ord_991", "amount_cents": 50000 }`을 무사히 통과시킵니다. 하지만 실행 계약 계층은 다음을 강제합니다.
- `order_id`가 현재 세션에서 인증된 테넌트(사용자) 소유가 맞는가?
- 현재 주문 상태가 `CAPTURED` 또는 `SETTLED` 상태인가?
- `amount_cents <= order.refundable_balance` 조건을 만족하는가?

이 불변식 중 하나라도 어긋나면, 계약 계층은 결제 게이트웨이(PG)에 요청이 닿기 **전에** 호출을 거부하고, 모호한 400 에러 대신 구조화된 도메인 오류를 반환합니다.

---

## 2. 스택 트레이스가 아닌 교정 가능한 행동 지침(Affordance) 반환

전통적인 소프트웨어에서 툴이 실패할 때는 예외를 던지거나 `"InvalidParameterError: status mismatch"`를 반환해도 로그를 읽는 개발자에게는 충분합니다. 그러나 자율 에이전트에게 날것의 스택 트레이스를 던져주면, 맹목적인 재시도 루프에 빠지거나 잘못된 우회 경로를 환각(Hallucination)으로 지어내게 됩니다.

모든 계약 거부는 **기계가 즉시 행동으로 옮길 수 있는 교정 안내(Machine-actionable Affordance)** 형태로 설계해야 합니다.

```json
{
  "status": "rejected",
  "reason": "AMOUNT_EXCEEDS_REFUNDABLE_BALANCE",
  "details": {
    "requested_cents": 50000,
    "max_refundable_cents": 32000
  },
  "next_valid_actions": [
    "Retry refund_order with amount_cents <= 32000",
    "Call escalate_to_billing_team if partial refund is insufficient"
  ]
}
```

툴 경계가 추론 루프에 *왜* 불변식이 실패했는지, 그리고 *어떤 다음 전이가 여전히 유효한지*를 명시적으로 알려주면, 에이전트는 불필요한 추측으로 토큰을 낭비하지 않고 다음 턴에 스스로 오류를 교정합니다.

---

## 3. 권한 범위 기반 샌드박싱과 폭발 반경(Blast-Radius) 계층화

자율 에이전트에게 거대한 관리자 토큰을 통째로 쥐여주고, 프롬프트 지침만으로 접근 범위를 제한하기를 바라서는 안 됩니다. 워크플로 단계에 따라 자격 증명과 툴 사용 권한을 동적으로 제한해야 합니다.

- **Tier 0 (읽기 전용 탐색)**: 읽기 툴은 자유롭게 허용하되, 상태 변경 권한은 일절 부여하지 않습니다. 외부 웹페이지나 이메일을 통해 프롬프트 인젝션 공격을 당하더라도, 에이전트 손에 상태를 바꿀 수 있는 핸들 자체가 없으므로 안전합니다.
- **Tier 1 (가역적 / 제한된 변경)**: 초안 작성, 스테이징 데이터 생성, 샌드박스 환경 배포, 혹은 엄격한 빈도와 예산 쿼터 내에서 이루어지는 상태 변경입니다.
- **Tier 2 (비가역적 / 외부 파급 작업)**: 외부 이메일 발송, 금전 이체, `main` 브랜치 병합, 데이터 삭제 등이 해당합니다. 이러한 툴은 호출 즉시 완료되지 않고, 정책 평가나 사람의 최종 승인을 거치도록 서명된 의도 페이로드(Intent Payload)를 발행합니다.

툴의 노출 여부와 자격 증명 범위를 현재 상태 머신 노드에 묶어두면, **폭발 반경(Blast Radius)은 시스템 프롬프트 속 막연한 바람이 아니라 오케스트레이터의 구조적 속성으로 보장됩니다.**

---

## 4. 고위험 실행 전 드라이런(Dry-Run) 시뮬레이션 강제

일괄 데이터베이스 업데이트, 인프라 변경, 다수 수신자 알림 발송처럼 파급력이 큰 다단계 변경 작업에서는 **계획 생성(Plan Generation)**과 **계획 확정(Plan Commitment)**을 분리해야 합니다.

에이전트가 먼저 `preview_*` 또는 `dry_run=true` 계약을 호출하도록 강제하세요. 툴 경계는 실시간 상태와 대조해 정확한 변경 내역(예: *"14개 행 일치, 2건의 충돌 감지, 총 $142.50 차액 발생"*)을 계산하고, 짧은 유효기간을 가진 `confirmation_hash`를 반환합니다. 실제 확정 툴은 이 `confirmation_hash`를 필수 파라미터로 요구합니다.

이 구조는 두 가지 효과를 동시에 달성합니다.
1. 확정 툴이 드라이런 결과의 해시값으로 암호학적으로 잠겨 있으므로, 모델이 검증 단계를 임의로 건너뛸 수 없습니다.
2. 미리보기와 실제 실행 사이에 기반 상태가 변경되면 해시값이 자동으로 무효화되므로, 검사 시점과 사용 시점 사이의 경쟁 상태(TOCTOU)를 원천 차단합니다.

---

*Written by Seohun (@yosyus-Yo), Founder & AI Engineer at AITHOR.*  
*Published automatically via GitHub Pages & Pickle.*
