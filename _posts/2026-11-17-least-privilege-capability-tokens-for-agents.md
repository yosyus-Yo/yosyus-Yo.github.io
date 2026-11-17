---
layout: post
title: "Least-Privilege Agents: Capability Tokens and Sandboxed Tool Execution in Production"
subtitle: "Giving an LLM agent blanket API keys is a security incident waiting to happen. Here is how to enforce per-step capability tokens, argument-level policies, and ephemeral sandboxes."
date: 2026-11-17 09:00:00 +0900
categories: [Architecture]
tags: [ai-agent, security, sandboxing, capability-tokens, autonomous-systems]
---

Most early agent prototypes share a dangerous shortcut: the runtime loads a `.env` file full of broad API keys—GitHub personal access tokens, Slack bot tokens, database admin strings—and hands the tool registry to an LLM.

As soon as that agent reads untrusted external data (an incoming email, a GitHub issue comment, a scraped web page), **prompt injection turns ambient credentials into an exfiltration or destruction vector**. Even without a malicious actor, a confused model can accidentally drop a table, push to `main`, or email the wrong recipient.

Production agent runtimes cannot rely on *"please be careful"* system prompts. Security has to be enforced at the **tool execution boundary** using least privilege.

Here are four architectural controls for running autonomous agents safely with real-world tools.

---

## 1. Replace Ambient Credentials with Scoped Capability Tokens

An LLM should never hold—or even see—raw API secrets. Instead, the orchestrator should mint short-lived **capability tokens** scoped to the exact task and step:

```text
capability = {
  sub: "workflow_8f2a",
  tool: "github.create_comment",
  scope: "repo:acme/web-app#issue:412",
  exp: now + 60s
}
```

Key rules for capability minting:

1. **Zero ambient authority:** Tool workers reject any call lacking a signed capability token from the policy engine.
2. **Resource-level narrowing:** If the user asked the agent to summarize issue `#412`, the token allows reading `#412`—not listing private repositories or reading billing settings.
3. **Short TTLs:** Tokens expire in seconds or single-use invocations so a leaked or replayed payload is useless.

---

## 2. Enforce Argument-Level Policies Outside the Model

Role-based access (allowing or blocking an entire tool) is too coarse. For example, an email tool might be safe when replying to a known coworker, but risky when emailing an external domain with attachments.

Place a deterministic **policy gate** between the model's tool call and the actual executor:

- **Allowlisted targets:** Restrict `http_fetch` or `send_email` destinations to approved domains or recipients for that workflow.
- **Schema and range bounds:** Cap monetary amounts, batch sizes, or file path prefixes (e.g., `/workspace/drafts/**` only) before execution.
- **Taint tracking:** Tag any context retrieved from untrusted external text as `TAINTED`. When tainted data influences a high-impact write call, automatically pause and escalate for explicit human approval.

---

## 3. Run Code and File Tools Inside Ephemeral Sandboxes

Whenever an agent executes shell commands, parses untrusted PDFs, or runs generated Python scripts, it needs OS-level isolation—not just a regex filter on command strings.

A practical sandbox setup includes:

- **Isolated filesystem:** Mount only the task's working directory; keep host secrets, SSH keys, and sibling user workspaces invisible.
- **Egress filtering:** Block outbound network access by default, opening only specific ports or allowlisted registries (like PyPI or npm) when required.
- **Resource quotas:** Enforce strict CPU, memory, and wall-clock timeouts (`ulimit` / cgroups) so a runaway loop or zip bomb cannot starve neighboring workflows.
- **Disposable state:** Tear down the container or microVM after the task finishes so no residual artifacts persist across sessions.

---

## 4. Separate Read-Only Planning from Mutating Execution

Finally, split the agent's lifecycle into two privilege tiers:

1. **Planner / Researcher:** Runs with read-only tools and untrusted external inputs. It can search, read documents, and draft proposals, but it has **no write capabilities**.
2. **Executor:** Receives a structured, validated execution plan (or human-approved action card) and runs with tightly scoped write tokens—without re-ingesting raw untrusted web or email text during the commit phase.

By isolating untrusted inputs from write credentials, you eliminate the most common prompt-injection blast radius before a single API call fires.

---

## Closing Thought

Autonomy and security are not opposites. When every tool call carries a scoped capability token, passes deterministic argument checks, and runs inside a disposable sandbox, you can give your agent more capable tools—while keeping the worst-case failure strictly bounded.

---

# 최소 권한 에이전트 설계: 프로덕션 환경을 위한 역량 토큰(Capability Tokens)과 샌드박스 툴 실행

초기 AI 에이전트 프로토타입의 상당수는 위험한 지름길을 택하곤 합니다. 런타임이 GitHub 개인 액세스 토큰, Slack 봇 토큰, 데이터베이스 관리자 연결 문자열 같은 광범위한 API 키를 `.env` 파일에서 통째로 불러온 뒤, 그 권한을 그대로 LLM의 툴 레지스트리에 넘겨주는 방식입니다.

하지만 에이전트가 외부의 신뢰할 수 없는 데이터(수신된 이메일, GitHub 이슈 댓글, 웹 스크래핑 본문 등)를 읽는 순간, **프롬프트 인젝션(Prompt Injection)은 상시 노출된 자격 증명을 데이터 유출이나 시스템 파괴의 통로로 바꿔 버립니다.** 악의적인 공격자가 없더라도, 맥락을 혼동한 모델이 실수로 테이블을 삭제하거나 `main` 브랜치에 바로 푸시하고, 엉뚱한 수신자에게 메일을 보낼 수 있습니다.

프로덕션 환경의 에이전트 런타임은 *"조심해서 실행해 달라"*는 시스템 프롬프트에만 보안을 맡길 수 없습니다. 보안은 반드시 **툴 실행 경계(Tool Execution Boundary)**에서 최소 권한 원칙으로 강제되어야 합니다.

