---
layout: post
title: "Idempotent Agents: Designing Retry-Safe Workflows When Tool Calls Fail Mid-Flight"
subtitle: "Network timeouts and partial failures are inevitable in multi-step agent loops. Here is how to build idempotency keys, write-ahead intent logs, and saga rollbacks so retries never double-charge or duplicate side effects."
date: 2026-11-16 09:00:00 +0900
categories: [Architecture]
tags: [ai-agent, reliability, idempotency, distributed-systems, autonomous-systems]
---

When an LLM agent calls a read-only tool and the request times out, retrying is free. When it calls an external write tool—sending an email, creating a calendar invite, charging a card, or dispatching a webhook—a blind retry is dangerous.

In distributed systems, this is the classic **at-least-once delivery** problem. In agentic systems, it is worse: the model itself often decides to retry when it sees an ambiguous error or a truncated response, turning a transient 504 timeout into two identical emails or a double booking.

If you want autonomous agents to operate safely in production, retry safety cannot be left to prompt instructions like *"do not repeat actions."* It has to be built into the execution layer.

Here are four patterns for building **idempotent, retry-safe agent workflows**.

---

## 1. Bind Every Side Effect to a Deterministic Idempotency Key

An agent loop should never invoke a mutating tool with raw parameters alone. Every write operation needs an **idempotency key** derived from the workflow step rather than the retry attempt:

```text
step_key = hash(workflow_id + step_index + target_entity + canonical_payload)
```

When the tool wrapper executes:

1. Check a local or Redis-backed **effect ledger** for `step_key`.
2. If the key is marked `COMMITTED`, return the cached receipt immediately without hitting the external API.
3. If the key is marked `IN_FLIGHT`, block concurrent retries and resolve the prior attempt first.
4. Otherwise, record `IN_FLIGHT`, forward `step_key` to the downstream API (if supported), and persist the receipt upon completion.

This single boundary ensures that even if the LLM hallucinates a duplicate tool call three turns later in the same workflow, the runtime returns the original outcome instead of executing the mutation twice.

---

## 2. Use Write-Ahead Intent Logs Before External Mutations

The hardest failure mode is the **ambiguous timeout**: the runtime sends `POST /bookings`, the network drops before the HTTP 200 arrives, and you do not know whether the server committed the booking.

```text
[Agent] ──► (1) Write INTENT to WAL (status: PENDING, key: k1)
        ──► (2) Execute external API call
        ──X (3) Network timeout! Outcome unknown.
        ──► (4) Recovery: Query external state by k1 / correlation_id
                    ├─ Found     ──► Mark COMMITTED, return receipt
                    └─ Not found ──► Safe to retry or abort
```

A **Write-Ahead Log (WAL)** records the agent's *intent* before the network call leaves the process. On recovery, the runtime never guesses—it runs a **reconciliation read** against the external system using the correlation ID or natural key (e.g., checking Sent Mail or listing recent reservations) before allowing any retry.

---

## 3. Separate Compensatable Actions from Irreversible Gates

Not every external action can be undone. Sending an HTTP request to reserve inventory can be compensated with a cancellation API; sending an email to a customer or executing a wire transfer cannot.

Classify every tool in your agent's registry into one of three tiers:

| Tier | Examples | Failure / Retry Strategy |
| --- | --- | --- |
| **Read / Pure** | Search, fetch, status check | Safe to retry immediately with exponential backoff |
| **Compensatable Write** | Drafts, calendar holds, cart items | Execute with idempotency key + register a compensating rollback action (Saga) |
| **Irreversible Commit** | Send email, submit payment, delete record | Require WAL reconciliation + human approval or strict pre-commit check |

By isolating irreversible commits at the very end of a plan—and doing all preparation in drafts or holds first—you shrink the blast radius of mid-flight failures dramatically.

---

## 4. Return Structured Ambiguity to the Model

When a tool fails with an uncertain outcome, **never** return a generic string like `"Error: timeout"` to the LLM. To an LLM, `"Error: timeout"` looks like *"it didn't work, try again."*

Instead, return a structured state object that explicitly forbids blind retries:

