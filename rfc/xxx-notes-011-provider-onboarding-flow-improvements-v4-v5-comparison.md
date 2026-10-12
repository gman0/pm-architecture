# Notes: RFC 011 provider onboarding, v4 vs. v4-b vs. v5

Working notes for [011](./011-provider-onboarding-flow-improvements.md). Not the RFC itself. Compares the [v4 notes](./xxx-notes-011-provider-onboarding-flow-improvements-v4.md), the [v4-b notes](./xxx-notes-011-provider-onboarding-flow-improvements-v4-b.md) and the [v5 notes](./xxx-notes-011-provider-onboarding-flow-improvements-v5.md) side by side, for choosing between them. All three documents are standalone. Each version's own "Comparison" and "Changes from …" tables stay the reference for its details; this document only puts them next to each other and spells out the trade-offs.

**Numbering.** "Finding N" means v4's findings 1 to 13, which v4-b repeats under the same numbers. v4-b's findings 14 to 17 restate v5 material: 14 is v5's observation 19 (maximal permission policy), 15 is observation 18 (cache-server credentials), 16 is v5's per-shard virtual-workspace analysis, and 17 is observation 15 (tunnel limits). "v5 observation N" refers to v5's own numbering.

## In one paragraph each

- **v4** keeps v2's model, where the Provider-side holds kcp credentials and talks to kcp itself through a dial-proxy tunnel, and hardens it against everything outside the owner's workspace. The PM-side proves a **pinned per-connection key** in an **inner mTLS** handshake that runs inside an ordinary HTTPS WebSocket, so the Provider's ingress or CDN sees nothing but opaque bytes. Access tokens and kcp's CA bundle travel only inside the inner TLS.
- **v4-b** is v4 with v5's lessons applied, keeping v4's core (kcp credentials on the Provider-side, a dial-proxy tunnel, end-to-end TLS to kcp). A stateful **connection controller** holds every connection key and makes all control calls, including token delivery, over v4's inner mTLS. Stateless **per-shard connectors** only carry tunnels, which run on a plain WebSocket authenticated by short-lived **session assertions** from a pinned **connector CA**. It adds v5's connection export with a maximal permission policy, rebind, and egress hardening.
- **v5** removes kcp credentials from the Provider-side altogether. Per-shard **connectors** become authenticating HTTP reverse proxies that add the connection's SA token themselves. Since nothing at the Provider's edge works against kcp any more, v5 declares **the Provider's edge the Provider's responsibility** and drops the inner mTLS, server-key pinning and the pinned connection key. The connection controller makes control calls with a bearer **connection secret**; connectors authenticate sessions with session assertions, as in v4-b.

## At a glance

| | v4 | v4-b | v5 |
|---|---|---|---|
| Core move | Protect kcp credentials on their way to the Provider-side | Same, with keys moved off the data path | Never send kcp credentials to the Provider-side |
| Trust boundaries | Owner's workspace is the owner's responsibility | Same as v4 | Same, plus, with plain sessions: the Provider's edge is the Provider's responsibility. Inner TLS on sessions (proposed default) removes that assumption |
| kcp credentials on the Provider-side | SA access token (hours) + kcp CA bundle | Same as v4 | **None** |
| PM-side identity, control path | Pinned connection key, proven in inner mTLS | Same as v4 | Bearer connection secret (PM-generated; Provider stores its hash) |
| PM-side identity, tunnel or session path | Same pinned key | Session assertion, signed with a short-lived connector certificate from a pinned connector CA | Same as v4-b |
| Provider-side identity | Web PKI at registration, then a pinned server key | Same as v4 (control path); none needed on tunnels | Web PKI |
| What crosses the WebSocket | Inner TLS 1.3 → connection API, or the dial-proxy tunnel → end-to-end TLS to kcp | Control: inner TLS 1.3 → connection API. Tunnel: dial proxy → end-to-end TLS to kcp | Stream multiplexer → plain HTTP to the connector |
| TLS layers on kcp traffic | 3 (outer, inner, end to end to kcp) | 2 (outer, end to end to kcp) | Plain sessions: 2 (outer to the Provider, connector to kcp). Inner TLS on sessions: 3 |
| Who terminates the Provider-side's kcp requests | kcp (front-proxy, shard virtual workspaces) | Same as v4 | The connector, which re-issues them to kcp |
| PM-side components | One connector (placement open) | Connection controller (stateful, HA) + stateless per-shard connectors + connector CA + connection export | Same as v4-b, connectors built as kcp virtual workspaces |
| Sessions per connection | Two (API session, tunnel session) from one connector | Control sessions on demand from the controller; at most one tunnel per (connection, shard), only where there's traffic | At most one session per (connection, shard), only where there's traffic |
| Revoking kcp access | Token TTL (finding 2), unless `DELETE` arrives and the Provider discards the token | Token TTL (finding 2); tunnels close at once | Immediate: connectors stop proxying |
| Recovery after a lost PM-side credential | Restore from a recent backup, otherwise a new connection with no state | Rebind with a waiting period; same `connectionID`, state kept | Same as v4-b |
| Main cost | Connector holds every key; complexity on both sides; placement open | Two authentication schemes on the Provider-side; a connector CA to run; controller on the token path | A second, lighter virtual-workspace hop for remote Providers' traffic; a connector CA to run; with plain sessions, the edge holds the Provider's end of every session, with inner TLS, pinning machinery on the session path |

