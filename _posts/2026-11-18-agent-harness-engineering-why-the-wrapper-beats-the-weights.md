---
layout: post
title: "Agent Harness Engineering: Why the Wrapper Matters More Than the Weights"
subtitle: "November 2026 benchmarks and production architectures—from CUGA vs. Hermes to Red Hat's enforcement boundary and RL-adapted harness weights—show that reliability lives in the harness."
date: 2026-11-18 09:00:00 +0900
categories: [Architecture]
tags: [agent-harness, hermes-agent, ai-engineering, autonomous-systems, evals]
---

For the past two years, the default reflex in AI engineering was simple: if an agent failed a task, wait for a smarter foundation model.

In late 2026, that assumption has collapsed under controlled evidence. Across industry benchmarks, open-source agent runtimes, and production post-mortems, **agent harness engineering**—the architecture wrapping the model with context, tool interfaces, loop guards, state ownership, and enforcement—has emerged as the primary determinant of whether an autonomous system actually works.

Here is what the latest November 2026 benchmarks, runtime architectures, and research tell us about building reliable harnesses.

---

## 1. Same Model, 50-Point Swing: The Harness Is the Bottleneck

In a controlled experiment published by IBM Research in November 2026, engineers ran three different open-source harnesses—**CUGA**, **Hermes**, and **Deep Agents**—on the exact same AppWorld benchmark (90 multi-step tasks across 30 scenarios spanning 9 consumer apps and 457 APIs), using the **exact same open-weight model (`GLM-5.3-flash`)** under the Harbor evaluation framework.

The model did not change. Only the harness did:

| Harness | Task Goal Completion (TGC) | Scenario Goal Completion (SGC, all 3 variants pass) |
| --- | --- | --- |
| **CUGA** | **81.1%** (73/90) | **66.7%** (20/30) |
| **Hermes** | 46.7% (42/90) | 26.7% (8/30) |
| **Deep Agents** | 30.0% (27/90) | 6.7% (2/30) |

The gap in **Scenario Goal Completion (SGC)**—which requires an agent to pass all three variations of a scenario consistently rather than getting lucky once—is stark: **66.7% vs. 6.7%**.

CUGA's AppWorld configuration did not possess a better LLM. It won because its harness matched the job: structured task decomposition, verification before execution, and automatic reflection tuned for multi-app API workflows. Meanwhile, on coding-agent consistency leaderboards analyzing nearly 29,000 trials, **Hermes Agent** led overall reliability for consecutive-attempt success (reaching a 71.3% solve rate on DeepSeek-V4.1-Flash), proving that **harness-task fit** matters far more than generic "agentic" scaffolding.

---

## 2. Move Capabilities In, Move Enforcement Out

As reasoning models absorb planning, native tool calling, and self-correction, many teams are tempted to strip down their orchestration code. Red Hat Developer's November 2026 analysis on harness engineering identifies the exact trap in that instinct:

- **Capabilities move into the model:** Multi-step planning, formatting tool arguments, and recovering from minor syntax errors are now better handled by the model itself. Hard-coding brittle prompt chains around them creates wasted scaffolding.
- **Enforcement must move out of the model:** Properties that must remain true *even when the model is wrong, confused, or prompt-injected* cannot live in a system prompt.

Specifically, production harnesses must enforce these invariants in deterministic code outside the LLM:

1. **Cryptographic identity & permissions:** A model that "respects permissions" in a prompt has no real permissions. Tool workers must validate signed tokens and argument policies independently.
2. **Isolation & custody:** Secrets, OAuth tokens, and third-party plugin credentials belong to the host runtime, never the model's context window.
3. **Hard budgets & loop guards:** Unattended agents will happily burn a month's inference budget retrying a broken call. In **Hermes Agent**, for instance, `tool_guardrails` warns on the 3rd identical call, detects repeating cycles up to period 4, and triggers a hard no-progress stop at 5 on unattended cron and gateway runs.
4. **Checkpoint recovery:** Resuming from a deterministic state checkpoint rather than a lossy LLM-generated summary.

