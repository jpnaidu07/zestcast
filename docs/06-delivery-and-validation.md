# Delivery roadmap and release acceptance

## 1. Build order and decision gates

Do not implement the full setup/UI before proving that the media path and input semantics meet the intended experience. The critical unknowns are native/browser interoperability, physical latency, GPU support, input compatibility and unattended Windows lifecycle.

| Phase | Deliverable | Exit evidence |
|---|---|---|
| 0: feasibility and scope | Reference Windows/GPU/browser lab; Electron capture experiment; native H.264 WebRTC experiment; relative mouse and audio tests; license review | Same workload latency and FPS comparison; actual hardware encoder evidence; browser/control compatibility report; go/no-go architecture ADR |
| 1: LAN vertical slice | Local owner setup, HTTPS, one display, one browser, WSS negotiation, exclusive input lease, Stop Sharing | Automated protocol tests and repeatable real Windows/browser session; no anonymous capture/input |
| 2: gaming worker | DXGI/WGC, one hardware encoder provider, WASAPI, isolated input helper, adaptation, metrics | Physical input-to-photon gate, frame pacing and 8-hour soak on reference hardware |
| 3: Internet readiness | Topology wizard, TURN assistant, forced-relay tests, reconnect, trusted TLS lifecycle | Public host/NAT/blocked-UDP matrix; tested certificate renewal and failure; relay capacity/cost report |
| 4: production hardening | Roles/MFA, app policy, signed installer/updater, crash recovery, diagnostics and backup | Clean-machine install/upgrade/rollback; independent security review; native fuzzing; recovery matrix |
| 5: compatibility beta | Mobile touch, Safari/Firefox coverage, AMD/Intel providers, headless/Server support as validated | Published per-platform feature matrix and user-visible fallback behavior |
| 6: advanced tiers | 120 FPS, AV1/HDR, virtual gamepads, optional reverse access gateway/viewers | Separate feature-specific performance/security gates; no inherited blanket certification |

Estimation assumption: a small team with Windows graphics/WebRTC, application/security and QA experience should budget several months, not a weekend. Phase 0 can be a 2-3 week timebox; estimate later phases from its results. These are planning assumptions, not a delivery commitment. A solo developer should reduce supported hardware and features substantially.

## 2. Initial implementation backlog

Each item should have an owner, acceptance criteria and evidence link when implemented.

1. **LAB-01:** Choose exact supported OS builds, reference GPU/driver, browser versions, display refresh, LAN and impairment profiles. Save reproducible workload configuration.
2. **SPIKE-01:** Electron capture-to-browser sender, source enumeration, actual codec and encode metrics. Measure latency without assuming Chromium flags worked.
3. **SPIKE-02:** Native DXGI -> one hardware encoder -> libwebrtc -> browser, including PLI/keyframe, rate changes, candidate gathering and audio.
4. **SPIKE-03:** Relative/absolute input, layout mapping, scan codes, stuck-key recovery, helper crash and browser blur tests.
5. **ADR-01:** Choose shipping media provider from measured gates; record native maintenance/build cost and fallback policy.
6. **AUTH-01:** Local enrollment, account roles, hashing, cookies/CSRF, abuse limits, revocation, MFA and recovery.
7. **PROTO-01:** Shared envelope/schema, REST contract, controller lease, epoch/revision handling and idempotent cleanup.
8. **HOST-01:** Interactive-session worker lifecycle, watchdog, GPU/display capability inventory and source policy.
9. **WEB-01:** Login, source/preset chooser, Connect progress, receiver, input modes and accessible touch controls.
10. **NET-01:** Topology model, certificates, firewall instructions, TURN credentials and actual direct/relay connectivity diagnostics.
11. **MEDIA-01:** Native capture, hardware encoding, audio, adaptive presets and honest stats overlay.
12. **POLICY-01:** Trusted desktop versus restricted-app mode, process identity and explicit limitations; VM isolation design if required.
13. **OPS-01:** Bounded logs, redacted diagnostics, readiness, crash-loop handling and tested backups.
14. **REL-01:** Clean build, signed packaging/helpers, dependency provenance, updater signature verification and rollback.

These are local backlog specifications, not GitHub issues already created.

## 3. Test strategy

Protocol/security tests run without a GPU where possible. Real media correctness and latency require real supported Windows devices. A headless CI browser alone cannot certify hardware encoding, display capture, game input, sound, mobile suspension or anti-cheat compatibility.

### Automated tests

