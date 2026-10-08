# Notes v5: RFC 011 provider onboarding flow

Working notes for [011](./011-provider-onboarding-flow-improvements.md). Not the RFC itself. v5 builds on the [v4 notes](./xxx-notes-011-provider-onboarding-flow-improvements-v4.md) and removes most of its machinery by reversing two of its assumptions:

- **The Provider-side never holds kcp credentials.** The connector becomes an authenticating HTTP proxy for kcp. It adds the SA token itself, so no access token, kcp CA bundle or token rotation ever crosses to the Provider-side.
- **The Provider's edge is the Provider's responsibility**, mirroring v4's "the owner's workspace is the owner's responsibility". The inner mTLS, server-key pinning and owner CA lose their purpose. A connection is authenticated by a PM-generated secret that rotates automatically.

What's left is: one registration call, one WebSocket carrying multiplexed streams (so the Provider-side can "dial" and use client-go's standard transport), and one secret that nobody has to touch. See "Changes from v4" and "Comparison".

## Terminology

- **PM-side**: the PM instance (kcp + PM operator). May be behind NAT; dials out.
- **Provider-side**: the service provider (controllers + portal in its runtime cluster). Serves the connection API and reconciles on the PM-side.
- **Owner**: whoever controls the workspace that holds the `Provider` object and its connection Secret on the PM-side.
- **Connection**: the relationship between one `Provider` object on the PM-side and one connection record on the Provider-side, identified by `connectionID`. Provider-side state belongs to its connection.
- **Onboarding token**: a single-use, short-lived token from the Provider's portal, with the Provider's base URL inside. Unchanged from v2.
- **Connection secret**: a high-entropy secret the PM-side generates and keeps in the connection Secret. The Provider-side stores only its hash. It authenticates the PM-side on every call after registration. It grants **no** kcp access.
- **Session**: one WebSocket from the connector to the Provider's base URL, authenticated with the connection secret. It carries a stream multiplexer. The Provider-side opens streams on demand, as if dialling, and speaks plain HTTP on each; the connector serves HTTP on them.
- **Connector**: the PM-side component that opens sessions and acts as an **authenticating reverse proxy** to kcp for requests arriving over them. Implemented as a session controller plus a virtual workspace in PM's virtual-workspaces server (see "Connector implementation: a virtual workspace").
- **Connection principal**: the identity a request arriving over a session of connection `{id}` is authenticated as inside the connector, e.g. `pm:connection:<id>`. The connector maps it to the connection's SA before forwarding to kcp.
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
Provider-side controllers ── client-go, standard http.Transport (real kcp URLs)
                                   │ "dial" = open a stream on the session
             ┌──── one WebSocket (outbound from PM), stream multiplexer inside ────┘
             ▼                       plain HTTP on each stream
          connector ── in-process http.Server + reverse proxy:
                       check host, strip auth headers, add SA token ── TLS ──▶ kcp
