# MyNtripCaster

A scalable NTRIP caster platform. Relays RTCM correction streams from base stations to rovers over NTRIP v2 (HTTP/1.1 chunked; NTRIP v1 is **not** a goal).

## Architecture

```
 Base ──► Gateway ──► NtripCaster ──► NATS ──► NtripCaster ──► Gateway ──► Rover
              │             │                         │             │
              │             ▼                         ▼             │
              │         Sessions  ◄────── NATS ──────►  Sessions    │
              │             ▲                         ▲             │
              │             │                         │             │
              │             ▼                         ▼             │
              │           Auth                       Auth           │
              │             ▲                         ▲             │
              │             │                         │             │
              │             └────── NATS ────┐   ┌────┘             │
              │                              ▼   ▼                  │
              │                           AdminUi (operators)       │
              │                                                     │
              └── GUIDv7 (per client connection) ───────────────────┘
```

**Flow:**
1. Client (base or rover) connects to a **Gateway** instance. Gateway mints a **GUIDv7** for the connection.
2. Gateway opens a downstream connection to a **NtripCaster** instance and forwards raw bytes, tagging every handshake with the GUIDv7.
3. NtripCaster asks **Sessions** (over NATS) what it knows about that GUIDv7:
   - **Cold session:** caster performs the NTRIP v2 handshake, calls **Auth** (over NATS), resolves the mountpoint, stores `{guid → {role, mountpoint, auth result}}` in Sessions.
   - **Warm session:** Sessions returns the cached decision — caster skips re-handshake/re-auth and resumes streaming immediately.
4. Caster attaches the connection to the correct NATS subject:
   - Base → **publish** on `rtcm.<mountpoint>`
   - Rover → **subscribe** to `rtcm.<mountpoint>`
5. Fan-out between caster instances happens via NATS, never via direct HTTP/TCP.

**Why the Gateway is thin:** when a caster instance restarts or is deployed, the Gateway keeps the client's TCP connection alive and transparently reconnects to a healthy caster, replaying the GUIDv7. The client never notices the caster restart.

All internal inter-service communication flows over **NATS** (JetStream where durability matters, core NATS for hot-path fan-out). Services scale horizontally and are independently deployable.

## Modules

| # | Module              | Responsibility                                                                 |
|---|---------------------|-------------------------------------------------------------------------------|
| 1 | **Sessions**        | Cache keyed by connection GUIDv7: role (pub/sub), mountpoint, auth result, heartbeats. Lets casters resume warm sessions without re-handshaking. |
| 2 | **Auth**            | Credential verification and mountpoint authorization. Called async by the caster on cold sessions only. |
| 3 | **NtripCaster**     | **Only** component that understands NTRIP v2. Performs the protocol handshake, decodes headers, and bridges client connections to `rtcm.<mountpoint>` NATS subjects. |
| 4 | **Gateway**         | Thin TCP connection holder. **No domain knowledge** — no NTRIP parsing, no auth, no mountpoint awareness. Mints a GUIDv7 per client connection, forwards raw bytes to a caster, and transparently reconnects to a healthy caster if the current one dies. |
| 5 | **AdminUi**         | Operator-facing web UI. Two primary views: **Sessions dashboard** (live list of active connections, mountpoint occupancy, throughput) and **Auth config** (users, credentials, mountpoint permissions). Pure client to Sessions + Auth over Wolverine/NATS — no domain logic of its own. |
| T1 | **TestBaseStation** | Simulated base station for integration tests — publishes canned RTCM          |
| T2 | **TestRover**       | Simulated rover — consumes a mountpoint and asserts stream correctness        |

### Async boundaries
- **Auth** and **Sessions** are called via **Wolverine over NATS**, never in-process. Use `IMessageBus.InvokeAsync<TResponse>(...)` (Core NATS, low-latency) for lookups; publish events (`PublishAsync`) for fire-and-forget. The caster must not block on either during connection setup beyond a short bounded wait.
- RTCM stream fan-out between caster nodes uses NATS subjects per mountpoint (`rtcm.<mountpoint>`). This stays on **Core NATS** (not JetStream) — RTCM is latency-critical and replayable from the base, not worth durable storage.
- JetStream is reserved for durable concerns: session heartbeats/audit, dead letters, scheduled retries.
- The **Gateway ↔ NtripCaster** hop is the only internal hop that may use a direct TCP/HTTP stream, because it is transporting raw client bytes. Everything else is Wolverine+NATS.

## Tech stack

- **.NET 10 / C#** across all services
- **.NET Aspire** for the app host, service discovery, and local orchestration
- **OpenTelemetry** wired from day one — traces, metrics, logs — via Aspire's OTel defaults
- **Wolverine** (`WolverineFx.NATS`) as the message bus / mediator. All NATS access goes through Wolverine — no direct `NATS.Net` client calls in service code.
- **NATS** as the underlying transport: Core NATS for low-latency hot paths (RTCM fan-out, request/reply), JetStream where durability matters (audit, session heartbeats, dead letters).
- **xUnit** for unit tests; T1/T2 drive integration tests

## Repository layout (target)