## Architecture

```
v4
connector ══ outer TLS (terminated at the Provider's edge) ══▶ ingress ──▶ connection server
          ╚═ WebSocket ═══════════════════════════════════════════════════╝
               ╚═ inner TLS 1.3: pinned connection key ⇄ pinned server key ═╝
                    ╚═ connection API (PUT /credentials: token + kcp CA bundle, PUT /key, …)
                    ╚═ tunnel (dial proxy) ═▶ Provider-side's own TLS to kcp, with its token

v4-b
connection controller ══ outer TLS ══▶ ingress ──▶ connection server
                      ╚═ WebSocket /control ═╗
                           ╚═ inner TLS 1.3: pinned connection key ⇄ pinned server key
                                ╚═ connection API (credentials, key, connector CAs, status, delete)
connector@shard ══ outer TLS ══▶ ingress ──▶ connection server
                ╚═ WebSocket /tunnel, Bearer <session assertion> ═╗
                     ╚═ dial proxy ═▶ Provider-side's own TLS to kcp, with its token

v5
Provider-side client-go ── dial hook: open a stream on the session for that host
       ▲ one WebSocket per (connection, shard), opened by the connector, multiplexer inside
connector@shard ── HTTP server: authn → authz (denylist) → audit → strip creds → add SA token ── TLS ──▶ kcp
connection controller ── HTTPS control calls (bearer connection secret) ──▶ Provider-side
```

- **v4's connector is a byte pipe** that also holds every connection key and makes every control call. Where it runs (and whether it's per shard) is an open item.
- **v4-b keeps the byte pipe and moves the keys out.** Connectors forward opaque streams and hold only their connector certificate; everything secret sits in the controller. Connectors are plain per-shard StatefulSets, not virtual workspaces: nothing on their path is request-shaped.
- **v5's connector is an application-level proxy.** It sees every request, enforces policy on it, audits it and attaches the identity. It's built as a kcp virtual workspace, one per shard, deployed through kcp-operator's `VirtualWorkspace` with `shardRef`.
- **v4-b and v5 separate control and data.** In v5 the controller is entirely off the traffic path; in v4-b it still delivers access tokens (see "Failure and recovery").

## Trust model

