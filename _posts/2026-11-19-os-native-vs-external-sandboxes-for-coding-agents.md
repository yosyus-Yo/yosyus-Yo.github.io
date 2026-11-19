---
layout: post
title: "OS-Native vs. External Sandboxes: Isolating Coding Agents and MCP Servers"
subtitle: "From OpenAI's Inside the Codex Sandbox (Nov 18) and Prime Radiant's external sandbox prototypes (Nov 17) to pinning stdio MCP servers against supply-chain drift."
date: 2026-11-19 09:00:00 +0900
categories: [Architecture, Security]
tags: [sandboxing, coding-agents, mcp, codex, ai-engineering]
---

Once an AI agent can edit files, run shell commands, and spawn Model Context Protocol (MCP) servers, prompt-level guardrails stop being a security boundary. What actually contains a bad tool call—or a prompt injection hidden inside a dependency—is the **execution sandbox**.

In mid-November 2026, two architectural blueprints landed back-to-back: OpenAI's deep dive into **the Codex sandbox** (November 18) and Prime Radiant's **external sandbox prototypes** documented by Simon Willison (November 17), alongside **Vercel's eve 1.0** general availability (November 17). Together, they map out the real trade-offs engineers face when isolating coding agents and their toolchains in production.

---

## 1. Host-Native Sandboxing: Why Codex Skips Docker on Developer Laptops

Most agent frameworks default to containers or microVMs. In *Inside the Codex sandbox*, OpenAI explains why Codex takes the opposite route on local machines: **OS-native sandboxing** (`spawn_command_under_sandbox()` across macOS, Windows, and Linux).

The reason is practical developer ergonomics. A developer's laptop holds bespoke toolchains—native Mac executables, Xcode, iOS simulators, local SDKs—that break or become painfully slow when stuffed into a Linux container. Instead of virtualizing the environment, Codex constrains the process tree directly on the host using platform primitives:

- **Workspace-scoped writes:** The built-in `:workspace` profile allows writes inside the repository while locking down `.git` (including `.git/hooks`) and `.codex` (`config.toml`) as read-only.
- **Broad reads, zero network by default:** Compilers and build tools need to read system SDKs outside the repo, so reads follow OS permissions while outbound network access is disabled to block silent data exfiltration.
- **Inherited process trees:** Every child process spawned by `exec_command` inherits the parent's `PermissionProfile` and optional `NetworkProxy`—even descendants that outlive the initial command.

Host-native sandboxing preserves local speed and native builds, but OpenAI explicitly flags its residual risk: an agent could still poison a gitignored local dependency (like `node_modules/prettier`) that the user's editor later executes *outside* the sandbox on save.

---

## 2. External Sandboxes: Decoupling the Agent Loop from the Guest OS

Where Codex embeds OS-level isolation inside the harness, **Prime Radiant** (November 17) explored the opposite pattern: **external sandboxes** that expose a standardized `localhost` HTTP API, allowing any coding agent to offload execution to an isolated guest OS:

| Prototype | Isolation Mechanism | How It Runs |
| --- | --- | --- |
| `external-sandbox-v86` | WebAssembly (`v86`) | Bundles Alpine Linux inside an npm package |
| `external-sandbox-container` | Apple Containers | Python wrapper running Linux containers natively on macOS |
| `external-sandbox-quicksand` | QEMU + Microsoft Quicksand | Full VM + Alpine image distributed via Python wheels |

Comparing how existing harnesses connect to this API reveals an important architectural split:
- **Claude Code** routes only its `Bash` tool into the external sandbox, leaving native `Read` and `Edit` on the host.
- **Codex** routes process and filesystem operations, including `apply_patch`.
- **Pi** routes `bash`, `read`, `write`, `edit`, `grep`, `find`, and `ls` into the guest.

Meanwhile, for hosted cloud agents, **Vercel's eve 1.0** (GA on November 17, cutting median turn latency from 3.036s to 414ms via Workflow SDK 5.5) pairs durable workflow execution with **Vercel Sandbox** to isolate agent-generated code in ephemeral cloud environments.

---

## 3. The Unsandboxed Blind Spot: `npx -y @latest` MCP Servers

A subtle failure mode highlighted across both OpenAI's Codex post and developer discussions on X this week is **MCP server privilege creep**.

By default, stdio MCP servers configured in a harness run *unsandboxed* with the user's full host permissions, because the harness cannot guess which files or APIs an arbitrary server needs. Worse, standard MCP READMEs routinely tell developers to paste `npx -y @scope/server@latest`. As engineers on X noted, running `@latest` auto-updates third-party code inside your agent's execution context—completely bypassing the agent's own tool-approval prompts.

