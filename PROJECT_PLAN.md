# A-Town Spotifast — Project Plan

> **Canonical living plan for the A-Town-corp/spotifast fork**  
> **Document version:** 1.0.0 (planning baseline)  
> **Created:** 2026-10-14  
> **Project status:** Discovery / proof of concept; no lossless implementation verified  
> **Repository:** https://github.com/A-Town-corp/spotifast  
> **Primary upstream:** https://github.com/crmne/spotifast

## 1. Executive decision

Build a maintainable, native Rust adaptation of Spotifast that **optionally controls Spotify's official Soloist player** for genuine Spotify lossless streams, while preserving its existing librespot playback engine and normal Spotify features. The initial target is **Windows 11**, where Soloist would run inside **WSL2**; native Linux is the reference environment and macOS must continue to compile. Soloist's Windows audio path is **not yet validated**.

**P0 is a feasibility experiment, not a promise to ship lossless.** No major player rewrite, advertised lossless support, or automatic Soloist setup may proceed until the documented proof-of-concept gates pass. A failure at the Windows feasibility gate is a legitimate project result; retain the existing client and revisit an officially supported integration when available.

### What success looks like

A user can continue using the familiar Spotifast interface, opt into an external Soloist playback mode, play Spotify catalogue content with a Premium account, and control playback from Spotifast without losing library, queue, device, or account behavior. When Soloist is unavailable, the client remains usable with librespot. The product **never calls audio lossless without adequate evidence**, and clearly distinguishes a lossless source stream from bit-perfect Windows/DAC output.

### Non-goals for v1

- Decrypting, downloading, intercepting, capturing, reverse-engineering, or bypassing Spotify's protected audio/DRM.
- Redistributing or modifying Spotify Soloist binaries, API keys, credentials, or copyrighted audio.
- Promising bit-perfect playback via WSLg without measurement; upsampling or decoding a lossy stream as FLAC does not create lossless audio.
- Replacing Spotify catalogue streams with YouTube or other providers.
- Rewriting the existing librespot engine or the application's entire UI/state model.
- Building a local FLAC-file player, an audio ripping workflow, an EQ/MilkDrop loopback capture bridge, or new streaming services in this project phase.
- Silent global changes to Windows firewall, WSL distributions, networking, audio-device settings, or existing Spotify credentials.

## 2. Verified baseline and assumptions

### Repository facts (checked 2026-10-14)

The A-Town fork was created from `crmne/spotifast` and its inspected `main` source tree matched upstream at review time. The application is Rust/egui with `librespot` dependencies, a native audio path and CI for Linux, macOS, Windows, and Windows ARM. GitHub is the source of truth; **recheck HEAD before coding**.

| Concern | Existing source | Consequence |
|---|---|---|
| Librespot player and Connect session | `src/player.rs` | Do not remove it; engine currently maps 96/160/320 kbps to librespot bitrate variants. |
| Main command/event orchestration | `src/backend.rs` | Keep networking off the egui/UI thread; integrate generation-safe engine state. |
| Native audio sink, EQ and visualization input | `src/sink.rs`, `src/eq.rs`, `src/vis.rs` | External Soloist audio does not automatically pass through these hooks. |
| Local versus remote Spotify Connect target | `src/app.rs`, `src/ui/devices.rs` | External Soloist device must not be misrepresented as the built-in local device. |
| Preferences and migration | `src/settings.rs`, `src/ui/settings.rs` | Add backward-compatible configuration only after feasibility gate. |
| Auth and credential storage | `src/auth.rs`, `src/credentials.rs` | Keep Spotify OAuth and Soloist device authorization separate; never copy session secrets. |
| Upstream engineering policy | `AGENTS.md`, `CONTRIBUTING.md` | Currently forbids pursuing lossless/alternate backends; tailor intentionally for fork. |
| CI policy checks | `.github/scripts/check-new-settings.py`, `.github/scripts/check-dependency-sources.py` | New settings and non-`crmne` Git sources may fail until fork policy is adapted. |
| Product capability reference | `docs/_reference/what-spotify-allows.md` | Update only after functionality actually works; don't make speculative capability claims. |

