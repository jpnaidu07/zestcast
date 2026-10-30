# ZestCast

Browser access to a Windows desktop or selected application, designed for interactive desktop use and low-latency gaming.

**Status: researched architecture and implementation plan. No streaming application or installer has been implemented or benchmarked.** Prepared 30 October 2026. Performance numbers in these documents are proposed acceptance targets, not measured results.

## Recommended design

Ship one Windows installer containing an Electron host console, the web client, an HTTPS API and signaling server, and a native Windows streaming worker. Keep the frontend and backend as two product areas within one monorepo. Use WebRTC for media and input; authenticated WebSocket signaling coordinates sessions. Support direct UDP connections and an external TURN relay.

Start with an Electron capture prototype to validate the flow. Make the native capture/encode/WebRTC path a release gate for the gaming tier unless measurements demonstrate that Electron already meets the agreed target. Do not promise arbitrary GPU selection, unrestricted game compatibility, or equal functionality on every browser.

## Read the plan

1. [Product requirements and architecture](docs/01-product-and-architecture.md)
2. [Installation, networking, HTTPS, and TURN](docs/02-installation-and-networking.md)
3. [Connection flow, APIs, and recovery protocol](docs/03-session-protocol.md)
4. [Capture, encoding, audio, input, and browser experience](docs/04-media-input-and-client.md)
5. [Security, isolation, and operations](docs/05-security-and-operations.md)
6. [Delivery roadmap and release acceptance](docs/06-delivery-and-validation.md)
7. [Research sources and decision register](docs/07-research-and-decisions.md)

The design describes intended behavior. It does not claim all Windows configurations, games, networks, or browsers can be supported. Unsupported states must be detected and explained, and supported states must pass the release matrix.

## Proposed implementation layout

```text
apps/
  host/                 Electron console, bootstrap, packaging
  web/                  Browser login, setup, player, admin
services/
  control/              HTTPS API, auth, policy, WSS signaling
native/
  stream-worker/        C++ capture, encode, libwebrtc, audio, input
  session-broker/       Optional Windows service for lifecycle only
packages/
  protocol/             Versioned schemas and generated bindings
  config/               Validated settings and migrations
  ui/                   Shared presentation components
infra/
  turn/                 Reviewed coturn deployment configuration
tests/
  protocol/             State transitions, malformed messages, recovery
  integration/          Browser/host interoperability
  hardware/             GPU, display, input, latency harnesses
docs/                   Requirements, ADRs, runbooks, evidence
```

These are planned code directories; only documentation exists in this initial repository.

## Initial scope

One Windows host, one active Windows user session, one display or app per stream, one controller. Windows 11 x64 and desktop Chromium browsers form the first certification target. Mobile touch access follows with explicit capability limits. Windows Server with Desktop Experience and a supported GPU is a separate validation target. Server Core is outside scope.

No default administrator password. No anonymous signaling or capture. No public recording, file transfer, arbitrary remote shell, automatic router configuration, or multi-tenant isolation in the first release.

## Repository publication

The GitHub repository is private: [jpnaidu07/zestcast](https://github.com/jpnaidu07/zestcast). This initial repository contains the researched production plan; application implementation has not started.

Do not commit real passwords, TURN secrets, private keys, recordings, packet captures, or unredacted diagnostic bundles. Licensing for this project's original code remains an owner decision; dependency licenses must be reviewed before distribution.
