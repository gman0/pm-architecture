# v5 implementation plan: Provider SDK

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). The library every Provider uses: the Go module `go.platform-mesh.io/provider-sdk` at `platform-mesh/provider-sdk`, next to `apis`, `subroutines` and `golang-commons`, implementing the [protocol](./xxx-notes-011-v5-impl-01-protocol.md). It implements the Provider-side of the protocol, so Providers don't need to understand kcp, TLS pinning or the multiplexer. The [quickstart](./xxx-notes-011-v5-impl-08-provider-quickstart.md) is its reference user.

## What a Provider gets

- **A connection server** to mount under its base URL: registration, rebind, server keys, control sessions, sessions.
- **Connections for its controllers**: for each connection, a client config whose requests travel over that connection's sessions, and events when connections come and go.
- **Portal primitives**: issuing onboarding tokens, listing, freezing and deleting connections, owner notifications.
- **A storage interface** with a Kubernetes reference store.

What a Provider still does itself: its portal and user accounts, its operator code (which endpoint slices it reconciles), its ingress, and where it stores state if it doesn't use the reference store.

## Principles

- **No kcp knowledge on the Provider-side.** No tokens, kubeconfigs or kcp CA bundles; requests carry no credentials ([01 Protocol](./xxx-notes-011-v5-impl-01-protocol.md)).
- **Everything security-relevant is in the SDK**, written once and tested with the protocol's test vectors, so Providers can't get it subtly wrong.
- **State belongs to its connection** (`connectionID`), never inherited by another connection.
- **Secrets are stored hashed**; comparisons are constant-time; nothing secret is logged.

## Components

### Onboarding tokens and portal primitives

- **Issue tokens** bound to a portal owner account: `pm1.<base URL>.<id>.<secret>`, single-use, short expiry; store only the hash.
- **List, freeze and delete** a portal owner's connections. Freezing refuses all control calls and sessions until unfrozen; deleting is final.
- **Notification hooks** the Provider implements: registration, secret rotation, connector CA change, rebind started, completed or cancelled, authentication failures above a threshold. The quickstart implements them as log lines and a portal list ([08 Quickstart](./xxx-notes-011-v5-impl-08-provider-quickstart.md)).

### Connection API

The handlers for the plain-HTTPS endpoints:
- **Registration**: verify and spend the token, apply idempotency rules, create the connection entry (owner, `clusterID`, front-proxy URL, secret hash, the pinned connector CA bundle with its sequence, negotiated version), return `connectionID` and server keys.
- **Rebind**: verify the token's owner matches the connection's, start the waiting period, notify.
- **Server keys**: return the current list.

### Control sessions

- **WebSocket upgrade, then the inner TLS server** with the current server key.
- **Bearer check** inside, against the current or staged secret hash; any success cancels a pending rebind and promotes a staged secret.
- **Handlers**: status, secret staging, connector CA bundle replacement, delete ([01 Protocol, "Control sessions"](./xxx-notes-011-v5-impl-01-protocol.md)).
- **Connector CA bundle**: parsed as a SPIFFE-format trust bundle (JWKS with `x5c`); every certificate is checked to be a CA before it's pinned. Replacement is monotonic in the bundle's sequence: lower is refused, equal and identical is a no-op, equal and different is refused ([01 Protocol, "Connector CA bundle"](./xxx-notes-011-v5-impl-01-protocol.md)). The pinned sequence and fingerprints are reported in `GET`. The bundle is only ever accepted on the control channel or at registration, never from a session.

### Session server