| Question | v4 | v4-b | v5 |
|---|---|---|---|
| Who is trusted with a connection's identity? | Whoever can read the connection Secret (owner's workspace) | Same, for control calls. Tunnels need a connector certificate or the connector CA (PM's control plane) | Same as v4-b |
| Is the Provider's edge trusted? | No. Inner mTLS and server-key pinning defend against it | No, for anything that works against kcp. On tunnels it sees dial targets and timing, and can inject dials that reach the allow-listed kcp hosts anonymously | Plain sessions: yes, as the Provider already trusts it with all its customers' traffic. Inner TLS on sessions (proposed default): no (see "The Provider's edge") |
| How does the Provider-side know it's talking to the real kcp? | kcp CA bundle, delivered by the connector, verified end to end | kcp CA bundle, delivered by the controller over the control channel, verified end to end | It doesn't separately: in tunnel mode, trusting kcp is trusting the connection (v5 observation 12) |
| Can an outsider (stolen token, portal account, edge) present a fake kcp? | No: needs the connection key | No: needs the connection key. A tunnel holder can't either | No: needs the connection secret, a connector certificate or the connector CA |
| Can the Provider harm PM from its own workspace? | Not addressed: the Provider holds an SA token with admin in its workspace, used directly against kcp | Records are read-only (maximal permission policy, enforced by kcp). Deleting the binding or forging endpoint-slice status can't be denied, but only hurts the Provider or opens idle tunnels | No: maximal permission policy on records, plus the connector's denylist for the binding, endpoint-slice status, token and CSR minting |

**Anchors per trust question** (v4's "Choosing the PM-side anchor", v4-b's "Choosing the anchors", v5's "Session credentials"):

| Who verifies whom | v4 | v4-b | v5 |
|---|---|---|---|
| Provider-side → PM-side, control calls | Pinned connection key, inner mTLS | Same as v4 | Hash of the bearer connection secret |
| Provider-side → PM-side, tunnels or sessions | Same pinned key | Session assertion chaining to the pinned connector CA | Same as v4-b |
| PM-side → Provider-side | Web PKI at registration, then the pinned server key | Same as v4, on control sessions; not needed on tunnels | Web PKI |
| Provider-side → kcp | Delivered kcp CA bundle, changeable only with the connection key | Same as v4 | Not separately: the Provider-side trusts responses on an authenticated session |

v4-b's case for a connector CA on tunnels is stronger than v5's for its sessions: a v4-b tunnel holder can't read or fake kcp traffic, while a v5 session holder can serve fake data. Both keep the CA away from control calls.

v4 and v4-b protect a credential from the Provider's edge; v5 stops sending it. With plain sessions, though, the edge holds the Provider's end of every session: it can act as the Provider in kcp while sessions last, fake kcp's responses, and read reconcile traffic, which is largely PM consumers' data. It keeps nothing usable once the sessions close. v5's proposed inner TLS on sessions removes all of that (see "The Provider's edge"). v4-b's one new edge exposure is network reach: dials injected into an unencrypted tunnel reach kcp without credentials, which matters only where kcp's hosts aren't already reachable from the internet (an open item in v4-b).

## Credentials

| Credential | v4 | v4-b | v5 |
|---|---|---|---|
| Onboarding token | `pm1.<url>.<id>.<secret>`, single-use | Unchanged | Unchanged |
| Long-lived PM-side credential | Connection **private key** in the owner's workspace; rotated by the connector via `PUT /key` | Same key, read and rotated only by the controller | Connection **secret** (`pmc1_…`, 256 bits); rotated by the controller via `PUT /secret`. Prefixed so secret scanners catch it |
| Short-lived PM-side credential | Self-signed connection certificate (hours), created on demand | Connection certificate on control sessions; session assertion (≤ 5 min, audience-bound, `jti`) with a connector certificate (hours) on tunnels | Session assertion with a connector certificate |
| PM-side CA | None, deliberately (v4 rejected a shared PM CA as the anchor) | Connector CA, immutable generations, pinned by certificate. Authorizes tunnels only: no control calls, no kcp credentials, no readable kcp traffic | Connector CA, same form. Authorizes sessions: no control calls or kcp access, but a forged session can serve fake data |
| Provider-side key | Server key, pinned, rotated with overlap and refetch | Same as v4 (control sessions only) | None |
| kcp SA token | Minted by the connector, `PUT` to the Provider-side; optional self-rotation | Minted by the controller, `PUT` over a control session; self-rotation on by default after the claim | Minted by the connector, kept in its memory, never sent |
| kcp CA bundle | Sent with every `PUT /credentials`, from PM's own trust | Same as v4, sent by the controller | Used by the connector only |

**Where the high-value secrets live:**
- **v4**: the connector reads every connection's private key. A compromised connector or its node exposes all connections at once (v4's top threat).
- **v4-b**: the controller holds every connection key and the connector CA. A compromised connector yields one connector certificate for hours: denial of service and traffic analysis on that shard's tunnels, nothing more.
- **v5**: the controller holds every connection secret and the connector CA. A compromised connector exposes its shard's sessions until its certificate expires, including plaintext reconcile traffic and the ability to serve fake data, but has no persistent secret.

**Threats compared** (v4 and v4-b rank theirs explicitly; v5 describes what moved):

| Path | v4 | v4-b | v5 |
|---|---|---|---|
| Highest-impact target | The connector or its node: every connection's key (ranked 1) | The controller or its node: every connection's key and the connector CA (ranked 1) | The control plane: every connection secret and the connector CA |
| A compromised connector | Same as above | One connector certificate for hours: denial of service and dial metadata (ranked 5) | One shard's sessions for hours: plaintext traffic, fake data |
| The owner's workspace | One connection per leaked key (ranked 2) | Same (ranked 2) | One connection per leaked secret; prefixed secrets are caught by scanners |
| A compromised portal account | New connections only (ranked 3) | New connections, plus rebinds that get cancelled (ranked 3) | Same as v4-b |
| Server key plus the Provider's edge | Impersonate the Provider-side, receive access tokens (ranked 4) | Same, on control sessions (ranked 4) | Not applicable: no server key, and the edge is trusted |
| A compromised Provider-side | A usable kcp token until it expires | Same as v4 | Nothing usable against kcp once its sessions close |

## Flow

| Step | v4 | v4-b | v5 |
|---|---|---|---|
| Register | `POST` with token, addressing and the connection key's public key; get `connectionID` and `serverKeys` | Same, plus the connector CA certificates | `POST` with token, addressing, the secret's hash and the connector CA certificate; get `connectionID` |
| After registration | Connector opens a session, `PUT`s the access token and kcp CA bundle, opens the tunnel | Controller writes the `ProviderConnection` record, then `PUT`s token and CA bundle over a control session; connectors discover the record and open tunnels where needed | Controller writes the record; connectors discover it and open sessions where needed |
| Readiness | Provider-side reports `kcpReachable` | Provider-side reports per required host whether a tunnel covers it and reaches kcp | Same as v4-b |
| Steady state | Connector `PUT`s a fresh token and CA bundle at half the TTL and after every reconnect | Controller `PUT`s at half the TTL; the Provider-side also rotates its own token | Nothing reaches the Provider-side; connectors refresh tokens in memory |
| Credential rotation | Key via `PUT /key` and a handshake with the new key (tunnel reconnects); certificates free | Same, by the controller; tunnels unaffected. Connector certificates by cert-manager; connector CA rolled over rarely via `PUT /connector-ca-bundle` | Secret via `PUT /secret`; connector certificates by cert-manager; connector CA rolled over rarely |
| Offboard | `DELETE` in a session; kcp access lasts until the token expires | Record deleted first, so tunnels close; then `DELETE`; kcp access lasts until the token expires | Record deleted first, so sessions close and kcp access ends; then `DELETE` |

## Data path and load

The load on kcp is the same in all three: the same requests reach the same targets (front-proxy for workspace traffic, each shard's APIExport virtual workspace for endpoint traffic) with the same SA identity. What differs is the hop that every NAT-traversing design adds.

| | v4 | v4-b | v5 |
|---|---|---|---|
| What the hop does | L4: copy opaque bytes between the tunnel and a TCP connection to kcp | Same as v4 | L7: parse HTTP, authenticate, authorize, audit, inject the token, copy the response |
| TLS work on the PM-side hop | Outer and inner TLS per session; the Provider-side's TLS to kcp passes through untouched | Outer TLS per tunnel; the Provider-side's TLS to kcp passes through untouched | Outer TLS per session; originates TLS to kcp, with pooled connections |
| Per watch | One tunnel stream | One tunnel stream | Two streams on the connector (multiplexer side and kcp side), a few goroutines and a 32 KB copy buffer each |
| Connections to kcp | One or more per Provider-side, opened through the tunnel | Same as v4 | Pooled from the connectors |
| Scaling | One connector for everything (placement open) | One connector per shard | One connector per shard, scaling like kcp's virtual workspaces |
| Failure isolation | A tunnel drop loses every watch of that connection | A tunnel drop loses one shard's watches; front-proxy traffic moves to another tunnel | Same as v4-b |
| Request policy and audit at the hop | None possible (opaque bytes) | None possible; per-connection stream and bandwidth limits only | Full: denylist, rate limits, per-connection audit with the connection principal and SA |
| Timeouts and limits | The tunnel library's | Same as v4 | The generic handler chain's; long-running and timeout settings must be configured for proxied LISTs and watches, and the `X-Kcp-Virtual-Resource-Hops` budget checked |

**All three against the standard flow.** Today a Provider runs wildcard LISTs and WATCHes per resource directly against each shard's APIExport virtual workspace. That virtual workspace is already an L7 hop: a full handler chain, a SubjectAccessReview for `apiexports/content`, and for every watch its own watch on the shard, with each event decoded into `Unstructured` and re-encoded (`kcp/pkg/virtual/apiexport/authorizer/content.go`, `virtual-workspace-framework/pkg/forwardingregistry/store.go`). None of that changes in any version. What each adds in front of it:

| | Standard flow | v4 | v4-b | v5 |
|---|---|---|---|---|
| Extra hop before the virtual workspace or front-proxy | none | one L4 connector for all shards | one L4 connector per shard | one L7 connector per shard |
| Work per watch on the extra hop | — | byte copy | byte copy | local authentication and policy checks, an audit entry, header rewrites, byte copy; no object decoding |
| Connections into PM | inbound from each Provider | none inbound | none inbound | none inbound |
| Watches carried by one connector | — | all of them | its shard's | its shard's, the same as its virtual workspace serves to remote Providers |

Measured this way, v5 is **two virtual workspaces in a chain**: the `providerconnections` virtual workspace is a client of the APIExport virtual workspace, the way the Provider is today. It carries the same watches, scales one-to-one with it, and does less per watch. The load is the same in kind and count; v5 adds a second, lighter copy of the per-watch state the APIExport virtual workspace already holds.

What's really different is small: a slow Provider has to be throttled through the multiplexer's per-stream flow control instead of TCP and HTTP/2 (a matter of choosing the library), the long-lived connections are opened outbound instead of inbound, and the second hop needs its timeouts and hop budget configured. v4-b's L4 hop is cheaper still, and gets the same per-shard scaling and isolation.

## Provider-side

| | v4 | v4-b | v5 |
|---|---|---|---|
| Infrastructure | HTTPS endpoint with WebSocket support | Same | Same |
| Keys and CAs of its own | A server key in a Secret | Same as v4 | None |
| Software (from the SDK) | WebSocket listener with an mTLS server inside; pinning callbacks on both sides; token and CA bundle handling; client rebuild on change; dial-proxy server | v4's control-path mTLS and pinning, plus assertion and x5c chain verification for tunnels; token and CA bundle handling; dial-proxy server routing dials by host | Bearer check; assertion and x5c chain verification against pinned connector CAs (with the `KeyUsages` and chain-length details v5 calls out); multiplexer server; a dial hook for client-go's standard transport |
| kcp knowledge | Uses the token and CA bundle | Same as v4 | None |
| Client config per connection | `Host`, token, `CAData`, the tunnel's `Dial`; rebuilt or updated when the token or bundle changes | Same, with `Dial` picking a tunnel by target host | `Host` plus `Transport: sdk.Transport(conn.ID)`; never changes |
| Multi-replica | Lease routing as in v2 | More pressing: tunnels for different shards can land on different replicas | Same as v4-b |
| What a compromise leaves behind | A usable kcp token until it expires | Same as v4 | Nothing: no kcp credential to exfiltrate |

## PM-side operations

| | v4 | v4-b | v5 |
|---|---|---|---|
| Components | Connector | Connection controller, per-shard connector StatefulSets, a connector CA, a connection export (APIExport with a maximal permission policy) and its binding in every provider workspace | Same as v4-b, connectors as kcp-operator `VirtualWorkspace`s |
| State | Connection Secrets (keys) read by the connector | Connection Secrets read by the controller only; records in provider workspaces; connectors stateless | Same as v4-b |
| Adding a shard | Nothing: the tunnel's allow-list follows `Shard` objects | Deploy its connector and connector certificate; nothing registered anywhere | Same as v4-b |
| Rotating kcp's CAs | Next `PUT` carries the new bundle | Same as v4 | Nothing to do: connectors use PM's own trust |
| Owner sees PM's shard layout | No | No | No |
| kcp RBAC changes | Provider controller's RFC 006 RBAC | Same, plus the controller's `create` on `serviceaccounts/token` for each SA, the Provider's own for self-rotation, and the export's maximal-permission-policy RBAC | Same as v4, plus connectors' `create` on `serviceaccounts/token` for each SA and the export's maximal-permission-policy RBAC |
| New operational duties | Connection key backups (owner's choice), server-key overlap | Running the connector CA (immutable generations, `rotationPolicy: Never`, rollover schedule, expiry alerts); controller HA, since it delivers tokens; per-shard connector health; server-key overlap | Running the connector CA; per-shard connector health; cache-server credentials kcp-operator mounts by default (v5 observation 18) |

## Failure and recovery

| Scenario | v4 | v4-b | v5 |
|---|---|---|---|
| PM-side component restarts | Connector creates a certificate, reconnects, `PUT`s credentials, reopens the tunnel | Connector replica reconnects with its mounted certificate; no control-plane or Provider call | Same as v4-b |
| Control-plane outage | Same component as the data path | Tunnels keep working; token delivery stops, so traffic lasts until the last token expires, or indefinitely with self-rotation | Traffic continues; onboarding, rotations, rebinds and offboarding wait |
| Outage of any length | Nothing PM needs expires | Same; the controller's next `PUT` restores an expired token | Same |
| Lost connection Secret | Restore a backup taken after the last rotation, else new connection, state lost | Rebind after a waiting period; cancelled automatically if the current key still exists | Same, with a secret instead of a key |
| Stolen PM-side credential, thief rotates | Owner notified; freezes or deletes in the portal | Same; the thief can also cancel the owner's rebinds | Same as v4-b |
| Stolen onboarding token or portal account | New connections only, billed to the owner | Same, plus a rebind that the controller cancels | Same as v4-b |
| Compromised shard | Not applicable (one connector) | Stop issuing to it; its certificate expires within hours | Same as v4-b |
| Wrong kcp CA bundle | Provider-side can't reach kcp; fixed by the next `PUT` | Same as v4 | Not possible on the Provider-side |
| Provider deletes the connection export's binding | Not applicable | Its tunnels close; the binding and record are recreated | Denied by the connector |
| A record points at the wrong connection (PM-side bug) | Not applicable (no records) | The assertion's `cluster` claim doesn't match the `clusterID` stored at registration; the Provider-side refuses the tunnel | Same as v4-b; also keeps a connector from using another workspace's SA |
| Provider-side unreachable at deletion | Finalizer gives up on `DELETE` after a timeout; kcp access ends with the token | Same as v4, but tunnels are already closed | Sessions are already closed; kcp access has ended |

## The Provider's edge

Leaving token lifetime and revocation aside, the versions differ in **where the authority to act as the Provider in kcp lives**, and therefore whether the Provider's inbound edge (ingress, CDN, WAF) can use it. Today and in v4-b, the authority is a credential at the Provider's runtime, behind TLS that runs end to end to kcp. In v5 it's the session itself, and with plain sessions the session's TLS ends at the edge. v5 now proposes inner TLS on sessions, on by default (v5, "The Provider's edge: inner TLS on sessions"). `══` marks encrypted legs, `──` plain ones.

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
v5, PLAIN SESSIONS (the edge is inside the Provider's trust boundary)

  PM                                        │ internet │   Provider
                                            │          │  ┌───────────────── trust boundary ─────────────────┐
  kcp ◀── APIExport VW ◀══TLS══ connector ══╪═ outer TLS (WebSocket) ══▶ ingress/CDN ──▶ SDK ──▶ controllers │
                              adds SA token │          │  │               outer TLS ends                       │
                                            │          │  │               ◀── multiplexer + plain HTTP ──▶     │
                                            │          │  │               readable and writable here          │
                                            │          │  └────────────────────────────────────────────────────┘

  acts as the Provider in kcp:    the runtime, or the ingress/CDN (by opening its own streams)
  can fake kcp to the Provider:   the ingress/CDN
  sees the connection secret:     the ingress/CDN (control calls are plain HTTPS)
```

```
v5, INNER TLS ON SESSIONS (proposed default; the edge is outside the trust boundary)

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
  sees the connection secret:     nobody (control calls move inside the inner TLS too)
```

```
v4-b: kcp credentials on the Provider-side; the edge is outside the trust boundary

  PM                                        │ internet │   Provider
                                            │          │                           ┌──── trust boundary ────┐
  control path                              │          │                           │                         │
  connection controller ════════════════════╪═ outer TLS (WebSocket /control) ══▶ ingress/CDN ──▶ SDK        │
    mints SA token                          │          │            outer TLS ends │                         │
                     ══ inner TLS 1.3 ══════╪══════════════════════════════════════════▶ SDK                  │
                        client: pinned connection key  │                           │   stores: SA token,     │
                        server: pinned server key      │                           │   kcp CA bundle         │
                          ╚═ PUT /credentials: SA token + kcp CA bundle            │                         │
                                            │          │                           │                         │
  tunnel path (one per shard)               │          │                           │                         │
  kcp ◀── VW / front-proxy ◀── TCP ── connector ═══════╪═ outer TLS (WebSocket /tunnel, assertion) ══▶ ingress/CDN ──▶ SDK ──▶ controllers
                              (byte pipe)   │          │            outer TLS ends │   ▲                     │
                                            │          │  ◀── multiplexer: dial targets visible ──▶          │
                                            │          │                           │   │                     │
  kcp ◀══════════════ TLS, end to end, with the SA token, inside the tunnel ═══════════╧═ controllers        │
                                            │          │                           └─────────────────────────┘

  ingress/CDN sees:               control path: opaque bytes; tunnels: dial targets and timing, opaque kcp TLS
  acts as the Provider in kcp:    the runtime only (the token never crosses the edge readable)
  can fake kcp to the Provider:   nobody without the connection key (the CA bundle is delivered on the control path)
  can reach kcp anonymously:      the ingress/CDN, by injecting dials into a tunnel (an open item in v4-b)
```

| | Today (RFC 006) | v4 | v4-b | v5, plain sessions | v5, inner TLS on sessions |
|---|---|---|---|---|---|
| Inbound edge on the path | No | Yes, opaque | Yes: opaque control path, tunnel metadata visible | Yes, readable | Yes, opaque |
| Edge can act as the Provider in kcp | No | No | No | **Yes**, while sessions last | No |
| Edge can fake kcp to the Provider | No | No | No | **Yes** | No |
| Edge sees PM consumers' data | No | No | No | **Yes** | No |
| Edge can lock the owner out | — | No | No | **Yes** (connection secret) | No |
| Edge gets anonymous reach to kcp | No | No | **Yes** (injected dials) | No | No |
| kcp credential on the Provider-side | Yes | Yes | Yes | No | No |

**On principles, the choice is a two-by-two:**

| | Edge inside the Provider's trust boundary | Edge outside it |
|---|---|---|
| kcp credentials on the Provider-side | — | v4, v4-b |
| No kcp credentials on the Provider-side | v5, plain sessions | **v5, inner TLS on sessions** |

v5 with inner TLS keeps the edge out as v4-b does, without the injected-dials gap, and keeps v5's credential, revocation and policy properties. Its extra cost over plain sessions is v4's pinning machinery on the session path (a server key, its rotation, the verification helper), without any token or CA-bundle delivery.

**Why the edge matters**, even though most Providers trust theirs: it's often someone else's (a third-party CDN or WAF, a platform team's shared ingress), edges log and inspect traffic without anyone being malicious, some regulated Providers must keep data encrypted end to end, and the reconcile traffic is largely PM consumers' data, which today never crosses the Provider's inbound edge. Plain sessions would make "the edge is trusted" an assumption PM places on every Provider without being able to check it.

## What each gives up

**v4 over v4-b and v5:**
- **The Provider's edge sees nothing**, not even which kcp hosts are dialled, and can't inject dials. v4-b exposes tunnel metadata and anonymous reach; v5 with plain sessions relies on the Provider trusting its own edge, and v5 with inner TLS on sessions matches v4 here.
- **No CA anywhere.** Every credential concerns exactly one connection.
- **Fewest PM-side moving parts.** No connector CA, no connection export, no per-shard deployment.
- **One authentication scheme** on the Provider-side.

**v4-b over v4:**
- **No connection keys on the data path.** v4's top threat becomes a minor one.
- **One TLS layer fewer** on kcp traffic, and key rotations no longer reconnect tunnels.
- **Per-shard connectors**, with locality, failure isolation and tiny allow-lists, settling v4's placement question.
- **Rebind** without losing state, so backups become optional.
- **Read-only connection records** for the Provider, and v5's egress hardening.

**v4-b over v5:**
- **Compared with plain sessions only:** the edge can't read reconcile traffic or the PM-side credential, can't act as the Provider in kcp, and a forged tunnel can't fake kcp. v5 with inner TLS on sessions closes all of these, without v4-b's injected-dials gap.
- **A cheaper L4 hop**, instead of v5's second virtual workspace with its timeouts, hop budget and multiplexer backpressure.
- **Connectors are plain forwarders**, not virtual workspaces, and need no TokenRequest rights or cache-server credentials.

**v5 over v4 and v4-b:**
- **No kcp credential ever leaves the PM-side.** Revocation is immediate, finding 2 stops mattering for remote Providers, and a compromised Provider-side leaves nothing usable against kcp.
- **PM controls what Providers can do in kcp**, with a connector denylist and per-request audit at the hop.
- **Less Provider-side machinery.** No token or CA bundle handling and no kcp knowledge. With plain sessions also no pinning, inner TLS or server key; with inner TLS on sessions, those come back, as one scheme instead of v4-b's two.
- **The controller is entirely off the traffic path.** A control-plane outage never affects traffic.

## Open items that could change the choice

- **v5: multiplexer backpressure.** The second virtual-workspace hop carries the same watches as the APIExport virtual workspace with less work per watch, so scale isn't the question. Whether the chosen multiplexer throttles a slow Provider per stream is. If no suitable multiplexer does, v4-b is the fallback that keeps most of v5's structure.
- **v5: the connector as a virtual workspace.** Timeouts for proxied LISTs and watches, the virtual-resource hop budget, and the cache-server credentials kcp-operator mounts.
- **v4-b: edge-injected dials.** Whether kcp's front-proxy and virtual-workspace hosts are reachable from the internet in PM's deployments, and what they serve anonymously. If they aren't, v4-b needs inner mTLS back on tunnels or a connector check for a TLS ClientHello with an allow-listed SNI.
- **v4-b: controller on the token path.** Token TTL, self-rotation by default and controller HA decide how long traffic survives a controller outage.
- **v4-b and v5: maximal permission policy for the controller's writes.** Reading the code says a prefixed-user grant is enough; no kcp e2e test covers writes through the APIExport virtual workspace to an export with a policy.
- **v5: inner TLS on sessions, a TSC decision.** Proposed on by default; the alternatives are required, or opt-in per connection. It decides whether the Provider's edge is inside the trust boundary (see "The Provider's edge").
- **v4-b and v5: signed control calls.** In v5, only relevant with plain sessions: with inner TLS on sessions, control calls run inside it and the edge never sees the connection secret. In v4-b, signed requests on plain HTTPS could replace the control-path inner mTLS and leave one authentication style on the Provider-side, but access tokens would then need payload encryption, since they mustn't cross the edge readable.
- **All: `direct` mode.** In v4 and v4-b the Provider-side already holds kcp credentials, so "direct" just means dialling kcp's URLs without tunnels, where PM's kcp is reachable. In v5 it means the Provider-side calling the connectors directly, which needs a credential in the other direction that v5 doesn't have yet.
- **All: Provider-side multi-replica.** Per-shard tunnels or sessions make it more pressing in v4-b and v5.
- **v4 and v4-b: kcp-operator's `--service-account-lookup=false`** (finding 2). Fixing it would bring SA deletion back as revocation, within about a minute, closing most of the revocation gap to v5. v5 doesn't depend on it.
