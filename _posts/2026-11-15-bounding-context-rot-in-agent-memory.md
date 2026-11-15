---
layout: post
title: "Why Agent Memory Fails Over Time: Bounding Context Rot and Retrieval Drift"
subtitle: "Appending everything to a vector store works for a week. Here is how to design tiered memory, conflict resolution, and write-time compaction for long-running autonomous agents."
date: 2026-11-15 09:00:00 +0900
categories: [Architecture]
tags: [ai-agent, memory-systems, context-engineering, rag, autonomous-systems]
---

Most agent prototypes start with the same memory architecture: embed every user turn and tool output into a vector database, retrieve the top-$k$ chunks at inference time, and stuff them into the system prompt.

For the first few sessions, it feels magical. By week four, the agent starts contradicting yesterday's decisions, hallucinating obsolete preferences, and burning half its context window on stale logs.

This failure mode is **Context Rot** paired with **Retrieval Drift**. Semantic similarity alone cannot distinguish a superseded draft from a settled decision, nor an old debugging trace from an active user rule.

Here are four architectural patterns to keep long-running agent memory sharp, bounded, and trustworthy.

---

## 1. Separate Episodic Logs from Mutable State

An agent needs two fundamentally different kinds of memory, and mixing them in one vector index guarantees drift:

```text
[Raw Interaction / Tool Output]
       │
       ├──► Episodic Store (Append-Only)
       │      └─ "What happened on Nov 12 at 14:00"
       │
       └──► Canonical State (Mutable / Versioned)
              └─ "User's current deployment target: GitHub Pages"
```

- **Episodic Logs** record *events* in time. They are immutable and useful for auditing or answering "what did we try last Tuesday?"
- **Canonical State** records *current truth* (active preferences, open tasks, settled constraints).

When a user switches from Platform A to Platform B, an append-only vector store retains both statements with near-identical embeddings. If your prompt pulls both chunks, the model is forced to guess which one is current. Keep canonical facts in a structured, updatable store and treat episodic logs as secondary evidence.

---

## 2. Score Retrieval by Recency, Authority, and Task Scope

Pure cosine similarity is blind to time and authority. A 400-token error stack from three weeks ago can easily outscore a one-line preference stated five minutes ago simply because it shares more keywords with the current query.

Replace raw similarity with a composite retrieval score:

1. **Temporal Decay:** Apply an exponential decay factor to episodic memories unless the query explicitly references a past date.
2. **Source Authority:** Weight explicit user instructions higher than inferred assistant summaries, and both higher than raw third-party tool outputs.
3. **Scope Filtering:** Tag memories by domain (`billing`, `infra`, `personal`) so a coding session does not surface unrelated travel notes just because both mention "booking" or "tokens."

---

## 3. Compact and Resolve Conflicts at Write Time

Most teams try to fix noisy retrieval at **read time** by adding rerankers or larger context windows. That treats the symptom while the store keeps growing noisier.

Instead, run **write-time compaction**:

- **Supersede, Don't Append:** When a new fact contradicts an existing state entry (e.g., deadline moved from Nov 12 to Nov 19), explicitly mark the old record `superseded` or overwrite it with a revision pointer.
- **Nightly Distillation:** Periodically compress raw daily transcripts into concise, attributed summaries. Strip transient tool payloads, retries, and intermediate stack traces.
- **Provenance Tags:** Attach *who* stated a fact (`user_explicit`, `assistant_inferred`, `external_email`) so downstream reasoning knows how much weight to give it.

---

## 4. Isolate Working Scratchpads from Long-Term Memory

During multi-step execution—like debugging a CI pipeline or comparing ten vendors—agents generate massive amounts of intermediate noise: failed search queries, partial DOM snapshots, and hypothesis dead ends.

Never persist raw working memory into long-term storage. Use an explicit **Scratchpad Boundary**:

- Keep intermediate reasoning and raw tool dumps scoped to the active task's ephemeral workspace.
- Promote **only** the final verified outcome, decision rationale, and open follow-ups to long-term memory once the task closes.

Long-running autonomy is not about remembering *everything*. It is about forgetting the right things on schedule so the facts that matter stay unmistakable.

---

## 한국어 완역: 에이전트 메모리는 왜 시간이 지날수록 망가지는가: 컨텍스트 부패와 검색 드리프트 제어하기

대부분의 AI 에이전트 프로토타입은 똑같은 메모리 구조로 시작합니다. 모든 사용자 대화와 툴 실행 결과를 벡터 데이터베이스에 임베딩해 넣고, 추론 시점마다 상위 $k$개의 청크를 검색해 시스템 프롬프트에 밀어 넣는 방식입니다.

처음 몇 번의 세션에서는 마법처럼 잘 동작합니다. 하지만 4주차쯤 되면 에이전트는 어제 내린 결정과 모순되는 말을 하고, 이미 폐기된 취향을 사실처럼 읊으며, 컨텍스트 윈도우의 절반을 오래된 로그로 낭비하기 시작합니다.

이 현상이 바로 **컨텍스트 부패(Context Rot)**와 **검색 드리프트(Retrieval Drift)**입니다. 의미론적 유사도(Semantic Similarity)만으로는 폐기된 초안과 확정된 결정을 구분할 수 없고, 옛날 디버깅 로그와 현재 유효한 규칙을 가려낼 수 없습니다.

