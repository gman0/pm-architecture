# v5 implementation plan: SDK

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). The library every Provider uses: the Go module `go.platform-mesh.io/provider-sdk` at `platform-mesh/provider-sdk`, next to `apis`, `subroutines` and `golang-commons`, implementing the [protocol](./xxx-notes-011-v5-impl-01-protocol.md). It implements the Provider-side of the protocol, so Providers don't need to understand kcp, TLS pinning or the multiplexer. The [quickstart](./xxx-notes-011-v5-impl-08-provider-quickstart.md) is its reference user.

## What a Provider gets

- **A Connection API server** to mount under its Base URL: Registration, Control Sessions, Sessions.
- **Connections for its controllers**: for each Connection, a client config whose requests travel over that Connection's Sessions, and events when Connections come and go.
- **Its state as Kubernetes resources**: `Connection`, `OnboardingToken` and `ServerKey` in the Provider's own cluster, with Events on each `Connection` (see "API resources"). The Provider's portal, administrators and controllers use them through the Kubernetes API like any other resource.
- **A token helper** for its portal.
- **Metrics** for its operators.

What a Provider still does itself: its portal and user accounts, its operator code (which endpoint slices it reconciles), its ingress, and telling its customers about changes to their Connections.

## Principles

- **No kcp knowledge on the Provider-side.** No tokens, kubeconfigs or kcp CA bundles; requests carry no credentials ([01 Protocol](./xxx-notes-011-v5-impl-01-protocol.md)).
- **Everything security-relevant is in the SDK**, written once and tested with the protocol's test vectors, so Providers can't get it subtly wrong.
- **The Kubernetes API is the only source of truth** for the SDK's state. The SDK keeps caches and live Sessions in memory, never a second copy of state; everything that has to survive a restart is in the resources.
- **State belongs to its Connection** (`connectionID`), never inherited by another Connection. Consumer data has to be keyed with the `connectionID` too, not only with the consumer's logical cluster and the object's UID: anyone with a portal account can register a Connection with a Connector CA of their own and serve made-up objects that reuse a real consumer's cluster and UID (see "Requirements on Providers").
- **`connectionID`s are stable.** A Connection keeps its ID through Secret Changes and recovery, so state keyed by it stays valid.
- **`connectionID`s are random and unguessable** (e.g. 128 random bits), so probing the Session and control paths can't find valid ones. They aren't secret, and nothing relies on them being so.
- **Only PM changes a Connection Secret**, through the Connection's `ProviderEndpoint` ([01 Protocol, "Changing the Connection Secret"](./xxx-notes-011-v5-impl-01-protocol.md)).
- **Secret values are stored hashed**; comparisons are constant-time; nothing secret is logged.

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
  pinnedBundle: |                              # the Pinned Bundle, PEM; PM push or administrator re-pin
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

