# Notes v4-b: RFC 011 provider onboarding flow

Working notes for [011](./011-provider-onboarding-flow-improvements.md). Not the RFC itself. v4-b is a variant of the [v4 notes](./xxx-notes-011-provider-onboarding-flow-improvements-v4.md) that applies what the [v5 notes](./xxx-notes-011-provider-onboarding-flow-improvements-v5.md) turned up, while keeping v4's core: **the Provider-side holds kcp credentials and reaches kcp through a dial-proxy tunnel, with end-to-end TLS to kcp** verified against a CA bundle the PM-side delivers. It splits the PM-side into a **control plane** (one stateful connection controller that holds every connection key and makes every control call) and a **data plane** (stateless connectors, one per kcp shard, that only carry tunnels). Control calls run in **mutual TLS inside an ordinary HTTPS WebSocket**, authenticated by a pinned **per-connection key** on the PM-side and a pinned **server key** on the Provider-side. Tunnels run on a plain WebSocket, authenticated by short-lived **session assertions** signed with certificates from a pinned **connector CA**. The trust boundary is v4's: **the owner's workspace is the owner's responsibility**, and nothing that works against kcp crosses the Provider's edge in readable form.

The [comparison notes](./xxx-notes-011-provider-onboarding-flow-improvements-v4-v5-comparison.md) put v4, v4-b and v5 side by side.

## Terminology

- **PM-side**: the PM instance (kcp + PM operator). May be behind NAT; dials out.
- **Provider-side**: the service provider (controllers + portal in its runtime cluster). Serves the connection API and reconciles on the PM-side.
- **Owner**: whoever controls the workspace that holds the `Provider` object and its connection Secret on the PM-side.
- **Connection**: the relationship between one `Provider` object on the PM-side and one connection entry on the Provider-side, identified by `connectionID`. Created by registration, ended by deletion on either side. **The Provider-side's state for a connection belongs to that connection and is never inherited by another one.**
- **Onboarding token**: a single-use, short-lived token from the Provider's portal. It contains the Provider's base URL.
- **Connection controller**: the PM-side **control plane**. One stateful component (HA, leader election) that registers connections, holds their connection keys, makes every control call, mints and delivers access tokens, writes each connection's record into its provider workspace, and reports status on the `Provider`. Never on the tunnel path.
- **Connector**: the PM-side **data plane**. A stateless process, **one per kcp shard** (several replicas), that opens tunnel sessions and forwards the streams dialled through them to kcp. An L4 byte pipe: it never sees kcp requests or credentials.
- **Connection key**: a keypair the connection controller generates for one connection and keeps in the connection Secret in the owner's workspace. The Provider-side pins its public key at registration as the connection's identity. Read only by the controller, which rotates it automatically. Authenticates the **control channel** only.
- **Connection certificate**: a short-lived, self-signed client certificate for the connection key, created by the controller whenever it needs one. Only the wrapper TLS needs to carry the key; the Provider-side checks the key, not a chain.
- **Server key**: the keypair the Provider-side's connection server uses in the inner TLS handshake of control sessions. The PM-side pins its public key at registration.
- **Control session**: a WebSocket from the controller to the Provider's base URL, carrying one **inner mTLS** connection (pinned connection key ⇄ pinned server key) with the connection API inside it.
- **Connector CA**: a PM connections CA that only issues connector certificates, run by the control plane (cert-manager, ideally with a KMS- or Vault-backed issuer). Every connection pins its **certificate** at registration. Each connector CA is an **immutable generation**: never renewed, and replaced by a new CA well before it expires.
- **Connector certificate**: a short-lived certificate (e.g. 24h) from the connector CA for one shard's connector, renewed automatically and mounted read-only. It carries no hostnames: the hosts a tunnel serves travel in its session assertion.
- **Session assertion**: a short-lived JWT signed with the connector certificate's key, carrying the connector certificate in `x5c`, bound to one connection. It authenticates a tunnel session upgrade and nothing else.
- **Tunnel session**: a WebSocket from one shard's connector to the Provider's base URL, authenticated with a session assertion, carrying the dial-proxy multiplexer. At most one per (connection, shard), opened only where the connection has traffic. **No inner TLS.**
- **Access token**: the bound SA token the Provider-side uses against kcp.
- **Connection export**: a PM APIExport serving `ProviderConnection` records, with a local **maximal permission policy** that allows only reads in the workspaces that bind it, and a read-only permission claim on `apiexportendpointslices`. The Provider controller binds it in every provider workspace when it creates the workspace.
- **Connection record**: a non-secret `ProviderConnection` object (`connectionID`, base URL) in the connection's **provider workspace**, written by the controller through the connection export. Connectors read it through the export's endpoint slice, so the record lives on the shard of the workspace it describes.
- **`ProviderOnboardingRequest`**: the owner's request to connect a `Provider` with an onboarding token. Also starts a rebind (see "Recovery: rebind").

## Problem

The Provider-side must reconcile on any number of PM instances; the PM-side may be behind NAT and opens every connection; the user gets a token from the Provider's portal; and both sides must recover from restarts and outages without manual steps.

How the PM-side proves "I own this connection" after registration has changed with each version:
- **v2**: a long-lived bearer secret, sent on every call, visible to every TLS-terminating hop.
- **v3**: short-lived kcp SA tokens, verified against kcp's signing keys. Tied every connection to kcp's key management, which rotates without overlap (finding 4).
- **v4**: a per-connection key, proven in a TLS handshake that runs end to end between the connector and the Provider's connection server, through whatever ingress the Provider has. But one connector reads every connection's key, which made it the highest-impact target, and where it runs stayed open.
- **v4-b**: the same key, held only by a control plane that isn't on the traffic path. Tunnels, which carry only end-to-end TLS to kcp, are authenticated separately with short-lived assertions from per-shard connectors.

**Constraints.**
- **Assume as little as possible about the Provider's infrastructure.** Many Providers sit behind cloud L7 load balancers, CDNs, WAFs, or a central platform team's ingress. TLS passthrough or client-certificate-verifying ingresses can't be required.
- **Assume nothing about how PM operates kcp's PKI.** PM must stay free to rotate kcp's CAs, add shards and change its edge. The design must not depend on kcp's rotation procedures or on operator-specific artefacts like merged CA bundles.
- **Owners learn nothing about PM's topology.** Nothing in an owner's workspace names shards or per-shard state.
- **Connectors stay stateless.** A connector holds no persistent state and writes nothing to kcp; everything it knows about its place comes from its deployment.

## Trust boundary

**The owner's workspace is the owner's responsibility.** It holds the `Provider` object, the connection Secret (with the connection key) and, under RFC 006, more credentials that reach the provider workspace. Whoever gets into it can act as the owner. v4-b doesn't try to defend a connection against its own owner's compromise; it makes sure that:
- **nothing outside the owner's workspace** can take over an existing connection: not a stolen onboarding token, not a compromised Provider portal account, not the Provider's ingress or CDN;
- **leaks are bounded** in time where possible (short-lived certificates, assertions and access tokens) and in scope (a leaked connection key affects exactly one connection; a leaked connector certificate yields no credentials and no readable traffic);
- **Provider-side state can't be hijacked** by registering a new connection for the same workspace;
- **the Provider can't rewrite PM's records** in its own workspace, even though it's admin there.

Consequence: an attacker with the connection key can do whatever the owner's connection can, including presenting a fake kcp to the Provider-side (see "kcp's CA bundle"). That's owner compromise, in the same category as leaking the workspace's other credentials.