The fix is straightforward and should be standard practice:
1. **Pin exact package versions** (`@1.0.0`, never `@latest`) and bump deliberately.
2. **Pre-populate caches and run `--offline`** inside a dedicated sandbox profile.

For example, wrapping a memory MCP server with Codex's CLI restricts writes strictly to `~/.npm` and `~/mcp-data` while cutting network access to zero:

```toml
[permissions.memory-mcp.filesystem]
":root" = "read"
"~/.npm" = "write"
"~/mcp-data" = "write"

[permissions.memory-mcp.network]
enabled = false

[mcp_servers.memory]
command = "codex"
args = ["sandbox", "--permission-profile", "memory-mcp", "npx", "--offline", "@modelcontextprotocol/server-memory@1.0.0"]
```

---

## Key Takeaway

Sandboxing is not one binary choice between "no isolation" and "heavy VM":
- Use **OS-native sandboxes** when agents must run local native toolchains (iOS, Metal, host compilers) with minimal latency.
- Use **external containers or microVMs** when you need clean cross-agent portability, ephemeral cloud execution, or strict host-filesystem separation.
- And never leave **stdio MCP servers** running unpinned and unsandboxed on your host machine.

<!-- lang:ko -->

AI 에이전트가 파일을 수정하고, 셸 명령을 실행하며, Model Context Protocol(MCP) 서버를 직접 띄우기 시작하면 프롬프트 수준의 가드레일은 더 이상 보안 경계가 되지 못합니다. 잘못된 도구 호출이나 외부 의존성에 숨어든 프롬프트 인젝션을 실제로 막아내는 것은 **실행 샌드박스(Execution Sandbox)**입니다.

2026년 11월 중순, 이 주제를 정면으로 다룬 두 가지 아키텍처 청사진이 연달아 공개되었습니다. OpenAI가 11월 18일 공개한 **Codex 샌드박스 내부 구조(*Inside the Codex sandbox*)**와, 11월 17일 Simon Willison이 소개한 Prime Radiant의 **외부 샌드박스 프로토타입 3종**, 그리고 같은 날 정식 출시(GA)된 **Vercel eve 1.0**입니다. 이 사례들은 프로덕션 환경에서 코딩 에이전트와 툴체인을 격리할 때 엔지니어가 마주하는 현실적인 트레이드오프를 선명하게 보여줍니다.

---

## 1. OS 네이티브 샌드박싱: Codex가 로컬에서 Docker를 쓰지 않는 이유

대부분의 에이전트 프레임워크는 컨테이너나 마이크로VM을 기본값으로 택합니다. 하지만 OpenAI는 *Inside the Codex sandbox*에서 로컬 개발 환경에 정반대 접근법인 **OS 네이티브 샌드박싱**(`spawn_command_under_sandbox()`)을 선택한 이유를 설명합니다.

핵심은 개발 환경의 현실성입니다. 개발자의 노트북에는 macOS 네이티브 실행 파일, Xcode, iOS 시뮬레이터, 로컬 SDK 등 컨테이너 안에 그대로 복제하기 어렵거나 느려지는 도구가 가득합니다. Codex는 환경을 가상화하는 대신 호스트 OS 자체의 샌드박스 기능으로 프로세스 트리를 직접 제약합니다.

- **워크스페이스 한정 쓰기:** 기본 `:workspace` 프로필은 저장소 내부 쓰기만 허용하고 `.git`(특히 `.git/hooks`)과 `.codex/config.toml`은 읽기 전용으로 잠급니다.
- **넓은 읽기, 기본 네트워크 차단:** 컴파일러와 빌드 도구가 저장소 밖의 시스템 SDK를 읽어야 하므로 읽기는 OS 권한을 따르되, 외부 네트워크 접근은 기본 차단해 데이터 유출을 막습니다.
- **자식 프로세스 트리 상속:** `exec_command`가 생성하는 모든 자식 프로세스(원래 명령보다 오래 살아남는 백그라운드 프로세스 포함)가 부모의 `PermissionProfile`과 `NetworkProxy` 제약을 그대로 상속합니다.

물론 OpenAI는 OS 네이티브 방식의 한계도 분명히 짚습니다. 에이전트가 Git에서 무시되는 로컬 패키지(`node_modules/prettier` 등)에 악성 코드를 심어두면, 이후 사용자의 에디터가 저장 시점에 샌드박스 *밖에서* 해당 코드를 실행할 수 있기 때문입니다.

