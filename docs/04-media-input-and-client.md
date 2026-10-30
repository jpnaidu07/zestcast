# Capture, encoding, audio, input, and browser experience

## 1. Capability negotiation

Create a capability intersection from host OS, capture source, capture GPU, encoder GPU, encoder/driver, browser RTP codec support, observed decode performance, display refresh and network limits. A browser advertising a codec does not prove hardware decoding or acceptable latency at the requested resolution.

Return both requested and effective settings with reasons: for example, `requested: 1080p120; effective: 1080p60; reason: client-tier-limit`. Offer Auto as the default. Never silently advertise 120 FPS while sending 60.

Distinguish **display**, **capture adapter** and **encoding adapter** in the admin UI. Persist Windows adapter LUID plus display-path identity where possible, not just enumeration index or a friendly name. Revalidate saved identities after reboot/driver changes. Cross-adapter copies may reduce performance; disable incompatible combinations with an explanation. Electron source IDs/display IDs assist enumeration but do not themselves promise encoder adapter control. [Electron capture source structure](https://www.electronjs.org/docs/latest/api/structures/desktop-capturer-source)

## 2. Native capture pipeline

Proposed pipeline:

```text
DXGI output / WGC window
  -> timestamped D3D11 texture
  -> GPU scaling and SDR color conversion
  -> hardware encode via libwebrtc encoder adapter
  -> RTP pacing / congestion control
  -> ICE-selected network path
  -> browser decode and presentation
```

Use DXGI Desktop Duplication for display capture and evaluate Windows.Graphics.Capture for application windows. DXGI exposes frames in GPU memory and cursor/dirty-region information; WGC supports capture of a display or application window. Source-specific behavior still requires testing. [Microsoft Desktop Duplication](https://learn.microsoft.com/en-us/windows-hardware/drivers/display/desktop-duplication-api), [Microsoft Windows Graphics Capture](https://learn.microsoft.com/en-us/windows/apps/develop/media-authoring-processing/screen-capture)

Required capture behavior:

- Probe source support before committing the session. Handle rotation, negative monitor origins, mixed DPI, scaling, resize, HDR desktop-to-SDR conversion and color range explicitly.
- Bound every stage's queue; prefer the latest eligible frame over accumulating stale frames. Target one pending capture frame, with the actual encoder buffers measured and reported.
- Keep textures on the GPU where supported. Record texture format, copies, conversion and cross-adapter costs. "Zero-copy" is an optimization to prove, not a blanket product claim.
- Recreate duplication/frame pools on source resize, access loss or GPU reset. Release old references before reallocating. Add crash/recreate limits to avoid loops.
- Decide cursor composition deliberately: host-composited cursor is accurate but delayed; a separate local cursor can feel faster but must match shape/hotspot and avoid a duplicate host cursor. Hide the local cursor for relative game input.
- For window mode, bind to native window identity, owning process and creation identity; handle PID/window-handle reuse, child windows, launcher-to-game transitions and popups through explicit policy.
- A minimized or occluded app can freeze or behave differently by capture backend. Detect lack of progress, communicate it and test real applications. Do not claim universal minimized capture.
- Protected content and secure desktops may be unavailable. Show an explicit protected/unavailable state; do not attempt to defeat content protection.
- In headless and RDP-disconnected VMs, verify a real render/capture path, encoder access and active desktop. Run tests without leaving an RDP session connected as an accidental dependency.

## 3. Encoding and adaptation

Initial interoperability profile: H.264 SDR 8-bit 4:2:0 with a mutually negotiated profile/level and packetization mode; VP8 fallback. Opus for audio. Implement the required baseline interoperability; AV1/HEVC/HDR are optional, feature-detected tiers. WebRTC's mandatory video baseline includes VP8 and H.264 Constrained Baseline. [RFC 7742](https://www.rfc-editor.org/info/rfc7742/)

Proposed native encoder providers: NVENC for NVIDIA, AMF for AMD and oneVPL/Quick Sync for Intel, with hardware and driver compatibility validated independently. Choose one reference provider first; do not promise all three on day one.

For the initial low-latency profile, disable conventional frame reordering, bound lookahead, use a small encoder buffer and adaptive target bitrate, and benchmark quality/performance presets. Evaluate hardware-specific newer features only after browser compatibility tests. NVIDIA documents low/ultra-low-latency tuning and small VBV configurations for game streaming; these are starting points, not universal settings for all vendors. [NVENC programming guide](https://docs.nvidia.com/video-technologies/video-codec-sdk/13.0/nvenc-video-encoder-api-prog-guide/index.html)

Integrate rate changes, keyframe requests, negotiated payload format, packet-loss feedback and encoder failures with libwebrtc. Hardware encoding by itself does not implement congestion control. Avoid independent bitrate controllers fighting each other: libwebrtc estimates capacity; an outer policy chooses capped resolution/FPS tiers with hysteresis.

Proposed starting presets, to tune using measured quality and motion:

| Preset | Ceiling | Initial video budget | Intended use |
|---|---|---|---|
| Data saver | 720p30 | 3-6 Mbps | Touch/weak links |
| Balanced | 1080p60 | 8-15 Mbps | Default interactive desktop and games |
| Quality | 1080p60 | 15-25 Mbps | Fast-motion games on a good path |
| Experimental high refresh | 1080p120 | 20-40 Mbps | Certified hardware/browser/refresh combination only |

Actual required bitrate varies with codec, content and quality. Reduce bitrate promptly under congestion, then resolution before FPS for a gaming profile; desktop text mode may prefer resolution over motion rate. Raise quality slowly after stability. Add manual ceilings and a policy for metered connections. Rate-limit keyframes; test scene-change bursts and recovery from packet loss.

Software encoding is a clearly labeled desktop fallback with reduced resolution/FPS. If gaming hardware encoding fails, fail or request an explicit downgrade rather than silently loading the CPU and claiming the gaming tier still holds.

## 4. Audio

Use native WASAPI endpoint loopback for whole-system audio and evaluate per-process loopback for restricted application capture. Microsoft's application-loopback sample supports a selected process tree and requires Windows build 20348 or later; gate by actual OS capability. [Microsoft application audio loopback](https://learn.microsoft.com/en-us/samples/microsoft/windows-classic-samples/applicationloopbackaudio-sample/)

Default to the selected app's audio for restricted mode; if that isolation is unavailable, offer no audio rather than silently capturing the whole system. Test launcher/process-tree changes. Detect missing audio devices, virtual audio endpoints, silence, endpoint changes and audio service restarts. Provide a test sound.

Use Opus with settings validated for stereo game audio. Do not apply voice echo cancellation/noise suppression to game output by default. Maintain monotonic capture timestamps and use stack synchronization; measure A/V skew and drift. Muting local speakers, if offered, is an explicit host setting and must not break capture.

Microphone upstream and voice chat are separate later features needing explicit client permissions, routing, privacy controls and feedback-loop tests.

## 5. Input transport and safety

Use WebRTC data channels for in-session input. Keep setup, policy and authoritative configuration on HTTPS/WSS. Suggested channels:

| Channel | Reliability | Payload |
|---|---|---|
| `input-critical` | Reliable, ordered, small messages | Key/button edges, mode transitions, release-all and acknowledgments |
| `input-motion` | Unordered, `maxRetransmits: 0` | Sequenced pointer/gamepad snapshots; discard obsolete state |
| `telemetry` | Unordered, bounded | Optional ephemeral client measurements |

WebRTC permits ordered and partially reliable channels and exposes buffering. Use `bufferedAmount` limits and coalesce motion instead of queueing it. Separate channels can still share transport congestion, so never put bulk file transfers on this connection in v1. [W3C WebRTC data channels](https://www.w3.org/TR/webrtc/)

### Native input helper

Run input injection in a small per-user helper process with its own watchdog, separate from media capture/encoding. If the media worker or control server dies, that helper releases its injected state. Authenticate its IPC, restrict the named-pipe ACL and pass only bounded, schema-validated events and a short authorization lease. A Session 0 service is not an input-injection substitute for the correct user session.

Even this cannot execute cleanup after the whole machine freezes or the input helper itself is killed. On restart, reconcile only tracked injected state where safe; do not indiscriminately release physical local input. Test helper crash/restart and document the residual OS-level failure boundary.

### Input rules

- Controller ownership is exclusive and versioned by lease/epoch. Reject expired, duplicate, oversized, future or stale-epoch events in the helper, not only in the browser.
- Include a sequence number and source/layout epoch. For absolute mouse coordinates, map normalized video-content coordinates through letterboxing, display crop, rotation and DPI. Ignore taps outside the actual video rectangle.
- For games, use pointer-lock relative deltas and tested native relative mouse injection. A latest-state snapshot is not sufficient for relative motion: send sequenced cumulative displacement counters, apply differences once, and reset baselines on epoch/mode changes. This preserves lost deltas without replaying stale movements.
- Treat physical key locations and text entry separately: scan-code style input for games; explicit Unicode/text mode for desktop text. Handle keyboard layouts, AltGr, modifiers, repeat, IME limitations and browser-reserved shortcuts.
- Send immediate key-up/button-up when possible and periodic full pressed-state snapshots for reconciliation. Use freshness deadlines; on uncertain state or buffer growth, release and require a clean resynchronization.
- Release injected input on blur, hidden tab, pointer-lock exit, fullscreen exit when relevant, heartbeat expiry, disconnect, lease transfer and policy revoke. A dropped key-up must not leave a key held indefinitely.
- Do not assume pointer injection can control elevated windows. `SendInput` is restricted by UIPI to equal or lower integrity targets. Do not run the entire host elevated to bypass this. Secure attention sequences such as Ctrl+Alt+Del are outside the ordinary browser/SendInput path. [Microsoft SendInput](https://learn.microsoft.com/windows/win32/api/winuser/nf-winuser-sendinput)
- Gamepad browser state must be translated into a supported Windows virtual device. The browser API does not create a remote controller by itself. Select a maintained, signed and redistributable provider before promising controller support. ViGEmBus is archived and must not become an unquestioned new production dependency. [ViGEmBus repository](https://github.com/nefarius/ViGEmBus)
- Never claim all anti-cheat systems accept synthetic input or VM/remote streaming. Maintain a tested game list and expose incompatibility without offering bypasses.

## 6. Browser experience

The user-facing path stays short: Sign in -> select permitted source/preset -> Connect -> player. Advanced controls remain in an optional panel.

Use a real `<video playsinline>` receiver, no per-frame canvas redraw by default. Support keyboard navigation, accessible labels, high contrast, responsive layouts, safe-area insets and large touch targets. Include a persistent escape/Stop Control affordance and explain which shortcut stays local.

Use an explicit user gesture for sound, fullscreen and pointer lock. Browser autoplay policy can require an additional tap; catch playback rejection and offer Enable Sound/Resume. Standard browser screen capture requires user activation and selection, but the receiving browser is not capturing its own display. Capture consent/source policy happens on the Windows host. [HTML media standard](https://html.spec.whatwg.org/multipage/media.html), [W3C screen capture](https://www.w3.org/TR/screen-capture/)

Offer touchpad mode, tap-to-click, drag gesture, scroll, right-click and an on-screen keyboard. Native multi-touch injection is a separate capability from touch-to-mouse. Landscape mode is a suggestion, not a forced assumption. On orientation/viewport changes, pause input until layout mapping is acknowledged.

Pointer lock and Gamepad support must be checked and tested per browser/device. Provide absolute desktop control where relative game input is unavailable. Require trusted HTTPS for supported secure-context browser features. Working drafts describe intended APIs; they are not proof of deployed compatibility. [W3C Pointer Lock](https://www.w3.org/TR/pointerlock-2/), [W3C Gamepad](https://www.w3.org/TR/gamepad/)

Detect tab suspension, mobile lock, Bluetooth controller disconnect, decoder overload and thermal degradation. Pause safely and resume through the session protocol. Mobile receive/display-refresh limits determine the actual offered tier; do not promise 120 FPS on every phone.

## 7. Metrics without misleading latency claims

Sample at approximately 1 Hz; reduce telemetry overhead on constrained devices. Keep local detailed diagnostics opt-in. Display fields that exist in the implementation; unavailable is different from zero.

| Metric | Definition |
|---|---|
| Capture FPS | Frames acquired by the host capture source per interval |
| Encode FPS / time | Encoded frames and measured encoder processing; include native queue age |
| Receive/decode FPS | Delta decoded frames divided by elapsed time |
| Presented FPS | Video presentation callback/counter where supported; label separately from decode |
| Video bitrate | `8 * delta(bytesReceived) / delta(timeSeconds)`; say whether media bytes or wire estimate |
| Decode time | `1000 * delta(totalDecodeTime) / delta(framesDecoded)` |
| Jitter buffer delay | `1000 * delta(jitterBufferDelay) / delta(jitterBufferEmittedCount)` |
| RTT | Current selected candidate-pair RTT, clearly labeled network round trip |
| Loss / jitter | Interval RTP loss estimate and RTP jitter converted to ms; handle late packets and resets |
| Route | Selected local/remote candidate types and transport; direct or relay |
| Quality | Actual dimensions, FPS ceiling, codec/profile where observable, degradation reason |
| Host health | GPU/CPU/RAM, encoder provider, adapter identities, capture/input watchdog status |

The WebRTC stats specification defines cumulative counters and optional fields. Reset baselines after a new stats ID, SSRC, peer or counter reset; guard zero denominators and avoid summing overlapping processing-time metrics. [W3C WebRTC stats](https://www.w3.org/TR/webrtc-stats/)

Do not call RTT or input acknowledgment time "total latency". True input-to-photon requires instrumentation across input, app rendering, capture, encoding, network, decoding and display scanout; certify using a physical input/LED and high-speed camera or photodiode method. Cross-machine timestamp subtraction needs validated clock alignment and error bounds. A software estimate must be labeled as an estimate with its missing stages.
