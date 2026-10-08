# Notes v5: RFC 011 provider onboarding flow

Working notes for [011](./011-provider-onboarding-flow-improvements.md). Not the RFC itself. v5 builds on the [v4 notes](./xxx-notes-011-provider-onboarding-flow-improvements-v4.md) and removes most of its machinery by reversing two of its assumptions:

- **The Provider-side never holds kcp credentials.** The connector becomes an authenticating HTTP proxy for kcp. It adds the SA token itself, so no access token, kcp CA bundle or token rotation ever crosses to the Provider-side.
- **The Provider's edge is the Provider's responsibility**, mirroring v4's "the owner's workspace is the owner's responsibility". The inner mTLS, server-key pinning and owner CA lose their purpose. A connection is authenticated by a PM-generated secret that rotates automatically.

What's left is: one registration call, one WebSocket carrying HTTP/2, and one secret that nobody has to touch. See "Changes from v4" and "Comparison".

## Terminology

- **PM-side**: the PM instance (kcp + PM operator). May be behind NAT; dials out.
- **Provider-side**: the service provider (controllers + portal in its runtime cluster). Serves the connection API and reconciles on the PM-side.
- **Owner**: whoever controls the workspace that holds the `Provider` object and its connection Secret on the PM-side.
- **Connection**: the relationship between one `Provider` object on the PM-side and one connection record on the Provider-side, identified by `connectionID`. Provider-side state belongs to its connection.
- **Onboarding token**: a single-use, short-lived token from the Provider's portal, with the Provider's base URL inside. Unchanged from v2.
- **Connection secret**: a high-entropy secret the PM-side generates and keeps in the connection Secret. The Provider-side stores only its hash. It authenticates the PM-side on every call after registration. It grants **no** kcp access.
- **Session**: one WebSocket from the connector to the Provider's base URL, authenticated with the connection secret. It carries HTTP/2, with the Provider-side as the HTTP client and the connector as the HTTP server.
- **Connector**: the PM-side component that opens sessions and acts as an **authenticating reverse proxy** to kcp for requests arriving over them. Where it runs is an open item.
- **`ProviderOnboardingRequest`**: as in v4, plus rebind (see "Recovery: rebind").

## Problem

Unchanged from v2: the Provider-side must reconcile on any number of PM instances, the PM-side may be behind NAT and opens every connection, the user gets a token from the Provider's portal, and both sides must recover from restarts and outages without manual steps.

**What v5 changes is where kcp credentials live.** v1 to v4 all delivered kcp credentials (a kubeconfig, then SA tokens and a CA bundle) to the Provider-side. Most of the complexity in v2 to v4 follows from that:
- the credentials must not be visible at a TLS-terminating hop (finding 10), which led to v4's inner mTLS;
- the Provider-side must trust kcp's CA, and whoever can change that bundle can present a fake kcp (finding 7);
- the credentials must be refreshed, and deleting the SA doesn't revoke them in PM's kcp (finding 2);
- the PM-side needs a strong, long-lived identity to push them, which led to v4's owner CA and its rotation procedure.

**Constraints** (from v4, still valid):
- **Assume as little as possible about the Provider's infrastructure.** Any HTTPS endpoint that supports WebSockets must do. No TLS passthrough, no client-certificate-verifying ingress.
- **Assume nothing about how PM operates kcp's PKI.** Rotating kcp's CAs, adding shards or changing PM's edge must not involve Providers.

## Trust boundaries

Two symmetric statements:

- **The owner's workspace is the owner's responsibility** (from v4). Whoever can read the connection Secret can act as the owner's connection.
- **The Provider's edge is the Provider's responsibility** (new). A Provider already trusts its ingress, load balancer or CDN with all of its customers' traffic. v5 doesn't defend a connection against the Provider's own edge.

v5 makes sure that:
- **nothing outside these two boundaries** can take over a connection: not a stolen onboarding token, not a compromised portal account on its own (see "Recovery: rebind"), not an observer of logged requests;
- **no kcp credential ever leaves the PM-side**, so a compromised Provider-side database or edge yields nothing that works against kcp once its sessions are gone;
- **revocation is immediate**: when the connector stops proxying, the Provider-side has nothing left to use.

