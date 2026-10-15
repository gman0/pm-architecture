# Notes v5: RFC 011 provider onboarding flow

Working notes for [011](./011-provider-onboarding-flow-improvements.md). Not the RFC itself. The design in brief:

- **The Provider-side never holds kcp credentials.** The connector is an authenticating HTTP proxy for kcp. It adds the SA token itself, so no access token, kcp CA bundle or token rotation ever crosses to the Provider-side.
- **Sessions and control calls run inside a TLS 1.3 handshake within the WebSocket**, authenticated by short-lived connector certificates (client) and a pinned server key (server). There's no credential to deliver, and the Provider's edge sees only opaque bytes (see "The Provider's edge: inner TLS on sessions").
- **The PM-side is split** into a **control plane** (platform-mesh-operator) and a **data plane** (stateless connectors, one per kcp shard). Control calls are authenticated by a PM-generated secret per connection, which only PM can change; sessions by short-lived connector certificates from a PM connections CA, presented in the inner TLS. Nothing needs anyone's involvement to rotate, and owners learn nothing about PM's shards.

The result: one registration call, at most one WebSocket per (connection, shard), only where the connection has traffic, carrying end-to-end encrypted, multiplexed streams (so the Provider-side can "dial" and use client-go's standard transport), and credentials that nobody has to touch.

The [implementation plan](./xxx-notes-011-v5-impl-00-overview.md) turns these notes into components and holds the detailed decisions; where the two differ, the plan wins.

## Terminology

- **PM-side**: the PM instance (kcp + PM operator). May be behind NAT; dials out.
- **Provider-side**: the service provider (controllers + portal in its runtime cluster). Serves the connection API and reconciles on the PM-side.
- **Owner**: whoever controls the workspace that holds the `Provider` object on the PM-side.
- **Connection**: the relationship between one `Provider` object on the PM-side and one connection entry in the Provider-side's own database, identified by `connectionID`. Provider-side state belongs to its connection.
- **Onboarding token**: a single-use, short-lived token from the Provider's portal, with the Provider's base URL inside.
- **platform-mesh-operator**: the PM-side **control plane**. It owns each `Provider`, local or remote, end to end ([03](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)), with one active replica by leader election. For remote Providers it reconciles onboarding requests, bootstraps provider workspaces, registers connections, makes every control call, changes secrets, writes each connection's `ProviderEndpoint` into its provider workspace, and reports the `Provider`'s status. Never on the path of session traffic.
- **Connector**: the PM-side **data plane**, implemented as `providerconnections-vw`. A stateless virtual-workspace server, **one per kcp shard**, that opens sessions and acts as an **authenticating reverse proxy** to kcp for requests arriving over them.
- **Connection secret**: a high-entropy secret platform-mesh-operator generates and keeps in the `ProviderEndpointRegistration`'s Secret in its own namespace in PM's runtime cluster, never in kcp. The Provider-side stores only its hash. It authenticates **control calls only**, sent inside control sessions. It grants **no** kcp access, and only PM can change it, through the `ProviderEndpoint` (see "Changing the connection secret").
- **Connector CA**: a PM connections CA that only issues connector certificates, which connectors present as the client certificate in each session's inner TLS. cert-manager manages it and renews it yearly with a new key; every connection pins PM's **trusted set** of CA certificates, which platform-mesh-operator pushes before connectors use a new CA (see "Automatic rollover").
- **Connector certificate**: a short-lived certificate (24h, renewed halfway) from the connector CA for one shard's connector, issued by cert-manager and mounted read-only into the connector. It carries no hostnames: the hosts a session serves travel in its hello.
- **Hello**: the first message a connector sends inside a session's inner TLS, naming the provider workspace's `clusterName` and the hosts the session serves. The inner TLS protects it, so it needs no signature.
- **Server keys**: keypairs the Provider SDK generates and keeps; their public keys are returned at registration and pinned by the PM-side. The Provider-side presents one as the inner TLS server certificate.
- **Session**: one WebSocket from one shard's connector to the Provider's base URL, with an inner TLS 1.3 handshake inside that authenticates both sides: the connector by its connector certificate, the Provider-side by a pinned server key. There's one per (connection, shard, connector replica), opened only where the connection has traffic (see "Per-shard connectors"). It carries a stream multiplexer (yamux) inside the inner TLS.
- **Stream**: a lightweight connection inside a session, opened by the Provider-side whenever client-go wants a new connection to a kcp host. It carries plain HTTP: requests one after another, or one long watch. Streams need no handshake or authentication of their own; the connector authorizes each request on them. A session carries many streams (see "Sessions: a stream multiplexer over a WebSocket").
- **Control session**: a WebSocket from platform-mesh-operator to the Provider's base URL, with inner TLS against a pinned server key, carrying the control calls.
- **Connection export**: the PM APIExport `connections.platform-mesh.io` in `:root:providers:system`, serving `ProviderEndpoint`, with a local **maximal permission policy** that allows only reads in the workspaces that bind it. platform-mesh-operator binds it in every provider workspace when it creates the workspace. It claims read-only access to `apiexportendpointslices` there.
- **`ProviderEndpointRegistration`**: platform-mesh-operator's authoritative state for one connection (`connectionID`, base URL, server keys), in its own namespace in PM's runtime cluster, with a Secret holding only the connection secret. Neither owners nor Providers can reach it.
- **`ProviderEndpoint`**: a non-secret object (`connectionID`, base URL, the pinned server keys, the `Provider`'s `caBundle` if set, and `pendingSecretHash` while the connection secret changes), written from the `ProviderEndpointRegistration`, in the connection's **provider workspace**, written by platform-mesh-operator through the connection export. The Provider is admin in that workspace (RFC 006), but the export's maximal permission policy makes kcp itself deny it any write to the `ProviderEndpoint` (see "Connector policy"). Connectors read it through the export's endpoint slice, so the `ProviderEndpoint` lives on the shard of the workspace it describes.
- **Connection principal**: the identity a request arriving over a session of connection `{id}` is authenticated as inside the connector, e.g. `pm:connection:<id>`. The connector maps it to the connection's SA before forwarding to kcp.
- **`ProviderOnboardingRequest`**: the owner's request to onboard a `Provider` with an onboarding token. It may also onboard an already registered `Provider` again, once the Provider-side reports its connection unknown (see "Recovery").

## Problem

The Provider-side must reconcile on any number of PM instances, the PM-side may be behind NAT and opens every connection, the user gets a token from the Provider's portal, and both sides must recover from restarts and outages without manual steps.

**kcp credentials stay on the PM-side.** Delivering them to the Provider-side (a kubeconfig, or SA tokens and a CA bundle) brings most of the complexity with it:
- the credentials must not be visible at a TLS-terminating hop on the way;
- the Provider-side must trust kcp's CA, and whoever can change that bundle can present a fake kcp;
- the credentials must be refreshed, and deleting the SA doesn't revoke bound tokens already issued in PM's kcp;
- the PM-side needs a strong, long-lived identity to push them, and a procedure to rotate it.

**Constraints:**
- **Assume as little as possible about the Provider's infrastructure.** Any HTTPS endpoint that supports WebSockets must do. No TLS passthrough, no client-certificate-verifying ingress.
- **Assume nothing about how PM operates kcp's PKI.** Rotating kcp's CAs, adding shards or changing PM's edge must not involve Providers.
- **Owners learn nothing about PM's topology.** Nothing in an owner's workspace names shards or per-shard state.
- **Virtual workspaces stay stateless.** A connector holds no persistent state and writes nothing to kcp except the TokenRequests it needs; everything it knows about its place comes from its deployment.

## Trust boundaries

- **The owner's workspace holds no connection credential.** The connection secret stays in PM's runtime cluster: in an owner's workspace, any Provider with an accepted Secrets claim there could read it (PM's bindings accept that claim with `matchAll`), and an owner could delete it. The owner's workspace keeps only the request, the `Provider` and the onboarding token, whose data is deleted once spent.
- **The Provider's edge is outside the boundary.** Sessions and control calls are end-to-end encrypted between PM's components and the Provider's runtime (inner TLS), so the Provider's ingress, load balancer or CDN sees only opaque bytes (see "The Provider's edge: inner TLS on sessions").

v5 makes sure that:
- **nothing outside PM and the Provider's runtime** can take over a connection: not a stolen onboarding token or a compromised portal account (they only register new connections, see "Recovery"), not someone holding the connection secret (they can't change it, see "Changing the connection secret"), not an observer of logged requests, not the Provider's edge;
- **no kcp credential ever leaves the PM-side**, so a compromised Provider-side database yields nothing that works against kcp once its sessions are gone;
- **revocation is immediate**: when the connectors stop proxying, the Provider-side has nothing left to use;
- **a Provider can't harm PM from within its own workspace**, even though it's admin there: kcp's maximal permission policy on the connection export keeps `ProviderEndpoint`s read-only for it on any path, and its only path into kcp is the connector, whose authorizer denies the few other operations that would affect PM (see "Connector policy").

**In tunnel mode, trusting kcp is trusting the connection.** The connector decides where every request goes, so the Provider-side can never verify kcp more strongly than it verifies the session. There's no separate kcp CA bundle for it to trust: the Provider-side trusts responses that arrive over an authenticated session, for that connection only.

## Core idea

```
Provider-side controllers ── client-go, standard http.Transport (real kcp URLs)
                                   │ "dial" = open a stream on the session for that host
             ┌──── one WebSocket per shard (outbound from PM) ──────────────────────┘
             │       inner TLS 1.3 (connector certificate ↔ pinned server key)
             │         yamux inside; plain HTTP on each stream
             ▼
   connector@shard ── in-process http.Server + reverse proxy:
                      check host, strip auth headers, add SA token ── TLS ──▶ kcp
```

- **The connectors dial out**: each shard's connector that the connection needs (see "Per-shard connectors") opens a WebSocket to `GET /v1/connections/{id}/session` on the base URL. The Provider's ingress terminates the outer TLS and routes it like any WebSocket; authentication happens inside it.
- **Inside the WebSocket, the connector and the Provider SDK run a TLS 1.3 handshake** (see "The Provider's edge: inner TLS on sessions"), and inside that a **stream multiplexer** (yamux). The Provider-side opens a new stream whenever client-go wants a new connection, so client-go's standard transport works unchanged: pooling, health checks, and separate connections for upgrades. See "Sessions" for why this matters more than the HTTP version.
- **The connector is an authenticating reverse proxy.** Each stream is handed to an in-process HTTP server. For each request it checks the target host against the allow-list, removes any credentials the Provider-side sent, adds a short-lived SA token for that connection, and forwards the request to the real kcp URL over TLS verified with PM's own trust.
- **The Provider-side keeps the real kcp URLs** (front-proxy, and the shard virtual-workspace URLs from endpoint slices). Only the transport's dial hook is replaced: it opens a stream on the session that serves the target host.

## Provider-side prerequisites

- **An HTTPS endpoint at the base URL from the token that passes the session and control paths as plain WebSocket pipes.** The quickstart's checklist lists what that takes ([08 Quickstart, "Connection server"](./xxx-notes-011-v5-impl-08-provider-quickstart.md)).
- **The connection server** (registration, control sessions, sessions, with the inner TLS server that verifies connector certificates). It ships in the Provider SDK.
- **One active replica** of the connection server and the Provider's controllers, by leader election; standbys answer upgrades with `503` and `Retry-After`.
- **State keyed by `connectionID`** (see "Provider-side: serving many PM instances"), in the SDK's custom resources in the Provider's own cluster (`Connection`, `OnboardingToken`, `ServerKey`), the only source of truth for that state, ideally as durable as consumer data. Consumer data is keyed with the `connectionID` too, found again from kcp objects before anything new is created, and never deleted because a connection went away (see "Recovery").
- **Owner notifications and controls** in the portal: Events on the SDK's `Connection`s report every registration, secret change and connector CA change, so the Provider can tell the token owner; owners can delete connections, which deletes the `Connection`. PM's side tells owners about tampering and failures too, through events and conditions on their `Provider`, so a Provider's own notifications add to that rather than being the only channel.

Not required: TLS passthrough, client-certificate handling at the ingress, a CA of the Provider's own, cert-manager, or any knowledge of kcp's tokens, keys or CAs. The Provider-side keeps its **server keys** in a Secret, generated by the SDK and rotated with overlap.

## Flow overview

1. **Get a token** from the Provider's portal.
2. **Create the `Provider` and the request.** The owner creates a remote `Provider` and a request referencing it and the token Secret, in either order. The oldest unfinished request per `Provider` wins; newer ones fail at once.
3. **Find the `Provider`.** The request waits until it exists. An already registered `Provider` is refused, unless the Provider-side reports its connection unknown; then it gets a new connection. platform-mesh-operator bootstraps the provider workspace: SA, connection export binding and connector RBAC, but no kubeconfig.
4. **Register.** platform-mesh-operator creates the connection's `ProviderEndpointRegistration` in its own namespace, generates the connection secret into its Secret, and calls `POST /v1/connections` with the onboarding token, the addressing, the **hash** of the secret, and the **connector CA bundle**. It gets back the `connectionID` and the Provider-side's **server keys**, and persists the `connectionID`, the base URL from the token and the server keys together in the `ProviderEndpointRegistration`, which stays authoritative: the `ProviderEndpoint` is written from it. No kcp credential and no secret crosses the wire. Then the controller deletes the spent token's data from the owner's token Secret, and the request is done (`Registered`); everything after is reported on the `Provider`. A crash before the result is persisted is recovered by retrying the same call, which the Provider-side honours for 7 days.
5. **Make a first control call** (`GET` over a control session). The `ProviderEndpoint` isn't written until it succeeds, so a control path that's broken from the start shows up here, with its edge-diagnosis label in the `Provider`'s `Registered` condition, before any session opens.
6. **Write the `ProviderEndpoint`.** The controller validates the base URL (egress rules) and writes the connection's `ProviderEndpoint` into the provider workspace, through the connection export, which the controller bound there when it created the workspace.
7. **Open sessions where they're needed.** The connector on the shard hosting the provider workspace opens a session right away, for front-proxy traffic. Connectors on shards listed in the connection's Service endpoint slices, i.e. shards with at least one consumer binding, open theirs as those shards appear. Each runs the inner TLS with its connector certificate, against the server keys in the `ProviderEndpoint`, and sends its hello. Nothing is registered per shard.
8. **Confirm kcp is reachable.** The controller polls `GET /v1/connections/{id}` over a control session until the Provider-side reports that the front-proxy and every host it needs are covered by a session and reach kcp. Then the `Provider`'s `Connected` condition is true, and it becomes `Ready`. There's no timeout: it keeps waiting, with the reason in its conditions.
9. **Run.** The Provider-side reconciles through the sessions. There's no credential delivery and no credential refresh on the Provider-side.
10. **Change the connection secret when needed**, through the `ProviderEndpoint` (see "Changing the connection secret"). Connector certificates renew on their own; once a year cert-manager renews the connector CA, and the controller pushes it to every Provider before connectors use it (see "Automatic rollover").
11. **Offboard.** Deleting the `Provider` removes its `ProviderEndpoint`, so every connector closes its sessions, which ends kcp access at once; then the controller calls `DELETE`, deletes the `ProviderEndpointRegistration` and its Secret, and tears down the connector RBAC, binding, SA and workspace, all in one finalizer.

Steps 7 to 10 handle every restart and outage, of any length: the connection secret doesn't expire, connector certificates are renewed by cert-manager, and the connectors mint SA tokens for themselves.

## Design

### Onboarding token

`pm1.<base64url(Provider base URL)>.<tokenID>.<tokenSecret>`, single-use, short-lived, stored in a Secret, with the URL inside so nothing is sent to the wrong host. The portal never puts tokens in URLs and hands them over as a `kubectl create secret` command, not a manifest.

### Control plane and data plane