- **Created at Registration** by the SDK. `spec.registration` is immutable (CEL: `self == oldSelf`); `spec.registration.owner` is a selectable field, so a portal lists one Owner's Connections with a field selector.
- **`spec.pinnedBundle`** is what Sessions are verified against. A PM push writes it ([01 Protocol, "Pinned Bundle"](./xxx-notes-011-v5-impl-01-protocol.md)), and so does an administrator re-pinning; either change makes the SDK re-check open Sessions (see "Session server").
- **`Ready`** says whether every host the Provider's clients use has a Session and reaches kcp. It reports `NoSessions` or `KcpUnreachable` only after the state has lasted a few minutes (e.g. 5), so a failover, when a new leader briefly sees no Sessions, doesn't flip every Connection. Per-host detail stays in memory, where `GET` reads it, and in metrics.
- **`lastSeen`**: the last Control Call or Session, written at most once an hour. Empty means never used, which is what orphan expiry looks for; an old value marks a Connection whose PM Instance went away without `DELETE`.
- **`listedServerKeys`**: the `ServerKey`s PM's `ProviderEndpoint` last listed. Set at Registration to the keys returned, then updated whenever the SDK reads the `ProviderEndpoint`. The Inner TLS handshake picks the key from it before any Session exists (see "Server Keys").
- **`secretHash`**: the current Connection Secret's SHA-256. In status because only the SDK writes it; it isn't in a `Secret` because a hash of a 256-bit random Connection Secret reveals nothing, and the same kind of hash is already public in the `ProviderEndpoint`. If it's lost (e.g. a restore without status), PM's next Control Call fails and PM repairs the Connection Secret ([03 Operator, "Secret Changes"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
- **Deleting the object is the delete**, whoever does it: a portal, an administrator with `kubectl`, the SDK on PM's `DELETE` call, or orphan expiry. The finalizer closes the Connection's Sessions and Control Sessions, takes it out of the registry, emits `Removed` and lets the object go. Deleting is final.
- **Events**: `Registered`, `SecretChanged`, `PinnedBundleChanged` (with the number of Sessions closed), `Removed` (with the reason) and `AdministrativeChange`. A Provider that tells its customers about changes watches `Connection`s and their Events, like any other resource. The SDK also logs every event, never with secrets.

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

Without `signerRef`, the private key is in a `Secret` of the same name, with an owner reference to the `ServerKey`, readable only by the SDK's identity. With it, the private key stays in the KMS or HSM. See "Server Keys".

### Access

RBAC in the SDK's namespace:

| Identity | `Connection` | `OnboardingToken` | `ServerKey` | `Secret`s |
|---|---|---|---|---|
| SDK | all, including status | all, including status | all, including status | private keys only |
| Portal | `get`, `list`, `delete` | `create`, `get`, `list` | — | none |
| Provider Administrators | all except status | `get`, `list`, `delete` | all except status | none |
| Provider's controllers | `get`, `list`, `watch` | — | — | — |

**Keep these resources as durable as consumer data.** If they survive what consumer data survives, a rebuilt cluster only needs them back; otherwise the administrators import `Connection`s (see "Administration").

## Components

### Connection API

The handler for the one plain-HTTPS endpoint:
- **Registration**: verify the token against its `OnboardingToken`, spend it, apply the idempotency rules, create the `Connection` (Owner, `clusterName`, front-proxy URL, Connection Secret hash, the Pinned Bundle, the Server Keys returned), return `connectionID` and Server Keys.
- **Orphan expiry**: a `Connection` with an empty `lastSeen` at the end of the retry window is deleted; it's a Registration whose result PM never stored. A Connection that was used once is never expired for being idle, however long its Provider or PM is away.

### Control Sessions

- **WebSocket upgrade, then the Inner TLS server** with the Connection's Server Key (see "Server Keys"), speaking HTTP/1.1 with keep-alive. The version is negotiated through ALPN ([01 Protocol, "Versioning"](./xxx-notes-011-v5-impl-01-protocol.md)).
- **Bearer check** on each request, against `secretHash` only. **The first failed check closes the Control Session**, so every guess costs a new WebSocket and handshake.
- **Handlers**: status, Pinned Bundle replacement, delete ([01 Protocol, "Control Sessions"](./xxx-notes-011-v5-impl-01-protocol.md)). None changes the Connection Secret. Delete deletes the `Connection`.
- **Pinned Bundle**: a PEM bundle, parsed strictly (only `CERTIFICATE` blocks, no PEM headers, only whitespace around blocks, no duplicates, every certificate a CA, bounded size); Go's `AppendCertsFromPEM` isn't used, since it skips what it can't parse. Replacement is compare-and-swap on the Pinned Bundle's `etag`, a hash over its certificates' fingerprints, computed from `spec.pinnedBundle`: a missing or stale `If-Match` is refused with `412`, the same set of certificates is a no-op ([01 Protocol, "Pinned Bundle"](./xxx-notes-011-v5-impl-01-protocol.md)). The `etag` and fingerprints are reported in `GET`. The bundle is only ever accepted on the control channel or at Registration, never from a Session. A push is accepted even if every pinned CA has expired: it's authenticated by the Connection Secret, not by the pinned CAs.

### The Connection's ProviderEndpoint

For each Connection, the SDK watches its `ProviderEndpoint` (`default`, cluster-scoped, in the Connection's Provider Workspace, through the front-proxy) over the Connection's own Sessions. Two things come from it:
- **`pendingSecretHash`**: when set and different from `secretHash`, it becomes the `Connection`'s `secretHash` and the old Connection Secret stops working ([01 Protocol, "Changing the Connection Secret"](./xxx-notes-011-v5-impl-01-protocol.md)). This is the only way the hash changes after Registration.
- **`serverKeys`**: which Server Keys the PM-side has pinned, kept as `listedServerKeys` (see "Server Keys").

The `ProviderEndpoint` is trusted because of how it arrives: over a Session whose Connector Certificate verified against the Connection's Pinned Bundle and whose Hello named the Connection's `clusterName`, from an object only PM can write.

### Session server

1. **Refuse unknown or deleted Connection IDs before the upgrade**, and callers beyond the handshake limits with `503` ([01 Protocol, "Endpoints at a glance"](./xxx-notes-011-v5-impl-01-protocol.md)).
2. **Accept the WebSocket** and run the **Inner TLS server**, requiring a client certificate issued directly by a CA in the Connection's Pinned Bundle, with the `clientAuth` EKU ([01 Protocol, "Inner layer"](./xxx-notes-011-v5-impl-01-protocol.md)). A short timeout covers the whole handshake.
3. **Read the Hello**; close the Session if its `clusterName` isn't the Connection's.
4. **Start yamux**; register the Session under its Connection, with the hosts it serves (from the Hello's `hosts`) plus the front-proxy.
5. **Honour go-away**: stop opening Streams on a Session the Connector is replacing.
6. **Close all Sessions** of a Connection that's deleted.
7. **Re-check open Sessions when the Pinned Bundle changes** (a push or a re-pin): a Session whose client certificate, kept from its handshake, no longer chains to a CA in the new bundle gets go-away, a short grace period for in-flight requests (e.g. 30 seconds, configurable), then is closed. Only the issuer counts, not the certificate's expiry. This ends Sessions opened with a leaked CA or Connector Certificate key once PM drops it; why it matters, and its limits, are in [01 Protocol, "Pinned Bundle"](./xxx-notes-011-v5-impl-01-protocol.md).

### Transport

- **A per-Connection `http.Transport`** whose dial hook opens a yamux Stream on a Session that serves the target host and returns it. The target address only chooses the Session; the request goes as plain HTTP/1.1 on the Stream, with the real kcp URL and `Host`, which the Connector checks.
- **Session choice**: any Session for the front-proxy; that shard's Sessions for a virtual-workspace host; spread across several Sessions serving the same host.
- **No TLS settings on the client config**: confidentiality comes from the Inner TLS, and client-go refuses both a custom transport and TLS options.
- **Per-host bookkeeping** of request outcomes, in memory, which feeds status and `Ready`.

### Connections for multicluster-runtime

- **A registry of Connections**, built from `Connection`s, with events (added, removed) and, per Connection, a client config: front-proxy URL as host plus the transport.
- **A helper** that, for each Connection, creates a multicluster-runtime endpoint-slice provider per endpoint slice the Provider names, and composes them with cluster names prefixed by `connectionID`.
- **Connections keep their `connectionID`** through Secret Changes, repairs and administrator imports, so nothing changes for the Provider's controllers.

### Status

Answers `GET` from memory with, per host the Provider's clients actually use: live Sessions, whether recent requests reached kcp, and the last error, without sensitive detail ([01 Protocol, "Status"](./xxx-notes-011-v5-impl-01-protocol.md)).

### Server Keys

- **Generated by the SDK**: on first start, with no `ServerKey`, it creates one; for every `ServerKey` without `signerRef` and with an empty `status.publicKey`, it generates the key pair into the `Secret` and sets `publicKey`. It never generates a new key under an existing name, since Connections have pinned the old one: a missing `Secret` is `Ready=False` with `KeyMissing`, and the fix is a rotation.
- **KMS or HSM keys** through `signerRef`. The Provider registers a signer factory in its binary, `sdk.RegisterSigner("gcp-kms", func(ctx, key string) (crypto.Signer, error) {…})`, with any `crypto.Signer` implementation (sigstore's KMS packages and `crypto11` for PKCS#11 are examples); the SDK ships no KMS clients. The SDK reads the public key through `Public()`, checks that the Inner TLS accepts its algorithm (e.g. ECDSA P-256 or P-384), and sets `publicKey`. Access to the KMS comes from the Provider's pod identity. Failures are `SignerUnavailable`. On every start the SDK compares `Public()` with `publicKey` and refuses a key it can't prove: `KeyMismatch`.
- **Which key a Connection presents**: every `ServerKey` not past `retireAt` is announced in `GET`; each Connection presents the newest announced key in its `listedServerKeys`.
- **Rotation only on demand** (see "Administration"), for example after a suspected key compromise. Nothing rotates on a schedule. There's no plain-HTTPS key endpoint ([01 Protocol, "Server Keys"](./xxx-notes-011-v5-impl-01-protocol.md)).

### Metrics

Prometheus collectors, which the Provider adds to the registry it already serves (controller-runtime's, for most):
- authentication failures on Control Sessions and Sessions;
- pushed Pinned Bundles refused, by reason;
- Sessions by state, and Sessions closed, by reason (go-away, Edge cut, bundle re-check, Connection removed);
- handshakes refused before or during the upgrade, by reason (unknown ID, limits, timeout);
- Connections;
- `providersdk_server_key_connections{key, listed}`: how many `Connection`s list each `ServerKey`, from the SDK's cache. A documented alert rule fires when a key with `retireAt` set is still missing from Connections seen in the last day, within a day of `retireAt`.

**No `connectionID` labels**: with thousands of Connections, they'd explode cardinality. Per-Connection detail is in the resources, their Events, the logs and the status `GET`.

### WebSocket framing

The SDK enforces the protocol's framing on every Session and Control Session ([01 Protocol, "WebSocket framing"](./xxx-notes-011-v5-impl-01-protocol.md)): binary messages only, at most 32 KB each, outer pings at the protocol's interval. Providers can't configure it away.

### Administration

Runbooks for Provider Administrators, for cases no protocol call can fix safely. Everything is `kubectl` on the resources; the SDK's controllers react. Inputs are exchanged out-of-band with the PM Administrators.
- **Re-pin a Connection's Pinned Bundle**, when no Session verifies (for example, a bundle that missed a whole Rollover while the Connection also lost its Connection Secret): edit `spec.pinnedBundle`. Once Sessions work again, PM repairs the Connection Secret through the `ProviderEndpoint`.
- **Import a Connection** after the Provider-side lost its state: apply a `Connection` with its `connectionID`, Owner, `clusterName`, front-proxy URL and the Pinned Bundle. No Connection Secret hash: `platform-mesh-operator` notices the failed authentication and changes the Connection Secret through the `ProviderEndpoint`. New Server Keys go to PM Administrators, since PM never fetches them over plain HTTPS.
- **Rotate the Server Key**:
  1. create a new `ServerKey`: without a KMS, a manifest with just `generateName: sk-` (`kubectl create`); with one, create the key in the KMS first and name its version in `signerRef`;
  2. the SDK announces it; `platform-mesh-operator` adds it to each Connection's `ProviderEndpoint` on its next poll ([03 Operator, "Writing ProviderEndpoints"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)), and each Connection moves to it;
  3. watch `providersdk_server_key_connections`, and list the Connections still missing the new key with their `lastSeen`:
     ```
     kubectl get connections -n <ns> -o json | jq -r '.items[]
       | select(.status.listedServerKeys | index("sk-2026-10") | not)
       | [.metadata.name, .spec.registration.owner, .status.lastSeen] | @tsv'
     ```
  4. set `retireAt` on the old key: soon for a planned rotation, a short deadline after a suspected compromise, accepting that Connections not reached by then lose their Sessions until a PM Administrator supplies the new key (`status.publicKey`);
  5. delete the old `ServerKey` once retired, never before; its `Secret` follows.

## Replicas

Sessions for different shards can land on different replicas behind the Provider's ingress, while the replica running a Connection's controllers needs all of them: the dial hook only reaches Sessions held by its own process. Controllers therefore run in the same process as the Connection API server.

### This delivery: one active replica

- **Leader election** picks the active replica; the vendor's controllers are leader-elected the usual controller-runtime way, with their reconcile logic unchanged. The SDK's controllers run on the leader.
- **Standby replicas answer Session and Control Session upgrades with `503` and `Retry-After: 5`.** Connectors use the value as the lower bound for their normal reconnect backoff, which grows on repeated failures, so a Provider with a longer leader-election Lease just sees a few more `503`s during a failover ([01 Protocol, "Errors and retries"](./xxx-notes-011-v5-impl-01-protocol.md)); with two or three replicas, Sessions are back within roughly 10–20 seconds after a failover, and watches relist as after any restart.
- **Constraints on the vendor**: nothing beyond leader election (Lease RBAC, which leader-elected operators already have) and an Edge that meets the quickstart's checklist ([08 Quickstart, "Connection API server"](./xxx-notes-011-v5-impl-08-provider-quickstart.md)). No extra Services, routes or permissions.
- **What vendors accept**: vertical scaling only (one process holds every Connection's caches and traffic), and a reconnect lottery plus relists after a failover.

### Later: horizontal scale

An opt-in mode for vendors who need more than one active replica is a [follow-up](./xxx-notes-011-v5-impl-todo.md). It needs no PM-side work: the protocol already makes no assumption about which replica a Session reaches.

## Requirements on Providers

Documented for Provider authors ([08 Quickstart, "Documentation for Provider authors"](./xxx-notes-011-v5-impl-08-provider-quickstart.md)):
- **Service endpoint slices live in the Provider Workspace**, for exports in that workspace. Connectors open Sessions on a shard only if such a slice lists it ([05 Connector, "Which Sessions"](./xxx-notes-011-v5-impl-05-connector.md)).
- **The Edge meets the quickstart's checklist** ([08 Quickstart, "Connection API server"](./xxx-notes-011-v5-impl-08-provider-quickstart.md)); `platform-mesh-operator` reports violations it sees on the `Provider` with a labelled message.
- **One active replica** of the Connection API server and its controllers, by leader election (see "Replicas").
- **Rate limits** on Registration per Owner: SDK defaults, adjustable.
- **Consumer data is keyed with the `connectionID`**, never only with the consumer's logical cluster and the object's UID (see "Principles").
- **Adopt, don't create.** A controller whose local state is missing finds existing external resources through the kcp object (an ID in its `status` or annotations) or through the external resource itself (tags naming the `connectionID`, cluster and UID), before creating anything.
- **Removing a Connection never deletes consumer data.** Only deleting a kcp object does, through its finalizer. A Connection can be removed and come back (deleted in the portal by mistake, imported after a disaster), and consumers' bindings outlive it.
- **Bootstrap only creates or adopts.** It never deletes and recreates the APIExport or its identity `Secret`; a new identity would break every consumer's binding.

## Testing support

- **A fake PM-side** for Providers' tests: registers, opens Sessions with test certificates, sends requests through them.
- **A test Provider-side server** `platform-mesh-operator` and Connector use in their own tests ([03](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md), [05](./xxx-notes-011-v5-impl-05-connector.md)).
- **The protocol's test vectors** ([01 Protocol, "Deliverables"](./xxx-notes-011-v5-impl-01-protocol.md)).
- **envtest** for the SDK's own controllers and handlers against real CRDs.

## Work breakdown

1. Module scaffold, protocol types, test vectors.
2. The CRDs (`Connection`, `OnboardingToken`, `ServerKey`) with their CEL rules, selectable fields and printer columns.
3. The token helper; metrics.
4. Connection API handlers.
5. Control Sessions (Inner TLS server, bearer check, compare-and-swap on the bundle).
6. Session server (handshake limits, inner mutual TLS, Hello, yamux, registry, re-checking Sessions on a bundle change).
7. Transport and Connection registry; multicluster-runtime helper.
8. The SDK's controllers: `Connection` (finalizer, `Ready`, `lastSeen`, orphan expiry, Events), `OnboardingToken` (garbage collection), `ServerKey` (generation, signers, checks); the `ProviderEndpoint` watch (`pendingSecretHash`, `serverKeys`); status.
9. Framing enforcement.
10. Fake PM-side and test server.
11. Unit tests for the rules above, with envtest and the protocol's test vectors.
12. Documentation for Provider authors, including the requirements on consumer data and the administration runbooks.

## Releases and protocol versions

- **Its own semver line**, tagged `provider-sdk/vX.Y.Z` through the monorepo's `cmd/release` as a library component (no image), released when the SDK changes, not with every PM release.
- **A new protocol version is a new SDK minor version**, supporting the new and the previous protocol version ([01 Protocol, "Versioning"](./xxx-notes-011-v5-impl-01-protocol.md)). Dropping a protocol version is a breaking change for Providers that haven't upgraded, announced one release ahead.
- **The API version of `provider-sdk.platform-mesh.io` evolves separately**, by Kubernetes' rules (`v1alpha1`, then `v1beta1`, then `v1`, with conversion), and is announced in the same release notes. The resources store the SDK's own representation, not wire messages, so a protocol change rarely needs one.
- **Signals that a Provider should upgrade**:
  - the SDK logs a warning when a PM-side offers a newer protocol version than it supports;
  - `platform-mesh-operator` sets a **`ProtocolOutdated`** warning condition on the `Provider` while its Control Sessions still negotiate the previous version ([03 Operator, "Health"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)), so the Owner sees it too.

## Decided

- **One active replica** (leader election, standbys answer `503` with `Retry-After: 5`); horizontal scale is a follow-up needing no PM-side work.
- **Kubernetes-API-first**: the SDK's state is `Connection`, `OnboardingToken` and `ServerKey` in `provider-sdk.platform-mesh.io/v1alpha1`, the only source of truth; deleting a `Connection` is the delete; the only `Secret`s hold private keys.
- **Events, not hooks**: Providers watch `Connection`s and their Events. Operational signals are metrics, without `connectionID` labels.
- **Server Keys**: generated by the SDK or behind a registered `crypto.Signer`; rotated only on demand, with `kubectl` and `retireAt`.
- **Administration** is `kubectl` on the resources, documented as runbooks.
- **Go only.** The protocol specification and test vectors exist, but third-party implementations aren't a supported target.
- **Pinned Bundle**: PEM, parsed strictly.
- **Recovery**: Connection Secrets change only through the `ProviderEndpoint`; recovery beyond that is out-of-band, through the runbooks.
- **Module**: `go.platform-mesh.io/provider-sdk`; independent semver; upgrade signals as above.
