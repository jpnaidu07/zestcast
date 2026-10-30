# Research sources and decision register

Research date: **30 October 2026**. Sources below are official project documentation, standards, source repositories or vendor documentation. This is a focused review of the important engineering decisions, not a claim to have searched the entire Internet. Documentation and working drafts can change; implementation must pin versions and recheck platform support at release time.

## 1. Evidence and its practical meaning

| Source | Evidence used in this plan | What still needs a prototype/test |
|---|---|---|
| [Electron desktopCapturer](https://www.electronjs.org/docs/latest/api/desktop-capturer) | Electron screen/window capture integration exists | Capture/encode latency, FPS, audio and packaged behavior |
| [Electron source structure](https://www.electronjs.org/docs/latest/api/structures/desktop-capturer-source) | Source and display identity fields | Durable mapping after hotplug/reboot; encoder GPU selection |
| [Electron security](https://www.electronjs.org/docs/latest/tutorial/security) | Renderer isolation, sandboxing and IPC hardening guidance | Actual packaged permissions and IPC review |
| [Electron ASAR integrity](https://github.com/electron/electron/blob/main/docs/tutorial/asar-integrity.md) | Optional packaged-code integrity and relevant fuse behavior | Platform/build compatibility and tamper tests |
| [Microsoft Desktop Duplication](https://learn.microsoft.com/en-us/windows-hardware/drivers/display/desktop-duplication-api) | GPU-memory desktop frames and metadata | Fullscreen/VM/driver behavior and GPU-copy cost |
| [Microsoft Windows Graphics Capture](https://learn.microsoft.com/en-us/windows/apps/develop/media-authoring-processing/screen-capture) | Display/window capture API and lifecycle | Specific app behavior, permission and resize handling |
| [Microsoft interactive services](https://learn.microsoft.com/en-us/windows/win32/services/interactive-services) | Separate interactive application and protected IPC pattern | Login/logoff/service recovery in supported deployments |
| [Microsoft SendInput](https://learn.microsoft.com/windows/win32/api/winuser/nf-winuser-sendinput) | Input injection and UIPI limitations | Game compatibility, layout semantics and latency |
| [Microsoft process audio loopback](https://learn.microsoft.com/en-us/samples/microsoft/windows-classic-samples/applicationloopbackaudio-sample/) | Per-process-tree capture with a stated OS-build requirement | App-tree transitions and audio isolation |
| [WebRTC peer connections](https://webrtc.org/getting-started/peer-connections) | ICE configuration, signaling and Trickle ICE | Ordering, restart and failure handling in this application |
| [WebRTC TURN guide](https://webrtc.org/getting-started/turn-server) | Configured authenticated TURN for relay connectivity | Real relay media/input in deployment topologies |
| [W3C WebRTC](https://www.w3.org/TR/webrtc/) | Peer/media/data-channel API semantics | Browser version interoperability and buffer behavior |
| [W3C WebRTC stats](https://www.w3.org/TR/webrtc-stats/) | Statistics counter definitions | Field availability and correlation with measured latency |
| [RFC 7742](https://www.rfc-editor.org/info/rfc7742/) | Baseline video codec interoperability requirements | Negotiated profile/level and hardware efficiency |
| [Native encoder factory](https://webrtc.googlesource.com/src/+/refs/heads/main/api/video_codecs/video_encoder_factory.h) | Native encoder integration interface; API may change | Provider implementation, build cost and maintenance |
| [Native port allocator](https://webrtc.googlesource.com/src/+/refs/heads/main/p2p/base/port_allocator.h) | Native port allocation control | Binding-range enforcement and firewall behavior |
| [NVENC programming guide](https://docs.nvidia.com/video-technologies/video-codec-sdk/13.0/nvenc-video-encoder-api-prog-guide/index.html) | Hardware encode and low-latency tuning options | End-to-end performance and negotiated codec compatibility |
| [PeerJS API](https://peerjs.com/client/api/peer) | Signaling abstraction and routing-ID caveat | Whether an abstraction helps the prototype |
| [PeerServer](https://github.com/peers/peerjs-server) | Self-hosted PeerJS signaling implementation | App-specific auth integration if used |
| [nut.js installation](https://nutjs.dev/docs/installation) | Automation package requirements and prebuilt subscription | Electron ABI, redistribution terms and gaming suitability |
| [coturn example configuration](https://github.com/coturn/coturn/blob/master/examples/etc/turnserver.conf) | Authentication, listener and relay configuration | Version-specific secure deployment and load |
| [coturn server reference](https://github.com/coturn/coturn/wiki/turnserver) | Configuration/command reference | Validate against pinned source/release when docs differ |
| [Let's Encrypt IP certificates](https://letsencrypt.org/2026/01/15/6day-and-ip-general-availability/) | Public IP certificate availability and short lifetime | ACME tooling, reachability and renewal reliability |
| [OWASP WebSocket security](https://cheatsheetseries.owasp.org/cheatsheets/WebSocket_Security_Cheat_Sheet.html) | Origin, auth, resource limits and session guidance | Adversarial testing of this implementation |
| [OWASP password storage](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html) | Password hashing guidance | Work-factor and abuse-control tuning |
| [electron-builder hardening](https://www.electron.build/docs/features/security/) | Windows update signature checks and packaging options | Clean-machine upgrade/rollback/tampering |
| [HTML media standard](https://html.spec.whatwg.org/multipage/media.html) | Playback and browser media behavior | Real-device sound/autoplay handling |
| [W3C screen capture](https://www.w3.org/TR/screen-capture/) | Browser capture activation and permission semantics | Host-specific Electron/native consent path |
| [W3C Pointer Lock](https://www.w3.org/TR/pointerlock-2/) | Relative pointer input API design | Browser/device support; draft status is not certification |
| [W3C Gamepad](https://www.w3.org/TR/gamepad/) | Browser gamepad API design | Real controllers, secure context and Windows virtual-device bridge |
| [Sunshine documentation](https://docs.lizardbyte.dev/projects/sunshine/latest/) | Mature game-streaming comparison and feature reference | Same-lab benchmark; not a drop-in browser player |
| [Sunshine license](https://github.com/LizardByte/Sunshine/blob/master/LICENSE) | Exact upstream license text for dependency review | Chosen reuse/distribution model |
| [ViGEmBus archive](https://github.com/nefarius/ViGEmBus) | Existing virtual gamepad driver is archived | Maintained, signed alternative and redistribution agreement |

Sources establish API capabilities and constraints. The architecture, presets, timeout values, security defaults, milestones and performance gates in this repository are proposed engineering decisions, not claims those sources certify this application.

## 2. Provisional architecture decisions

| ID | Decision | Rationale / revisit condition |
|---|---|---|
| ADR-001 | One monorepo; frontend/backend product boundary | Shared protocol and installer; split repos only for a concrete ownership need |
| ADR-002 | Electron console plus native media worker target | Explicit GPU/encode/input control; confirm with Phase 0 measurements |
| ADR-003 | Native WebRTC APIs and custom authenticated WSS | Explicit ownership, ordering and recovery; PeerJS only if justified by prototype |
| ADR-004 | One designated host offerer | Simplifies negotiation; add perfect negotiation only if needed |
| ADR-005 | Direct path plus external TURN | Real Internet reachability needs relay fallback; separate UI access gateway |
| ADR-006 | Domain HTTPS preferred; trusted IP HTTPS allowed | Meets URL access requirement without certificate bypass |
| ADR-007 | One controller and one stream initially | Predictable resource/lease policy; viewer/SFU scope deferred |
| ADR-008 | Separate per-user input helper/watchdog | Media crash should not leave injected controls held |
| ADR-009 | H.264/VP8 SDR first; advanced codecs later | Baseline interoperability and measured latency before optional features |
| ADR-010 | App allowlist is policy, VM is stronger isolation | Shared desktop capture/input is not a tenant boundary |
| ADR-011 | One installer, explicit external infrastructure | Avoid pretending an installer supplies GPU drivers, public routing or relay capacity |
| ADR-012 | Documentation-only initial repository | User requested a production plan; implementation begins after feasibility scope is understood |

## 3. Highest risks and mitigations

| Risk | Impact | Mitigation / decision gate |
|---|---|---|
| Browser playout prevents target latency | Core gaming experience misses goal | Phase 0 physical latency gate; narrow tier or reconsider client requirement |
| Native libwebrtc/provider complexity | Schedule and ongoing patch cost | One provider first, pinned builds, adapter contract, budget maintenance |
| Internet UI unreachable despite TURN | User cannot even log in | Topology diagnosis; VPN/gateway route; separate page/media checks |
| App-only access escapes shared desktop | Host data/control exposure | Trusted mode labeling; dedicated VM/session and escape tests for untrusted access |
| Unattended VM has no active display/session | Black screen after reboot/RDP exit | Explicit session/virtual-display provisioning and out-of-band recovery |
| Virtual input driver unavailable or stale | Gamepad support/security blocked | Separate maintained signed driver selection gate; keyboard/mouse scope first |
| Authentication/signaling compromise | Full remote desktop takeover | Small attack surface, short grants, independent worker enforcement and security review |
| TURN billing or abuse | Unexpected operational expense | Auth, quotas, peer restrictions, real usage/cost reporting |
| GPU/game/anti-cheat incompatibility | Claims exceed real support | Published tested matrix; detect and explain, no bypass assumptions |
| Auto-update or migration failure | Unattended host inaccessible | Verified signatures, staged rollout, recovery package and tested rollback |

## 4. Owner decisions to finalize before implementation

Proceed with the plan's defaults unless changed, but record the choices before certifying a release:

- Personal access versus untrusted users: default personal trusted desktop, no multi-tenancy claim.
- Browser-only versus optional native client: default browser-only; revise performance tier if the browser misses measured goals.
- Exact first Windows/GPU/browser support: default Windows 11 x64, one reference NVIDIA GPU and desktop Chromium.
- Deployment: default LAN/VPN first, public direct host plus TURN next; hosted gateway is a separate deliverable.
- Resolution/FPS: default 1080p60 SDR; 120 FPS/HDR/4K are conditional later tiers.
- Distribution model/license, nut.js use, driver provider and codec redistribution review: unresolved owner/release decisions.
- Authentication: individual accounts, local owner bootstrap; public admin requires MFA implementation.
- Recording/clipboard/file transfer: disabled and out of initial scope.

No additional response is required to retain these planning assumptions. Actual reference hardware, game list and deployment budget are required before performance certification or a credible final delivery estimate.
