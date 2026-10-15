# v5 implementation plan: Provider SDK

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). The library every Provider uses: the Go module `go.platform-mesh.io/provider-sdk` at `platform-mesh/provider-sdk`, next to `apis`, `subroutines` and `golang-commons`, implementing the [protocol](./xxx-notes-011-v5-impl-01-protocol.md). It implements the Provider-side of the protocol, so Providers don't need to understand kcp, TLS pinning or the multiplexer. The [quickstart](./xxx-notes-011-v5-impl-08-provider-quickstart.md) is its reference user.

## What a Provider gets

- **A connection server** to mount under its base URL: registration, control sessions, sessions.
- **Connections for its controllers**: for each connection, a client config whose requests travel over that connection's sessions, and events when connections come and go.
- **Its state as Kubernetes resources**: `Connection`, `OnboardingToken` and `ServerKey` in the Provider's own cluster, with Events on each `Connection` (see "API resources"). The Provider's portal, administrators and controllers use them through the Kubernetes API like any other resource.
- **A token helper** for its portal.
- **Metrics** for its operators.

What a Provider still does itself: its portal and user accounts, its operator code (which endpoint slices it reconciles), its ingress, and telling its customers about changes to their connections.

## Principles

- **No kcp knowledge on the Provider-side.** No tokens, kubeconfigs or kcp CA bundles; requests carry no credentials ([01 Protocol](./xxx-notes-011-v5-impl-01-protocol.md)).
- **Everything security-relevant is in the SDK**, written once and tested with the protocol's test vectors, so Providers can't get it subtly wrong.
- **The Kubernetes API is the only source of truth** for the SDK's state. The SDK keeps caches and live sessions in memory, never a second copy of state; everything that has to survive a restart is in the resources.
- **State belongs to its connection** (`connectionID`), never inherited by another connection. Consumer data has to be keyed with the `connectionID` too, not only with the consumer's logical cluster and the object's UID: anyone with a portal account can register a connection with a connector CA of their own and serve made-up objects that reuse a real consumer's cluster and UID (see "Requirements on Providers").
- **`connectionID`s are stable.** A connection keeps its ID through secret changes and recovery, so state keyed by it stays valid.
- **`connectionID`s are random and unguessable** (e.g. 128 random bits), so probing the session and control paths can't find valid ones. They aren't secret, and nothing relies on them being so.
- **Only PM changes a connection's secret**, through the connection's `ProviderEndpoint` ([01 Protocol, "Changing the secret"](./xxx-notes-011-v5-impl-01-protocol.md)).
- **Secrets are stored hashed**; comparisons are constant-time; nothing secret is logged.

## API resources