1. **Verify the session assertion** in the outer header, exactly as specified ([01 Protocol, "Outer layer"](./xxx-notes-011-v5-impl-01-protocol.md)), including the `jti` cache.
2. **Accept the WebSocket**, run the **inner TLS server**, and require the client certificate to equal the assertion's leaf.
3. **Start yamux**; register the session under its connection, with the hosts it serves (from the assertion's `hosts` claim) plus the front-proxy.
4. **Honour go-away**: stop opening streams on a session the connector is replacing.
5. **Close all sessions** of a connection that's deleted or frozen.

### Transport

- **A per-connection `http.Transport`** whose dial hook opens a yamux stream on a session that serves the target host, writes the preamble, and returns the stream. The request then goes as plain HTTP/1.1 on it; the real kcp URL stays in the request.
- **Session choice**: any session for the front-proxy; that shard's sessions for a virtual-workspace host; spread across several sessions serving the same host.
- **No TLS settings on the client config**: confidentiality comes from the inner TLS, and client-go refuses both a custom transport and TLS options.
- **Per-host bookkeeping** of request outcomes, which feeds status.

### Connections for multicluster-runtime

- **A registry of connections** with events (added, removed) and, per connection, a client config: front-proxy URL as host plus the transport.
- **A helper** that, for each connection, creates a multicluster-runtime endpoint-slice provider per endpoint slice the Provider names, and composes them with cluster names prefixed by `connectionID`.
- **Rebound connections keep their `connectionID`**, so nothing changes for the Provider's controllers.

### Status

Answers `GET` with, per host the Provider's clients actually use: live sessions, whether recent requests reached kcp, and the last error, without sensitive detail ([01 Protocol, "Status"](./xxx-notes-011-v5-impl-01-protocol.md)).

### Server keys

- **Generated by the SDK** on first start, stored with the reference store or the Provider's own.
- **Rotation with overlap**: announce the new key in `GET`, wait for the overlap (long enough for the operator to update the record, see [03 Operator, "Records"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)), switch, retire the old key.

### WebSocket framing

The SDK enforces the protocol's framing on every session and control session ([01 Protocol, "WebSocket framing"](./xxx-notes-011-v5-impl-01-protocol.md)): binary messages only, at most 32 KB each, outer pings at the protocol's interval. Providers can't configure it away.

### Edge self-test

Most onboarding failures are edge configuration: a bot challenge on the upgrade, a WAF dropping binary messages, timeouts that are too short. The SDK makes these visible before a PM instance ever connects:
- **An optional edge-check endpoint** in the connection server, enabled only when the Provider configures a test secret ([01 Protocol, "Diagnosing the edge"](./xxx-notes-011-v5-impl-01-protocol.md)). It echoes binary messages and holds connections open.
- **A self-test command** the Provider runs from outside its network (a laptop, CI) against its base URL. It checks, with a clear verdict for each:
  - the upgrade succeeds for a non-browser client (no bot challenge, no HTML response);
  - binary messages up to 32 KB pass in both directions, unchanged;
  - the connection survives idle periods up to the ping interval and beyond;
  - the maximum connection age (runs as long as asked, e.g. an hour, and reports when the edge cuts it).
- **The same checks in reverse are reported by PM**: the operator classifies edge failures on control sessions and puts the reason on the request or `Provider` ([01 Protocol, "Diagnosing the edge"](./xxx-notes-011-v5-impl-01-protocol.md)).

### Rebind

- **Waiting period** per pending rebind; **cancelled** by any successful control-session authentication with the current secret.
- **On completion**, replace the hash; everything else about the connection stays.

## Storage

**An interface**, with entities for: connections (owner, addressing, secret hash and staged hash, pinned connector CA bundle and its sequence, negotiated version, frozen flag, rebind state), tokens (hash, owner, expiry, spent), server keys (current, announced, retired, private keys), and the `jti` cache.

**The Kubernetes reference store** keeps these in the Provider's runtime cluster: non-secret state as **custom resources** (compare-and-swap through `resourceVersion`, inspectable with kubectl), private keys and hashes in **Secrets**. The SDK ships the CRDs. Requirements on any store:
- atomic updates of a connection entry (compare-and-swap), since several replicas may handle calls;
- a shared `jti` cache across replicas, or session upgrades pinned to one replica;
- private keys never readable by anything but the connection server.

## Replicas

Sessions for different shards can land on different replicas behind the Provider's ingress, while the replica running a connection's controllers needs all of them: the dial hook only reaches sessions held by its own process. Controllers therefore run in the same process as the connection server.

### This delivery: one active replica

- **Leader election** picks the active replica; the vendor's controllers are leader-elected the usual controller-runtime way, with their reconcile logic unchanged.
- **Standby replicas answer session and control-session upgrades with `503` and `Retry-After = max(5s, ceil(LeaseDuration / 3))`**, computed once at startup from the leader-election settings. With controller-runtime's default 15-second Lease that's 5 seconds: about three attempts span the worst-case window after a leader crash, when no replica can be leader yet. A vendor's longer Lease gives a longer delay (a new leader can't appear sooner anyway); the 5-second floor keeps an aggressively short Lease from turning reconnects into a storm against the vendor's own edge. Connectors use the value as the lower bound for their normal reconnect backoff ([01 Protocol, "Errors and retries"](./xxx-notes-011-v5-impl-01-protocol.md)); with two or three replicas, sessions are back within roughly 10–20 seconds after a failover, and watches relist as after any restart.
- **The edge must spread upgrades across replicas**: no client-IP affinity or sticky sessions on the session and control paths. Otherwise a connector can keep reaching the same standby and never find the leader.
- **Constraints on the vendor**: nothing beyond leader election (Lease RBAC, which leader-elected operators already have), plain WebSocket support at the edge, and no affinity on the session and control paths. No extra Services, routes or permissions.
- **What vendors accept**: vertical scaling only (one process holds every connection's caches and traffic), and a reconnect lottery plus relists after a failover.

### Follow-up: owner replicas, opt-in

For vendors who need horizontal scale, a later SDK release adds an opt-in mode:
- **Each connection has an owner replica** (rendezvous hashing over live replicas, or a Lease per connection). Any replica accepts an upgrade and forwards the raw bytes, in-cluster, to the owner, which verifies the assertion and terminates the inner TLS; forwarding replicas see only ciphertext. Control sessions are forwarded the same way.
- **Each replica runs controllers only for the connections it owns**, through the SDK's connection-aware manager.
- **When connections move** on scaling, the old owner sends go-away and the connector reconnects to whichever replica the ingress picks, which forwards to the new owner.
- **Constraints on vendors who opt in**: controllers sharded by connection, so **their reconcile logic must not assume one replica sees all connections** (anything aggregated across connections needs coordination of their own); pod-to-pod connectivity within their namespace; RBAC for Leases or EndpointSlices; relists when connections move.
- **No PM-side work**: from the connector's side an owner-replica Provider is indistinguishable from any other. This delivery already provides what it needs: go-away from either side, reconnection after any graceful close, `503` with `Retry-After` as a backoff lower bound, and no assumption about which replica a session reaches ([01 Protocol, "Provider replicas"](./xxx-notes-011-v5-impl-01-protocol.md)).

## Requirements on Providers

Documented for Provider authors ([08 Quickstart, "Documentation for Provider authors"](./xxx-notes-011-v5-impl-08-provider-quickstart.md)):
- **Service endpoint slices live in the provider workspace**, for exports in that workspace. Connectors open sessions on a shard only if such a slice lists it ([05 Connector, "Which sessions"](./xxx-notes-011-v5-impl-05-connector.md)).
- **The edge passes the session and control paths as plain WebSocket pipes**: upgrades allowed, no bot challenges, binary messages up to 32 KB untouched, idle timeouts above the ping interval, and a maximum connection age the connector can replace sessions within. Message-based WebSocket API gateways (which route individual messages and cap connection lifetimes) don't qualify. The quickstart's checklist spells this out ([08 Quickstart, "Connection server"](./xxx-notes-011-v5-impl-08-provider-quickstart.md)); the edge self-test verifies it (above).
- **One active replica** of the connection server and its controllers, by leader election (see "Replicas"); the owner-replica mode is a later opt-in.
- **Rate limits** on registration and rebind per owner: SDK defaults, adjustable.

## Testing support

- **A fake PM side** for Providers' tests: registers, opens sessions with test certificates, sends requests through them.
- **A test Provider-side server** the operator and connector use in their own tests ([03](./xxx-notes-011-v5-impl-03-providerconnections-operator.md), [05](./xxx-notes-011-v5-impl-05-connector.md)).
- **The protocol's test vectors** ([01 Protocol, "Deliverables"](./xxx-notes-011-v5-impl-01-protocol.md)).

## Work breakdown

1. Module scaffold, protocol types, test vectors.
2. Storage interface, in-memory store for tests, Kubernetes reference store.
3. Tokens and portal primitives, notification hooks.
4. Connection API handlers.
5. Control sessions (inner TLS server, bearer check).
6. Session server (assertion verification, inner TLS, yamux, registry).
7. Transport and connection registry; multicluster-runtime helper.
8. Status, server keys, rebind.
9. Framing enforcement; the edge-check endpoint and the self-test command.
10. Fake PM side and test server.
11. Documentation for Provider authors, including the edge requirements and how to run the self-test.

## Releases and protocol versions

- **Its own semver line**, tagged `provider-sdk/vX.Y.Z` through the monorepo's `cmd/release` as a library component (no image), released when the SDK changes, not with every PM release.
- **A new protocol version is a new SDK minor version**, supporting the new and the previous protocol version ([01 Protocol, "Versioning"](./xxx-notes-011-v5-impl-01-protocol.md)). Dropping a protocol version is a breaking change for Providers that haven't upgraded, announced one release ahead.
- **Signals that a Provider should upgrade**:
  - the SDK logs a warning when a PM-side offers a newer protocol version than it supports;
  - the operator sets a **`ProtocolOutdated`** warning condition on the `Provider` while its connection still negotiates the previous version ([03 Operator, "Readiness and health"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)), so the owner sees it too.

## Decided

- **One active replica** (leader election, standbys answer `503` with `Retry-After = max(5s, ceil(LeaseDuration / 3))`) in this delivery; **owner replicas** as an opt-in follow-up needing no PM-side work.
- **Go only.** The protocol specification and test vectors exist, but third-party implementations aren't a supported target.
- **Reference store**: custom resources plus Secrets.
- **Module**: `go.platform-mesh.io/provider-sdk`; independent semver; upgrade signals as above.
