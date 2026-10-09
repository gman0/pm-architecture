# Notes v5: RFC 011 provider onboarding flow

Working notes for [011](./011-provider-onboarding-flow-improvements.md). Not the RFC itself. v5 builds on the [v4 notes](./xxx-notes-011-provider-onboarding-flow-improvements-v4.md) and removes most of its machinery by reversing two of its assumptions:

- **The Provider-side never holds kcp credentials.** The connector becomes an authenticating HTTP proxy for kcp. It adds the SA token itself, so no access token, kcp CA bundle or token rotation ever crosses to the Provider-side.
- **The Provider's edge is the Provider's responsibility**, mirroring v4's "the owner's workspace is the owner's responsibility". The inner mTLS, server-key pinning and the pinned connection key proven inside it lose their purpose.

It then splits the PM-side into a **control plane** (one stateful connection controller) and a **data plane** (stateless connectors, one per kcp shard). Control calls are authenticated by a PM-generated secret per connection; sessions by short-lived assertions signed with short-lived certificates from a PM connections CA. Nothing needs anyone's involvement to rotate, and owners learn nothing about PM's shards.

What's left is: one registration call, at most one WebSocket per (connection, shard), only where the connection has traffic, carrying multiplexed streams (so the Provider-side can "dial" and use client-go's standard transport), and credentials that nobody has to touch. See "Changes from v4" and "Comparison".

## Terminology