```json
{
  "status": "UNCERTAIN_OUTCOME",
  "action": "send_invoice_email",
  "idempotency_key": "wf_92a1_step_04",
  "safe_to_retry": false,
  "next_step": "Run check_sent_mail(correlation_id='wf_92a1_step_04') before taking further action."
}
```

When the tool response makes uncertainty explicit and names the exact verification step, the model stops guessing and follows the recovery path.

---

## Summary

Reliable agents are not built by hoping the network stays up or asking the LLM to be careful. They are built by treating every tool call like a distributed transaction: **idempotency keys** for deduplication, **write-ahead logs** for crash recovery, **saga tiers** for blast-radius control, and **structured ambiguity** so the model never turns a timeout into a duplicate side effect.

---

<!-- lang:ko -->

# [한국어 완역] 멱등한 에이전트 설계하기: 툴 호출이 중간에 실패해도 안전하게 재시도하는 워크플로

LLM 에이전트가 읽기 전용(Read-only) 툴을 호출하다가 타임아웃이 나면, 그냥 다시 시도하면 그만입니다. 하지만 이메일 발송, 캘린더 초대장 생성, 카드 결제, 웹훅 전송 같은 외부 쓰기(Write) 툴을 호출할 때 무작정 재시도하는 것은 매우 위험합니다.

분산 시스템에서는 이를 고전적인 **최소 1회 전달(At-least-once delivery)** 문제라고 부릅니다. 에이전트 시스템에서는 상황이 더 까다롭습니다. 모호한 에러나 잘린 응답을 본 모델이 스스로 판단해 재시도를 감행하면서, 일시적인 504 타임아웃을 '동일한 메일 2통 발송'이나 '중복 예약'으로 키워버리기 때문입니다.

프로덕션 환경에서 자율 에이전트를 안전하게 운영하려면, 재시도 안전성을 *"같은 행동을 반복하지 마세요"* 같은 프롬프트 지시에 맡겨서는 안 됩니다. 실행 레이어(Execution Layer) 자체에 구조적으로 심어야 합니다.

**멱등하고(Idempotent) 재시도에 안전한 에이전트 워크플로**를 구축하는 4가지 아키텍처 패턴을 정리합니다.

---

## 1. 모든 부수 효과(Side Effect)에 결정적 멱등성 키를 바인딩하기

에이전트 루프가 상태를 변경하는(Mutating) 툴을 날것의 파라미터만으로 호출하게 두면 안 됩니다. 모든 쓰기 작업에는 재시도 횟수가 아니라 **워크플로 단계 자체에서 파생된 멱등성 키(Idempotency Key)**가 필요합니다.

```text
step_key = hash(workflow_id + step_index + target_entity + canonical_payload)
```

툴 래퍼(Wrapper)는 실행 시 다음 순서로 동작합니다.

1. 로컬 또는 Redis 기반의 **실행 원장(Effect Ledger)**에서 `step_key`를 조회합니다.
2. 키가 이미 `COMMITTED` 상태라면 외부 API를 호출하지 않고 저장해 둔 영수증(결과)을 즉시 반환합니다.
3. `IN_FLIGHT`(진행 중) 상태라면 동시 재시도를 막고 이전 시도의 결과를 먼저 확정합니다.
4. 기록이 없다면 `IN_FLIGHT`로 마킹한 뒤, 하위 API에 `step_key`를 전달해 실행하고 완료 시 영수증을 영속화합니다.

이 경계 하나만 있어도, LLM이 같은 워크플로 안에서 몇 턴 뒤에 실수로 똑같은 툴을 다시 호출하더라도 런타임이 중복 실행 대신 원래 결과를 돌려줄 수 있습니다.

---

## 2. 외부 변경 전 Write-Ahead Intent Log(WAL) 기록하기

가장 다루기 힘든 장애는 **모호한 타임아웃(Ambiguous Timeout)**입니다. 런타임이 `POST /bookings`를 보냈는데 HTTP 200 응답이 오기 전에 네트워크가 끊기면, 서버가 예약을 확정했는지 안 했는지 알 수 없습니다.