```

- **The connector dials out**, as in every version: a WebSocket to `GET /v1/connections/{id}/session` on the base URL, with `Authorization: Bearer <connection secret>`. The Provider's ingress terminates TLS and routes it like any WebSocket.
- **Inside the WebSocket runs a stream multiplexer** (remotedialer, or yamux/smux). The Provider-side opens a new stream whenever client-go wants a new connection, so client-go's standard transport works unchanged: pooling, health checks, and separate connections for upgrades. See "Sessions" for why this matters more than the HTTP version.
- **The connector is an authenticating reverse proxy.** Each stream is handed to an in-process HTTP server. For each request it checks the target host against the allow-list, removes any credentials the Provider-side sent, adds a short-lived SA token for that connection, and forwards the request to the real kcp URL over TLS verified with PM's own trust.
- **The Provider-side keeps the real kcp URLs** (front-proxy, and the shard virtual-workspace URLs from endpoint slices). Only the transport's dial hook is replaced: it opens a stream on the connection's session instead of a TCP connection.

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
5. **Open a session.** The connector opens the WebSocket with the connection secret and runs the stream multiplexer over it; the Provider-side opens streams to it as needed.
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

GET /v1/connections/{id}/session                 (WebSocket upgrade; stream multiplexer inside)
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

### Sessions: a stream multiplexer over a WebSocket

**What really differs from a normal client-go connection.** A regular client-go client already sends all requests and watches to a host over one HTTP/2 TCP connection. It already accepts head-of-line blocking, shared flow control, and losing every watch (and relisting) when that connection dies. kube-apiserver even sends GOAWAY on purpose (`--goaway-chance`) to rebalance clients. None of that is new here. Two things are:

1. **The client can't open connections.** client-go normally dials whenever it needs to: a new host, a full or dead connection, or an upgrade (`exec` uses a separate HTTP/1.1 connection). Behind NAT, only the connector can create sessions. A design that hands the Provider-side a fixed set of connections (e.g. HTTP/2 directly on the WebSocket) has to replace client-go's pooling, health checks and upgrade handling with custom code.
2. **The path isn't under PM's control.** Traffic crosses the internet and the Provider's L7 edge, with its own idle timeouts, maximum connection age and WAF rules, and higher round-trip times than in-cluster clients.

**A multiplexer fixes the first.** Each stream is a connection the Provider-side opens on demand, so client-go's standard machinery works unchanged. The HTTP version on each stream then hardly matters: HTTP/1.1 is the natural default over plain streams; HTTP/2 without TLS (h2c) would also work.

- **Provider-side**: the session handler checks the bearer secret against connection `{id}`, accepts the WebSocket and runs the multiplexer on it. The session is registered under `{id}`; deleting the connection or rotating the secret closes it. The connection's `http.Transport` uses a dial hook that opens a stream on one of the connection's sessions:

  ```go
  tr := &http.Transport{
      // Called for https:// URLs. The stream isn't TLS, so the transport
      // speaks plain HTTP/1.1 on it; the real kcp URL stays in the request.
      DialTLSContext: func(ctx context.Context, _, addr string) (net.Conn, error) {
          return sessions.OpenStream(ctx, conn.ID, addr) // addr: target host:port, checked by the connector
      },
  }
  ```

- **PM-side**: the connector opens the WebSocket and runs the other end of the multiplexer. Every stream the Provider-side opens is handed to an in-process `http.Server`, whose handler is the reverse proxy below (`httputil.ReverseProxy`, which also proxies upgrades).
- **This is remotedialer's designed direction**: the side that accepted the WebSocket (the Provider-side) dials through the side that opened it (the connector). Instead of dialling the target, the connector's dialer hook routes the stream to the local HTTP server.
- **Several sessions per connection** are allowed: for connector HA, for spreading load, and to limit the head-of-line blocking that all streams on one TCP connection share. The dial hook spreads new streams across a connection's sessions.
- **Liveness**: WebSocket pings at the outer level, so ingresses see traffic, and multiplexer keepalives inside. client-go's own HTTP health checks work as usual.
- **Graceful replacement.** The connector replaces sessions before the Provider's edge cuts them at its maximum connection age: it opens a new session, then tells the Provider-side to stop opening streams on the old one (the multiplexer's go-away, where available) and closes it once in-flight requests finish. An abrupt cut by the edge still drops all watches on that session at once, as with any apiserver connection.
- **Why not a dial proxy to kcp** (remotedialer or Konnectivity as used in v1 to v4): there the stream carried opaque TLS from the Provider-side to kcp, so the Provider-side had to hold kcp credentials and trust kcp's CA. In v5 the same kind of stream ends at the connector's HTTP server, which owns both.

### The connector as an authenticating proxy

For every request arriving over a session of connection `{id}`:

1. **Check the target.** The request's host must be on the allow-list: the front-proxy and the `spec.virtualWorkspaceURL` host of every kcp `Shard`, built only from PM-side data (kcp's cache-backed `Shard` view at `/services/admin`, which the front-proxy also uses). Resolve once, check every address, dial those exact addresses; refuse link-local, multicast and cloud-metadata addresses. Unchanged from v2.
2. **Strip credentials and identity headers**: `Authorization`, `Impersonate-*`, `Proxy-*`, cookies and hop-by-hop headers. Otherwise the Provider-side could ask kcp to act as someone else, using the connector's token.
3. **Add the SA's identity** for connection `{id}`. Baseline: a bound token from TokenRequest, minted by the connector, cached in memory, refreshed at half its TTL, never written anywhere and never sent to the Provider-side. Impersonating the SA instead was checked and rejected: it works for the provider workspace but not for the APIExport virtual workspace (observation 16).
4. **Forward** to the real URL over TLS, verified against the CA bundle PM's own components use for kcp. Rotating kcp's CAs or adding shards needs no coordination: if PM trusts its kcp, the connector does.
5. **Log and limit** per connection: method, path, status, duration; rate limits, concurrent-stream limits and header-size limits per connection, so one Provider can't exhaust the connector for others. The connector is in the same position as kcp's front-proxy towards its clients, and needs the same defences.

- **RBAC is unchanged**: kcp sees the connection's SA, with the RBAC the Provider controller created (RFC 006). The connector doesn't widen it.
- **Upgrade requests** (`exec`, `attach`, `port-forward`) are refused by default; a Provider reconciling an APIExport doesn't need them. The transport could carry them (HTTP/1.1 upgrades on a stream), so it's a policy choice. See "Open items".
- **Build on kcp's serving stack** rather than writing a proxy from scratch. The next section does that with a virtual workspace.

### Connector implementation: a virtual workspace

The connector has two halves. Only one is request-shaped, and that one maps directly onto kcp's virtual workspace framework.

**What already exists** (checked on 2026-10-08):
- **PM's virtual-workspaces server** (`platform-mesh/services/virtual-workspaces`) is built on `kcp-dev/virtual-workspace-framework` and serves `/services/marketplace` and `/services/contentconfigurations` behind the front-proxy (`additionalPathMappings` in `pm-helm-charts/charts/infra/values.yaml`). It builds its own authenticator union (`cmd/start.go`: `union.New(authentication.New(clientCfg), …)`), so a new authenticator needs no change in kcp.
- **kcp's initializing-workspaces virtual workspace already does the proxy half.** Its workspace-content handler (`kcp/pkg/virtual/initializingworkspaces/builder/build.go`, a `handler.VirtualWorkspace` with a `HandlerFactory`) authorizes the caller, strips `Authorization` and `Impersonate-*` headers, impersonates a chosen identity and reverse-proxies to the shard, watches included (`kcp/pkg/virtual/shared/proxy.go`, `ServeProxy`).
- **kcp-operator's `VirtualWorkspace` resource** (v0.9+) deploys an external virtual-workspace server: Deployment, Service, certificates, a kubeconfig identity, and a target of the root shard (singleton) or one shard. PM's charts already use it for `kcp-access-vw`.
- **kcp's admin virtual workspace** (`/services/admin`) serves every shard's `Shard` object from the cache server; the front-proxy discovers shards through it.

**Design:**

```
                        PM's virtual-workspaces server (one process)
 ┌──────────────────────────────────────────────────────────────────────────────┐
 │ session controller (post-start hook)    in-process listener per connection   │
 │   watches Providers + connection Secrets  ConnContext: connectionID          │
 │   dials WebSocket ──▶ Provider            │                                  │
 │   multiplexer: streams ───────────────────┘                                  │
 │                                           ▼                                  │
 │   full handler chain (authn → authz → audit → timeouts → …)                  │
 │     authn: session authenticator reads connectionID from context             │
 │     path:  /services/providerconnections/<id>/<original path>, Host kept      │
 │                                           ▼                                  │
 │   providerconnections VW handler: check Host, strip auth headers,             │
 │   add SA identity, reverse-proxy ──▶ front-proxy / shard VW URLs              │
 └──────────────────────────────────────────────────────────────────────────────┘
