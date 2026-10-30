# Installation, networking, HTTPS, and TURN

All settings and commands here describe the proposed product. Configuration snippets are design examples, not a tested deployment bundle.

## 1. Distribution and install flow

Ship a signed Windows installer, not a promise that one executable can contain every external dependency. The installer bundles Electron, compiled web assets, the control server, native streaming worker, migrations and required redistributables with redistribution permission. A running installation has multiple processes. GPU drivers, Windows features, optional virtual devices, certificates and an Internet relay may require separate provisioning.

1. Verify package signature and supported OS/architecture; detect an existing installation and offer upgrade or repair. Preserve owner configuration with versioned migrations and backup.
2. Show the installation scope: per-user recommended initially; optional machine broker requires elevation. Do not run the whole Electron application as Administrator.
3. Check GPU adapters, driver/API support, hardware encoder session availability, active Windows user session, display presence, audio endpoints and disk space. Distinguish fatal, degraded and optional findings.
4. If headless, explain that a GPU alone does not guarantee an active capture display. Offer instructions for a supported virtual display or hardware display emulator; require a separately trusted, signed driver where applicable.
5. Bind setup to loopback only. Create the owner account through the local console with a single-use bootstrap capability. No remotely accessible first-user-wins setup endpoint.
6. Choose exposure mode: local machine, LAN/VPN, public host, or a separately deployed access gateway. Choose exact interfaces; avoid binding every network adapter by default.
7. Configure trusted HTTPS, certificate renewal and recovery. No external login while TLS is invalid.
8. Configure display/app policy, audio, session timeout, local approval requirements and input rights. Provide a view-only test first.
9. Configure STUN/TURN if needed, then execute real connectivity tests. A successful TCP port probe is insufficient for WebRTC readiness.
10. Present topology-specific firewall changes and router/cloud-security-group instructions. Apply narrowly scoped Windows rules only after local owner approval. Do not silently change UPnP or disable endpoint protection.
11. Verify the URL from another device or external probe appropriate to the deployment; display distinct local, LAN and public URLs with their tested status. QR codes contain the URL only, no password or durable access token.
12. End with Ready, Degraded or Action Required, with a runnable validation checklist. Installation success and Internet streaming readiness are different states.

### Subsequent starts

Acquire a single-instance lock, load and validate configuration, check certificate and database health, start control server, locate the intended interactive session and start its worker. Run a quick capture/encoder check. Do not start capturing until an authorized session requests it.

If no user is logged in, management may report `HOST_SESSION_UNAVAILABLE`; capture remains unavailable. Do not enable automatic Windows login as an invisible workaround. A public unattended VM requires an explicit operational policy for login, reboot, maintenance and out-of-band recovery.

## 2. HTTPS and address policy