장기 실행 자율 에이전트의 메모리를 선명하고, 통제 가능하며, 신뢰할 수 있게 유지하는 4가지 아키텍처 패턴을 정리합니다.

### 1. 일화 기억(Episodic Logs)과 가변 상태(Mutable State)를 분리하라

에이전트에게는 성격이 완전히 다른 두 종류의 메모리가 필요합니다. 이 둘을 하나의 벡터 인덱스에 섞어 두면 드리프트는 필연적으로 발생합니다.

- **일화 기억(Episodic Logs):** 시간순으로 일어난 *사건*을 기록합니다. 수정되지 않는(Append-only) 기록이며, "지난 화요일에 뭘 시도했지?" 같은 감사나 회고에 쓰입니다.
- **정본 상태(Canonical State):** *현재 시점의 진실*(활성화된 선호도, 열려 있는 태스크, 확정된 제약 조건)을 기록합니다.

사용자가 플랫폼 A에서 플랫폼 B로 갈아탔을 때, 추가만 가능한 벡터 스토어에는 두 문장이 거의 비슷한 임베딩 값으로 공존합니다. 프롬프트에 두 청크가 모두 딸려 들어오면 모델은 어느 쪽이 현재 상태인지 찍어야 합니다. 현재의 사실은 갱신 가능한 구조화 저장소에 두고, 일화 로그는 보조 근거로만 취급해야 합니다.

### 2. 유사도 대신 최신성·권위·작업 범위로 점수를 매겨라

순수 코사인 유사도는 시간과 정보의 권위를 보지 못합니다. 3주 전의 400토큰짜리 에러 로그가 방금 5분 전에 사용자가 말한 한 줄짜리 원칙보다 현재 쿼리와 키워드가 더 많이 겹친다는 이유로 더 높은 점수를 받기 쉽습니다.

단순 유사도 대신 복합 검색 점수를 사용해야 합니다.

1. **시간 감쇠(Temporal Decay):** 사용자가 명시적으로 과거 날짜를 묻지 않는 한, 일화 기억에는 시간에 따른 지수 감쇠 계수를 적용합니다.
2. **출처 권위(Source Authority):** 사용자의 명시적 지시를 에이전트의 추론 요약보다 높게, 그리고 둘 다 외부 툴의 원시 출력보다 높게 가중치를 부여합니다.
3. **범위 격리(Scope Filtering):** 메모리에 도메인 태그(`billing`, `infra`, `personal`)를 붙여, 코딩 세션 중에 단순히 "토큰"이나 "예약"이라는 단어가 겹쳤다는 이유로 여행 메모가 튀어나오지 않게 차단합니다.

### 3. 읽기 시점이 아니라 쓰기 시점에 압축하고 충돌을 해소하라

많은 팀이 노이즈가 낀 검색 결과를 해결하려고 리랭커(Reranker)를 붙이거나 컨텍스트 윈도우를 키우는 등 **읽기 시점(Read Time)**을 고치려 합니다. 하지만 이는 저장소가 계속 더러워지는 근본 원인을 둔 채 증상만 덮는 방식입니다.

대신 **쓰기 시점 압축(Write-time Compaction)**을 실행해야 합니다.

- **추가 대신 대체:** 새로운 사실이 기존 상태와 충돌할 때(예: 마감일이 11월 12일에서 19일로 변경됨) 이전 기록을 명시적으로 `superseded` 상태로 전환하거나 수정 포인터로 덮어씁니다.
- **야간 정제(Nightly Distillation):** 하루 동안 쌓인 원시 대화 로그를 주기적으로 간결하고 출처가 명확한 요약으로 압축합니다. 일시적인 툴 페이로드, 재시도 기록, 중간 스택 트레이스는 이때 걷어냅니다.
- **출처 태깅(Provenance Tags):** 사실의 출처(`user_explicit`, `assistant_inferred`, `external_email`)를 함께 저장해 후속 추론 시 신뢰도 비중을 조절할 수 있게 합니다.

### 4. 작업용 스크래치패드를 장기 기억과 격리하라

CI 파이프라인을 디버깅하거나 여러 벤더를 비교하는 다단계 작업 도중, 에이전트는 실패한 검색 쿼리, 불완전한 DOM 스냅샷, 막다른 가설 같은 엄청난 양의 중간 노이즈를 생성합니다.

이 작업 메모리를 그대로 장기 기억에 영구 저장해서는 안 됩니다. 명확한 **스크래치패드 경계(Scratchpad Boundary)**를 두어야 합니다.

- 중간 추론 과정과 원시 툴 덤프는 현재 태스크의 임시 워크스페이스 안에만 머물게 합니다.
- 태스크가 종료된 후 **검증된 최종 결과, 결정 근거, 남은 후속 과제**만 선별하여 장기 기억으로 승격합니다.

장기 실행 자율성의 핵심은 *모든 것을* 기억하는 것이 아닙니다. 중요한 사실이 흐려지지 않도록 **불필요한 것을 제때 잊어버리는 구조**를 설계하는 데 있습니다.