```

1. **Session controller.** Not a virtual workspace: a controller in the same process, started from a post-start hook, the way kcp's APIExport virtual workspace runs its controllers. It watches `Provider` objects and connection Secrets, dials sessions and accepts multiplexer streams. It serves each stream with an in-process `http.Server` whose handler is the server's **full handler chain**, not the virtual workspace's handler directly, so authentication, authorization, audit and timeouts all apply. `ConnContext` tags each stream with its `connectionID`, and the server prefixes the path with `/services/providerconnections/<id>`, keeping the `Host`.
2. **Session authenticator.** A new member of the authenticator union maps the `connectionID` in the request context to the connection principal. The value is set only on in-process listeners, so nothing arriving over the network can forge it, and requests from the front-proxy never carry it.
3. **The `providerconnections` virtual workspace.** A `handler.VirtualWorkspace` following the initializing-workspaces pattern:
   - its **authorizer** lets the connection principal reach only its own connection's paths: its provider workspace and the APIExport virtual-workspace endpoints. Upgrade requests are refused here;
   - its **handler** checks the `Host` against the allow-list, strips client credentials (`ServeProxy`'s header list plus `Proxy-*` and cookies), adds the SA's identity, and reverse-proxies to the real URL.
4. **Connection status as a resource.** The same virtual workspace can serve a read-only `ProviderConnection` resource from the session controller's in-memory registry (sessions, `kcpReachable`, last error), the way the admin virtual workspace serves `Shard`s. Owners see it with `kubectl get providerconnections` through the front-proxy, and the request's readiness gate can read it, without frequent status writes to the `Provider`.

**What this gives:**
- **Placement**: PM's existing virtual-workspaces server, or a dedicated one deployed through a kcp-operator `VirtualWorkspace`.
- **A full serving stack**: audit (each request recorded with the connection principal and the SA it acts as), long-running request detection, timeouts, max-in-flight limits, panic recovery and `apiserver_request_*` metrics, instead of a hand-written proxy.
- **Request policy in one place**: the virtual workspace's authorizer, instead of ad-hoc checks in proxy code.
- **A natural home for `direct` mode** (see "Open items"): the same virtual workspace is reachable through the front-proxy.

**Identity: SA tokens, not impersonation.** The connector mints TokenRequest tokens (step 3 above) for all traffic. Impersonating the SA looked attractive here, since the initializing-workspaces pattern impersonates, but it only works for half of the traffic (observation 16):
- **Provider workspace (front-proxy, then shard): works.** Needs `access` on `/` and `impersonate` on `serviceaccounts` with `resourceNames: [<sa>]` in the provider workspace; no user extra.
- **APIExport virtual workspace (standalone server): doesn't work.** Per-cluster requests are denied, and the grant that would allow wildcard requests lets the connector impersonate anyone.

Tokens are needed for the APIExport virtual workspace anyway, so a mix would add RBAC and code paths without removing anything.

**Caveats:**
- **Timeouts.** The generic handler chain applies a timeout (60 s by default) to requests it doesn't classify as long-running. Large proxied LISTs may exceed it; the long-running check and timeout must be configured for proxied traffic.
- **Hop limit.** Proxying into the APIExport virtual workspace counts toward kcp's `X-Kcp-Virtual-Resource-Hops` limit (maximum 4).
- **Sessions stay pinned to one replica.** Replicas of the virtual-workspaces server need a Lease per connection. Start as a singleton with leader election; one per shard later if needed.
- **`ServeProxy` is kcp-internal** (`kcp/pkg/virtual/shared`), not part of the framework module. Copy it (about 30 lines) or propose moving it into the framework.

### Provider-side: consuming N connections

None of the earlier notes covered this, though serving any number of PM instances was the first problem in the RFC stub.

- **One base config per connection.** The SDK gives each connection an `http.Transport` whose dial hook opens streams on that connection's sessions (see "Sessions"):

  ```go
  cfg := &rest.Config{
      Host:      conn.FrontProxyURL,     // real URL; the connector routes on it
      Transport: sdk.Transport(conn.ID), // no TLS options: client-go refuses both
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
| Ingress drops a session | The connector reconnects. Pings keep idle timeouts from triggering, and the connector replaces sessions before the edge's maximum connection age. On an abrupt cut, in-flight requests and watches on that session fail and client-go retries and relists, as after any apiserver disconnect. |
| PM rotates kcp's CA, adds shards or changes its edge | Nothing to coordinate. The connector uses PM's own trust and watches `Shard`s for the allow-list. |
| `POST` response lost | Retry with the same token, idempotency key and `secretHash`: same `connectionID`. A different hash is refused. |
| kcp unreachable from the connector | `kcpReachable=false` with the error; during registration the request stays in `VerifyingKcp` and fails after a timeout. |
| Rotation response lost, or connector crash mid-rotation | Try `pendingSecret`, then `secret`; resume. An unused pending hash expires. |
| Connection Secret lost | A new request rebinds; the connection and its state survive after the waiting period. |
| Rebind started by someone else | The live connector's next authentication cancels it; the owner is notified. |
| Thief rotates the secret | The legitimate connector can't authenticate and the owner is notified. The owner freezes or deletes the connection in the portal. |
| Connection deleted on the Provider-side | Sessions and calls are refused. `Registered=False` on the `Provider`. Recovery is a new connection. |
| Connector reconnects while an old session is open | Both are valid sessions for the connection; new streams go to the live one, and the old one dies by liveness timeout or is closed. No "replace the tunnel" race, because sessions aren't exclusive. |
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
15. **The tunnel's limits are mostly those of any client-go connection.** One TCP connection for all requests, shared flow control, and mass relists when it dies are normal client-go behaviour against kube-apiserver. What a reverse tunnel really changes is that the client can't open connections on its own, and that the path runs through the Provider's L7 edge. A multiplexer restores the first; the second is handled by pings and graceful session replacement.
16. **Impersonating the connection's SA works for the provider workspace, not for the APIExport virtual workspace** (kcp `3523eb866` and its vendored Kubernetes fork, read on 2026-10-08; not tested).
    - **Provider workspace, through the front-proxy to the shard.** kcp's fork treats an SA user without the `authentication.kcp.io/cluster-name` extra as local to whichever workspace the request targets (`vendor/k8s.io/kubernetes/pkg/registry/rbac/validation/kcp.go`, `IsForeign`). The workspace authorizer then authorizes it as a local SA (`pkg/authorization/workspace_content_authorizer.go`). The shard's chain allows the impersonation: kcp's gatekeeper only blocks privileged groups (`pkg/server/filters/impersonation.go`), the upstream filter checks `impersonate` on `serviceaccounts` for that namespace and name and adds the SA groups itself (`vendor/k8s.io/apiserver/pkg/endpoints/filters/impersonation/impersonation.go`), and kcp's scoping filter confines the impersonated user to that workspace. RBAC: `access` on `/` plus `impersonate` on `serviceaccounts` with `resourceNames: [<sa>]`. Adding the cluster-name extra would make the identity identical to a real token, at the cost of `impersonate` on `userextras/authentication.kcp.io/cluster-name` with `resourceNames: [<cluster>]`. Constrained impersonation (beta, on by default in the fork) still accepts the plain `impersonate` verb.
    - **APIExport virtual workspace, standalone server.** It uses the upstream default handler chain, without kcp's gatekeeper or scoping, and the `impersonate` check goes to the APIExport virtual workspace's authorizer (`pkg/virtual/apiexport/builder/build.go`, `newAuthorizer`). For per-cluster requests, `boundAPIAuthorizer` only allows resources bound or claimed in the consumer's APIBinding, so impersonating `serviceaccounts` is denied (`pkg/virtual/apiexport/authorizer/binding.go`). For wildcard requests, the content authorizer checks `impersonate` on `apiexports/content` without looking at the impersonation target (`content.go`), and the maximal-permission authorizer allows unclaimed resources (`maximal_permission_policy.go`). Granting that verb would let the caller impersonate any user or group, including `system:masters`, which the virtual-workspace server always allows (`virtual-workspace-framework/pkg/options/authorization.go`).

## Comparison

| | v2 | v4 | v5 |
|---|---|---|---|
| PM-side proof after registration | Long-lived bearer secret, issued by the Provider | Short-lived certs from an owner CA, inner mTLS | Long-lived bearer secret, generated by the PM-side, rotated automatically |
| Provider-side authenticated by | Web PKI | Web PKI, then a pinned server key | Web PKI |
| kcp credentials on the Provider-side | SA token + CA bundle | SA token + CA bundle | **None** |
| What a TLS-terminating edge sees | Secret, first access token | Spent onboarding token only | Spent onboarding token, connection secret, reconcile traffic (Provider's own edge) |
| Tunnel | Dial proxy (remotedialer/Konnectivity) | Dial proxy inside inner TLS | Multiplexed streams over a WebSocket, ending at the connector's reverse proxy |
| TLS layers on kcp traffic | 2 | 3 | 2 (outer to the Provider, connector to kcp) |
| Revoking kcp access | Token TTL (finding 2) | Token TTL (finding 2) | Immediate: sessions closed |
| Rotation | Open item | Certs automatic; owner CA via `PUT /anchor`, manual for owner-supplied CAs | Automatic, no owner involvement |
| Recovery after a lost PM-side credential | Re-onboard with a token (finding 7) | New connection, state lost | Rebind with a waiting period, state kept |
| Fake kcp by an outsider | Yes (finding 7) | No | No |
| Replay of a logged registration | Hole (finding 9) | Safe | Safe |
| Provider-side software | Bearer check, token handling, dial-proxy server | TLS over WebSocket, pinning, CA checks, token handling | Bearer check, multiplexer server, a dial hook for the standard transport, connection set for multicluster-runtime |
| Main cost | Security holes | Complexity | Connector in the L7 data path |

## Changes from v4

| v4 | v5 | Why |
|---|---|---|
| Provider-side holds access tokens and kcp's CA bundle | Connector injects SA tokens; Provider-side holds no kcp credentials | Removes credential delivery, refresh, CA bundle handling, and finding 2 for remote `Provider`s |
| Dial proxy, end-to-end TLS to kcp | Streams over the session end at the connector's HTTP reverse proxy | The connector must see requests to inject credentials; observation 12 |
| Inner mTLS inside the WebSocket | Plain WebSocket over HTTPS | Provider's edge is the Provider's responsibility; nothing at the edge works against kcp |
| Pinned server key, rotation and refetch | Web PKI on the base URL | Only needed against an untrusted Provider edge |
| Owner CA, active/desired split, `PUT /anchor` | PM-generated secret, automatic rotation | No owner-supplied key material; observation 13 |
| No re-onboarding; lost CA means lost state | Rebind with a waiting period, cancelled by the live connector | Recovery without letting outsiders take over |
| Custom dialer, pinning callbacks and tunnel inside inner TLS | Multiplexer over a plain WebSocket; client-go's standard transport with a dial hook | Observation 15: keep client-go's own pooling, health checks and upgrades |
| Connector placement an open item | Session controller plus a `providerconnections` virtual workspace in PM's virtual-workspaces server | Reuses kcp's serving stack (authn, authz, audit, timeouts) and an existing deployment path |
| Multi-PM consumption unspecified | Per-connection `RoundTripper`, dynamic connection set, prefixed cluster names | Observation 14 |

## Rejected alternatives

All of v4's rejected alternatives still apply, except "a PM-owned front key, with the connector terminating TLS", whose objection (the connector sees plaintext) doesn't hold: the connector is PM-side infrastructure seeing traffic to PM's own kcp, which the front-proxy sees in plaintext anyway. In addition:

- **v4's inner mTLS and owner CA.** Correct, but they only defend against the Provider's own edge and against owner-supplied key handling. Too much machinery for those threats.
- **A dial proxy with credentials on the Provider-side (v1 to v4).** Forces credential delivery, refresh and kcp CA trust onto every Provider, and leaves a usable kcp token behind on every Provider-side compromise.
- **The connector impersonating with general impersonation rights.** Avoids minting per-connection tokens, but broad impersonation rights are wider than TokenRequest on specific SAs, and a header-stripping bug would become privilege escalation.
- **The connector impersonating the connection's SA, scoped by `resourceNames`.** Narrow and verified to work for requests to the provider workspace, but not for the APIExport virtual workspace, which carries most Provider traffic (observation 16). Tokens are needed there anyway.
- **A standalone connector proxy.** Would have to rebuild authentication, authorization, audit, timeouts, in-flight limits and metrics that the virtual workspace framework and PM's virtual-workspaces server already provide.
- **A Provider-issued secret (v2).** The response would carry a secret, which reopens finding 9.
- **A gateway container exposing a local proxy and kubeconfig files** (discussed after v4). Doesn't help with multiple PM instances: the Provider still needs SDK code to consume a changing set of connections. It adds a hop and a directory format without adding a capability.
- **Rebind without a waiting period**, or one that a live connector can't cancel. Turns a stolen token into a takeover again (finding 7).
- **HTTP/2 directly on the WebSocket** (an earlier v5 draft), with the Provider-side as client and the connector as server. Works, and HTTP/2's own limits are no worse than any client-go connection's (observation 15). But the Provider-side then has a fixed set of connections it can't add to, so the SDK would have to replace client-go's pooling, stream-limit handling, health checks and upgrade connections with custom code, on an unusual reversed setup with little tooling.
- **HTTP/1.1 directly on the WebSocket.** One request at a time per connection; the first watch would block the session.

## Open items

- **Proxying at scale (the main risk).** Watches and long-running requests from every Provider pass through the connector. Measure memory and goroutines per stream, streams per session, and backpressure, in the virtual-workspaces server. Whether connector traffic should share a server with the marketplace and content-configuration virtual workspaces, or get its own `VirtualWorkspace` deployment.
- **Multiplexer choice.** remotedialer (built for this direction, widely deployed) vs. yamux/smux. Check: whether remotedialer applies backpressure per stream (it may buffer instead), whether its client side accepts a custom dialer so streams can go to the in-process HTTP server, and whether it has a go-away for graceful session replacement. yamux has per-stream windows and a go-away, but no built-in WebSocket integration or dial addressing.
- **Transport check.** Confirm that `http.Transport` with a `DialTLSContext` returning a non-TLS stream speaks HTTP/1.1 to `https://` URLs as expected, and that client-go accepts it as `rest.Config.Transport` together with multicluster-runtime's per-endpoint configs.
- **Edge timing.** A session ping interval and a session replacement schedule that fit common idle timeouts and maximum connection ages; window or buffer sizes for high round-trip times (large LISTs).
- **Request policy in the virtual workspace's authorizer.** The exact paths the connection principal may reach, the headers stripped and added, and whether upgrade requests (`exec`, `port-forward`) are ever allowed.
- **Connector HA.** Placement is settled (a virtual workspace, see "Connector implementation"). Open: singleton with leader election vs. one server per shard (kcp-operator `VirtualWorkspace` with `shardRef`), Leases per connection across replicas, and horizontal scaling now that it's in the data path.
- **Possible kcp issue: impersonation on the APIExport virtual workspace.** From reading the code (observation 16), anyone with `impersonate` (or `*`) on `apiexports/content` may be able to send wildcard requests to that export's virtual workspace while impersonating any group, including `system:masters`, which the server always allows. Its effect looks limited, because wildcard requests are forwarded with the virtual workspace's own client anyway. Confirm with an e2e test before reporting it to kcp.
- **Proxied requests in the generic handler chain.** Configure the long-running check and request timeout for proxied LISTs and watches; confirm the `X-Kcp-Virtual-Resource-Hops` budget when proxying into the APIExport virtual workspace.
- **`ServeProxy` upstream.** Propose moving `kcp/pkg/virtual/shared.ServeProxy` into the virtual workspace framework, or copy it.
- **`ProviderConnection` status resource.** Its schema, and whether the request's readiness gate reads it instead of calling `GET /v1/connections/{id}` on the Provider-side.
- **Provider-side multi-replica.** Sharding vs. Lease routing, and whether opening one session per Provider replica can be made reliable through common ingresses.
- **Rebind details.** Length of the waiting period, rate limits, and what happens when a thief holding the secret keeps cancelling the owner's rebinds (portal freeze is the current answer).
- **JWT assertion variant.** For Providers that don't trust their own edge: replace the bearer secret with a PM-side keypair, pinned at registration, that signs a short-lived JWT (RFC 7523 `private_key_jwt` style) on each call and session upgrade. Rotation stays automatic. The edge would then see no reusable credential, but still sees reconcile traffic.
- **`direct` mode.** In v5, the Provider-side never talks to kcp directly. "Direct" would mean the Provider-side calls the connector instead of the connector dialling out. The virtual workspace makes this cheap: it's already reachable through the front-proxy at `/services/providerconnections/<id>/...`, with the same routing, identity injection and authorizer. What's missing is a credential in the other direction: one that the virtual workspace accepts for its connection only, is useless against kcp directly, and is revoked by deleting the connection. Defer until there's a use case.
- **WebSockets through ingresses.** Confirm that a long-lived binary WebSocket survives common ingresses, CDNs and WAFs (binary frames, maximum message size, maximum connection age).
- **multicluster-runtime composition.** Confirm the `multi` provider (or an equivalent) supports adding and removing per-connection providers at runtime with prefixed cluster names.
- **Audit.** kcp's audit log shows the connection's SA; the virtual-workspaces server's audit log records the connection principal and the SA it acted as. Decide the audit policy and retention for connector traffic.
- **Readiness gate timeout**, **request permissions**, **addressing changes**: as in v4.

## Prior art

- **kcp front-proxy**: an authenticating reverse proxy for kcp, with streaming and watch support.
- **kcp's initializing-workspaces virtual workspace**: a `handler.VirtualWorkspace` that authorizes, strips credentials, impersonates and reverse-proxies into a workspace; the pattern for the connector's proxy half.
- **kcp's admin virtual workspace** (`/services/admin`): a cache-backed view of all `Shard`s; the allow-list source, and the pattern for a read-only status resource.
- **PM's virtual-workspaces server** and **kcp-operator's `VirtualWorkspace` resource**: where the connector runs and how it's deployed.
- **Teleport Kubernetes service**: a proxy that attaches identity to Kubernetes requests, so clients never hold cluster credentials.
- **Cloudflare Tunnel** and similar: an outbound connection from the private side, with HTTP served back over it.
- **Rancher `remotedialer`**: multiplexed streams over a WebSocket, dialled from the side that accepted it; v5's candidate multiplexer, with streams ending at the connector's HTTP server instead of at kcp.
- **Konnectivity**: the dial-proxy approach v1 to v4 used, with the client holding kcp credentials.
- **client-go and kube-apiserver connection handling**: one HTTP/2 connection per host, health checks, and server-sent GOAWAY (`--goaway-chance`) for rebalancing; the baseline v5's tunnel is measured against.
- **Open Cluster Management cluster-proxy**: reverse tunnel to managed clusters.
- **kubeadm bootstrap tokens**: `tokenID.tokenSecret`, stored hashed, short-lived.
- **Prefixed secrets for scanning**: GitHub's token prefixes and secret-scanning partner programme.
- **Account recovery with a waiting period**: common in consumer identity providers; recovery completes only if the current credential holder doesn't object in time.
