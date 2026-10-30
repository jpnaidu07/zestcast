# Security, isolation, and operations

## 1. Threat model

Protect the Windows desktop, credentials, user content, input authority, account database, app policies, signing/update keys and relay budget. Assume hostile Internet clients, malicious authenticated users, cross-site browser requests, malformed SDP/input, compromised renderer content and unexpected local processes. The product does not protect a host from an already fully compromised Windows administrator or kernel.

Trust boundaries: browser to HTTPS/WSS; control server to native worker; worker to input helper; user session to privileged broker; host to TURN; release infrastructure to installed application. For a reverse gateway deployment, the gateway also enters the control-plane trust boundary.

## 2. Identity and session controls

Owner setup occurs locally. Create individual viewer, controller and admin roles. Viewing does not imply control, and control does not imply changing host exposure, launching arbitrary applications or adding accounts. Admin actions that change users, network exposure, app policy, TURN or update settings require recent reauthentication and an audit record.

Hash passwords using a maintained Argon2id implementation with unique salts and benchmarked work factors. Begin at or above current OWASP minimums and tune on supported host hardware. Protect the hashing endpoint with concurrency and rate limits so expensive hashing cannot exhaust the host. Password reset invalidates sessions and uses a local owner recovery procedure; do not expose a password retrieval API. [OWASP password storage](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)

Use revocable opaque cookie sessions: Secure, HttpOnly, SameSite appropriate to the same-origin deployment, rotation after login/privilege changes, bounded idle and absolute lifetimes. Do not store administrator bearer tokens in localStorage. Suggested defaults: admin idle 15 minutes, account session absolute lifetime 12 hours; explicitly choose and test how active streaming renews user presence. Authentication expiry closes the stream or requires reauthentication before its authorization lease expires.

For public administration, support MFA before general Internet release. Recovery codes are hashed; recovery is auditable and documented. Offer local approval/pairing for owners who require it. Never ship a common default password or expose a bootstrap secret in a QR URL.

## 3. HTTPS, API and signaling defenses

- Validate exact allowed Host and Origin values, including expected scheme/port; protect against DNS rebinding and cross-site WebSocket hijacking.
- Require CSRF protection for cookie-authenticated mutations and the socket binding flow. Restrict CORS to actual intended origins; a same-origin application needs no wildcard CORS.
- Authenticate WSS upgrades and authorize each message/session action. Expire and revoke live sockets and worker grants, not only REST cookies.
- Bound connections, reservations, SDP size, candidate queues, message rates, login concurrency and diagnostic exports. Use a reverse proxy only with a documented trusted-proxy configuration; do not trust arbitrary forwarded IP headers.
- Reject duplicate JSON keys or ambiguous parser representations where the chosen schema/parser permits; canonicalize IDs and paths before policy checks. Fuzz all native-facing payloads.
- Restrict CSP to bundled assets, prevent framing, avoid third-party scripts, and avoid exposing credentials in query strings or referers. Static asset caching must not cache authenticated API responses.
- Return sanitized error codes, no stack traces, source thumbnails, candidate IPs or native handles to unauthorized callers.