### Official Spotify Soloist facts

As documented by Spotify on 2026-10-14:

- Soloist is an **official Linux headless Spotify Connect player** using PipeWire or PulseAudio, with Premium lossless support up to **24-bit / 44.1 kHz**. Premium is required for initial Soloist pairing; Premium is required for lossless playback.
- It requires a **user-specific Soloist API key**. The user must fetch its binary directly from Spotify. **The fork must not redistribute it**.
- It provides an **opt-in unauthenticated local WebSocket API**, activated with `--ws ADDR:PORT`. Its protocol supports state events and play/pause/seek/skip/volume/shuffle/repeat/queue/control commands.
- The API has **no TLS, authentication, Origin checks, authorization, or CSRF controls**. Binding to loopback is necessary, not sufficient to isolate it from all processes on the host.
- `command_result` acknowledges dispatch, **not** completed playback. Control state must be reconciled with subsequent playback events.
- It provides optional `ws.addr` / `ws.port` files for endpoint discovery and expires **90 days after its build date**; expired daemon exits with code `10`.
- The WebSocket API does **not** currently provide authoritative proof of the delivered codec or bit depth. A selected Soloist backend is not proof that a particular track reached the DAC losslessly.

Documentation: [Soloist overview](https://developer.spotify.com/documentation/soloist), [runtime](https://developer.spotify.com/documentation/soloist/concepts/overview), [command line](https://developer.spotify.com/documentation/soloist/reference/command-line), [WebSocket protocol](https://developer.spotify.com/documentation/soloist/reference/websocket-api), [downloads](https://developer.spotify.com/documentation/soloist/reference/downloads-and-updates).

### Unverified hypotheses

| Hypothesis | How to settle it | Status |
|---|---|---|
| Soloist can complete Premium pairing and lossless playback in a normal Linux test environment | Pair via official Spotify app; enable lossless; inspect supported output/diagnostics | **Open** |
| Soloist runs correctly under WSL2 on Windows 11 | Fresh WSL2 manual experiment; verify process, auth, audio, logs | **Open** |
| WSL2 discovers/advertises Soloist over Spotify Connect consistently | Test NAT then mirrored network as needed; test phone and desktop discovery | **Open** |
| WSLg PipeWire/PulseAudio bridge delivers useful audio to selected Windows device | Test volume, device switching, sleep, reconnect, clipping/resample constraints | **Open** |
| End-to-end delivery is bit-perfect | Document output format and measure with permissible tools; don't infer from API | **Open — not a v1 promise** |
| Soloist `play URI` reproduces Spotifast's context+offset semantics | Integration test track, album, playlist, first-track and clicked-middle-of-playlist | **Open** |

## 3. Architecture direction

```text
A-Town Spotifast (native Rust/egui)
    ├── Web API + existing librespot session (library/search/metadata/lyrics)
    ├── Existing Spotifast app state and optimistic actions
    └── Playback routing / capability checks (incremental, not big-bang)
         ├── Built-in librespot engine ── native sink ── Windows/Linux/macOS audio
         └── Optional Soloist controller ── local WebSocket
                                          └── official Soloist daemon (Linux)
                                              └── PipeWire/PulseAudio
                                                  └── WSLg bridge on Windows [unproven]
```

**Separation of concerns:** The existing Spotify API/session stays responsible for catalogue and library behavior. The chosen *playback* provider owns transport/control state. UI actions go through a single routing point; backend events are normalized into a small, versioned provider state and applied only while that provider owns playback. Remote Spotify Connect playback retains its existing handling; do not conflate a local Soloist process with `Target::Local` without deliberate modeling.

### Proposed incremental module layout (subject to code review)

```text
src/playback/
  mod.rs                # Provider selection and capabilities, minimal routing
  capabilities.rs       # Play/seek/queue/position/volume/source metadata availability
  soloist/
    mod.rs              # Adapter and lifetime boundaries
    protocol.rs         # Strict JSON request/event types (serde)
    client.rs           # Async WebSocket connection, reconnection and timeouts
    state.rs            # Event reconciliation, generation/sequence fencing
    discover.rs         # Safe, validated local endpoint discovery
    platform.rs         # Linux / Windows (WSL) behavior behind cfg gates
```

This is a **candidate layout**, not an instruction to create empty scaffolding or replace working code. Keep the smallest seams necessary. Existing `src/player.rs` continues to implement librespot. Prefer explicit typed commands/capabilities over an oversized universal trait.

### Contract and state invariants

1. Exactly one selected *local* provider at a time. Switching providers must stop/hand off the old provider safely; never create two competing active players silently.
2. Every provider change increments a **generation ID**. Late events, timers, and callbacks from old generations may not overwrite the new provider's state.
3. UI is optimistic as upstream requires. An acknowledgment of dispatch must not incorrectly establish successful playback. Reconcile pending actions with authoritative events, expiration, and actionable errors.
4. Source selection must not leak across providers: never hand a librespot authorization token to Soloist, or vice versa.
5. Device identity, session data, and queue ownership are not implicitly interchangeable. Document any unavoidable differences when transferring between providers.
6. Non-supported operations must be disabled/explained or deliberately routed through an authorized equivalent—not simulated as success.
7. Existing librespot mode should behave identically in settings, queue, reconnect, lyrics, and media keys when Soloist is disabled. *(The word 'librespot' everywhere in code is ASCII.)*
8. Avoid blocking the egui thread, storing plaintext tokens, or publishing sensitive information in debug logs.

### Provider capability design

Do **not** use a single `is_lossless` boolean. Model separately:

- **Backend kind:** built-in librespot / external Soloist.
- **Backend availability:** missing, needs setup, ready, connecting, active, degraded, failed, expired.
- **Supported actions:** play URI, pause, skip, seek, volume, repeat, shuffle, add-to-queue, query state, transfer/activate.
- **Quality provenance:** configured preference; engine reported selection (if available); verified output evidence; unknown.
- **Audio pipeline provenance:** Spotifast native PCM path versus external Soloist-owned output. EQ and visualizer integrations depend on this.

## 4. Milestones and work packages

Each milestone is a PR-sized set of tasks where practical. PRs must show the baseline SHA, reproducible evidence, CI results, and explicit deferred work. **Do not begin dependent milestones until the gate above passes.**

### M0 — Baseline, legality and risk inventory

**Deliverables:** Record current fork HEAD; establish working builds on Windows x64 and Linux CI; document baseline RAM/CPU/startup on target machine and existing Spotify playback behavior; read Soloist terms, official API, and fork policies. Add a tracking issue/decision record if needed.

**Gate M0:** Build/test commands and baseline versions are reproducible; scope and no-redistribution rules are agreed. No runtime change yet.

### M1 — Isolated Soloist feasibility (highest priority)

1. **Native Linux reference**: user obtains official current x86_64 Soloist build and key; run independently with `--ws 127.0.0.1:0`; pair using the official Spotify app; use `soloist ctl status/now/queue/trace`; verify pause, seek, next, playlist/album/track URIs, queue and volume. Do not include keys in recorded commands or screenshots.
2. **Source quality**: enable lossless on the official Spotify device where supported; collect engine-reported setting/output device characteristics and applicable diagnostic evidence. Label what is *not* proven; do not upload copyrighted audio or decrypted streams.
3. **Windows/WSL2**: test a Windows 11 PC with current WSL2/WSLg. Reproduce Soloist pairing, Spotify Connect visibility (including from phone), audio to Windows output, and WebSocket connectivity **from a native Windows process**. Measure sleep/wake, network reconnect, headset switch, VPN impact and audio glitches.
4. **Network modes**: log NAT/mirrored WSL2 modes; prefer a local bind `127.0.0.1`; document why changes are needed, scope exposure, and how to revert them. Never enable `0.0.0.0` as a shortcut.
5. **Expiry check**: deliberately document what happens with an expired build (or reproduce safely with a stub/simulated launcher), and how the user downloads a fresh official build.

**Evidence artifact:** `docs/experiments/soloist-feasibility.md` containing commands **with redacted keys**, machine/software versions, outcomes, audio-path constraints, networking configuration, and screenshots/log excerpts stripped of secrets.

**Go/no-go (all required for Windows v1):**

- Spotify's official Soloist plays on Linux, with lossless enabled where eligible.
- Under WSL2, Soloist authenticates, is controllable, and outputs audio through Windows reliably during the smoke-test matrix.
- A native Windows test client reaches the restricted local WebSocket endpoint safely.
- Discovery and API exposure have an acceptable, documented security model.
- Playback survives a reasonable set of start/stop/reconnect/device scenarios without corrupting sessions.

**If NO-GO:** stop Windows integration; either keep Soloist as *Linux-only experimental support* with explicit scope, or leave the feature deferred. Do not ship a mock lossless toggle or market it as Windows lossless. Record the blocker and the exact reproduction.

### M2 — Fork policy and CI adaptation

**Deliverables:** Update `AGENTS.md`, `CONTRIBUTING.md` and capability documentation to allow **official/authorized Soloist control** and narrowly scoped additional playback providers without relaxing DRM, privacy or native-performance principles. Preserve upstream behavior for contributions intended upstream. Scope new settings approval to A-Town maintainers. Permit only approved/pinned dependencies from approved sources, with supply-chain checks; do **not** simply remove `.github/scripts/check-dependency-sources.py`. Add a PR template section for Soloist/Windows compatibility evidence.

**Gate M2:** Existing policy tests pass on the fork and new policy explicitly rejects untrusted Git dependencies, secret logging, bundled Soloist binary, and external-listening APIs.

### M3 — Playback seam and normalized state

**Deliverables:** Extract minimal `ProviderKind`, capabilities and state routing; preserve existing librespot player and Web API session; add provider transition and generation fencing. Keep serialization backward compatible, preferably with `#[serde(default)]` on new optional settings. Define explicit local versus remote Connect behavior, local quality labels, and user-visible errors.

**Gate M3:** With Soloist feature disabled, existing playback and queue tests still pass on all supported targets; deterministic tests reject stale events, double playback and mismatched provider state.

### M4 — Soloist protocol adapter (Rust)

**Deliverables:** A small async JSON/WebSocket integration, typed protocol, reconnection with bounded exponential backoff, nonblocking event stream, strict local endpoint rules, finite timeouts, state/queue refresh, error classification, and telemetry-free diagnostics. Start with user-managed Soloist process; avoid WSL process orchestration until the adapter is stable.

**Required command coverage:** `get_auth_state`, `get_state`, `get_queue`, `play [uri]`, `pause`, `skip_next`, `skip_prev`, `seek`, `set_volume`, `set_shuffle`, `set_repeat_context`, `set_repeat_track`, `add_to_queue`, `activate`, `deactivate`. Handle `command_result`, `error`, `auth_state`, `playback_state`, position and queue events. New/unknown event fields must not crash the UI.

**Crucial semantics:** `play(uri)` starts a playable Spotify URI but is **not** a documented direct replacement for librespot's context + exact offset load request. Support the proven subset, implement tested equivalent commands where authorized, or expose the limitation clearly. Queue sizes may be limited by Soloist. Do not silently promise Spotify queue parity.

**Gate M4:** A mock WebSocket server drives fully deterministic protocol tests for login, race conditions, stale state, queue, out-of-order messages, disconnect, server errors, and unsupported commands. Linux live integration confirms basic commands.

### M5 — Windows WSL2 integration (conditional on M1)

**Deliverables:** First: written user-managed installation workflow. Second, if warranted: opt-in native Rust detection/launcher with an explicit WSL distribution selection, path/version validation, safe process lifecycle and no destructive distro changes. Ensure Win32-to-WSL localhost forwarding is verified for the actual networking mode; WSL security depends on both Linux and Windows listener exposure.

**Operational behavior:** distinguish WSL absent, distro stopped, binary absent/expired, no key, unpaired, audio unavailable, and WebSocket unreachable. Offer clear recovery instead of retry loops. Provide a *manual*, reversible procedure for any `.wslconfig` networking changes.

**Gate M5:** Windows x64 end-to-end pass; no unexpected listener outside intended host; clean startup/shutdown; reconnect/upgrade; missing WSL leaves standard Spotifast entirely functional. Continue Windows ARM build checks without claiming availability of an x64 Soloist Linux binary under ARM without testing.

### M6 — UI integration and truthful playback indicators

**Deliverables:** Accessible engine selection with a simple setup/status panel; show current playback device, engine readiness, key/expiry setup state, and controls limited to supported capabilities. Reuse existing themes and translation patterns. Native media keys and optimistic controls must remain coherent.

**Quality language:**

- `Very high · 320 kbps` for the known built-in librespot quality.
- `Spotify Soloist · Lossless eligible` only if configured/eligible; **not** proof of a specific stream.
- `Source/output quality unverified` whenever the engine lacks explicit stream/output confirmation.
- Never display a definitive `24-bit/44.1 kHz` badge just because Soloist is selected.

**Audio features:** Preserve EQ and visualizer behavior for librespot. For Soloist, explicitly show that Spotifast's audio-tap EQ/MilkDrop features are unavailable unless a separately reviewed, legally appropriate, supported audio-tap API exists. No silently nonfunctional controls.

**Gate M6:** End-user can switch in both directions, recover from missing/expired Soloist, understand lossless uncertainty, and use every supported control without misleading states. UI evidence captured in PR, with accessibility review.

### M7 — Hardening, packaging and candidate release

**Deliverables:** Automated unit/integration tests, Rust formatting/linting/documentation checks, cross-platform compile/test matrix, reproducible Windows smoke-test checklist, security review, updated user guide, issue template, and release notes. Check GitHub Actions dependency pinning; do not weaken existing quality gates to ship sooner.

**Distribution:** Ship only A-Town Spotifast binaries and source. Provide links/instructions for each user to download a *current* Soloist archive and obtain a personal API key. Detect imminent 90-day expiry locally if possible; never auto-download/host the Soloist executable unless Spotify explicitly authorizes it.

**Gate M7 / release candidate:** No regressions in librespot mode, successful conditional Windows live tests, verified isolation and secret handling, documented limitations, and a reviewed rollback path. Only then consider any public 'lossless-capable via Soloist' messaging.

## 5. Test strategy and acceptance matrix

### Automation required from first implementation PR

- Rust: `cargo fmt --all --check`, `cargo clippy --locked --all-targets -- -D warnings`, `cargo test --locked --all-targets`. Account for existing feature-specific CI, toolchain pins and GUI dependencies before selecting exact commands.
- Tests with fake protocol/server; *no Spotify secret* required in CI.
- Backward-compatibility test reading a representative pre-fork `settings.json` fixture (do not commit private user settings).
- Reconnect/invalidation, provider switch, double-event, malformed JSON, timeouts and missing-output tests.
- Linux + macOS + Windows x64 build/test; Windows ARM compile where the existing CI already supports it. Feature-gated WSL-specific logic must not break other targets.
- New dependencies pinned, reviewed, compatible with the project license, and scanned; no uncontrolled Git forks or moving branches.

### Manual test matrix (record PASS / FAIL / NOT RUN with evidence)

| Test | Linux | Win11 + WSL2 | Built-in librespot | Expected |
|---|:---:|:---:|:---:|---|
| Premium pairing and initial play | Required | Required | Regression | User can hear content |
| Track, album and playlist URI behavior | Required | Required | Regression | Document supported semantics |
| Seek/skip/volume/shuffle/repeat | Required | Required | Regression | UI and backend agree |
| Queue add/query, context switch | Required | Required | Regression | Never falsify queue |
| Another Spotify Connect controller takes over | Required | Required | Regression | Predictable state/ownership |
| WebSocket disconnect/reconnect | Required | Required | N/A | No stuck spinner or duplicate command |
| WSL restart / Windows sleep-wake | N/A | Required | Regression | Recovery without session loss |
| Bluetooth/USB/headphone audio device switch | Recommended | Required | Regression | Audio recovered or error is clear |
| No Soloist installation | Required | Required | Required | Librespot still works |
| Expired Soloist binary | Mock/live | Mock/live | N/A | Actionable update path, no key leak |
| API accessible from LAN | Required | Required | N/A | **MUST NOT be exposed** |
| Quality provenance/output-path diagnostics | Required | Required | Regression | Never overstate quality |
| UI media keys/tray and visualizers | Required | Required | Regression | Clear backend-specific capability |

### Performance budgets (baseline first)

Record startup, steady idle CPU, steady idle memory, wake-to-ready time and switch-to-play latency for both modes on the same device. Set **numerical regression budgets after M0 measurements**; don't invent targets that ignore WSL startup cost. The default/no-Soloist mode must not regress materially in startup, memory, or idle CPU. WebSocket handling and probing must be asynchronous.

## 6. Security and privacy requirements

**Assets:** user's Spotify account/session, Soloist API key, protected Spotifast credentials, Soloist data directory, local socket control of playback, and device information.

**Threats and mitigations:**

| Risk | Required handling |
|---|---|
| Unauthenticated Soloist WebSocket | Bind only to `127.0.0.1` in Soloist Linux environment; verify effective exposure from Windows and LAN; never add a public port proxy or `0.0.0.0`. |
| Same-host/process compromise | Treat local access as a meaningful residual risk; loopback does **not** authenticate local processes, and Windows/WSL forwarding may broaden access. Document residual risk honestly. |
| API key or auth token leakage | Do not hardcode, commit, log, send to GitHub Actions, show in screenshots, write to world-readable files, or place in ordinary Spotifast settings. Use OS credential store/user-protected startup workflow where possible. |
| API key visible as a process argument | Soloist officially takes `--api-key`; evaluate exposure via Linux `ps`/process inspection, terminal history and diagnostic capture. Require documented mitigation and user consent before auto-launch. |
| Spoofed endpoint | Restrict accepted host to loopback, validate port and endpoint discovery files, ownership/permissions where feasible, and refuse unexpected redirects/proxies. |
| Unbounded input/event storms | Size caps, strict JSON parsing, finite timeouts, bounded queues/backoff, discard stale events, privacy-safe logs. |
| Untrusted updates and binary substitution | Soloist downloaded by user from Spotify only; verify version/expiry; apply OS file permission discipline. Spotifast build artifacts should have traceable CI/source revision. |
| Missing/expired dependencies | Fail closed for Soloist, fall back *only after explicit/visible engine selection*; no silent change of quality or source. |
| DRM/license violations | No bypass, redistribution, ripping, protocol impersonation beyond authorized interfaces, or protected-stream capture. |

The local WebSocket server doesn't become secure solely because its endpoint uses an ephemeral port. Security review must account for Windows local applications, WSL integration, browser-origin risks, other WSL processes, firewalls, and the actual listener bind.

## 7. Decisions, scope changes and open questions

### Accepted architectural decisions (initial)

| ID | Decision | Rationale |
|---|---|---|
| ADR-001 | Preserve librespot as the default backend and compatibility baseline. | Minimize regression and retain native performance. |
| ADR-002 | Soloist is a separate, official, optional **playback** backend, not a replacement metadata session. | Existing librespot APIs serve library and other features. |
| ADR-003 | First prove the Linux reference, then Windows WSL2 before full integration. | Avoid a speculative Windows architecture. |
| ADR-004 | Rust-first; no embedded web engine, hosted backend, or DRM bypass. | Native performance, security and project scope. |
| ADR-005 | Honest quality/feature reporting. | API capabilities do not guarantee DAC output. |
| ADR-006 | Keep Soloist binary/key user-supplied and distinct from fork releases. | Official distribution/access rules. |
| ADR-007 | Develop in focused branches and PRs; don't mutate `main` as a shortcut. | Durable reviewable history. |

### Open questions (resolve before claiming support)

1. Does current official Soloist work under **WSL2** with Spotify Connect discovery and reliable audio on the actual target machine?
2. Does Windows-to-WSL loopback forwarding expose more attack surface in NAT or mirrored mode than acceptable? What are the safe defaults?
3. Which Spotify contexts, offset semantics and queue operations can Soloist actually reproduce? What must be feature-gated?
4. Can audio source quality and output format be verified with supported APIs/diagnostics without inspecting protected audio?
5. Can device selection and output routing be supported without changing global Windows settings?
6. Which features that read native PCM (EQ, visualizer, MilkDrop) must be disabled or reinterpreted in Soloist mode?
7. Should v1 support **user-managed external Soloist only**, postponing automatic WSL launch until v2? (Default recommendation: yes.)
8. How will local Soloist device ownership interact with Spotifast's current local-versus-remote Spotify Connect state machine?
9. What minimum permissions and safe lifecycle model are viable for a Soloist API key that must be supplied on its CLI?

### Scope-change rule

Any new backend, local FLAC feature, automatic installer, audio capture, quality assertion or settings migration must be proposed as a separate ADR with: motivation, security impact, source of truth, test gate, rollback plan, and maintainer approval. No feature is accepted merely because it seems to appear in Spotify/private protocol metadata.

## 8. Work tracking protocol

This document is the canonical plan. Add links to GitHub issues/PRs below as work begins; status is **evidence-based**.

| Stage | Status | Tracking issue/PR | Evidence |
|---|---|---|---|
| M0 Baseline | `NOT STARTED` | — | — |
| M1 Soloist feasibility | `NOT STARTED` | — | — |
| M2 Fork policy/CI | `NOT STARTED` | — | — |
| M3 Playback seam | `BLOCKED: M1` | — | — |
| M4 Soloist adapter | `BLOCKED: M1` | — | — |
| M5 WSL2 integration | `BLOCKED: M1` | — | — |
| M6 UI and quality | `BLOCKED: M1` | — | — |
| M7 Release hardening | `BLOCKED: M1` | — | — |

**Allowed statuses:** `NOT STARTED`, `IN PROGRESS`, `BLOCKED: <reason>`, `PASSED`, `FAILED`, `DEFERRED`. Do not mark `PASSED` without a linked evidence artifact, CI run or test report. Record the exact SHA and dates for significant decisions. Maintain this document at the same time as scope changes; don't rewrite history or quietly erase failed experiments.

### Definition of done for every PR

- [ ] Linked milestone and precise behavioral change; no unrelated refactor.
- [ ] Tests for regressions and failure paths, with exact CI run links/results.
- [ ] No plaintext secrets, unreviewed external endpoints, bundled Soloist binaries, or automatic system changes.
- [ ] Public/user docs reflect actual behavior; unsupported features not advertised.
- [ ] Existing librespot and cross-platform build/test compatibility considered.
- [ ] Backwards-compatible state/settings migration or explicit rollback evidence.
- [ ] Reviewer's approval and one clearly reversible merge.

## 9. Immediate next actions

1. **M0:** Record current commit SHA, branch policy, known-good build/test results, and target Windows 11/WSL environment versions.
2. **M1:** Write and run a *manual* Soloist Linux/WSL proof-of-concept checklist. Do not commit keys or installer archives.
3. Record results in `docs/experiments/soloist-feasibility.md`; make a documented **GO**, **CONDITIONAL GO**, or **NO-GO** decision for Windows lossless playback.
4. Only on a justified GO: open M2/M3 implementation PRs, using this file as the scope and acceptance reference.

## 10. Reference material

**Upstream/code**

- https://github.com/A-Town-corp/spotifast
- https://github.com/crmne/spotifast
- https://github.com/librespot-org/librespot

**Official Spotify**

- https://developer.spotify.com/blog/2026-08-13-introducing-spotify-soloist
- https://developer.spotify.com/documentation/soloist
- https://developer.spotify.com/documentation/soloist/concepts/overview
- https://developer.spotify.com/documentation/soloist/reference/websocket-api
- https://developer.spotify.com/documentation/soloist/reference/command-line
- https://developer.spotify.com/documentation/soloist/reference/downloads-and-updates

**Microsoft WSL**

- https://learn.microsoft.com/en-us/windows/wsl/networking
- https://learn.microsoft.com/en-us/windows/dev-environment/wsl-interop

## 11. Document change log

| Version | Date | Change |
|---|---|---|
| 1.0.0 | 2026-10-14 | Initial implementation-ready, gate-driven plan based on inspected A-Town fork and official Soloist/WSL documentation. |