The paradox of modern AI engineering is clear: **the smarter the model gets, the more dangerous a weak harness becomes.**

---

## 3. Single-Writer Gateways and Custody Rules "Even in YOLO Mode"

Recent architectural updates in **Hermes Agent** (detailed in *The State of Hermes Agent — November 2026*) highlight a practical lesson from real-world multi-surface deployments.

Early agent setups allowed the CLI, TUI, desktop client, editor plugins, chat bots, and cron jobs to each spin up their own agent loop against the same conversation—causing duplicate writes, race conditions, and lost messages. Modern harness design solves this with a **single gateway per machine**:

- **One queue, one history, one writer per session:** Every client surface attaches to the central gateway rather than mutating state independently.
- **Autonomy != Custody:** Even when a user enables permissive auto-approve flags (such as `--yolo`), custody rules remain locked—paid-tool spend limits still pause for confirmation, locked user skills cannot be overwritten by the agent, and OAuth tokens are injected by the core runtime only for declared hosts.

Separating *autonomy settings* (what the agent is allowed to attempt) from *custody rules* (what assets and invariants the agent can never bypass) is fast becoming standard harness hygiene.

---

## 4. Co-Designing Models and Harnesses

The newest frontier in November 2026 goes one step further: **adapting open models directly to a specific harness**.

Hugging Face recently released `Qwen3.8-27B-pi-adapted`, a ~640 MB rank-32 LoRA adapter trained with reinforcement learning *inside the `pi` coding agent harness*. Keeping the base 27B weights fixed and optimizing purely for how the model interacts with `pi`'s tools and loop yielded dramatic gains:

- **SWE-bench Pro (Hard 51):** jumped from **29.4% to 62.7%** (`pass@1`)
- **Terminal-Bench 2.1 (89 tasks):** jumped from **39.5% to 55.8%**
- **Efficiency:** **55% to 62% fewer tool calls** per task

Alongside automated harness search frameworks like **HarnessFlow** (using Monte Carlo Tree Search to refine executable harnesses under fixed evaluation budgets), the message for AI engineers is unmistakable.

---

## Takeaways for Builders

1. **Benchmark your harness, not just your model:** Hold the model fixed and measure multi-run consistency (e.g., passing 3 out of 3 variants), not single-run `pass@1` luck.
2. **Delete cognitive scaffolding, harden infrastructure:** Let strong models plan, but enforce budgets, loop stops, sandboxing, and token custody in code.
3. **Centralize state ownership:** Use a single writer/gateway per session when supporting interactive chat, CLI, and background cron runs together.
4. **Record what the agent actually saw:** Production logs only work as evaluations if they capture the exact context, permissions, and tool outputs exposed to the model at each step.

The weights provide the intelligence. The **harness** turns it into engineering.

<!-- lang:ko -->

지난 2년 동안 AI 엔지니어링의 기본 해법은 단순했습니다. 에이전트가 작업을 실패하면 더 똑똑한 파운데이션 모델이 나오기를 기다리는 것이었습니다.

하지만 2026년 후반, 이 가정은 통제된 실험 데이터 앞에서 무너졌습니다. 최신 벤치마크와 오픈소스 에이전트 런타임, 프로덕션 장애 회고록들은 한목소리로 말합니다. 자율 시스템의 실제 성패를 가르는 핵심 병목은 모델 가중치가 아니라, 모델 바깥에서 컨텍스트·툴 인터페이스·루프 가드·상태 소유권·보안 정책을 감싸는 **에이전트 하네스 엔지니어링(Agent Harness Engineering)**이라는 점입니다.

2026년 11월 기준 최신 벤치마크와 오픈소스 아키텍처, 연구 동향이 보여주는 핵심 원칙을 정리합니다.

---

## 1. 같은 모델, 50%p 격차: 병목은 하네스에 있다

2026년 11월 IBM Research가 공개한 통제 실험은 하네스의 영향력을 극명하게 보여줍니다. 연구진은 Harbor 평가 프레임워크 위에서 **완전히 동일한 오픈웨이트 모델(`GLM-5.3-flash`)**을 고정한 채, 세 가지 오픈소스 하네스(**CUGA**, **Hermes**, **Deep Agents**)만으로 AppWorld 벤치마크(9개 앱·457개 API에 걸친 30개 시나리오, 총 90개 멀티스텝 태스크)를 실행했습니다.

