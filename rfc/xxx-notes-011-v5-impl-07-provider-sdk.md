# v5 implementation plan: Provider SDK

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). The library every Provider uses: the Go module `go.platform-mesh.io/provider-sdk` at `platform-mesh/provider-sdk`, next to `apis`, `subroutines` and `golang-commons`, implementing the [protocol](./xxx-notes-011-v5-impl-01-protocol.md). It implements the Provider-side of the protocol, so Providers don't need to understand kcp, TLS pinning or the multiplexer. The [quickstart](./xxx-notes-011-v5-impl-08-provider-quickstart.md) is its reference user.

## What a Provider gets

- **A connection server** to mount under its base URL: registration, control sessions, sessions.
- **Connections for its controllers**: for each connection, a client config whose requests travel over that connection's sessions, and events when connections come and go.
- **Portal primitives**: issuing onboarding tokens, listing and deleting connections, and hooks reporting what happened.
- **A storage interface** with a Kubernetes reference store.

What a Provider still does itself: its portal and user accounts, its operator code (which endpoint slices it reconciles), its ingress, and where it stores state if it doesn't use the reference store.

## Principles

- **No kcp knowledge on the Provider-side.** No tokens, kubeconfigs or kcp CA bundles; requests carry no credentials ([01 Protocol](./xxx-notes-011-v5-impl-01-protocol.md)).
- **Everything security-relevant is in the SDK**, written once and tested with the protocol's test vectors, so Providers can't get it subtly wrong.
- **State belongs to its connection** (`connectionID`), never inherited by another connection. Consumer data has to be keyed with the `connectionID` too, not only with the consumer's logical cluster and the object's UID: anyone with a portal account can register a connection with a connector CA of their own and serve made-up objects that reuse a real consumer's cluster and UID (see "Requirements on Providers").
- **`connectionID`s are stable.** A connection keeps its ID through secret changes and recovery, so state keyed by it stays valid.
- **`connectionID`s are random and unguessable** (e.g. 128 random bits), so probing the session and control paths can't find valid ones. They aren't secret, and nothing relies on them being so.
- **Only PM changes a connection's secret**, through the connection's `ProviderEndpoint` ([01 Protocol, "Changing the secret"](./xxx-notes-011-v5-impl-01-protocol.md)).
- **Secrets are stored hashed**; comparisons are constant-time; nothing secret is logged.

## Components

### Onboarding tokens and portal primitives

- **Issue tokens** bound to a portal owner account: `pm1.<base URL>.<id>.<secret>`, single-use, short expiry; store only the hash.
- **List and delete** a portal owner's connections. Deleting is final.

### Hooks

The SDK tells the Provider's code what happened, so the Provider can inform the owning customer, audit, or alert its operators. A struct of optional, typed functions, one per event, passed in when the connection server is set up:

```go
type Hooks struct {
    OnRegistered                func(ctx context.Context, conn Connection)
    OnConnectionDeleted         func(ctx context.Context, conn Connection, by DeletedBy)
    OnConnectionExpired         func(ctx context.Context, conn Connection)
    OnSecretChanged             func(ctx context.Context, conn Connection)
    OnConnectorCABundleChanged  func(ctx context.Context, conn Connection, old, new []Fingerprint)
    OnConnectorCABundleRejected func(ctx context.Context, conn Connection, reason RejectReason)
    OnSessionsClosed            func(ctx context.Context, conn Connection, count int)
    OnAdministrativeChange      func(ctx context.Context, conn Connection, change AdminChange)
    OnAuthenticationFailures    func(ctx context.Context, conn Connection, count int)
}
```

- **Called directly** when the event happens, after the change is committed, so a hook can't change the outcome.
- **The implementor keeps hooks fast**, or hands the work off to run asynchronously, and retries whatever it does if it needs to. The SDK has no queue and no retries for hooks.
- **A hook left `nil`** means the Provider doesn't handle that event. The SDK logs every event either way, never with secrets.
- **New events in later SDK releases are new fields**, so adding one never breaks a Provider's code.
- **Authentication failures** are counted by the SDK and reported once they pass a threshold per connection.

The quickstart's hooks log events and list them in its portal ([08 Quickstart](./xxx-notes-011-v5-impl-08-provider-quickstart.md)).

### Connection API

The handler for the one plain-HTTPS endpoint:
- **Registration**: verify and spend the token, apply idempotency rules (retries with the same idempotency key and hash return the same response for 7 days by default), create the connection entry (owner, `clusterName`, front-proxy URL, secret hash, the pinned connector CA bundle, negotiated version), return `connectionID` and server keys.
- **Orphan expiry**: a connection that has seen neither a control call nor a session by the end of the retry window is removed; it's a registration whose result PM never stored. A connection that was used once is never expired for being idle, however long its Provider or PM is away.

