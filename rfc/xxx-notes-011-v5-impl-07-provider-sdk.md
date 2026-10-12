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
- **Notification hooks** the Provider implements: registration, secret rotation, session CA change, rebind started, completed or cancelled, authentication failures above a threshold. The quickstart implements them as log lines and a portal list ([08 Quickstart](./xxx-notes-011-v5-impl-08-provider-quickstart.md)).

### Connection API

The handlers for the plain-HTTPS endpoints:
- **Registration**: verify and spend the token, apply idempotency rules, create the connection entry (owner, `clusterID`, front-proxy URL, secret hash, pinned session CAs, negotiated version), return `connectionID` and server keys.
- **Rebind**: verify the token's owner matches the connection's, start the waiting period, notify.
- **Server keys**: return the current list.

### Control sessions

- **WebSocket upgrade, then the inner TLS server** with the current server key.
- **Bearer check** inside, against the current or staged secret hash; any success cancels a pending rebind and promotes a staged secret.
- **Handlers**: status, secret staging, session CA replacement, delete ([01 Protocol, "Control sessions"](./xxx-notes-011-v5-impl-01-protocol.md)).

### Session server

1. **Verify the session assertion** in the outer header, exactly as specified ([01 Protocol, "Outer layer"](./xxx-notes-011-v5-impl-01-protocol.md)), including the `jti` cache.
2. **Accept the WebSocket**, run the **inner TLS server**, and require the client certificate to equal the assertion's leaf.
3. **Start yamux**; register the session under its connection, with the hosts it serves (from the shard certificate) plus the front-proxy.
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

**An interface**, with entities for: connections (owner, addressing, secret hash and staged hash, pinned session CAs, negotiated version, frozen flag, rebind state), tokens (hash, owner, expiry, spent), server keys (current, announced, retired, private keys), and the `jti` cache.

**The Kubernetes reference store** keeps these in the Provider's runtime cluster: non-secret state as **custom resources** (compare-and-swap through `resourceVersion`, inspectable with kubectl), private keys and hashes in **Secrets**. The SDK ships the CRDs. Requirements on any store:
- atomic updates of a connection entry (compare-and-swap), since several replicas may handle calls;
- a shared `jti` cache across replicas, or session upgrades pinned to one replica;
- private keys never readable by anything but the connection server.

## Replicas: one active for now

Sessions for different shards can land on different replicas behind the Provider's ingress, while the replica running a connection's controllers needs all of them. This delivery avoids the problem:
- **One active replica**, chosen by leader election; standby replicas refuse session and control-session upgrades (so the ingress retries elsewhere) until they become leader.
- **Failover**: connectors reconnect within their backoff; controllers relist, as after any restart.
- **Later**: forwarding sessions to a per-connection owner replica, or routing dials between replicas (as railgrid's edges provider does). The storage interface already assumes several replicas (compare-and-swap, shared `jti` cache), so this doesn't change the store.

## Requirements on Providers

Documented for Provider authors ([08 Quickstart, "Documentation for Provider authors"](./xxx-notes-011-v5-impl-08-provider-quickstart.md)):
- **Service endpoint slices live in the provider workspace**, for exports in that workspace. Connectors open sessions on a shard only if such a slice lists it ([05 Connector, "Which sessions"](./xxx-notes-011-v5-impl-05-connector.md)).
- **The edge passes the session and control paths as plain WebSocket pipes**: upgrades allowed, no bot challenges, binary messages up to 32 KB untouched, idle timeouts above the ping interval, and a maximum connection age the connector can replace sessions within. Message-based WebSocket API gateways (which route individual messages and cap connection lifetimes) don't qualify. The quickstart's checklist spells this out ([08 Quickstart, "Connection server"](./xxx-notes-011-v5-impl-08-provider-quickstart.md)); the edge self-test verifies it (above).
- **One active replica** of the connection server (above).
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

- **One active replica** (leader election) in this delivery.
- **Go only.** The protocol specification and test vectors exist, but third-party implementations aren't a supported target.
- **Reference store**: custom resources plus Secrets.
- **Module**: `go.platform-mesh.io/provider-sdk`; independent semver; upgrade signals as above.