Group `provider-sdk.platform-mesh.io/v1alpha1`, shipped as CRDs with the SDK; all namespaced, in one namespace the Provider configures (the SDK's namespace). The group's version is independent of the protocol version: it describes stored objects, not wire messages (see "Releases and protocol versions").

### Connection

```yaml
apiVersion: provider-sdk.platform-mesh.io/v1alpha1
kind: Connection
metadata:
  name: c-7f3k9q2mx8w4                         # the connectionID
  namespace: wild-west-system
  finalizers: [provider-sdk.platform-mesh.io/sessions]
spec:
  registration:                                # immutable as a whole
    owner: alice@example.com                   # selectable field
    clusterName: 2x8k9abc
    frontProxyURL: https://pm.example.com
  connectorCABundle: |                         # PEM; PM push or administrator re-pin
    -----BEGIN CERTIFICATE-----
    ...
status:
  conditions:
    - type: Ready                              # AsExpected | NoSessions | KcpUnreachable; failures after ~5 min
      status: "True"
      reason: AsExpected
      observedGeneration: 4
  lastSeen: "2026-10-15T10:00:00Z"             # hourly; empty: never used
  listedServerKeys: [sk-2026-04, sk-2026-10]
  secretHash: "sha256:..."                     # current connection secret's hash; only the SDK writes it
```

- **Created at registration** by the SDK. `spec.registration` is immutable (CEL: `self == oldSelf`); `spec.registration.owner` is a selectable field, so a portal lists one owner's connections with a field selector.
- **`spec.connectorCABundle`** is what sessions are verified against. A PM push writes it ([01 Protocol, "Connector CA bundle"](./xxx-notes-011-v5-impl-01-protocol.md)), and so does an administrator re-pinning; either change makes the SDK re-check open sessions (see "Session server").
- **`Ready`** says whether every host the Provider's clients use has a session and reaches kcp. It reports `NoSessions` or `KcpUnreachable` only after the state has lasted a few minutes (e.g. 5), so a failover, when a new leader briefly sees no sessions, doesn't flip every connection. Per-host detail stays in memory, where `GET` reads it, and in metrics.
- **`lastSeen`**: the last control call or session, written at most once an hour. Empty means never used, which is what orphan expiry looks for; an old value marks a connection whose PM installation went away without `DELETE`.
- **`listedServerKeys`**: the `ServerKey`s PM's `ProviderEndpoint` last listed. Set at registration to the keys returned, then updated whenever the SDK reads the `ProviderEndpoint`. The inner TLS handshake picks the key from it before any session exists (see "Server keys").
- **`secretHash`**: the current connection secret's SHA-256. In status because only the SDK writes it; it isn't in a Secret because a hash of a 256-bit random secret reveals nothing, and the same kind of hash is already public in the `ProviderEndpoint`. If it's lost (e.g. a restore without status), PM's next control call fails and PM repairs the secret ([03 Operator, "Secret changes"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
- **Deleting the object is the delete**, whoever does it: a portal, an administrator with `kubectl`, the SDK on PM's `DELETE` call, or orphan expiry. The finalizer closes the connection's sessions and control sessions, takes it out of the registry, emits `Removed` and lets the object go. Deleting is final.
- **Events**: `Registered`, `SecretChanged`, `ConnectorCABundleChanged` (with the number of sessions closed), `Removed` (with the reason) and `AdministrativeChange`. A Provider that tells its customers about changes watches `Connection`s and their Events, like any other resource. The SDK also logs every event, never with secrets.

### OnboardingToken

```yaml
apiVersion: provider-sdk.platform-mesh.io/v1alpha1
kind: OnboardingToken
metadata:
  name: t-q8m2vx                               # the token ID
  namespace: wild-west-system
spec:                                          # immutable as a whole
  owner: alice@example.com                     # selectable field
  expiresAt: "2026-10-15T09:30:00Z"
  tokenHash: "sha256:..."
status:
  registration:                                # present: spent
    connectionID: c-7f3k9q2mx8w4
    idempotencyKey: "<ProviderOnboardingRequest UID>"
    at: "2026-10-15T09:12:00Z"
```

- **Issued by the Provider's portal** with `NewOnboardingToken(baseURL, owner, ttl)`, which returns the token string `pm1.<base URL>.<id>.<secret>`, shown once, and the object, which the portal creates. The plaintext token is never stored.
- **Spent** when `status.registration` is set, by compare-and-swap, so a token can't be spent twice. Registration retries with the same `idempotencyKey` and the hash the `Connection` holds get the same answer, for 7 days by default.
- **Garbage-collected by the SDK**: unspent tokens at `expiresAt`, spent ones once the retry window has passed since `registration.at`.

### ServerKey

```yaml
apiVersion: provider-sdk.platform-mesh.io/v1alpha1
kind: ServerKey
metadata:
  name: sk-2026-10
  namespace: wild-west-system
spec:
  signerRef:                                   # optional, immutable; key names a specific key version, not an alias
    provider: gcp-kms                          # a signer factory registered with sdk.RegisterSigner
    key: projects/acme/locations/europe-west1/keyRings/pm/cryptoKeys/pm-server-2026-10/cryptoKeyVersions/1
  retireAt: "2026-10-22T00:00:00Z"             # optional
status:
  conditions:
    - type: Ready                              # AsExpected | SignerUnavailable | KeyMissing | KeyMismatch
      status: "True"
      reason: AsExpected
      observedGeneration: 2
  publicKey: "<PEM SPKI>"
```

Without `signerRef`, the private key is in a Secret of the same name, with an owner reference to the `ServerKey`, readable only by the SDK's identity. With it, the private key stays in the KMS or HSM. See "Server keys".

### Access

RBAC in the SDK's namespace:

| Identity | `Connection` | `OnboardingToken` | `ServerKey` | Secrets |
|---|---|---|---|---|
| SDK | all, including status | all, including status | all, including status | private keys only |
| Portal | `get`, `list`, `delete` | `create`, `get`, `list` | — | none |
| Provider's administrators | all except status | `get`, `list`, `delete` | all except status | none |
| Provider's controllers | `get`, `list`, `watch` | — | — | — |

**Keep these resources as durable as consumer data.** If they survive what consumer data survives, a rebuilt cluster only needs them back; otherwise the administrators import `Connection`s (see "Administration").

## Components

### Connection API

The handler for the one plain-HTTPS endpoint:
- **Registration**: verify the token against its `OnboardingToken`, spend it, apply the idempotency rules, create the `Connection` (owner, `clusterName`, front-proxy URL, secret hash, the pinned connector CA bundle, the server keys returned), return `connectionID` and server keys.
- **Orphan expiry**: a `Connection` with an empty `lastSeen` at the end of the retry window is deleted; it's a registration whose result PM never stored. A connection that was used once is never expired for being idle, however long its Provider or PM is away.

### Control sessions

- **WebSocket upgrade, then the inner TLS server** with the connection's server key (see "Server keys"), speaking HTTP/1.1 with keep-alive. The version is negotiated through ALPN ([01 Protocol, "Versioning"](./xxx-notes-011-v5-impl-01-protocol.md)).
- **Bearer check** on each request, against `secretHash` only. **The first failed check closes the control session**, so every guess costs a new WebSocket and handshake.
- **Handlers**: status, connector CA bundle replacement, delete ([01 Protocol, "Control sessions"](./xxx-notes-011-v5-impl-01-protocol.md)). None changes the secret. Delete deletes the `Connection`.
- **Connector CA bundle**: a PEM bundle, parsed strictly (only `CERTIFICATE` blocks, no PEM headers, only whitespace around blocks, no duplicates, every certificate a CA, bounded size); Go's `AppendCertsFromPEM` isn't used, since it skips what it can't parse. Replacement is compare-and-swap on the pinned bundle's `etag`, a hash over its certificates' fingerprints, computed from `spec.connectorCABundle`: a missing or stale `If-Match` is refused with `412`, the same set of certificates is a no-op ([01 Protocol, "Connector CA bundle"](./xxx-notes-011-v5-impl-01-protocol.md)). The `etag` and fingerprints are reported in `GET`. The bundle is only ever accepted on the control channel or at registration, never from a session. A push is accepted even if every pinned CA has expired: it's authenticated by the secret, not by the pinned CAs.

### The connection's ProviderEndpoint

For each connection, the SDK watches its `ProviderEndpoint` (`default`, cluster-scoped, in the connection's provider workspace, through the front-proxy) over the connection's own sessions. Two things come from it:
- **`pendingSecretHash`**: when set and different from `secretHash`, it becomes the `Connection`'s `secretHash` and the old secret stops working ([01 Protocol, "Changing the secret"](./xxx-notes-011-v5-impl-01-protocol.md)). This is the only way the hash changes after registration.
- **`serverKeys`**: which server keys PM has pinned, kept as `listedServerKeys` (see "Server keys").

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
- **Per-host bookkeeping** of request outcomes, in memory, which feeds status and `Ready`.

### Connections for multicluster-runtime

- **A registry of connections**, built from `Connection`s, with events (added, removed) and, per connection, a client config: front-proxy URL as host plus the transport.
- **A helper** that, for each connection, creates a multicluster-runtime endpoint-slice provider per endpoint slice the Provider names, and composes them with cluster names prefixed by `connectionID`.
- **Connections keep their `connectionID`** through secret changes, repairs and administrator imports, so nothing changes for the Provider's controllers.

### Status

Answers `GET` from memory with, per host the Provider's clients actually use: live sessions, whether recent requests reached kcp, and the last error, without sensitive detail ([01 Protocol, "Status"](./xxx-notes-011-v5-impl-01-protocol.md)).

### Server keys

- **Generated by the SDK**: on first start, with no `ServerKey`, it creates one; for every `ServerKey` without `signerRef` and with an empty `status.publicKey`, it generates the key pair into the Secret and sets `publicKey`. It never generates a new key under an existing name, since connections have pinned the old one: a missing Secret is `Ready=False` with `KeyMissing`, and the fix is a rotation.
- **KMS or HSM keys** through `signerRef`. The Provider registers a signer factory in its binary, `sdk.RegisterSigner("gcp-kms", func(ctx, key string) (crypto.Signer, error) {…})`, with any `crypto.Signer` implementation (sigstore's KMS packages and `crypto11` for PKCS#11 are examples); the SDK ships no KMS clients. The SDK reads the public key through `Public()`, checks that the inner TLS accepts its algorithm (e.g. ECDSA P-256 or P-384), and sets `publicKey`. Access to the KMS comes from the Provider's pod identity. Failures are `SignerUnavailable`. On every start the SDK compares `Public()` with `publicKey` and refuses a key it can't prove: `KeyMismatch`.
- **Which key a connection presents**: every `ServerKey` not past `retireAt` is announced in `GET`; each connection presents the newest announced key in its `listedServerKeys`.
- **Rotation only on demand** (see "Administration"), for example after a suspected key compromise. Nothing rotates on a schedule. There's no plain-HTTPS key endpoint ([01 Protocol, "Server keys"](./xxx-notes-011-v5-impl-01-protocol.md)).

### Metrics

Prometheus collectors, which the Provider adds to the registry it already serves (controller-runtime's, for most):
- authentication failures on control sessions and sessions;
- connector CA bundles refused, by reason;
- sessions by state, and sessions closed, by reason (go-away, edge cut, bundle re-check, connection removed);
- handshakes refused before or during the upgrade, by reason (unknown ID, limits, timeout);
- connections;
- `providersdk_server_key_connections{key, listed}`: how many `Connection`s list each `ServerKey`, from the SDK's cache. A documented alert rule fires when a key with `retireAt` set is still missing from connections seen in the last day, within a day of `retireAt`.

**No `connectionID` labels**: with thousands of connections, they'd explode cardinality. Per-connection detail is in the resources, their Events, the logs and the status `GET`.

### WebSocket framing

The SDK enforces the protocol's framing on every session and control session ([01 Protocol, "WebSocket framing"](./xxx-notes-011-v5-impl-01-protocol.md)): binary messages only, at most 32 KB each, outer pings at the protocol's interval. Providers can't configure it away.

### Administration

Runbooks for the Provider's administrators, for cases no protocol call can fix safely. Everything is `kubectl` on the resources; the SDK's controllers react. Inputs are exchanged out-of-band with the people running the PM installation.
- **Re-pin a connection's connector CA bundle**, when no session verifies (for example, a bundle that missed a whole rollover while the connection also lost its secret): edit `spec.connectorCABundle`. Once sessions work again, PM repairs the secret through the `ProviderEndpoint`.
- **Import a connection** after the Provider-side lost its state: apply a `Connection` with its `connectionID`, owner, `clusterName`, front-proxy URL and the connector CA bundle. No secret hash: platform-mesh-operator notices the failed authentication and changes the secret through the `ProviderEndpoint`. New server keys go to PM's administrators, since PM never fetches them over plain HTTPS.
- **Rotate the server key**:
  1. create a new `ServerKey`: without a KMS, a manifest with just `generateName: sk-` (`kubectl create`); with one, create the key in the KMS first and name its version in `signerRef`;
  2. the SDK announces it; platform-mesh-operator adds it to each connection's `ProviderEndpoint` on its next poll ([03 Operator, "Writing ProviderEndpoints"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)), and each connection moves to it;
  3. watch `providersdk_server_key_connections`, and list the connections still missing the new key with their `lastSeen`:
     ```
     kubectl get connections -n <ns> -o json | jq -r '.items[]
       | select(.status.listedServerKeys | index("sk-2026-10") | not)
       | [.metadata.name, .spec.registration.owner, .status.lastSeen] | @tsv'
     ```
  4. set `retireAt` on the old key: soon for a planned rotation, a short deadline after a suspected compromise, accepting that connections not reached by then lose their sessions until a PM administrator supplies the new key (`status.publicKey`);
  5. delete the old `ServerKey` once retired, never before; its Secret follows.

## Replicas

Sessions for different shards can land on different replicas behind the Provider's ingress, while the replica running a connection's controllers needs all of them: the dial hook only reaches sessions held by its own process. Controllers therefore run in the same process as the connection server.

### This delivery: one active replica

- **Leader election** picks the active replica; the vendor's controllers are leader-elected the usual controller-runtime way, with their reconcile logic unchanged. The SDK's controllers run on the leader.
- **Standby replicas answer session and control-session upgrades with `503` and `Retry-After: 5`.** Connectors use the value as the lower bound for their normal reconnect backoff, which grows on repeated failures, so a Provider with a longer leader-election Lease just sees a few more `503`s during a failover ([01 Protocol, "Errors and retries"](./xxx-notes-011-v5-impl-01-protocol.md)); with two or three replicas, sessions are back within roughly 10–20 seconds after a failover, and watches relist as after any restart.
- **Constraints on the vendor**: nothing beyond leader election (Lease RBAC, which leader-elected operators already have) and an edge that meets the quickstart's checklist ([08 Quickstart, "Connection server"](./xxx-notes-011-v5-impl-08-provider-quickstart.md)). No extra Services, routes or permissions.
- **What vendors accept**: vertical scaling only (one process holds every connection's caches and traffic), and a reconnect lottery plus relists after a failover.

### Later: horizontal scale

An opt-in mode for vendors who need more than one active replica is a [follow-up](./xxx-notes-011-v5-impl-todo.md). It needs no PM-side work: the protocol already makes no assumption about which replica a session reaches.

## Requirements on Providers

Documented for Provider authors ([08 Quickstart, "Documentation for Provider authors"](./xxx-notes-011-v5-impl-08-provider-quickstart.md)):
- **Service endpoint slices live in the provider workspace**, for exports in that workspace. Connectors open sessions on a shard only if such a slice lists it ([05 Connector, "Which sessions"](./xxx-notes-011-v5-impl-05-connector.md)).
- **The edge meets the quickstart's checklist** ([08 Quickstart, "Connection server"](./xxx-notes-011-v5-impl-08-provider-quickstart.md)); platform-mesh-operator reports violations it sees on the `Provider` with a labelled message.
- **One active replica** of the connection server and its controllers, by leader election (see "Replicas").
- **Rate limits** on registration per owner: SDK defaults, adjustable.
- **Consumer data is keyed with the `connectionID`**, never only with the consumer's logical cluster and the object's UID (see "Principles").
- **Adopt, don't create.** A controller whose local state is missing finds existing external resources through the kcp object (an ID in its `status` or annotations) or through the external resource itself (tags naming the `connectionID`, cluster and UID), before creating anything.
- **Removing a connection never deletes consumer data.** Only deleting a kcp object does, through its finalizer. A connection can be removed and come back (deleted in the portal by mistake, imported after a disaster), and consumers' bindings outlive it.
- **Bootstrap only creates or adopts.** It never deletes and recreates the APIExport or its identity Secret; a new identity would break every consumer's binding.

## Testing support

- **A fake PM side** for Providers' tests: registers, opens sessions with test certificates, sends requests through them.
- **A test Provider-side server** platform-mesh-operator and connector use in their own tests ([03](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md), [05](./xxx-notes-011-v5-impl-05-connector.md)).
- **The protocol's test vectors** ([01 Protocol, "Deliverables"](./xxx-notes-011-v5-impl-01-protocol.md)).
- **envtest** for the SDK's own controllers and handlers against real CRDs.

## Work breakdown

1. Module scaffold, protocol types, test vectors.
2. The CRDs (`Connection`, `OnboardingToken`, `ServerKey`) with their CEL rules, selectable fields and printer columns.
3. The token helper; metrics.
4. Connection API handlers.
5. Control sessions (inner TLS server, bearer check, compare-and-swap on the bundle).
6. Session server (handshake limits, inner mutual TLS, hello, yamux, registry, re-checking sessions on a bundle change).
7. Transport and connection registry; multicluster-runtime helper.
8. The SDK's controllers: `Connection` (finalizer, `Ready`, `lastSeen`, orphan expiry, Events), `OnboardingToken` (garbage collection), `ServerKey` (generation, signers, checks); the `ProviderEndpoint` watch (`pendingSecretHash`, `serverKeys`); status.
9. Framing enforcement.
10. Fake PM side and test server.
11. Documentation for Provider authors, including the requirements on consumer data and the administration runbooks.

## Releases and protocol versions

- **Its own semver line**, tagged `provider-sdk/vX.Y.Z` through the monorepo's `cmd/release` as a library component (no image), released when the SDK changes, not with every PM release.
- **A new protocol version is a new SDK minor version**, supporting the new and the previous protocol version ([01 Protocol, "Versioning"](./xxx-notes-011-v5-impl-01-protocol.md)). Dropping a protocol version is a breaking change for Providers that haven't upgraded, announced one release ahead.
- **The API version of `provider-sdk.platform-mesh.io` evolves separately**, by Kubernetes' rules (`v1alpha1`, then `v1beta1`, then `v1`, with conversion), and is announced in the same release notes. The resources store the SDK's own representation, not wire messages, so a protocol change rarely needs one.
- **Signals that a Provider should upgrade**:
  - the SDK logs a warning when a PM-side offers a newer protocol version than it supports;
  - platform-mesh-operator sets a **`ProtocolOutdated`** warning condition on the `Provider` while its control sessions still negotiate the previous version ([03 Operator, "Health"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)), so the owner sees it too.

## Decided

- **One active replica** (leader election, standbys answer `503` with `Retry-After: 5`); horizontal scale is a follow-up needing no PM-side work.
- **Kubernetes-API-first**: the SDK's state is `Connection`, `OnboardingToken` and `ServerKey` in `provider-sdk.platform-mesh.io/v1alpha1`, the only source of truth; deleting a `Connection` is the delete; the only Secrets hold private keys.
- **Events, not hooks**: Providers watch `Connection`s and their Events. Operational signals are metrics, without `connectionID` labels.
- **Server keys**: generated by the SDK or behind a registered `crypto.Signer`; rotated only on demand, with `kubectl` and `retireAt`.
- **Administration** is `kubectl` on the resources, documented as runbooks.
- **Go only.** The protocol specification and test vectors exist, but third-party implementations aren't a supported target.
- **Connector CA bundle**: PEM, parsed strictly.
- **Recovery**: secrets change only through the `ProviderEndpoint`; recovery beyond that is out-of-band, through the runbooks.
- **Module**: `go.platform-mesh.io/provider-sdk`; independent semver; upgrade signals as above.