### Control sessions

- **WebSocket upgrade, then the inner TLS server** with the server key the connection's `ProviderEndpoint` lists (see "Server keys"), speaking HTTP/1.1 with keep-alive.
- **Bearer check** on each request, against the current secret hash only. **The first failed check closes the control session**, so every guess costs a new WebSocket and handshake.
- **Handlers**: status, connector CA bundle replacement, delete ([01 Protocol, "Control sessions"](./xxx-notes-011-v5-impl-01-protocol.md)). None changes the secret.
- **Connector CA bundle**: parsed as a SPIFFE-format trust bundle (JWKS with `x5c`); every certificate is checked to be a CA before it's pinned. Replacement is compare-and-swap on the pinned bundle's `etag`: a missing or stale `If-Match` is refused with `412`, an identical body is a no-op ([01 Protocol, "Connector CA bundle"](./xxx-notes-011-v5-impl-01-protocol.md)). The `etag` and fingerprints are reported in `GET`. The bundle is only ever accepted on the control channel or at registration, never from a session. A push is accepted even if every pinned CA has expired: it's authenticated by the secret, not by the pinned CAs.

### The connection's ProviderEndpoint

For each connection, the SDK watches its `ProviderEndpoint` (`default`, cluster-scoped, in the connection's provider workspace, through the front-proxy) over the connection's own sessions. Two things come from it:
- **`pendingSecretHash`**: when set and different from the current hash, it becomes the current hash, atomically in the store, and the old secret stops working ([01 Protocol, "Changing the secret"](./xxx-notes-011-v5-impl-01-protocol.md)). This is the only way the hash changes after registration.
- **`serverKeys`**: which server keys PM has pinned, so the SDK presents a new key on this connection only once it's listed (see "Server keys").

The `ProviderEndpoint` is trusted because of how it arrives: over a session whose connector certificate verified against the connection's pinned bundle and whose hello named the connection's `clusterName`, from an object only PM can write.

### Session server

1. **Refuse unknown or deleted connection IDs before the upgrade**, and callers beyond the handshake limits with `503` ([01 Protocol, "Endpoints at a glance"](./xxx-notes-011-v5-impl-01-protocol.md)).
2. **Accept the WebSocket** and run the **inner TLS server**, requiring a client certificate issued directly by a CA in the connection's pinned bundle, with the `clientAuth` EKU ([01 Protocol, "Inner layer"](./xxx-notes-011-v5-impl-01-protocol.md)). A short timeout covers the whole handshake.
3. **Read the hello**; close the session if its `clusterName` isn't the connection's.
4. **Start yamux**; register the session under its connection, with the hosts it serves (from the hello's `hosts`) plus the front-proxy.
5. **Honour go-away**: stop opening streams on a session the connector is replacing.
6. **Close all sessions** of a connection that's deleted.
7. **Re-check open sessions when the pinned bundle changes** (a push or a re-pin): a session whose client certificate, kept from its handshake, no longer chains to a CA in the new bundle gets go-away, a short grace period for in-flight requests (e.g. 30 seconds, configurable), then is closed. Only the issuer counts, not the certificate's expiry. This ends sessions opened with a leaked CA or connector certificate key once PM drops it; why it matters, and its limits, are in [01 Protocol, "Connector CA bundle"](./xxx-notes-011-v5-impl-01-protocol.md).

### Transport

- **A per-connection `http.Transport`** whose dial hook opens a yamux stream on a session that serves the target host and returns it. The target address only chooses the session; the request goes as plain HTTP/1.1 on the stream, with the real kcp URL and `Host`, which the connector checks.
- **Session choice**: any session for the front-proxy; that shard's sessions for a virtual-workspace host; spread across several sessions serving the same host.
- **No TLS settings on the client config**: confidentiality comes from the inner TLS, and client-go refuses both a custom transport and TLS options.
- **Per-host bookkeeping** of request outcomes, which feeds status.

### Connections for multicluster-runtime

- **A registry of connections** with events (added, removed) and, per connection, a client config: front-proxy URL as host plus the transport.
- **A helper** that, for each connection, creates a multicluster-runtime endpoint-slice provider per endpoint slice the Provider names, and composes them with cluster names prefixed by `connectionID`.
- **Connections keep their `connectionID`** through secret changes, repairs and administrator imports, so nothing changes for the Provider's controllers.

### Status

Answers `GET` with, per host the Provider's clients actually use: live sessions, whether recent requests reached kcp, and the last error, without sensitive detail ([01 Protocol, "Status"](./xxx-notes-011-v5-impl-01-protocol.md)).

### Server keys

- **Generated by the SDK** on first start, stored with the reference store or the Provider's own.
- **Rotation only on demand**, through an administration command (see "Administration"), for example after a suspected key compromise. Nothing rotates on a schedule.
- **Rotation with overlap**: announce the new key in `GET`; switch each connection to it once that connection's `ProviderEndpoint` lists it ([03 Operator, "Writing ProviderEndpoints"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)); retire the old key once every connection's `ProviderEndpoint` lists the new one, or at the deadline given to the command. A long deadline (e.g. 7 days) gives every connection time to catch up; after a suspected compromise, a short one accepts that connections not reached in time lose their sessions until a PM administrator supplies the new key. There's no plain-HTTPS key endpoint ([01 Protocol, "Server keys"](./xxx-notes-011-v5-impl-01-protocol.md)).

### WebSocket framing

The SDK enforces the protocol's framing on every session and control session ([01 Protocol, "WebSocket framing"](./xxx-notes-011-v5-impl-01-protocol.md)): binary messages only, at most 32 KB each, outer pings at the protocol's interval. Providers can't configure it away.

### Administration

Commands for the Provider's administrators, for cases no protocol call can fix safely. Their inputs are exchanged out-of-band with the people running the PM installation; the commands are tooling and can come after the first release, but the store and the connection server support them from the start:
- **Re-pin a connection's connector CA bundle**, when no session verifies (for example, a bundle that missed a whole rollover while the connection also lost its secret). Once sessions work again, PM repairs the secret through the `ProviderEndpoint`.
- **Rotate the server keys**, with a deadline (see "Server keys").
- **Import a connection entry** after the Provider-side lost its store: `connectionID`, owner, `clusterName`, front-proxy URL and the pinned connector CA bundle. No secret hash: platform-mesh-operator notices the failed authentication and changes the secret through the `ProviderEndpoint`. New server keys are handed to PM's administrators, since PM never fetches them over plain HTTPS.

All of them write to the store with compare-and-swap and report through `OnAdministrativeChange`.

## Storage

**An interface**, with entities for: connections (owner, addressing, secret hash, pinned connector CA bundle, negotiated version), tokens (hash, owner, expiry, spent), and server keys (current, announced, retired, private keys).

**The Kubernetes reference store** keeps these in the Provider's runtime cluster: non-secret state as **custom resources** (compare-and-swap through `resourceVersion`, inspectable with kubectl), private keys and hashes in **Secrets**. The SDK ships the CRDs. Requirements on any store:
- atomic updates of a connection entry (compare-and-swap), since several replicas may handle calls;
- private keys never readable by anything but the connection server.

**Keep the store as durable as consumer data.** If the store survives what consumer data survives, a rebuilt cluster only needs the store back; otherwise the administrators import connection entries (see "Administration").

## Replicas

Sessions for different shards can land on different replicas behind the Provider's ingress, while the replica running a connection's controllers needs all of them: the dial hook only reaches sessions held by its own process. Controllers therefore run in the same process as the connection server.

### This delivery: one active replica

- **Leader election** picks the active replica; the vendor's controllers are leader-elected the usual controller-runtime way, with their reconcile logic unchanged.
- **Standby replicas answer session and control-session upgrades with `503` and `Retry-After: 5`.** Connectors use the value as the lower bound for their normal reconnect backoff, which grows on repeated failures, so a Provider with a longer leader-election Lease just sees a few more `503`s during a failover ([01 Protocol, "Errors and retries"](./xxx-notes-011-v5-impl-01-protocol.md)); with two or three replicas, sessions are back within roughly 10–20 seconds after a failover, and watches relist as after any restart.
- **The edge must spread upgrades across replicas**: no client-IP affinity or sticky sessions on the session and control paths. Otherwise a connector can keep reaching the same standby and never find the leader.
- **Constraints on the vendor**: nothing beyond leader election (Lease RBAC, which leader-elected operators already have), plain WebSocket support at the edge, and no affinity on the session and control paths. No extra Services, routes or permissions.
- **What vendors accept**: vertical scaling only (one process holds every connection's caches and traffic), and a reconnect lottery plus relists after a failover.

### Later: horizontal scale

An opt-in mode for vendors who need more than one active replica is a [follow-up](./xxx-notes-011-v5-impl-todo.md). It needs no PM-side work: the protocol already makes no assumption about which replica a session reaches.

## Requirements on Providers

Documented for Provider authors ([08 Quickstart, "Documentation for Provider authors"](./xxx-notes-011-v5-impl-08-provider-quickstart.md)):
- **Service endpoint slices live in the provider workspace**, for exports in that workspace. Connectors open sessions on a shard only if such a slice lists it ([05 Connector, "Which sessions"](./xxx-notes-011-v5-impl-05-connector.md)).
- **The edge passes the session and control paths as plain WebSocket pipes**: upgrades allowed, no bot challenges, binary messages up to 32 KB untouched, idle timeouts above the ping interval, and a maximum connection age the connector can replace sessions within. Message-based WebSocket API gateways (which route individual messages and cap connection lifetimes) don't qualify. The quickstart's checklist spells this out ([08 Quickstart, "Connection server"](./xxx-notes-011-v5-impl-08-provider-quickstart.md)); platform-mesh-operator reports violations it sees on the `Provider` with a labelled message.
- **One active replica** of the connection server and its controllers, by leader election (see "Replicas").
- **Several sessions per connection and shard**: every replica of PM's connector for a shard keeps its own session, so expect up to R sessions per (connection, shard), with R typically 2–3, and size the edge's concurrent-connection limits accordingly, including per source IP: all of a PM installation's connectors leave through the same few addresses. Streams, and so watches and requests, aren't multiplied ([01 Protocol, "Sessions and streams"](./xxx-notes-011-v5-impl-01-protocol.md)).
- **Rate limits** on registration per owner: SDK defaults, adjustable.
- **Consumer data is keyed with the `connectionID`**, never only with the consumer's logical cluster and the object's UID (see "Principles").
- **Adopt, don't create.** A controller whose local state is missing finds existing external resources through the kcp object (an ID in its `status` or annotations) or through the external resource itself (tags naming the `connectionID`, cluster and UID), before creating anything.
- **Removing a connection never deletes consumer data.** Only deleting a kcp object does, through its finalizer. A connection can be removed and come back (deleted in the portal by mistake, imported after a disaster), and consumers' bindings outlive it.
- **Bootstrap only creates or adopts.** It never deletes and recreates the APIExport or its identity Secret; a new identity would break every consumer's binding.

## Testing support

- **A fake PM side** for Providers' tests: registers, opens sessions with test certificates, sends requests through them.
- **A test Provider-side server** platform-mesh-operator and connector use in their own tests ([03](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md), [05](./xxx-notes-011-v5-impl-05-connector.md)).
- **The protocol's test vectors** ([01 Protocol, "Deliverables"](./xxx-notes-011-v5-impl-01-protocol.md)).

## Work breakdown

1. Module scaffold, protocol types, test vectors.
2. Storage interface, in-memory store for tests, Kubernetes reference store.
3. Tokens and portal primitives, hooks.
4. Connection API handlers.
5. Control sessions (inner TLS server, bearer check, compare-and-swap on the bundle).
6. Session server (handshake limits, inner mutual TLS, hello, yamux, registry, re-checking sessions on a bundle change).
7. Transport and connection registry; multicluster-runtime helper.
8. Status, server keys, the `ProviderEndpoint` watch (`pendingSecretHash`, `serverKeys`).
9. Framing enforcement.
10. Fake PM side and test server.
11. Documentation for Provider authors, including the edge requirements and the requirements on consumer data.
12. Administration commands (re-pin, import, server key rotation); may follow the first release.

## Releases and protocol versions

- **Its own semver line**, tagged `provider-sdk/vX.Y.Z` through the monorepo's `cmd/release` as a library component (no image), released when the SDK changes, not with every PM release.
- **A new protocol version is a new SDK minor version**, supporting the new and the previous protocol version ([01 Protocol, "Versioning"](./xxx-notes-011-v5-impl-01-protocol.md)). Dropping a protocol version is a breaking change for Providers that haven't upgraded, announced one release ahead.
- **Signals that a Provider should upgrade**:
  - the SDK logs a warning when a PM-side offers a newer protocol version than it supports;
  - platform-mesh-operator sets a **`ProtocolOutdated`** warning condition on the `Provider` while its connection still negotiates the previous version ([03 Operator, "Health"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)), so the owner sees it too.

## Decided

- **One active replica** (leader election, standbys answer `503` with `Retry-After: 5`); horizontal scale is a follow-up needing no PM-side work.
- **Hooks**: a struct of optional typed functions, called directly; reliability is the implementor's choice.
- **Server keys rotate only on demand**, through an administration command with a deadline.
- **Go only.** The protocol specification and test vectors exist, but third-party implementations aren't a supported target.
- **Reference store**: custom resources plus Secrets.
- **Recovery**: secrets change only through the `ProviderEndpoint`; recovery beyond that is out-of-band, through the administration commands.
- **Module**: `go.platform-mesh.io/provider-sdk`; independent semver; upgrade signals as above.