```
                         PM control plane                                    PM data plane
 ┌──────────────────────────────────────────────────────┐      ┌────────────────────────────────────────────┐
 │ platform-mesh-operator                               │      │ providerconnections-vw@shard-A (stateless) │
 │  (one active replica)                                │      │  mounted: connector cert + key, own VW     │
 │  · ProviderOnboardingRequest / Provider reconcile    │      │           URL, front-proxy URL             │
 │  · ProviderEndpointRegistrations and their Secrets   │      │  reads:   connection export                │
 │  · control sessions to Providers (status, CAs, …)    │ ───▶ │           (ProviderEndpoints, claimed      │
 │                                                      │      │           endpoint slices),                │
 │  · Provider status (conditions, phase)               │      │           ConnectorCABundle                │
 │  · ProviderEndpoints (provider workspaces,           │      │  writes:  TokenRequests only               │
 │    through the connection export)                    │      │  dials sessions, proxies to kcp            │
 │  · connector CA trusted set, ConnectorCABundle status│      │                                            │
 │ connector CA and connector certificates,             │ ───▶ │ connector@shard-B …  connector@shard-C …   │
 │  managed by cert-manager                             │      │                                            │
 └──────────────────────────────────────────────────────┘      └────────────────────────────────────────────┘
```

- **Everything stateful sits in the control plane**: connection secrets, control calls, status, and the connector CA trusted set. platform-mesh-operator owns each `Provider` end to end, and leader election gives one active replica, so each connection has exactly one writer.
- **Everything on the traffic path sits in the data plane**, which is stateless. A connector replica can be killed at any time; a new one reads its mounted certificate and the connection export and carries on. No registration, no writes, no shard list.
- **A control-plane outage doesn't stop traffic.** It delays onboarding, secret changes, rollovers and offboarding. The one exception is an outage of more than about 12 hours right when cert-manager renews the connector CA: new sessions then fail until the controller is back, while open ones keep running (see "Restart and failure semantics").

### Connection API (on the base URL)

The full wire contract is [01 Protocol](./xxx-notes-011-v5-impl-01-protocol.md). Only registration is plain HTTPS, and it carries the onboarding token (spent on first use) and public material only. Everything that grants anything travels inside inner TLS: control calls in **control sessions**, traffic in **sessions**.

```
# ── plain HTTPS (web PKI on the base URL, or the Provider's spec.remote.caBundle) ──

POST /v1/connections
Authorization: Bearer <tokenID>.<tokenSecret>
{
  "idempotencyKey": "<ProviderOnboardingRequest UID>",
  "clusterName": "<logical cluster name of the provider workspace>",
  "frontProxyURL": "https://...",
  "secretHash": "sha256:<hex>",
  "connectorCABundle": "-----BEGIN CERTIFICATE-----\n…"    # PEM bundle, CA certificates only (see below)
}
→ 201 { "connectionID": "...", "serverKeys": [ … ] }

# ── control session: WebSocket upgrade, then inner TLS 1.3 against a pinned server key;
#    HTTP/1.1 inside, each call with Authorization: Bearer <connection secret>;
#    the first failed authentication closes the control session ──

GET /v1/connections/{id}/control                 (WebSocket upgrade)

  GET /v1/connections/{id}
  → 200 { "requiredHosts": { "<host:port>": { "sessions": 1, "kcpReachable": true, "lastError": "" } },
          "serverKeys": [ … ], "connectorCABundle": { "etag": "...", "fingerprints": [ … ] } }

  PUT /v1/connections/{id}/connector-ca-bundle   # connector CA rollover, compare-and-swap
  If-Match: "<etag from GET>"
  { "connectorCABundle": "<PEM>" }               # the trusted set: current CA, and the previous one during a rollover
  → 204                                          # identical body: a no-op
  → 412 { "etag": "..." }                        # changed since: read again

  # no call changes the secret: it changes only through the `ProviderEndpoint`

  DELETE /v1/connections/{id}
  → 204

# ── session: WebSocket upgrade without credentials, then inner TLS 1.3 (mutual:
#    connector certificate ↔ pinned server key), a hello, then yamux ──

GET /v1/connections/{id}/session                 (WebSocket upgrade)

  hello (first message inside the inner TLS, from the connector):
  { "clusterName": "<logical cluster name of the provider workspace this session is for>",
    "hosts": [ "<virtual-workspace host(s) this session serves>" ],  # routing only; old and new during a rename
    "sub": "<opaque session-group ID>" }                             # e.g. a hash of shard and connectionID; audit only

  then yamux; each stream carries plain HTTP/1.1 from its first byte
```

- **The PM-side generates the secret; only its hash is sent.** The response to `POST` contains nothing secret. Replaying a logged registration gains nothing, and a retry is accepted only with the same `secretHash`.
- **Secret format**: a prefix plus 256 random bits, e.g. `pmc1_<base64url>`. The prefix lets secret scanners (GitHub, GitLab, gitleaks) recognise leaked secrets. High entropy means a plain SHA-256 is enough to store it; no slow password hash.
- **Idempotency**, keyed on the token ID and the request's UID, with a retry window after token expiry (7 days by default); addressing (`clusterName`, front-proxy URL) is immutable; registration carries no APIExport details ([01 Protocol, "Registration"](./xxx-notes-011-v5-impl-01-protocol.md)).
- **The registration response is trusted through web PKI** on the base URL, or the `Provider`'s `caBundle` for a private CA, which is what makes the returned server keys safe to pin.
- **Two kinds of credential, two planes.**
  - The **connection secret** (bearer) only authenticates control calls, which are rare, and only platform-mesh-operator uses it. It's sent only inside a control session's inner TLS, so the Provider's edge never sees it, and it grants no kcp access. No control call can change it.
  - **Connector certificates** (24 hours) only authenticate sessions, inside the inner TLS. Nothing reusable crosses the edge on the session path: the edge sees only an upgrade and opaque bytes.
- **WebSocket framing**, for both kinds of session: binary messages only, at most 32 KB each, outer pings every 30 seconds, no reliance on message boundaries. Most onboarding problems come from the Provider's edge; the controller classifies them (upgrade refused, a bot-challenge page, a close right after a binary message, regular idle or age cuts) and reports distinct reasons to the owner.
- **Versioning**: path version for the endpoints, ALPN inside the inner TLS for the session and control protocols (e.g. `pm-session/1`, `pm-control/1`), negotiated per channel; the SDK and every PM-side component support the current and the previous version.
- **Status is per host, not per shard.** `requiredHosts` lists the front-proxy and the hosts from the endpoint slices the Provider-side actually uses, with whether a live session covers each and reaches kcp. The Provider-side knows which hosts it needs; nobody has to tell it the shard layout.

### Session credentials: the connector CA

Each connection pins the **connector CA bundle**: PM's **trusted set** of connector CA certificates, sent at registration and pushed whenever it changes (see "Automatic rollover"). Each shard's connector gets a short-lived **connector certificate** issued directly by the current CA, mounted read-only and renewed by cert-manager with a new key each time. The connector presents it as its client certificate in each session's inner TLS.

**Authenticating a session** (in the Provider SDK; the details are in [01 Protocol, "Inner layer" and "Hello"](./xxx-notes-011-v5-impl-01-protocol.md)):
1. **the inner TLS 1.3 handshake** requires a client certificate issued directly by a CA in the connection's pinned bundle: the standard verifier (`x509.Verify`) with **only** those CA certificates as roots, which checks validity and the CA constraints. The connector certificate must also allow signing and client authentication (key usage `digitalSignature`, EKU `clientAuth`); Go's verifier requires EKU `serverAuth` unless `VerifyOptions.KeyUsages` says otherwise, so the SDK passes `clientAuth` explicitly. The connector signs the handshake transcript with the certificate's key, which proves it holds the key; the transcript's fresh random values rule out replay; and since it includes the Provider's server certificate, which the connector pins per connection, a handshake can't be forwarded to another Provider. No separate session token is needed;
2. **the hello**, the connector's first message inside the inner TLS, names the `clusterName` this session is for; it must equal the one stored for this connection at registration. The connector takes it from the logical cluster of the `ProviderEndpoint` it acts on, and uses that workspace's SA for the session's requests. `ProviderEndpoint`s can only be written by PM (see "Connector policy"), so this is defense in depth: even a PM-side bug that mixed up `ProviderEndpoint`s couldn't make a connector serve one connection's sessions with another provider workspace's SA;
3. the session serves the hosts in the hello's `hosts`, plus the connection's front-proxy, which every session may carry.

The upgrade itself carries no credentials, so the Provider-side accepts it before knowing the caller, as it does for control sessions. It refuses unknown connection IDs before the upgrade, caps handshakes in progress and times them out; a caller without a certificate from a pinned CA gets no further than a failed handshake.

**Hosts are routing, not authority.** The Provider-side never resolves kcp's hostnames (its dial hook opens streams on sessions), so they're labels it couldn't verify independently, and it doesn't need to: a session that passed checks 1 and 2 belongs to PM's connector for this connection, and the hosts only decide which session a stream goes to. Keeping them out of connector certificates means a hostname change on PM's side never waits for certificates to be reissued; during a rename, connectors list old and new hosts until endpoint slices have switched, so the order of changes doesn't matter. Connector certificates therefore carry no hostnames, and the CA needs no name constraints.

**What binds a session to its PM installation** is the CA pinned for this connection: each installation has its own connector CA, managed by its own cert-manager; with an external PKI, its own intermediate, which is what Providers pin. If two installations shared one, a compromised connector of either could open sessions for the other's connections, since the hello's `clusterName` is the connector's own statement. Connector certificates carry no installation identity, and PM has no installation identifier to put there.

**Why a CA:**
- **It only authorizes sessions.** A forged session can feed a Provider fake data for that connection. It can't make control calls, which still need each connection's own secret, and it gets no kcp access. The CA is never the whole connection identity.
- **It concentrates nothing new.** The control plane already holds every connection secret; anything that could register a key on every connection would already be equivalent to a CA. The CA key doesn't even sit with platform-mesh-operator: it stays in a Secret only cert-manager reads, so it doesn't widen what a control-plane compromise yields.
- **Short certificates replace revocation.** A compromised shard's certificate dies within a day, and the CA stops issuing to it. No per-connection fan-out is needed.
- **One certificate is safe for every connection.** A handshake with one Provider can't be reused or forwarded to another, because it's bound to that Provider's pinned server key.

**Automatic rollover** ([04 Connector CA](./xxx-notes-011-v5-impl-04-connector-ca.md)):
- **A CA is a keypair plus a certificate.** The private key signs connector certificates and never leaves PM. The certificate wraps the public key with a name, a validity period and constraints (basic constraints, key usage). Providers pin the **certificate**, so they get all of it, and verify with standard library code.
- **cert-manager manages the CA**: one `Certificate` with `isCA: true`, valid 1 year and renewed 90 days before expiry, with a new key each time. It's self-signed by default, or issued by an external PKI (Vault, a cloud CA) through `issuerRef`, which makes it an intermediate that Providers pin instead of the external root. A stolen CA key is useful for at most about a year.
- **Renewing the CA's key is safe because nobody uses the new CA before Providers trust it.** platform-mesh-operator reads each new CA certificate from cert-manager's `CertificateRequest`s (public data, no key access), adds it to the trusted set and pushes the set to every connection within minutes. Each connector switches to a certificate from the new CA only once every connection platform-mesh-operator reaches pins it, or when its current certificate is about to expire. 48 hours after the renewal, platform-mesh-operator drops the old CA, connection by connection over the next 6 hours; at each Provider, sessions still on the old CA's certificate are closed with go-away and reopened at once, so their relists are spread too. Everything platform-mesh-operator needs is derived from the CA's `CertificateRequest`s, which cert-manager keeps; it stores nothing itself, and its tamper check only considers unexpired CAs, of which PM has at most two.
- **Nothing waits for connections platform-mesh-operator can't reach.** A Provider that's down catches up on its first successful control call, however many rollovers it missed: control calls don't depend on the connector CA, and the Provider-side accepts a push even if every CA it pins has expired. A connection whose control path is broken loses its sessions once connectors switch, until the control path works again; owners see why through the edge-diagnosis reasons. Only a connection that also can't authenticate (a lost or rejected secret) needs its Provider's administrators to re-pin the bundle out-of-band (see "Recovery").
- **Leaves are short-lived too.** Connector certificates are nobody's pin, so they use `rotationPolicy: Always` and a 24-hour lifetime: a new key on every renewal.

**The `ConnectorCABundle`** is a status-only singleton in PM's runtime cluster: the trusted CAs with their fingerprints, whether each has converged, per-shard counts of connections pinning the current CA and of connections not reached, and the conditions `Valid` and `Converged`. Connectors read it to decide when to switch certificates ([04 Connector CA, "ConnectorCABundle"](./xxx-notes-011-v5-impl-04-connector-ca.md)).

**Distributing the connector CA to Providers:**

Giving an external relying party CA certificates through an API is an established pattern, in two forms:
- **Push: the relying party offers an API to register trust anchors.** AWS IAM Roles Anywhere lets a customer register its own CA as a trust anchor (`CreateTrustAnchor`, `UpdateTrustAnchor`), authenticated with the customer's account credentials; workloads then authenticate with certificates from that CA. Google Cloud's workload identity federation takes an X.509 trust store the same way. Business-to-business mTLS integrations (payment providers, banks) have partners upload their client CA, usually by hand.
- **Pull: the issuer publishes its trust material, and relying parties fetch it periodically.** SPIFFE federation exchanges trust bundles (CA certificates as a JWKS with `x5c`) through bundle endpoints, with a sequence number and a refresh hint, and rotates by overlapping generations. OIDC JWKS and SAML metadata URLs do the same for signing keys. Pull is the more common model for ongoing rotation.

v5 pushes because **the PM-side may be behind NAT**: Providers can't fetch anything from PM, only PM can open connections. Registration and `PUT /connector-ca-bundle` are the pull model turned around, authenticated with the credential that controls the whole connection (the onboarding token, then the connection secret), as AWS authenticates `UpdateTrustAnchor` with the account's credentials. Structurally, v5's registration plus rollover is the same as Roles Anywhere's trust anchor plus update.

