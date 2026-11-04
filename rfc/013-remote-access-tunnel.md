---
title: Remote access — tunnel mode
---

# RFC 013: Remote access — tunnel mode

| Status  | Proposed   |
|---------|------------|
| Author  | TBD        |
| Created | 2026-11-03 |
| Updated | 2026-11-03 |

## Summary

This RFC extends [RFC 012](./012-remote-access-baseline.md) with persistent tunnelling for the case where PM's kcp front-proxy is not directly reachable from the user-side cluster. It adds:

- A **reverse L4 tunnel** carried over the same TLS connection as `/access`, kept open by `access-operator` with leader election and reconnected on drop.
- An **L4 SNI router** upstream of PM's kcp front-proxy, routing incoming tunnel traffic to the right front-proxy or shard without terminating TLS.
- A **reconnect-authentication plane** — short-lived session tokens, signed by a per-`Access` session-signing key — disjoint from the initial-handshake HMAC, so that rotating the HMAC on `AccessIdentity` does not disturb running tunnels and rotating one `Access`'s session key does not disturb sibling `Access`es.
- Tunnel negotiation on `/access`: a tunnel-request flag from PM and a tunnel-policy field on `AccessIdentity` for user-side opt-out.

The baseline handshake, CRDs, and `Provider` extension defined in RFC 012 are unchanged. Tunnel mode is opt-in per deployment.

## Motivation

RFC 012 requires that PM's kcp front-proxy be reachable from the user-side cluster — that's where the delivered kubeconfig's `server` field points. For PM instances that are private (internal networks, regulated environments, split-brain deployments), this does not hold. The Customer cannot reach kcp directly.

