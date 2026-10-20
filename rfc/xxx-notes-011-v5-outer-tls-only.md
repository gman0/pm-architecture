# Notes v5: a sketch with Outer TLS only

Working notes for [011](./011-provider-onboarding-flow-improvements.md), next to the [v5 notes](./xxx-notes-011-provider-onboarding-flow-improvements-v5.md) and the [implementation plan](./xxx-notes-011-v5-impl-00-overview.md). Not the RFC itself. A sketch of v5 without the Inner TLS: every channel is protected only by the TLS that the Provider's Edge terminates. It's what v5 lists as the rejected alternative "Plain Sessions, trusting the Provider's Edge", worked out far enough to compare it with v5 and with the [two-RFC split](./xxx-notes-011-v5-rfc-split.md).

## The trust model

- **The Connection relies on the Provider's Edge**, which joins the trust boundary. In v5 the Connection is end to end and only passes through the Edge; here it ends there. Whatever terminates the Base URL's TLS (an ingress, a load balancer, a CDN or WAF) sees and could change everything that crosses it: the Onboarding Token, the Connection Secret, Session traffic in the clear, and the credentials the Provider's controllers write back for the data plane.
- **The outer TLS is the trust anchor** for the PM-side: web PKI on the Base URL, or the `Provider`'s `caBundle`, verified strictly. Whoever presents a valid certificate for the Base URL is the Provider-side.
- **PM can't check what sits at a Provider's Edge.** The assumption is placed on every Provider, and stated as a requirement in the Edge checklist.

## The layers

```
 Registration         Control Calls              Session
 ┌───────────────┐    ┌───────────────────────┐  ┌───────────────────────────────┐
 │ HTTP/1.1 POST │    │ HTTP/1.1 requests     │  │ HTTP/1.1 per Stream           │
 │ bearer: token │    │ bearer: Conn. Secret  │  ├───────────────────────────────┤
 │               │    │                       │  │ yamux Streams                 │
 │               │    │                       │  ├───────────────────────────────┤
 │               │    │                       │  │ WebSocket, binary messages    │
 │               │    │                       │  │ upgrade: Session token        │
 ├───────────────┤    ├───────────────────────┤  ├───────────────────────────────┤
 │ outer TLS     │    │ outer TLS             │  │ outer TLS                     │  ◀── ends at the Edge
 ├───────────────┤    ├───────────────────────┤  ├───────────────────────────────┤
 │ TCP           │    │ TCP                   │  │ TCP                           │
 └───────────────┘    └───────────────────────┘  └───────────────────────────────┘
```

Compared with v5: no Inner TLS anywhere, no WebSocket for Registration or Control Calls (plain HTTPS requests suffice when nothing needs protecting from the Edge), and yamux directly in the WebSocket's binary messages.

## What changes

### Registration
- **`POST /v1/connections`** over plain HTTPS, with `Authorization: Bearer <tokenID>.<tokenSecret>`, as in v5's earlier drafts. Request: idempotency key, `clusterName`, the Connection Secret's hash, the Connector CA Set. Response: `connectionID`, `unusedExpiresAt`.
- **The token format drops `keyfp`**: `pm1.<base64url(base URL)>.<tokenID>.<tokenSecret>`. There's no Server Key to pin.
- **Trust in the response comes from the outer TLS.** The Edge could use the token first, or change the Connector CA Set or the hash on the way in.

### Control Calls
- **Plain HTTPS requests** to `/v1/connections/{id}`, `…/connector-ca-set`, with the Connection Secret as bearer. No Control Session, no WebSocket, no ALPN.
- **"Unknown Connection"** is an ordinary HTTPS response, trusted through the outer TLS. Item 4's authenticated "unknown", the unknown-ID handshake limits and `PM-Server-Key` all go.
- **Path binding** (item 5) is moot: the request's path names the Connection, and the Edge is trusted not to rewrite it.