**The Provider's edge is not trusted with anything that works against kcp.** Access tokens and kcp's CA bundle travel only inside the inner TLS of a control session. Tunnel sessions have no inner TLS, so the edge can:
- **see** which kcp hosts the Provider-side dials, and when; the bytes themselves are the Provider-side's own TLS to kcp;
- **drop or delay** traffic, as it always could;
- **inject its own dials** through the multiplexer, reaching the allow-listed kcp hosts (a shard's virtual-workspace host and the front-proxy) **without credentials**. kcp treats such requests as anonymous. Where kcp's hosts are already reachable from the internet, this adds nothing; where they aren't, it's new network exposure (see "Open items").

It can't read access tokens, impersonate kcp to the Provider-side, or make control calls.

## Core idea

```
control plane
connection controller ══ outer TLS (web PKI, terminated at ingress/CDN) ══▶ ingress ──▶ connection server
                      ╚═ WebSocket: GET /v1/connections/{id}/control ═══════════════════════╝
                           ╚═ inner TLS 1.3: pinned connection key ⇄ pinned server key ═════╝
                                ╚═ connection API: credentials, key, connector CAs, status, delete

data plane (one per shard, only where the connection has traffic)
connector@shard ══ outer TLS ══▶ ingress ──▶ connection server
                ╚═ WebSocket: GET /v1/connections/{id}/tunnel, Bearer <session assertion> ═╝
                     ╚═ dial proxy (remotedialer): the Provider-side dials through the connector
                          ╚═ Provider-side's own TLS to kcp (delivered CA bundle, access token) ═▶ kcp
```

- **Two kinds of session, two planes.** Control sessions carry everything secret and are authenticated by the pinned connection key and the pinned server key in inner mTLS. Tunnel sessions carry only opaque TLS to kcp, so they need authentication but not confidentiality.
- **The connection's identity is a pinned public key**, held by the controller only. The controller proves possession in every control handshake with a self-signed certificate it creates itself, and rotates the key on its own. No owner step is ever needed.
- **Tunnels need no per-connection secret.** Each shard's connector signs assertions with its mounted connector certificate; the Provider-side verifies them against the connection's pinned connector CA certificates.
- **A forged tunnel can't fake kcp.** The Provider-side verifies kcp end to end against the CA bundle it received over the control channel, and sends its access token only inside that TLS. A tunnel holder can drop traffic and see dial targets, nothing more.
- **The connector is an L4 hop.** It never sees kcp requests or credentials.
- **The Provider-side's identity is a pinned server key**, returned at registration over web PKI, checked on control sessions.
- **kcp's CA travels with the credentials.** The controller sends the CA bundle PM's own components use for kcp, so it follows PM's rotations automatically.
- **No re-onboarding.** A connection can only be changed through a control session authenticated by its connection key, or by a rebind that the current key holder can cancel. Registration only creates new connections.
- **Revocation is a database delete.** The Provider-side checks the connection entry on every handshake, session and request.

## Provider-side prerequisites

- **An HTTPS endpoint at the base URL from the token that supports WebSocket upgrades** and long-lived connections: idle timeouts above the session ping interval (e.g. > 60 s), and a maximum connection age the connectors can reconnect after. Any ingress, cloud L7 load balancer or WebSocket-capable CDN qualifies. A WAF must let binary WebSocket frames through after the upgrade.
- **A connection server** that serves the connection API on control sessions (inner mTLS behind a WebSocket listener), verifies session assertions on tunnel sessions, and runs the dial-proxy server. The Provider SDK ships it.
- **A server key** for the inner TLS, kept in a Secret on the Provider-side.
- **Per-connection pins**: the connection key and the connector CA certificates, stored at registration and changed only over a control session or by rebind.
- **State keyed by `connectionID`**, never by the PM-side's addressing alone (see "Provider-side state").
- **Owner notifications and controls.** The portal notifies the token owner of every registration, connection key change and rebind (in the portal and by email), and lets the owner freeze or delete their connections.

Not required: a second hostname, TLS passthrough, client-certificate verification at the ingress, cert-manager, or a CA of its own.

## Flow overview

1. **Get a token.** The user gets an onboarding token from the Provider's portal.
2. **Create the request.** The owner stores the token in a Secret and creates a `ProviderOnboardingRequest` that references it.
3. **Create or reuse the `Provider`.** The request creates the `Provider` if it's missing, reuses an unregistered one, refuses a registered one, or starts a rebind for a registered one whose connection Secret is gone (see "Resources").
4. **Register.** The controller waits for the provider workspace and SA, generates the connection key, writes it to the connection Secret, then calls `POST /v1/connections` with the onboarding token, the addressing, the connection key's public key and the connector CA certificates. It gets back the `connectionID` and the connection server's public keys, and stores them in the connection Secret. No kcp credential is sent.
5. **Write the connection record.** The controller validates the base URL (see "Egress") and writes the `ProviderConnection` record into the provider workspace through the connection export, which the Provider controller bound there when it created the workspace.
6. **Deliver credentials.** The controller opens a control session and `PUT`s an access token and kcp's CA bundle.
7. **Open tunnels where they're needed.** The connector on the shard hosting the provider workspace opens a tunnel session right away, for front-proxy traffic. Connectors on shards listed in the connection's Service endpoint slices, i.e. shards with at least one consumer binding, open theirs as those shards appear.
8. **Confirm kcp is reachable.** The controller polls `GET /v1/connections/{id}` until the Provider-side reports that every host it needs is covered by a tunnel and reaches kcp. Only then is the request `Ready`. From here on, the Provider-side has **claimed** the provider workspace.
9. **Refresh credentials.** The controller `PUT`s a fresh access token and the current kcp CA bundle at about half the token's TTL. After the claim, the Provider-side also mints its own (see "Credentials and rotation").
10. **Rotate** the connection key, automatically and periodically (see "Connection key rotation"). Connector certificates renew on their own; connector CA generations roll over rarely.
11. **Offboard.** Deleting the `Provider` deletes the record first, so every connector closes its tunnels; then the controller calls `DELETE` in a control session.

Steps 6 to 10 handle every restart and outage, of any length: the connection key doesn't expire, the controller can always create a new certificate for it, and connector certificates are renewed by cert-manager.

## Design

### Onboarding token

`pm1.<base64url(Provider base URL)>.<tokenID>.<tokenSecret>`: single-use, short-lived, stored in a Secret, with the URL inside so nothing is ever sent to the wrong host. Trust in the base URL comes from web PKI, or from the `Provider`'s `spec.remote.caBundle`. The Provider-side stores only a hash of `tokenSecret`.

- **The portal never puts tokens in URLs**, which end up in browser history, proxy logs and referrer headers.
- **The portal hands the token over as a command, not a manifest.** Offer `kubectl create secret generic … --from-literal=token=…`, or apply it directly. A "copy as Secret manifest" button invites committing the token to git.

### Control plane and data plane

```
                         PM control plane                                    PM data plane
 ┌──────────────────────────────────────────────────────┐      ┌───────────────────────────────────────────┐
 │ connection controller (HA, leader election)          │      │ connector@shard-A (StatefulSet, stateless) │
 │  · ProviderOnboardingRequest / Provider reconcile    │      │  mounted: shard cert + key, own VW host,   │
 │  · connection Secrets: connection keys, server pins  │      │           front-proxy host                 │
 │  · control sessions: credentials, key, CAs, delete   │      │  reads:   connection export (records,      │
 │  · TokenRequests for each connection's SA            │ ───▶ │           claimed endpoint slices)         │
 │  · ProviderConnection records (through the export)   │      │  writes:  nothing                          │
 │  · status on Provider                                │      │  dials tunnels, forwards streams to kcp    │
 │ connector CA (cert-manager, KMS/Vault-backed issuer)   │ ───▶ │ connector@shard-B …  connector@shard-C …   │
 └──────────────────────────────────────────────────────┘      └───────────────────────────────────────────┘
```

| | Connection controller | Connector@shard |
|---|---|---|
| Holds | Every connection key; the connector CA's issuer | Its connector certificate (hours), mounted |
| Reads | Connection Secrets (owners' workspaces) | Connection export: records and claimed endpoint slices |
| Writes | Connection records (through the export); TokenRequests | Nothing |
| Talks to the Provider | Registration, control sessions, rebind | Tunnel sessions |
| Talks to kcp | Front-proxy, as itself | Forwards opaque TCP streams to its shard's virtual-workspace host and the front-proxy |
| Outage | Onboarding, rotations, rebinds and offboarding wait; **token delivery stops** (see "Credentials and rotation") | That shard's tunnels stop |

- **Everything secret sits in the control plane**, which is one component, not on the tunnel path, and can be kept small and locked down.
- **Everything on the traffic path sits in the data plane**, which holds nothing long-lived. A connector replica can be killed at any time; a new one reads its mounted certificate and the connection export and carries on.

### Registration

The only call that changes connections outside a control session, apart from rebind. Plain HTTPS on the base URL, so the outer TLS is all that protects it. It carries nothing secret except the onboarding token, which is spent by the time anyone could reuse it.

```
POST /v1/connections
Authorization: Bearer <tokenID>.<tokenSecret>
{
  "idempotencyKey": "<ProviderOnboardingRequest UID>",
  "clusterID": "<logical cluster name of the provider workspace>",
  "frontProxyURL": "https://...",
  "transport": "tunnel" | "direct",
  "connectionKey": "<PEM SPKI of the connection key>",
  "sessionCAs": [ "<PEM certificate of the connector CA>" ]
}
→ 201 {
  "connectionID": "...",
  "serverKeys": [ "<PEM SPKI>", ... ]            // connection server's public keys; the PM-side pins them
}
```

- **One-step registration.** The controller waits until the workspace and SA exist, then registers with real addressing.
- **Only new connections.** Registration never touches an existing connection (see "No re-onboarding").
- **No access token in `POST`.** Behind a TLS-terminating ingress or CDN, that would be a live kcp credential in plaintext at the edge (finding 10). Every access token goes inside the inner TLS of a control session.
- **`serverKeys` is trustworthy** because the response arrives over the outer TLS to the URL inside the token, authenticated by web PKI (or `spec.remote.caBundle`). It's a list, so the Provider-side can rotate its server key with overlap.
- **Idempotency is bound to the connection key.** `POST` is keyed on `(tokenID, idempotencyKey)`. A retry returns the same `connectionID` **only if `connectionKey` matches the first attempt**; a different key is refused. If the response is lost and the token has expired, a retry within a bounded window still returns the entry. The response contains nothing secret, so a replay of a logged request gains nothing (finding 9).
- **Addressing is immutable**: `clusterID` and `frontProxyURL` are fixed at registration. Changing them means a new connection.
- **Only public material leaves the PM-side**: the connection key's public key and the connector CA certificates.
- **Nothing about APIExports is negotiated.** The Provider owns its exports and knows its endpoint slices (see "Provider-side: serving many PM instances").

### Choosing the anchors

v4-b has four separate trust questions:

| Who verifies | Whom | Kind of trust | v4-b's answer |
|---|---|---|---|
| Provider-side | the PM-side, on control calls | client authentication | **the pinned connection key** |
| Provider-side | the PM-side, on tunnels | client authentication | **a session assertion chaining to the pinned connector CA** |
| PM-side | the Provider-side, on control calls | server authentication | web PKI at registration, then the pinned server key |
| Provider-side | kcp, through a tunnel | server authentication | kcp's CA bundle, delivered over the control channel (see "kcp's CA bundle") |

The PM-side identity must rest on client-authentication trust only. kcp's serving CA answers the last question ("is this the real kcp?"), never the first two. Using it for client authentication would let every front-proxy, shard and virtual-workspace server, which all hold serving keys from it, pose as any connector or controller.

**Control calls: a pinned key per connection.** Candidates, from smallest to largest blast radius:

| Option | Who can mint an identity | If its key leaks | What the certificate also grants |
|---|---|---|---|
| **Pinned key per connection** (chosen) | only that connection's key holder | that one connection | nothing |
| Generated CA per connection | only that connection's CA holder | that one connection | nothing |
| CA per owner | the owner's CA holder | every connection of that owner | nothing |
| Dedicated PM connections CA, alone | whoever runs it | every connection of the PM instance | nothing in kcp |
| kcp's client CA (kcp-operator `Kubeconfig`) | anyone who can create `Kubeconfig` resources | every connection, and all of kcp | a live kcp credential (finding 13) |
| kcp's serving CA | every kcp serving process | — | not an option: wrong trust question and wrong certificate type |

A per-connection key has the smallest blast radius, needs no CA anywhere, and rotates without anyone else's involvement. Control calls carry access tokens and the kcp CA bundle, so they get the strongest anchor.

**Tunnels: a connector CA.** A shared CA would be the wrong anchor for the connection's identity, which is why the control path doesn't use one. Tunnels are different: a tunnel session can't make control calls, can't obtain kcp credentials, and can't read or alter kcp traffic, which is end-to-end TLS verified against a CA bundle from the control channel. A leaked connector certificate or CA key therefore allows denial of service and traffic analysis for the connections it covers, until the certificate expires or the generation is unpinned. In exchange, connectors need no per-connection secret, and adding shards or replicas involves no one (see "The connector CA").

### Connection key and certificates

- **Generated per connection** by the controller (e.g. ECDSA P-256 or Ed25519) and stored in the connection Secret in the owner's workspace. Never supplied by the owner, so there's no owner-held copy to coordinate with.
- **The controller creates connection certificates for it**: self-signed, short-lived (e.g. 24h), EKU `clientAuth` only, no DNS or IP SANs. Kept in memory. A new one can be created at any time, so a certificate's expiry never needs the Provider-side.
- **The Provider-side checks the key, not a chain**: the certificate's public key must be the connection's pinned key (or its pending key during a rotation), and the certificate must be within its validity and carry EKU `clientAuth`.
- **Only the controller reads it.** Connectors never see connection keys.
- **Backups are optional.** A lost Secret is recovered by rebind (see "Recovery: rebind"). Restoring a backup taken after the last rotation is faster, and keeps the connection without a waiting period.

### Connection key rotation

Fully automatic, by the controller, which holds both the current and the new key throughout:

1. The controller generates a new key and writes it to the connection Secret as `pendingKey` **before** sending anything, so a crash can't lose it.
2. It opens a control session with a certificate for the **current** key and calls `PUT /key` with the new public key. The Provider-side stores it as **pending**; certificates for either key are accepted.
3. It opens a control session with a certificate for the **new** key. That successful handshake proves the PM-side holds the new private key, so the Provider-side **promotes** it and drops the old one.
4. The Provider-side closes control sessions still authenticated with the old key. **Tunnels are unaffected**: they don't use the key.
5. The controller moves `pendingKey` to `key` in the connection Secret.

If the `PUT /key` response is lost, the old key still works and the controller retries. If the new key is never used, the pending entry expires (e.g. after 24h). If the controller crashes partway, it compares the pinned and pending fingerprints (from `GET`) with `key` and `pendingKey` and resumes at step 2 or 3.

**When it happens.** Periodically (e.g. every 90 days), on demand (an annotation on the `Provider`), and right after a suspected leak.

**Lock-out.** Whoever holds the current key can rotate to a key of their own, after which the legitimate controller can't authenticate any more, and can cancel the owner's rebinds. Under the trust boundary that's owner compromise. Mitigations: the Provider-side **notifies the owner of every key change**, and the owner can **freeze or delete a connection in the Provider's portal**.

### Control sessions

```
GET /v1/connections/{id}/control        (WebSocket upgrade, on the base URL)
  └─ inner TLS 1.3 handshake
       client: self-signed certificate for the connection key
       server: self-signed certificate for its server key
       ALPN:   "h2" or "http/1.1"
  └─ HTTP inside the inner TLS
```

- **Provider-side.** The upgrade handler checks that connection `{id}` exists, wraps the WebSocket as a `net.Conn`, tags it with `{id}`, and hands it to a listener. On that listener runs an HTTPS server with `ClientAuth: RequireAnyClientCert` and a verification callback that checks connection `{id}` still exists, and that the client certificate's public key is the connection's pinned key (or pending key), within its validity and with EKU `clientAuth`. Every inner request must name the same `{id}` in its path.
- **PM-side.** The controller's HTTP client has a custom dialer: open the WebSocket, wrap it as a `net.Conn`, run the TLS client handshake over it. The verification callback checks that the server certificate's public key is one of the pinned `serverKeys`.
- **Both sides pin keys, symmetrically.** In Go both callbacks mean `InsecureSkipVerify: true` (client) or `RequireAnyClientCert` (server) plus a verification callback, a well-known footgun if written ad hoc. Both belong in one small, tested helper in the Provider SDK.
- **Control sessions are opened on demand.** The controller opens one when it has calls to make; HTTP/2 inside lets one session carry many calls.
- **Path routing.** The outer path contains the `connectionID`, which isn't secret. Ingresses can route, rate-limit or log per connection without seeing anything inside.

### Connection API (inside a control session)

```
GET /v1/connections/{id}
→ 200 { "requiredHosts": { "<host:port>": { "tunnels": 1, "kcpReachable": true, "lastError": "" } },
        "serverKeys": [ ... ],
        "connectionKey": "<fingerprint of the pinned key>", "pendingConnectionKey": "<fingerprint, or empty>",
        "pendingRebind": null }

PUT /v1/connections/{id}/credentials
{
  "token": "<fresh access token>",
  "expirationTimestamp": "...",
  "kcpCABundle": "<PEM>"                      // the CAs PM's own components use for kcp; empty = web PKI
}
→ 204

PUT /v1/connections/{id}/key                  # connection key rotation
{ "connectionKey": "<PEM SPKI of the NEW connection key>" }
→ 202                                         # pending until a handshake with a certificate for it

PUT /v1/connections/{id}/connector-ca-bundle          # connector CA rollover (rare)
{ "sessionCAs": [ "<PEM certificate>", ... ] } # current generation, plus the next one during a rollover
→ 204

DELETE /v1/connections/{id}
→ 204
```

Outside control sessions, on plain HTTPS: `POST /v1/connections` (registration), `POST /v1/connections/{id}/rebind` (see "Recovery: rebind"), and `GET /v1/connections/{id}/server-keys` (web PKI), which a controller that missed a whole server-key overlap uses to re-read `serverKeys`.

- **Readiness per host.** `requiredHosts` lists the front-proxy and the hosts from the endpoint slices the Provider-side actually uses, whether a tunnel covers each, and whether it reaches kcp. The request becomes `Ready` only when every listed host does, so a wrong CA bundle or a missing tunnel fails registration visibly. Nobody needs the shard layout.
- **No kcp-specific verification on the Provider-side.** It doesn't need to understand kcp's tokens or keys to authenticate the PM-side. It uses the access token only to talk to kcp.

### Tunnel sessions

```
GET /v1/connections/{id}/tunnel         (WebSocket upgrade, on the base URL; dial-proxy multiplexer inside)
Authorization: Bearer <session assertion>
  header: { "alg": "ES256", "x5c": [ "<connector certificate>", "<intermediate, if any>" ] }
  claims: { "aud": "<base URL>/v1/connections/{id}/tunnel",
            "cluster": "<clusterID of the provider workspace>",
            "hosts": [ "<virtual-workspace host(s) this tunnel serves>" ],
            "sub": "<opaque session-group ID>",            # e.g. a hash of shard and connectionID; audit only
            "iat": …, "exp": iat + ≤ 5m, "jti": "<random>" }
```

**Verifying a session assertion** (in the Provider SDK):
1. The chain in `x5c` verifies with the standard verifier (`x509.Verify`) against **only** the connection's pinned connector CA certificates as roots, with at most one intermediate. That checks every certificate's validity and the CA constraints. The connector certificate must be allowed for signing (key usage `digitalSignature`, plus whatever EKU the connector CA uses to mark connector certificates). Two details the SDK must get right: Go's verifier requires EKU `serverAuth` unless `VerifyOptions.KeyUsages` says otherwise, so it has to pass the connector certificates' EKU explicitly; and "at most one intermediate" has to be enforced, by the root's path-length constraint or by checking the chain length.
2. The JWT signature verifies with the connector certificate's public key.
3. `aud` is exactly this connection's tunnel URL; `exp` and `iat` are within a clock-skew tolerance, and `exp - iat` is at most 5 minutes; the `jti` hasn't been seen within the validity window.
4. `cluster` equals the `clusterID` stored for this connection at registration. The connector takes it from the logical cluster of the connection record it acts on.
5. The session serves the hosts in the assertion's `hosts` claim (routing, not authority: the Provider-side never resolves them), plus the connection's front-proxy, which every tunnel may carry.

**What runs inside**: a dial proxy, in remotedialer's designed direction. The Provider-side, which accepted the WebSocket, dials through the connector, which opened it. For each stream, the connector checks the target against its allow-list (see "Egress") and forwards it as a TCP connection to kcp. The Provider-side then runs its own TLS to kcp on that stream.

**Provider-side dial hook**: picks a tunnel session that serves the target host. The front-proxy can use any tunnel of the connection; a shard's virtual-workspace host uses that shard's tunnels. More than one tunnel per host is allowed, for connector replicas during handovers and to spread load.

**Why no inner TLS here.** The payload is the Provider-side's own TLS to kcp, verified against the CA bundle from the control channel, and the access token travels inside it. Inner TLS on tunnels would add only authentication of the two ends; the assertion authenticates the PM-side, and the far end is irrelevant for confidentiality. The residual is the edge's ability to inject dials (see "Trust boundary"). Kcp traffic crosses two TLS layers instead of three: the outer (to the ingress) and the end-to-end TLS to kcp.

**Liveness and replacement.** WebSocket pings at the outer level, so ingresses see traffic and don't time the connection out; multiplexer keepalives inside. The connector replaces tunnels before the Provider's edge cuts them at its maximum connection age: it opens a new tunnel, sends the multiplexer's go-away on the old one, and closes it when its streams finish. An abrupt cut drops that tunnel's streams at once, as with any apiserver connection; client-go retries and relists.

### The connector CA

- **A CA is a keypair plus a certificate.** The private key signs connector certificates and never leaves PM's control plane. Providers pin the **certificate**, so they get its name, validity and constraints, and verify with standard library code.
- **Each connector CA is an immutable generation.** Its lifetime is well beyond the planned rollover interval (e.g. a 5-year CA replaced yearly), and it's never renewed. Changing anything about the CA means a **new** CA, added alongside the old one with an overlap. So the pinned certificate never changes under a connection.
- **The pinned certificate's expiry is a hard deadline.** A generation that was never rolled over stops being trusted, including at Providers that couldn't be reached when it was replaced.
- **cert-manager must never renew the CA by itself.** With `rotationPolicy: Always` (the default in recent cert-manager versions), a renewal silently replaces the key: new connector certificates are signed by a key no Provider has pinned, and new tunnels fail everywhere. It's the same trap as kcp's per-shard SA keys (finding 4). A cert-manager-managed connector CA sets `rotationPolicy: Never` as a safety net. A Vault- or KMS-backed issuer keeps the key out of Kubernetes Secrets and makes the policy irrelevant.
- **An optional intermediate** lets PM rotate the issuing key without involving Providers. Nothing revokes an intermediate, so it must be short-lived (e.g. weeks).
- **Connector certificates are the opposite**: nobody pins them, so they keep `rotationPolicy: Always` and a short lifetime, with a new key on every renewal.
- **Rollover**: the controller `PUT`s the next generation on every connection's control session, lets cert-manager issue connector certificates from it, then unpins the old one. It's the one operation that fans out to every connection, and it's scheduled, not urgent.
- **A compromised shard**: stop issuing to it; its certificate expires within hours. Roll over the CA only if its key is in doubt.

**What it removes**, compared with per-connection credentials on connectors: no connection keys on the data path, no registration call per shard, replica or key rotation, and no state in the connector. Adding a shard or a replica involves neither the controller nor any Provider.

### Per-shard connectors

One connector per kcp shard, following kcp's own model for virtual workspaces (finding 16). Each one knows which shard it serves from how it was deployed, nothing else.

```
Provider ingress
   ▲  ▲  ▲   at most one tunnel per (connection, shard), only where needed
   │  │  └── connector@shard-C ──▶ shard-C virtual-workspace host     consumers on C
   │  └───── connector@shard-B ──▶ shard-B virtual-workspace host     consumers on B
   └──────── connector@shard-A ──▶ front-proxy                        provider workspace on A
```

1. **Which tunnels exist.** A shard's connector keeps a tunnel for a connection when either:
   - **the connection's record came from its own shard's virtual-workspace endpoint** of the connection export. The provider workspace is hosted on this shard, and this connector carries the connection's front-proxy tunnel, from the start and even with no consumers yet; or
   - **one of the connection's Service endpoint slices lists its own shard's virtual-workspace URL**. kcp lists a shard there exactly when it has at least one binding to the export. Only slices in the provider workspace that point at an export **in that same workspace** count; otherwise a Provider could create slices for someone else's export.
2. **Shards come and go with their deployments.** A new shard's connector gets a connector certificate and starts watching the connection export. When kcp adds the shard to a connection's endpoint slices, the connector dials. Nobody registers anything. "Is there a connector for every shard?" is a deployment-health question, not something checked at runtime.
3. **Discovery: the connection export.** kcp's per-shard virtual workspaces only see their own shard, while `Provider` objects live in owners' workspaces on any shard. Each connector instead consumes PM's connection export like any multi-shard controller: it reads the export's APIExportEndpointSlice through the front-proxy, then list/watches through every shard virtual-workspace URL in it (multicluster-runtime's endpoint-slice provider):
   - **`ProviderConnection` records**: every object arrives with its logical cluster, the provider workspace's `clusterID`, and which endpoint delivered it tells the connector which shard hosts the workspace;
   - **the connections' Service `apiexportendpointslices`**, read-only, through the export's permission claim. That's how the connector learns which shards have consumers, without any shard-wide read of APIBindings.
4. **Allow-list from deployment configuration**: its own shard's virtual-workspace host and the front-proxy host. No `Shard` objects, no `/services/admin`.
5. **Deployment**: one StatefulSet per shard from PM's chart or operator, plus a cert-manager `Certificate` for its connector certificate. The connector isn't request-shaped, so it needs neither the virtual workspace framework nor kcp-operator's `VirtualWorkspace` resource, and doesn't get the cache-server credentials that resource mounts (finding 15).
6. **Front-proxy traffic on any tunnel.** The front-proxy routes workspace traffic to the right shard anyway, so every tunnel may carry it. The hosting shard's tunnel guarantees there's always one, even before any consumer binds.

**Replicas.** Within one shard, the connector runs several replicas and shares the connections among them:
- **StatefulSet**, so a restarted replica comes back as the same member and keeps its connections.
- **Rendezvous hashing of `connectionID` over the live replicas**, with membership from the headless Service's EndpointSlices (or a Lease per replica). Scaling moves only about 1/R of the connections.
- **No agreement needed.** Tunnels aren't exclusive, so a short overlap where two replicas both serve a connection is harmless.
- **Hand over before cutting off** on graceful changes: the new owner connects first, the old one sends go-away and drains.

**Load.** kcp's load is the same as with any direct Provider: the same requests to the same targets with the same SA identity. Each connector copies bytes for its shard's tunnels: per stream, a TCP connection to kcp and a copy loop, with no HTTP parsing and no TLS termination on kcp traffic. A dropped tunnel relists only that shard's watches; front-proxy traffic moves to another tunnel.

### The connection export and the Provider in its own workspace

The Provider holds an admin SA token for its provider workspace (RFC 006) and uses it directly against kcp, so no PM component sees its requests. What kcp enforces itself still applies:

**Connection records are read-only for the Provider.** The connection export sets `maximalPermissionPolicy.local`. kcp then checks every request to a resource bound from that export against RBAC in **PM's export workspace**, with the user and groups prefixed `apis.kcp.io:binding:` (finding 14). The check runs in each shard's authorizer chain before local RBAC, so admin in the provider workspace doesn't bypass it, and the Provider has no rights in the export workspace to widen it. PM grants there:
- `get`, `list`, `watch` on `providerconnections` to the group `apis.kcp.io:binding:system:authenticated`;
- writes (`create`, `update`, `patch`, `delete`, `deletecollection`, including `/status`) only to `apis.kcp.io:binding:<connection controller user>`.

A Provider therefore can't redirect connectors to another URL, or claim another connection's workspace.

**What the Provider can still do, and why it's acceptable:**

| Action | Effect | Handling |
|---|---|---|
| Delete the connection export's binding | The records disappear with it; connectors close the connection's tunnels | Hurts only the Provider. The Provider controller recreates the binding; the controller rewrites the record and reports `TunnelsConnected=False` meanwhile |
| Forge `apiexportendpointslices/status` | Connectors open idle tunnels on shards without consumers | Bounded by the number of shards; each connector's allow-list still covers only its own host. Alert on tunnels with no traffic |
| `create` on `serviceaccounts/token` for its own SA | Mints its own access tokens | Granted on purpose for self-rotation (see "Credentials and rotation") |
| Anything else within its admin RBAC | Its own exports, schemas, endpoint slices, RBAC and resources | The Provider's own business, as in RFC 006 |

The owner has no rights in the provider workspace; only PM's own components and the Provider do.

### kcp's CA bundle

- **What the controller sends**: the CA bundle PM's own components use to verify its kcp endpoints, i.e. whatever is in the kubeconfig the controller itself uses for `frontProxyURL`, plus whatever covers the shard virtual-workspace hosts. The Provider-side verifies the front-proxy and every endpoint-slice URL against it, end to end through the tunnels.
- **It follows PM's rotations automatically.** The controller reads it fresh and sends it with every `PUT /credentials`. If PM's own trust in its kcp works, the Provider-side's does too. Nothing depends on kcp-operator artefacts or rotation procedures (findings 8, 11, 12).
- **It's server-authentication trust only.** It tells the Provider-side whether it's talking to the real kcp. It plays no part in authenticating the PM-side.
- **How the Provider-side uses it**: it rebuilds its kcp clients when the bundle changes. When `CAData` is set, client-go trusts only that bundle, not the system roots.
- **Who can change it**: only a control session authenticated by the connection key. So a fake kcp needs the connection key, which is owner compromise. The onboarding token, the Provider's portal, the Provider's edge and a tunnel holder can't change it.

### No re-onboarding

v2 let a new request re-onboard an existing connection using only an onboarding token, which comes from the Provider's portal, outside the owner's workspace. Combined with a CA bundle delivered by the connection, a stolen token or a compromised portal account could take over an existing connection and point it at a fake kcp (finding 7). v4-b doesn't allow that:
- **Changes to an existing connection** (credentials, kcp CA bundle, key rotation, connector CAs, deletion) only happen inside a control session authenticated by the connection key.
- **Registration only creates new connections.**
- **Recovering a connection whose key is lost** goes through rebind, which the current key holder cancels automatically.
- **A server key rollover missed during a long outage** is recovered by re-reading `serverKeys` from `/server-keys` (web PKI), as trustworthy as registration, and only used after the pinned keys fail.

### Recovery: rebind

A recovery path after a lost connection key that an outsider can't use quietly, modelled on account recovery with a waiting period:

```
POST /v1/connections/{id}/rebind        (plain HTTPS, outside sessions)
Authorization: Bearer <tokenID>.<tokenSecret>
{ "idempotencyKey": "...", "connectionKey": "<PEM SPKI of the NEW connection key>" }
→ 202 { "completesAt": "..." }
```

1. The owner creates a `ProviderOnboardingRequest` for a `Provider` that is registered but whose connection Secret is gone. The controller generates a new key and calls `POST /rebind` with a fresh onboarding token.
2. The Provider-side accepts it only if the token's owner is the connection's owner. The connection's addressing is already fixed, so the request sends none. It starts a **waiting period** (e.g. 72h) and notifies the owner through every channel.
3. **Any successful control-session handshake with the current key during the waiting period cancels the rebind.** The controller's periodic `GET` does that, so a rebind against a connection whose key still exists never completes. Tunnels don't count: they don't prove the key is held.
4. After the waiting period, the Provider-side pins the new key. The connection keeps its `connectionID` and all its Provider-side state. The controller re-reads `serverKeys` from `/server-keys`, since the lost Secret held the pins.

- **A stolen onboarding token or a compromised portal account** can start a rebind, but the controller cancels it, and the owner is notified. A token alone never takes over a connection.
- **A leaked key is a rotation, not a rebind.** If the controller can still authenticate, it rotates.
- **Tunnels keep working throughout**, until the last delivered or self-minted access token expires.
- **Remaining gap**: a thief with the key who rotates it locks the owner out and can cancel the owner's rebinds. Owner compromise; the owner freezes or deletes the connection in the portal.

### Provider-side state

- **Everything the Provider-side keeps for a connection is keyed by `connectionID`**: provisioned resources, their credentials, mappings to consumer objects.
- **A new connection never inherits state from an old one**, even with the same `clusterID` and `frontProxyURL`. Otherwise registering a second connection for someone else's workspace (with a stolen token or a compromised portal account) and serving a fake kcp through it could make the Provider-side deprovision real resources, or write their credentials into the fake kcp.
- **A rebound connection keeps its state**, because it keeps its `connectionID`.
- **Moving state between connections** is a Provider-specific migration with the owner's explicit consent, out of scope here.

### Provider-side: serving many PM instances

- **The Provider already knows its endpoint slices.** It owns its APIExports, so its operator code knows which endpoint slices it reconciles against. Nothing about them is negotiated at registration, and the SDK doesn't discover them.
- **The SDK hands the Provider connections; the Provider creates the providers.** For each connection, the SDK supplies a client config. The Provider's operator code creates one multicluster-runtime endpoint-slice provider per endpoint slice it's interested in, using that config, and reconciles all of them as one set of clusters.
- **Connections come and go at runtime.** The SDK notifies the Provider's code on registration and deletion, which starts or stops that connection's providers without restarting the Provider.
- **Cluster names are prefixed with the `connectionID`.** Logical cluster names are only unique within one kcp, so clusters (and all Provider-side state) are keyed by `(connectionID, clusterName)`. multicluster-runtime's `multi` provider composes providers with a name prefix and may fit as is.
- **Each connection's client config** carries its `frontProxyURL`, the latest access token, the kcp CA bundle and a `Dial` that picks a tunnel by target host:

  ```go
  cfg := &rest.Config{
      Host:            conn.FrontProxyURL,
      BearerToken:     conn.Token(),                                   // kept current by the SDK
      TLSClientConfig: rest.TLSClientConfig{CAData: conn.KcpCABundle}, // end to end to kcp; empty = web PKI
      Dial:            tunnels.DialContext(conn.ID),                   // picks a tunnel serving the host
  }
  ```

  multicluster-runtime derives per-endpoint configs (shard virtual-workspace URLs) by changing `Host`; they inherit the `Dial`. The SDK keeps the token and bundle current, so the Provider's providers don't have to be recreated.

### Credentials and rotation

- **Send parts, not a kubeconfig.** Token, CA bundle and addressing, assembled into a client config by the SDK.
- **The PM-side mints, the Provider-side holds.** The controller mints a token for the connection's SA through TokenRequest and `PUT`s it at about half its TTL.
- **Keep the access-token TTL short (e.g. 1 hour).** In PM's kcp deployments, deleting the SA doesn't revoke tokens already issued (finding 2), so the TTL is the revocation window for kcp access if `DELETE` never arrives.
- **A controller outage stops token delivery.** Traffic would continue only as long as the last delivered token lasts. So **Provider-side self-rotation is on by default after the claim**: the Provider controller grants the connection's SA `create` on `serviceaccounts/token` with `resourceNames` for that SA, in the provider workspace. The controller's `PUT` stays the baseline and recovers a Provider-side whose token expired. Self-minted tokens live until they expire (finding 2), so their TTL must be short too.
- **Controller HA** (leader election, or connections sharded by hash) is a requirement, not an option.
- **Long-lived key, rotated automatically; short-lived everything else.** The connection key stays in the owner's workspace and is rotated by the controller. Connection certificates, connector certificates, session assertions and access tokens are short-lived and re-issued freely.
- **Access tokens never pass a TLS-terminating hop in readable form.** They only travel inside the inner TLS of control sessions, and inside the Provider-side's own TLS to kcp.
- **No static kubeconfig for remote `Provider`s.** The owner's workspace holds no long-lived credential for the provider workspace.

### Egress

- **Tunnel dials to kcp.** The Provider-side chooses each dial's target, so the allow-list is a security boundary. Each connector allows only its own shard's virtual-workspace host and the front-proxy host, from deployment configuration. It resolves once, checks every address, dials those exact addresses, and refuses link-local, multicast and cloud-metadata addresses. Per-connection limits on concurrent streams and bandwidth stop one Provider from exhausting a connector for others.
- **Dials to Providers.** The controller validates the base URL before registering and before writing a record; the controller and connectors apply the same rules when dialling: `https` only, public addresses only, resolve once and check. Records are PM-written only, so this is defense in depth against a bad URL reaching PM's internal network.

### What each credential exposes

| Credential | Lifetime | Stored at | If stolen |
|---|---|---|---|
| Onboarding token | minutes, single-use | PM-side Secret until spent; hash on the Provider-side | The thief can register **their own** new connection under the owner's portal account, including one that claims the owner's `clusterID`, or start a rebind that the controller cancels. No access to existing connections or their state. The owner is notified. |
| Provider portal account | long-lived | Provider-side | Same as issuing tokens, plus whatever the portal allows (deleting or freezing connections). Can't take over an existing connection or its state. |
| Connection key (private) | long-lived, rotated automatically | Owner's workspace; read only by the controller | **Owner compromise, for one connection**: control calls, including a fake kcp CA bundle, rotating to the thief's own key and cancelling the owner's rebinds. No other connection is affected, and nothing in kcp is granted. The owner is notified of key changes. A copy from before the last rotation is useless. |
| Connection certificate | hours | Controller memory | Public: useless without the private key. |
| Connector certificate key | hours | Mounted in that shard's connector | Tunnels on that shard for every connection, until it expires: denial of service, dial metadata. No control calls, no kcp credentials, no readable kcp traffic. |
| Connector CA key | one generation, rolled over with overlap | Control plane (ideally KMS or Vault) | The same for every shard, until the generation is unpinned or expires. The control plane already holds every connection key, so this doesn't widen a control-plane compromise. |
| Intermediate key (if used) | short (e.g. weeks) | Control plane | Same as the CA's, until the intermediate expires. |
| Session assertion | minutes | On the wire only | Bound to one connection; a `jti` cache stops replays. Useless at any other Provider. |
| Provider's server private key | long-lived, rotatable | Provider-side Secret | Only useful together with control of the path to the connection server (the Provider's own edge, or DNS for the base URL). Then the thief can impersonate the Provider-side on control sessions and receive access tokens. Rotate the server key with overlap and re-check the edge. |
| Outer TLS / ingress / CDN | — | Provider's edge | Sees the spent onboarding token and opaque control sessions. On tunnels: dial targets and timing, opaque kcp TLS, and anonymous reach to the allow-listed kcp hosts by injecting dials. Can drop or delay traffic. |
| Access token | hours (keep it short) | Provider-side | Access to the provider workspace and its exports' virtual workspaces within RFC 006 RBAC, until it expires. Records stay read-only. Deleting the SA doesn't end this early in PM's current kcp deployments. |

### Threat model: how credentials get stolen

**The connection keys** live in owners' workspaces and in the controller's memory.

| How | Notes |
|---|---|
| **A compromised controller** | The controller reads every connection's key and runs the connector CA's issuer. Code execution in it, a supply-chain compromise of its image, or root on its node exposes **all** connections at once. The highest-impact path; it's one component, off the traffic path, and can be locked down. |
| **Read access to the connection Secret** | Anyone with `get`/`list` on Secrets where it's stored: workspace admins, GitOps tools, other controllers with wildcard Secret permissions. |
| **Backups and etcd snapshots** | kcp's etcd and backup tools contain the Secret. Without encryption at rest it's just base64. Periodic rotation makes old copies useless. |
| **Dumps** | `kubectl get secret -o yaml` pasted into a ticket, support bundles, audit logging at `RequestResponse` level for Secrets. |
| **Insiders** | PM administrators. |

**Connector certificates** live in connectors. A compromised connector or its node yields one shard's certificate for hours, which allows denial of service and dial metadata on that shard's tunnels, nothing more.

**The onboarding token.**

| How | Notes |
|---|---|
| **A compromised portal account** (phishing, password reuse) | The most likely path. The attacker doesn't steal a token; they **issue their own**. With no re-onboarding, per-connection state and cancellable rebinds, it only yields new connections. |
| **Handling by people** | Tokens pasted into chat, email or tickets, picked up by clipboard managers, seen in screen shares, or left in browser history if the portal puts them in URLs. |
| **Committed to git** | A user saves the token Secret as a manifest and commits it. |
| **Ingress or CDN logs on the base URL** | TLS is terminated there, so the token is visible in plaintext. It's spent by then; key-bound retries stop replays. |

**Ranked by likelihood × impact:**
1. **The controller or its node.** Less likely, but every connection is exposed at once.
2. **The owner's workspace (RBAC, backups, dumps).** The owner's responsibility; same exposure as the workspace's other credentials, limited to one connection per leaked key.
3. **A compromised portal account.** Likely, but limited to new connections and rebinds that get cancelled.
4. **Server key plus control of the Provider's edge.** Needs two compromises on the Provider-side.
5. **A connector or its node.** Denial of service and metadata for one shard, for hours.
6. **Logged tokens or requests.** Harmless.

**Mitigations.**
- **PM-side / owner:**
  - keep connection Secrets where only the controller can read them, and never in the provider workspace, which the Provider-side's own SA can reach;
  - enable kcp etcd encryption at rest, and encrypt backups or exclude these Secrets from them;
  - rotate connection keys periodically, so leaked copies expire;
  - keep the connector CA key in Vault or a KMS;
  - later: connection keys in a KMS or HSM, with the controller only requesting the one signature per handshake.
- **Provider-side:** key all state by `connectionID`; protect the server key like any TLS key; rotate it with a generous overlap.
- **Portal:** MFA, no tokens in URLs, a list of issued and used tokens, owner notifications on registrations, key changes and rebinds, freezing or deleting connections, and a command instead of a manifest for the token Secret. GitOps users should use Sealed Secrets or External Secrets.
- **Protocol:** no re-onboarding, cancellable rebinds, key-bound idempotent retries, and no kcp credentials outside the inner TLS.

### Resources

```yaml
apiVersion: providers.platform-mesh.io/v1alpha1
kind: ProviderOnboardingRequest
metadata:
  name: my-provider-2026-10-11
spec:
  tokenSecretRef: { name: my-provider-token, key: token }
  provider:
    name: my-provider                                # created if missing; see the table below
    spec:
      remote:
        caBundle: ""                                 # optional, for a private CA on the Provider's base URL
        transport: Tunnel
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
  name: my-provider-connection                       # owner's workspace; owned by the Provider; read by the controller only
type: Opaque
stringData:
  key: "<PEM private key: the pinned connection key>"
  pendingKey: ""                                     # only during a rotation or rebind
  serverKeys: "<pinned PEM SPKIs of the connection server>"
  url: https://provider.example.com/pm               # base URL from the token
  connectionID: "..."
---
apiVersion: apis.kcp.io/v1alpha2
kind: APIExport                                      # in a PM workspace
metadata:
  name: connections.platform-mesh.io
spec:
  resources:
    - name: providerconnections
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
    resources: [ providerconnections ]
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
    resources: [ providerconnections, providerconnections/status ]
    verbs: [ create, update, patch, delete, deletecollection ]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: connections-mpp-write
roleRef: { apiGroup: rbac.authorization.k8s.io, kind: ClusterRole, name: connections-mpp-write }
subjects:
  - { apiGroup: rbac.authorization.k8s.io, kind: User, name: "apis.kcp.io:binding:<connection controller user>" }
---
apiVersion: connections.platform-mesh.io/v1alpha1
kind: ProviderConnection                             # in the provider workspace :root:providers:<name>-<suffix>
metadata:
  name: connection                                   # one per provider workspace
spec:
  connectionID: "..."
  url: https://provider.example.com/pm               # validated against the egress rules
```

- **How the request treats `provider.name`**:

  | The `Provider`... | The request... |
  |---|---|
  | doesn't exist | creates it, waits for the workspace and SA, then registers |
  | exists, not registered (a leftover from a failed attempt, or created by hand) | reuses it and registers |
  | exists, registered, connection Secret present | **refuses**. Changes to a live connection happen through the controller. To start over, delete the `Provider` |
  | exists, registered, connection Secret missing | starts a rebind |
  | exists, but isn't a remote `Provider` (e.g. a ManagedProvider) | refuses |

- **No `clusterID` in the record**: its logical cluster is the provider workspace. **No SA**: connectors don't mint tokens.
- **RBAC**: the controller writes records through the export's virtual workspace and mints tokens for each connection's SA; connectors get read-only `apiexports/content` on the export. The Provider's SA stays admin in its workspace, as in RFC 006, plus `create` on `serviceaccounts/token` for itself.
- `Provider.status.connection` mirrors `url` and `connectionID`. The `Provider`'s conditions are `Registered`, `KcpReachable`, `CredentialsValid`, `TunnelsConnected`, `KeyCurrent` (False while a rotation is pending) and `Rebinding`. Per-shard detail stays in metrics.

### Restart and failure semantics

| Scenario | Behavior |
|---|---|
| Connector replica restarts, scales or rolls out | It signs new assertions with the mounted certificate and reconnects. No control-plane or Provider call. Graceful changes hand over before cutting off. |
| One shard's connector is down | Only that shard's endpoint traffic stops; front-proxy traffic uses other tunnels of the connection where there are any. |
| Controller down | Onboarding, rotations, rebinds and offboarding wait. Tunnels keep working. Token delivery stops: self-rotation keeps the Provider-side going; without it, kcp access ends when the last token expires. |
| Provider-side restarts | Tunnels drop and connectors reconnect; the controller's next `PUT` restores credentials if the Provider-side lost them. |
| Outage of any length | Nothing the PM-side needs expires: the controller creates a new certificate for its key, cert-manager renews connector certificates, and the next `PUT` restores an expired token. |
| Ingress drops a long-lived session | The connector reconnects; pings keep idle timeouts from triggering, and tunnels are replaced before the edge's maximum connection age. |
| Shard added or removed | Its connector comes or goes with it. Nobody registers anything. |
| First consumer binding on a shard | kcp adds the shard to the endpoint slice; that shard's connector sees it through the permission claim and dials. The URL may briefly appear before the tunnel is up; client-go retries. |
| Connector certificate expires or renews | cert-manager renews it under the same pinned CA. Nothing reaches the Provider. |
| Connector CA generation nears expiry without a rollover | Hard deadline: tunnels under it fail everywhere after it. Alert on its remaining lifetime. |
| PM rotates kcp's CA | Nothing to coordinate: the next `PUT` carries the CA bundle PM's own components now trust. |
| Wrong or incomplete kcp CA bundle | `kcpReachable` is `false` with the TLS error for the affected hosts. During registration the request stays in `VerifyingKcp` and fails after a timeout; later, `KcpReachable=False`. Fix PM's trust; the next `PUT` repairs it. |
| `POST` response lost | Retry with the same token, idempotency key and connection key: same `connectionID`. A different key is refused. |
| Key rotation response lost, or controller crash mid-rotation | The old key still works; the controller compares fingerprints from `GET` with `key` and `pendingKey`, and resumes. Tunnels are unaffected. |
| New key never used | The pending entry expires; the connection stays on the old key. |
| Server key rotated | The controller picks up the new key from `GET` during the overlap. A controller that missed the whole overlap re-reads `serverKeys` from `/server-keys`. |
| Connection Secret lost | Restore a backup taken after the last rotation, or rebind. Tunnels keep working throughout, until the last token expires. |
| Rebind started by someone else | The controller's next control handshake cancels it; the owner is notified. |
| Thief rotates the key | The controller can't authenticate; the owner is notified and freezes or deletes the connection in the portal. |
| Provider deletes the connection export's binding | Its tunnels close; the Provider controller recreates the binding and the controller rewrites the record. |
| A record points at the wrong connection (PM-side bug) | The assertion's `cluster` claim doesn't match the `clusterID` stored at registration, so the Provider-side refuses the tunnel. |
| Connection deleted on the Provider-side | Control handshakes and tunnels are refused. `CredentialsValid=False` on the `Provider`. Recovery is a new connection. |
| Provider-side runs multiple replicas | Tunnels for different shards can land on different replicas, while the replica running a connection's endpoint-slice providers needs all of them. Route through Lease holders, or shard reconciliation by connection and forward dials to the replicas holding the tunnels. |

### Offboarding and revocation

- **Deleting the `Provider`** makes the controller delete the connection record first, so every connector closes its tunnels. Then it calls `DELETE` in a control session, and the Provider controller cleans up the SA and RBAC (RFC 006). The connection Secret is garbage-collected with the `Provider`.
- **An unreachable Provider-side doesn't block deletion forever**: after a timeout the finalizer gives up on `DELETE` and cleans up the PM-side.
- **Deleting the connection on the Provider-side** takes effect immediately: open control sessions and tunnels are closed and new ones refused. No CRL or OCSP is needed.
- **kcp access** ends when the last access token expires, unless `DELETE` arrives and the Provider-side discards it first. In PM's current kcp deployments, deleting the SA doesn't end it earlier (finding 2). Closing the tunnels ends access sooner where kcp isn't reachable from the Provider-side otherwise.

## Findings

Findings 1 to 13 come from v4 (checked on 2026-10-07 to 09). Findings 14 to 17 come from v5 (2026-10-08 to 11).

1. **kcp bound tokens carry the cluster and the SA UID.** The claims are nested under `kubernetes.io`: `clusterName` (a kcp addition), `namespace`, and `serviceaccount.{name, uid}`. `sub` doesn't include the cluster, so it isn't unique across workspaces. Legacy Secret-based tokens use a flat `kubernetes.io/serviceaccount/clusterName` claim. Sources: `kcp/vendor/k8s.io/kubernetes/pkg/serviceaccount/claims.go:53-94`, `legacy.go:65`, `kcp/pkg/server/filters/serviceaccounts.go:70`.
2. **In PM's kcp, deleting an SA doesn't revoke its bound tokens.** kcp-operator always passes `--service-account-lookup=false` to every shard (`kcp-operator/internal/resources/compiledrootshard/deployment.go:232`, `compiledshard/deployment.go:271`), and a kcp carry patch then skips the SA existence and UID check (`claims.go:196`). kcp itself has supported cross-shard SA lookup since v0.32, with a 1-minute cache (`kcp/pkg/server/serviceaccount/cache.go`, `pkg/shardlookup/ttl_cache.go`); the operator's flag predates that. Not yet checked on a running PM cluster.
3. **Each shard has its own SA signing key.** kcp-operator issues one cert-manager certificate per shard (RSA 4096, valid 1 year, renewed 7 days before expiry; `kcp-operator/internal/resources/shard/certificates.go:125`). Every shard and the front-proxy trust all shards' public keys (`utils/authentication.go:82`), so a token minted through the front-proxy works against every shard's virtual workspace.
4. **kcp signing-key rotation has no overlap.** The same certificate is both the signing key and the verification key, and kcp reads it only at startup. When a shard restarts with a renewed key, it signs with the new key at once. Whether renewal creates a new key depends on cert-manager's `rotationPolicy`; recent versions default to `Always`. The same trap applies to any long-lived CA managed by cert-manager, including the connector CA.
5. **kcp serves its JWKS per workspace through the front-proxy** (`/openid/v1/jwks`, tested on 2026-10-08). Key distribution wasn't what ruled out v3; finding 4 was.
6. **Railgrid's edges provider** uses bound SA tokens as the only credential after enrolment, with Lease-based tunnel routing between replicas (`internal/tunnel/registry.go`, `remote.go`, `connman.go`). Its agent refreshes a token by presenting the current one, so an agent offline for longer than the 24h TTL needs a new join token: the power-off budget v4-b avoids. Its tunnel uses `revdial`, which opens a new connection per dial.
7. **A connection-supplied kcp CA bundle plus token-only re-onboarding lets outsiders present a fake kcp.** This affected v2 and v3: whoever holds the connection identity can set the CA bundle and answer the Provider-side's dials with a fake kcp. With re-onboarding, a stolen onboarding token was enough. v4-b has no re-onboarding, rebinds are cancelled by the current key holder, and Provider-side state is keyed by connection.
8. **kcp-operator signs every serving certificate with one CA** (`<rootshard>-server-ca`, valid 10 years; `kcp-operator/internal/resources/frontproxy/certificates.go:123`, `rootshard/certificates.go`, `shard/certificates.go`, `virtualworkspace/certificate.go:122`). Context for what the CA bundle contains in PM today; v4-b doesn't depend on it.
9. **v2's retry rule let a replayer get a working secret.** Retries keyed only on `(tokenID, idempotencyKey)` issued a fresh secret to whoever sent them. v4-b binds retries to the first attempt's connection key and returns nothing secret.
10. **A registration body with an access token exposes a live kcp credential at the edge.** v2, v3 and early v4 drafts did that. v4-b sends access tokens only inside the inner TLS of control sessions.
11. **kcp-operator's merged CA bundle is not a general trust source.** `<rootshard>-merged-ca-bundle` only exists when `spec.caBundleSecretRef` is set, which PM doesn't do, and only the `external-logical-cluster-admin-kubeconfig` uses it (`rootshard/ca_bundle.go`, `pkg/controller/rootshard/controller.go:212-217`). v4-b takes kcp's CA bundle from PM's own trust instead.
12. **kcp-operator's `Bundle` was never a trust-distribution mechanism, and is gone.** It copied full CA Secrets including private keys for migration and DR, and was removed in commit `406169b` (#293, 2026-08-18).
13. **kcp's client CA would turn connection certificates into kcp credentials.** kcp-operator's `Kubeconfig` resource issues certificates with any `username` and `groups`. Anchoring connections on it would make every connection key a kcp credential and give the PM-side the power to mint any kcp identity.
14. **An APIExport's maximal permission policy caps even an admin of the binding workspace** (kcp `ee9e6f3f3`, read on 2026-10-11; not tested for this export). `MaximalPermissionPolicyAuthorizer` runs in each shard's authorizer chain before bootstrap, local and global RBAC (`pkg/server/options/authorization.go`). For a resource listed in an APIBinding's `status.boundResources`, it checks RBAC in the export's workspace with the user name, groups and warrants prefixed `apis.kcp.io:binding:` (`pkg/authorization/maximal_permission_policy_authorizer.go`). Writes through the APIExport virtual workspace are forwarded impersonating the caller plus a warrant, and each effective user is authorized separately (`pkg/virtual/apiexport/builder/build.go`, `pkg/authorization/warrants.go`), so a grant to `apis.kcp.io:binding:<controller user>` should be enough. kcp's e2e test `TestMaximalPermissionPolicyAuthorizer` (`test/e2e/apibinding/maximalpermissionpolicy_authorizer_test.go`) shows a consumer-workspace admin refused a write the policy doesn't grant, and that deleting the APIBinding deletes the bound resources. The policy doesn't cover claimed or `apis.kcp.io` system resources, and kcp's system-CRD authorizer only protects `apibindings/status` and `apiexports/status`.
15. **kcp-operator gives `VirtualWorkspace` servers the cache-server credentials when an external cache server exists**, which PM runs (`kcp-operator/internal/resources/compiledvirtualworkspace/deployment.go`). Far broader than a connector needs; one reason v4-b's connectors are plain StatefulSets.
16. **kcp runs its virtual workspaces per shard.** Each shard's `spec.virtualWorkspaceURL` points at its own virtual-workspace server, and APIExportEndpointSlice URLs are built from it (`kcp/pkg/reconciler/apis/apiexportendpointsliceurls`), so endpoint traffic goes straight to each shard, bypassing the front-proxy. The front-proxy routes only `/clusters/` by shard (`kcp/pkg/proxy/mapping.go`). Hence one connector per shard, each with that shard's host on its allow-list.
17. **A tunnel's limits are mostly those of any client-go connection.** One TCP connection for many requests, shared flow control and mass relists when it dies are normal client-go behaviour. What a reverse tunnel really changes is that the client can't open connections on its own, and that the path runs through the Provider's L7 edge. A multiplexed dial proxy restores the first; pings and graceful replacement handle the second.

## Comparison

| | v4 | v4-b | v5 |
|---|---|---|---|
| kcp credentials on the Provider-side | Access token + CA bundle | Access token + CA bundle | None |
| PM-side identity, control path | Pinned connection key, inner mTLS | Pinned connection key, inner mTLS | Bearer connection secret |
| Tunnel authentication | Same key, inner mTLS | Session assertion, connector certificate from a pinned connector CA | Same as v4-b |
| Who reads connection keys or secrets | The connector | The controller only | The controller only |
| A compromised connector yields | Every connection's key | One shard's tunnels for hours (DoS, metadata) | One shard's sessions for hours, with plaintext traffic and fake data |
| TLS layers on kcp traffic | 3 | 2 | 2 |
| Hop on the data path | L4 byte copy | L4 byte copy, per shard | L7 proxy, per shard |
| Connector placement | Open item | Per shard | Per shard |
| What the Provider's edge gets | Spent token, opaque bytes | Spent token, opaque control sessions, tunnel metadata, anonymous reach to allow-listed kcp hosts | Spent token, the connection secret on control calls, assertions, reconcile traffic |
| Fake kcp by an outsider | No | No | No |
| Protection of PM objects in the provider workspace | None needed (no records) | Maximal permission policy on records | Maximal permission policy plus connector denylist |
| Request-level policy and audit at the hop | No | No | Yes |
| Revoking kcp access | Token TTL (finding 2) | Token TTL (finding 2); tunnels close at once | Immediate |
| Traffic during a control-plane outage | Same component | Until tokens expire, or indefinitely with self-rotation | Indefinitely |
| Recovery after a lost PM-side credential | Recent backup, else new connection | Rebind, state kept | Rebind, state kept |
| Provider-side software | Inner mTLS both ways, pinning, token and CA handling, dial-proxy server | Inner mTLS for control, assertion verification for tunnels, token and CA handling, dial-proxy server with host routing | Bearer check, assertion verification, multiplexer server, dial hook |
| PM-side components | Connector | Controller, per-shard connectors, connector CA, connection export | Same as v4-b, connectors as virtual workspaces |
| Main cost | Connector holds every key; placement open | Two authentication schemes on the Provider-side; a connector CA to run; controller on the token path | A second, lighter virtual-workspace hop for remote Providers' traffic; a connector CA to run; edge sees traffic |

## Changes from v4

| v4 | v4-b | Why |
|---|---|---|
| One connector reads every connection key and does everything | Connection controller (keys, control calls, tokens) plus stateless per-shard connectors (tunnels) | Removes v4's top threat; control/data split from v5 |
| Tunnel inside inner mTLS, authenticated by the connection key | Tunnel on a plain WebSocket, authenticated by a session assertion from a pinned connector CA | Connectors need no per-connection secret; the payload is already end-to-end TLS |
| Three TLS layers on kcp traffic | Two | The inner layer only added authentication on tunnels |
| Connector placement an open item; allow-list from `Shard` objects | One connector per shard; allow-list from deployment configuration; tunnels only where there's traffic | Locality, failure isolation, tiny allow-lists (finding 16) |
| No PM objects in the provider workspace | Connection export and records, read-only for the Provider by maximal permission policy | Per-shard discovery without shard-wide reads (finding 14) |
| `kcpReachable` for the connection | Per-host status (`requiredHosts`) | Readiness without topology |
| Lost key without a recent backup means a new connection | Rebind with a waiting period, cancelled by any control handshake with the current key | Recovery without letting outsiders take over |
| Self-rotation optional | Self-rotation on by default after the claim | Bridges controller outages now that the controller delivers tokens |
| Allow-list check on dial | Resolve once, check every address, dial exact addresses, refuse metadata; base-URL egress rules; per-connection limits | Egress hardening from v5 |
| Key rotation reconnects the tunnel | Tunnels unaffected by key rotation | Tunnels don't use the key |

## Rejected alternatives

- **mTLS directly on a dedicated hostname.** A second hostname reached through TLS passthrough, or through an ingress that verifies client certificates and forwards them in a header. Assumes too much about the Provider's edge: cloud L7 load balancers can't pass TLS through, passthrough bypasses CDNs and WAFs, and verifying ingresses need fragile configuration.
- **An owner CA, generated or supplied by the owner** (an earlier revision of v4). A generated CA per connection has the same blast radius as a pinned key, with an extra layer. Owner-supplied CAs brought manual rotation steps and a lock-out risk.
- **kcp's client CA as the anchor.** Connection certificates would be kcp credentials, and the PM-side would need the power to mint any kcp identity (finding 13).
- **kcp's serving CA as the anchor.** Answers the wrong question, and every kcp serving process holds a key from it.
- **A dedicated PM connections CA as the whole connection identity.** One leaked CA key would expose every connection's control path. The connector CA authorizes tunnels only (see "Choosing the anchors").
- **Pinning kcp's CA at registration**, changeable only with a fresh onboarding token. Every kcp CA rotation would then need a portal step per connection.
- **A PM-published trust bundle the Provider-side follows.** Depends on PM publishing bundles with an overlap before every switch; kcp-operator's merged bundle isn't a reliable source (finding 11), and its `Bundle` never was one (finding 12).
- **Re-onboarding existing connections with an onboarding token.** Lets outsiders take over a connection (finding 7).
- **Rebind without a waiting period**, or one that the current key holder can't cancel. Turns a stolen token into a takeover again.
- **Application-level signatures** (v1's DPoP-style proofs). Work through any ingress, but are custom crypto and need payload encryption for confidentiality past a terminating hop.
- **Other ways to tunnel TLS through an ingress** (HTTP `CONNECT`, HTTP/2 extended `CONNECT`, raw TCP upgrades). Less widely supported than WebSockets.
- **kcp SA tokens as the only credential** (v3). Every connection depends on kcp's SA signing keys, which rotate without overlap (finding 4).
- **Revocation through CRLs or OCSP.** Not needed: the Provider-side checks the connection entry on every handshake and session.
- **Keeping inner mTLS on tunnels, with the connector certificate as the client certificate.** Closes the edge's injected dials, at the cost of a third TLS layer on all kcp traffic and inner-TLS handling on both ends of every tunnel. Worth reconsidering only for deployments where kcp's hosts aren't otherwise reachable from the internet (see "Open items").
- **Connectors delivering access tokens over tunnels.** Would keep traffic independent of the controller, but without inner TLS on tunnels the tokens would cross the Provider's edge in plaintext (finding 10); with it, connectors would need the connection keys again.
- **Connectors keeping the connection keys** (v4 with per-shard placement only). Multiplies the top threat by the number of shards.
- **Connectors terminating the Provider-side's kcp traffic and injecting tokens** (v5). Removes kcp credentials from the Provider-side and allows a request-level policy, but adds a second virtual-workspace hop in front of the APIExport virtual workspace (lighter than it, but with its own timeouts and multiplexer backpressure), and, with plain sessions, requires trusting the Provider's edge with reconcile traffic and the session's authority. v5's proposed inner TLS on sessions removes that requirement, which leaves v5 ahead of v4-b on principles (see the comparison notes, "The Provider's edge"). The alternative design, not a variant of this one.
- **One connector for all shards.** One tunnel per connection would carry every shard's traffic, so losing it relists everything; load wouldn't scale with shards; the allow-list would need the whole shard list.
- **Every shard's connector opening a tunnel for every connection.** N connections × N shards tunnels, mostly idle.
- **Per-shard keys registered on every connection.** Needs connectors to persist keys and costs a call per connection whenever a shard is added or rotates its key. The connector CA removes all of that.
- **Pinning the connector CA's bare public key** instead of its certificate. Loses the built-in expiry of trust, the CA constraints, standard verification and room for an intermediate.
- **Per-shard status in the owner's workspace.** Exposes PM's topology to owners and needs connector writes.
- **`/services/admin` for the connector's allow-list.** Exposes the whole installation's shard layout to a component that only needs its own shard's host.
- **Discovery from owners' `Provider` objects.** Owners can edit those objects, including the URL connectors would dial.
- **Connection records in a PM-internal workspace, through the cache (`ClusterCachedResource`), or as annotations on `Workspace` objects.** Puts records on one shard away from what they describe, needs a separate lookup for the hosting shard, or needs cache-server access (finding 15).
- **Wildcard read of APIBindings on each connector's shard** to find consumers. Needs shard-wide read access; the permission claim on endpoint slices uses kcp's own answer.
- **A connector policy denying the Provider's writes**, as in v5. Needs a hop that sees requests; v4-b's Provider talks to kcp with its own token. Records are protected by kcp instead (finding 14).

## Open items

- **Edge-injected dials.** Confirm what kcp's front-proxy and virtual-workspace hosts serve to anonymous requests in PM's deployments, and whether they're reachable from the internet anyway. If not, decide between accepting the exposure, keeping inner mTLS on tunnels as a per-deployment option, or having the connector refuse streams that don't start with a TLS ClientHello for an allow-listed SNI.
- **Two authentication schemes on the Provider-side.** Inner mTLS with key pinning for control sessions and assertion verification for tunnels, both in the SDK. Check whether control calls could use signed requests (e.g. RFC 7523-style assertions with the connection key) inside plain HTTPS instead, which would leave one handshake style; tokens would then need payload encryption.
- **Controller on the token path.** Token TTL, self-rotation by default and controller HA together decide how long traffic survives a controller outage.
- **The verification helpers.** One small, tested implementation of the control-path callbacks (pinning `serverKeys` and the connection key) and of session-assertion verification, shipped in the SDK and used by the controller.
- **Connection key parameters.** Key type (ECDSA P-256 or Ed25519), certificate lifetime, rotation period, pending-key expiry, how quickly old-key control sessions are closed after promotion, and clock-skew tolerance.
- **Connector CA.** Issuer (cert-manager with a KMS- or Vault-backed issuer), generation lifetime and rollover interval, whether to use an intermediate, alerting on a generation's remaining lifetime, connector certificate lifetime, assertion lifetime, `jti` cache size.
- **Connection export.** Which PM workspace hosts it; confirm that the permission claim on `apiexportendpointslices` works and is readable through the export's virtual workspace; the binding's lifecycle in the Provider controller (create with the workspace, recreate if deleted); an e2e test of the maximal permission policy, modelled on `TestMaximalPermissionPolicyAuthorizer`, including the controller's writes through the export's virtual workspace with only the prefixed-user grant.
- **Where Service endpoint slices live.** The tunnel rule reads them in the provider workspace and only counts slices for exports in that workspace. RFC 006 leaves APIExport bootstrap out of scope, so state it as a requirement.
- **Naming.** pm-operator already has a `ProviderConnection` type (`corev1alpha1.ProviderConnection`, `pm-operator/pkg/subroutines/defaults.go`) for its own components' kubeconfigs. Pick a different name for the record.
- **Rebind details.** Length of the waiting period, rate limits, and what happens when a thief holding the key keeps cancelling the owner's rebinds (portal freeze is the current answer).
- **Connector replicas.** The membership source, replica count per shard, sizing for an L4 forwarder against the shard's endpoint traffic, pacing of rollouts so relists don't burst; alerting on shards without a running connector and on tunnels with no traffic.
- **Tunnel library.** remotedialer vs. Konnectivity: per-stream backpressure, a go-away for graceful replacement, and dial addressing by host.
- **Ingress limits for WebSockets.** Document what Providers must check: idle timeouts, maximum connection age, maximum frame or message size, and WAF behaviour on binary frames. Pick a ping interval that fits common defaults.
- **Non-exportable connection keys** (KMS or HSM). Possible because a TLS handshake needs one signature from the key; worth it to shrink the controller's blast radius.
- **Server key rotation.** The overlap period, whether `GET` announces retirement dates, and rate limits on the `server-keys` refetch.
- **Garbage collection of orphaned connections** on the Provider-side (no sessions for N days), and how owners delete them in the portal.
- **Which CA bundle the controller sends.** Its own kubeconfig's CA for `frontProxyURL` covers the front-proxy; confirm what covers shard virtual-workspace hosts in PM's setups.
- **Readiness gate timeout.** How long the request waits in `VerifyingKcp` before failing, and how the Provider-side reports TLS errors in `lastError` without leaking details.
- **Fix in kcp-operator: drop `--service-account-lookup=false`.** Finding 2. kcp has supported cross-shard lookup since v0.32. That would bring back SA deletion as revocation, within about a minute, for this design and for RFC 006. Confirm the flag on a running PM cluster first.
- **Provider-side multi-replica.** Lease routing between replicas (railgrid's edges provider is a working example, finding 6), or reconciliation sharded by connection with dials forwarded to the replicas holding the tunnels. More pressing with per-shard tunnels.
- **`direct` mode.** The Provider-side already holds kcp credentials, so "direct" just means dialling kcp's URLs without tunnels, where PM's kcp is reachable. Control sessions, rotation and rebind stay the same; readiness covers the same hosts.
- **Addressing changes.** `clusterID` and `frontProxyURL` are immutable; decide whether moving a workspace or changing PM's front-proxy URL should ever be supported without a new connection.
- **Request permissions.** Who may create `ProviderOnboardingRequest`s in an owner's workspace, and what RBAC the controller needs there.

## Prior art

- **kubeadm bootstrap tokens**: `tokenID.tokenSecret`, stored hashed, short-lived.
- **Kubernetes CSR-based node bootstrapping (kubelet TLS bootstrap)**: a bootstrap token, then a certificate-based identity with automatic rotation.
- **SSH public-key authentication and host keys**: pinned client keys and pinned server keys, with no CA in between.
- **Tunnelling a protocol through a WebSocket to get past L7 proxies**: Chisel (SSH over WebSocket), Rancher `remotedialer`; Teleport's connection upgrade for L7 load balancers.
- **Rancher `remotedialer`** and **Konnectivity (apiserver-network-proxy)**: multiplexed reverse tunnels with dial callbacks.
- **kcp's per-shard virtual workspaces**: the deployment model for per-shard connectors.
- **Multi-shard APIExport consumers** (multicluster-runtime's endpoint-slice provider, as used by PM's own operators): how connectors read the connection export across shards.
- **cert-manager with KMS- or Vault-backed issuers**: short-lived workload certificates from a CA whose key stays in a KMS.
- **RFC 7515 `x5c` and RFC 7523 JWT assertions**: signed assertions carrying their certificate chain.
- **Account recovery with a waiting period**: common in consumer identity providers; recovery completes only if the current credential holder doesn't object in time.
- **Railgrid edges provider**: Lease-based tunnel routing; write-once SSH host-key pinning; the power-off budget of refresh-by-presenting-itself.
- **Open Cluster Management**: bootstrap kubeconfig, then CSR-based registration; cluster-proxy for the reverse tunnel.