Preferred: a stable domain name with a trusted, automatically renewed certificate. An IP-based URL is possible when its certificate covers that IP. Public IP certificates are no longer categorically unavailable: Let's Encrypt announced general availability in January 2026 with a 160-hour lifetime. Validate ACME client support and automate renewal before choosing this route. [Let's Encrypt IP certificate announcement](https://letsencrypt.org/2026/01/15/6day-and-ip-general-availability/)

For private LAN IPs, provide an organization-managed CA installed on client devices, a correctly configured internal domain/certificate arrangement, or a VPN hostname with trusted HTTPS. A private IP is not solved by obtaining an unrelated public-IP certificate. Do not instruct users to bypass browser certificate warnings as the normal setup path.

Use HTTPS for UI/API and WSS for signaling on the same origin. Default external port: TCP 443; allow an explicit alternate such as 8443 when a reverse proxy or another service owns 443. Local onboarding can use loopback, but it must not become a remote plaintext password path.

Renew before the certificate's renewal deadline with jittered retries. Alert locally and through configured administrative channels well before expiry. Reload certificates without terminating healthy streams when supported. If renewal fails, do not fall back to plaintext. Re-test after DHCP/public IP changes. Certificate issuance challenges may need additional reachability; expose TCP 80 only if the selected challenge requires it.

## 3. Separate three kinds of reachability

1. **Page/API reachability:** Can the client reach the host's HTTPS endpoint?
2. **Signaling reachability:** Can authenticated WSS remain connected through proxies and idle timeouts?
3. **Media reachability:** Can ICE select a usable direct or relayed WebRTC path?

STUN discovers candidates; TURN relays WebRTC packets. Neither serves the web UI through CGNAT. A host behind CGNAT still needs a VPN, a public reverse access gateway, or another way to make its HTTPS endpoint reachable. Direct public-IP access requires routing and firewall configuration regardless of whether TURN is configured. [WebRTC peer connections](https://webrtc.org/getting-started/peer-connections), [WebRTC TURN](https://webrtc.org/getting-started/turn-server)

### Deployment modes

| Topology | UI/signaling path | Media path | Expected action |
|---|---|---|---|
| Same LAN | Private trusted HTTPS address | Direct UDP if LAN allows | Firewall/interface validation; TURN optional |
| VPN | Reachable VPN address/name | Direct over VPN if supported | Check MTU/routing and performance |
| Public Windows host/VM | Public domain/IP, TCP 443 | Direct UDP or TURN | Windows and cloud ingress rules, valid TLS |
| Router NAT with public WAN IP | Domain plus port-forwarded HTTPS | ICE direct when possible, TURN otherwise | Verify external address and hairpin limitations |
| Double NAT / CGNAT | VPN or outbound connector to external HTTPS gateway | TURN or working direct path | Router forwarding alone may not work |
| Corporate network blocks UDP | HTTPS gateway/host | TURN over TLS/TCP if allowed | Label latency degradation; proxies can still block it |
| No external connectivity | LAN UI only | LAN direct only | Explain offline mode; no mandatory public STUN dependency |

A reverse gateway is an additional optional infrastructure product. It must authenticate the host connector, bind each host to its owner, preserve end-to-end application authorization, support WebSocket upgrades and prevent cross-host routing. Do not tunnel the media through an HTTP server accidentally; TURN handles relay media. A gateway operator terminates HTTPS in the usual proxy deployment and is part of the control-plane trust boundary.

## 4. Port and firewall matrix

These are proposed defaults. The setup wizard must generate instructions from actual bindings and selected topology, not tell every user to open every port.

| Machine / direction | Port/protocol | Purpose | Requirement |
|---|---|---|---|
| Windows host inbound | TCP 443, or configured HTTPS port | UI, REST, WSS | Needed where clients reach host directly; scope to trusted networks where possible |
| Windows host inbound | UDP 50000-50100, proposed configurable range | Native worker direct ICE media/data | Only if the native port allocator is configured and verified to use it |
| Windows host outbound | Configured STUN/TURN endpoints | Discovery/relay allocation | UDP preferred; TCP/TLS fallback as configured |
| TURN node inbound | UDP/TCP 3478 | STUN/TURN listener | Enable only transports actually offered |
| TURN node inbound | TCP 5349 | TURN over TLS | Optional fallback, valid TURN hostname certificate |
| TURN node inbound | TCP 443 on a dedicated TURN IP | Restrictive-network TLS fallback | Optional; conflicts with HTTPS on the same socket |
| TURN node inbound/outbound | UDP 49152-49251, example relay range | TURN relayed peer traffic | Must match coturn configuration and NAT forwarding |
| Client outbound | HTTPS plus negotiated UDP / TURN transport | UI, signaling, media/input | No manual inbound client port forwarding normally required |
| Certificate endpoint | TCP 80 if HTTP-01 is selected | ACME challenge | Conditional, not required for every certificate method |

Do not equate the host media range with the TURN relay range. Chromium/Electron sender media ports are not controlled by a standard browser JavaScript setting. For the native worker, configure the supported libwebrtc port allocator and verify actual socket bindings; the API exposes range control. Until this is implemented, use program-scoped firewall rules rather than falsely advertising a fixed media range. [libwebrtc port allocator](https://webrtc.googlesource.com/src/+/refs/heads/main/p2p/base/port_allocator.h)

For TURN behind NAT, relay port mappings must preserve the configured ports and advertise the correct public/private mapping. Opening only 3478 is insufficient. Check IPv4 and IPv6 separately. Disable unexpected alternate listeners or include them in the generated matrix. Avoid oversized open ranges; size the relay range using measured allocation demand plus headroom. [coturn configuration](https://github.com/coturn/coturn/blob/master/examples/etc/turnserver.conf)

## 5. TURN provisioning workflow

Recommend a managed TURN service or a dedicated Linux VM near the Windows host and likely users. A TURN process on the same unreachable private network cannot solve reachability. The Windows installer can contain a TURN setup assistant without bundling and operating an Internet relay itself.

1. Provision public networking and DNS for the relay; choose UDP, TCP and TLS listeners, relay range, capacity and region.
2. Install a maintained, pinned coturn release using an official source; create a least-privileged service account and protected configuration.
3. Install and automate a valid certificate for the TURN hostname when offering TLS.
4. Configure authenticated allocations, quotas, bandwidth limits, relay range, peer-address policy and logging. Anonymous/open relay operation is forbidden.
5. Store the shared authentication secret on the control server and TURN node only. Mint short-lived per-session credentials after authorization. The browser receives those derived credentials, never the shared secret.
6. Configure peer-address restrictions for the deployment. A public relay must not become an internal-network scanner. If a private Windows peer is an intentional supported destination, use explicit narrow allowances rather than a blanket private-network exemption.
7. Configure host and browser ICE server URLs, then test a forced-relay connection in both UDP and TLS/TCP modes. Gathered relay candidates alone do not prove media works.
8. Test secret rotation and long-running allocation refresh. Credential expiry is not an instant way to terminate an established stream; use application revocation and close the peers.

Illustrative configuration, to be completed and validated against the selected release:

```ini
# DESIGN EXAMPLE ONLY: never start unchanged.
listening-port=3478
tls-listening-port=5349
min-port=49152
max-port=49251
realm=turn.example.com
fingerprint
use-auth-secret
static-auth-secret=REPLACE_FROM_SECRET_STORE
cert=/etc/turn/tls/fullchain.pem
pkey=/etc/turn/tls/privkey.pem
no-cli
no-multicast-peers
# Configure listening/relay addresses, NAT mapping if needed,
# explicit peer restrictions, allocation quotas and bandwidth limits.
```

coturn supports shared-secret credentials and relay-port configuration; use one documented authentication mode and review its deployed version's configuration. [coturn configuration reference](https://github.com/coturn/coturn/wiki/turnserver)

### Credential issuance contract

For coturn REST-style authentication, the intended username is `expiryUnix:sessionOpaqueId`, with a credential computed server-side using the documented HMAC mechanism. Proposed initial TTL: 10 minutes, with credentials refreshed before a required new allocation/ICE restart. Validate allocation refresh and long-session behavior in the chosen client/server builds. Synchronize clocks and reject issuance if host time is badly skewed.

Keep raw credentials out of URLs, logs and support bundles. Apply per-user allocation and bandwidth quotas; protect the issuance endpoint from abuse. Document relay cost separately from server cost.

## 6. Diagnostics the product must provide

Show each check as Pass, Warning, Fail or Not Tested, with a reason and next action:

- DNS resolution and certificate identity/chain/expiry; client device clock.
- Local HTTPS binding, port conflict, interface selection and external reachability.
- Windows firewall profile, configured rule and cloud firewall/router steps that need manual action.
- Host session, desktop availability, GPU/driver/encoder support and display state.
- WSS upgrade, authentication, ping/pong and proxy idle timeout.
- Host/server-reflexive/relay candidate gathering; selected pair type and transport.
- Forced-relay video, audio and bidirectional input test, including return traffic.
- Measured RTT, loss, bandwidth, decode capacity, browser controls and recommended preset.
- NAT hairpin, CGNAT indication, IPv6 route and VPN route differences. NAT diagnosis is advisory, not proof from one HTTP IP check.

Never describe a firewall as open based solely on local rule existence. Offer redacted diagnostic export and a copyable remediation guide. Any active external probe must be explicit about the endpoint and data sent.