Three choices follow from this:
- **CA updates travel only on the control channel**, never over a session. A session authenticated by one CA could announce the next and save a control call, but then a stolen connector certificate could plant a CA of the attacker's choosing, turning a day of compromise into a permanent one.
- **Updates are compare-and-swap**, so PM always knows what it replaces and can always put its own bundle back. `GET` returns the pinned bundle's `etag` and fingerprints, and `PUT` needs `If-Match`: reordered or duplicated calls (for example around a leader failover) fail with `412` and re-read, and PM can always restore the right bundle. A pinned CA that's unexpired and isn't one of PM's unexpired CAs can only come from someone holding the connection secret: the controller restores the bundle, changes the secret through the `ProviderEndpoint` (which makes the stolen secret useless) and records an event on the `Provider` ([03 Operator, "Connector CA trusted set"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)). PM knows its unexpired CAs from the CA's `CertificateRequest`s, without storing anything; an expired pinned CA verifies nothing, so it's never treated as tampering, whoever issued it, and a Provider returning with an old CA is simply caught up ([04 Connector CA, "The trusted set"](./xxx-notes-011-v5-impl-04-connector-ca.md)).
- **A bundle change also ends sessions it no longer covers.** Sessions are only verified when they open, so the Provider-side re-checks its open sessions whenever it accepts a new bundle, and closes (with go-away and a short grace period) those whose certificate no longer chains to it. See "Sessions opened with a leaked key" in the threat model for why, and for its limits.
- **The CA set is a PEM bundle**, like every other CA bundle in Kubernetes (`caBundle` fields, cert-manager's `ca.crt`), parsed strictly by the SDK ([01 Protocol, "Connector CA bundle"](./xxx-notes-011-v5-impl-01-protocol.md)). Only the SDK consumes it, so a format other tooling recognises, such as SPIFFE's JWKS, would add a second copy of each key next to its certificate and buy nothing.

**What it removes**, compared with keys registered per shard: no registration call per shard or per key rotation, no self-registration objects, no control-plane involvement when shards are added or credentials renew, and no state in the connector. Calls to Providers happen only for connection lifecycle and the yearly push of a renewed connector CA.

**Options considered:**

| | Per-shard keys registered per connection | **Connector CA with short-lived connector certificates** (chosen) | One PM key everywhere |
|---|---|---|---|
| Connector state | mounted key | mounted, auto-renewed certificate | mounted key |
| Calls to Providers when a shard is added | one per connection | none | none |
| Calls to Providers when shard credentials rotate | two per connection | none | per connection, when the key rotates |
| Blast radius of one shard's credential | that shard's sessions, until removed from every connection | that shard's sessions, until the certificate expires (a day at most) | every shard's sessions |
| Anchor | the control plane (registers keys everywhere) | the CA key | the key itself |

### Sessions: a stream multiplexer over a WebSocket

**What really differs from a normal client-go connection.** A regular client-go client already sends all requests and watches to a host over one HTTP/2 TCP connection. It already accepts head-of-line blocking, shared flow control, and losing every watch (and relisting) when that connection dies. kube-apiserver even sends GOAWAY on purpose (`--goaway-chance`) to rebalance clients. None of that is new here. Two things are:

1. **The client can't open connections.** client-go normally dials whenever it needs to: a new host, a full or dead connection, or an upgrade (`exec` uses a separate HTTP/1.1 connection). Behind NAT, only the connector can create sessions. A design that hands the Provider-side a fixed set of connections (e.g. HTTP/2 directly on the WebSocket) has to replace client-go's pooling, health checks and upgrade handling with custom code.
2. **The path isn't under PM's control.** Traffic crosses the internet and the Provider's L7 edge, with its own idle timeouts, maximum connection age and WAF rules, and higher round-trip times than in-cluster clients.

**A multiplexer fixes the first.** Each stream is a connection the Provider-side opens on demand, so client-go's standard machinery works unchanged. The HTTP version on each stream then hardly matters: HTTP/1.1 is the natural default over plain streams; HTTP/2 without TLS (h2c) would also work.

- **Provider-side**: the session handler accepts the WebSocket, runs the inner TLS server (verifying the connector certificate), reads the hello, and runs the multiplexer inside the inner TLS. The session is registered under `{id}` with the hosts from its hello; deleting the connection closes it. The connection's `http.Transport` uses a dial hook that opens a stream on a session serving the target host:

  ```go
  tr := &http.Transport{
      // Called for https:// URLs. The stream isn't TLS, so the transport
      // speaks plain HTTP/1.1 on it; the real kcp URL stays in the request.
      DialTLSContext: func(ctx context.Context, _, addr string) (net.Conn, error) {
          return sessions.OpenStream(ctx, conn.ID, addr) // picks a session serving addr; the request's Host says where it goes
      },
  }
  ```

- **PM-side**: the connector opens the WebSocket, runs the inner TLS client and the other end of the multiplexer. Every stream the Provider-side opens is handed to an in-process `http.Server`, whose handler is the reverse proxy below (`httputil.ReverseProxy`, which also proxies upgrades).
- **The multiplexer is yamux.** It has per-stream flow control windows (a slow reader on one stream doesn't stall others or force unbounded buffering) and a go-away. remotedialer, built for exactly this direction, was the other candidate, but its per-stream backpressure and go-away were unconfirmed.
- **Only the Provider-side opens streams**: the side that accepted the WebSocket dials through the side that opened it. The connector refuses streams it didn't expect and never opens any.
- **Each stream carries plain HTTP/1.1 from its first byte.** The connector checks each request's `Host` against its allow-list; the Provider-side uses the target address only to choose the session.
- **Routing by host.** A shard's session serves that shard's virtual-workspace host (from its hello) and the front-proxy. Front-proxy traffic may go over any session; the front-proxy routes it to the right shard anyway. Endpoint-slice URLs are per shard, so the Provider's code sees ordinary URLs.
- **One session per connector replica.** Every replica of a shard's connector keeps its own session for each connection the shard needs, so there are usually several per (connection, shard). The dial hook spreads new streams across sessions serving the same host.
- **Sessions and streams in practice.** The Provider's controller wants to watch `cowboys` on shard B. client-go asks the SDK for a connection to shard B's virtual-workspace host. The SDK picks one of the sessions serving that host and opens a stream on it. client-go sends `GET …/cowboys?watch=true` over that stream; the connector authorizes it, adds the SA token and forwards it to kcp; the watch events stream back on the same stream for hours. Other watches and requests on shard B get their own streams, on the same session or another one serving that host. A session is set up once and is costly to open (a WebSocket through the Provider's edge, a TLS handshake); a stream is a yamux frame ([01 Protocol, "Sessions and streams"](./xxx-notes-011-v5-impl-01-protocol.md)).
- **Liveness**: WebSocket pings at the outer level every 30 seconds, so ingresses see traffic, and yamux keepalives inside. client-go's own HTTP health checks work as usual.
- **Graceful replacement.** The connector replaces sessions before the Provider's edge cuts them at its maximum connection age: it opens a new session, then sends yamux's go-away on the old one, so the Provider-side stops opening streams there, and closes it once in-flight requests finish. The Provider-side may send go-away too; the connector then reconnects at once, with jitter. An abrupt cut by the edge still drops all watches on that session at once, as with any apiserver connection.
- **No assumption about Provider replicas.** Two sessions of one connection may end on different Provider replicas; a standby answers the upgrade with `503` and `Retry-After`, which the connector takes as the lower bound for its normal reconnect backoff.
- **Why not a dial proxy to kcp** (remotedialer or Konnectivity): there the stream carries opaque TLS from the Provider-side to kcp, so the Provider-side has to hold kcp credentials and trust kcp's CA. Here the same kind of stream ends at the connector's HTTP server, which owns both.

### The Provider's edge: inner TLS on sessions

Moving the kcp credential to the PM-side moves the authority from a credential onto the session: whoever holds the Provider's end of a session can act as the Provider in kcp. With sessions on a plain WebSocket, that end would be the Provider's **inbound edge**, where the outer TLS ends, not its runtime. v5 therefore runs inner TLS on every session and control session, so the Provider's end is its runtime.

**Today and v5** (`══` encrypted, `──` plain):

```
TODAY: RFC 006 static kubeconfig (baseline)

  Provider cluster                                      PM
  ┌─────────────────────────────────┐
  │ controllers                     │
  │   holds: admin SA token         │══ TLS, end to end, outbound ══▶ front-proxy / shard VW ──▶ kcp
  │                                 │
  │ ingress / CDN (inbound)         │   not on this path
  └─────────────────────────────────┘

  acts as the Provider in kcp:    whoever holds the token (the Provider's runtime)
  can fake kcp to the Provider:   nobody (TLS to kcp's serving certificates)
```

```
v5: inner TLS on sessions (the edge is outside the trust boundary)

  PM                                        │ internet │   Provider
                                            │          │                           ┌──── trust boundary ────┐
  kcp ◀── APIExport VW ◀══TLS══ connector ══╪═ outer TLS (WebSocket) ══▶ ingress/CDN ──▶ SDK ──▶ controllers │
                              adds SA token │          │                outer TLS  │                         │
                                            │          │                 ends here │                         │
                                connector ══╪═ inner TLS 1.3 ════════════════════════▶ SDK                    │
                                            │  client: connector certificate (chains to the pinned connector CA)   │
                                            │  server: pinned server key                                     │
                                            │    ╚═ multiplexer + plain HTTP, inside the inner TLS           │
                                            │          │                           └─────────────────────────┘

  ingress/CDN sees:               opaque bytes
  acts as the Provider in kcp:    the runtime only
  can fake kcp to the Provider:   nobody without a connector certificate (PM's side)
  sees the connection secret:     nobody (control calls run inside the inner TLS too)
```

Without the inner TLS, the edge could open its own streams and act as the Provider in kcp, fake kcp's responses, and read the connection secret and consumers' data.

**Why the edge matters, even though Providers usually trust it:**
- **The edge is often someone else's.** A third-party CDN or WAF, or a central platform team's shared ingress that the service team doesn't control. In-cluster ingress controllers often hold cluster-wide access to Secrets and serve many tenants; vulnerabilities in them have allowed remote code execution (ingress-nginx's "IngressNightmare", 2025).
- **Edges keep data without anyone being malicious.** WAFs inspect bodies, CDNs sample and log requests, and people capture traffic while debugging.
- **Compliance.** Some regulated Providers must keep certain data encrypted end to end, or out of a third party's jurisdiction.
- **It's PM's consumers' data.** Today a Provider's reconcile traffic goes outbound to PM and never crosses its inbound edge. With sessions it does: consumers' objects for the Provider's APIs, and whatever the export claims in their workspaces. Consumers trust the Provider to process that data; they didn't choose its CDN.
- **PM can't check it.** "The Provider's edge is the Provider's responsibility" is an assumption PM would place on every Provider, and some can't meet it.

**Design.**
- **Sessions.** The upgrade is a plain WebSocket upgrade. Inside the WebSocket, the connector and the Provider SDK run a TLS 1.3 handshake. The connector's client certificate is its connector certificate, which the SDK verifies as in "Session credentials"; that handshake is the session's whole authentication, and the connector's hello follows inside it. The SDK's server certificate is self-signed for its **server key**; the connector checks it against the connection's pinned server keys. ALPN names the session protocol and version (e.g. `pm-session/1`). The multiplexer runs inside the inner TLS. Connector certificates carry EKU `clientAuth` and key usage `digitalSignature`.
- **Server keys.** The Provider-side returns `serverKeys` in the `POST` response. The controller writes them into the `ProviderEndpoint`, so stateless connectors can read them; the `ProviderEndpoint` is PM-written only (see "Connector policy"). The list allows rotation with overlap: the Provider-side announces a new key in `GET`, the controller updates the `ProviderEndpoint`, and the Provider-side switches a connection to the new key only once that connection's `ProviderEndpoint` lists it, which it reads over sessions anyway. So a controller that's down during the overlap doesn't cost the connection its sessions. Keys are never re-read over plain HTTPS: that would let the edge swap them. After a lost key, a PM administrator supplies the new one (see "Recovery").
- **Control calls.** The controller makes them on a control session (`GET /v1/connections/{id}/control`): a WebSocket with inner TLS, server key pinned, no client certificate, and the connection secret sent as a bearer on each call inside it, over HTTP/2 or HTTP/1.1 by ALPN. Only `POST /v1/connections` stays plain HTTPS; it carries the onboarding token (spent) and public material only.
- **TLS layers.** Three on traffic between the Provider's runtime and kcp: inner (runtime ↔ connector), outer (connector ↔ edge, around the inner) and connector ↔ kcp. The extra layer's CPU cost is small next to the virtual workspace's decoding.

**What it costs.** The Provider SDK terminates TLS inside the WebSocket and pins keys in both directions; the Provider-side keeps a server key in a Secret, which the SDK generates and rotates with overlap; the `ProviderEndpoint` carries `serverKeys`; one more handshake per session. Most of it is SDK code PM writes once.

**What it gives.** The authority to act as the Provider in kcp, the integrity of kcp's responses to the Provider, the connection secret and PM consumers' data never cross the Provider's edge readable. That holds by construction, whatever each Provider's infrastructure.

**Inner TLS is required on every session and control session**, with no opt-out: one mode is simpler to test and reason about.

### The connector as an authenticating proxy

For every request arriving over a session of connection `{id}`:

1. **Check the target.** The request's host must be on the allow-list: this shard's virtual-workspace host and the front-proxy host, both **from the connector's deployment configuration**. Nothing else, and nothing a Provider or an owner can influence. Resolve once, check every address, dial those exact addresses; refuse link-local, multicast and cloud-metadata addresses.
2. **Strip credentials and identity headers**: `Authorization`, `Impersonate-*`, `Proxy-*`, cookies and hop-by-hop headers. Otherwise the Provider-side could ask kcp to act as someone else, using the connector's token.
3. **Add the SA's identity** for connection `{id}`: a bound token from TokenRequest, minted by the connector, cached in memory, refreshed at half its TTL, never written anywhere and never sent to the Provider-side. Impersonating the SA instead was checked and rejected: it works for the provider workspace but not for the APIExport virtual workspace (observation 5).
4. **Forward** to the real URL over TLS, verified against the CA bundle PM's own components use for kcp. Rotating kcp's CAs needs no coordination: if PM trusts its kcp, the connector does. The dial maps the external kcp host to its in-cluster Service, keeping `Host` and the TLS server name, so traffic doesn't hairpin through PM's load balancer. Responses are streamed without decoding objects.
5. **Log and limit** per connection: method, path, status, duration; rate limits, concurrent-stream limits and header-size limits per connection, so one Provider can't exhaust the connector for others. The connector is in the same position as kcp's front-proxy towards its clients, and needs the same defences.

- **RBAC is unchanged**: kcp sees the connection's SA, with the RBAC platform-mesh-operator created (RFC 006), admin in the provider workspace. The connector doesn't widen it; it narrows it by its path restrictions (see "Connector implementation") and the denylist in "Connector policy". platform-mesh-operator also grants the connectors' own identity `create` on `serviceaccounts/token` for that SA; that's the connector minting tokens, which the denylist doesn't affect.
- **Upgrade requests** (`exec`, `attach`, `port-forward`) are refused by the connector policy. The transport could carry them (HTTP/1.1 upgrades on a stream), so allowing them later is a policy change, not a design change.
- **Egress to Providers.** Connectors dial base URLs from `ProviderEndpoint`s. The controller validates them before writing a `ProviderEndpoint`, and connectors enforce the same rules when dialling: `https` only, public addresses only, resolve once and check. `ProviderEndpoint`s are PM-written only (see "Connector policy"), so this is defense in depth against a bad URL reaching PM's internal network.

### Connector policy: what an admin Provider can't do

The Provider is admin in its provider workspace (RFC 006) and free to deploy whatever it needs, so PM doesn't restrict its RBAC there. But in v5 the connector is the Provider's **only path into kcp**: the Provider-side holds no kcp credentials, and every request it makes passes through the connector's handler chain. Two mechanisms apply on top of the Provider's RBAC.

**`ProviderEndpoint`s: kcp's maximal permission policy.** `ProviderEndpoint`s must stay PM-written, so the Provider can't redirect connectors or take another connection's identity. This doesn't need the connector: the connection export sets `maximalPermissionPolicy.local`, and kcp checks every request to a resource bound from that export against RBAC in **PM's export workspace**, with the user and groups prefixed `apis.kcp.io:binding:` (`kcp/pkg/authorization/maximal_permission_policy_authorizer.go`). The check runs in each shard's authorizer chain before local RBAC, so admin in the provider workspace doesn't bypass it, and the Provider has no rights in the export workspace to widen it. PM grants there:
- `get`, `list`, `watch` on `providerendpoints` to the group `apis.kcp.io:binding:system:authenticated`;
- writes (`create`, `update`, `patch`, `delete`, `deletecollection`, including `/status`) only to `apis.kcp.io:binding:<platform-mesh-operator user>`, and to any other PM component that writes `ProviderEndpoint`s.

kcp's own e2e test shows the pattern: a consumer-workspace admin is refused a write the export's policy doesn't grant (`kcp/test/e2e/apibinding/maximalpermissionpolicy_authorizer_test.go`, `TestMaximalPermissionPolicyAuthorizer`). Because kcp enforces it, it holds on every path, including a leaked connector SA token, and the connector's authorizer needs no rule for `ProviderEndpoint`s (observation 8).

**Everything else: the connector's denylist.** The policy only covers resources served by the export, not `apis.kcp.io` system resources or core resources. For those, the connector's authorizer applies a small **denylist** for the connection principal:

| Denied | Why |
|---|---|
| `update`, `patch` and `delete` on the connection export's APIBinding (a fixed name, set by platform-mesh-operator) | The Provider can't drop the binding or reject its permission claim. Deleting the binding also deletes the `ProviderEndpoint`s, and with them the policy's protection |
| Writes to `apiexportendpointslices/status` | kcp maintains that status; forging it is the only way to make connectors open sessions on shards without consumers. kcp's system-CRD authorizer only protects the status of `apiexports` and `apibindings` |
| `create` on `serviceaccounts/token` | No TokenRequest: the Provider never gets a kcp credential it could keep |
| `create` on `certificatesigningrequests`, and their approval | Closes the other route to a credential, if kcp signs CSRs |
| Upgrade requests (`exec`, `attach`, `port-forward`) | Not needed to reconcile an APIExport |
| Writes of Secrets of type `kubernetes.io/service-account-token` (create, update, patch, server-side apply) | kcp issues tokens for them (platform-mesh-operator uses them for local Providers), so they're a token-minting path. The authorizer can't see a body's type, so the proxy handler inspects Secret writes before forwarding |
| `create` on `workspaces` below the provider workspace | Not needed to reconcile an APIExport, and avoids unbounded workspaces under `:root:providers` |

Everything else, the Provider's own exports, schemas, endpoint-slice specs, RBAC and arbitrary resources, passes through unchanged, subject to its admin RBAC.

**The invariant the denylist rests on: the connector is the Provider's only path to kcp.** `ProviderEndpoint` protection doesn't depend on it. That makes these requirements:
- **No static kubeconfig for remote Providers**: RFC 006's kubeconfig Secret is not generated or handed out for them.
- **No token minting through the proxy**: the TokenRequest, CSR and SA-token Secret rules above.
- **`direct` mode, if ever added, goes through the same connector virtual workspace**, so it hits the same authorizer.
- **PM gives the Provider no second credential** for its workspace. Role bindings the Provider creates for others are its own business.

**Implementation notes:**
- The authorizer needs the original request's group, resource, subresource, verb, name and cluster, for both kcp path shapes the Provider uses: `/clusters/<id>/…` through the front-proxy and `/services/apiexport/…` through a shard's virtual workspace. The generic request-info parser doesn't understand kcp's cluster prefix, so use kcp's resolver or parse both shapes explicitly.
- The denylist is the security guarantee, so it needs thorough tests, including every verb and both path shapes. Denied attempts are audited.
- The owner and every other kcp user reach the provider workspace through the front-proxy with their own credentials, outside the denylist but still under the export's policy. The owner has no rights in the provider workspace; only PM's own components and the Provider (admin, through the connector) do.

### Connector implementation: a virtual workspace

The connector has two halves. Only one is request-shaped, and that one maps directly onto kcp's virtual workspace framework.

**What already exists** (checked on 2026-10-08/09):
- **PM's virtual-workspaces server** (`platform-mesh/services/virtual-workspaces`) is built on `kcp-dev/virtual-workspace-framework` and serves `/services/marketplace` and `/services/contentconfigurations`. It builds its own authenticator union (`cmd/start.go`: `union.New(authentication.New(clientCfg), …)`), so a new authenticator needs no change in kcp. It's deployed as a plain Helm Deployment (`pm-helm-charts/charts/virtual-workspaces`), a singleton reached through the front-proxy's `additionalPathMappings`, talking to kcp through the front-proxy as `kcp-admin` (`system:kcp:admin`), with no cache-server access. The connector reuses its code, not its deployment.
- **kcp runs its virtual workspaces per shard.** Each shard's `spec.virtualWorkspaceURL` points at that shard's own virtual-workspace server, which kcp-operator exposes under the shard's external, SNI-routed hostname or an external `VirtualWorkspace` (`kcp-operator/internal/resources/compiledshard/deployment.go`, `--shard-virtual-workspace-url`). APIExportEndpointSlice URLs are built from it (`kcp/pkg/reconciler/apis/apiexportendpointsliceurls`), so endpoint traffic goes straight to each shard's virtual workspace, bypassing the front-proxy.
- **kcp's initializing-workspaces virtual workspace already does the proxy half.** Its workspace-content handler (`kcp/pkg/virtual/initializingworkspaces/builder/build.go`, a `handler.VirtualWorkspace` with a `HandlerFactory`) authorizes the caller, strips `Authorization` and `Impersonate-*` headers, impersonates a chosen identity and reverse-proxies to the shard, watches included (`kcp/pkg/virtual/shared/proxy.go`, `ServeProxy`).
- **kcp-operator's `VirtualWorkspace` resource** (v0.9+) deploys an external virtual-workspace server: Deployment, Service, certificates, a kubeconfig identity, and a target of the root shard or one shard (`shardRef`). PM's charts already use it for `kcp-access-vw`. When an external cache server is configured, which PM does, it also mounts the cache-server kubeconfig and client certificate (`kcp-operator/internal/resources/compiledvirtualworkspace/deployment.go`), see observation 7.

**Design:**

```
            connector virtual-workspace server, one per shard (one process each, stateless)
 ┌──────────────────────────────────────────────────────────────────────────────┐
 │ session controller (post-start hook)    in-process listener per connection   │
 │   watches the connection export,          ConnContext: connectionID          │
 │   uses the mounted connector certificate                                     │
 │   as inner TLS client certificate                                            │
 │   dials WebSocket ──▶ Provider            │                                  │
 │   inner TLS, hello, yamux: streams ───────┘                                  │
 │                                           ▼                                  │
 │   full handler chain (authn → authz → audit → timeouts → …)                  │
 │     authn: session authenticator reads connectionID from context             │
 │     path:  /services/providerconnections/<id>/<original path>, Host kept      │
 │                                           ▼                                  │
 │   providerconnections VW handler: check Host, strip auth headers,             │
 │   add SA identity, reverse-proxy ──▶ this shard's VW (local) / front-proxy    │
 └──────────────────────────────────────────────────────────────────────────────┘
```

1. **Session controller.** Not a virtual workspace: a controller in the same process, started from a post-start hook, the way kcp's APIExport virtual workspace runs its controllers. It watches the connection export (`ProviderEndpoint`s and claimed endpoint slices) and decides which sessions this shard needs (see "Per-shard connectors"), dials the sessions of the connections this replica owns (see "Connector replicas") and accepts multiplexer streams. It serves each stream with an in-process `http.Server` whose handler is the server's **full handler chain**, not the virtual workspace's handler directly, so authentication, authorization, audit and timeouts all apply. `ConnContext` tags each stream with its `connectionID`, and the server prefixes the path with `/services/providerconnections/<id>`, keeping the `Host`.
2. **Session authenticator.** A new member of the authenticator union maps the `connectionID` in the request context to the connection principal. The value is set only on in-process listeners, so nothing arriving over the network can forge it, and requests from the front-proxy never carry it.
3. **The `providerconnections` virtual workspace.** A `handler.VirtualWorkspace` following the initializing-workspaces pattern:
   - its **authorizer** lets the connection principal reach only its own connection's paths: its provider workspace through the front-proxy, and the APIExport virtual-workspace endpoints of the exports in that workspace. It also enforces the denylist in "Connector policy";
   - its **handler** checks the `Host` against the allow-list, strips client credentials (`ServeProxy`'s header list plus `Proxy-*` and cookies), adds the SA's identity, and reverse-proxies to the real URL.
4. **No status writes.** Per-session health is in metrics. Connection health reaches the `Provider` through the controller, which asks the Provider-side (see "Control plane: platform-mesh-operator").

**What the connector needs at runtime**: mounted, its connector certificate and key, its own shard's virtual-workspace URL and the front-proxy URL, with in-cluster addresses for both; read-only, `apiexports/content` on the connection export (`ProviderEndpoint`s and claimed endpoint slices); write, TokenRequests; in the runtime cluster, read access to the `ConnectorCABundle` (public data, for checking a renewed certificate's chain). No `Shard` objects, no `/services/admin`, no shard-wide reads, no use of the cache server, no persistent state. Its kcp identity comes from the `VirtualWorkspace`'s `kubeconfigSecretRef`, which must always be set (otherwise kcp-operator mounts its privileged logical-cluster-admin credential); all shards' connectors share one group.

**What this gives:**
- **Placement**: one connector virtual-workspace server per shard, deployed through a kcp-operator `VirtualWorkspace` with `shardRef`, next to that shard's own virtual workspaces. The `providerconnections` chart renders one per entry in its `kcpTopology` values, with the connector certificate mounted through `extraVolumes`.
- **A full serving stack**: audit (each request recorded with the connection principal and the SA it acts as), long-running request detection, timeouts, max-in-flight limits, panic recovery and `apiserver_request_*` metrics, instead of a hand-written proxy.
- **Request policy in one place**: the virtual workspace's authorizer, instead of ad-hoc checks in proxy code.

**Identity: SA tokens, not impersonation.** The connector mints TokenRequest tokens (step 3 above) for all traffic. Impersonating the SA looked attractive here, since the initializing-workspaces pattern impersonates, but it only works for half of the traffic (observation 5):
- **Provider workspace (front-proxy, then shard): works.** Needs `access` on `/` and `impersonate` on `serviceaccounts` with `resourceNames: [<sa>]` in the provider workspace; no user extra.
- **APIExport virtual workspace (standalone server): doesn't work.** Per-cluster requests are denied, and the grant that would allow wildcard requests lets the connector impersonate anyone.

Tokens are needed for the APIExport virtual workspace anyway, so a mix would add RBAC and code paths without removing anything.

**Caveats:**
- **Timeouts.** The generic handler chain applies a timeout (60 s by default) to requests it doesn't classify as long-running. Large proxied LISTs may exceed it; the long-running check and timeout must be configured for proxied traffic.
- **Hop limit.** Proxying into the APIExport virtual workspace counts toward kcp's `X-Kcp-Virtual-Resource-Hops` limit (maximum 4); to be confirmed.
- **`ServeProxy` is kcp-internal** (`kcp/pkg/virtual/shared`), not part of the framework module. The connector copies it (about 30 lines).
- **Cache-server credentials come along** when deployed as a kcp-operator `VirtualWorkspace` in PM's setup (observation 7). The connector doesn't need them; this is accepted and documented as part of a compromised connector's blast radius.
- **Buffers.** A small copy buffer per stream, and a cap on the upstream HTTP/2 receive buffer per stream (Go's default is 4 MB), so a slow Provider's backlog stays upstream.

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
| Carries | `Provider`, `ProviderOnboardingRequest`; the token Secret sits next to them | one `ProviderEndpoint` per connection | the Provider's own services |
| Exported by | PM | PM, from `:root:providers:system` | the Provider-side, from `:root:providers:<Provider name>-<Suffix>` |
| Bound in | the owner's workspace, `:root:orgs:<Org>:<User>` | each provider workspace, by platform-mesh-operator | consumers' workspaces, on any shard |
| Written by | the owner (request), PM (status) | PM only, through the export. The Provider is admin in its workspace, but the export's maximal permission policy denies it writes to `ProviderEndpoint`s | the Provider (exports), consumers (bindings) |
| Read by | platform-mesh-operator | every shard's connector, through the export's endpoint slice | kcp, which lists the shards with bindings in each Service endpoint slice; connectors read those slices through (2)'s permission claim |
| Decides | nothing about sessions | where the front-proxy session lives (the shard hosting the `ProviderEndpoint`) | where endpoint sessions live (shards listed in the slices) |

**Load.** In today's direct model, a Provider's endpoint traffic (wildcard LIST and WATCH per resource, per-workspace reads and writes) goes straight to each shard's virtual workspace; only its workspace traffic (the endpoint slice and a few objects) uses the front-proxy. v5 sends the same requests to the same targets with the same SA identity, so kcp's load doesn't change; connections to kcp now come pooled from the connectors instead of from each Provider. What's added is a hop: every request and response byte passes a connector, which parses HTTP, authorizes, audits, injects the token and copies the response, including watch streams held for hours. Each watch holds two streams on the connector, roughly a few goroutines and a 32 KB copy buffer each. Per shard, this is the same shape as the virtual-workspace load kcp already carries, plus the per-request L7 overhead. Any NAT-traversing design adds a hop.

**Compared with the standard flow through the APIExport virtual workspace** (kcp `ee9e6f3f3`, read on 2026-10-12): the connector sits in front of a hop that already exists, and that hop is the heavier of the two.

| | Standard flow | v5 |
|---|---|---|
| Requests reaching the shard's virtual workspace and the shard | per Provider × resource × shard | the same |
| Identity kcp sees | the Provider's SA token | the same token, added by the connector |
| The virtual workspace's work | full handler chain; `apiexports/content` checked by a SubjectAccessReview against kcp (`pkg/virtual/apiexport/authorizer/content.go`); every watch opens its own watch on the shard and decodes each event into `Unstructured` and re-encodes it (`virtual-workspace-framework/pkg/forwardingregistry/store.go`) | unchanged |
| Extra hop | none | the connector, on the same shard: session authentication and policy checks, both local; an audit entry; header rewrites; bytes streamed without decoding objects |
| Per long-running watch on the extra hop | — | two streams, a few goroutines and a 32 KB copy buffer each |
| Connections into PM | one per Provider and host, inbound; kcp's endpoints must be reachable from the Provider | none inbound; one outbound WebSocket per (connection, shard) |
| Bytes over the internet | LIST and WATCH payloads | the same, plus multiplexer framing |
| Extra calls to kcp | none | one TokenRequest per connection at half the token's TTL |

**In short: two virtual workspaces in a chain.** The `providerconnections` virtual workspace is a client of the APIExport virtual workspace, the way the Provider is today. It carries exactly the watches its shard's APIExport virtual workspace already serves to remote Providers, scales one-to-one with it, and does less per watch: no object decoding and no calls to kcp per request. The per-watch goroutines, buffers and flow-control windows are the kind the APIExport virtual workspace already pays; v5 adds a second, lighter copy of them. Front-proxy traffic follows the same pattern: connector → front-proxy → shard instead of Provider → front-proxy → shard.

What's really different is small:
- **Backpressure runs through the multiplexer.** Today a slow Provider is throttled by TCP and HTTP/2 flow control towards the APIExport virtual workspace. In v5 it's throttled through the multiplexer streams inside the WebSocket, which is why the multiplexer must apply flow control per stream (yamux does), with the connector's upstream HTTP/2 receive buffer capped per stream.
- **The long-lived connections change direction**: one outbound WebSocket per (connection, shard) instead of the Provider's inbound HTTP/2 connections.
- **Configuration of the second hop**: the timeout settings for proxied LISTs and watches, and whether it counts toward `X-Kcp-Virtual-Resource-Hops`.

**What per-shard gives:**
- **Load scales with shards**, as virtual-workspace servers already do.
- **Locality.** Each connector forwards endpoint traffic to the virtual workspace next to it.
- **Failure isolation.** A connector or session failure on shard B only stops shard B's endpoint traffic and relists only shard B, as when one shard's virtual workspace has trouble today. Front-proxy traffic can move to any other session of the connection.
- **Tiny allow-lists**, from deployment configuration: its own virtual-workspace host and the front-proxy.
- **Data next to what it describes.** Each connection's `ProviderEndpoint` lives in its provider workspace, on that workspace's shard, and kcp itself keeps the list of shards with consumers in each endpoint slice.

**Design consequences:**
1. **Which sessions exist.** A shard's connector keeps a session for a connection when either:
   - **the connection's `ProviderEndpoint` came from its own shard's virtual-workspace endpoint** of the connection export. Then the provider workspace is hosted on this shard, and this connector carries the connection's front-proxy session, from the start and even with no consumers yet; or
   - **one of the connection's Service endpoint slices lists its own shard's virtual-workspace URL** (known from its deployment). kcp lists a shard there exactly when it has at least one binding to the export, so this is the shard's endpoint traffic. Only slices in the provider workspace that point at an export **in that same workspace** count. Otherwise a Provider could create slices for someone else's export and make connectors open sessions on shards it has no consumers on.

   Sessions therefore follow real usage: one for the hosting shard, plus one per shard with consumers of the connection's own exports. No idle sessions on shards a connection doesn't use, and nothing the owner or the Provider can steer: the `ProviderEndpoint` is PM's, and consumers create the bindings. This assumes the Provider keeps its Service endpoint slices in its provider workspace. RFC 006 leaves APIExport bootstrap out of scope, so it's a documented requirement for Provider authors (SDK and quickstart documentation).
2. **Shards come and go with their deployments.** A new shard's connector gets a connector certificate from the connector CA and starts watching the connection export. When kcp adds the shard to a connection's endpoint slices, after the first consumer binding there, the connector dials. A removed shard takes its connector with it, and its endpoint slices stop listing it anyway. Nobody is told or registered anything. "Is there a connector for every shard?" is a deployment-health question (kcp-operator status, alerts), not something checked at runtime.
3. **Discovery: the connection export.** kcp's per-shard virtual workspaces only see their own shard plus the cache server, while `Provider` objects live in owners' workspaces on any shard. Instead of reading those, each connector consumes PM's **connection export** like any multi-shard controller: it reads the export's APIExportEndpointSlice through the front-proxy, then list/watches through every shard virtual-workspace URL in it (multicluster-runtime's endpoint-slice provider):
   - **`ProviderEndpoint`s**: every object arrives with its logical cluster, which is the provider workspace's `clusterName`. So the `clusterName → (connectionID, base URL)` mapping is just the informer's index, and which endpoint delivered the `ProviderEndpoint` tells the connector which shard hosts the workspace;
   - **the connections' Service `apiexportendpointslices`**, read-only, through the export's **permission claim**. That's how the connector learns which shards have consumers, without any shard-wide read of APIBindings.

   The `ProviderEndpoint`s are written only by PM through the export. The Provider stays admin in its workspace, as in RFC 006, but the export's maximal permission policy denies it writes to `ProviderEndpoint`s, and it reaches kcp only through the connector, whose policy denies writes to the connection export's binding and to endpoint-slice status (see "Connector policy"). So neither the owner nor the Provider can change the URL connectors dial, drop the binding or forge which shards have consumers. The owner has no rights in the provider workspace at all. The export claims nothing else in the provider workspace. An outage of one shard's virtual workspace only freezes the `ProviderEndpoint`s and slices from that shard; informers keep the last state.
4. **SA tokens.** Each shard's connector mints tokens for the connection's SA through TokenRequest via the front-proxy, since the SA lives in the provider workspace on one shard. All shards trust all shards' SA signing keys, so those tokens work against every shard's virtual workspace.
5. **Front-proxy traffic on any session.** Workspace traffic goes through the front-proxy today anyway, and the front-proxy routes it to the right shard, so every session may carry it. The hosting shard's session (point 1) guarantees there's always one, even before any consumer binds; that's what lets the Provider watch its own exports and slices from the start. Which shard that is comes from the connection export itself, not from any lookup.
6. **Provider-side routing.** The dial hook picks a session by target host: the front-proxy can use any session; a shard's virtual-workspace host uses that shard's sessions. Because sessions open by the same rule that puts shards into endpoint slices, a URL can briefly appear in a slice before its session is up; client-go retries. Sessions for different shards could land on different Provider replicas, while a replica running the endpoint-slice providers needs all of a connection's hosts. So the Provider runs **one active replica** of the connection server and its controllers, by leader election; standbys answer upgrades with `503` and `Retry-After: 5`, and the edge must not pin the session paths to one backend. Horizontal scale on the Provider-side is a later, opt-in SDK mode that needs no PM-side work ([follow-ups](./xxx-notes-011-v5-impl-todo.md)).
7. **Readiness without topology.** The Provider-side reports, per host it actually needs, whether a session covers it and reaches kcp (`GET /v1/connections/{id}`). The controller turns that into one condition. Nobody needs the shard list. At onboarding, before any consumer binds, that's just the front-proxy.
8. **Deployment and `direct` mode.**
   - One kcp-operator `VirtualWorkspace` per shard (`shardRef`), plus a cert-manager `Certificate` for its connector certificate from the connector CA, both rendered by the new `providerconnections` chart from an explicit `kcpTopology` in its values. That list duplicates the `infra` service's shards and must match them; platform-mesh-operator's profile validation will enforce it as a follow-up, and a shard runbook covers it until then ([09 Deployment, "Charts"](./xxx-notes-011-v5-impl-09-deployment.md)). PM's marketplace and content-configuration virtual workspaces stay on their singleton server.
   - The connection export lives in `:root:providers:system`; platform-mesh-operator binds it (accepting its one permission claim) in every provider workspace it creates (RFC 006).
   - **`direct` mode should use per-shard URLs, not the front-proxy.** The front-proxy routes only `/clusters/` by shard; every other path mapping, `/services/...` included, goes to one fixed backend (`kcp/pkg/proxy/mapping.go`: `// TODO: handle virtual workspace apiservers per shard`). Putting the `clusterName` in the URL (`/services/providerconnections/clusters/<clusterName>/…`) would let the front-proxy's cluster lookup find the provider workspace's shard (`kcp/pkg/proxy/lookup/lookup.go` already extracts clusters from `/services/<name>/clusters/<cluster>/…`), but it wouldn't choose the backend without that TODO being done. And it would only ever reach one shard, while endpoint traffic needs every shard's connector, including for wildcard requests that name no cluster. Instead, expose each shard's connector under its own external hostname, as kcp-operator already does for per-shard virtual workspaces (`VirtualWorkspace` `external`), and let the Provider-side use one URL per shard, mirroring its endpoint slices. The `clusterName` in the path is still useful as a consistency check and audit field.

### Connector replicas

Within one shard, the connector runs several replicas, and **every replica keeps a session for every connection the shard needs**. Replicas don't know about each other.

- **Sessions aren't exclusive**, so this needs no coordination: no membership, no assignment, no Lease per connection. Each replica applies the session rule on its own.
- **Availability.** A crashed replica cuts only the streams on its own sessions; client-go reopens them on the other replicas' sessions at once. Rollouts go one replica at a time without a gap: a leaving replica sends the multiplexer's go-away and drains its streams.
- **Spreading traffic.** The Provider-side spreads new streams across sessions serving the same host, so requests and watches, the costly part of a connector's work, spread over the replicas. Each replica still holds a session per connection, mints its own SA tokens and watches the whole connection export.
- **The cost is at the Provider**: up to R sessions per (connection, shard), where R is the replica count, typically 2–3. Watches and requests aren't multiplied; sessions are, with their idle overhead and the Provider edge's connection limits. Typical Providers hold single digits of sessions per PM installation.
- **No registration of any kind.** Every replica uses the same mounted connector certificate; the opaque `sub` is for audit. Restarts, scaling and rollouts involve neither the control plane nor the Provider's control API, and leave nothing on the Provider-side to clean up.
- **If session counts at Providers ever matter more** than spreading traffic, replicas can divide connections by rendezvous hashing of `connectionID` over the live replicas. That's a contained change inside the connector, and a rollout between the two is harmless.
- **Later, with an external issuer (`issuerRef`): per-replica certificates.** Each replica generates an in-memory key at start and gets a short-lived certificate for it from the connector CA, so no private key is mounted at all. Costs an issuing round trip per replica start.

### Control plane: platform-mesh-operator

**platform-mesh-operator** owns every `Provider`, local or remote, from creation to deletion: one reconciler, one status writer, one finalizer ([03 Operator](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)). For remote Providers it also reconciles onboarding requests.

- **One active replica**, by leader election, with standbys. It runs one multicluster-runtime provider over all shards of the providers export and one over the connection export; each shard's endpoint gets its own caches, so a stuck shard only stalls its own objects. The work is mostly waiting on Providers, so it scales with reconcile concurrency, not replicas.
- **Internet-facing code in PM's most privileged component.** Talking to Providers on the internet is accepted there while the `Provider` part of platform-mesh-operator is still part of it; the connection code and protocol client sit in their own packages, with input limits and fuzz tests, so they can move with it.
- **Connection state in its own namespace, no Secret informers in kcp.** `ProviderEndpointRegistration`s and their Secrets live in its namespace in PM's runtime cluster. Token Secrets in owners' workspaces are read by name, uncached; nothing about other Secrets there enters its memory.
- **Never trusts owner-writable fields for links.** It derives the provider workspace from the `Provider`'s name and its `kcp.io/cluster` annotation, reads the `Workspace` in `:root:providers`, and takes `connectionID` and URL from the `ProviderEndpoint`, never from the `Provider`'s spec or status.

**What it does:**

| Work | When | Calls to the Provider |
|---|---|---|
| Register the connection (with the connector CA bundle), make a first control call, write its `ProviderEndpoint` through the connection export | once per connection | `POST /v1/connections`, one `GET` |
| Confirm readiness, then keep health current | every few minutes, per connection, with jitter | one `GET`, in a control session |
| Change the connection secret | on demand, after tampering, or to repair a lost or rejected secret (rate-limited) | none to change it (the hash goes into the `ProviderEndpoint`); one confirming `GET` |
| Push the connector CA trusted set | about once a year, when cert-manager renews the CA, and when a connection is reached again | one `PUT /connector-ca-bundle` with `If-Match` per connection whose pinned fingerprints differ |
| Update server keys in the `ProviderEndpoint` | when the Provider-side announces a new key | none (seen in `GET`) |
| `DELETE` | at offboarding | one call |

- **Nothing on this list is on the traffic path.** A controller outage delays onboarding, secret changes, rollovers and offboarding; sessions on every shard, including replica restarts and new shards, keep working.
- **Load is small.** With 1,000 connections and a 5-minute poll: about 3–4 `GET`s per second in steady state. Shard changes cost nothing. The calls go to external Provider endpoints, so they need jitter, a rate limit per Provider and backoff.
- **It reports aggregate status only** on the `Provider`: one condition per reconcile step (`Workspace`, `ConnectionBinding`, `ConnectorAccess`, `Registered`, `Connected`), the aggregate `Ready`, the warning `ProtocolOutdated`, and `phase` (`Pending → Provisioning → Ready ⇄ Degraded`, and `Deleting`). Failures carry a message starting with a stable label: the edge-diagnosis labels (`EdgeUpgradeRefused`, `EdgeBotChallenge`, `EdgePayloadBlocked`, `EdgeTimeout`), `ControlUnreachable`, `ConnectionUnknown`, `NoSessions`, `KcpUnreachable`. Per-shard detail stays in metrics; tampering is an event.
- **Connector CA rollovers are automatic** (see "Automatic rollover"): the controller reads each new CA certificate cert-manager issues, pushes the trusted set, and reports convergence in the `ConnectorCABundle`, which connectors read before switching certificates. Nobody takes part ([04 Connector CA, "Rollover"](./xxx-notes-011-v5-impl-04-connector-ca.md)).
- **Responding to a compromise**: for a shard, stop issuing its certificate; for the CA's key, list it as distrusted. In both cases renew the CA now, so the old CA is dropped at once and the Provider-side closes the sessions opened under it (see "Sessions opened with a leaked key").

### Provider-side: serving many PM instances

The SDK hands the Provider a client config per connection and tells it when connections come and go. The Provider's operator code, which already knows the endpoint slices of the exports it owns, creates one multicluster-runtime endpoint-slice provider per slice it's interested in, with cluster names prefixed by the `connectionID`.

- **The client config per connection.** The SDK gives each connection an `http.Transport` whose dial hook opens streams on that connection's sessions (see "Sessions"):

  ```go
  cfg := &rest.Config{
      Host:      conn.FrontProxyURL,     // real URL; the connector routes on it
      Transport: sdk.Transport(conn.ID), // no TLS options: client-go refuses both
  }
  ```

  multicluster-runtime derives per-endpoint configs (shard virtual-workspace URLs) by changing `Host`; they inherit the `Transport`, so every request for that connection goes over its sessions.
- **The SDK's connection registry** emits events when connections are added or removed, and a helper builds the per-connection endpoint-slice providers and composes them with `connectionID`-prefixed cluster names.
- **Secret changes and recovery** don't need the Provider's code to do anything: a connection keeps its `connectionID` through them, so its providers, clusters and state carry over.
- **No credentials to manage per connection**: no token, CA bundle or client rebuild on rotation; a connection is just a set of sessions.

### Changing the connection secret

Fully automatic, with no owner involvement, done by platform-mesh-operator, and **only through the `ProviderEndpoint`**, never through a call ([01 Protocol, "Changing the secret"](./xxx-notes-011-v5-impl-01-protocol.md)):

1. The controller generates a new secret and writes it to the `ProviderEndpointRegistration`'s Secret as `pendingSecret` **before** anything else, so a crash can't lose it.
2. It writes the new secret's hash into the `ProviderEndpoint`'s `pendingSecretHash`.
3. The Provider-side reads its `ProviderEndpoint` over one of the connection's sessions and makes that hash the current one; the old secret stops working.
4. The controller authenticates with the new secret (a `GET`), moves `pendingSecret` to `secret`, and clears `pendingSecretHash`.

**Why the `ProviderEndpoint`.** Only PM can write it (the connection export's maximal permission policy, see "Connector policy"), and the Provider-side reads it only over a session whose connector certificate verified against the connection's pinned connector CA, and whose hello named the connection's `clusterName`. So the `ProviderEndpoint` is a channel only PM can use. Anyone else, holding the current secret, an onboarding token or a portal account, can't change the secret, and so can't lock PM out.

- **Sessions are unaffected**: they never use the connection secret. A change needs one, though; without sessions it waits, and the old secret keeps working.
- **Lost response or crash**: on restart the controller tries `pendingSecret` first, then `secret`, and resumes; the `ProviderEndpoint` still names the pending hash.
- **When**: on demand (an annotation on the `Provider`), after tampering with the connector CA bundle (see "Distributing the connector CA to Providers"), and to **repair** a secret the controller lost or the Provider-side rejects (e.g. its store was restored from an older backup), rate-limited per connection. Whether to also rotate periodically (e.g. every 30 days) is open (see "Open items").
- **No owner-supplied key material**: the secret is always generated by the controller, which is what keeps every change automatic (observation 2).

### Recovery

Each loss has its own way back:

| What's lost | How it comes back |
|---|---|
| The PM-side's `ProviderEndpointRegistration` or its Secret, or the Provider-side's copy of the hash | The controller repairs the secret through the `ProviderEndpoint`. The `connectionID` and all Provider-side state stay; nobody has to act |
| The connection on the Provider-side (deleted in the portal, or its store lost) | The Provider-side answers "unknown connection" and the `Provider` shows `Registered=False` with `ConnectionUnknown`. **If the Provider still has the data** (or a backup of its store), its administrators import the old entry (apply its `Connection`): same `connectionID`, everything carries over. **Otherwise the owner onboards the same `Provider` again**: a new connection, with a new `connectionID`, on the same provider workspace, so consumers' bindings stay, but Provider-side data carries over only where the Provider can find it from kcp objects. The same path switches to a different Provider: the old one deletes the connection, and a token from the new one names its own base URL. Only a definite "unknown" allows it, never an unreachable Provider-side, so a new connection never appears next to a live one |
| Every session (the pinned bundle missed a whole rollover, and the secret was lost too, so the controller couldn't push the bundle) | The Provider's administrators re-pin the bundle out-of-band; then the controller repairs the secret |

**Disasters on either side are out-of-band**, between the people running PM and the Provider. Their procedures aren't specified here; the design only makes sure they remain possible:
- **`connectionID`s are stable**, and Providers key consumer data with them, so a recovered connection finds its data.
- **A connection can be imported** after the Provider-side lost its state: its administrators apply the `Connection` (`connectionID`, owner, `clusterName`, front-proxy URL, pinned bundle) with `kubectl`; the controller then repairs the secret through the `ProviderEndpoint`.
- **A PM administrator can give the controller a connection's base URL and server keys**, for a Provider rebuilt elsewhere. Never through owner-writable fields, and never fetched over plain HTTPS.
- **Removing a connection never deletes consumer data**, only deleting a kcp object does, and **bootstrap never recreates an APIExport's identity**, so bindings survive a connection going away and coming back. These are documented rules for Providers ([07 SDK, "Requirements on Providers"](./xxx-notes-011-v5-impl-07-provider-sdk.md)).

What recovery can't undo is an owner deleting the `Provider`: its workspace and the APIExport's identity go with it, and kcp can't re-pair existing bindings with a new export. Protecting `Provider`s with consumer bindings from deletion is a follow-up.

**Why consumer data is keyed with the `connectionID`**, not only with kcp's identities: anyone with a portal account can register a connection with a connector CA of their own, in effect running a fake PM, and serve made-up objects that reuse a real consumer's logical cluster name and object UID. If the Provider found external resources by those alone, the fake PM could make it delete a real consumer's resources. With the `connectionID` in the key, a connection only ever reaches its own.

### What each credential exposes

| Credential | Lifetime | Stored at | If stolen |
|---|---|---|---|
| Onboarding token | minutes, single-use | PM-side Secret until spent; hash on the Provider-side | The thief can register **their own** new connection under the owner's portal account (billing, quotas). It receives no sessions from PM. No access to existing connections, their secrets or their state. The owner is notified. |
| Provider portal account | long-lived | Provider-side | Same as issuing tokens, plus whatever the portal allows (deleting connections). Can't take over a live connection or change its secret. |
| Connection secret | long-lived, changed on demand and after tampering | platform-mesh-operator's namespace in PM's runtime cluster (the `ProviderEndpointRegistration`'s Secret); hash on the Provider-side | **PM-side compromise, control calls only.** The thief can push a connector CA bundle for that connection (to then open sessions answering with fake data), but the controller detects an unexpired CA that isn't one of PM's, restores the bundle and changes the secret through the `ProviderEndpoint`, which makes the stolen one useless. The thief can't change the secret, so can't lock PM out. Or the thief deletes the connection; the owner onboards the `Provider` again. **No kcp access**: the secret authenticates the PM-side to the Provider, not the other way round. |
| Connector certificate key | at most 24 hours (certificate lifetime) | Mounted in that shard's connector (a Secret issued by cert-manager), outside kcp | Opening sessions on that shard for every connection and answering their requests with fake data, until the certificate expires; sessions already open outlive it until PM drops its CA (see "Sessions opened with a leaked key"). No control calls. On its own, no kcp access; stolen from the connector pod itself, it comes with the connector's kcp identity. |
| Connector CA key | about a year, renewed automatically with a new key | A Secret in cert-manager's namespace, which only cert-manager reads | Issuing connector certificates: sessions for every connection on every shard, with fake data, until PM drops that CA (at once when it's distrusted) and the Provider-side closes the sessions opened under it. No control calls. On its own, no kcp access. platform-mesh-operator and connectors never see it; they only see public certificates. |
| SA token | minutes to hours | Connector memory only | Admin access to the provider workspace (RFC 006) and its exports' virtual workspaces until it expires, without the connector's denylist in the way. `ProviderEndpoint`s stay read-only (maximal permission policy). Only reachable by compromising a connector; never sent to the Provider-side. |
| Provider-side runtime | — | Provider-side | Whatever its open sessions allow: kcp requests within its admin RBAC in its own workspace, minus the connector policy, while the sessions last. Nothing persistent: there's no kcp credential to exfiltrate, and the policy refuses minting one. Ends when the PM-side closes the sessions. |
| Provider's edge (ingress, CDN) | — | Provider's infrastructure | Sees the spent onboarding token, the public registration response, connection IDs in paths, WebSocket upgrades without credentials, and opaque bytes. It can't act as the Provider in kcp, fake kcp's responses or use the connection secret. It can only refuse or cut connections. |
| Provider server key | until rotated, with overlap | Provider-side Secret by default, or an HSM or KMS named in the SDK's `ServerKey` (`signerRef`) | Together with control of the Provider's edge or DNS: posing as the Provider-side to PM's connectors and controller for that Provider's connections. Alone, nothing. |
| `ProviderEndpoint` | — | Provider workspace, PM-written only | Not a credential, but the only channel that changes a connection's secret. Writing it needs PM's operator identity, which the maximal permission policy enforces on every path; reading it reveals only hashes and public keys. |

### Threat model

- **The control plane** is the highest-impact target: it holds every connection secret and drives connector CA rollovers. It's one component, not on the traffic path, so it can be locked down and kept small. The connector CA's keys stay with cert-manager or an external PKI, outside its reach.
- **A connector** holds only its connector certificate (a day at most) and mints SA tokens. A compromised connector exposes that shard's sessions and traffic until its certificate expires, can't make control calls, and has no persistent secret to steal.
- **Sessions opened with a leaked key.** Whoever holds a connector CA key or a connector certificate key can open sessions and act as PM's connector: feed a Provider made-up consumer objects, so it creates, changes or deletes real resources, and receive whatever the Provider writes back, such as credentials it puts into consumers' workspaces. They get no kcp access from it. Sessions are only verified when they open, so after PM responds (renews the CA and drops the leaked one) such sessions would otherwise stay open until the Provider's edge cuts them, if it ever does. The Provider-side therefore re-checks its open sessions whenever its pinned bundle changes, and closes those that no longer chain to it. This matters for **keys that leak on their own**: a compromised external PKI, or a single Secret leaked through a backup, a log or a support bundle. Someone with access to PM's runtime cluster itself (cluster admin, a compromised node or cert-manager, a leaked etcd backup) also gets kcp's own credentials, which live there too, and can change consumers' objects in kcp directly; closing sessions doesn't help against them, and recovering from that is a much larger procedure.
- **The owner's workspace** holds no connection credential, only a spent onboarding token, whose data is deleted. Secret scanning catches dumps and commits of the prefixed connection secret wherever it leaks.
- **Someone holding a connection secret** can't lock PM out: the secret changes only through the PM-written `ProviderEndpoint`. What they can push (a connector CA bundle) is detected and undone (see "Distributing the connector CA to Providers").
- **The Provider's edge** is on the path of every session, unlike today's RFC 006 kubeconfig flow, but sees only opaque bytes: inner TLS keeps it from acting as the Provider in kcp, faking kcp's responses, reading PM consumers' data or using the connection secret (see "The Provider's edge: inner TLS on sessions").
- **A compromised Provider-side** has no kcp token to give away; its access ends when PM closes its sessions.
- **Owners** see only aggregate conditions; `ProviderEndpoint`s live in provider workspaces, and the connector CA and connector certificates in PM's runtime cluster, all outside owners' workspaces.
- **The Provider** is admin in its provider workspace and can see its own `ProviderEndpoint`, but kcp denies it any write to the `ProviderEndpoint` (the connection export's maximal permission policy, on any path), and the connector policy denies it changes to the connection export's binding, endpoint-slice status, and token or certificate minting. The latter holds as long as the connector is its only path to kcp (see "Connector policy"). The connection export's only claim is read-only access to the workspace's endpoint slices.

### Resources

```yaml
apiVersion: providers.platform-mesh.io/v1alpha1
kind: ProviderOnboardingRequest
metadata:
  name: my-provider-2026-10-09
spec:
  providerRef: { name: my-provider }                 # created by the owner; waited for if missing; refused if registered,
                                                     # unless the Provider-side reports its connection unknown
  tokenSecretRef: { name: my-provider-token, key: token }   # token data deleted by the controller once spent
status:
  phase: Registered                                  # Pending → Registering → Registered; Failed only before registration succeeds
                                                     # everything after registration is reported on the Provider
  connectionID: "..."
---
apiVersion: providers.platform-mesh.io/v1alpha1
kind: Provider
metadata:
  name: my-provider
spec:
  remote:
    caBundle: ""                                     # optional, for a private CA on the Provider's base URL: verifies the outer TLS
                                                     # of registration, control sessions and sessions; rotated out-of-band by the owner
---
apiVersion: providers.platform-mesh.io/v1alpha1
kind: ProviderEndpointRegistration                   # PM's runtime cluster, NOT kcp; authoritative
metadata:
  name: <clusterName>                                # named after the provider workspace; exists before registration returns
  namespace: <platform-mesh-operator namespace>      # written by the controller (and PM administrators for recovery); deleted at offboarding
spec:                                                # no status: nothing here can be recomputed
  connectionID: "..."                                # the ProviderEndpoint is written from these three
  url: https://provider.example.com/pm
  serverKeys: [ "<PEM SPKI>", ... ]
  rotationRequest: ""                                # last value of the rotation annotation acted on
---
apiVersion: v1
kind: Secret                                         # PM's runtime cluster, NOT kcp
metadata:
  name: <clusterName>-secret
  namespace: <platform-mesh-operator namespace>      # read and written by the controller only
  ownerReferences: [ { kind: ProviderEndpointRegistration, name: <clusterName>, ... } ]
type: Opaque
stringData:
  secret: "pmc1_..."
  pendingSecret: ""                                  # only while the secret changes
---
apiVersion: apis.kcp.io/v1alpha2
kind: APIExport                                      # in :root:providers:system
metadata:
  name: connections.platform-mesh.io
spec:
  resources:
    - name: providerendpoints                        # served by this export
      group: connections.platform-mesh.io
      # …schema reference
  maximalPermissionPolicy:
    local: {}                                        # RBAC below, in this workspace, caps every binding workspace
  permissionClaims:
    - group: apis.kcp.io
      resource: apiexportendpointslices
      verbs: [ get, list, watch ]                    # read-only; the export's only claim
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole                                    # in the export's workspace
metadata:
  name: connections-mpp-read
rules:
  - apiGroups: [ connections.platform-mesh.io ]
    resources: [ providerendpoints ]
    verbs: [ get, list, watch ]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: connections-mpp-read
roleRef: { apiGroup: rbac.authorization.k8s.io, kind: ClusterRole, name: connections-mpp-read }
subjects:
  - { apiGroup: rbac.authorization.k8s.io, kind: Group, name: "apis.kcp.io:binding:system:authenticated" }
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole                                    # in the export's workspace
metadata:
  name: connections-mpp-write
rules:
  - apiGroups: [ connections.platform-mesh.io ]
    resources: [ providerendpoints, providerendpoints/status ]  # no status today; covered if ever added
    verbs: [ create, update, patch, delete, deletecollection ]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: connections-mpp-write
roleRef: { apiGroup: rbac.authorization.k8s.io, kind: ClusterRole, name: connections-mpp-write }
subjects:
  - { apiGroup: rbac.authorization.k8s.io, kind: User, name: "apis.kcp.io:binding:<platform-mesh-operator user>" }
---
apiVersion: connections.platform-mesh.io/v1alpha1
kind: ProviderEndpoint                               # cluster-scoped, in the provider workspace :root:providers:<name>-<suffix>
metadata:
  name: default                                      # fixed name, one per provider workspace
spec:
  connectionID: "..."
  url: https://provider.example.com/pm               # validated against the egress rules (https only, in the schema too)
  serverKeys: [ "<PEM SPKI>", ... ]                  # the Provider-side's pinned server keys, for the inner TLS
  caBundle: ""                                       # the Provider's spec.remote.caBundle, for the outer TLS; web PKI if empty
  pendingSecretHash: ""                              # set while the connection secret changes; the Provider-side adopts it
```

- **`ProviderEndpoint`, in its own group.** pm-operator already has a `ProviderConnection` type for its own components' kubeconfigs, and `providers.platform-mesh.io` had an unserved `ProviderConnection`; that one is removed, and the `ProviderEndpoint` lives only in `connections.platform-mesh.io/v1alpha1`, served only by the connection export.
- **No `clusterName` field and no status**: the `ProviderEndpoint`'s logical cluster is the provider workspace, and connectors write nothing. The front-proxy URL is deployment configuration, and the SA connectors act as has a fixed name, `default/provider`, in every remote Provider's workspace.
- **RBAC**: platform-mesh-operator writes `ProviderEndpoint`s through the export's virtual workspace; connectors get read-only `apiexports/content`. The Provider's SA stays admin in its workspace, as in RFC 006; the export's maximal permission policy keeps it from writing `ProviderEndpoint`s, and the connector's denylist keeps it from changing the binding.
- **Who may create requests**: whoever manages `Provider`s in the owner's workspace; PM ships one ClusterRole that grants both together, and the controller doesn't check the creator. The oldest unfinished request per `Provider` wins; newer ones fail at once, naming the active one.

**How the request treats `providerRef`**:

| The `Provider`... | The request... |
|---|---|
| doesn't exist (yet) | waits; the owner creates it |
| exists, not registered | waits for the workspace and SA, then registers; this includes a `Provider` left behind by a failed onboarding |
| exists and is registered, the Provider-side knows the connection | **refuses**. A lost or rejected secret is repaired by the controller; there's nothing to onboard |
| exists and is registered, the Provider-side reports the connection unknown or deleted | **registers a new connection** (new `connectionID`) at the token's base URL, and rewrites the `ProviderEndpoint`. For switching Providers, or a connection the Provider won't restore; an import by the Provider keeps the `connectionID` and is preferred when it still has the data (see "Recovery") |
| exists and is registered, the Provider-side doesn't answer | **waits**, with a condition: only a definite "unknown" allows a new connection |
| exists, but isn't a remote `Provider` (e.g. a ManagedProvider's) | refuses |

- `Provider.status.connection` mirrors `url` and `connectionID` for the owner's information. The owner can edit it, so the controller never uses it as a link; it uses the `ProviderEndpointRegistration` in its own namespace instead. Nothing in it names shards.
- The `Provider`'s status has one writer, platform-mesh-operator: a condition per reconcile step (`Workspace`, `ConnectionBinding`, `ConnectorAccess`, `Registered`, `Connected`), the aggregate `Ready`, the warning `ProtocolOutdated`, and `phase` (`Pending → Provisioning → Ready ⇄ Degraded`, and `Deleting`, the same for local Providers). Failures carry a message that starts with a stable label and says what to fix. A CEL rule rejects `providerKubeconfigSecret` and `hostOverride` on remote Providers.
- The annotation `providers.platform-mesh.io/rotate-connection-secret` on the `Provider` triggers one secret change per new value (see "Changing the connection secret").
- **No static kubeconfig for remote `Provider`s.** Nothing on the PM-side is issued to the Provider-side except the sessions themselves.

### Restart and failure semantics

| Scenario | Behavior |
|---|---|
| Connector replica restarts, scales or rolls out | No registration: each replica connects with the mounted certificate. The other replicas' sessions keep serving every connection; only the streams on the affected replica's sessions are cut and reopened elsewhere. A new replica opens its sessions spread with jitter (see "Connector replicas"). |
| One shard's connector is down | Only that shard's endpoint traffic stops, for every connection; its watches relist when it's back. Front-proxy traffic uses other sessions of the connection, where there are any; for connections hosted on that shard with no consumers elsewhere, it waits. |
| One shard's virtual workspace is down | Connectors keep the last state of the `ProviderEndpoint`s and slices from that shard; only changes from it wait. |
| Shard added | Its connector is deployed with a connector certificate and watches the connection export. It dials a connection once kcp lists the shard in that connection's endpoint slices. Nobody registers anything. |
| Shard removed | Its connector goes away with it; the Provider-side drops the sessions; endpoint slices stop listing the shard. |
| First consumer binding on a shard | kcp adds the shard to the endpoint slice; that shard's connector sees it through the permission claim and dials. The Provider may briefly see the URL before the session is up; client-go retries. |
| Last binding on a shard removed | kcp drops the shard from the slice; that shard's connector closes the session after a grace period. |
| Provider tries to edit its `ProviderEndpoint` | kcp denies it (the connection export's maximal permission policy), and kcp's audit log records it. |
| Provider tries to drop the binding, forge endpoint-slice status or mint a token | The connector's authorizer denies it, and the attempt is audited. |
| Connection export's binding or `ProviderEndpoint` lost anyway (e.g. PM operator error) | Connectors drop that connection's sessions; the controller reports `Connected=False` and recreates them. |
| A `ProviderEndpoint` points at the wrong connection (PM-side bug) | The hello's `clusterName` doesn't match the one the Provider-side stored at registration, so it closes the session. |
| Connector certificate renews | cert-manager renews it halfway through its 24 hours, with a new key. Under the same CA the connector uses it at once; open sessions keep their handshake. Nothing reaches the Provider's control API. |
| cert-manager down past the 12-hour renewal budget | The connector certificate expires. Open sessions keep carrying traffic until they end (edge maximum age, a drop, a Provider or connector restart); the connector doesn't replace them and can't open new ones. Readiness reports it, so no connector rollouts until renewal resumes. |
| cert-manager renews the connector CA | The controller pushes the new CA to every connection it reaches within minutes; connectors switch once it has converged; 48 hours later the old CA is dropped connection by connection over 6 hours, and sessions still on its certificates hand over. No connection is left without a session. |
| CA renewal failing | The CA is renewed 90 days before it expires, so there's time; documented alert rules warn when the renewal is 30 days overdue. |
| Provider down during a CA rollover, however long | Nothing waits for it. Its first successful control call brings it the current trusted set, even if every CA it pins has expired, and sessions open within seconds. |
| Control path broken during a CA rollover | The connection isn't reached, so it keeps its old pins and loses each session at its next replacement after the connectors switch, until its control path works and the catch-up push restores them. The `Provider` shows why through the edge-diagnosis reasons. |
| Shard or CA key compromised | Stop issuing to that shard, or list the CA as distrusted, and renew the CA now. The old CA is dropped at once, and the Provider-side closes every session opened under it. |
| platform-mesh-operator down | Onboarding, secret changes, CA rollovers and offboarding wait. One exception: if it's down for more than about 12 hours right when cert-manager renews the CA (once a year), the new CA never reaches Providers, and connectors whose old certificates expire can't open new sessions, as during a cert-manager outage. Open sessions keep running; everything resumes once the controller pushes the new CA. Sessions on every shard keep working. |
| platform-mesh-operator's active replica lost | A standby takes the leader Lease within seconds, rebuilds its caches and reconciles every connection once, within the per-Provider rate limits. |
| Provider-side restarts | Sessions drop; connectors reconnect. The Provider-side has no credentials to restore. |
| Outage of any length | Nothing expires that the PM-side needs: the connection secret doesn't expire, connector certificates are renewed by cert-manager, and SA tokens are minted locally. |
| Ingress drops a session | The connector reconnects. Pings keep idle timeouts from triggering, and the connector replaces sessions before the edge's maximum connection age. On an abrupt cut, in-flight requests and watches on that session fail and client-go retries and relists, as after any apiserver disconnect. |
| PM rotates kcp's CA | Nothing to coordinate. Connectors use PM's own trust. |
| `POST` response lost, or the controller crashed before persisting it | Retry with the same token, idempotency key and `secretHash`: same `connectionID` and server keys, for 7 days. A different hash is refused. The result is persisted in one write; every later onboarding step resumes from it. A registration whose result is never stored leaves an orphan on the Provider-side, which expires because it never sees a control call or session. |
| kcp unreachable from a connector | The Provider-side reports `kcpReachable=false` for that host; the `Provider` shows `Connected=False` with the `KcpUnreachable` label and the last error, without a timeout, and becomes `Ready` once the problem is fixed. |
| Misconfigured Provider edge | The controller classifies its control-session failures (`EdgeUpgradeRefused`, `EdgeBotChallenge`, `EdgePayloadBlocked`, `EdgeTimeout`) and reports them on the `Provider` with a labelled message. |
| Controller crash or lost response during a secret change | Try `pendingSecret`, then `secret`; resume. The `ProviderEndpoint` still names the pending hash. |
| `ProviderEndpointRegistration` or its Secret lost (PM-side) | The controller rebuilds a lost `ProviderEndpointRegistration` from the `ProviderEndpoint` and repairs the secret through it. The connection and its state stay; nobody has to act. Sessions keep working throughout. |
| Provider-side rejects PM's secret (e.g. its store restored from an older backup) | The controller repairs the secret through the `ProviderEndpoint`, rate-limited per connection. |
| Secret change while no session is up | It waits; the old secret keeps working until a session lets the Provider-side read the `ProviderEndpoint`. |
| Someone with the connection secret pushes a connector CA bundle | The controller sees an unexpired CA that isn't one of PM's, restores the bundle (compare-and-swap), changes the secret through the `ProviderEndpoint` and records an event on the `Provider`. The thief can't change the secret. |
| Stolen onboarding token or portal account | Registers a separate connection that never receives sessions from PM; existing connections are untouched. The owner is notified. |
| Connection deleted on the Provider-side, or its store lost | Sessions and calls are refused. `Registered=False` with `ConnectionUnknown`. The Provider imports the old entry if it still has the data (same `connectionID`); otherwise the owner onboards the same `Provider` again: a new connection on the same workspace, consumers' bindings stay. |
| No session verifies (out-of-date bundle) and the secret is lost too | The Provider's administrators re-pin the bundle out-of-band; then the controller repairs the secret. |
| Provider-side runs multiple replicas | One is active by leader election; standbys answer upgrades with `503` and `Retry-After`, which connectors use as the lower bound of their backoff, so sessions land on the leader within a few attempts. After a leader failover, sessions are back within roughly 10–20 seconds and watches relist. |

### Offboarding and revocation

- **Deleting the `Provider`** makes the controller delete its `ProviderEndpoint` first, so every shard's connector closes its sessions. That ends the Provider-side's kcp access immediately, whether or not anything else succeeds. Then the controller calls `DELETE` in a control session, with bounded retries, deletes the `ProviderEndpointRegistration` and its Secret in its namespace, and removes its finalizer. platform-mesh-operator waits for that finalizer before deleting the connector RBAC, SA, binding and workspace (RFC 006), so connectors keep their token rights while sessions close.
- **Nothing to revoke after the SA is deleted.** Deleting an SA doesn't revoke bound tokens already issued for it, but remote `Provider`s never get any. (RFC 006's static kubeconfigs still have that problem.)
- **An unreachable Provider-side doesn't block deletion**: access is already gone, `DELETE` is only cleanup.
- **Deleting the connection on the Provider-side** refuses further sessions and calls; the PM-side surfaces it as `Registered=False` with `ConnectionUnknown`. Consumer data stays (see "Recovery"), and the owner can onboard the `Provider` again.

## Observations

Facts about kcp, kcp-operator and client-go the design relies on (2026-10-08 to 10-12):

1. **In tunnel mode, kcp trust equals connection trust.** The connector controls every request's destination, so a separately pinned or delivered kcp CA can't make the Provider-side's view of kcp more trustworthy than the session.
2. **Owner-supplied key material makes rotation manual.** A key or CA the owner supplies needs the owner for every rotation; one the controller generates rotates on its own.
3. **Serving many PM instances is simple on the Provider-side:** the SDK supplies a client config per connection, and the Provider's operator code creates a multicluster-runtime provider for each endpoint slice of its own exports, with cluster names prefixed by `connectionID`.
4. **The tunnel's limits are mostly those of any client-go connection.** One TCP connection for all requests, shared flow control, and mass relists when it dies are normal client-go behaviour against kube-apiserver. What a reverse tunnel really changes is that the client can't open connections on its own, and that the path runs through the Provider's L7 edge. A multiplexer restores the first; the second is handled by pings and graceful session replacement.
5. **Impersonating the connection's SA works for the provider workspace, not for the APIExport virtual workspace** (kcp `3523eb866` and its vendored Kubernetes fork, read on 2026-10-08; not tested).
    - **Provider workspace, through the front-proxy to the shard.** kcp's fork treats an SA user without the `authentication.kcp.io/cluster-name` extra as local to whichever workspace the request targets (`vendor/k8s.io/kubernetes/pkg/registry/rbac/validation/kcp.go`, `IsForeign`). The workspace authorizer then authorizes it as a local SA (`pkg/authorization/workspace_content_authorizer.go`). The shard's chain allows the impersonation: kcp's gatekeeper only blocks privileged groups (`pkg/server/filters/impersonation.go`), the upstream filter checks `impersonate` on `serviceaccounts` for that namespace and name and adds the SA groups itself (`vendor/k8s.io/apiserver/pkg/endpoints/filters/impersonation/impersonation.go`), and kcp's scoping filter confines the impersonated user to that workspace. RBAC: `access` on `/` plus `impersonate` on `serviceaccounts` with `resourceNames: [<sa>]`. Adding the cluster-name extra would make the identity identical to a real token, at the cost of `impersonate` on `userextras/authentication.kcp.io/cluster-name` with `resourceNames: [<cluster>]`. Constrained impersonation (beta, on by default in the fork) still accepts the plain `impersonate` verb.
    - **APIExport virtual workspace, standalone server.** It uses the upstream default handler chain, without kcp's gatekeeper or scoping, and the `impersonate` check goes to the APIExport virtual workspace's authorizer (`pkg/virtual/apiexport/builder/build.go`, `newAuthorizer`). For per-cluster requests, `boundAPIAuthorizer` only allows resources bound or claimed in the consumer's APIBinding, so impersonating `serviceaccounts` is denied (`pkg/virtual/apiexport/authorizer/binding.go`). For wildcard requests, the content authorizer checks `impersonate` on `apiexports/content` without looking at the impersonation target (`content.go`), and the maximal-permission authorizer allows unclaimed resources (`maximal_permission_policy.go`). Granting that verb would let the caller impersonate any user or group, including `system:masters`, which the virtual-workspace server always allows (`virtual-workspace-framework/pkg/options/authorization.go`).
6. **`ClusterCachedResource` replicates per workspace, but a shared identity makes one cache-wide set.** A `ClusterCachedResource` publishes objects of one GVR from the workspace it lives in into the cache server (`pkg/reconciler/cache/clustercachedresources`). Workspaces that use the same identity key land in one set that can be listed and watched across all workspaces and shards in one call, `<resource>:<identityHash>` (`test/e2e/cache/replication_api_cache_test.go`, `TestReplicationWithWildcardListing`). Reads also work, read-only, through the replication virtual workspace (`pkg/virtual/replication`), authorized by `apiexports/content` on an APIExport whose resource has virtual storage pointing at a `ClusterCachedResourceEndpointSlice`. It was considered for connection discovery and dropped in favour of the connection export (see "Rejected alternatives").
7. **kcp-operator gives `VirtualWorkspace` servers the cache-server credentials when an external cache server exists.** PM runs an external `CacheServer` that the root shard and shards reference (`pm-helm-charts/charts/infra/templates/kcp/cache-server.yaml`, `root-shard.yaml`, `shards.yaml`). kcp-operator then mounts the cache-server kubeconfig and client certificate into every `VirtualWorkspace` deployment and passes `--cache-kubeconfig` (`kcp-operator/internal/resources/compiledvirtualworkspace/deployment.go`, with a TODO to make it optional). It's presumably the same credential kcp's own components use, so it's far broader than a connector needs. PM's existing virtual-workspaces server, deployed as a plain Helm Deployment, has no cache access.
8. **An APIExport's maximal permission policy caps even an admin of the binding workspace** (kcp `ee9e6f3f3`, read on 2026-10-11; not tested for this export).
    - **Shard side.** `MaximalPermissionPolicyAuthorizer` runs in each shard's authorizer chain before bootstrap, local and global RBAC (`pkg/server/options/authorization.go`). For a resource listed in an APIBinding's `status.boundResources`, it checks RBAC in the export's workspace with the user name, groups and warrants prefixed `apis.kcp.io:binding:` (`pkg/authorization/maximal_permission_policy_authorizer.go`, `PrefixUser` in the vendored `pkg/registry/rbac/validation/kcp.go`). If the export has no local policy, or no longer exists, the check is skipped. `system:masters` bypasses the chain.
    - **Writes through the APIExport virtual workspace.** Per-cluster requests are forwarded impersonating the caller, plus a warrant for a local admin SA (`pkg/virtual/apiexport/builder/build.go`). The shard authorizes each effective user separately and accepts the first allow (`pkg/authorization/warrants.go`, `WithWarrantsAndScopes`), so granting `apis.kcp.io:binding:<controller user>` should be enough. Granting `apis.kcp.io:binding:system:kcp:admin`, as kcp does for its own root exports (`config/root/clusterrolebinding-kcp-admin-maximal-permission-policy.yaml`), would work too but is broader. Wildcard requests use the virtual workspace's own client.
    - **Evidence.** `test/e2e/apibinding/maximalpermissionpolicy_authorizer_test.go`, `TestMaximalPermissionPolicyAuthorizer`: the export grants `*` on `cowboys` only to `apis.kcp.io:binding:system:authenticated`; a consumer-workspace admin can create cowboys but is refused `UpdateStatus`. The same test shows that deleting the APIBinding deletes the bound resources. No e2e test covers writes through the APIExport virtual workspace to an export with a policy.
    - **Not covered.** The policy only applies to resources the export serves. Claimed resources and `apis.kcp.io` system resources (the binding itself, `apiexportendpointslices/status`) stay outside it, and kcp's system-CRD authorizer only protects `apibindings/status` and `apiexports/status` (`pkg/authorization/system_crd_authorizer.go`).

## Rejected alternatives

- **A pinned per-connection key as the PM-side's session proof.** Every connector would need every connection's key, and each rotation would cost a call per connection. Connector certificates from a pinned connector CA need neither (see "Session credentials").
- **Plain sessions, trusting the Provider's edge.** Without inner TLS the edge holds the Provider's end of every session: it could act as the Provider in kcp, fake kcp's responses, read PM consumers' data and use the connection secret. Many edges belong to third parties, and PM can't check the assumption for every Provider.
- **A session token on the upgrade**, e.g. a short-lived JWT signed with the connector certificate's key, carrying audience, expiry and a replay ID, checked before the WebSocket is accepted. The inner mutual TLS already proves possession of the certificate's key, freshness and the intended Provider, so the token adds a second proof plus clock-skew rules, a replay cache that Provider replicas would have to share, and a check that both proofs use the same certificate. All it saves is accepting a WebSocket before the caller is known, which control sessions do anyway and handshake limits keep cheap.
- **A preamble naming each stream's target** before its HTTP. The connector reads every request on a stream anyway and checks its `Host`, so the preamble would only state the target twice.
- **A separate operator for remote Providers' connections**, next to platform-mesh-operator. Two controllers would act on every remote `Provider`: two status writers, a hint condition passed between them, a finalizer handshake and naming functions shared as a contract. Keeping the connection code in platform-mesh-operator gives each `Provider` one owner; the price is internet-facing code in PM's most privileged component, while the `Provider` part is still part of it.
- **Partitioning the control plane by kcp shard**, with a Lease per shard and a worker per held shard. The load (a few calls per second for 1,000 connections, mostly waiting) doesn't need horizontal scale; leader election already gives one writer per object, and multicluster-runtime keeps a cache per shard endpoint, so a stuck shard only stalls its own objects.
- **Folding `ProviderOnboardingRequest` into `Provider`** (a token reference in `spec.remote`). It would remove one resource and the "oldest request wins" rule, but not the states, which would move into `Provider` conditions. Onboarding is a one-off event spending a single-use token; in a spec, the operator would have to rebuild that event (remembering the last token used) and notice new tokens in Secrets it doesn't watch.
- **The request creating the `Provider`**, before or after registration. Before, the request needs a copy of `spec.remote` and a rule for when it disagrees with an existing `Provider`. After, registration would also have to work without a `clusterName` (the workspace doesn't exist yet), and a spent token could leave a registered connection with no `Provider`. A `Provider` left behind by a failed onboarding is harmless: it waits, and the next request uses it.
- **Inlining the token in the request** instead of referencing a Secret. One object to copy, but an unspent token in a custom resource is exposed to more readers and more copies (`last-applied-configuration`, `kubectl get -o yaml`, audit bodies) than a Secret, and it can't come through GitOps or a secret store, which keep the request in Git and supply the token later.
- **A per-connection opt-out of inner TLS.** Two modes to test and reason about, for a saving that falls mostly on SDK code PM writes once.
- **Signed control calls** (a per-connection keypair signing a short-lived JWT per call, instead of the bearer secret). They add nothing here. Inside inner TLS the edge never sees the secret, and the Provider-side stores only a hash of a 256-bit secret, which is as useless to a thief as a public key. Whoever steals the PM-side's credential can use it either way, secret or private key. What limits such a thief is that the credential can't change itself (see "Changing the connection secret") and that bundle tampering is detected and undone.
- **Rebind with a waiting period**: the owner's token starts a 72-hour wait, cancelled by any authentication with the current secret. A thief with the secret could change it to lock PM out and then cancel every rebind. Changing secrets only through the PM-written `ProviderEndpoint` does the same job with no waiting period, no owner token, and no lock-out.
- **Re-onboarding into the same connection with an owner token.** The Provider-side would still need the `ProviderEndpoint` to tell PM's new secret from a token thief's, which makes the token redundant: the controller repairs a lost secret on its own.
- **The connection secret in the owner's workspace**. Any Provider with an accepted Secrets claim there could read it (PM's bindings accept that claim with `matchAll`), and an owner deleting it needed a recovery flow.
- **Refetching server keys over plain HTTPS** after a missed overlap. Trusted only through web PKI, so the edge could swap the keys and undo the inner TLS. The Provider-side switches a connection only once its `ProviderEndpoint` lists the new key, and lost keys go through a PM administrator.
- **remotedialer as the multiplexer.** Built for this direction and widely deployed, but per-stream backpressure (it may buffer instead) and a go-away for graceful replacement were unconfirmed; yamux has both.
- **A dial proxy with credentials on the Provider-side.** Forces credential delivery, refresh and kcp CA trust onto every Provider, and leaves a usable kcp token behind on every Provider-side compromise.
- **The connector impersonating with general impersonation rights.** Avoids minting per-connection tokens, but broad impersonation rights are wider than TokenRequest on specific SAs, and a header-stripping bug would become privilege escalation.
- **The connector impersonating the connection's SA, scoped by `resourceNames`.** Narrow and verified to work for requests to the provider workspace, but not for the APIExport virtual workspace, which carries most Provider traffic (observation 5). Tokens are needed there anyway.
- **A standalone connector proxy.** Would have to rebuild authentication, authorization, audit, timeouts, in-flight limits and metrics that the virtual workspace framework provides.
- **One connector for all shards.** One session per connection would carry every shard's traffic, so losing it relists everything; load wouldn't scale with shards; and the allow-list would need the whole shard list.
- **A "home connector" role inside the per-shard connectors.** Couples shards (a restart on one shard waited for the home shard), puts stateful control work and connection secrets into virtual workspaces, and spreads the control plane over every shard.
- **Per-session or per-replica bearer secrets, registered on each restart.** Every restart would depend on the control plane and on the Provider's control API. A shared bearer secret across connections would be worse: one Provider could replay it to another.
- **Per-shard keys registered on every connection, with self-registration** (`ConnectorShard` objects). Needs the connector to persist a key and write to kcp, exposes the shard layout, and costs a call per connection whenever a shard is added or rotates its key. The connector CA removes all of that.
- **Pinning the connector CA's bare public key** instead of its certificate. Every rollover changes the key anyway, so it saves nothing. It loses the built-in expiry of trust (a key pinned at an unreachable Provider would stay trusted forever), the CA constraints, and standard verification (Go's verifier needs a parent certificate, so the SDK would check signatures by hand or build a synthetic root).
- **One PM key shared by every shard.** Simplest, but one leaked key exposes every shard's sessions, and rotating it fans out to every connection.
- **Per-shard status in the owner's workspace** (`Provider.status.connection.shards`). Exposes PM's topology to owners and needs connector writes. Aggregate conditions from the controller instead.
- **`/services/admin` for the connector's allow-list and shard discovery.** Exposes the whole installation's shard layout to a component that only needs its own shard's host, which its deployment already knows.
- **Discovery from owners' `Provider` objects** (consuming the providers APIExport, or a `ClusterCachedResource` per owner workspace). Owners can edit those objects, including the URL connectors would dial.
- **`ProviderEndpoint`s in a PM-internal workspace**. Puts all `ProviderEndpoint`s on one shard, away from what they describe, needs a `clusterName` field and a separate lookup to find the hosting shard. The connection export carries both inherently.
- **`ProviderEndpoint`s replicated through the cache** (a `ClusterCachedResource` with a shared identity, observation 6). Needs a `ClusterCachedResource` and a shared identity key per provider workspace, and cache-server access for connectors, which in kcp-operator deployments means credentials far broader than they need (observation 7).
- **Annotations on the provider workspace's `Workspace` object in `:root:providers`.** No new API, but untyped, all on one shard, and finding the hosting shard needs a separate lookup.
- **Immutable, long-lived CA generations with a manual rollover.** Providers would pin a generation that never changes for 10 years, and people would roll over to the next one in gated steps: a CRD with lifecycle states, conditions to wait for, an override for connections whose control path is broken, a runbook and alerts. It avoids ever pushing a new CA automatically, but the push to every Provider is needed for a rollover anyway, and a 10-year key is exposed for 10 years.
- **platform-mesh-operator triggering CA renewals itself**, only while it's healthy, with cert-manager's schedule as a fallback. It would avoid new sessions failing when platform-mesh-operator is down for more than about 12 hours at a renewal, but needs rights on cert-manager's `Certificate`s in cert-manager's namespace. That's cert-manager's scope; a broken operator breaking new sessions after a renewal is an acceptable consequence.
- **A long-lived pinned root with an intermediate managed by cert-manager.** Routine key rotation would be automatic, but rolling over the root would still need all the machinery of a manual rollover. And for cert-manager to renew the intermediate, the root's key would have to sit in the cluster too, so it isn't safer than one CA that rolls over automatically.
- **Every shard's connector opening a session for every connection.** N connections × N shards sessions, mostly idle. Following kcp's own rule for endpoint slices opens only the ones that carry traffic.
- **Scoping the Provider's RBAC in its provider workspace**, so it can't write `ProviderEndpoint`s or the binding. PM doesn't know what a Provider needs to deploy, so the Provider stays admin (RFC 006); the connection export's maximal permission policy caps it for `ProviderEndpoint`s, and the connector policy denies the few other operations that matter to PM.
- **Signed `ProviderEndpoint`s**, so connectors can detect a `ProviderEndpoint` the Provider edited. Unnecessary once kcp denies the Provider writes to `ProviderEndpoint`s; it would add a signing key and verification for no extra protection.
- **Keeping the connection data in the `Provider` object**, with only an empty placeholder in the provider workspace. The owner can edit the `Provider` object (spec and status), so the owner could redirect the connection, and, without a PM-owned link between `Provider` and provider workspace, point it at another Provider's workspace. It also spreads the data back over owner shards and `:root:providers`.
- **Wildcard read of APIBindings on each connector's own shard** to find local consumers. Works, but needs shard-wide read access. The permission claim on endpoint slices keeps every connector privilege scoped to the connection export, and uses kcp's own answer.
- **A Provider-issued secret.** The registration response would carry a secret, usable by anyone who logged it.
- **A gateway container exposing a local proxy and kubeconfig files**. Doesn't help with multiple PM instances: the Provider still needs SDK code to consume a changing set of connections. It adds a hop and a directory format without adding a capability.
- **HTTP/2 directly on the WebSocket**, with the Provider-side as client and the connector as server. Works, and HTTP/2's own limits are no worse than any client-go connection's (observation 4). But the Provider-side then has a fixed set of connections it can't add to, so the SDK would have to replace client-go's pooling, stream-limit handling, health checks and upgrade connections with custom code, on an unusual reversed setup with little tooling.
- **HTTP/1.1 directly on the WebSocket.** One request at a time per connection; the first watch would block the session.

## Open items

Detailed decisions are in the [implementation plan](./xxx-notes-011-v5-impl-00-overview.md); this section summarises them, with what's still open, what the plan's tests must confirm, and deferred work.

**Decided** (see the linked documents):
- **Inner TLS on sessions**: required, the only mode; no per-connection opt-out ([00 Overview](./xxx-notes-011-v5-impl-00-overview.md)).
- **Multiplexer**: yamux, with per-stream flow control and go-away; the connector caps its upstream HTTP/2 receive buffer per stream ([01 Protocol](./xxx-notes-011-v5-impl-01-protocol.md), [05 Connector](./xxx-notes-011-v5-impl-05-connector.md)).
- **Protocol details**: sessions authenticated by the inner mutual TLS alone, with a hello; streams carry HTTP from the first byte; no intermediate certificates; control calls over HTTP/1.1, closed on the first failed authentication; outer pings every 30 seconds, binary messages up to 32 KB, current and previous protocol version supported ([01 Protocol](./xxx-notes-011-v5-impl-01-protocol.md)).
- **Connector CA**: managed by cert-manager, 1 year, renewed 90 days before expiry with a new key; self-signed by default, `issuerRef` optional; platform-mesh-operator reads new CA certificates from `CertificateRequest` status, pushes the trusted set, and drops the previous CA after 48 hours; only connections it reaches count for convergence; 24-hour connector certificates renewed halfway, switched once their CA has converged; per-replica certificates later, with an external issuer ([04 Connector CA](./xxx-notes-011-v5-impl-04-connector-ca.md)).
- **Control plane**: platform-mesh-operator, owning each `Provider` end to end; one active replica by leader election; `ProviderEndpointRegistration`s and their Secrets in its namespace, no Secret informers in kcp; internet-facing code in PM's most privileged component accepted while the `Provider` part is part of it ([03 Operator](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
- **`Provider` status**: one writer; one condition per reconcile step, with labelled messages; `phase` `Pending → Provisioning → Ready ⇄ Degraded`, and `Deleting`, for both kinds ([03 Operator](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
- **Connection secret changes**: only through the PM-written `ProviderEndpoint` (`pendingSecretHash`), adopted by the Provider-side over a session; no call changes a secret; on demand (an annotation), after tampering, and repair (automatic, rate-limited); no periodic rotation ([01 Protocol](./xxx-notes-011-v5-impl-01-protocol.md), [03 Operator](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
- **Connector CA bundle at Providers**: compare-and-swap with `etag` and `If-Match`; an unexpired pinned CA that isn't one of PM's unexpired CAs triggers restore, a secret change and an event on the `Provider` ([01 Protocol](./xxx-notes-011-v5-impl-01-protocol.md), [03 Operator](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
- **Recovery**: the `ProviderEndpointRegistration` is authoritative and the `ProviderEndpoint` is written from it; a lost `ProviderEndpointRegistration` is rebuilt from the `ProviderEndpoint` and its secret repaired; PM administrators supply recovery input by editing it; owners onboard a `Provider` again only after the Provider-side reports its connection unknown; disasters are handled out-of-band, with stable `connectionID`s, importing and re-pinning `Connection`s with `kubectl`, and PM administrator input keeping that possible ([01 Protocol](./xxx-notes-011-v5-impl-01-protocol.md), [07 SDK](./xxx-notes-011-v5-impl-07-provider-sdk.md)).
- **Connection export and `ProviderEndpoint`**: `connections.platform-mesh.io/v1alpha1` `ProviderEndpoint`, cluster-scoped with a fixed name, exported from `:root:providers:system`; platform-mesh-operator creates the binding with the workspace and recreates it if deleted ([02 APIs](./xxx-notes-011-v5-impl-02-apis.md), [03 Operator](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
- **Connector policy**: SA-token Secret writes are inspected and refused; child workspaces are denied; both kcp path shapes are parsed explicitly ([05 Connector](./xxx-notes-011-v5-impl-05-connector.md)).
- **Where Service endpoint slices live**: a documented requirement for Providers, in the provider workspace, for exports in that workspace ([07 SDK](./xxx-notes-011-v5-impl-07-provider-sdk.md)).
- **Connector deployment**: kcp-operator `VirtualWorkspace` per shard with `kubeconfigSecretRef` always set; cache-server credentials accepted and documented; every replica serves every connection its shard needs; `ServeProxy` copied ([05 Connector](./xxx-notes-011-v5-impl-05-connector.md), [09 Deployment](./xxx-notes-011-v5-impl-09-deployment.md)).
- **Provider-side replicas**: one active replica by leader election, standbys answer `503` with `Retry-After` ([07 SDK](./xxx-notes-011-v5-impl-07-provider-sdk.md)).
- **Requests**: owners create them, referencing a `Provider` the owner creates, with one ClusterRole granting requests and `Provider`s together; the oldest wins; a request ends at registration (`Pending → Registering → Registered`, or `Failed` before registration succeeds); the token stays in its own Secret ([02 APIs](./xxx-notes-011-v5-impl-02-apis.md)).
- **Outer TLS to base URLs**: web PKI, or the `Provider`'s `caBundle`, copied into the `ProviderEndpoint` for connectors; rotating a private CA is out-of-band, by the owner ([02 APIs](./xxx-notes-011-v5-impl-02-apis.md)).
- **Egress rules**: `https` only, public addresses only, resolve once and dial the checked addresses, no redirects, for the controller and the connectors alike ([03 Operator](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).

**Still to confirm** (by the plan's tests, [10 Testing](./xxx-notes-011-v5-impl-10-testing.md)):
- **Maximal permission policy** on writes through the APIExport virtual workspace with only the prefixed-user grant, and refusal for a provider-workspace admin (observation 8).
- **The permission claim** on `apiexportendpointslices` is readable through the connection export.
- **SA-token Secrets**: that kcp issues tokens for them, and that the inspection blocks every write form.
- **Sizing the second hop**: memory and goroutines per watch on the connector against the APIExport virtual workspace's, and cost per session (load tests).
- **Transport check**: `http.Transport` with a `DialTLSContext` returning a non-TLS stream speaks HTTP/1.1 to `https://` URLs, and client-go accepts it with multicluster-runtime's per-endpoint configs.
- **Proxied requests in the generic handler chain**: the long-running check and timeouts for proxied LISTs and watches, and the `X-Kcp-Virtual-Resource-Hops` budget.
- **multicluster-runtime composition**: adding and removing per-connection providers at runtime with prefixed cluster names.
- **Duplicate bindings**: whether kcp refuses a second binding to the same export in one workspace, and whether a duplicate matters.
- **Possible kcp issue: impersonation on the APIExport virtual workspace.** From reading the code (observation 5), anyone with `impersonate` (or `*`) on `apiexports/content` may be able to send wildcard requests to that export's virtual workspace while impersonating any group, including `system:masters`, which the server always allows. Its effect looks limited, because wildcard requests are forwarded with the virtual workspace's own client anyway. Confirm with an e2e test before reporting it to kcp.

**Deferred design:**
- **`direct` mode.** In v5, the Provider-side never talks to kcp directly. "Direct" would mean the Provider-side calls the connectors instead of the connectors dialling out, with the same routing, identity injection and authorizer. Each shard's connector would be exposed under its own external hostname, not through the front-proxy, which routes `/services/...` to a single backend (see "Per-shard connectors", point 8). What's missing is a credential in the other direction: one that the virtual workspace accepts for its connection only, is useless against kcp directly, and is revoked by deleting the connection. Defer until there's a use case.

Work that can be done separately (cleanups, larger follow-ups, items outside this plan) is listed in the [follow-ups](./xxx-notes-011-v5-impl-todo.md).

## Prior art

- **kcp front-proxy**: an authenticating reverse proxy for kcp, with streaming and watch support.
- **kcp's initializing-workspaces virtual workspace**: a `handler.VirtualWorkspace` that authorizes, strips credentials, impersonates and reverse-proxies into a workspace; the pattern for the connector's proxy half.
- **kcp's per-shard virtual workspaces** and **kcp-operator's `VirtualWorkspace` resource**: the deployment model for per-shard connectors.
- **kcp's admin virtual workspace** (`/services/admin`): the pattern for a read-only, cache-backed view, if a live connection-status resource is ever wanted.
- **Multi-shard APIExport consumers** (multicluster-runtime's endpoint-slice provider, as used by PM's own operators): how connectors read the connection export across shards.
- **cert-manager CA `Certificate`s and CA issuers**, optionally under Vault- or cloud-backed issuers: a CA that cert-manager renews with a new key, issuing short-lived workload certificates.
- **Teleport Kubernetes service**: a proxy that attaches identity to Kubernetes requests, so clients never hold cluster credentials.
- **Cloudflare Tunnel** and similar: an outbound connection from the private side, with HTTP served back over it.
- **Rancher `remotedialer`**: multiplexed streams over a WebSocket, dialled from the side that accepted it; the other multiplexer candidate (see "Rejected alternatives").
- **Konnectivity**: a dial proxy, with the client holding kcp credentials.
- **client-go and kube-apiserver connection handling**: one HTTP/2 connection per host, health checks, and server-sent GOAWAY (`--goaway-chance`) for rebalancing; the baseline v5's tunnel is measured against.
- **Open Cluster Management cluster-proxy**: reverse tunnel to managed clusters.
- **kubeadm bootstrap tokens**: `tokenID.tokenSecret`, stored hashed, short-lived.
- **Prefixed secrets for scanning**: GitHub's token prefixes and secret-scanning partner programme.
- **HTTP conditional requests** (RFC 9110: `ETag`, `If-Match`, `412 Precondition Failed`): compare-and-swap for the connector CA bundle.