**In tunnel mode, trusting kcp is trusting the connection.** The connector decides where every request goes, so the Provider-side can never verify kcp more strongly than it verifies the session. v2 to v4 treated "kcp's CA bundle" as a separate concern; v5 drops it. The Provider-side trusts responses that arrive over an authenticated session, for that connection only.

## Core idea

```
Provider-side controllers ── HTTP request (real kcp URL) ──▶ SDK RoundTripper
                                                                   ║
             ┌─────────────── one WebSocket (outbound from PM) ────╜
             ▼               HTTP/2 inside: Provider = client, connector = server
          connector ── strips auth headers, checks host, adds SA token ── TLS ──▶ kcp
```

- **The connector dials out**, as in every version: a WebSocket to `GET /v1/connections/{id}/session` on the base URL, with `Authorization: Bearer <connection secret>`. The Provider's ingress terminates TLS and routes it like any WebSocket.
- **Inside the WebSocket runs HTTP/2, with the roles reversed.** The connector serves HTTP/2 on the WebSocket connection; the Provider-side is the client. HTTP/2 provides multiplexing, flow control and long-lived streams (watches), so no separate tunnel library is needed.
- **The connector is an authenticating reverse proxy.** For each request it checks the target host against the allow-list, removes any credentials the Provider-side sent, adds a short-lived SA token for that connection, and forwards the request to the real kcp URL over TLS verified with PM's own trust.
- **The Provider-side keeps the real kcp URLs** (front-proxy, and the shard virtual-workspace URLs from endpoint slices). An SDK `RoundTripper` sends every request over the connection's session instead of the network.

## Provider-side prerequisites

- **An HTTPS endpoint at the base URL from the token that supports WebSocket upgrades** and long-lived connections: idle timeouts above the session ping interval (e.g. > 60 s), and a maximum connection age the connector can reconnect after. A WAF must pass binary frames after the upgrade.
- **The connection API** (registration, status, secret rotation, rebind, delete) and a session endpoint. Both should ship in the Provider SDK.
- **State keyed by `connectionID`** (see "Provider-side: consuming N connections").
- **Owner notifications and controls** in the portal: notify the token owner of every registration, rotation and rebind, and let them freeze or delete connections.

Not required: TLS passthrough, client-certificate handling, a CA, a server key, cert-manager, or any knowledge of kcp's tokens, keys or CAs.

## Flow overview

1. **Get a token.** As in v2.
2. **Create the request.** As in v2.
3. **Create or reuse the `Provider`.** As in v4: an already registered `Provider` is refused, except for rebind.
4. **Register.** The request generates the connection secret, writes it to the connection Secret, and calls `POST /v1/connections` with the onboarding token, the addressing and the **hash** of the secret. It gets back the `connectionID`. No kcp credential and no secret crosses the wire.
5. **Open a session.** The connector opens the WebSocket with the connection secret and serves HTTP/2 over it.
6. **Confirm kcp is reachable.** The request polls `GET /v1/connections/{id}` until the Provider-side reports that it reached kcp through the session. Only then is the request `Ready`.
7. **Run.** The Provider-side reconciles through the session. There's no credential delivery and no credential refresh on the Provider-side.
8. **Rotate the secret**, automatically and periodically (see "Connection secret rotation").
9. **Offboard.** Deleting the `Provider` closes its sessions, which ends kcp access at once, then calls `DELETE`.

Steps 5 to 8 handle every restart and outage, of any length: the connection secret doesn't expire, and the connector mints SA tokens for itself.

## Design

### Onboarding token

Unchanged from v4: `pm1.<base64url(Provider base URL)>.<tokenID>.<tokenSecret>`, single-use, short-lived, stored in a Secret, with the URL inside so nothing is sent to the wrong host. The portal never puts tokens in URLs and hands them over as a `kubectl create secret` command, not a manifest.

### Connection API (on the base URL, plain HTTPS)