모델은 단 하나도 바뀌지 않았고, 오직 하네스만 달랐습니다.

| 하네스(Harness) | 태스크 완료율 (TGC) | 시나리오 완료율 (SGC, 3개 변형 모두 통과) |
| --- | --- | --- |
| **CUGA** | **81.1%** (73/90) | **66.7%** (20/30) |
| **Hermes** | 46.7% (42/90) | 26.7% (8/30) |
| **Deep Agents** | 30.0% (27/90) | 6.7% (2/30) |

우연히 한 번 성공하는 것이 아니라 동일 시나리오의 3가지 변형을 모두 통과해야 인정되는 **SGC(Scenario Goal Completion)** 지표에서 **66.7% 대 6.7%**라는 압도적인 차이가 벌어졌습니다.

CUGA가 이긴 이유는 더 좋은 LLM을 써서가 아닙니다. 멀티 앱 API 작업에 맞춰 태스크 분해, 실행 전 검증, 자동 반성(reflection) 루프를 하네스 수준에서 정교하게 설계했기 때문입니다. 반대로 약 29,000건의 코딩 에이전트 연속 성공률을 측정한 Harness Leaderboard에서는 **Hermes Agent**가 전체 1위(DeepSeek-V4.1-Flash 기준 71.3%)를 기록했습니다. 결국 범용적인 스캐폴딩보다 **작업 특성에 맞는 하네스 설계**가 성능을 결정짓습니다.

---

## 2. 역량(Capability)은 모델 안으로, 강제(Enforcement)는 모델 밖으로

추론 모델이 발전하면서 계획 수립(Planning)과 네이티브 툴 호출, 자가 수정까지 모델이 직접 처리하게 되자, 많은 팀이 오케스트레이션 코드를 덜어내기 시작했습니다. 하지만 2026년 11월 Red Hat Developer가 짚은 것처럼, 여기에는 치명적인 함정이 있습니다.

- **역량은 모델 안으로 이동한다:** 멀티스텝 계획이나 툴 인자 포맷팅처럼 학습 가능한 능력은 모델이 흡수하는 편이 낫습니다. 이를 외부 프롬프트 체인으로 억지로 감싸는 것은 불필요한 비용만 늘립니다.
- **강제는 반드시 모델 밖으로 빼야 한다:** 모델이 틀리거나, 혼란에 빠지거나, 프롬프트 인젝션에 당하더라도 **절대 깨져서는 안 되는 불변식(Invariant)**은 시스템 프롬프트에 둘 수 없습니다.

프로덕션 하네스가 모델 바깥의 결정론적 코드로 강제해야 할 핵심 요소는 다음과 같습니다.

1. **암호화된 신원과 권한:** 프롬프트로 "권한을 지켜라"고 지시하는 것은 보안이 아닙니다. 툴 워커가 서명된 토큰과 인자 정책을 독립적으로 검증해야 합니다.
2. **격리와 자격 증명 보관:** API 키, OAuth 토큰, 플러그인 비밀값은 호스트 런타임이 보관하며 모델 컨텍스트에 절대 노출하지 않습니다.
3. **예산 상한과 루프 가드:** 무인(unattended) 에이전트는 실패한 호출을 반복하며 한 달 치 추론 예산을 순식간에 태울 수 있습니다. 예를 들어 **Hermes Agent**의 `tool_guardrails`는 동일 호출이 3회 반복되면 경고를 보내고, 최대 주기 4의 반복 사이클을 탐지하며, 크론(cron)·게이트웨이 같은 무인 실행 환경에서는 진전이 없을 때 5회째에 강제 중단합니다.
4. **체크포인트 복구:** 손실이 큰 LLM 요약본이 아니라 결정론적 상태 체크포인트에서 작업을 재개해야 합니다.