- **PM-side**: the PM instance (kcp + PM operator). May be behind NAT; dials out.
- **Provider-side**: the service provider (controllers + portal in its runtime cluster). Serves the connection API and reconciles on the PM-side.
- **Owner**: whoever controls the workspace that holds the `Provider` object and its connection Secret on the PM-side.
- **Connection**: the relationship between one `Provider` object on the PM-side and one connection entry in the Provider-side's own database, identified by `connectionID`. Provider-side state belongs to its connection.
- **Onboarding token**: a single-use, short-lived token from the Provider's portal, with the Provider's base URL inside. Unchanged from v2.
- **Connection controller**: the PM-side **control plane**. One stateful component (HA, leader election) that registers connections, makes every control call, rotates credentials, writes each connection's record into its provider workspace, and reports aggregate status on the `Provider`. Never on the path of session traffic.
- **Connector**: the PM-side **data plane**. A stateless virtual-workspace server, **one per kcp shard**, that opens sessions and acts as an **authenticating reverse proxy** to kcp for requests arriving over them.
- **Connection secret**: a high-entropy secret the connection controller generates and keeps in the connection Secret in the owner's workspace. The Provider-side stores only its hash. It authenticates **control calls only**. It grants **no** kcp access.
- **Session CA**: a PM connections CA that only issues shard certificates for signing session assertions, run by the control plane (cert-manager, ideally with a KMS- or Vault-backed issuer). Every connection pins the CA's **certificate** at registration. Each session CA is an **immutable generation**: never renewed, and replaced by a new CA well before it expires (see "Pinning immutable CA generations").
- **Shard certificate**: a short-lived certificate (e.g. 24h) from the session CA for one shard's connector, renewed automatically and mounted read-only into the connector. It names the virtual-workspace host its sessions serve.
- **Session assertion**: a short-lived JWT signed with the shard certificate's key, carrying the shard certificate (and an intermediate, if PM uses one), bound to one connection. It authenticates a session upgrade and nothing else. The Provider-side verifies it locally against the pinned session CA certificates.
- **Session**: one WebSocket from one shard's connector to the Provider's base URL, authenticated with a session assertion. There's at most one per (connection, shard), opened only where the connection has traffic (see "Per-shard connectors"). It carries a stream multiplexer. The Provider-side opens streams on demand, as if dialling, and speaks plain HTTP on each; the connector serves HTTP on them.
- **Connection export**: a PM APIExport that serves the `ProviderConnection` resource. The Provider controller binds it in every provider workspace when it creates the workspace. It claims read-only access to `apiexportendpointslices` there.
- **Connection record**: a non-secret `ProviderConnection` object (`connectionID`, base URL, the connection's SA) in the connection's **provider workspace**, written by the connection controller through the connection export. The Provider is admin in that workspace (RFC 006), but the connector's authorizer denies it any write to the record (see "Connector policy"). Connectors read it through the export's endpoint slice, so the record lives on the shard of the workspace it describes.
- **Connection principal**: the identity a request arriving over a session of connection `{id}` is authenticated as inside the connector, e.g. `pm:connection:<id>`. The connector maps it to the connection's SA before forwarding to kcp.
- **`ProviderOnboardingRequest`**: as in v4, plus rebind (see "Recovery: rebind").

## Problem

Unchanged from v2: the Provider-side must reconcile on any number of PM instances, the PM-side may be behind NAT and opens every connection, the user gets a token from the Provider's portal, and both sides must recover from restarts and outages without manual steps.

**What v5 changes is where kcp credentials live.** v1 to v4 all delivered kcp credentials (a kubeconfig, then SA tokens and a CA bundle) to the Provider-side. Most of the complexity in v2 to v4 follows from that:
- the credentials must not be visible at a TLS-terminating hop (finding 10), which led to v4's inner mTLS;
- the Provider-side must trust kcp's CA, and whoever can change that bundle can present a fake kcp (finding 7);
- the credentials must be refreshed, and deleting the SA doesn't revoke them in PM's kcp (finding 2);
- the PM-side needs a strong, long-lived identity to push them, which led to v4's pinned per-connection key, proven inside the inner mTLS, and its rotation procedure.

**Constraints** (from v4, still valid):
- **Assume as little as possible about the Provider's infrastructure.** Any HTTPS endpoint that supports WebSockets must do. No TLS passthrough, no client-certificate-verifying ingress.
- **Assume nothing about how PM operates kcp's PKI.** Rotating kcp's CAs, adding shards or changing PM's edge must not involve Providers.

**New constraints in this revision:**
- **Owners learn nothing about PM's topology.** Nothing in an owner's workspace names shards or per-shard state.
- **Virtual workspaces stay stateless.** A connector holds no persistent state and writes nothing to kcp except the TokenRequests it needs; everything it knows about its place comes from its deployment.

## Trust boundaries

Two symmetric statements:

- **The owner's workspace is the owner's responsibility** (from v4). Whoever can read the connection Secret can act as the owner's connection for control calls.
- **The Provider's edge is the Provider's responsibility** (new). A Provider already trusts its ingress, load balancer or CDN with all of its customers' traffic. v5 doesn't defend a connection against the Provider's own edge.

v5 makes sure that:
- **nothing outside these two boundaries** can take over a connection: not a stolen onboarding token, not a compromised portal account on its own (see "Recovery: rebind"), not an observer of logged requests;
- **no kcp credential ever leaves the PM-side**, so a compromised Provider-side database or edge yields nothing that works against kcp once its sessions are gone;
- **revocation is immediate**: when the connectors stop proxying, the Provider-side has nothing left to use;
- **a Provider can't harm PM from within its own workspace**, even though it's admin there: its only path into kcp is the connector, whose authorizer denies the few operations that would affect PM (see "Connector policy").

**In tunnel mode, trusting kcp is trusting the connection.** The connector decides where every request goes, so the Provider-side can never verify kcp more strongly than it verifies the session. v2 to v4 treated "kcp's CA bundle" as a separate concern; v5 drops it. The Provider-side trusts responses that arrive over an authenticated session, for that connection only.

## Core idea

```
Provider-side controllers ── client-go, standard http.Transport (real kcp URLs)
                                   │ "dial" = open a stream on the session for that host
             ┌──── one WebSocket per shard (outbound from PM), multiplexer inside ────┘
             ▼                       plain HTTP on each stream
   connector@shard ── in-process http.Server + reverse proxy:
                      check host, strip auth headers, add SA token ── TLS ──▶ kcp
```

- **The connectors dial out**, as in every version: each shard's connector that the connection needs (see "Per-shard connectors") opens a WebSocket to `GET /v1/connections/{id}/session` on the base URL, with `Authorization: Bearer <session assertion>`. The Provider's ingress terminates TLS and routes it like any WebSocket.
- **Inside the WebSocket runs a stream multiplexer** (remotedialer, or yamux/smux). The Provider-side opens a new stream whenever client-go wants a new connection, so client-go's standard transport works unchanged: pooling, health checks, and separate connections for upgrades. See "Sessions" for why this matters more than the HTTP version.
- **The connector is an authenticating reverse proxy.** Each stream is handed to an in-process HTTP server. For each request it checks the target host against the allow-list, removes any credentials the Provider-side sent, adds a short-lived SA token for that connection, and forwards the request to the real kcp URL over TLS verified with PM's own trust.
- **The Provider-side keeps the real kcp URLs** (front-proxy, and the shard virtual-workspace URLs from endpoint slices). Only the transport's dial hook is replaced: it opens a stream on the session that serves the target host.

## Provider-side prerequisites

- **An HTTPS endpoint at the base URL from the token that supports WebSocket upgrades** and long-lived connections: idle timeouts above the session ping interval (e.g. > 60 s), and a maximum connection age the connector can reconnect after. A WAF must pass binary frames after the upgrade.
- **The connection API** (registration, status, secret rotation, session-CA updates, rebind, delete) and a session endpoint with session-assertion verification. Both ship in the Provider SDK.
- **State keyed by `connectionID`** (see "Provider-side: serving many PM instances").
- **Owner notifications and controls** in the portal: notify the token owner of every registration, rotation and rebind, and let them freeze or delete connections.

Not required: TLS passthrough, client-certificate handling, a CA of the Provider's own, a server key, cert-manager, or any knowledge of kcp's tokens, keys or CAs.

## Flow overview

1. **Get a token.** As in v2.
2. **Create the request.** As in v2.
3. **Create or reuse the `Provider`.** As in v4: an already registered `Provider` is refused, except for rebind.
4. **Register.** The connection controller generates the connection secret, writes it to the connection Secret, and calls `POST /v1/connections` with the onboarding token, the addressing, the **hash** of the secret and the **session CA** certificate. It gets back the `connectionID`. No kcp credential and no secret crosses the wire.
5. **Write the connection record.** The controller validates the base URL (egress rules) and writes the connection's `ProviderConnection` object into the provider workspace, through the connection export, which the Provider controller bound there when it created the workspace.
6. **Open sessions where they're needed.** The connector on the shard hosting the provider workspace opens a session right away, for front-proxy traffic. Connectors on shards listed in the connection's Service endpoint slices, i.e. shards with at least one consumer binding, open theirs as those shards appear. Each signs its session assertion with its shard certificate. Nothing is registered per shard.
7. **Confirm kcp is reachable.** The controller polls `GET /v1/connections/{id}` until the Provider-side reports that the front-proxy and every host it needs are covered by a session and reach kcp. Only then is the request `Ready`.
8. **Run.** The Provider-side reconciles through the sessions. There's no credential delivery and no credential refresh on the Provider-side.
9. **Rotate the connection secret**, automatically and periodically (see "Connection secret rotation"). Shard certificates renew on their own under the same pinned CA.
10. **Offboard.** Deleting the `Provider` removes its connection record, so every connector closes its sessions, which ends kcp access at once; then the controller calls `DELETE`.

Steps 6 to 9 handle every restart and outage, of any length: the connection secret doesn't expire, shard certificates are renewed by cert-manager, and the connectors mint SA tokens for themselves.

## Design

### Onboarding token

Unchanged from v4: `pm1.<base64url(Provider base URL)>.<tokenID>.<tokenSecret>`, single-use, short-lived, stored in a Secret, with the URL inside so nothing is sent to the wrong host. The portal never puts tokens in URLs and hands them over as a `kubectl create secret` command, not a manifest.

### Control plane and data plane

```
                         PM control plane                                    PM data plane
 ┌──────────────────────────────────────────────────────┐      ┌───────────────────────────────────────────┐
 │ connection controller (HA, leader election)          │      │ connector@shard-A (StatefulSet, stateless) │
 │  · ProviderOnboardingRequest / Provider reconcile    │      │  mounted: shard cert + key, own VW URL,    │
 │  · connection Secret (owner workspace)               │      │           front-proxy URL                  │
 │  · control calls to Providers (secret, rebind, …)    │      │  reads:   connection export (records,      │
 │  · aggregate status on Provider                      │ ───▶ │           claimed endpoint slices)         │
 │  · ProviderConnection records (provider workspaces,  │      │  writes:  TokenRequests only               │
 │    through the connection export)                    │      │  dials sessions, proxies to kcp            │
 │ session CA (cert-manager, KMS/Vault-backed issuer)   │ ───▶ │ connector@shard-B …  connector@shard-C …   │
 └──────────────────────────────────────────────────────┘      └───────────────────────────────────────────┘
```

- **Everything stateful sits in the control plane**: connection secrets, control calls, status, and the session CA. It's one component, so its dependencies are explicit and it can be made highly available like any controller.
- **Everything on the traffic path sits in the data plane**, which is stateless. A connector replica can be killed at any time; a new one reads its mounted certificate and the connection export and carries on. No registration, no writes, no shard list.
- **A control-plane outage never stops traffic.** It delays onboarding, rotations, rebinds and offboarding.

### Connection API (on the base URL, plain HTTPS)

There's no inner TLS any more, so the connection API is just HTTPS on the base URL. Sessions are the only long-lived connections.

```
POST /v1/connections
Authorization: Bearer <tokenID>.<tokenSecret>
{
  "idempotencyKey": "<ProviderOnboardingRequest UID>",
  "clusterID": "<logical cluster name of the provider workspace>",
  "frontProxyURL": "https://...",
  "secretHash": "sha256:<hex>",
  "sessionCAs": [ "<PEM certificate of the session CA>" ]
}
→ 201 { "connectionID": "..." }

GET /v1/connections/{id}
Authorization: Bearer <connection secret>
→ 200 { "requiredHosts": { "<host:port>": { "sessions": 1, "kcpReachable": true, "lastError": "" } },
        "pendingSecret": false, "pendingRebind": null }

PUT /v1/connections/{id}/secret                  # connection secret rotation
Authorization: Bearer <current connection secret>
{ "secretHash": "sha256:<hex of the NEW secret>" }
→ 202                                            # pending until the new secret is used

PUT /v1/connections/{id}/session-cas             # session CA rollover (rare)
Authorization: Bearer <connection secret>
{ "sessionCAs": [ "<PEM certificate>", ... ] }   # current CA, plus the next generation during a rollover
→ 204

GET /v1/connections/{id}/session                 (WebSocket upgrade; stream multiplexer inside)
Authorization: Bearer <session assertion>
  header: { "alg": "ES256", "x5c": [ "<shard certificate>", "<intermediate, if any>" ] }
  claims: { "aud": "<base URL>/v1/connections/{id}/session",
            "cluster": "<clusterID of the provider workspace the connector serves this session for>",
            "sub": "<opaque session-group ID>",  # e.g. a hash of shard and connectionID; audit only
            "iat": …, "exp": iat + ≤ 5m, "jti": "<random>" }

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
- **Two kinds of credential, two planes.**
  - The **connection secret** (bearer) only authenticates control calls, which are rare, and only the connection controller uses it. Bearer is fine here because of the trust boundary: the only hop that sees it is the Provider's own edge, and it grants no kcp access.
  - **Session assertions** (signed, minutes) only authenticate sessions. No reusable credential crosses the edge on the session path.
- **Status is per host, not per shard.** `requiredHosts` lists the front-proxy and the hosts from the endpoint slices the Provider-side actually uses, with whether a live session covers each and reaches kcp. The Provider-side knows which hosts it needs; nobody has to tell it the shard layout.

### Session credentials: the session CA

Each connection pins the **session CA certificate** at registration. Each shard's connector gets a short-lived **shard certificate** from it, directly or through at most one intermediate, mounted read-only and renewed by cert-manager with a new key each time. Session assertions carry the shard certificate, and the intermediate if any, in `x5c`.

**Verifying a session assertion** (in the Provider SDK):
1. the chain in `x5c` verifies with the standard verifier (`x509.Verify`) against **only** the connection's pinned session CA certificates as roots, with at most one intermediate. That checks every certificate's validity and the CA constraints (basic constraints, key usage, name constraints, if used). The shard certificate must also be allowed for signing (key usage `digitalSignature`, plus whatever EKU the session CA uses to mark shard certificates). Two details the SDK must get right: Go's verifier requires EKU `serverAuth` unless `VerifyOptions.KeyUsages` says otherwise, so it has to pass the shard certificates' EKU explicitly; and "at most one intermediate" has to be enforced, by the root's path-length constraint or by checking the chain length;
2. the JWT signature verifies with the shard certificate's public key;
3. `aud` is exactly this connection's session URL; `exp` and `iat` are within a clock-skew tolerance, and `exp - iat` is at most 5 minutes; the `jti` hasn't been seen within the validity window;
4. `cluster` equals the `clusterID` stored for this connection at registration. The connector takes it from the logical cluster of the connection record it acts on, and uses that workspace's SA for the session's requests. Records can only be written by PM (see "Connector policy"), so this is defense in depth: even a PM-side bug that mixed up records couldn't make a connector serve one connection's sessions with another provider workspace's SA;
5. the session serves the hosts named in the shard certificate (URI SANs listing its virtual-workspace host), plus the connection's front-proxy, which every session may carry.

**Why a CA, after v4 rejected a shared PM CA as the anchor:**
- **It only authorizes sessions.** A forged session can feed a Provider fake data for that connection. It can't make control calls, which still need each connection's own secret, and it gets no kcp access. In v4 a shared CA would have been the whole connection identity.
- **It concentrates nothing new.** The control plane already holds every connection secret; anything that could register a key on every connection would already be equivalent to a CA. Keeping the CA key with the control plane, preferably in a KMS or Vault, doesn't widen what a control-plane compromise yields.
- **Short certificates replace revocation.** A compromised shard's certificate dies within hours, and the CA stops issuing to it. No per-connection fan-out is needed.
- **Audience binding makes one certificate safe for every connection.** An assertion given to one Provider is useless at any other.

**Pinning immutable CA generations:**
- **A CA is a keypair plus a certificate.** The private key signs shard certificates and never leaves PM. The certificate wraps the public key with a name, a validity period and constraints (basic constraints, key usage, name constraints). Providers pin the **certificate**, so they get all of it, and verify with standard library code.
- **Each session CA is an immutable generation.** Its lifetime is well beyond the planned rollover interval (e.g. a 5-year CA replaced yearly), and it's never renewed. Changing anything about the CA means a **new** CA, added alongside the old one with an overlap. So the pinned certificate never changes under a connection.
- **The pinned certificate's expiry is a hard deadline.** A generation that was never rolled over simply stops being trusted, including at Providers that couldn't be reached when it was replaced. Trust in a forgotten CA can't outlive it.
- **cert-manager must never renew the CA by itself.** With `rotationPolicy: Always` (the default in recent cert-manager versions), a renewal silently replaces the key: new shard certificates are signed by a key no Provider has pinned, and new sessions fail everywhere. It's the same trap as kcp's per-shard SA keys in v3 (v4 finding 4). A cert-manager-managed session CA therefore sets `rotationPolicy: Never` as a safety net: should it be renewed anyway, the key stays the same, and the old pinned certificate keeps verifying until it expires itself (cert-manager renews at about two-thirds of the lifetime, so there's ample time to pin the renewed one). A Vault- or KMS-backed issuer keeps the key out of Kubernetes Secrets and makes cert-manager's rotation policy irrelevant for the CA.
- **An optional intermediate** lets PM rotate the issuing key without involving Providers: they pin the root generation, and the intermediate changes underneath it. Nothing revokes an intermediate, so a pinned root trusts it until it expires: intermediates must be short-lived (e.g. weeks), not long-lived like the root.
- **Leaves are the opposite.** Shard certificates are nobody's pin, so they keep `rotationPolicy: Always` and a short lifetime: a new key on every renewal.
- **The cost of a long-lived CA key** is exposure over time. It's bounded by the rollover schedule, by the pinned certificate's expiry, and by keeping the key in Vault or a KMS.

**What it removes**, compared with keys registered per shard: no registration call per shard or per key rotation, no self-registration objects, no control-plane involvement when shards are added or credentials renew, and no state in the connector. Calls to Providers happen only for connection lifecycle and the rare rollover to a new CA generation.

**Options considered:**

| | Per-shard keys registered per connection | **Session CA with short-lived shard certificates** (chosen) | One PM key everywhere |
|---|---|---|---|
| Connector state | mounted key | mounted, auto-renewed certificate | mounted key |
| Calls to Providers when a shard is added | one per connection | none | none |
| Calls to Providers when shard credentials rotate | two per connection | none | per connection, when the key rotates |
| Blast radius of one shard's credential | that shard's sessions, until removed from every connection | that shard's sessions, until the certificate expires (hours) | every shard's sessions |
| Anchor | the control plane (registers keys everywhere) | the CA key | the key itself |

### Sessions: a stream multiplexer over a WebSocket

**What really differs from a normal client-go connection.** A regular client-go client already sends all requests and watches to a host over one HTTP/2 TCP connection. It already accepts head-of-line blocking, shared flow control, and losing every watch (and relisting) when that connection dies. kube-apiserver even sends GOAWAY on purpose (`--goaway-chance`) to rebalance clients. None of that is new here. Two things are:

1. **The client can't open connections.** client-go normally dials whenever it needs to: a new host, a full or dead connection, or an upgrade (`exec` uses a separate HTTP/1.1 connection). Behind NAT, only the connector can create sessions. A design that hands the Provider-side a fixed set of connections (e.g. HTTP/2 directly on the WebSocket) has to replace client-go's pooling, health checks and upgrade handling with custom code.
2. **The path isn't under PM's control.** Traffic crosses the internet and the Provider's L7 edge, with its own idle timeouts, maximum connection age and WAF rules, and higher round-trip times than in-cluster clients.

**A multiplexer fixes the first.** Each stream is a connection the Provider-side opens on demand, so client-go's standard machinery works unchanged. The HTTP version on each stream then hardly matters: HTTP/1.1 is the natural default over plain streams; HTTP/2 without TLS (h2c) would also work.

- **Provider-side**: the session handler verifies the session assertion, accepts the WebSocket and runs the multiplexer on it. The session is registered under `{id}` with the hosts from its shard certificate; deleting the connection closes it. The connection's `http.Transport` uses a dial hook that opens a stream on a session serving the target host:

  ```go
  tr := &http.Transport{
      // Called for https:// URLs. The stream isn't TLS, so the transport
      // speaks plain HTTP/1.1 on it; the real kcp URL stays in the request.
      DialTLSContext: func(ctx context.Context, _, addr string) (net.Conn, error) {
          return sessions.OpenStream(ctx, conn.ID, addr) // picks a session serving addr; the connector checks it again
      },
  }
  ```

- **PM-side**: the connector opens the WebSocket and runs the other end of the multiplexer. Every stream the Provider-side opens is handed to an in-process `http.Server`, whose handler is the reverse proxy below (`httputil.ReverseProxy`, which also proxies upgrades).
- **This is remotedialer's designed direction**: the side that accepted the WebSocket (the Provider-side) dials through the side that opened it (the connector). Instead of dialling the target, the connector's dialer hook routes the stream to the local HTTP server.
- **Routing by host.** A shard's session serves that shard's virtual-workspace host (from its certificate) and the front-proxy. Front-proxy traffic may go over any session; the front-proxy routes it to the right shard anyway. Endpoint-slice URLs are per shard, so the Provider's code sees ordinary URLs.
- **More than one session per shard** is allowed, for connector replicas during handovers and to spread load. The dial hook spreads new streams across sessions serving the same host.
- **Liveness**: WebSocket pings at the outer level, so ingresses see traffic, and multiplexer keepalives inside. client-go's own HTTP health checks work as usual.
- **Graceful replacement.** The connector replaces sessions before the Provider's edge cuts them at its maximum connection age: it opens a new session, then tells the Provider-side to stop opening streams on the old one (the multiplexer's go-away, where available) and closes it once in-flight requests finish. An abrupt cut by the edge still drops all watches on that session at once, as with any apiserver connection.
- **Why not a dial proxy to kcp** (remotedialer or Konnectivity as used in v1 to v4): there the stream carried opaque TLS from the Provider-side to kcp, so the Provider-side had to hold kcp credentials and trust kcp's CA. In v5 the same kind of stream ends at the connector's HTTP server, which owns both.

### The connector as an authenticating proxy

For every request arriving over a session of connection `{id}`:

1. **Check the target.** The request's host must be on the allow-list: this shard's virtual-workspace host and the front-proxy host, both **from the connector's deployment configuration**. Nothing else, and nothing a Provider or an owner can influence. Resolve once, check every address, dial those exact addresses; refuse link-local, multicast and cloud-metadata addresses.
2. **Strip credentials and identity headers**: `Authorization`, `Impersonate-*`, `Proxy-*`, cookies and hop-by-hop headers. Otherwise the Provider-side could ask kcp to act as someone else, using the connector's token.
3. **Add the SA's identity** for connection `{id}`: a bound token from TokenRequest, minted by the connector, cached in memory, refreshed at half its TTL, never written anywhere and never sent to the Provider-side. Impersonating the SA instead was checked and rejected: it works for the provider workspace but not for the APIExport virtual workspace (observation 16).
4. **Forward** to the real URL over TLS, verified against the CA bundle PM's own components use for kcp. Rotating kcp's CAs needs no coordination: if PM trusts its kcp, the connector does.
5. **Log and limit** per connection: method, path, status, duration; rate limits, concurrent-stream limits and header-size limits per connection, so one Provider can't exhaust the connector for others. The connector is in the same position as kcp's front-proxy towards its clients, and needs the same defences.

- **RBAC is unchanged**: kcp sees the connection's SA, with the RBAC the Provider controller created (RFC 006), admin in the provider workspace. The connector doesn't widen it; it narrows it by its path restrictions (see "Connector implementation") and the denylist in "Connector policy". The Provider controller also grants the connectors' own identity `create` on `serviceaccounts/token` for that SA; that's the connector minting tokens, which the denylist doesn't affect.
- **Upgrade requests** (`exec`, `attach`, `port-forward`) are refused by the connector policy. The transport could carry them (HTTP/1.1 upgrades on a stream), so allowing them later is a policy change, not a design change.
- **Egress to Providers.** Connectors dial base URLs from connection records. The controller validates them before writing a record, and connectors enforce the same rules when dialling: `https` only, public addresses only, resolve once and check. Records are PM-written only (see "Connector policy"), so this is defense in depth against a bad URL reaching PM's internal network.

### Connector policy: what an admin Provider can't do

The Provider is admin in its provider workspace (RFC 006) and free to deploy whatever it needs, so PM doesn't restrict its RBAC there. But in v5 the connector is the Provider's **only path into kcp**: the Provider-side holds no kcp credentials, and every request it makes passes through the connector's handler chain. The connector's authorizer therefore applies a small **denylist** on top of the Provider's RBAC, for the connection principal:

| Denied | Why |
|---|---|
| Any write (`create`, `update`, `patch`, `delete`, `deletecollection`) to `providerconnections` | Records stay PM-written: the Provider can't redirect connectors or take another connection's identity |
| `update`, `patch` and `delete` on the connection export's APIBinding (a fixed name, set by the Provider controller) | The Provider can't drop the binding or reject its permission claim |
| Writes to `apiexportendpointslices/status` | kcp maintains that status; forging it is the only way to make connectors open sessions on shards without consumers |
| `create` on `serviceaccounts/token` | No TokenRequest: the Provider never gets a kcp credential it could keep |
| `create` on `certificatesigningrequests`, and their approval | Closes the other route to a credential, if kcp signs CSRs |
| Upgrade requests (`exec`, `attach`, `port-forward`) | Not needed to reconcile an APIExport |
| (Policy choice) `create` on `workspaces` below the provider workspace | Only if PM wants to cap what a Provider can create there |

Everything else, the Provider's own exports, schemas, endpoint-slice specs, RBAC and arbitrary resources, passes through unchanged, subject to its admin RBAC.

**The invariant this rests on: the connector is the Provider's only path to kcp.** That makes these requirements:
- **No static kubeconfig for remote Providers**: RFC 006's kubeconfig Secret is not generated or handed out for them.
- **No token minting through the proxy**: the TokenRequest and CSR rules above, plus legacy SA-token Secrets if kcp issues them (see "Open items").
- **`direct` mode, if ever added, goes through the same connector virtual workspace**, so it hits the same authorizer.
- **PM gives the Provider no second credential** for its workspace. Role bindings the Provider creates for others are its own business.

**Implementation notes:**
- The authorizer needs the original request's group, resource, subresource, verb, name and cluster, for both kcp path shapes the Provider uses: `/clusters/<id>/…` through the front-proxy and `/services/apiexport/…` through a shard's virtual workspace. The generic request-info parser doesn't understand kcp's cluster prefix, so use kcp's resolver or parse both shapes explicitly.
- The denylist is the security guarantee, so it needs thorough tests, including every verb and both path shapes. Denied attempts are audited.
- The owner and every other kcp user reach the provider workspace through the front-proxy with their own credentials, outside this policy. The owner has no rights in the provider workspace; only PM's own components and the Provider (admin, through the connector) do.

### Connector implementation: a virtual workspace

The connector has two halves. Only one is request-shaped, and that one maps directly onto kcp's virtual workspace framework.

**What already exists** (checked on 2026-10-08/09):
- **PM's virtual-workspaces server** (`platform-mesh/services/virtual-workspaces`) is built on `kcp-dev/virtual-workspace-framework` and serves `/services/marketplace` and `/services/contentconfigurations`. It builds its own authenticator union (`cmd/start.go`: `union.New(authentication.New(clientCfg), …)`), so a new authenticator needs no change in kcp. It's deployed as a plain Helm Deployment (`pm-helm-charts/charts/virtual-workspaces`), a singleton reached through the front-proxy's `additionalPathMappings`, talking to kcp through the front-proxy as `kcp-admin` (`system:kcp:admin`), with no cache-server access. The connector reuses its code, not its deployment.
- **kcp runs its virtual workspaces per shard.** Each shard's `spec.virtualWorkspaceURL` points at that shard's own virtual-workspace server, which kcp-operator exposes under the shard's external, SNI-routed hostname or an external `VirtualWorkspace` (`kcp-operator/internal/resources/compiledshard/deployment.go`, `--shard-virtual-workspace-url`). APIExportEndpointSlice URLs are built from it (`kcp/pkg/reconciler/apis/apiexportendpointsliceurls`), so endpoint traffic goes straight to each shard's virtual workspace, bypassing the front-proxy.
- **kcp's initializing-workspaces virtual workspace already does the proxy half.** Its workspace-content handler (`kcp/pkg/virtual/initializingworkspaces/builder/build.go`, a `handler.VirtualWorkspace` with a `HandlerFactory`) authorizes the caller, strips `Authorization` and `Impersonate-*` headers, impersonates a chosen identity and reverse-proxies to the shard, watches included (`kcp/pkg/virtual/shared/proxy.go`, `ServeProxy`).
- **kcp-operator's `VirtualWorkspace` resource** (v0.9+) deploys an external virtual-workspace server: Deployment, Service, certificates, a kubeconfig identity, and a target of the root shard or one shard (`shardRef`). PM's charts already use it for `kcp-access-vw`. When an external cache server is configured, which PM does, it also mounts the cache-server kubeconfig and client certificate (`kcp-operator/internal/resources/compiledvirtualworkspace/deployment.go`), see observation 18.

**Design:**

```
            connector virtual-workspace server, one per shard (one process each, stateless)
 ┌──────────────────────────────────────────────────────────────────────────────┐
 │ session controller (post-start hook)    in-process listener per connection   │
 │   watches the connection export,          ConnContext: connectionID          │
 │   signs session assertions with the                                          │
 │   mounted shard certificate                                                  │
 │   dials WebSocket ──▶ Provider            │                                  │
 │   multiplexer: streams ───────────────────┘                                  │
 │                                           ▼                                  │
 │   full handler chain (authn → authz → audit → timeouts → …)                  │
 │     authn: session authenticator reads connectionID from context             │
 │     path:  /services/providerconnections/<id>/<original path>, Host kept      │
 │                                           ▼                                  │
 │   providerconnections VW handler: check Host, strip auth headers,             │
 │   add SA identity, reverse-proxy ──▶ this shard's VW (local) / front-proxy    │
 └──────────────────────────────────────────────────────────────────────────────┘
```

1. **Session controller.** Not a virtual workspace: a controller in the same process, started from a post-start hook, the way kcp's APIExport virtual workspace runs its controllers. It watches the connection export (records and claimed endpoint slices) and decides which sessions this shard needs (see "Per-shard connectors"), signs session assertions with the mounted shard certificate, dials the sessions of the connections this replica owns (see "Connector replicas") and accepts multiplexer streams. It serves each stream with an in-process `http.Server` whose handler is the server's **full handler chain**, not the virtual workspace's handler directly, so authentication, authorization, audit and timeouts all apply. `ConnContext` tags each stream with its `connectionID`, and the server prefixes the path with `/services/providerconnections/<id>`, keeping the `Host`.
2. **Session authenticator.** A new member of the authenticator union maps the `connectionID` in the request context to the connection principal. The value is set only on in-process listeners, so nothing arriving over the network can forge it, and requests from the front-proxy never carry it.
3. **The `providerconnections` virtual workspace.** A `handler.VirtualWorkspace` following the initializing-workspaces pattern:
   - its **authorizer** lets the connection principal reach only its own connection's paths: its provider workspace through the front-proxy, and the APIExport virtual-workspace endpoints of the exports in that workspace. It also enforces the denylist in "Connector policy";
   - its **handler** checks the `Host` against the allow-list, strips client credentials (`ServeProxy`'s header list plus `Proxy-*` and cookies), adds the SA's identity, and reverse-proxies to the real URL.
4. **No status writes.** Per-session health is in metrics. Connection health reaches the `Provider` through the controller, which asks the Provider-side (see "Connection controller").

**What the connector needs at runtime**: mounted, its shard certificate and key, its own shard's virtual-workspace URL and the front-proxy URL; read-only, `apiexports/content` on the connection export (records and claimed endpoint slices); write, TokenRequests. No `Shard` objects, no `/services/admin`, no shard-wide reads, no cache access, no persistent state.

**What this gives:**
- **Placement**: one connector virtual-workspace server per shard, deployed through a kcp-operator `VirtualWorkspace` with `shardRef`, next to that shard's own virtual workspaces.
- **A full serving stack**: audit (each request recorded with the connection principal and the SA it acts as), long-running request detection, timeouts, max-in-flight limits, panic recovery and `apiserver_request_*` metrics, instead of a hand-written proxy.
- **Request policy in one place**: the virtual workspace's authorizer, instead of ad-hoc checks in proxy code.

**Identity: SA tokens, not impersonation.** The connector mints TokenRequest tokens (step 3 above) for all traffic. Impersonating the SA looked attractive here, since the initializing-workspaces pattern impersonates, but it only works for half of the traffic (observation 16):
- **Provider workspace (front-proxy, then shard): works.** Needs `access` on `/` and `impersonate` on `serviceaccounts` with `resourceNames: [<sa>]` in the provider workspace; no user extra.
- **APIExport virtual workspace (standalone server): doesn't work.** Per-cluster requests are denied, and the grant that would allow wildcard requests lets the connector impersonate anyone.

Tokens are needed for the APIExport virtual workspace anyway, so a mix would add RBAC and code paths without removing anything.

**Caveats:**
- **Timeouts.** The generic handler chain applies a timeout (60 s by default) to requests it doesn't classify as long-running. Large proxied LISTs may exceed it; the long-running check and timeout must be configured for proxied traffic.
- **Hop limit.** Proxying into the APIExport virtual workspace counts toward kcp's `X-Kcp-Virtual-Resource-Hops` limit (maximum 4).
- **`ServeProxy` is kcp-internal** (`kcp/pkg/virtual/shared`), not part of the framework module. Copy it (about 30 lines) or propose moving it into the framework.
- **Cache-server credentials come along** when deployed as a kcp-operator `VirtualWorkspace` in PM's setup (observation 18). The connector doesn't need them; see "Open items".

### Per-shard connectors

One connector per kcp shard, following kcp's own model for virtual workspaces. Each one knows which shard it serves from how it was deployed, nothing else.

```
Provider ingress
   ▲  ▲  ▲   at most one session per (connection, shard), only where needed
   │  │  └── connector@shard-C ──▶ shard-C APIExport virtual workspace (local)   consumers on C
   │  └───── connector@shard-B ──▶ shard-B APIExport virtual workspace (local)   consumers on B
   └──────── connector@shard-A ──▶ front-proxy                                   provider workspace on A
```

The three kinds of bindings involved, and which one decides what:

| | (1) Providers API | (2) Connection export | (3) Service APIs |
|---|---|---|---|
| Carries | `Provider`, `ProviderOnboardingRequest`; the connection Secret sits next to them | one `ProviderConnection` record per connection | the Provider's own services |
| Exported by | PM | PM | the Provider-side, from `:root:providers:<Provider name>-<Suffix>` |
| Bound in | the owner's workspace, `:root:orgs:<Org>:<User>` | each provider workspace, by the Provider controller | consumers' workspaces, on any shard |
| Written by | the owner (request), PM (status) | PM only, through the export. The Provider is admin in its workspace, but the connector policy denies it writes to records | the Provider (exports), consumers (bindings) |
| Read by | the connection controller | every shard's connector, through the export's endpoint slice | kcp, which lists the shards with bindings in each Service endpoint slice; connectors read those slices through (2)'s permission claim |
| Decides | nothing about sessions | where the front-proxy session lives (the shard hosting the record) | where endpoint sessions live (shards listed in the slices) |

**Load.** In today's direct model, a Provider's endpoint traffic (wildcard LIST and WATCH per resource, per-workspace reads and writes) goes straight to each shard's virtual workspace; only its workspace traffic (the endpoint slice and a few objects) uses the front-proxy. v5 sends the same requests to the same targets with the same SA identity, so kcp's load doesn't change; connections to kcp now come pooled from the connectors instead of from each Provider. What's added is a hop: every request and response byte passes a connector, which parses HTTP, authorizes, audits, injects the token and copies the response, including watch streams held for hours. Each watch holds two streams on the connector, roughly a few goroutines and a 32 KB copy buffer each. Per shard, this is the same shape as the virtual-workspace load kcp already carries, plus the per-request L7 overhead. Any NAT-traversing design adds a hop; v4's is an L4 byte copy (see the [comparison](./xxx-notes-011-provider-onboarding-flow-improvements-v4-v5-comparison.md)).

**What per-shard gives:**
- **Load scales with shards**, as virtual-workspace servers already do.
- **Locality.** Each connector forwards endpoint traffic to the virtual workspace next to it.
- **Failure isolation.** A connector or session failure on shard B only stops shard B's endpoint traffic and relists only shard B, as when one shard's virtual workspace has trouble today. Front-proxy traffic can move to any other session of the connection.
- **Tiny allow-lists**, from deployment configuration: its own virtual-workspace host and the front-proxy.
- **Data next to what it describes.** Each connection's record lives in its provider workspace, on that workspace's shard, and kcp itself keeps the list of shards with consumers in each endpoint slice.

**Design consequences:**
1. **Which sessions exist.** A shard's connector keeps a session for a connection when either:
   - **the connection's record came from its own shard's virtual-workspace endpoint** of the connection export. Then the provider workspace is hosted on this shard, and this connector carries the connection's front-proxy session, from the start and even with no consumers yet; or
   - **one of the connection's Service endpoint slices lists its own shard's virtual-workspace URL** (known from its deployment). kcp lists a shard there exactly when it has at least one binding to the export, so this is the shard's endpoint traffic. Only slices in the provider workspace that point at an export **in that same workspace** count. Otherwise a Provider could create slices for someone else's export and make connectors open sessions on shards it has no consumers on.

   Sessions therefore follow real usage: one for the hosting shard, plus one per shard with consumers of the connection's own exports. No idle sessions on shards a connection doesn't use, and nothing the owner or the Provider can steer: the record is PM's, and consumers create the bindings. This assumes the Provider keeps its Service endpoint slices in its provider workspace. RFC 006 leaves APIExport bootstrap out of scope, so that's a requirement to state, not something it already says (see "Open items").
2. **Shards come and go with their deployments.** A new shard's connector gets a shard certificate from the session CA and starts watching the connection export. When kcp adds the shard to a connection's endpoint slices, after the first consumer binding there, the connector dials. A removed shard takes its connector with it, and its endpoint slices stop listing it anyway. Nobody is told or registered anything. "Is there a connector for every shard?" is a deployment-health question (kcp-operator status, alerts), not something checked at runtime.
3. **Discovery: the connection export.** kcp's per-shard virtual workspaces only see their own shard plus the cache server, while `Provider` objects live in owners' workspaces on any shard. Instead of reading those, each connector consumes PM's **connection export** like any multi-shard controller: it reads the export's APIExportEndpointSlice through the front-proxy, then list/watches through every shard virtual-workspace URL in it (multicluster-runtime's endpoint-slice provider):
   - **`ProviderConnection` records**: every object arrives with its logical cluster, which is the provider workspace's `clusterID`. So the `clusterID → (connectionID, base URL)` mapping is just the informer's index, and which endpoint delivered the record tells the connector which shard hosts the workspace;
   - **the connections' Service `apiexportendpointslices`**, read-only, through the export's **permission claim**. That's how the connector learns which shards have consumers, without any shard-wide read of APIBindings.

   The records are written only by PM through the export. The Provider stays admin in its workspace, as in RFC 006, but it reaches kcp only through the connector, whose policy denies writes to records, to the connection export's binding and to endpoint-slice status (see "Connector policy"). So neither the owner nor the Provider can change the URL connectors dial, drop the binding or forge which shards have consumers. The owner has no rights in the provider workspace at all. The export claims nothing else in the provider workspace. An outage of one shard's virtual workspace only freezes the records and slices from that shard; informers keep the last state.
4. **SA tokens.** Each shard's connector mints tokens for the connection's SA through TokenRequest via the front-proxy, since the SA lives in the provider workspace on one shard. All shards trust all shards' signing keys (v4 finding 3), so those tokens work against every shard's virtual workspace.
5. **Front-proxy traffic on any session.** Workspace traffic goes through the front-proxy today anyway, and the front-proxy routes it to the right shard, so every session may carry it. The hosting shard's session (point 1) guarantees there's always one, even before any consumer binds; that's what lets the Provider watch its own exports and slices from the start. Which shard that is comes from the connection export itself, not from any lookup.
6. **Provider-side routing.** The dial hook picks a session by target host: the front-proxy can use any session; a shard's virtual-workspace host uses that shard's sessions. Because sessions open by the same rule that puts shards into endpoint slices, a URL can briefly appear in a slice before its session is up; client-go retries. The Provider-side multi-replica problem gets more pressing: sessions for different shards can land on different Provider replicas, while a replica running the endpoint-slice providers needs all of a connection's hosts. Lease routing between replicas, or reconciliation sharded by connection with forwarding.
7. **Readiness without topology.** The Provider-side reports, per host it actually needs, whether a session covers it and reaches kcp (`GET /v1/connections/{id}`). The controller turns that into one condition. Nobody needs the shard list. At onboarding, before any consumer binds, that's just the front-proxy.
8. **Deployment and `direct` mode.**
   - One kcp-operator `VirtualWorkspace` per shard (`shardRef`), plus a cert-manager `Certificate` for its shard certificate from the session CA, created by the same chart or operator. PM's marketplace and content-configuration virtual workspaces stay on their singleton server.
   - The connection export lives in a PM workspace; the Provider controller binds it (accepting its one permission claim) in every provider workspace it creates (RFC 006).
   - **`direct` mode should use per-shard URLs, not the front-proxy.** The front-proxy routes only `/clusters/` by shard; every other path mapping, `/services/...` included, goes to one fixed backend (`kcp/pkg/proxy/mapping.go`: `// TODO: handle virtual workspace apiservers per shard`). Putting the `clusterID` in the URL (`/services/providerconnections/clusters/<clusterID>/…`) would let the front-proxy's cluster lookup find the provider workspace's shard (`kcp/pkg/proxy/lookup/lookup.go` already extracts clusters from `/services/<name>/clusters/<cluster>/…`), but it wouldn't choose the backend without that TODO being done. And it would only ever reach one shard, while endpoint traffic needs every shard's connector, including for wildcard requests that name no cluster. Instead, expose each shard's connector under its own external hostname, as kcp-operator already does for per-shard virtual workspaces (`VirtualWorkspace` `external`), and let the Provider-side use one URL per shard, mirroring its endpoint slices. The `clusterID` in the path is still useful as a consistency check and audit field.

### Connector replicas

Within one shard, the connector runs several replicas and shares the connections among them.

- **StatefulSet, not Deployment.** Stable ordinals mean a restarted replica comes back as the same member and keeps its connections. With a Deployment, a new pod name moves the restarted pod's connections to other replicas and back again.
- **Rendezvous hashing of `connectionID` over the live replicas.** Membership comes from the headless Service's EndpointSlices (or a Lease per replica). Scaling up or down moves only about 1/R of the connections.
- **No agreement needed.** Sessions aren't exclusive, so a short overlap where two replicas both own a connection is harmless: both sign assertions and both connect. A short gap where nobody owns it pauses that connection's endpoint traffic on this shard until views converge. No Lease per connection.
- **No registration of any kind.** Every replica signs assertions with the same mounted shard certificate; the opaque `sub` is for audit. Restarts, scaling and rollouts involve neither the control plane nor the Provider's control API, and leave nothing on the Provider-side to clean up.
- **Hand over before cutting off.** On graceful changes (scale-down, rollout), the new owner connects first; the old owner sends the multiplexer's go-away and drains its streams. A crash is break-then-make by nature: the moved connections' watches relist against this shard's virtual workspace, as when a virtual-workspace pod restarts today.
- **SA tokens** are minted only by the replica that owns the connection.
- **Optional, with a KMS-backed issuer: per-replica certificates.** Each replica generates an in-memory key at start and gets a short-lived certificate for it from the session CA, so no private key is mounted at all. Costs an issuing round trip per replica start.

### Connection controller

The control plane. One stateful component, run like any controller: HA replicas with leader election, or connections shared among replicas by hashing if one leader isn't enough. It could live in pm-operator or in its own Deployment.

**What it does:**

| Work | When | Calls to the Provider |
|---|---|---|
| Register the connection (with the session CA), write its `ProviderConnection` record through the connection export | once per connection | `POST /v1/connections` |
| Confirm readiness, then keep rebinds cancelled and health current | every few minutes, per connection | one `GET` |
| Rotate the connection secret | rotation period, per connection | `PUT /secret` plus one confirming call |
| Roll over to a new session CA generation | rare (planned rollover interval) | two `PUT /session-cas` per connection, spread over the overlap |
| Rebind, `DELETE` | rare, per connection | one call each |

- **Nothing on this list is on the traffic path.** A controller outage delays onboarding, rotations, rebinds and offboarding; sessions on every shard, including replica restarts and new shards, keep working.
- **Load is small.** With 1,000 connections and a 5-minute poll: about 3–4 `GET`s per second in steady state. Shard changes cost nothing. The calls go to external Provider endpoints, so they need jitter, a rate limit per Provider and backoff.
- **It reports aggregate status only** on the `Provider`: `Registered`, `SessionsConnected`, `KcpReachable`, `SecretCurrent`, `Rebinding`. Per-shard detail stays in metrics.
- **Rolling over to a new session CA generation** follows the usual overlap: pin the next generation on every connection, let cert-manager issue shard certificates from it, unpin the old one. It's the one operation that fans out to every connection, and it's scheduled, not urgent. Rotating an intermediate, if PM uses one, doesn't reach Providers.
- **Responding to a compromised shard**: stop issuing certificates to it; its current certificate expires within hours. Rolling over the session CA is only needed if the root's key itself is in doubt; a compromised intermediate is replaced and expires on its own, within its short lifetime.

### Provider-side: serving many PM instances

The principle is the same in every version (see v4, "Provider-side: serving many PM instances"): the SDK hands the Provider a client config per connection and tells it when connections come and go. The Provider's operator code, which already knows the endpoint slices of the exports it owns, creates one multicluster-runtime endpoint-slice provider per slice it's interested in, with cluster names prefixed by the `connectionID`. Only each connection's client config differs between versions.

- **v5's client config per connection.** The SDK gives each connection an `http.Transport` whose dial hook opens streams on that connection's sessions (see "Sessions"):

  ```go
  cfg := &rest.Config{
      Host:      conn.FrontProxyURL,     // real URL; the connector routes on it
      Transport: sdk.Transport(conn.ID), // no TLS options: client-go refuses both
  }
  ```

  multicluster-runtime derives per-endpoint configs (shard virtual-workspace URLs) by changing `Host`; they inherit the `Transport`, so every request for that connection goes over its sessions.
- **Rebinds** don't need the Provider's code to do anything: a rebound connection keeps its `connectionID`, so its providers, clusters and state carry over.
- **No credentials to manage per connection.** Compared to v4, there's no token, CA bundle or client rebuild on rotation; a connection is just a set of sessions.

### Connection secret rotation

Fully automatic, with no owner involvement, done by the connection controller:

1. The controller generates a new secret and writes it to the connection Secret as `pendingSecret` **before** sending anything, so a crash can't lose it.
2. It calls `PUT /secret` with the new hash, authenticated with the current secret. The Provider-side stores it as pending; both secrets are accepted.
3. The controller authenticates once with the new secret (e.g. `GET`). The Provider-side **promotes** it and invalidates the old one.
4. The controller moves `pendingSecret` to `secret`.

- **Sessions are unaffected**: they never used the connection secret.
- **Lost response or crash**: on restart the controller tries `pendingSecret` first, then `secret`, and resumes at the matching step. A pending hash that's never used expires (e.g. after 24h).
- **When**: periodically (e.g. every 30 days), on demand (an annotation on the `Provider`), and right after any suspected leak.
- **No owner-supplied key material**, as in v4: the secret is always generated by the controller. An earlier revision of v4 let owners supply their own CA, which is what made its rotation manual (observation 13).

### Recovery: rebind

v4 has no way back after a lost connection key without a recent backup: a new connection, with no Provider-side state carried over. v5 adds a recovery path that an outsider can't use quietly, modelled on account recovery with a waiting period:

1. The owner creates a `ProviderOnboardingRequest` for a `Provider` that is registered but whose connection Secret is gone. The controller generates a new secret and calls `POST /rebind` with a fresh onboarding token and the new hash.
2. The Provider-side accepts it only if the token's owner is the connection's owner. The Provider-side's connection entry already fixes the addressing, so the request sends none. It then starts a **waiting period** (e.g. 72h) and notifies the owner through every channel.
3. **Any successful authentication with the current connection secret during the waiting period cancels the rebind.** The controller's periodic `GET` does that, so a rebind against a connection whose secret still exists never completes. Sessions don't count: they don't prove the connection secret is still held, and they keep working through a rebind.
4. After the waiting period, the Provider-side swaps the hash. The connection keeps its `connectionID` and all its Provider-side state.

- **A stolen onboarding token or a compromised portal account** can start a rebind, but the controller cancels it automatically, and the owner is notified. Unlike v2 and v3, a token alone never takes over a connection.
- **A leaked secret is a rotation, not a rebind.** If the controller can still authenticate, it rotates.
- **Remaining gap**: a thief with the secret who rotates it locks the owner out, and can then also cancel the owner's rebinds. That's owner compromise under the trust boundary; the owner freezes or deletes the connection in the portal. See "Open items".

### What each credential exposes

| Credential | Lifetime | Stored at | If stolen |
|---|---|---|---|
| Onboarding token | minutes, single-use | PM-side Secret until spent; hash on the Provider-side | The thief can register **their own** new connection under the owner's portal account (billing, quotas), or start a rebind that the controller cancels. No access to existing connections or their state. The owner is notified. |
| Provider portal account | long-lived | Provider-side | Same as issuing tokens, plus whatever the portal allows (deleting or freezing connections). Can't take over a live connection. |
| Connection secret | long-lived, rotated automatically | Owner's workspace (connection Secret); hash on the Provider-side | **Owner compromise, control calls only.** The thief can replace the session CA for that connection (and then open sessions answering with fake data), rotate the secret to lock the owner out, or delete the connection. **No kcp access**: the secret authenticates the PM-side to the Provider, not the other way round. |
| Shard certificate key | hours (certificate lifetime) | Mounted in that shard's connector, outside kcp (or a KMS) | Opening sessions on that shard for every connection and answering their requests with fake data, until the certificate expires. No control calls. No kcp access. |
| Session CA key (root of a generation) | one generation, rolled over with overlap | Control plane (cert-manager issuer; ideally KMS or Vault) | Issuing shard certificates: sessions for every connection on every shard, with fake data, until that generation is unpinned everywhere or expires. No control calls, no kcp access. The control plane already holds every connection secret, so this doesn't widen a control-plane compromise. |
| Intermediate key (if used) | short (e.g. weeks) | Control plane | Same as the root's, until the intermediate expires: nothing revokes it. |
| Session assertion | minutes | On the wire only | Bound to one connection; a `jti` cache stops replays within its validity. Useless at any other Provider. |
| SA token | minutes to hours | Connector memory only | Admin access to the provider workspace (RFC 006) and its exports' virtual workspaces until it expires, without the connector policy in the way. Only reachable by compromising a connector; never sent to the Provider-side. |
| Provider-side runtime | — | Provider-side | Whatever its open sessions allow: kcp requests within its admin RBAC in its own workspace, minus the connector policy, while the sessions last. Nothing persistent: there's no kcp credential to exfiltrate, and the policy refuses minting one. Ends when the PM-side closes the sessions. |
| Provider's edge (ingress, CDN) | — | Provider's infrastructure | Sees the onboarding token (spent), the connection secret on rare control calls, short-lived session assertions, and the reconcile traffic. Within the Provider's trust boundary, like the rest of its customers' traffic. |

### Threat model: what moved

Compared to v4's threat model:
- **The control plane** is now the highest-impact target: it holds every connection secret and the session CA. It's one component, not on the traffic path, so it can be locked down and kept small.
- **A connector** holds only its shard certificate (hours) and mints SA tokens. A compromised connector exposes that shard's sessions and traffic until its certificate expires, can't make control calls, and has no persistent secret to steal.
- **The owner's workspace**: same exposure as v4, with a bearer secret instead of a private key, limited to one connection. Secret scanning catches dumps and commits of the prefixed secret; v4's PEM keys are only caught by generic rules.
- **The Provider's edge** sees short-lived session assertions, the connection secret on rare control calls, and the reconcile traffic; v4's edge saw only handshake signatures. That's acceptable because of the trust boundary. Signing control calls too would remove the last bearer credential (see "Open items"); the edge would still see reconcile traffic.
- **A compromised Provider-side** loses most of its value: there's no kcp token to take away. v1 to v4 all left a usable kcp token behind.
- **Owners** see only aggregate conditions; connection records live in provider workspaces, and the session CA and shard certificates in the control plane, all outside owners' workspaces.
- **The Provider** is admin in its provider workspace and can see its own record, but the connector policy denies it any write to the record, the connection export's binding, endpoint-slice status, and token or certificate minting. That holds as long as the connector is its only path to kcp (see "Connector policy"). The connection export's only claim is read-only access to the workspace's endpoint slices.

### Resources

```yaml
apiVersion: providers.platform-mesh.io/v1alpha1
kind: ProviderOnboardingRequest
metadata:
  name: my-provider-2026-10-09
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
  name: my-provider-connection                       # owner's workspace; ownerReference: the Provider object; read by the controller only
type: Opaque
stringData:
  url: https://provider.example.com/pm               # base URL from the token
  connectionID: "..."
  secret: "pmc1_..."
  pendingSecret: ""                                  # only during a rotation or rebind
---
apiVersion: apis.kcp.io/v1alpha2
kind: APIExport                                      # in a PM workspace
metadata:
  name: connections.platform-mesh.io
spec:
  resources:
    - name: providerconnections                      # served by this export
      group: connections.platform-mesh.io
      # …schema reference
  permissionClaims:
    - group: apis.kcp.io
      resource: apiexportendpointslices
      verbs: [ get, list, watch ]                    # read-only; the export's only claim
---
apiVersion: connections.platform-mesh.io/v1alpha1
kind: ProviderConnection                             # in the provider workspace :root:providers:<name>-<suffix>
metadata:
  name: connection                                   # one per provider workspace
spec:
  connectionID: "..."
  url: https://provider.example.com/pm               # validated against the egress rules
  serviceAccount: { namespace: ..., name: ... }      # the SA the Provider controller created in this workspace
```

- **No `clusterID` field**: the record's logical cluster is the provider workspace. The front-proxy URL is deployment configuration. The SA is named explicitly, because RFC 006 doesn't fix its name (pm-operator derives it from a prefix and the provider suffix).
- **RBAC**: the connection controller writes records through the export's virtual workspace; connectors get read-only `apiexports/content`. The Provider's SA stays admin in its workspace, as in RFC 006; the connector policy, not RBAC, keeps it from writing records or the binding.

**How the request treats `provider.name`**:

| The `Provider`... | The request... |
|---|---|
| doesn't exist | creates it, waits for the workspace and SA, then registers |
| exists, not registered | reuses it and registers |
| exists and is registered, connection Secret present | **refuses**. The controller rotates the secret; there's nothing to re-onboard |
| exists and is registered, connection Secret missing | **rebinds** using `Provider.status.connection` (URL and `connectionID`), and waits out the rebind period |
| exists, but isn't a remote `Provider` | refuses |

- `Provider.status.connection` mirrors `url` and `connectionID`, which is what makes rebind possible after the Secret is lost. Nothing in it names shards.
- The `Provider`'s conditions, all aggregate: `Registered`, `SessionsConnected`, `KcpReachable` (every host the Provider-side needs), `SecretCurrent` (False while a rotation is pending), `Rebinding`.
- **No static kubeconfig for remote `Provider`s**, as in v2. Nothing on the PM-side is issued to the Provider-side except the sessions themselves.

### Restart and failure semantics

| Scenario | Behavior |
|---|---|
| Connector replica restarts, scales or rolls out | No registration: the owning replica signs new assertions with the mounted certificate and connects. Rendezvous hashing over stable StatefulSet ordinals keeps ownership on restart and moves only about 1/R of the connections on scaling; graceful changes hand over before cutting off (see "Connector replicas"). |
| One shard's connector is down | Only that shard's endpoint traffic stops, for every connection; its watches relist when it's back. Front-proxy traffic uses other sessions of the connection, where there are any; for connections hosted on that shard with no consumers elsewhere, it waits. |
| One shard's virtual workspace is down | Connectors keep the last state of the records and slices from that shard; only changes from it wait. |
| Shard added | Its connector is deployed with a shard certificate and watches the connection export. It dials a connection once kcp lists the shard in that connection's endpoint slices. Nobody registers anything. |
| Shard removed | Its connector goes away with it; the Provider-side drops the sessions; endpoint slices stop listing the shard. |
| First consumer binding on a shard | kcp adds the shard to the endpoint slice; that shard's connector sees it through the permission claim and dials. The Provider may briefly see the URL before the session is up; client-go retries. |
| Last binding on a shard removed | kcp drops the shard from the slice; that shard's connector closes the session after a grace period. |
| Provider tries to edit its record, drop the binding, forge endpoint-slice status or mint a token | The connector's authorizer denies it, and the attempt is audited. |
| Connection export's binding or record lost anyway (e.g. PM operator error) | Connectors drop that connection's sessions; the controller reports `SessionsConnected=False` and recreates them. |
| A record points at the wrong connection (PM-side bug) | The `cluster` claim doesn't match the `clusterID` the Provider-side stored at registration, so it refuses the session. |
| Shard certificate expires or renews | cert-manager renews it, with a new key, under the same pinned CA; the connector picks up the new files. Nothing reaches the Provider's control API. |
| Session CA generation nears its expiry without a rollover | Its expiry is a hard deadline: after it, sessions under it fail everywhere. Roll over well before; alert on the CA's remaining lifetime. |
| Session CA renewed by cert-manager anyway | With `rotationPolicy: Never` the key stays the same, and the old pinned certificate keeps verifying until it expires itself; pin the renewed one in that window. |
| Shard compromised | Stop issuing to that shard; its certificate expires within hours. Roll over the session CA only if the root's key is in doubt. |
| Connection controller down | Onboarding, rotations, rebind handling and offboarding wait. Sessions on every shard keep working. |
| Provider-side restarts | Sessions drop; connectors reconnect. The Provider-side has no credentials to restore. |
| Outage of any length | Nothing expires that the PM-side needs: the connection secret doesn't expire, shard certificates are renewed by cert-manager, and SA tokens are minted locally. |
| Ingress drops a session | The connector reconnects. Pings keep idle timeouts from triggering, and the connector replaces sessions before the edge's maximum connection age. On an abrupt cut, in-flight requests and watches on that session fail and client-go retries and relists, as after any apiserver disconnect. |
| PM rotates kcp's CA | Nothing to coordinate. Connectors use PM's own trust. |
| `POST` response lost | Retry with the same token, idempotency key and `secretHash`: same `connectionID`. A different hash is refused. |
| kcp unreachable from a connector | The Provider-side reports `kcpReachable=false` for that host; during registration the request stays in `VerifyingKcp` and fails after a timeout. |
| Secret rotation response lost, or controller crash mid-rotation | Try `pendingSecret`, then `secret`; resume. An unused pending hash expires. |
| Connection Secret lost | A new request rebinds; the connection and its state survive after the waiting period. Sessions keep working throughout. |
| Rebind started by someone else | The controller's next authentication with the current secret cancels it; the owner is notified. |
| Thief rotates the secret | The controller can't authenticate and the owner is notified. The owner freezes or deletes the connection in the portal. |
| Connection deleted on the Provider-side | Sessions and calls are refused. `Registered=False` on the `Provider`. Recovery is a new connection. |
| Provider-side runs multiple replicas | Sessions for different shards can land on different replicas, while the replica running a connection's endpoint-slice providers needs all of them. Either route through Lease holders as in v2, or shard reconciliation by connection and forward requests to the replicas holding the sessions. |

### Offboarding and revocation

- **Deleting the `Provider`** makes the controller delete its connection record first, so every shard's connector closes its sessions. That ends the Provider-side's kcp access immediately, whether or not anything else succeeds. Then the controller calls `DELETE`, and the Provider controller cleans up the SA and RBAC (RFC 006).
- **Finding 2 doesn't matter for remote `Provider`s any more.** Tokens are never handed out, so there's nothing to revoke after the SA is deleted. It still matters for RFC 006's static kubeconfigs.
- **An unreachable Provider-side doesn't block deletion**, as in v2: access is already gone, `DELETE` is only cleanup.
- **Deleting the connection on the Provider-side** refuses further sessions and calls; the PM-side surfaces it as `Registered=False`.

## Observations

Findings 1 to 11 from v4 still stand as facts about kcp, kcp-operator and earlier versions. v5 changes which of them matter:

| Finding | Relevance in v5 |
|---|---|
| 1, 3, 4, 5 (kcp token shapes, per-shard signing keys, no-overlap rotation, JWKS) | None for the Provider-side, which never sees a kcp token. 3 explains why connector-minted tokens work on every shard |
| 2 (SA deletion doesn't revoke bound tokens) | None for remote `Provider`s; still relevant for RFC 006 |
| 7 (fake kcp via connection-supplied CA plus token re-onboarding) | Closed differently: no CA bundle is sent, and rebind can't complete while the connection secret exists |
| 8, 11 (kcp-operator's server CA and merged bundle) | None: the connectors use PM's own trust |
| 9 (replay of a logged registration) | Closed: PM-generated secret, only the hash sent, retries bound to the hash |
| 10 (kcp credential at the edge) | Closed: no kcp credential leaves the PM-side |

New observations (2026-10-08/09):

12. **In tunnel mode, kcp trust equals connection trust.** The connector controls every request's destination, so a separately pinned or delivered kcp CA can't make the Provider-side's view of kcp more trustworthy than the session. v2 to v4 spent much of their complexity on that separation.
13. **The owner-CA revision of v4 had manual rotation steps because owners could supply their own CA**, not because of rotation itself; generated CAs already rotated automatically. v4 has since replaced the owner CA with a pinned per-connection key that the connector rotates on its own (2026-10-09).
14. **Serving many PM instances is version-independent on the Provider-side.** The notes stated multi-PM support only at the protocol level, but the consumer side is simple and the same everywhere: the SDK supplies a client config per connection, and the Provider's operator code creates a multicluster-runtime provider for each endpoint slice of its own exports, with cluster names prefixed by `connectionID`. Only the per-connection client config differs. Now described in v4.
15. **The tunnel's limits are mostly those of any client-go connection.** One TCP connection for all requests, shared flow control, and mass relists when it dies are normal client-go behaviour against kube-apiserver. What a reverse tunnel really changes is that the client can't open connections on its own, and that the path runs through the Provider's L7 edge. A multiplexer restores the first; the second is handled by pings and graceful session replacement.
16. **Impersonating the connection's SA works for the provider workspace, not for the APIExport virtual workspace** (kcp `3523eb866` and its vendored Kubernetes fork, read on 2026-10-08; not tested).
    - **Provider workspace, through the front-proxy to the shard.** kcp's fork treats an SA user without the `authentication.kcp.io/cluster-name` extra as local to whichever workspace the request targets (`vendor/k8s.io/kubernetes/pkg/registry/rbac/validation/kcp.go`, `IsForeign`). The workspace authorizer then authorizes it as a local SA (`pkg/authorization/workspace_content_authorizer.go`). The shard's chain allows the impersonation: kcp's gatekeeper only blocks privileged groups (`pkg/server/filters/impersonation.go`), the upstream filter checks `impersonate` on `serviceaccounts` for that namespace and name and adds the SA groups itself (`vendor/k8s.io/apiserver/pkg/endpoints/filters/impersonation/impersonation.go`), and kcp's scoping filter confines the impersonated user to that workspace. RBAC: `access` on `/` plus `impersonate` on `serviceaccounts` with `resourceNames: [<sa>]`. Adding the cluster-name extra would make the identity identical to a real token, at the cost of `impersonate` on `userextras/authentication.kcp.io/cluster-name` with `resourceNames: [<cluster>]`. Constrained impersonation (beta, on by default in the fork) still accepts the plain `impersonate` verb.
    - **APIExport virtual workspace, standalone server.** It uses the upstream default handler chain, without kcp's gatekeeper or scoping, and the `impersonate` check goes to the APIExport virtual workspace's authorizer (`pkg/virtual/apiexport/builder/build.go`, `newAuthorizer`). For per-cluster requests, `boundAPIAuthorizer` only allows resources bound or claimed in the consumer's APIBinding, so impersonating `serviceaccounts` is denied (`pkg/virtual/apiexport/authorizer/binding.go`). For wildcard requests, the content authorizer checks `impersonate` on `apiexports/content` without looking at the impersonation target (`content.go`), and the maximal-permission authorizer allows unclaimed resources (`maximal_permission_policy.go`). Granting that verb would let the caller impersonate any user or group, including `system:masters`, which the virtual-workspace server always allows (`virtual-workspace-framework/pkg/options/authorization.go`).
17. **`ClusterCachedResource` replicates per workspace, but a shared identity makes one cache-wide set.** A `ClusterCachedResource` publishes objects of one GVR from the workspace it lives in into the cache server (`pkg/reconciler/cache/clustercachedresources`). Workspaces that use the same identity key land in one set that can be listed and watched across all workspaces and shards in one call, `<resource>:<identityHash>` (`test/e2e/cache/replication_api_cache_test.go`, `TestReplicationWithWildcardListing`). Reads also work, read-only, through the replication virtual workspace (`pkg/virtual/replication`), authorized by `apiexports/content` on an APIExport whose resource has virtual storage pointing at a `ClusterCachedResourceEndpointSlice`. It was considered for connection discovery and dropped in favour of the connection export (see "Rejected alternatives").
18. **kcp-operator gives `VirtualWorkspace` servers the cache-server credentials when an external cache server exists.** PM runs an external `CacheServer` that the root shard and shards reference (`pm-helm-charts/charts/infra/templates/kcp/cache-server.yaml`, `root-shard.yaml`, `shards.yaml`). kcp-operator then mounts the cache-server kubeconfig and client certificate into every `VirtualWorkspace` deployment and passes `--cache-kubeconfig` (`kcp-operator/internal/resources/compiledvirtualworkspace/deployment.go`, with a TODO to make it optional). It's presumably the same credential kcp's own components use, so it's far broader than a connector needs. PM's existing virtual-workspaces server, deployed as a plain Helm Deployment, has no cache access.

## Comparison

| | v2 | v4 | v5 |
|---|---|---|---|
| PM-side proof after registration | Long-lived bearer secret, issued by the Provider | Pinned per-connection key, proven with short-lived self-signed certs in inner mTLS | Control calls: bearer secret generated by the PM-side. Sessions: short-lived assertions with short-lived shard certificates from a pinned session CA |
| Blast radius of a leaked PM-side credential | One connection | One connection | One connection's control calls (connection secret); one shard's sessions for hours (shard certificate) |
| Provider-side authenticated by | Web PKI | Web PKI, then a pinned server key | Web PKI |
| kcp credentials on the Provider-side | SA token + CA bundle | SA token + CA bundle | **None** |
| What a TLS-terminating edge sees | Secret, first access token | Spent onboarding token only | Spent onboarding token, connection secret on control calls, short-lived assertions, reconcile traffic (Provider's own edge) |
| Tunnel | Dial proxy (remotedialer/Konnectivity) | Dial proxy inside inner TLS | Multiplexed streams over WebSockets, at most one per (connection, shard) where there's traffic, ending at that shard's connector |
| TLS layers on kcp traffic | 2 | 3 | 2 (outer to the Provider, connector to kcp) |
| Revoking kcp access | Token TTL (finding 2) | Token TTL (finding 2) | Immediate: sessions closed (the connector policy refuses minting tokens to keep) |
| Rotation | Open item | Automatic: certs free, key via `PUT /key` | Automatic: secret via `PUT /secret`, shard certificates by cert-manager |
| Recovery after a lost PM-side credential | Re-onboard with a token (finding 7) | New connection, state lost | Rebind with a waiting period, state kept |
| Fake kcp by an outsider | Yes (finding 7) | No | No |
| Replay of a logged registration | Hole (finding 9) | Safe | Safe |
| PM-side components | Connector | Connector | Connection controller (stateful) + per-shard connectors (stateless) |
| Owner sees PM's shard layout | No | No | No |
| Provider-side software | Bearer check, token handling, dial-proxy server | TLS over WebSocket, pinning, CA checks, token handling | Bearer check, assertion and chain verification, multiplexer server, a dial hook for the standard transport |
| Main cost | Security holes | Complexity | Connectors in the L7 data path; a session CA to run |

## Changes from v4

| v4 | v5 | Why |
|---|---|---|
| Provider-side holds access tokens and kcp's CA bundle | Connectors inject SA tokens; Provider-side holds no kcp credentials | Removes credential delivery, refresh, CA bundle handling, and finding 2 for remote `Provider`s |
| Dial proxy, end-to-end TLS to kcp | Streams over the sessions end at the connectors' HTTP reverse proxy | The connector must see requests to inject credentials; observation 12 |
| Inner mTLS inside the WebSocket | Plain WebSocket over HTTPS | Provider's edge is the Provider's responsibility; nothing at the edge works against kcp |
| Pinned server key, rotation and refetch | Web PKI on the base URL | Only needed against an untrusted Provider edge |
| Pinned per-connection key, proven in inner mTLS, rotated via `PUT /key` | Connection secret for control calls; session assertions with shard certificates from a pinned session CA | Without inner TLS a key needs a separate proof format; a CA takes all registration off the session path |
| No re-onboarding; lost key means lost state | Rebind with a waiting period, cancelled while the connection secret exists | Recovery without letting outsiders take over |
| Custom dialer, pinning callbacks and tunnel inside inner TLS | Multiplexer over a plain WebSocket; client-go's standard transport with a dial hook | Observation 15: keep client-go's own pooling, health checks and upgrades |
| One connector, placement an open item | Connection controller (control plane) plus one stateless connector virtual workspace per shard (data plane) | kcp's serving stack and per-shard model; control-plane outages never stop traffic |

## Rejected alternatives

All of v4's rejected alternatives still apply, except "a PM-owned front key, with the connector terminating TLS", whose objection (the connector sees plaintext) doesn't hold: the connector is PM-side infrastructure seeing traffic to PM's own kcp, which the front-proxy sees in plaintext anyway. v4's rejection of "a dedicated PM connections CA as the only anchor" doesn't apply to v5's session CA, which only authorizes sessions (see "Session credentials"). In addition:

- **v4's inner mTLS with its pinned connection key.** Correct, but the inner mTLS only defends against the Provider's own edge, and without it a pinned key needs a separate proof format. Too much machinery for that threat.
- **A dial proxy with credentials on the Provider-side (v1 to v4).** Forces credential delivery, refresh and kcp CA trust onto every Provider, and leaves a usable kcp token behind on every Provider-side compromise.
- **The connector impersonating with general impersonation rights.** Avoids minting per-connection tokens, but broad impersonation rights are wider than TokenRequest on specific SAs, and a header-stripping bug would become privilege escalation.
- **The connector impersonating the connection's SA, scoped by `resourceNames`.** Narrow and verified to work for requests to the provider workspace, but not for the APIExport virtual workspace, which carries most Provider traffic (observation 16). Tokens are needed there anyway.
- **A standalone connector proxy.** Would have to rebuild authentication, authorization, audit, timeouts, in-flight limits and metrics that the virtual workspace framework provides.
- **One connector for all shards.** One session per connection would carry every shard's traffic, so losing it relists everything; load wouldn't scale with shards; and the allow-list would need the whole shard list.
- **A "home connector" role inside the per-shard connectors.** Couples shards (a restart on one shard waited for the home shard), puts stateful control work and connection secrets into virtual workspaces, and spreads the control plane over every shard.
- **Per-session or per-replica bearer secrets, registered on each restart.** Every restart would depend on the control plane and on the Provider's control API. A shared bearer secret across connections would be worse: one Provider could replay it to another.
- **Per-shard keys registered on every connection, with self-registration** (`ConnectorShard` objects). Needs the connector to persist a key and write to kcp, exposes the shard layout, and costs a call per connection whenever a shard is added or rotates its key. The session CA removes all of that.
- **Pinning the session CA's bare public key** instead of its certificate. Makes a renewal of the CA certificate invisible to Providers, but immutable generations are never renewed anyway. In exchange it loses the built-in expiry of trust (a key pinned at an unreachable Provider would stay trusted forever), the CA constraints, standard verification (Go's verifier needs a parent certificate, so the SDK would check signatures by hand or build a synthetic root), and room for an intermediate.
- **One PM key shared by every shard.** Simplest, but one leaked key exposes every shard's sessions, and rotating it fans out to every connection.
- **Per-shard status in the owner's workspace** (`Provider.status.connection.shards`). Exposes PM's topology to owners and needs connector writes. Aggregate conditions from the controller instead.
- **`/services/admin` for the connector's allow-list and shard discovery.** Exposes the whole installation's shard layout to a component that only needs its own shard's host, which its deployment already knows.
- **Discovery from owners' `Provider` objects** (consuming the providers APIExport, or a `ClusterCachedResource` per owner workspace). Owners can edit those objects, including the URL connectors would dial.
- **Connection records in a PM-internal workspace** (an earlier revision of v5). Puts all records on one shard, away from what they describe, needs a `clusterID` field and a separate lookup to find the hosting shard. The connection export carries both inherently.
- **Connection records replicated through the cache** (a `ClusterCachedResource` with a shared identity, observation 17). Needs a `ClusterCachedResource` and a shared identity key per provider workspace, and cache-server access for connectors, which in kcp-operator deployments means credentials far broader than they need (observation 18).
- **Annotations on the provider workspace's `Workspace` object in `:root:providers`.** No new API, but untyped, all on one shard, and finding the hosting shard needs a separate lookup.
- **Every shard's connector opening a session for every connection.** N connections × N shards sessions, mostly idle. Following kcp's own rule for endpoint slices opens only the ones that carry traffic.
- **Scoping the Provider's RBAC in its provider workspace**, so it can't write records or the binding. PM doesn't know what a Provider needs to deploy, so the Provider stays admin (RFC 006); the connector policy denies the few operations that matter to PM instead.
- **Signed connection records**, so connectors can detect a record the Provider edited. Unnecessary once the connector policy denies writes to records; it would add a signing key and verification for no extra protection.
- **Keeping the connection data in the `Provider` object**, with only an empty placeholder in the provider workspace. The owner can edit the `Provider` object (spec and status), so the owner could redirect the connection, and, without a PM-owned link between `Provider` and provider workspace, point it at another Provider's workspace. It also spreads the data back over owner shards and `:root:providers`.
- **Wildcard read of APIBindings on each connector's own shard** to find local consumers. Works, but needs shard-wide read access. The permission claim on endpoint slices keeps every connector privilege scoped to the connection export, and uses kcp's own answer.
- **A Provider-issued secret (v2).** The response would carry a secret, which reopens finding 9.
- **A gateway container exposing a local proxy and kubeconfig files** (discussed after v4). Doesn't help with multiple PM instances: the Provider still needs SDK code to consume a changing set of connections. It adds a hop and a directory format without adding a capability.
- **Rebind without a waiting period**, or one that the PM-side can't cancel. Turns a stolen token into a takeover again (finding 7).
- **HTTP/2 directly on the WebSocket** (an earlier v5 draft), with the Provider-side as client and the connector as server. Works, and HTTP/2's own limits are no worse than any client-go connection's (observation 15). But the Provider-side then has a fixed set of connections it can't add to, so the SDK would have to replace client-go's pooling, stream-limit handling, health checks and upgrade connections with custom code, on an unusual reversed setup with little tooling.
- **HTTP/1.1 directly on the WebSocket.** One request at a time per connection; the first watch would block the session.

## Open items

- **Proxying at scale (the main risk).** Watches and long-running requests from every Provider pass through a connector. Measure memory and goroutines per stream, streams per session, and backpressure, for one shard's connector at realistic Provider and cluster counts.
- **Multiplexer choice.** remotedialer (built for this direction, widely deployed) vs. yamux/smux. Check: whether remotedialer applies backpressure per stream (it may buffer instead), whether its client side accepts a custom dialer so streams can go to the in-process HTTP server, and whether it has a go-away for graceful session replacement. yamux has per-stream windows and a go-away, but no built-in WebSocket integration or dial addressing.
- **Transport check.** Confirm that `http.Transport` with a `DialTLSContext` returning a non-TLS stream speaks HTTP/1.1 to `https://` URLs as expected, and that client-go accepts it as `rest.Config.Transport` together with multicluster-runtime's per-endpoint configs.
- **Edge timing.** A session ping interval and a session replacement schedule that fit common idle timeouts and maximum connection ages; window or buffer sizes for high round-trip times (large LISTs).
- **Session CA.** Issuer (cert-manager with a KMS- or Vault-backed issuer), generation lifetime and rollover interval, whether to use an intermediate, alerting on a generation's remaining lifetime, shard certificate lifetime, how the host is encoded in the shard certificate (URI SAN format), assertion lifetime, clock-skew tolerance and `jti` cache size on the Provider-side. Whether per-replica certificates are worth it.
- **Connection controller.** Where it runs (pm-operator or its own Deployment), HA and sharding, poll interval, rate limits and backoff per Provider.
- **Connection export.** The `ProviderConnection` schema; which PM workspace hosts the export; confirm that a permission claim on `apis.kcp.io` `apiexportendpointslices` works and is readable through the export's virtual workspace; the binding's lifecycle in the Provider controller (create with the workspace, recreate if deleted).
- **Connector policy.**
  - **Legacy SA-token Secrets:** confirm whether kcp issues tokens for Secrets of type `kubernetes.io/service-account-token`. If it does, the proxy handler has to inspect Secret writes (`create`, `update`, `patch`, server-side apply) before forwarding, since an authorizer can't see the type.
  - **Duplicate bindings:** `create` doesn't name the existing binding. Check whether kcp refuses a second binding to the same export in one workspace, and whether a duplicate matters.
  - **Workspace creation:** whether to cap child workspaces below the provider workspace.
  - **Path parsing:** use kcp's request-info resolver for both path shapes, or parse them explicitly; test the denylist against both.
- **Where Service endpoint slices live.** The session rule reads them in the provider workspace and only counts slices for exports in that workspace. Confirm that's where Providers keep them, or document it as a requirement.
- **Naming.** pm-operator already has a `ProviderConnection` type (`corev1alpha1.ProviderConnection`, `pm-operator/pkg/subroutines/defaults.go`) for its own components' kubeconfigs. Pick a different name for the record before implementing.
- **Connector cache-server credentials.** kcp-operator mounts them into every `VirtualWorkspace` in PM's setup (observation 18). The connector doesn't need them: find out whether kcp-operator can omit them (its TODO) or give a scoped credential, or deploy the connector without them.
- **Connector replicas.** The membership source (EndpointSlices vs. a Lease per replica), replica count per shard, sizing against the shard's endpoint traffic, and pacing of rollouts so relists don't burst.
- **Deployment health.** Alerting when a shard has no running connector, since nothing checks it at runtime.
- **Request policy in the virtual workspace's authorizer.** The exact paths the connection principal may reach and the headers stripped and added, alongside the denylist in "Connector policy".
- **Egress rules for connectors.** The exact checks on Provider base URLs, both when the controller writes a record and when a connector dials.
- **Possible kcp issue: impersonation on the APIExport virtual workspace.** From reading the code (observation 16), anyone with `impersonate` (or `*`) on `apiexports/content` may be able to send wildcard requests to that export's virtual workspace while impersonating any group, including `system:masters`, which the server always allows. Its effect looks limited, because wildcard requests are forwarded with the virtual workspace's own client anyway. Confirm with an e2e test before reporting it to kcp.
- **Proxied requests in the generic handler chain.** Configure the long-running check and request timeout for proxied LISTs and watches; confirm the `X-Kcp-Virtual-Resource-Hops` budget when proxying into the APIExport virtual workspace.
- **`ServeProxy` upstream.** Propose moving `kcp/pkg/virtual/shared.ServeProxy` into the virtual workspace framework, or copy it.
- **Provider-side multi-replica.** Sharding vs. Lease routing, and whether opening one session per Provider replica can be made reliable through common ingresses. More pressing with per-shard sessions.
- **Rebind details.** Length of the waiting period, rate limits, and what happens when a thief holding the secret keeps cancelling the owner's rebinds (portal freeze is the current answer).
- **Signed control calls.** Sessions already use signed assertions. The remaining bearer credential is the connection secret on rare control calls. For Providers that don't trust their own edge, replace it with a per-connection keypair, pinned at registration, that signs a short-lived JWT (RFC 7523 `private_key_jwt` style) on each control call: v4's per-connection key with a different proof. The edge would then see no reusable credential, but still sees reconcile traffic.
- **`direct` mode.** In v5, the Provider-side never talks to kcp directly. "Direct" would mean the Provider-side calls the connectors instead of the connectors dialling out, with the same routing, identity injection and authorizer. Each shard's connector would be exposed under its own external hostname, not through the front-proxy, which routes `/services/...` to a single backend (see "Per-shard connectors", point 8). What's missing is a credential in the other direction: one that the virtual workspace accepts for its connection only, is useless against kcp directly, and is revoked by deleting the connection. Defer until there's a use case.
- **WebSockets through ingresses.** Confirm that a long-lived binary WebSocket survives common ingresses, CDNs and WAFs (binary frames, maximum message size, maximum connection age).
- **multicluster-runtime composition.** Confirm the `multi` provider (or an equivalent) supports adding and removing per-connection providers at runtime with prefixed cluster names.
- **Audit.** kcp's audit log shows the connection's SA; the connector's audit log records the connection principal and the SA it acted as. Decide the audit policy and retention for connector traffic.
- **Readiness gate timeout**, **request permissions**, **addressing changes**: as in v4.

## Prior art

- **kcp front-proxy**: an authenticating reverse proxy for kcp, with streaming and watch support.
- **kcp's initializing-workspaces virtual workspace**: a `handler.VirtualWorkspace` that authorizes, strips credentials, impersonates and reverse-proxies into a workspace; the pattern for the connector's proxy half.
- **kcp's per-shard virtual workspaces** and **kcp-operator's `VirtualWorkspace` resource**: the deployment model for per-shard connectors.
- **kcp's admin virtual workspace** (`/services/admin`): the pattern for a read-only, cache-backed view, if a live connection-status resource is ever wanted.
- **Multi-shard APIExport consumers** (multicluster-runtime's endpoint-slice provider, as used by PM's own operators): how connectors read the connection export across shards.
- **cert-manager with KMS- or Vault-backed issuers**: short-lived workload certificates from a CA whose key stays in a KMS.
- **Teleport Kubernetes service**: a proxy that attaches identity to Kubernetes requests, so clients never hold cluster credentials.
- **Cloudflare Tunnel** and similar: an outbound connection from the private side, with HTTP served back over it.
- **Rancher `remotedialer`**: multiplexed streams over a WebSocket, dialled from the side that accepted it; v5's candidate multiplexer, with streams ending at the connector's HTTP server instead of at kcp.
- **Konnectivity**: the dial-proxy approach v1 to v4 used, with the client holding kcp credentials.
- **client-go and kube-apiserver connection handling**: one HTTP/2 connection per host, health checks, and server-sent GOAWAY (`--goaway-chance`) for rebalancing; the baseline v5's tunnel is measured against.
- **RFC 7515 `x5c` and RFC 7523 JWT client assertions**: signed assertions carrying their certificate chain.
- **Open Cluster Management cluster-proxy**: reverse tunnel to managed clusters.
- **kubeadm bootstrap tokens**: `tokenID.tokenSecret`, stored hashed, short-lived.
- **Prefixed secrets for scanning**: GitHub's token prefixes and secret-scanning partner programme.
- **Account recovery with a waiting period**: common in consumer identity providers; recovery completes only if the current credential holder doesn't object in time.