There's no inner TLS any more, so the connection API is just HTTPS on the base URL. The session is the only long-lived connection.

```
POST /v1/connections
Authorization: Bearer <tokenID>.<tokenSecret>
{
  "idempotencyKey": "<ProviderOnboardingRequest UID>",
  "clusterID": "<logical cluster name of the provider workspace>",
  "frontProxyURL": "https://...",
  "secretHash": "sha256:<hex>"
}
→ 201 { "connectionID": "..." }

GET /v1/connections/{id}
Authorization: Bearer <connection secret>
→ 200 { "kcpReachable": true, "lastError": "", "sessions": 1,
        "pendingSecret": false, "pendingRebind": null }

PUT /v1/connections/{id}/secret                  # rotation (see below)
Authorization: Bearer <current connection secret>
{ "secretHash": "sha256:<hex of the NEW secret>" }
→ 202                                            # pending until the new secret is used

GET /v1/connections/{id}/session                 (WebSocket upgrade; HTTP/2 inside)
Authorization: Bearer <connection secret>

POST /v1/connections/{id}/rebind                 # recovery after a lost secret (see below)
Authorization: Bearer <tokenID>.<tokenSecret>
{ "idempotencyKey": "...", "secretHash": "sha256:<hex>" }
→ 202 { "completesAt": "..." }

DELETE /v1/connections/{id}
Authorization: Bearer <connection secret>
→ 204
```

- **The PM-side generates the secret; only its hash is sent.** The response to `POST` contains nothing secret. Replaying a logged registration gains nothing, and a retry is accepted only with the same `secretHash` (finding 9 stays closed).
- **Secret format**: a prefix plus 256 random bits, e.g. `pmc1_<base64url>`. The prefix lets secret scanners (GitHub, GitLab, gitleaks) recognise leaked secrets. High entropy means a plain SHA-256 is enough to store it; no slow password hash.
- **Idempotency**, the bounded retry window after token expiry, immutable addressing and no APIExport details at registration: as in v4.
- **Bearer credentials are fine here** because of the trust boundary: the only hop that sees the secret is the Provider's own edge. A stolen secret also grants no kcp access (see "What each credential exposes").

### Sessions: HTTP/2 over a WebSocket

