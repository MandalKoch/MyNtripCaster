# 🛰️ MyNtripCaster

[![.NET 10](https://img.shields.io/badge/.NET-10-512BD4?logo=dotnet&logoColor=white)](https://dotnet.microsoft.com/)
[![C#](https://img.shields.io/badge/C%23-12-239120?logo=csharp&logoColor=white)](https://learn.microsoft.com/dotnet/csharp/)
[![Aspire](https://img.shields.io/badge/Aspire-orchestration-0078D4?logo=dotnet&logoColor=white)](https://learn.microsoft.com/dotnet/aspire/)
[![Wolverine](https://img.shields.io/badge/Wolverine-messaging-7C3AED)](https://wolverinefx.net/)
[![NATS](https://img.shields.io/badge/NATS-transport-27AAE1?logo=natsdotio&logoColor=white)](https://nats.io/)
[![OpenTelemetry](https://img.shields.io/badge/OpenTelemetry-observability-425CC7?logo=opentelemetry&logoColor=white)](https://opentelemetry.io/)
[![Blazor](https://img.shields.io/badge/Blazor-Server-512BD4?logo=blazor&logoColor=white)](https://learn.microsoft.com/aspnet/core/blazor/)
[![Docker](https://img.shields.io/badge/Docker-chiseled-2496ED?logo=docker&logoColor=white)](https://hub.docker.com/r/ubuntu/dotnet-runtime)
[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-brightgreen.svg)](LICENSE)
[![Built with Claude Code](https://img.shields.io/badge/built%20with-Claude%20Code-D97757?logo=anthropic&logoColor=white)](https://claude.com/claude-code)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-ff69b4.svg)](https://github.com/MandalKoch/MyNtripCaster/pulls)

A simple, scalable **NTRIP v2** caster. Relays RTCM correction streams from GNSS base stations to rovers over HTTP/1.1 chunked — built to run horizontally behind thin connection-holding gateways so clients stay online through caster restarts.

> 🚧 **Status:** early scaffolding. Architecture is defined in [`CLAUDE.md`](./CLAUDE.md); code is on the way.

---

## ✨ What it does

- 📡 Speaks **NTRIP v2** to bases (publishers) and rovers (subscribers).
- 🔀 Fans out RTCM between caster instances over **NATS** — scale out by adding nodes, not by vertical scaling.
- 🔁 **Session resumption** via GUIDv7: if a caster restarts, clients stay connected through the gateway and resume seamlessly.
- 🖥️ **Operator UI** (Blazor Server) for live session monitoring and auth configuration.
- 🔭 **Observability by default** — OpenTelemetry traces, metrics, and logs wired from day one.

## 🏗️ Architecture at a glance

```mermaid
flowchart LR
    Base["🛰️ Base Station<br/>(publisher)"]
    Rover["📡 Rover<br/>(subscriber)"]
    GW1["🚪 Gateway A"]
    GW2["🚪 Gateway B"]
    Caster1["⚙️ NtripCaster 1"]
    Caster2["⚙️ NtripCaster 2"]
    NATS[("🔀 NATS<br/>rtcm.&lt;mountpoint&gt;")]
    Sessions["💾 Sessions"]
    Auth["🔐 Auth"]
    Admin["🖥️ AdminUi"]

    Base -- "raw TCP + GUIDv7" --> GW1
    GW1 -- "X-Session-Id header" --> Caster1
    Caster1 -- "publish" --> NATS
    NATS -- "subscribe" --> Caster2
    Caster2 -- "chunked RTCM" --> GW2
    GW2 --> Rover

    Caster1 -. "Wolverine/NATS" .-> Sessions
    Caster1 -. "Wolverine/NATS" .-> Auth
    Caster2 -. "Wolverine/NATS" .-> Sessions
    Caster2 -. "Wolverine/NATS" .-> Auth
    Admin -. "live updates" .-> Sessions
    Admin -. "CRUD" .-> Auth

    classDef ingress fill:#FDE68A,stroke:#B45309,color:#78350F;
    classDef caster  fill:#C7D2FE,stroke:#4338CA,color:#1E1B4B;
    classDef infra   fill:#A7F3D0,stroke:#047857,color:#064E3B;
    classDef service fill:#FBCFE8,stroke:#BE185D,color:#500724;
    classDef client  fill:#E5E7EB,stroke:#374151,color:#111827;

    class Base,Rover client;
    class GW1,GW2 ingress;
    class Caster1,Caster2 caster;
    class NATS infra;
    class Sessions,Auth,Admin service;
```

The **Gateway** is a dumb TCP connection holder. It mints a per-connection GUIDv7, forwards raw bytes, and transparently reconnects to a healthy caster if the current one dies. All NTRIP protocol logic and authorization live in the **NtripCaster**. The **Sessions** and **Auth** services are called async over NATS so the caster can resume warm sessions without re-handshaking.

Full details, module responsibilities, and conventions are in [`CLAUDE.md`](./CLAUDE.md).

## 🔄 Connection flow: cold vs. warm session

```mermaid
sequenceDiagram
    autonumber
    participant C as 📡 Client
    participant G as 🚪 Gateway
    participant N as ⚙️ NtripCaster
    participant S as 💾 Sessions
    participant A as 🔐 Auth
    participant B as 🔀 NATS (rtcm.*)

    rect rgb(254, 243, 199)
    note over C,G: ❄️ Cold session (first connect)
    C->>G: TCP connect
    G->>G: Mint GUIDv7
    G->>N: Forward raw bytes + X-Session-Id
    N->>S: Lookup(guid)
    S-->>N: miss
    N->>N: Parse NTRIP v2 handshake
    N->>A: Verify(creds, mountpoint)
    A-->>N: ✅ authorized
    N->>S: Store(guid, role, mountpoint)
    N->>B: publish / subscribe rtcm.<mountpoint>
    B-->>C: RTCM stream (chunked)
    end

    rect rgb(209, 250, 229)
    note over C,G: ♻️ Warm resume (caster restart)
    G->>N: Reconnect + same X-Session-Id
    N->>S: Lookup(guid)
    S-->>N: hit (role, mountpoint)
    N->>B: re-attach rtcm.<mountpoint>
    B-->>C: stream resumes — client notices nothing
    end
```

## 🧩 Modules

| # | Module | Role |
|---|---|---|
| 1 | 💾 **Sessions** | Per-GUIDv7 cache: role, mountpoint, auth result, heartbeats |
| 2 | 🔐 **Auth** | Credential verification and mountpoint authorization |
| 3 | ⚙️ **NtripCaster** | The **only** component that parses NTRIP v2; bridges clients ↔ NATS |
| 4 | 🚪 **Gateway** | Thin TCP holder — mints GUIDv7, forwards bytes, transparent reconnect |
| 5 | 🖥️ **AdminUi** | Blazor Server — sessions dashboard + auth config |
| T1 | 🧪 **TestBaseStation** | Simulated base — publishes canned RTCM |
| T2 | 🧪 **TestRover** | Simulated rover — asserts stream correctness |

## 🙌 Open-source shoutouts

Built on the shoulders of these excellent open-source projects — thank you to the maintainers:

### 📚 Libraries

| Project | Purpose | Link |
|---|---|---|
| 🐺 **Wolverine** (JasperFx) | Message bus and mediator (`WolverineFx.NATS` transport) | <https://wolverinefx.net/> |
| 🔀 **NATS** | Message transport (Core NATS + JetStream) | <https://nats.io/> |
| 🔭 **OpenTelemetry** | Traces, metrics, and logs | <https://opentelemetry.io/> |
| 🧪 **xUnit** | Unit testing | <https://xunit.net/> |

### 🤖 Claude Code plugins & skills

Development on this repo is powered by a stack of community Claude Code plugins and skills:

| Plugin / Skill | What it does |
|---|---|
| 🦸 [**superpowers**](https://github.com/obra/superpowers) | Jesse Vincent's toolkit of Claude Code skills and workflows |
| 📖 [**context7**](https://github.com/upstash/context7) | Upstash's MCP server for fetching up-to-date library docs |
| 🎨 [**frontend-design**](https://github.com/anthropics/claude-code) | Design-quality frontend generation |
| 🔍 [**code-review**](https://github.com/anthropics/claude-code) | PR review workflow |
| ⚡ [**claude-code-setup**](https://github.com/anthropics/claude-code) | Automation recommender for Claude Code projects |
| 🛠️ [**csharp-lsp**](https://github.com/anthropics/claude-code) | C# language server integration |
| 🎭 **design-taste-frontend**, **emil-design-eng**, **gpt-taste**, **high-end-visual-design**, **industrial-brutalist-ui**, **minimalist-ui**, **redesign-existing-projects**, **stitch-design-taste**, **full-output-enforcement** | Community UI/UX and output-quality skills that raise the bar on generated code |

## 🤝 Built with Claude Code

This project is being developed with [**Claude Code**](https://claude.com/claude-code), Anthropic's CLI for collaborative software engineering. [`CLAUDE.md`](./CLAUDE.md) serves as both the project charter for humans and the working brief for Claude.

## 📜 License

Licensed under the [Apache License 2.0](LICENSE) — you are free to use, modify, and distribute this software, including in commercial products. Attribution is appreciated but not enforced beyond the license terms.