- State machine transitions, cancellation from every state, duplicate stop/create and simultaneous controller claims.
- SDP/ICE ordering, generation/revision rejection, bounded candidate queues, reconnect snapshot and version mismatch.
- Auth/session expiry, role/source denial, CSRF/Origin, grant replay, rate limits and input authorization.
- Input sequence/cumulative motion reconciliation, layout transforms, release on disconnect and watchdog expiry.
- Config migration, backup/restore, malformed config rejection, update signature and anti-rollback behavior.
- Native parser fuzzing, IPC peer verification and bounded memory under hostile signaling/input.
- Browser UI tests for login, supported/unsupported states, Connect/playback rejection, error remediation and accessibility.

### Hardware and environment matrix

| Dimension | Required cases |
|---|---|
| Windows | Certified Windows 11 builds; standard/elevated target; lock/unlock, UAC, user switch, logoff, sleep/resume, reboot |
| GPUs | Reference NVIDIA first; AMD and Intel before claiming each; hybrid laptop, cross-adapter capture, encoder exhaustion, driver reset |
| Displays | Single/multiple; negative origins; mixed DPI; rotated; HDR source to SDR; hotplug; resolution/refresh changes; headless; RDP disconnected |
| Applications | Window/fullscreen/borderless; launcher-to-game; resize/minimize/close; protected surface; popup/file dialog; tested anti-cheat games |
| Audio | System/app tree; mute/silence; absent endpoint; endpoint change; device restart; 30-minute drift |
| Input | Relative mouse; absolute desktop; every modifier; AltGr/IME; wheel; drag; touch; gamepad attach/detach when supported; local/remote contention |
| Browsers | Exact Chrome/Edge baseline; Firefox/Safari; Android/iOS physical devices; low-memory/thermal/low-power conditions |
| Network | LAN; public IPv4; IPv6; router NAT; hairpin fail; double NAT; CGNAT with gateway/VPN; corporate proxy; UDP blocked; TURN UDP and TLS |
| Impairments | Controlled RTT 5/40/80/150 ms; loss 0/1/3/5%; jitter 0/10/30 ms; bandwidth collapse; reorder; MTU/fragmentation limits |
| Lifecycle | WSS loss while media alive; ICE path change; refresh/duplicate tab; host/worker/helper crash; stale authorization; TURN outage/rotation |
| Packaging | Clean VM; offline launch after install; port conflict; standard user; upgrade; rollback; uninstall preserving/removing data by explicit choice |

Not every impairment is a passing gaming tier. The release report must state when the experience degrades, which fallback occurs and when the product stops safely.

## 4. Performance methodology

Define the gaming workload: deterministic scrolling desktop, video motion and representative game scenes with documented resolution/FPS settings. Test at least 100 connection attempts per named network path for connection percentiles and record failure counts. For latency, collect at least 100 physical input-to-photon samples per reference scenario and disclose camera/measurement uncertainty; increase sample size for stable p95 claims.

Measure cold and warm starts separately. Test with UI metrics overlay off and on. Record capture, encode, receive, decode and presented frame cadence, dropped frames, end-to-end latency, CPU/GPU/VRAM/RAM, handles, network overhead and game FPS impact. Compare against the same host/client workload using a mature game-streaming baseline where feasible; do not infer equivalent performance from matching codec names.

Use network shaping in a controlled test environment. Publish the exact shaping configuration and account for asymmetric links. Measure direct and relay paths separately, including geographic relay placement. Browser stats help diagnose stages but cannot replace the physical end-to-end measurement.

## 5. Release gates

A production release requires:

- Every supported platform/path passes documented mandatory cases; unsupported features are detected before use.
- Proposed product SLOs from [architecture](01-product-and-architecture.md) pass on the reference configurations, or the product tier is revised honestly.
- No open critical/high issues allowing unauthorized desktop access, input, privileged IPC or update execution; independent review findings resolved or explicitly release-blocking.
- Revocation and local Stop Sharing reliably terminate capture/input; a severed WSS does not leave an indefinitely authorized WebRTC path.
- Eight-hour soak, repeated connect/disconnect and fault-injection tests show bounded resources and no stuck injected state in tested recoverable cases.
- Signed installer/update/rollback works on clean machines; no development servers, secrets or debug interfaces ship enabled.
- TLS and TURN credential rotation, configuration backup/restore and owner recovery have been rehearsed.
- Known limitations, game/browser support, ports, privacy behavior and operating runbooks are published with the release.

## 6. Repository and release hygiene

Keep frontend/backend together initially, with path ownership and shared schema changes reviewed together. Use a short-lived `codex/` feature branch convention for agent-created branches, protected `main`, required checks and review before release. The initial local planning snapshot can exist without implementation CI; add meaningful lint/build/tests when code is introduced rather than a green placeholder workflow.

Publish this project to public `jpnaidu07/zestcast` and verify the remote tree after each push. Do not accidentally publish the enclosing learning workspace or its unrelated project. Add issue templates and security reporting details after the owner chooses a real reporting contact. Enable available repository security features and review actions permissions; release signing secrets belong in protected environments.