OWASP documents WSS, explicit Origin validation, message authorization, limits and session handling as WebSocket defenses. [OWASP WebSocket security](https://cheatsheetseries.owasp.org/cheatsheets/WebSocket_Security_Cheat_Sheet.html)

WebRTC encrypts its transport, but it does not decide who may view this desktop. Secure signaling and policy bind the negotiated peer to the authorized session. TURN relays encrypted WebRTC packets; do not describe that as protection from a compromised endpoint or signaling authority.

## 4. Electron and native process hardening

Use `nodeIntegration: false`, `contextIsolation: true`, renderer sandboxing and normal web security. The Electron renderer loads packaged trusted assets, with navigation/popups denied by default. Expose only narrow preload operations; validate IPC sender origin/frame and payload. Never forward raw Electron or filesystem APIs to the browser. [Electron security guidance](https://www.electronjs.org/docs/latest/tutorial/security)

The control server runs without Administrator rights. Keep privileged broker commands narrow: enumerate authorized sessions, start the matching worker, apply reviewed install/repair operations. No arbitrary executable path, PowerShell, shell text, registry writer or filesystem operation accepted from network clients.

Use ACL-protected local named pipes with explicit local-user/session identities; prevent remote named-pipe access, pipe-name squatting and unauthorized reconnects. Pin protocol version and verify peer process identity where available. Workers receive a short session grant and read-only effective policy, not the master password hash or TURN shared secret.

Keep mutable configuration/data under appropriate Windows application-data directories with restricted ACLs, separate from signed program files. Use Windows-protected secret storage for private keys/TURN secrets and define recovery/backup rules. Avoid untrusted DLL search paths and writable executable locations.

Enable supported Electron fuses and ASAR integrity options only after testing subprocess/update compatibility. Sign native helpers as well as installer/application. Native modules must match the packaged Electron ABI. [Electron ASAR integrity](https://github.com/electron/electron/blob/main/docs/tutorial/asar-integrity.md), [electron-builder hardening](https://www.electron.build/docs/features/security/)

## 5. App permission is not desktop isolation

Provide two explicitly different operating modes:

**Trusted desktop mode:** The authorized controller controls the selected Windows desktop. An app launcher list is a convenience/policy list. Other visible apps, notifications, file dialogs and OS shortcuts may remain reachable. This mode is suitable for the owner's own access, not an untrusted customer sandbox.

**Restricted application mode:** Capture only the selected window/process tree, optionally only its audio, and enforce input/focus policy. Stop or blank on unexpected source changes, blocked foreground windows, invalid identity or process exit. Do not infer that this alone provides isolation; focus checks race, dialogs can expose files, and allowed apps may launch other programs.

For an actual untrusted-user boundary, use a dedicated Windows VM or appropriately restricted Windows user/session plus operating-system application control, file/network policy and a validated escape test suite. A separate VM per untrusted user is the preferred stronger boundary. Multi-tenant scheduling, VM lifecycle, credentials and licensing are a later product scope, not hidden inside v1's app allowlist.

### Application policy schema requirements

Use opaque application IDs mapped server-side to reviewed executable paths, allowed publisher/hash identity, approved arguments, working directory, allowed process descendants and capture/audio scope. Define how legitimate app updates refresh trust. Block arbitrary command-line text and shell expansion. Canonicalize paths and check ownership/ACLs to prevent an untrusted user replacing an allowed binary.

Revalidate process identity on every restart and transition from launcher to game. Do not trust process name alone or persist raw PID/window handle. Log policy decisions without recording the screen or key contents. Clipboard and file transfer are disabled initially; later implementations need separate directional grants, size limits and abuse controls.

## 6. Updates and supply chain

Choose a maintained Electron release, lock exact JavaScript/native dependencies and pin the libwebrtc revision/toolchain. Define an update cadence and emergency patch process. No "latest" container/dependency tags in released builds.

Use signed Windows packages and verify update signatures/publisher identity before execution. Authenticate the update manifest and prevent downgrade to a disallowed version. Stage updates, keep a verified known-good recovery package and back up configuration before migrations. Avoid forced mid-game restarts; expose pending critical updates and a maintenance window. Database migrations must be reversible or have a tested restore path. [electron-builder update signature verification](https://www.electron.build/docs/features/security/)

CI produces SBOMs, vulnerability/license scans, provenance, hashes and reproducible build instructions. Protect release signing keys outside ordinary build jobs; require protected release environments. Keep security updates for the control plane, Electron/Chromium, native dependencies, relay and any input/display drivers.

Review exact distribution obligations for Electron, libwebrtc, codecs, GPU SDKs, optional nut.js packages, drivers and any Sunshine-derived components. Codec availability does not resolve patent/distribution obligations. Choose this project's license before public publication. This document records a release dependency review task, not a legal determination.

## 7. Operations and observability

Use structured, bounded logs with request/session correlation represented by non-secret diagnostic identifiers. Record login outcomes, policy changes, controller acquire/release, route class, worker lifecycle, recovery attempts, update outcome and certificate health. Never log passwords, raw input, full SDP, TURN credentials, private keys or capture frames.

Default: local metrics and a rolling 7-day bounded log store, configurable by owner. No external telemetry or recordings without explicit configuration. Support bundles redact account/network identifiers and omit screen content by default; provide a preview before export.

Health dimensions: control readiness, interactive worker availability, display/capture, encoder, audio, input watchdog, TLS renewal and TURN test status. Expose basic liveness separately from authenticated readiness. Alert through the local console and any explicitly configured channel; do not assume an email/SMS integration exists.

Runbooks must cover certificate renewal failure, lost owner credentials, host lock/no session, black screen, blocked media, TURN outage/quota, GPU driver reset, update rollback, corrupted config/database and disk full. On disk-full or log failure, bound retries, preserve the local stop function and never keep authorizing sessions through an unknown policy state.

Back up configuration, policy and user database with authenticated encryption and restricted access. Do not automatically export machine-bound secret protection keys; document how a restored host re-enrolls TLS/TURN secrets. Test restoration on a clean machine and account for certificate identity changes.

Capacity: start with one session and reject excess requests predictably. Admit a session only with available capture/encoder/GPU/network resources. If viewers are later added, budget host egress and per-peer encode/transport cost; add an SFU only when the requirements justify it.

For rough planning, a 20 Mbps one-way video stream transfers about 9 GB per hour before audio/protocol overhead. A relay handles both incoming and outgoing legs; billing treatment varies, so model each provider's charged directions rather than blindly doubling an egress bill. Reserve upload headroom for bursts and game traffic.

## 8. Required security verification

Test anonymous source enumeration/capture/input, cross-user session access, forged source IDs, stale grants, cross-origin WSS, CSRF, DNS rebinding, oversized SDP/candidate floods, native IPC impersonation, PID reuse, local binary replacement, update tampering, privilege boundaries and crash-loop behavior. Verify revocation while media continues and WSS is severed. An independent security review is a general-availability gate for public remote control.