PM can still reach the user-side `/access` endpoint (that's the RFC 012 requirement). This asymmetry means PM can *pull* the user-side cluster toward it: establish a tunnel from PM out to the user-side access SDK, then route kcp traffic back through that tunnel. The operator continues to use a normal-looking kubeconfig; the traffic flows through a local proxy that the access SDK exposes, through the tunnel, into PM's SNI router, into kcp.

Tunnels drop. Reconnecting requires authentication. If reconnect had to re-use the access token, then rotating the HMAC secret on `AccessIdentity` would kill every running tunnel at the next drop — an undesirable coupling between credential hygiene and running-connection stability. Introducing session tokens on a disjoint signing key decouples these: the running tunnel holds a session token that is re-authenticated independently of the access token.

**Goals:**

- Make PM-initiated remote access work when PM's kcp is not publicly reachable from the user-side cluster.
- Decouple reconnect authentication from the access token so HMAC rotation never disturbs running tunnels.
- Provide an incident-response lever that forces re-authentication of currently-active tunnels (session-signing-key rotation).
- Keep the on-the-wire protocol version-selectable (ALPN); stay within `pm-access/1` while additions are additive, bump only on a real shape break.

**Non-goals:**

- Tunnelling non-kcp traffic (e.g., the Provider's own service data plane).
- Removing the reachability requirement that PM can still reach `/access` — this is a one-side reachability relaxation, not a two-sided one.
- Changing the access token format, `AccessIdentity` HMAC semantics, or any RFC 012 CRD shape beyond the additions documented here.

## Terminology

In addition to RFC 012 terminology:

| Term | Meaning |
|---|---|
| **tunnel** | Persistent reverse L4 channel between access-operator (PM) and the access SDK (user-side), established over `/access`. |
| **SNI router** | New L4 component in PM's kcp front-proxy stack; routes incoming TLS traffic by `ClientHello` SNI without terminating TLS. |
| **session token** | Short-lived JWT, signed by a per-`Access` session-signing key, issued on `/access` 200 and presented on subsequent reconnect dials. |
| **session-signing key** | Per-`Access` asymmetric key held by the access SDK (in a Secret owned by the `Access`); used to sign and verify session tokens. |
| **`NeedsReonboarding`** | Reason surfaced on `Provider.status.conditions[Connected]=False` when session-token verification fails and recovery requires a new PAR. |

## Architecture

### SNI router (PM-side)

kcp's front-proxy is extended with an L4 SNI router placed *upstream* of TLS termination. It reads the `ClientHello` SNI hostname and forwards the raw TCP stream to:

- the front-proxy itself, if SNI matches the front-proxy's hostname;
- a specific kcp shard, if SNI matches a shard's VW hostname.

TLS is never terminated at the SNI router. The operator's connection is end-to-end TLS with the front-proxy or shard it addresses; the SNI router is transparent. This is new kcp work and the main implementation cost of this RFC.

### Tunnel transport

After the RFC 012 handshake completes on `/access`, if PM requested a tunnel and the access SDK accepted, the same TCP connection is upgraded to a persistent bidirectional L4 channel. The access SDK exposes a per-`Access` local HTTPS endpoint that writes operator traffic into the tunnel; access-operator on the PM side writes tunnel traffic into the SNI router on PM's network.

One tunnel per `Access`. A single tunnel carries all kcp traffic for that connection — front-proxy calls and per-shard VW calls — distinguished by the TLS SNI in each inner TLS handshake.

**Framing: WebSocket + yamux.** The TCP+TLS connection is upgraded via `Upgrade: websocket` (RFC 6455); inside the WebSocket, a yamux stream multiplexer carries each operator kcp call as its own stream. WebSocket is chosen for middlebox compatibility — HTTP/2 extended CONNECT would be cleaner but doesn't survive the long tail of ingress stacks PM has to work with. Yamux provides real multiplexing without requiring HTTP/2 end-to-end, is battle-tested (Consul, Nomad), and has mature Go support (`hashicorp/yamux`).

### Kubeconfig in tunnel mode

The delivered kubeconfig points `server` at kcp's hostname (same as direct mode) and additionally sets `proxy-url` to a local access-SDK endpoint that fronts the tunnel. Operator Pods see a normal kubeconfig; they don't know a tunnel exists.

```yaml
clusters:
- cluster:
    server: https://kcp.pm.example               # same as direct mode
    proxy-url: https://<access-uid>:@access-tunnel.svc:443   # trailing colon: empty password
    certificate-authority-data: <PM's kcp CA>
```

The `<access-uid>` is encoded in the proxy URL's userinfo (as the Basic-auth username, with an empty password), so Go's HTTP transport sends it as `Proxy-Authorization: Basic base64("<access-uid>:")` on the CONNECT request. The access SDK uses this header to route the CONNECT to the correct per-`Access` tunnel backend. See [Local proxy endpoint URL](#local-proxy-endpoint-url) below for why path-based routing doesn't work and for alternative deployment schemes.

The `proxy-url` host is internal to the user-side cluster; PM never sees or references it.

### Tunnel negotiation

RFC 013 adds two fields:

**On the `/access` request** (PM → user-side): `tunnel` flag.

```json
{
  "kcpURL": "...",
  "kcpCABundle": "...",
  "saToken": "...",
  "tunnel": "required"
}
```

Values: `required` (PM is non-public; the tunnel must be established or the connection is useless) or absent (direct mode).

**On `AccessIdentity.spec`** (user-side): `tunnel` policy.

```go
type AccessIdentitySpec struct {
    // ... baseline fields ...
    Tunnel TunnelPolicy `json:"tunnel,omitempty"`  // Allow (default) | Deny
}
```

`Allow`: access SDK honours the tunnel request. `Deny`: access SDK refuses. On a PM `required` + user-side `Deny` mismatch, `/access` returns `403 TunnelDenied` and `AccessGrant` surfaces the reason.

### Reconnect-authentication plane

Baseline `/access` is a one-shot call. In tunnel mode, the tunnel can drop (network blip, access SDK restart, access-operator failover). Reconnecting requires authentication, and that authentication must not couple to the long-lived access token (so HMAC rotation doesn't disturb running tunnels). It must also be **scoped per-`Access`**, so that rotating one `Access`'s reconnect credential does not affect sibling `Access`es under the same `AccessIdentity`.

#### Per-`Access` session-signing key

On `Access` creation (i.e. when the access SDK accepts a `/access` initial handshake and creates the `Access` object), the SDK additionally generates a **per-`Access` session-signing key** and stores it, together with JTI state (below), in a Secret owned by the `Access`.

State per `Access`, held by the access SDK:

- `signing_key` — private key material for signing session tokens. `EdDSA` (Ed25519) is the default; `ES256` is a conforming alternative.
- `kid` — SHA-256 fingerprint of the signing public key (self-describing; used in JWT headers).
- `current_jti` — `jti` of the most recently issued session token.
- `previous_jti` — the one before (grace-window entry; see [Lost responses and grace window](#lost-responses-and-grace-window) below).

#### Session token: shape and issuance

On `/access` 200, the access SDK issues a **session token** — a short-TTL JWT signed by the `Access`'s signing key — in the response body:

```json
{
  "sessionToken": "<JWT>",
  "sessionTokenExpiresAt": "<RFC3339 timestamp>"
}
```

JWT header:
- `alg` — `EdDSA` (default) or `ES256`. The verifier rejects any other value.
- `kid` — SHA-256 fingerprint of the signing public key; identifies which generation of the `Access`'s key signed this token.

JWT body:
- `sub` — `Access` UID (authoritative binding; this is what the token is "about").
- `iat`, `exp` — standard.
- `jti` — random 128-bit, unique per issuance.

No `aid`, `aud`, `iss`, `nbf`: the chain `sub → Access → signing key` resolves verification without needing an Identity hint; no meaningful audience exists for an SDK that is the sole recipient of its own tokens; audit claims can be added later if the need arises.

access-operator stores the token on `AccessGrant.status.sessionToken` + `.sessionTokenExpiresAt`.

#### Session token: presentation and verification

Subsequent `/access` dials (tunnel reconnects) present the session token instead of an access-token-derived HMAC credential:

```
Authorization: Bearer <sessionToken>
```

Verification chain on the SDK:

1. Parse JWT header → `kid`.
2. Parse JWT body → `sub`, `jti`, `exp`.
3. Look up `Access` by UID. Missing → reject (`403`, operator → `NeedsReonboarding`).
4. Check `exp`. Past → reject.
5. Look up signing key on this `Access` by `kid`. Missing → reject (key rotated or state lost).
6. Verify signature.
7. Check `jti` against `current_jti` and `previous_jti` (see grace window below).
8. On success: generate `new_jti`, roll state, issue a fresh session token in the response, access-operator rolls the stored value.

HMAC and session-signing-key verification are performed against disjoint keys and never substitute for each other. The access token never flows on the reconnect path.

#### Lost responses and grace window

There is an inherent ordering gap in any HTTP exchange that advances server state: the SDK must persist `current_jti ← new_jti` **before** sending the response, so an intercepted response can't be replayed against unwritten state. But that persist-before-respond rule creates a window where the SDK has moved on while the operator hasn't received the new token — a lost response leaves both sides out of sync.

HTTP gives the SDK no reliable signal that this has happened (TCP ACK is below application semantics) and no push channel to retry. Recovery is always client-initiated.

The **grace window** tolerates these client retries by keeping the previous `jti` valid for a bounded time:

- `jti == current_jti` → accept, roll normally.
- `jti == previous_jti` AND within the grace window (default **60 s**, configurable) → accept (operator missed the previous response), re-issue a fresh token, roll.
- Otherwise → reject.

`previous_jti` is cleared on the next successful roll or when the grace window expires. At most two live `jti`s per `Access` at any time.

The grace window opens a bounded replay window: a captured `jti` is valid for at most the grace duration after the legitimate operator rolls it. This is strictly looser than "zero replay window" but strictly tighter than "no replay tracking at all"; the alternative (no grace window) trades the sliver for frequent spurious `NeedsReonboarding` incidents on lost responses.

Ordering guarantee: the operator holding a valid token always implies the SDK has persisted that token's `jti` (as `current_jti`, or as `previous_jti` within the window).

#### Rejection → `NeedsReonboarding`

If the session token is rejected for any reason other than grace-window acceptance (expired, signing key rotated, Access deleted, replay outside window, SDK state lost), access-operator does **not** fall back to the access token. It stops dialing. The provider reconciler reflects the failure as `Provider.status.conditions[Connected]=False` with `reason=NeedsReonboarding` and flips phase to `Degraded`. Recovery is a new PAR against the same `Provider` with a current access token, which propagates the fresh access token into `AccessGrant.spec.accessToken` and triggers a new initial handshake (as in RFC 012). The initial handshake creates a fresh `Access` with a fresh signing key; the stale one is garbage-collected.

#### Refresh and restart

A stable tunnel can stay open long past the session token's 30 min TTL. For the stored token not to go stale, access-operator periodically dials `/access` with the current session token on a schedule (default every TTL/2 = 15 min). This is the same `/access` endpoint, with the same request shape, as initial handshake and reconnect — the difference is what the SDK does with it.

**SDK behavior on an incoming `/access` dial:**

| Credential presented | SDK has active tunnel for this `Access`? | Response | Tunnel |
|---|---|---|---|
| HMAC call credential (RFC 012) | — (initial handshake) | 200 + body incl. `sessionToken` | upgrade on this connection if `tunnel: required` |
| Session token | No | 200 + fresh `sessionToken` | upgrade on this connection (reconnect) |
| Session token | Yes | 200 + fresh `sessionToken` | no upgrade; existing tunnel untouched (refresh) |

In the refresh case, access-operator rolls the stored value and closes the HTTP connection; the pre-existing tunnel on its own TCP connection keeps carrying operator traffic without interruption.

**access-operator restart.** The TCP connection holding the tunnel dies when access-operator exits; the SDK sees the tunnel drop. When access-operator comes back (leader failover or Pod restart) it reads `AccessGrant.status.sessionToken` from etcd and dials `/access` with it — the reconnect path. If the stored token's `exp` is in the future, reconnect succeeds and the tunnel re-establishes. If the stored token has expired (extended outage), `NeedsReonboarding`. With TTL = 30 min and refresh = 15 min, the stored token always has 15–30 min of remaining lifetime at any given time; brief restarts recover without re-onboarding, extended outages (> 30 min from the last refresh) don't.

**access SDK restart.** The TCP peer on the user side goes away; access-operator sees the tunnel drop. When the SDK Pod comes back it loads per-`Access` signing keys, `jti` state, and all CRD state from persisted Secrets. access-operator reconnects and presents its stored session token; the SDK verifies against the persisted signing key and `current_jti`/`previous_jti`. Grace window still applies if the reconnect is within 60 s of the last roll. Tunnel re-establishes. No duration limit on SDK downtime as long as the operator's stored session token remains within `exp`.

**Both sides restart simultaneously.** Same analysis as each individually: whoever comes up first finds the other gone; when both are up, the reconnect path runs. Stored state on both sides is sufficient to resume.

### Reconnect binding

The access SDK's local HTTPS proxy endpoint URL is **stable per `Access` across reconnects**. The kubeconfig's `proxy-url` written at initial handshake does not change over the life of the `Access`.

On a reconnect dial, the SDK:

1. Identifies the target `Access` from the connection-level identifier — see [Local proxy endpoint URL](#local-proxy-endpoint-url) below.
2. Verifies the JWT (chain above). `sub` claim must match the identified `Access` UID; mismatch → reject.
3. On tunnel upgrade, binds the newly-established tunnel as the backend for the `Access`'s local proxy endpoint. Operator traffic flowing through `proxy-url` is now served by the new tunnel.

If the `Access` named by the connection and/or `sub` no longer exists on the user-side cluster (deleted since token issuance) → reject → `NeedsReonboarding`.

The JWT `sub` claim is the **authoritative identifier** for verification; the connection-level identifier (Proxy-Authorization, SNI, or loopback) is routing metadata that lets the SDK dispatch the CONNECT to the right local endpoint. Both must agree.

### Local proxy endpoint URL

#### Why path-based routing doesn't work

`proxy-url` in a kubeconfig drives client-go's HTTP CONNECT proxy. For an HTTPS target (kcp), the client dials `proxy-url`'s host:port, performs a TLS handshake with the proxy (SNI = proxy hostname, not path), and sends:

```
CONNECT kcp.pm.example:443 HTTP/1.1
Host: kcp.pm.example:443
```

The CONNECT request-line carries the kcp target, not the proxy URL's path or query. Those are never transmitted to the SDK. So the SDK has to identify the `Access` from something that *does* reach it: TLS SNI, the `Proxy-Authorization` header, or similar.

#### Default scheme: Proxy-Authorization routing

The default this RFC prescribes:

- The kubeconfig's `proxy-url` is `https://<access-uid>:@access-tunnel.svc:443` (trailing colon: empty password).
- Go's HTTP transport extracts userinfo from the proxy URL and sends it on CONNECT as `Proxy-Authorization: Basic base64("<access-uid>:")`.
- The access SDK decodes the header, extracts the username = `Access` UID, and routes the CONNECT to that `Access`'s tunnel backend.

Deployment shape for this default:

- One central access SDK Deployment + one ClusterIP Service `access-tunnel.svc`.
- One TLS cert for `access-tunnel.svc` (no wildcard).
- No wildcard DNS.

**Cross-check against JWT.** On reconnect, the `Access` UID presented via Proxy-Authorization must equal the `sub` claim in the JWT. Mismatch → reject. The JWT remains the authoritative authentication; Proxy-Authorization is a routing hint.

**Trade-off.** This is a semantic abuse of Basic auth: the SDK uses it as an identifier channel, not for authentication. `Access` UIDs appear in Proxy-Authorization headers on the wire (TLS-protected) and in SDK logs; UIDs are not secret. Some HTTP proxies strip or normalize `Proxy-Authorization`, but the connection from operator Pods to the access SDK is in-cluster and under deployer control, so this is a theoretical concern.

#### Alternative deployment patterns

Two alternatives are documented for environments where the default doesn't fit:

**SNI-based routing (wildcard DNS).** `proxy-url: https://<access-uid>.access-tunnel.svc:443`. The SDK routes by SNI on the outer TLS handshake. Needs wildcard DNS (`*.access-tunnel.svc`) and a wildcard TLS cert covering it. Cleaner semantics (SNI is a legitimate routing signal); heavier deployment.

**Sidecar-per-operator (loopback).** `proxy-url: https://localhost:9999`. Each operator Pod runs an access SDK sidecar handling one `Access` at a fixed loopback address. No routing needed. Multiplies SDK instances; fits strict-isolation or single-`Access` deployments but contradicts the "one SDK, many Accesses" model of RFC 012.

#### Invariants regardless of scheme

1. The local proxy endpoint URL is **stable per `Access` across reconnects**.
2. The URL is **reachable from operator Pods** on the user-side cluster.
3. The **JWT `sub` claim remains the authoritative identifier** for verification; the connection-level identifier (Proxy-Auth / SNI / loopback) is routing metadata only.

### `AccessGrant` additions

```go
type AccessGrantStatus struct {
    // ... baseline fields ...

    // Tunnel-mode reconnect credential.
    SessionToken          string       `json:"sessionToken,omitempty"`
    SessionTokenExpiresAt *metav1.Time `json:"sessionTokenExpiresAt,omitempty"`

    // Raised when session-token verification fails and no fallback is available.
    // Reflected as Provider.status.conditions[Connected]=False with reason=NeedsReonboarding.
    // Recovery: new PAR against the same Provider.
}
```

When the stored session token is rejected and access-operator stops dialing, the provider reconciler surfaces this on the `Provider` as `conditions[Connected]=False` with `reason=NeedsReonboarding` (and `phase=Degraded`).

Per-`Access` session-signing key material and `jti` tracking state (`current_jti`, `previous_jti`) live in access SDK state (likely a Secret owned by the `Access`), not on any PM-side CRD.

### `AccessIdentity` additions

```go
type AccessIdentitySpec struct {
    // ... baseline fields ...
    Tunnel TunnelPolicy `json:"tunnel,omitempty"`  // Allow (default) | Deny
}
```

No crypto additions on `AccessIdentity`: the HMAC secret for initial handshakes is the only crypto the Identity holds. Session-signing keys live on the `Access` (see [Reconnect-authentication plane](#reconnect-authentication-plane)).

Nothing about the HMAC verification or baseline `/access` handling changes. `spec.accept` continues to govern the **initial handshake only**; reconnects with a valid session token are accepted without consulting `spec.accept` or the pending queue.

### Protocol versioning

Tunnel mode is served under the baseline ALPN token `pm-access/1`. The additions are purely additive: new request-body fields (`tunnel`), new response-body fields (`sessionToken`, `sessionTokenExpiresAt`), new `Authorization` header shape on reconnect dials. Unknown fields are ignored under baseline-RFC-012 handling.

Capability detection via field presence:

- A baseline (RFC-012-only) access SDK ignores the `tunnel` request field and returns a baseline 200 (ack, empty body). access-operator sees no `sessionToken` in the response — a clean signal of "this SDK doesn't support tunnel mode." If PM required tunnel, this surfaces as a connection failure on the `Provider`.
- A baseline access-operator doesn't send the `tunnel` field; the SDK, even if tunnel-capable, treats the call as a plain RFC-012 handshake.

`pm-access/2` is reserved for a future protocol break that actually changes shape or semantics. Until that happens, everything lives in `pm-access/1`.

## Flow

### Phase 2 under tunnel mode

```
access-operator  ──(4) picks up AccessGrant (Pending)
access-operator  ──(4) mints SA token via kcp TokenRequest
access-operator  ──(4) POST /access (ALPN: pm-access/1) ──────────────►  access SDK
                       Authorization: Bearer <JWT signed with access token HMAC>
                       [JWT: sub=tokenID, body_sha256, jti, iat, exp;
                        see RFC 012 "Call credential"]
                       body: { kcpURL, kcpCABundle, saToken, tunnel: "required" }
                 ◄──── 200 (ack + { sessionToken, sessionTokenExpiresAt }) ─
                       [connection upgraded to persistent tunnel]
access-operator      ──(5) marks AccessGrant Active
                             stores sessionToken on AccessGrant.status
access SDK           ──(5) creates Access, generates per-Access signing key,
                           writes kubeconfig (proxy-url carries Access UID in userinfo)
                           from request body, binds Access to the tunnel
provider reconciler  ──(5) sets Provider.status.conditions[Connected]=True; phase → Ready
operator             ──(6) uses kubeconfig → CONNECT via proxy (Proxy-Auth: Access UID)
                           → tunnel → SNI router → kcp
```

### Reconnect

```
                 [tunnel drops: network blip, pod restart, leader failover...]
access-operator  ──(R) CONNECT proxy ──────────────────────────────────►  access SDK
                       (TCP+TLS to access-tunnel.svc; access-operator initiates
                        the exchange on behalf of its own /access reconnect, not
                        on behalf of operator Pods — this diagram elides that
                        internal step; see "Reconnect binding")

                       POST /access (ALPN: pm-access/1)
                       Proxy-Authorization: Basic base64("<access-uid>:")
                         [SDK: decode header → route to Access by UID]
                       Authorization: Bearer <sessionToken from AccessGrant.status>
                         [SDK: parse JWT, verify signature against Access's
                          signing key, check sub == routed Access UID,
                          check jti against current_jti / previous_jti]
                       body: { kcpURL, kcpCABundle, saToken, tunnel: "required" }
                 ◄──── 200 (ack + { sessionToken: <fresh>, sessionTokenExpiresAt })
                       [SDK rolled jti and persisted before responding]
                       [tunnel re-established; backend of Access's local
                        proxy endpoint now points at the new tunnel]
access-operator      stores fresh sessionToken on AccessGrant.status
```

The reconnect dial is identical to the initial dial except that:

- the Authorization header carries the session token (not an access-token-derived credential),
- the SA token in the body is a fresh one (access-operator always delivers a fresh SA token on every dial),
- no new `Access` or kubeconfig Secret is created; the existing ones are reused.

If the session token is rejected (expired, signing key rotated, access SDK state loss), access-operator does **not** fall back to the access token. It stops dialing. The provider reconciler reflects the failure as `Provider.status.conditions[Connected]=False` with `reason=NeedsReonboarding` and flips phase to `Degraded`. Recovery is a fresh PAR against the same `Provider` with a current access token, which propagates the fresh value into `AccessGrant.spec.accessToken` and triggers a new initial handshake (as in RFC 012).

## Leader election

access-operator uses leader election for tunnel management only. One access-operator replica owns a given tunnel at a time; on leader failover the new leader dials the tunnel using the stored session token, picking up where the old leader left off. Credential delivery (the initial handshake path) is stateless and does not require leader election.

## Rotation and revocation

### Credential planes

- **access token**: initial-handshake credential. Baseline semantics (RFC 012). **Identity-scoped** (one HMAC secret per `AccessIdentity`).
- **session token**: reconnect credential. Lives in `AccessGrant.status`, rolled on each dial. **Per-`Access` scoped** (one signing key per `Access`).

The two planes are verified against disjoint keys. Rotating one does not substitute for the other on verification. The scope split is deliberate: rotating one `Access`'s session key must not affect sibling `Access`es under the same `AccessIdentity`.

### Rotation operations (user-side)

In addition to the RFC 012 operations:

- **Rotate `AccessIdentity` HMAC secret** (new access token) — Identity-scoped. Blocks future initial handshakes under this Identity. **Zero effect on running tunnels** in any mode: reconnects use session tokens, which are independent of the HMAC.
- **Rotate one `Access`'s session-signing key** — Access-scoped. Invalidates the session tokens outstanding for that one `Access`. That tunnel enters `NeedsReonboarding` on next drop (or immediately, if the SDK proactively tears the tunnel down on key rotation — deployment choice). Sibling `Access`es under the same `AccessIdentity` are unaffected.
- **Delete one `Access`** — Access-scoped. The tunnel drops and can't reconnect; the signing key is garbage-collected with the Access. Siblings unaffected.
- **Delete `AccessIdentity`** — Identity-scoped. Blocks future initial handshakes under this Identity. Existing `Access`es keep working on their own per-`Access` signing keys and delivered SA tokens until individually deleted or revoked PM-side.

### Incident response: leaked access token

Rotate the HMAC secret on the `AccessIdentity`. The attacker cannot open a new tunnel under this Identity. Existing tunnels are unaffected because the access token is not valid currency on the reconnect path.

If there is reason to believe the attacker *already* opened a tunnel before the leak was detected, additionally rotate the session-signing key on **each suspect `Access`** under that Identity. If unsure which, rotate all of them. Each targeted tunnel enters `NeedsReonboarding`; the Owner creates fresh PARs against the same `Provider` with the new access token, and the legitimate tunnels are restored. The attacker's tunnel cannot reconnect (they have neither a valid access token nor a valid session token for the compromised `Access`), and sibling `Access`es that weren't touched keep running without disruption.

The per-`Access` scope of the session-key lever gives surgical control: in a many-Customer deployment, you can isolate the response to just the connection(s) suspected to be compromised rather than kicking every tunnel under the shared Identity.

### Teardown

Unchanged from RFC 012: deleting the `Provider` tears down the Provider Workspace, taking the `AccessGrant` with it. access-operator observes the deletion, closes the tunnel, and stops. In tunnel mode, the operator-side 401 chain additionally propagates through the tunnel and triggers tunnel closure as a redundancy.

## Security considerations

- **Session-signing key is per-`Access`.** A compromise of one `Access`'s signing key affects only that one `Access`'s outstanding session tokens, not siblings under the same `AccessIdentity`. Rotation is Access-scoped; see [Rotation operations](#rotation-operations-user-side).
- **Session tokens are short-lived.** Default TTL **30 min**; configurable. Shorter TTLs reduce the window a captured token can be replayed, at the cost of more frequent reconnects to roll the token (bounded by a scheduled refresh).
- **Scheduled refresh.** access-operator dials `/access` on a timer to keep the stored token fresh even without organic tunnel drops. Default **15 min** (= TTL/2). Soft constraint: `refresh ≤ TTL/2` so the operator always holds a non-expired token even after one missed refresh. Bounds the recovery time after a session-signing-key rotation to one refresh interval.
- **Replay tracking with grace window.** The SDK tracks `current_jti` per `Access` and rejects stale `jti`s. A bounded grace window (default **60 s**, configurable) accepts `previous_jti` once, to tolerate lost responses. The grace window allows at most one replay of a captured token during the window after the legitimate operator rolls — strictly tighter than no tracking, strictly looser than zero-replay; the alternative (no grace window) trades the sliver for frequent spurious `NeedsReonboarding` incidents on lost responses.
- **No fallback is a feature.** Rejecting session tokens without falling back to the access token is deliberate: it prevents an attacker who leaked the access token after a tunnel was established from using it to re-authenticate. The cost is that session-token issues require explicit re-onboarding (a new PAR). This is acceptable because session-token issues should be rare (short-lived, auto-refreshed, signed by a stable key).
- **Proxy-Authorization carries `Access` UID.** Under the default routing scheme, the `Access` UID appears in the `Proxy-Authorization: Basic` header on CONNECT, inside the user-side-cluster TLS, and in SDK logs. UIDs are not secret; the JWT `sub` remains the authoritative identifier.
- **SNI router sees SNI hostnames.** The SNI router does not terminate TLS. It can observe which SNI hostname each inner connection targets; this is not sensitive. kcp's front-proxy / shards perform TLS termination and client authentication.
- **PM-side cluster compromise** remains out of scope, same as RFC 012.

## Operations

### Status surface

Tunnel state surfaces on the `Provider` using the baseline status convention from RFC 012 — `status.phase` + `status.conditions`, with phase computed from conditions. RFC 013 adds reasons to the existing `Connected` condition:

- `Connected=True` — tunnel open and traffic flowing.
- `Connected=False` reasons introduced by tunnel mode: `Reconnecting` (tunnel dropped, access-operator is dialing), `TunnelDown`, `SessionExpired`, `NeedsReonboarding`.

Phase flips to `Degraded` on any `Connected=False` after the Provider was `Ready` at least once.

`AccessGrant.status` carries per-reconciler detail for operator debugging (reconnect counters, last error, etc.); Customers consume the `Provider`'s condition + phase.

### Metrics

Recommended access-operator metrics:

- `tunnel_up{provider, access_grant}` — gauge (1/0).
- `tunnel_reconnects_total{provider, access_grant}` — counter.
- `access_dials_total{provider, access_grant, outcome}` — counter (outcome: `200`, `202`, `403`, `tls_fail`, `session_expired`).
- `session_token_age_seconds{provider, access_grant}` — gauge.

## Deployment considerations

- **access SDK HA.** Single-active via leader election: one access SDK replica owns all tunnels at a time; standbys take over on failover. Simplest, bounded by one replica's tunnel capacity, which is well within mid-size deployments. Horizontal scaling (consistent-hashed sharding across replicas) and per-operator-Pod sidecars are documented as follow-ups in [Future work](#future-work); neither is required for MVP.
- **PM's SNI router.** New kcp component; needs its own design sketch (likely a separate ADR or kcp-side proposal). This RFC assumes it exists.
- **Tunnel keepalive.** TCP keepalives or app-layer pings to detect drops promptly. Default values TBD; implementation should allow tuning.
- **Egress restrictions on the user-side cluster.** The access SDK needs outbound-only HTTPS. It does not initiate connections to PM's kcp — all PM-side traffic arrives via the inbound tunnel.

## Future work

- **`Access` status for tunnel health** on the user-side, to let Customer tooling see connection state without round-tripping to PM.
- **Multi-tunnel per `Access`** (bandwidth / HA), if single-tunnel throughput becomes a limit.
- **Horizontal scaling of access SDK** via consistent-hashed sharding across replicas (each replica owns a shard of tunnels keyed by `Access` UID). Preserves the per-`Access` session-key state model unchanged.
- **Sidecar-per-operator-Pod access SDK**, for strict-isolation deployments where each operator Pod runs its own SDK instance holding exactly one `Access`.
- **SNI router — external dependency on kcp.** RFC 013 depends on kcp growing an L4 SNI router upstream of front-proxy TLS termination. Design and review happen in the kcp project; RFC 013 implementation is blocked on that landing. Interim option: a PM-operated standalone L4 router component that fakes the role until kcp absorbs it.