```text
[Agent] ──► (1) WAL에 의도(INTENT) 기록 (status: PENDING, key: k1)
        ──► (2) 외부 API 호출 실행
        ──X (3) 네트워크 타임아웃! 결과 불명확
        ──► (4) 복구: k1 / correlation_id로 외부 상태 선조회
                    ├─ 존재함   ──► COMMITTED 마킹 후 결과 반환
                    └─ 존재 안 함 ──► 안전하게 재시도 또는 중단
```

**Write-Ahead Log(WAL)**는 네트워크 요청이 프로세스를 떠나기 *전에* 에이전트의 의도를 먼저 기록합니다. 장애 복구 시 런타임은 추측하지 않습니다. 재시도를 허용하기 전에 먼저 상관관계 ID(Correlation ID)나 자연 키를 이용해 외부 시스템에 **대사 조회(Reconciliation Read)**(예: 보낸 메일함 확인, 최근 예약 목록 조회)를 수행합니다.

---

## 3. 보상 가능한 액션과 되돌릴 수 없는 커밋 분리하기

모든 외부 액션을 취소할 수 있는 것은 아닙니다. 재고 예약 API는 취소 API로 보상(Compensate)할 수 있지만, 고객에게 이미 발송된 이메일이나 계좌 이체는 되돌릴 수 없습니다.

에이전트 레지스트리의 모든 툴을 아래 3단계 티어로 분류하세요.

| 티어 | 예시 | 실패 / 재시도 전략 |
| --- | --- | --- |
| **읽기 / 순수 연산** | 검색, 조회, 상태 확인 | 지수 백오프(Exponential backoff)로 즉시 재시도 가능 |
| **보상 가능한 쓰기** | 임시저장, 캘린더 가예약, 장바구니 담기 | 멱등성 키와 함께 실행 + 보상 롤백 액션(Saga) 등록 |
| **되돌릴 수 없는 커밋** | 메일 발송, 결제 승인, 레코드 삭제 | WAL 대사 조회 + 사용자 승인 또는 엄격한 사전 검증 필수 |

되돌릴 수 없는 커밋을 계획의 맨 마지막 단계로 격리하고, 그 전까지는 임시저장이나 가예약 상태로 모든 준비를 끝내두면 중간 실패로 인한 피해 반경을 극적으로 줄일 수 있습니다.

---

## 4. 모델에게 '구조화된 불확실성' 반환하기

툴이 불확실한 결과로 실패했을 때, LLM에게 `"Error: timeout"` 같은 단순 문자열을 돌려주어서는 **절대** 안 됩니다. LLM 입장에서 `"Error: timeout"`은 *"실패했으니 다시 시도해 보자"*는 신호로 읽히기 쉽습니다.

대신 맹목적인 재시도를 명시적으로 금지하는 구조화된 상태 객체를 반환해야 합니다.

```json
{
  "status": "UNCERTAIN_OUTCOME",
  "action": "send_invoice_email",
  "idempotency_key": "wf_92a1_step_04",
  "safe_to_retry": false,
  "next_step": "Run check_sent_mail(correlation_id='wf_92a1_step_04') before taking further action."
}
```

툴 응답 자체가 불확실성을 분명히 밝히고 다음 검증 단계를 정확히 지시하면, 모델은 추측으로 재시도하는 대신 안전한 복구 경로를 따르게 됩니다.

---

## 요약

신뢰할 수 있는 에이전트는 네트워크가 끊기지 않기를 바라거나 LLM에게 조심해 달라고 부탁해서 만들어지지 않습니다. 모든 툴 호출을 분산 트랜잭션처럼 다뤄야 합니다. 중복 방지를 위한 **멱등성 키**, 크래시 복구를 위한 **Write-Ahead Log**, 피해 반경 제어를 위한 **Saga 티어 분리**, 그리고 타임아웃이 중복 실행으로 번지지 않게 막는 **구조화된 불확실성 응답**이 결합될 때 비로소 안심하고 맡길 수 있는 자율 워크플로가 완성됩니다.