- **Provider-side**: the session handler checks the bearer secret against connection `{id}`, wraps the WebSocket as a `net.Conn`, and creates an HTTP/2 client connection on it (in Go: `http2.Transport.NewClientConn(conn)`, with `AllowHTTP`). The connection is registered under `{id}`. Deleting the connection or rotating the secret closes it.
- **PM-side**: the connector opens the WebSocket, wraps it as a `net.Conn`, and serves HTTP/2 on it (`http2.Server.ServeConn`). Its handler is the reverse proxy below.
- **Several sessions per connection** are allowed: for connector HA, for spreading load, and to raise the stream limit (HTTP/2 caps concurrent streams per connection; Go's server default is 250, and every watch is a stream). The Provider-side round-robins requests across a connection's sessions.
- **Liveness**: WebSocket pings at the outer level, so ingresses see traffic, and HTTP/2 PING frames with read deadlines inside, so dead sessions are detected and replaced.
- **Why not a dial proxy** (remotedialer, Konnectivity, as in v1 to v4): a dial proxy carries opaque TLS from the Provider-side to kcp, so the Provider-side has to hold kcp credentials and trust kcp's CA. An HTTP proxy lets the connector own both. Go has both HTTP/2 halves in the standard extended library, so the "which tunnel library" open item goes away.

### The connector as an authenticating proxy

For every request arriving over a session of connection `{id}`:

1. **Check the target.** The request's host must be on the allow-list: the front-proxy and the `spec.virtualWorkspaceURL` host of every kcp `Shard`, built only from PM-side data. Resolve once, check every address, dial those exact addresses; refuse link-local, multicast and cloud-metadata addresses. Unchanged from v2.
2. **Strip credentials and identity headers**: `Authorization`, `Impersonate-*`, `Proxy-*`, cookies and hop-by-hop headers. Otherwise the Provider-side could ask kcp to act as someone else, using the connector's token.
3. **Add the SA token** for connection `{id}`: a bound token from TokenRequest, minted by the connector, cached in memory, refreshed at half its TTL, never written anywhere and never sent to the Provider-side.
4. **Forward** to the real URL over TLS, verified against the CA bundle PM's own components use for kcp. Rotating kcp's CAs or adding shards needs no coordination: if PM trusts its kcp, the connector does.
5. **Log and limit** per connection: method, path, status, duration; rate limits and concurrent-stream limits per connection.

- **RBAC is unchanged**: kcp sees the connection's SA, with the RBAC the Provider controller created (RFC 006). The connector doesn't widen it.
- **Upgrade requests** (`exec`, `attach`, `port-forward`) are refused by default; a Provider reconciling an APIExport doesn't need them. See "Open items".
- **Build on kcp's front-proxy code** where possible. It's already an authenticating reverse proxy for kcp, with streaming and watch handling.

### Provider-side: consuming N connections

None of the earlier notes covered this, though serving any number of PM instances was the first problem in the RFC stub.

- **One base config per connection.** The SDK gives each connection a `RoundTripper` bound to its sessions:

  ```go
  cfg := &rest.Config{
      Host:      conn.FrontProxyURL,              // real URL; the connector routes on it
      Transport: sdk.SessionRoundTripper(conn.ID), // no TLS options: client-go refuses both
  }
  ```

  multicluster-runtime derives per-endpoint configs (shard virtual-workspace URLs) by changing `Host`; they inherit the `Transport`, so every request for that connection goes over its sessions.
- **The set of connections changes at runtime.** Registrations, deletions and rebinds add or remove connections while the Provider runs. The SDK exposes them as a watchable set, and starts or stops a multicluster-runtime provider (one per connection, over that connection's endpoint slice) accordingly.
- **Cluster names are prefixed with `connectionID`.** Logical cluster names are only unique within one kcp. A Provider serving several PM instances keys clusters, and all state, by `(connectionID, clusterName)`. Composing the per-connection providers with a prefix is what multicluster-runtime's `multi` provider does, if it fits (to verify).
- **No credentials to manage per connection.** Compared to v4, there's no token, CA bundle or client rebuild on rotation; a connection is just a session set.

### Connection secret rotation

Fully automatic, with no owner involvement:

1. The connector generates a new secret and writes it to the connection Secret as `pendingSecret` **before** sending anything, so a crash can't lose it.
2. It calls `PUT /secret` with the new hash, authenticated with the current secret. The Provider-side stores it as pending; both secrets are accepted.
3. The connector authenticates once with the new secret (e.g. `GET`). The Provider-side **promotes** it and invalidates the old one.
4. The connector moves `pendingSecret` to `secret`, and replaces sessions opened with the old secret. The Provider-side closes any left after a short grace period.

- **Lost response or crash**: on restart the connector tries `pendingSecret` first, then `secret`, and resumes at the matching step. A pending hash that's never used expires (e.g. after 24h).
- **When**: periodically (e.g. every 30 days), on demand (an annotation on the `Provider`), and right after any suspected leak.
- **No owner-supplied key material.** v4's manual steps came from letting owners supply their own CA, which needed the active/desired split. v5 drops that option: the secret is always generated and owned by the connector.

### Recovery: rebind

v4 had no way back after a lost owner CA without a backup: a new connection, with no Provider-side state carried over. v5 adds a recovery path that an outsider can't use quietly, modelled on account recovery with a waiting period:

1. The owner creates a `ProviderOnboardingRequest` for a `Provider` that is registered but whose connection Secret is gone. The request generates a new secret and calls `POST /rebind` with a fresh onboarding token and the new hash.
2. The Provider-side accepts it only if the token's owner is the connection's owner. The connection record already fixes the addressing, so the request sends none. It then starts a **waiting period** (e.g. 72h) and notifies the owner through every channel.
3. **Any successful authentication with the current secret during the waiting period cancels the rebind.** The connector authenticates regularly (sessions, periodic `GET`), so a rebind against a connection whose connector is still alive never completes. A rebind only succeeds when the current secret really is gone.
4. After the waiting period, the Provider-side swaps the hash. The connection keeps its `connectionID` and all its Provider-side state.

- **A stolen onboarding token or a compromised portal account** can start a rebind, but the live connector cancels it automatically, and the owner is notified. Unlike v2 and v3, a token alone never takes over a connection.
- **A leaked secret is a rotation, not a rebind.** If the connector can still authenticate, it rotates.
- **Remaining gap**: a thief with the secret who rotates it locks the owner out, and can then also cancel the owner's rebinds. That's owner compromise under the trust boundary; the owner freezes or deletes the connection in the portal. See "Open items".

### What each credential exposes

| Credential | Lifetime | Stored at | If stolen |
|---|---|---|---|
| Onboarding token | minutes, single-use | PM-side Secret until spent; hash on the Provider-side | The thief can register **their own** new connection under the owner's portal account (billing, quotas), or start a rebind that the live connector cancels. No access to existing connections or their state. The owner is notified. |
| Provider portal account | long-lived | Provider-side | Same as issuing tokens, plus whatever the portal allows (deleting or freezing connections). Can't take over a live connection. |
| Connection secret | long-lived, rotated automatically | Owner's workspace (connection Secret); hash on the Provider-side | **Owner compromise.** The thief can open sessions as the PM-side and answer the Provider-side's requests for that connection with fake data, rotate the secret to lock the owner out, or delete the connection. **No kcp access**: the secret authenticates the PM-side to the Provider, not the other way round. |
| SA token | minutes to hours | Connector memory only | Access to the provider workspace within RFC 006 RBAC until it expires. Only reachable by compromising the connector; never sent to the Provider-side. |
| Provider-side runtime | — | Provider-side | Whatever its open sessions allow: kcp requests within RBAC, while the sessions last. Nothing persistent: there's no kcp credential to exfiltrate. Ends when the PM-side closes the sessions. |
| Provider's edge (ingress, CDN) | — | Provider's infrastructure | Sees the onboarding token (spent), the connection secret, and the reconcile traffic. Within the Provider's trust boundary, like the rest of its customers' traffic. |

### Threat model: what moved

Compared to v4's threat model:
- **The connector** is still the highest-impact target: it holds every connection secret it serves and mints SA tokens. Unchanged in kind; the owner CA keys are gone, but it's now also in the data path.
- **The owner's workspace**: same exposure as v4, with a secret instead of a CA key. Secret scanning catches dumps and commits that v4's PEM keys would only be caught by generic rules.
- **A compromised Provider-side** loses most of its value: there's no kcp token to take away. v1 to v4 all left a usable kcp token behind.
- **The Provider's edge** moves out of the threat model by definition. A Provider that doesn't trust its own edge should use the JWT assertion variant (see "Open items") and accept that its edge still sees reconcile traffic.

### Resources

```yaml
apiVersion: providers.platform-mesh.io/v1alpha1
kind: ProviderOnboardingRequest
metadata:
  name: my-provider-2026-10-08
spec:
  tokenSecretRef: { name: my-provider-token, key: token }
  provider:
    name: my-provider                                # created if missing; refused if registered, unless rebind applies
    spec:
      remote:
        caBundle: ""                                 # optional, for a private CA on the Provider's base URL
status:
  phase: Ready                                       # WaitingForProvider → Registering → VerifyingKcp → Ready | Failed
                                                     # or: Rebinding → Ready | Failed
  providerRef: { name: my-provider }
  connectionID: "..."
  rebindCompletesAt: ""                              # only while Rebinding
---
apiVersion: v1
kind: Secret
metadata:
  name: my-provider-connection                       # owned by the Provider; readable only by the connector
type: Opaque
stringData:
  url: https://provider.example.com/pm               # base URL from the token
  connectionID: "..."
  secret: "pmc1_..."
  pendingSecret: ""                                  # only during a rotation or rebind
```

**How the request treats `provider.name`**:

| The `Provider`... | The request... |
|---|---|
| doesn't exist | creates it, waits for the workspace and SA, then registers |
| exists, not registered | reuses it and registers |
| exists and is registered, connection Secret present | **refuses**. The connector rotates the secret; there's nothing to re-onboard |
| exists and is registered, connection Secret missing | **rebinds** using `Provider.status.connection` (URL and `connectionID`), and waits out the rebind period |
| exists, but isn't a remote `Provider` | refuses |

- `Provider.status.connection` mirrors `url` and `connectionID`, which is what makes rebind possible after the Secret is lost.
- The `Provider`'s conditions: `Registered`, `KcpReachable`, `SessionConnected`, `SecretCurrent` (False while a rotation is pending), `Rebinding`.
- **No static kubeconfig for remote `Provider`s**, as in v2. Nothing on the PM-side is issued to the Provider-side except the sessions themselves.

### Restart and failure semantics

| Scenario | Behavior |
|---|---|
| PM-side restarts | The connector reads the connection Secret, opens sessions, mints SA tokens for itself. Nothing is pushed. |
| Provider-side restarts | Sessions drop; connectors reconnect. The Provider-side has no credentials to restore. |
| Outage of any length | Nothing expires that the PM-side needs: the secret doesn't expire and SA tokens are minted locally. |
| Ingress drops a session | The connector reconnects. Pings keep idle timeouts from triggering. In-flight requests and watches on that session fail and client-go retries them, as after any apiserver disconnect. |
| PM rotates kcp's CA, adds shards or changes its edge | Nothing to coordinate. The connector uses PM's own trust and watches `Shard`s for the allow-list. |
| `POST` response lost | Retry with the same token, idempotency key and `secretHash`: same `connectionID`. A different hash is refused. |
| kcp unreachable from the connector | `kcpReachable=false` with the error; during registration the request stays in `VerifyingKcp` and fails after a timeout. |
| Rotation response lost, or connector crash mid-rotation | Try `pendingSecret`, then `secret`; resume. An unused pending hash expires. |
| Connection Secret lost | A new request rebinds; the connection and its state survive after the waiting period. |
| Rebind started by someone else | The live connector's next authentication cancels it; the owner is notified. |
| Thief rotates the secret | The legitimate connector can't authenticate and the owner is notified. The owner freezes or deletes the connection in the portal. |
| Connection deleted on the Provider-side | Sessions and calls are refused. `Registered=False` on the `Provider`. Recovery is a new connection. |
| Connector reconnects while an old session is open | Both are valid sessions for the connection; the old one dies by liveness timeout or is closed. No "replace the tunnel" race, because sessions aren't exclusive. |
| Provider-side runs multiple replicas | A session lands on one replica, which then has to send that connection's kcp requests. Either shard reconciliation by connection to the replicas holding its sessions, or route through Lease holders as in v2. More sessions per connection (one per replica, if the ingress spreads them) makes this easier but isn't guaranteed. |

### Offboarding and revocation

- **Deleting the `Provider`** makes the connector close its sessions first. That ends the Provider-side's kcp access immediately, whether or not anything else succeeds. Then a finalizer calls `DELETE` and the Provider controller cleans up the SA and RBAC (RFC 006).
- **Finding 2 doesn't matter for remote `Provider`s any more.** Tokens are never handed out, so there's nothing to revoke after the SA is deleted. It still matters for RFC 006's static kubeconfigs.
- **An unreachable Provider-side doesn't block deletion**, as in v2: access is already gone, `DELETE` is only cleanup.
- **Deleting the connection on the Provider-side** refuses further sessions and calls; the PM-side surfaces it as `Registered=False`.

## Observations

Findings 1 to 11 from v4 still stand as facts about kcp, kcp-operator and earlier versions. v5 changes which of them matter:

| Finding | Relevance in v5 |
|---|---|
| 1, 3, 4, 5 (kcp token shapes, per-shard signing keys, no-overlap rotation, JWKS) | None: the Provider-side never sees a kcp token |
| 2 (SA deletion doesn't revoke bound tokens) | None for remote `Provider`s; still relevant for RFC 006 |
| 7 (fake kcp via connection-supplied CA plus token re-onboarding) | Closed differently: no CA bundle is sent, and rebind can't complete against a live connector |
| 8, 11 (kcp-operator's server CA and merged bundle) | None: the connector uses PM's own trust |
| 9 (replay of a logged registration) | Closed: PM-generated secret, only the hash sent, retries bound to the hash |
| 10 (kcp credential at the edge) | Closed: no kcp credential leaves the PM-side |

New observations from reviewing v4 (2026-10-08):

12. **In tunnel mode, kcp trust equals connection trust.** The connector controls every request's destination, so a separately pinned or delivered kcp CA can't make the Provider-side's view of kcp more trustworthy than the session. v2 to v4 spent much of their complexity on that separation.
13. **v4's manual rotation steps came from owner-supplied CAs**, not from rotation itself. Generated CAs already rotated automatically.
14. **None of the notes described how a Provider consumes N connections.** "Multiple PM instances are supported" was only true at the protocol level.

## Comparison

| | v2 | v4 | v5 |
|---|---|---|---|
| PM-side proof after registration | Long-lived bearer secret, issued by the Provider | Short-lived certs from an owner CA, inner mTLS | Long-lived bearer secret, generated by the PM-side, rotated automatically |
| Provider-side authenticated by | Web PKI | Web PKI, then a pinned server key | Web PKI |
| kcp credentials on the Provider-side | SA token + CA bundle | SA token + CA bundle | **None** |
| What a TLS-terminating edge sees | Secret, first access token | Spent onboarding token only | Spent onboarding token, connection secret, reconcile traffic (Provider's own edge) |
| Tunnel | Dial proxy (remotedialer/Konnectivity) | Dial proxy inside inner TLS | HTTP/2 over a WebSocket, connector as reverse proxy |
| TLS layers on kcp traffic | 2 | 3 | 2 (outer to the Provider, connector to kcp) |
| Revoking kcp access | Token TTL (finding 2) | Token TTL (finding 2) | Immediate: sessions closed |
| Rotation | Open item | Certs automatic; owner CA via `PUT /anchor`, manual for owner-supplied CAs | Automatic, no owner involvement |
| Recovery after a lost PM-side credential | Re-onboard with a token (finding 7) | New connection, state lost | Rebind with a waiting period, state kept |
| Fake kcp by an outsider | Yes (finding 7) | No | No |
| Replay of a logged registration | Hole (finding 9) | Safe | Safe |
| Provider-side software | Bearer check, token handling, dial-proxy server | TLS over WebSocket, pinning, CA checks, token handling | Bearer check, HTTP/2 client over WebSocket, connection set for multicluster-runtime |
| Main cost | Security holes | Complexity | Connector in the L7 data path |

## Changes from v4

| v4 | v5 | Why |
|---|---|---|
| Provider-side holds access tokens and kcp's CA bundle | Connector injects SA tokens; Provider-side holds no kcp credentials | Removes credential delivery, refresh, CA bundle handling, and finding 2 for remote `Provider`s |
| Dial proxy, end-to-end TLS to kcp | HTTP/2 reverse proxy over the session | The connector must see requests to inject credentials; observation 12 |
| Inner mTLS inside the WebSocket | Plain WebSocket over HTTPS | Provider's edge is the Provider's responsibility; nothing at the edge works against kcp |
| Pinned server key, rotation and refetch | Web PKI on the base URL | Only needed against an untrusted Provider edge |
| Owner CA, active/desired split, `PUT /anchor` | PM-generated secret, automatic rotation | No owner-supplied key material; observation 13 |
| No re-onboarding; lost CA means lost state | Rebind with a waiting period, cancelled by the live connector | Recovery without letting outsiders take over |
| Tunnel library open item | Standard HTTP/2 over a `net.Conn` | Nothing to choose |
| Multi-PM consumption unspecified | Per-connection `RoundTripper`, dynamic connection set, prefixed cluster names | Observation 14 |

## Rejected alternatives

All of v4's rejected alternatives still apply, except "a PM-owned front key, with the connector terminating TLS", whose objection (the connector sees plaintext) doesn't hold: the connector is PM-side infrastructure seeing traffic to PM's own kcp, which the front-proxy sees in plaintext anyway. In addition:

- **v4's inner mTLS and owner CA.** Correct, but they only defend against the Provider's own edge and against owner-supplied key handling. Too much machinery for those threats.
- **A dial proxy with credentials on the Provider-side (v1 to v4).** Forces credential delivery, refresh and kcp CA trust onto every Provider, and leaves a usable kcp token behind on every Provider-side compromise.
- **The connector authenticating with its own identity plus impersonation of the SA.** Avoids minting per-connection tokens, but impersonation rights are broader than TokenRequest on specific SAs, and a header-stripping bug would become privilege escalation.
- **A Provider-issued secret (v2).** The response would carry a secret, which reopens finding 9.
- **A gateway container exposing a local proxy and kubeconfig files** (discussed after v4). Doesn't help with multiple PM instances: the Provider still needs SDK code to consume a changing set of connections. It adds a hop and a directory format without adding a capability.
- **Rebind without a waiting period**, or one that a live connector can't cancel. Turns a stolen token into a takeover again (finding 7).

## Open items

- **Proxying at scale (the main risk).** Watches and long-running requests from every Provider pass through the connector. Measure memory and goroutines per stream, HTTP/2 stream limits per session, and backpressure. Decide whether to build on kcp's front-proxy code.
- **Header and request policy.** The exact headers stripped and added, and whether upgrade requests (`exec`, `port-forward`) are ever allowed.
- **Connector placement and HA.** As in v2, plus: it's now in the data path, so it needs horizontal scaling, and several connector replicas may each hold sessions for the same connection.
- **Provider-side multi-replica.** Sharding vs. Lease routing, and whether opening one session per Provider replica can be made reliable through common ingresses.
- **Rebind details.** Length of the waiting period, rate limits, and what happens when a thief holding the secret keeps cancelling the owner's rebinds (portal freeze is the current answer).
- **JWT assertion variant.** For Providers that don't trust their own edge: replace the bearer secret with a PM-side keypair, pinned at registration, that signs a short-lived JWT (RFC 7523 `private_key_jwt` style) on each call and session upgrade. Rotation stays automatic. The edge would then see no reusable credential, but still sees reconcile traffic.
- **`direct` mode.** In v5, the Provider-side never talks to kcp directly. "Direct" would mean the Provider-side calls a public connector endpoint instead of having the connector dial out. That needs a credential in the other direction; defer until there's a use case.
- **HTTP/2 through ingresses.** Confirm that HTTP/2-in-WebSocket survives common ingresses, CDNs and WAFs (binary frames, maximum message size, maximum connection age).
- **multicluster-runtime composition.** Confirm the `multi` provider (or an equivalent) supports adding and removing per-connection providers at runtime with prefixed cluster names.
- **Audit.** kcp's audit log shows the connection's SA; the connector's per-connection request log is the complement. Decide retention and format.
- **Readiness gate timeout**, **request permissions**, **addressing changes**: as in v4.

## Prior art

- **kcp front-proxy**: an authenticating reverse proxy for kcp, with streaming and watch support.
- **Teleport Kubernetes service**: a proxy that attaches identity to Kubernetes requests, so clients never hold cluster credentials.
- **Cloudflare Tunnel** and similar: an outbound connection from the private side, with HTTP served back over it.
- **Rancher `remotedialer`** and **Konnectivity**: the dial-proxy approach v5 moves away from.
- **Open Cluster Management cluster-proxy**: reverse tunnel to managed clusters.
- **kubeadm bootstrap tokens**: `tokenID.tokenSecret`, stored hashed, short-lived.
- **Prefixed secrets for scanning**: GitHub's token prefixes and secret-scanning partner programme.
- **Account recovery with a waiting period**: common in consumer identity providers; recovery completes only if the current credential holder doesn't object in time.