---

## 2. 외부 샌드박스: 에이전트 루프와 게스트 OS의 분리

Codex가 하네스 내부에 OS 격리를 녹여냈다면, 11월 17일 **Prime Radiant**가 공개한 3가지 프로토타입은 정반대로 표준화된 `localhost` HTTP API를 통해 실행 환경을 아예 외부 게스트 OS로 분리하는 **외부 샌드박스(External Sandbox)** 패턴을 실험합니다.

| 프로토타입 | 격리 방식 | 실행 구조 |
| --- | --- | --- |
| `external-sandbox-v86` | WebAssembly (`v86`) | npm 패키지 안에 Alpine Linux를 통째로 번들링 |
| `external-sandbox-container` | Apple Containers | macOS에서 Linux 컨테이너를 직접 띄우는 Python 래퍼 |
| `external-sandbox-quicksand` | QEMU + Microsoft Quicksand | Python wheel로 배포되는 경량 QEMU VM + Alpine 이미지 |

기존 코딩 에이전트들이 이 API에 연결되는 방식을 보면 하네스별 경계 설계 차이도 드러납니다.
- **Claude Code**는 `Bash` 도구만 외부 샌드박스로 보내고, `Read`·`Edit`는 호스트에서 실행합니다.
- **Codex**는 `apply_patch`를 포함한 프로세스·파일시스템 작업을 함께 보냅니다.
- **Pi**는 `bash`, `read`, `write`, `edit`, `grep`, `find`, `ls` 도구 전체를 게스트 OS 안에서 실행합니다.

한편 클라우드 호스팅 환경에서는 11월 17일 GA로 출시된 **Vercel의 eve 1.0**(Workflow SDK 5.5 기반으로 턴당 중간 지연시간을 3.036초에서 414ms로 단축)이 내구성 있는 워크플로우 실행과 함께 **Vercel Sandbox**를 결합해 에이전트가 생성한 코드를 일회용 격리 환경에서 실행합니다.

---

## 3. 놓치기 쉬운 사각지대: `npx -y @latest` MCP 서버

OpenAI 글과 이번 주 X(트위터) 개발자 커뮤니티에서 공통으로 경고한 취약점은 **MCP 서버의 권한 비대화**입니다.

일반적으로 하네스에 설정된 stdio MCP 서버는 어떤 파일과 네트워크를 쓸지 하네스가 미리 추측할 수 없다는 이유로 **샌드박스 없이 호스트 사용자 권한 그대로** 실행됩니다. 여기에 많은 MCP README가 `npx -y @scope/server@latest`를 그대로 복사해 쓰도록 안내하는데, 이는 에이전트의 승인 프롬프트 바깥에서 서드파티 코드가 자동 업데이트되어 내 권한으로 실행되는 공급망 리스크로 이어집니다.

해결책은 명확합니다.
1. **패키지 버전을 정확히 고정**(`@latest` 금지, `@1.0.0`처럼 명시)하고 의도적으로만 올립니다.
2. **캐시를 미리 받아 둔 뒤 `--offline` 옵션과 전용 샌드박스 프로필**로 감싸 실행합니다.

예를 들어 Codex에서는 메모리 MCP 서버의 쓰기 권한을 `~/.npm`과 `~/mcp-data`로만 제한하고 네트워크를 완전히 끌 수 있습니다.

```toml
[permissions.memory-mcp.filesystem]
":root" = "read"
"~/.npm" = "write"
"~/mcp-data" = "write"

[permissions.memory-mcp.network]
enabled = false

[mcp_servers.memory]
command = "codex"
args = ["sandbox", "--permission-profile", "memory-mcp", "npx", "--offline", "@modelcontextprotocol/server-memory@1.0.0"]
```

---

## 핵심 정리

에이전트 샌드박싱은 "무격리"와 "무거운 VM" 사이의 양자택일이 아닙니다.
- 로컬 네이티브 툴체인(iOS 빌드, Metal, 호스트 컴파일러)과 빠른 속도가 중요하다면 **OS 네이티브 샌드박스**를,
- 에이전트 간 이식성, 클라우드 일회용 실행, 엄격한 호스트 파일시스템 분리가 필요하다면 **외부 컨테이너·마이크로VM**을 선택해야 합니다.
- 그리고 어떤 경우에도 **stdio MCP 서버를 버전 고정 없이 무방비로 호스트에 띄워두지 않는 것**이 기본입니다.