모델이 똑똑해질수록 부실한 하네스는 오히려 더 위험해집니다.

---

## 3. 단일 쓰기 게이트웨이와 'YOLO 모드'에서도 유지되는 보관 규칙

2026년 11월 **Hermes Agent** 아키텍처 보고서(*The State of Hermes Agent*)는 멀티 인터페이스 환경에서 얻은 실전 교훈을 잘 보여줍니다.

초기에는 CLI, TUI, 데스크톱 앱, 에디터, 챗봇, 크론 작업이 각자 독립된 에이전트 루프를 돌려 같은 대화에 쓰기를 시도하면서 메시지 유실과 중복 기록이 발생했습니다. 이를 해결하기 위해 최신 하네스는 **머신당 단일 게이트웨이(Single Gateway)** 구조를 채택합니다.

- **세션당 하나의 큐, 하나의 히스토리, 하나의 라이터(Writer):** 모든 클라이언트는 중앙 게이트웨이에 붙어 상태를 일관되게 관리합니다.
- **자율성(Autonomy) != 보관 책임(Custody):** 사용자가 `--yolo` 같은 자동 승인 옵션을 켜더라도, 유료 툴 지출 한도 초과 시 확인 요청, 잠긴 사용자 스킬 수정 금지, 코어 런타임의 OAuth 토큰 대리 주입 같은 보관 규칙은 절대 해제되지 않습니다.

에이전트가 "무엇을 시도할 수 있는가"(자율성)와 "무엇을 절대 우회할 수 없는가"(보관·보안 규칙)를 분리하는 것이 표준 설계로 자리 잡고 있습니다.

---

## 4. 모델과 하네스의 공동 설계(Co-Design)

2026년 11월의 가장 앞선 흐름은 아예 **특정 하네스에 맞춰 오픈 모델을 직접 적응시키는 것**입니다.

최근 Hugging Face가 공개한 `Qwen3.8-27B-pi-adapted`는 `pi` 코딩 에이전트 하네스 환경 내부에서 강화학습(RL)으로 훈련한 약 640MB 크기(rank-32)의 LoRA 어댑터입니다. 기반 27B 모델은 그대로 둔 채 특정 하네스의 툴 루프에 맞춰 어댑터만 학습시켰을 뿐인데도 놀라운 결과를 냈습니다.

- **SWE-bench Pro (Hard 51):** `pass@1` 기준 **29.4% → 62.7%** 급등
- **Terminal-Bench 2.1 (89개 태스크):** **39.5% → 55.8%** 향상
- **효율성:** 태스크당 툴 호출 횟수 **55%~62% 감소**

여기에 MCTS(몬테카를로 트리 탐색)로 제한된 평가 예산 안에서 실행 가능한 하네스 코드를 자동 탐색·개선하는 **HarnessFlow** 같은 연구까지 더해지며, 모델과 하네스를 함께 최적화하는 시대가 열리고 있습니다.

---

## 실무 적용을 위한 핵심 정리

1. **모델이 아니라 하네스를 벤치마킹하세요:** 모델을 고정한 채 단일 실행 점수가 아니라 다회 실행 일관성(예: 변형 3종 모두 통과)을 측정해야 합니다.
2. **인지 스캐폴딩은 줄이고, 인프라 방어선은 높이세요:** 계획은 좋은 모델에 맡기되 예산, 루프 차단, 샌드박싱, 토큰 보관은 코드로 강제하세요.
3. **상태 소유권을 단일화하세요:** 대화형 UI, CLI, 백그라운드 스케줄러가 함께 돌 때는 단일 게이트웨이/라이터 패턴으로 동시성 충돌을 막아야 합니다.
4. **에이전트가 '실제로 본 것'을 기록하세요:** 프로덕션 로그가 평가 데이터셋(Eval)으로 쓰이려면 모델의 출력뿐 아니라 그 시점에 모델이 볼 수 있었던 컨텍스트와 권한, 툴 결과를 함께 남겨야 합니다.

지능은 모델 가중치에서 나오지만, **신뢰할 수 있는 시스템은 하네스에서 나옵니다.**