```
/src
  MyNtripCaster.AppHost/          # Aspire host
  MyNtripCaster.ServiceDefaults/  # Shared OTel + resilience defaults
  MyNtripCaster.Sessions/
  MyNtripCaster.Auth/
  MyNtripCaster.NtripCaster/
  MyNtripCaster.Gateway/
  MyNtripCaster.AdminUi/          # Blazor Server — sessions dashboard + auth config
  MyNtripCaster.Contracts/        # NATS message schemas shared across services
/tests
  MyNtripCaster.TestBaseStation/
  MyNtripCaster.TestRover/
  MyNtripCaster.Tests.Unit/
```

## Conventions

- Every service references `ServiceDefaults` — OTel, health checks, and resilience live there, not duplicated.
- Message contracts (records) and subject constants live in `MyNtripCaster.Contracts`. Never define a subject string inline.
- Message handlers follow Wolverine conventions: a class named `FooHandler` with a `Handle(FooMessage, ...)` method. No manual subscriber wiring — Wolverine discovers handlers.
- Use `IMessageBus.InvokeAsync<TResponse>(cmd)` for request/reply, `PublishAsync(evt)` for events. Do not inject `INatsConnection` into handlers or services.
- No synchronous HTTP between internal services. Wolverine/NATS only. (Public ingress through Gateway is raw TCP/HTTP, but carries raw client bytes — not a service call.)
- Log with structured logging (`ILogger` with named properties), never string interpolation into the message template.

## Build & run

**Artifacts are Docker images, one per deployable service.** Use the .NET SDK's built-in container publishing — no hand-written `Dockerfile` per project unless a service has a genuinely unusual need.

```bash
dotnet publish src/MyNtripCaster.NtripCaster -t:PublishContainer \
    -p:ContainerRepository=myntripcaster/caster \
    -p:ContainerImageTag=$(git rev-parse --short HEAD)
```

**Conventions:**
- Each deployable project sets container metadata in its `.csproj` (`ContainerRepository`, `ContainerFamily=noble-chiseled`, `ContainerUser=app`). Do not duplicate this in a Dockerfile.
- Base images:
  - HTTP services (AdminUi, Auth, Sessions, Gateway management port): `mcr.microsoft.com/dotnet/aspnet` chiseled.
  - Pure workers / raw TCP services (NtripCaster, Gateway data path): `mcr.microsoft.com/dotnet/runtime` chiseled.
  - Chiseled (Ubuntu noble-chiseled) is the default — no shell, no package manager, tiny attack surface.
- **Multi-arch** images: build for `linux/amd64` and `linux/arm64`. Set `ContainerRuntimeIdentifiers=linux-x64;linux-arm64` in the project files.
- **Tagging:** short git SHA for every build; `latest` only on main; semantic tags (`v1.2.3`) on release.
- **Local dev** runs everything through the Aspire AppHost (`dotnet run --project src/MyNtripCaster.AppHost`). Aspire handles service discovery and can launch dependencies (NATS, databases) as containers automatically — no `docker compose` file needed alongside Aspire.
- **CI** publishes images via `dotnet publish -t:PublishContainer -p:ContainerRegistry=<registry>`; the same invocation works locally and in CI.
- Test harnesses T1/T2 also publish as images so they can be used for container-based integration tests in CI.

## NTRIP v2 notes

- Server → caster: `POST` with chunked transfer encoding carrying RTCM.
- Client → caster: `GET /<mountpoint>` with `Ntrip-Version: Ntrip/2.0`, response is chunked RTCM.
- Source table at `GET /` (`STR` records, `ENDSOURCETABLE` terminator).
- Auth is HTTP Basic over TLS. The **Gateway may terminate TLS** (it's still public ingress), but it does **not** inspect or parse the NTRIP payload — it forwards raw bytes to the caster along with the connection's GUIDv7.

## Admin UI

- **Blazor Server** app. Live session updates land via Wolverine subscriptions → SignalR push to connected operators. Chosen over WASM so the UI can be a first-class Wolverine message handler (subscribe to session events, push live to the browser) without exposing NATS to clients.
- **Two top-level views:**
  - **Sessions dashboard** — live table of active GUIDv7 sessions: role (base/rover), mountpoint, caster node owning the session, connection age, throughput, last heartbeat. Operator can forcibly terminate a session (publishes a `TerminateSession(guid)` command; the owning caster drops the client).
  - **Auth config** — CRUD for users, credentials, and mountpoint permissions. All mutations go to the Auth service via `InvokeAsync<TResponse>`; Auth is the source of truth, AdminUi never writes to its own store.
- **No domain logic in the UI.** It queries and commands; it does not compute session state or auth decisions locally.
- **Operator auth** is separate from NTRIP client auth — the Admin UI uses ASP.NET Core Identity (or an external OIDC provider). Do not reuse the NTRIP credential store for operator login.

## Session resumption (the GUIDv7 contract)

- Gateway generates a **GUIDv7** (UUIDv7 — time-ordered, sortable) the moment a client connects.
- The GUIDv7 travels with the connection in a dedicated header (e.g. `X-Session-Id`) on every Gateway→Caster handshake, including reconnects.
- NtripCaster's first action on any incoming connection: `Sessions.Lookup(guid)` over NATS.
  - Miss → full NTRIP handshake + Auth, then `Sessions.Store(guid, {role, mountpoint, authResult})`.
  - Hit → skip handshake/auth, attach directly to the `rtcm.<mountpoint>` pub or sub.
- Sessions entries have a TTL and a heartbeat from the owning caster. If no caster owns a GUID for longer than the TTL, the session is evicted.
- This is the mechanism that lets caster instances be restarted/upgraded with zero client-visible disruption.