### Sessions
- **A Session token on the upgrade**, because without the Inner TLS nothing else proves the caller is PM's Connector, and the Edge can't verify client certificates (v5's constraint). A short-lived JWS, signed with the Connector Certificate's key:
  - header: `x5c` with the Connector Certificate;
  - claims: `aud` = the `connectionID`, `clusterName`, the served hosts, `exp` (e.g. 60 seconds), a unique `jti`;
  - sent as `Authorization: Bearer …` on `GET /v1/connections/{id}/session`, or as a WebSocket subprotocol value.
- **The Provider-side verifies it** against the Connection's Pinned Bundle (the certificate issued directly by a pinned CA, valid, with `clientAuth` and `digitalSignature`), checks the signature, `aud`, `clusterName` and `exp`, and refuses a `jti` seen before. With one active replica, the replay cache is in memory; with several, it would have to be shared, or `exp` kept so short that replays don't matter.
- **The token replaces the Hello**: the Connection, workspace and hosts are signed claims instead of a message inside the Inner TLS. That's stronger binding than v5's Hello, at the cost of a token format and a replay cache.
- **Versioning** moves from ALPN to the WebSocket subprotocol (`Sec-WebSocket-Protocol: pm-session.v1`).
- **yamux, the Streams and the Connector's routing by path** are unchanged.

### What stays as in v5
- The Connector CA, Connector Certificates, the Connector CA Set with compare-and-swap, Rollover, distrust, and the re-check of open Sessions when the bundle changes (the Provider-side keeps the certificate from each Session's token).
- The Connection Secret and its changes through the `ProviderEndpoint`; the tamper check; the leaked-Connection-Secret limitation.
- Everything on the PM-side: the Connector, the Connection Export, the policies, `platform-mesh-operator`'s Onboarding, health, recovery, abandoning, egress.
- `provider-gateway` with its operator proxy and kubeconfig `Secret`s.

### What goes away
- The Inner TLS on all three channels, and its test vectors.
- **Server Keys**: the `ServerKey` resource, generation, rotation, announcement in `GET`, `serverKeys` in the `ProviderEndpoint` and the `ProviderEndpointRegistration`, the administration runbook, KMS considerations.
- `keyfp` in the Onboarding Token, Registration inside the Inner TLS, `PM-Server-Key`, the authenticated "unknown", `ConnectionMismatch` and `EdgePathRewritten`, Edge diagnosis of failed inner handshakes.

## What it costs

Against a Provider's Edge that's compromised, malicious or careless (logging, sampling, captures):
- **It acts as the Provider in kcp** for as long as Sessions last: it holds the Provider's end of every Session and can open Streams of its own.
- **It fakes kcp to the Provider's controllers**, which then change or delete real resources.
- **It reads what crosses Sessions**: consumers' objects, claimed resources, and the credentials the controllers write back for the data plane (database passwords, access keys), even when the data plane's own traffic never passes that Edge.
- **It has the Connection Secret** from every Control Call, so it can push Pinned Bundles of its own, and, with a CA of its own pinned, change the Connection Secret through a Session and lock PM out.
- **It sees Onboarding Tokens before PM uses them**, and can register in the Owner's name or change what Registration sends and returns.
- **Session tokens in its logs** are replayable until they expire, unless the replay cache catches them.

That's a regression from RFC 006, where the Provider dials kcp outbound and its inbound Edge isn't on the path at all. v5's reasons for rejecting it ("Why the Edge matters") all apply: Edges often belong to someone else, keep data without anyone being malicious, may be ruled out by compliance, and PM can't check them.

## A narrower variant: TLS terminated by `provider-gateway`

If the Provider's Edge passes TLS through (an L4 load balancer, SNI routing) and `provider-gateway` terminates the Base URL's TLS itself, the outer TLS is end to end, and the Edge sees only ciphertext, as with v5's Inner TLS. The Provider-side is then authenticated by the Base URL's certificate (web PKI or `caBundle`), which `provider-gateway` holds.
- **What it gives**: v5's confidentiality and integrity through the Edge, without a second TLS layer or Server Keys.
- **What it costs**: it contradicts v5's constraint that any HTTPS endpoint with WebSocket support must do, with no TLS passthrough. Many Providers' Edges are CDNs or WAFs that terminate TLS by design. It could be a recommendation, not a requirement; PM still couldn't check it.
- **The certificate becomes the Provider's identity**: whoever can get a certificate for the Base URL (a CA mis-issuance, a DNS takeover, the Edge itself if it ever terminates) is the Provider-side. v5's pinned Server Keys don't depend on web PKI.

## Compared with v5

| | v5 (Inner TLS) | Outer TLS only |
|---|---|---|
| Layers on Sessions | outer TLS, WebSocket, Inner TLS, Hello, yamux, HTTP | outer TLS, WebSocket (with a Session token), yamux, HTTP |
| Registration and Control | WebSocket + Inner TLS each | plain HTTPS requests |
| Provider-side keys | Server Keys, pinned per Connection, rotated with overlap | none |
| Connector authentication | mutual TLS inside the WebSocket | a signed token on the upgrade, with a replay cache |
| Trust anchor for the Provider | pinned Server Keys (first one from the token) | web PKI or `caBundle` |
| The Provider's Edge | carries the Connection, not relied on (end to end) | relied on, inside the trust boundary |
| Against a compromised or careless Edge | sees opaque bytes | acts as the Provider in kcp, fakes kcp, reads Connection Secrets, data-plane credentials and tokens |
| Relative to RFC 006 | equal or better | worse: the inbound Edge joins the credential path |
| Design and implementation size | larger (01's inner layer, Server Keys in 07, item 1, 4 and 5 machinery) | noticeably smaller |

## Where it could fit

- **On its own**: only if PM accepts relying on every Provider's Edge, and states it as a requirement Providers must meet.
- **As an interim first RFC**, with the Inner TLS as the second: possible, but it ships the regression for a while and adds the Session token that the second RFC removes (see the [split notes](./xxx-notes-011-v5-rfc-split.md) for a split that avoids both).
- **The narrower variant** (TLS terminated by `provider-gateway`) as a recommendation inside v5: it doesn't replace the Inner TLS, since PM can't rely on it, but it removes the Edge from the path for Providers that can do it.

## Documents it would change

- **v5 notes**: trust boundaries, core idea, Connection API, "The Provider's Edge: Inner TLS on Sessions" (replaced by the trust statement above), threat model and credential table, the rejected alternatives (the plain-Sessions entry and the Session-token entry reversed).
- **01 Protocol**: endpoints, Registration, Control Calls as plain HTTPS, the Session token in place of the inner layer and the Hello, versioning by subprotocol, test vectors.
- **02 APIs**: no `serverKeys` in the `ProviderEndpoint` and the `ProviderEndpointRegistration`; no `keyfp`-related fields.
- **03 Operator**: Registration and the first Control Call without Inner TLS; health and Edge diagnosis without inner-handshake outcomes; no Server Key upkeep.
- **05 Connector**: opening a Session with a token instead of the inner handshake.
- **07 SDK**: no `ServerKey`, no Inner TLS server; token verification and a replay cache; the token format without `keyfp`.
- **08, 10**: the Edge checklist states the trust requirement; the security tests for a malicious Edge become tests of what it can do, not of what it can't.