실제 외부 도구를 다루는 자율 에이전트를 안전하게 운영하기 위한 4가지 아키텍처 제어 패턴을 정리합니다.

---

## 1. 상시 자격 증명(Ambient Credentials)을 범위가 제한된 역량 토큰으로 대체하기

LLM은 원본 API 비밀 키를 직접 보유하거나 볼 수 없어야 합니다. 대신 오케스트레이터가 현재 작업과 단계에 꼭 맞게 수명이 짧은 **역량 토큰(Capability Token)**을 발급해야 합니다.

```text
capability = {
  sub: "workflow_8f2a",
  tool: "github.create_comment",
  scope: "repo:acme/web-app#issue:412",
  exp: now + 60s
}
```

역량 토큰 발급 시 지켜야 할 핵심 원칙은 다음과 같습니다.

1. **상시 권한 제로(Zero ambient authority):** 툴 워커는 정책 엔진이 서명한 역량 토큰이 없는 모든 호출을 거부합니다.
2. **리소스 단위 축소:** 사용자가 이슈 `#412` 요약을 요청했다면, 토큰은 `#412` 읽기만 허용할 뿐 비공개 저장소 목록 조회나 결제 설정 열람을 허용하지 않습니다.
3. **짧은 유효 시간(TTL):** 토큰이 몇 초 내 혹은 1회 실행 후 즉시 만료되게 하여, 페이로드가 유출되거나 재전송(Replay)되더라도 무력화합니다.

---

## 2. 모델 바깥에서 인자(Argument) 수준 정책 강제하기

툴 전체를 허용하거나 차단하는 역할 기반 접근 제어만으로는 너무 거칩니다. 예를 들어 이메일 전송 툴은 사내 동료에게 답장할 때는 안전하지만, 외부 도메인에 첨부파일을 보낼 때는 위험할 수 있습니다.

모델의 툴 호출과 실제 실행기 사이에 결정론적 **정책 게이트(Policy Gate)**를 배치하세요.

- **허용 목록(Allowlist) 기반 대상 제한:** 해당 워크플로에서 승인된 도메인이나 수신자에게만 `http_fetch` 또는 `send_email` 호출을 허용합니다.
- **스키마 및 범위 상한선:** 실행 전 결제 금액 상한, 배치 크기 제한, 허용된 파일 경로 접두사(예: `/workspace/drafts/**` 내부만 허용)를 검증합니다.
- **오염 추적(Taint Tracking):** 신뢰할 수 없는 외부 텍스트에서 읽어 온 컨텍스트에 `TAINTED` 태그를 붙입니다. 오염된 데이터가 영향력이 큰 쓰기(Write) 호출에 관여하면 자동으로 실행을 멈추고 사용자의 명시적 승인을 요청합니다.

---

## 3. 코드와 파일 툴은 일회성 샌드박스 내부에서 실행하기

에이전트가 셸 명령을 실행하거나, 신뢰할 수 없는 PDF를 파싱하거나, 직접 생성한 Python 스크립트를 돌릴 때는 명령어 문자열에 대한 정규식 필터가 아니라 OS 수준의 격리가 필요합니다.

실용적인 샌드박스 구성 요소는 다음과 같습니다.

- **격리된 파일시스템:** 해당 작업의 워크스페이스 디렉터리만 마운트하고, 호스트의 비밀 키나 SSH 키, 다른 사용자의 작업 공간은 보이지 않게 차단합니다.
- **이그레스(Egress) 필터링:** 기본적으로 외부 네트워크 접속을 막고, 꼭 필요한 경우에만 특정 포트나 허용된 패키지 레지스트리(PyPI, npm 등)만 열어 줍니다.
- **리소스 할당량:** CPU, 메모리, 실행 시간 제한(`ulimit` / cgroups)을 엄격히 걸어 무한 루프나 압축 폭탄(Zip Bomb)이 인접 워크플로의 자원을 고갈시키지 못하게 합니다.
- **일회성 상태(Disposable State):** 작업이 끝나면 컨테이너나 microVM을 즉시 폐기해 세션 간 잔여 파일이나 오염된 상태가 남지 않게 합니다.

---

## 4. 읽기 전용 계획 단계와 쓰기 실행 단계 분리하기

마지막으로 에이전트의 수명 주기를 두 가지 권한 계층으로 나눕니다.

1. **플래너 / 리서처(Planner / Researcher):** 읽기 전용 툴과 외부 비신뢰 입력을 함께 다룹니다. 검색하고 문서를 읽고 실행 초안을 만들 수는 있지만, **쓰기 권한은 전혀 갖지 않습니다.**
2. **실행기(Executor):** 검증된 구조화 실행 계획(또는 사용자가 승인한 액션 카드)만 전달받아 좁은 범위의 쓰기 토큰으로 동작합니다. 커밋 및 반영 단계에서는 신뢰할 수 없는 원본 웹페이지나 이메일 본문을 다시 컨텍스트에 들이지 않습니다.

비신뢰 입력과 쓰기 자격 증명을 구조적으로 분리하면, 단 한 번의 API 호출이 나가기 전에 가장 흔한 프롬프트 인젝션 피해 반경을 원천 차단할 수 있습니다.

---

## 마치며

자율성과 보안은 서로 반대되는 개념이 아닙니다. 모든 툴 호출이 범위가 제한된 역량 토큰을 지니고, 결정론적인 인자 검증을 통과하며, 일회성 샌드박스 안에서 실행된다면, 최악의 실패 반경을 엄격하게 통제하면서도 에이전트에게 훨씬 더 유능한 도구를 맡길 수 있습니다.
