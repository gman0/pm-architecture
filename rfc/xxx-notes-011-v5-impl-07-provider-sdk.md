# v5 implementation plan: SDK

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). The Provider-side of the [protocol](./xxx-notes-011-v5-impl-01-protocol.md), which every Provider runs, so Providers don't need to understand kcp, TLS pinning or the multiplexer. **The SDK is delivered as a component**: `provider-gateway`, an image and a Helm chart PM releases, which the Provider installs in its own cluster next to its operator. It's built from the Go module `go.platform-mesh.io/provider-sdk` at `platform-mesh/provider-sdk`, next to `apis`, `subroutines` and `golang-commons`, which also holds a small library for portals and the test support. **The Provider's operator stays unchanged**: it reaches kcp through `provider-gateway` with an ordinary kubeconfig per Connection, using client-go, multicluster-runtime and kcp-multicluster-provider as it would against kcp directly. The [quickstart](./xxx-notes-011-v5-impl-08-provider-quickstart.md) is its reference user. The design and the alternatives considered are in the [Connection gateway notes](./xxx-notes-011-v5-connection-gateway.md).

## What a Provider gets

- **`provider-gateway`**, installed from PM's chart:
  - **the Connection API server** under its Base URL: Registration, Control Sessions, Sessions;
  - **an in-cluster forward proxy** through which the Provider's operator reaches kcp over each Connection's Sessions;
  - **one kubeconfig `Secret` per Connection**, in its own namespace (see "Kubeconfig per Connection");
  - **its state as Kubernetes resources**: `Connection`, `OnboardingToken` and `ServerKey` in the Provider's own cluster, with Events on each `Connection` (see "API resources"). The Provider's portal, administrators and controllers use them through the Kubernetes API like any other resource;
  - **metrics** for its operators.
- **A token helper** for its portal, in the Go module.

What a Provider still does itself: its portal and user accounts, its operator (which endpoint slices it reconciles, against which kubeconfigs), its ingress, and telling its customers about changes to their Connections.

## Principles

- **No kcp knowledge on the Provider-side.** No kcp tokens, kcp kubeconfigs or kcp CA bundles, and no kcp hostname it would have to keep: the front-proxy has no name on the Provider-side, and requests carry no kcp credentials ([01 Protocol](./xxx-notes-011-v5-impl-01-protocol.md)). The kubeconfigs `provider-gateway` writes point at `provider-gateway`, never at kcp.
- **Everything security-relevant is in `provider-gateway`**, written once by PM and tested with the protocol's test vectors, so Providers can't get it subtly wrong. Nothing protocol-related is compiled into the Provider's operator.
- **The operator uses standard machinery only**: a kubeconfig per Connection, with nothing in it a stock client-go doesn't understand.
- **The Kubernetes API is the only source of truth** for the SDK's state. `provider-gateway` keeps caches and live Sessions in memory, never a second copy of state; everything that has to survive a restart is in resources: its CA (see "Operator proxy"), the Connections' proxy credentials (as hashes), and the resources below.
- **State belongs to its Connection** (`connectionID`), never inherited by another Connection. Consumer data has to be keyed with the `connectionID` too, not only with the consumer's logical cluster and the object's UID: anyone with a portal account can register a Connection with a Connector CA of their own and serve made-up objects that reuse a real consumer's cluster and UID (see "Requirements on Providers").
- **`connectionID`s are stable.** A Connection keeps its ID through Secret Changes and recovery, so state keyed by it stays valid.
- **`connectionID`s are random and unguessable** (e.g. 128 random bits), so probing the Session and control paths can't find valid ones. They aren't secret, and nothing relies on them being so.
- **Only PM changes a Connection Secret**, through the Connection's `ProviderEndpoint` ([01 Protocol, "Changing the Connection Secret"](./xxx-notes-011-v5-impl-01-protocol.md)).
- **A failed authentication changes nothing.** It's logged and counted, never written to a resource or an Event: `lastSeen` and every other field only move on authenticated Control Calls and verified Sessions.
- **Secret values are stored hashed**; comparisons are constant-time; nothing secret is logged.

## API resources

Group `provider-sdk.platform-mesh.io/v1alpha1`, shipped as CRDs with `provider-gateway`'s chart; all namespaced, in one namespace the Provider configures (the SDK's namespace). The group's version is independent of the protocol version: it describes stored objects, not wire messages (see "Releases and protocol versions").

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
    unusedExpiresAt: "2026-10-23T10:00:00Z"    # removed if never reached by then; empty on imports
  pinnedBundle: |                              # the Pinned Bundle, PEM; PM push or administrator re-pin
    -----BEGIN CERTIFICATE-----
    ...
status:
  conditions:
    - type: Ready                              # AsExpected | NoSessions | KcpUnreachable; failures after ~5 min
      status: "True"
      reason: AsExpected
      observedGeneration: 4
  lastSeen: "2026-10-15T10:00:00Z"             # first contact at once, then hourly; empty: never used
  listedServerKeys: [sk-2026-04, sk-2026-10]
  secretHash: "sha256:..."                     # current connection secret's hash; only the SDK writes it
  proxyCredentialHash: "sha256:..."            # the operator's proxy credential for this connection
```

- **Created at Registration** by `provider-gateway`. `spec.registration` is immutable (CEL: `self == oldSelf`); `spec.registration.owner` is a selectable field, so a portal lists one Owner's Connections with a field selector.
- **`spec.registration.unusedExpiresAt`**: fixed at Registration, the end of the retry window, and returned to PM in the Registration response (see "Connection API").
- **`spec.pinnedBundle`** is what Sessions are verified against. A PM push writes it ([01 Protocol, "Pinned Bundle"](./xxx-notes-011-v5-impl-01-protocol.md)), and so does an administrator re-pinning; either change makes `provider-gateway` re-check open Sessions (see "Session server").
- **`Ready`** says whether the Provider Workspace and every virtual-workspace host the operator uses have a Session and reach kcp. It reports `NoSessions` or `KcpUnreachable` only after the state has lasted a few minutes (e.g. 5), so a failover, when a new leader briefly sees no Sessions, doesn't flip every Connection. The detail stays in memory, where `GET` reads it, and in metrics.
- **`lastSeen`**: the last authenticated Control Call or verified Session. The first one is written at once; after that, at most once an hour. Empty means never used, which is what orphan expiry looks for; an old value marks a Connection whose PM Instance went away without `DELETE`.
- **`listedServerKeys`**: the `ServerKey`s PM's `ProviderEndpoint` last listed. Set at Registration to the keys returned, then updated whenever `provider-gateway` reads the `ProviderEndpoint`. The Inner TLS handshake picks the key from it before any Session exists (see "Server Keys").
- **`secretHash`**: the current Connection Secret's SHA-256. In status because only `provider-gateway` writes it; it isn't in a `Secret` because a hash of a 256-bit random Connection Secret reveals nothing, and the same kind of hash is already public in the `ProviderEndpoint`. If it's lost (e.g. a restore without status), PM's next Control Call fails and PM repairs the Connection Secret ([03 Operator, "Secret Changes"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
- **`proxyCredentialHash`**: the hash of the credential in this Connection's kubeconfig `Secret` (see "Kubeconfig per Connection"), so any `provider-gateway` replica can check the operator's tunnels.
- **Deleting the object is the delete**, whoever does it: a portal, an administrator with `kubectl`, `provider-gateway` on PM's `DELETE` call, or orphan expiry. The finalizer closes the Connection's Sessions and Control Sessions, deletes its kubeconfig `Secret`, emits `Removed` and lets the object go. Deleting is final.
- **Events**: `Registered`, `SecretChanged`, `PinnedBundleChanged` (with the number of Sessions closed), `Removed` (with the reason) and `AdministrativeChange`. A Provider that tells its customers about changes watches `Connection`s and their Events, like any other resource. Failed authentications aren't Events: they go to logs and metrics only. `provider-gateway` also logs every event, never with secrets.

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
  serverKey: sk-2026-10                        # the ServerKey whose key-sha256 is in the token
status:
  registration:                                # present: spent
    connectionID: c-7f3k9q2mx8w4
    idempotencyKey: "<ProviderOnboardingRequest UID>"
    at: "2026-10-15T09:12:00Z"
```

- **Issued by the Provider's portal** with the token helper, `NewOnboardingToken(baseURL, owner, ttl)` from the Go module, which picks the newest `ServerKey` that is `Ready` and has no `retireAt`, and returns the token string `pm1.<base URL>.<keyfp>.<id>.<secret>`, shown once, and the object, which the portal creates. `keyfp` is that key's `key-sha256` ([01 Protocol, "Onboarding Token"](./xxx-notes-011-v5-impl-01-protocol.md)). The plaintext token is never stored. A portal in another language reproduces the helper from the protocol specification: it's a random secret, its hash, and the token format.
- **Usable only while its `ServerKey` is**: present, `Ready`, and without `retireAt`. Setting `retireAt` on a key refuses the unspent tokens issued over it; their Owners ask the portal for new ones.
- **Spent** when `status.registration` is set, by compare-and-swap, so a token can't be spent twice. Registration retries that match get the same answer, for 7 days by default (see "Connection API").
- **Garbage-collected by `provider-gateway`**: unspent tokens at `expiresAt`, spent ones once the retry window has passed since `registration.at`.

### ServerKey

```yaml
apiVersion: provider-sdk.platform-mesh.io/v1alpha1
kind: ServerKey
metadata:
  name: sk-2026-10
  namespace: wild-west-system
spec:
  retireAt: "2026-10-22T00:00:00Z"             # optional
status:
  conditions:
    - type: Ready                              # AsExpected | KeyMissing | KeyMismatch
      status: "True"
      reason: AsExpected
      observedGeneration: 2
  publicKey: "<PEM SPKI>"
  fingerprint: key-sha256:...                  # SHA-256 of the SPKI (DER); printer column
```

The private key is in a `Secret` of the same name, with an owner reference to the `ServerKey`, readable only by `provider-gateway`'s identity. See "Server Keys".

### Access

RBAC in the SDK's namespace:

| Identity | `Connection` | `OnboardingToken` | `ServerKey` | `Secret`s |
|---|---|---|---|---|
| `provider-gateway` | all, including status | all, including status | all, including status | private keys, its CA and the kubeconfig `Secret`s (its namespace) |
| Portal | `get`, `list`, `delete` | `create`, `get`, `list` | `get`, `list` (to pick the key) | none |
| Provider Administrators | all except status | `get`, `list`, `delete` | all except status | none |
| Provider's operator | `get`, `list`, `watch` | — | — | the kubeconfig `Secret`s, read-only, through a Role the Provider adds in the SDK's namespace |

**Keep these resources as durable as consumer data.** If they survive what consumer data survives, a rebuilt cluster only needs them back; otherwise the administrators import `Connection`s (see "Administration").

## Kubeconfig per Connection

`provider-gateway` writes one kubeconfig `Secret` per Connection into its own namespace, the SDK's, labelled with the `connectionID`. `provider-gateway` writes `Secret`s nowhere else; the Provider's operator reads them through a Role the Provider adds there (`get`, `list`, `watch` on `secrets`). It represents the Connection, not its Sessions: created with the `Connection` at Registration or import, deleted with it, unchanged while Sessions come and go. Whether Sessions are up is the `Connection`'s `Ready`.

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: connection-c-7f3k9q2mx8w4
  namespace: wild-west-system                          # the SDK's namespace
  labels:
    provider-sdk.platform-mesh.io/connection: c-7f3k9q2mx8w4
    app.kubernetes.io/managed-by: provider-gateway
  annotations:
    provider-sdk.platform-mesh.io/cluster-name: 2x8k9abc
stringData:
  kubeconfig: |
    apiVersion: v1
    kind: Config
    clusters:
    - name: c-7f3k9q2mx8w4
      cluster:
        server: https://front-proxy.pm.invalid/clusters/2x8k9abc     # placeholder for the Provider Workspace; never resolved
        proxy-url: https://c-7f3k9q2mx8w4:<credential>@provider-gateway.wild-west-system.svc:3128
        tls-server-name: provider-gateway.wild-west-system.svc       # every handshake verified against the gateway's name
        certificate-authority-data: <provider-gateway's CA>
    users:
    - name: c-7f3k9q2mx8w4
      user: {}                                         # nothing: towards kcp, the Connector adds the SA token
    contexts:
    - name: c-7f3k9q2mx8w4
      context: { cluster: c-7f3k9q2mx8w4, user: c-7f3k9q2mx8w4 }
    current-context: c-7f3k9q2mx8w4
```

- **Only the Provider Workspace's URL.** The virtual-workspace URLs come from the endpoint slices at runtime, as kcp writes them; client-go reaches them through the same proxy.
- **A credential for `provider-gateway` only**, in the proxy URL: generated with the `Connection`, kept as a hash in its status, useless anywhere else, never sent to PM.
- **No kcp credential, no kcp CA, no kcp hostname** other than the placeholder.
- **Changes** only when the `Connection` is created or deleted, when the proxy credential is rotated, or when the CA's trust bundle changes, which is a rare, deliberate operation (see "Operator proxy"). Renewing the serving certificate changes nothing.

## Components

### Connection API

Serves the three endpoints under the Base URL. It **routes by the path below `/v1/connections/`** and isn't bound to one host or path prefix, so a Provider can move its Base URL, or serve it under several names during a move, without changes.

**Registration** (`/v1/connections/register/{tokenID}`):
1. **Before the upgrade**, look up the `OnboardingToken`: it must exist, be unexpired and unspent, and its `ServerKey` must be present, `Ready` and without `retireAt`. Otherwise refuse with a stable status, which PM reports as `TokenRefused`. The handshake limits apply.
2. **Accept the WebSocket** and run the Inner TLS server with the token's `ServerKey`, ALPN `pm-register/1`.
3. **Verify `tokenSecret`** against `tokenHash`, constant-time.
4. **Apply the idempotency rules**: a retry with the same idempotency key, `secretHash` and `clusterName` gets the stored answer; a different one is refused; a changed Connector CA Set on a retry is ignored ([01 Protocol, "Registration"](./xxx-notes-011-v5-impl-01-protocol.md)).
5. **Spend the token and create the `Connection`** (Owner, `clusterName`, `unusedExpiresAt`, Connection Secret hash, the Pinned Bundle, the Server Keys returned), with its proxy credential and kubeconfig `Secret`, then return `connectionID`, Server Keys and `unusedExpiresAt`.

**Orphan expiry**: a `Connection` with an empty `lastSeen` at its `unusedExpiresAt` is deleted, after a fresh, uncached read confirms `lastSeen` is still empty. It's meant for a Registration whose result PM never stored; it also removes a Connection PM stored but never reached within the window, which loses nothing, since it never had a Session. A Connection that was used once is never expired for being idle, however long its Provider or PM is away. An imported `Connection`, without `unusedExpiresAt`, is never expired.

### Control Sessions

- **Accept the upgrade**, for known IDs within the per-Connection and total limits, and for unknown IDs within their own small total limit ([01 Protocol, "Endpoints at a glance"](./xxx-notes-011-v5-impl-01-protocol.md)); beyond them, `503` with `Retry-After`. Slots are released when the handshake ends.
- **Run the Inner TLS server** with the Server Key named in the `PM-Server-Key` header if `provider-gateway` has it, otherwise the Connection's (see "Server Keys"), speaking HTTP/1.1 with keep-alive. The version is negotiated through ALPN ([01 Protocol, "Versioning"](./xxx-notes-011-v5-impl-01-protocol.md)).
- **Bound to the path's `{id}`**: an inner request for another Connection gets `ConnectionMismatch` and closes the Control Session. It isn't counted as an authentication failure.
- **Unknown Connections are answered inside the Inner TLS**, with the stable "unknown" status, so PM can trust it. Before answering, `provider-gateway` reads the `Connection` uncached; if its cache isn't synced or the Kubernetes API can't be reached, it answers `503` instead. Standbys answer `503` before the upgrade (see "Replicas").
- **Bearer check** on each request, against `secretHash` only. **The first failed check closes the Control Session**, so every guess costs a new WebSocket and handshake.
- **Handlers**: status, Pinned Bundle replacement, delete ([01 Protocol, "Control Sessions"](./xxx-notes-011-v5-impl-01-protocol.md)). None changes the Connection Secret. Delete deletes the `Connection`.
- **Pinned Bundle**: a PEM bundle, parsed strictly (only `CERTIFICATE` blocks, no PEM headers, only whitespace around blocks, no duplicates, every certificate a CA, at least one certificate and at least one unexpired, bounded size); Go's `AppendCertsFromPEM` isn't used, since it skips what it can't parse. The same rules apply at Registration and to administrators' re-pins. Replacement is compare-and-swap on the Pinned Bundle's `etag`, a hash over its certificates' `cert-sha256` fingerprints, computed from `spec.pinnedBundle`: a missing or stale `If-Match` is refused with `412`, the same set of certificates is a no-op ([01 Protocol, "Pinned Bundle"](./xxx-notes-011-v5-impl-01-protocol.md)). The `etag` and fingerprints are reported in `GET`. The bundle is only ever accepted on the control channel or at Registration, never from a Session. A push is accepted even if every pinned CA has expired: it's authenticated by the Connection Secret, not by the pinned CAs.

### The Connection's ProviderEndpoint

For each Connection, `provider-gateway` watches its `ProviderEndpoint` (`default`, cluster-scoped, in the Connection's Provider Workspace, through the front-proxy) over the Connection's own Sessions. It's applied only if its `connectionID` is the Connection's; otherwise it's ignored and logged. Two things come from it:
- **`pendingSecretHash`**: when set and different from `secretHash`, it becomes the `Connection`'s `secretHash` and the old Connection Secret stops working ([01 Protocol, "Changing the Connection Secret"](./xxx-notes-011-v5-impl-01-protocol.md)). This is the only way the hash changes after Registration.
- **`serverKeys`**: which Server Keys the PM-side has pinned, kept as `listedServerKeys` (see "Server Keys").

The `ProviderEndpoint` is trusted because of how it arrives: over a Session whose Connector Certificate verified against the Connection's Pinned Bundle and whose Hello named the Connection's `connectionID` and `clusterName`, from an object only PM can write, naming this Connection.

### Session server

1. **Refuse unknown or deleted Connection IDs before the upgrade**, and callers beyond the handshake limits with `503` ([01 Protocol, "Endpoints at a glance"](./xxx-notes-011-v5-impl-01-protocol.md)).
2. **Accept the WebSocket** and run the **Inner TLS server**, requiring a client certificate issued directly by a CA in the Connection's Pinned Bundle, with the `clientAuth` EKU ([01 Protocol, "Inner layer"](./xxx-notes-011-v5-impl-01-protocol.md)). A short timeout covers the whole handshake.
3. **Read the Hello**; close the Session unless its `connectionID` is the path's `{id}` and its `clusterName` is the Connection's. This check is essential, not defense in depth: one Connector Certificate serves every Connection of a PM Instance ([01 Protocol, "Hello"](./xxx-notes-011-v5-impl-01-protocol.md)).
4. **Start yamux**; register the Session under its Connection, with the virtual-workspace hosts it serves (from the Hello's `hosts`). Every Session also serves the Provider Workspace.
5. **Honour go-away**: stop opening Streams on a Session the Connector is replacing.
6. **Close all Sessions** of a Connection that's deleted.
7. **Re-check open Sessions when the Pinned Bundle changes** (a push or a re-pin): a Session whose client certificate, kept from its handshake, no longer chains to a CA in the new bundle gets go-away, a short grace period for in-flight requests (e.g. 30 seconds, configurable), then is closed. Only the issuer counts, not the certificate's expiry. This ends Sessions opened with a leaked CA or Connector Certificate key once PM drops it; why it matters, and its limits, are in [01 Protocol, "Pinned Bundle"](./xxx-notes-011-v5-impl-01-protocol.md).

### Operator proxy

How the Provider's operator reaches kcp: an in-cluster HTTPS forward proxy, `provider-gateway`'s Service on a separate port, which only the active replica serves.
1. **`CONNECT host:443`** from client-go, with `Proxy-Authorization: Basic` from the kubeconfig's proxy URL. The credential names the Connection and is checked against `proxyCredentialHash`; a wrong or missing one is refused with `407`, an unknown Connection with `404`.
2. **Pick a Session**: for the workspace placeholder, any Session of the Connection; for a virtual-workspace host, a Session whose Hello announced it. A host no Session of that Connection serves is refused with `404`; no Session at the moment gets `503`, and client-go retries.
3. **Open a Stream** on that Session and answer `200`.
4. **Terminate the TLS inside the tunnel** with `provider-gateway`'s own serving certificate. client-go verifies it against the kubeconfig's `tls-server-name`, the gateway's own name, so one certificate covers every host. ALPN is `http/1.1` only: a Stream carries one HTTP/1.1 connection to the Connector, whose server doesn't speak h2c.
5. **Copy bytes** both ways between the tunnel and the Stream, without parsing HTTP. The Connector routes, authorizes and adds the SA token as for any Stream ([05 Connector, "Request pipeline"](./xxx-notes-011-v5-impl-05-connector.md)); its responses, refusals included, pass through unchanged.

- **One tunnel is one Stream**: client-go's pooling, keep-alive and separate connections for upgrades work as against kube-apiserver.
- **The serving certificate and its CA**, the standard way for in-cluster services: a long-lived CA that clients trust and that practically never changes, issuing short-lived serving certificates that renew on their own. Kubeconfigs carry the CA, so renewals need nothing from the operator.
  - **With cert-manager** (the chart's default): a `SelfSigned` Issuer, a long-lived CA `Certificate`, a CA Issuer, and a serving `Certificate` for `provider-gateway`'s Service names. `provider-gateway` mounts the serving certificate's `Secret`, reloads it when cert-manager renews it, and takes the trust bundle from its `ca.crt`.
  - **Without cert-manager** (a chart value): `provider-gateway` generates a CA once into a `Secret` in its namespace, in the same layout, without expiry, and issues serving certificates from it in memory.
  - **Changing the CA** is a rare, deliberate operation, not a rotation the SDK performs. The overlap, the old and the new CA trusted side by side, is the ecosystem's job (e.g. trust-manager); `provider-gateway` rewrites the kubeconfig `Secret`s whenever the trust bundle changes.
- **Bookkeeping** of request outcomes for the Provider Workspace and per virtual-workspace host, in memory, which feeds status and `Ready`. `provider-gateway` terminates the tunnel's TLS, so it can read the status line of each HTTP/1.1 response as it passes, without changing or buffering anything; the Connector's `502` and `504` mean kcp wasn't reached.

### Status

Answers `GET` from memory with what the operator actually uses: `workspace` (the Provider Workspace, served by every Session) and `virtualWorkspaces` (per host); for each, live Sessions, whether recent requests reached kcp, and the last error, without sensitive detail ([01 Protocol, "Status"](./xxx-notes-011-v5-impl-01-protocol.md)).

### Server Keys

- **Generated by `provider-gateway`**: on first start, with no `ServerKey`, it creates one; for every `ServerKey` with an empty `status.publicKey`, it generates the key pair (ECDSA P-256) into the `Secret` and sets `publicKey` and `fingerprint`. On every start it compares the `Secret`'s public key with `publicKey`, and refuses a key it can't prove: `KeyMismatch`. It never generates a new key under an existing name, since Connections have pinned the old one: a missing `Secret` is `Ready=False` with `KeyMissing`, and the fix is a rotation.
- **KMS or HSM keys aren't supported yet**, and can be added later without protocol, API-breaking or PM-side changes: the PM-side pins Server Keys by their public key and never sees where the private key lives, a key source would be a new optional field on `ServerKey`, and moving to a KMS-held key is a normal rotation to a new `ServerKey`. To keep that easy, the Inner TLS uses each Server Key as a `crypto.Signer`, never as raw key bytes; a `ServerKey`'s key source is immutable, so a different source means a new `ServerKey`; and keys are ECDSA P-256, which every common KMS supports.
- **Which key a Connection presents**: every `ServerKey` not past `retireAt` is announced in `GET`; on Sessions, each Connection presents the newest announced key in its `listedServerKeys`; on Control Sessions, the key PM names in `PM-Server-Key`, if `provider-gateway` has it; at Registration, the token's key.
- **Tokens are issued and used only over keys without `retireAt`**, so a rotation never leaves PM pinning a key the Provider is about to drop.
- **Rotation only on demand** (see "Administration"), for example after a suspected key compromise. Nothing rotates on a schedule. There's no plain-HTTPS key endpoint ([01 Protocol, "Server Keys"](./xxx-notes-011-v5-impl-01-protocol.md)).

### Metrics

Prometheus collectors on `provider-gateway`'s metrics endpoint, with a `ServiceMonitor` in its chart:
- authentication failures on Control Sessions and Sessions, and `ConnectionMismatch`;
- Registrations refused, by reason (token unknown, expired, spent, key unusable);
- pushed Pinned Bundles refused, by reason;
- Sessions by state, and Sessions closed, by reason (go-away, Edge cut, bundle re-check, Connection removed);
- handshakes refused before or during the upgrade, by reason (unknown ID, limits for known IDs, the limit for unknown IDs, timeout);
- operator tunnels open, and refused, by reason (credential, unknown host, no Session);
- Connections;
- `providersdk_server_key_connections{key, listed}`: how many `Connection`s list each `ServerKey`, from `provider-gateway`'s cache. A documented alert rule fires when a key with `retireAt` set is still missing from Connections seen in the last day, within a day of `retireAt`.

**No `connectionID` labels**: with thousands of Connections, they'd explode cardinality. Per-Connection detail is in the resources, their Events, the logs and the status `GET`.

### WebSocket framing

`provider-gateway` enforces the protocol's framing on every Registration, Session and Control Session ([01 Protocol, "WebSocket framing"](./xxx-notes-011-v5-impl-01-protocol.md)): binary messages only, at most 32 KB each, outer pings at the protocol's interval. Providers can't configure it away.

### Administration

Runbooks for Provider Administrators, for cases no protocol call can fix safely. Everything is `kubectl` on the resources; `provider-gateway`'s controllers react. Inputs are exchanged out-of-band with the PM Administrators.
- **Re-pin a Connection's Pinned Bundle**, when no Session verifies (for example, a bundle that missed a whole Rollover while the Connection also lost its Connection Secret): edit `spec.pinnedBundle`. Once Sessions work again, PM repairs the Connection Secret through the `ProviderEndpoint`.
- **Import a Connection** after the Provider-side lost its state: apply a `Connection` with its `connectionID`, Owner, `clusterName` and the Pinned Bundle. No Connection Secret hash: `platform-mesh-operator` notices the failed authentication and changes the Connection Secret through the `ProviderEndpoint`. New Server Keys go to PM Administrators, since PM never fetches them over plain HTTPS. `provider-gateway` writes the Connection's kubeconfig `Secret` anew.
- **Move to a new Base URL**: serve the Connection API under the new URL as well, then ask the PM Administrators to change each Connection's Base URL ([03 Operator, "Writing ProviderEndpoints"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)); if the new hostname's certificate comes from a different private CA, tell Owners to add it to their `caBundle` first. Stop serving the old URL once no Session arrives there.
- **Rotate the Server Key**:
  1. create a new `ServerKey`, a manifest with just `generateName: sk-` (`kubectl create`);
  2. `provider-gateway` announces it; `platform-mesh-operator` adds it to each Connection's `ProviderEndpoint` on its next poll ([03 Operator, "Writing ProviderEndpoints"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)), and each Connection moves to it;
  3. watch `providersdk_server_key_connections`, and list the Connections still missing the new key with their `lastSeen`:
     ```
     kubectl get connections -n <ns> -o json | jq -r '.items[]
       | select(.status.listedServerKeys | index("sk-2026-10") | not)
       | [.metadata.name, .spec.registration.owner, .status.lastSeen] | @tsv'
     ```
  4. set `retireAt` on the old key: soon for a planned rotation, a short deadline after a suspected compromise, accepting that Connections not reached by then lose their Sessions until a PM Administrator supplies the new key (`status.publicKey`). From then on, new tokens use the new key, and unspent tokens issued over the old one are refused;
  5. delete the old `ServerKey` once retired, never before; its `Secret` follows.
- **Rotate an operator's proxy credential**: annotate the `Connection` (e.g. `provider-sdk.platform-mesh.io/rotate-proxy-credential`); `provider-gateway` writes a new credential into the kubeconfig `Secret`, accepts both until the old one is unused, then drops it.

## Replicas

### `provider-gateway`: one active replica

- **Leader election** picks the active replica. Sessions for different shards can land on different replicas behind the Provider's ingress, while every operator tunnel needs all of a Connection's Sessions, so one replica holds them all.
- **Standby replicas answer Registration, Session and Control Session upgrades with `503` and `Retry-After: 5`.** Connectors use the value as the lower bound for their normal reconnect backoff, which grows on repeated failures ([01 Protocol, "Errors and retries"](./xxx-notes-011-v5-impl-01-protocol.md)); with two or three replicas, Sessions are back within roughly 10–20 seconds after a failover.
- **Standbys fail readiness on the operator proxy's port**, so the operator's Service reaches only the leader. On a failover, open tunnels break; client-go reconnects and informers resume their watches, or relist if kcp no longer has the resource version.
- **Every piece of state that must survive** is in resources (see "Principles"), so the new leader serves the same kubeconfigs, credentials and certificate.
- **Constraints on the vendor**: an Edge that meets the quickstart's checklist ([08 Quickstart, "Connection API server"](./xxx-notes-011-v5-impl-08-provider-quickstart.md)), routing the Base URL to `provider-gateway`'s Service. No extra permissions beyond the chart's.

### The Provider's operator

Independent of `provider-gateway`: its own replicas and leader election, the usual controller-runtime way. A restart of either doesn't restart the other; a `provider-gateway` failover only breaks the operator's open connections, and its caches stay warm.

### Later: horizontal scale

An opt-in mode for more than one active `provider-gateway` replica is a [follow-up](./xxx-notes-011-v5-impl-todo.md). It needs no PM-side work: the protocol already makes no assumption about which replica a Session reaches.

## Requirements on Providers

Documented for Provider authors ([08 Quickstart, "Documentation for Provider authors"](./xxx-notes-011-v5-impl-08-provider-quickstart.md)):
- **Install `provider-gateway`** from PM's chart, and route the Base URL's connection paths to it.
- **The operator uses the kubeconfig `Secret`s**: one per Connection, in the SDK's namespace, read through a Role the Provider adds there, loaded with standard client-go (`clientcmd`), with kcp-multicluster-provider's APIExport provider per Connection, cluster names prefixed by the `connectionID` from the `Secret`'s label. Any language whose Kubernetes client supports `proxy-url` and `tls-server-name` works.
- **Connection pooling.** client-go can't share transports between configs that use a proxy: it caches transports by TLS settings, and a proxy is a function it can't compare. So every client built from these kubeconfigs gets its own connection pool, and every new connection is a tunnel and a Stream. kcp-multicluster-provider builds clients per engaged consumer cluster, so after an operator restart or relist, a reconcile burst across many consumer clusters opens many tunnels at once, each counting against the Connector's per-Connection Stream limit. An operator with many consumer clusters can avoid it with one shared transport per Connection, built from the kubeconfig and set as `rest.Config.Transport` (a few lines, shown in the quickstart's README); it's an optimisation, not a requirement. The details are in the [Connection gateway notes](./xxx-notes-011-v5-connection-gateway.md).
- **Service endpoint slices live in the Provider Workspace**, for exports in that workspace. Connectors open Sessions on a shard only if such a slice lists it ([05 Connector, "Which Sessions"](./xxx-notes-011-v5-impl-05-connector.md)).
- **Only the Provider Workspace and the APIExport virtual workspace are reachable**, with logical cluster names, never workspace paths; other kcp virtual workspaces aren't available to Remote Providers ([05 Connector, "Authorization and policy"](./xxx-notes-011-v5-impl-05-connector.md)).
- **RBAC in the Provider Workspace is restricted**: role bindings may name only ServiceAccounts in the workspace and kcp's `apis.kcp.io:binding:` subjects, and `Secret`s of type `kubernetes.io/service-account-token` are refused. To let consumers bind its APIExports, a Provider binds the ClusterRole `providers.platform-mesh.io:apiexport-bind` to them ([02 APIs, "Policies in Provider Workspaces"](./xxx-notes-011-v5-impl-02-apis.md)).
- **A Provider's own admission policies on its exported resources apply in consumers' workspaces too**: kcp applies an export's `ValidatingAdmissionPolicy`s to the bound resources wherever they're bound.
- **The Edge meets the quickstart's checklist** ([08 Quickstart, "Connection API server"](./xxx-notes-011-v5-impl-08-provider-quickstart.md)); `platform-mesh-operator` reports violations it sees on the `Provider` with a labelled message.
- **Rate limits** on Registration per Owner: `provider-gateway` defaults, adjustable in its chart.
- **Consumer data is keyed with the `connectionID`**, never only with the consumer's logical cluster and the object's UID (see "Principles").
- **Adopt, don't create.** A controller whose local state is missing finds existing external resources through the kcp object (an ID in its `status` or annotations) or through the external resource itself (tags naming the `connectionID`, cluster and UID), before creating anything.
- **Removing a Connection never deletes consumer data.** Only deleting a kcp object does, through its finalizer. A Connection can be removed and come back (deleted in the portal by mistake, imported after a disaster), and consumers' bindings outlive it.
- **Bootstrap only creates or adopts.** It never deletes and recreates the APIExport or its identity `Secret`; a new identity would break every consumer's binding.

## Testing support

In the Go module:
- **A fake PM-side** for Providers' tests: registers, opens Sessions with test certificates, sends requests through them, so a Provider can run `provider-gateway` against it in its own CI.
- **A test Provider-side server** `platform-mesh-operator` and the Connector use in their own tests ([03](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md), [05](./xxx-notes-011-v5-impl-05-connector.md)).
- **The protocol's test vectors** ([01 Protocol, "Deliverables"](./xxx-notes-011-v5-impl-01-protocol.md)).
- **envtest** for `provider-gateway`'s controllers and handlers against real CRDs.

## Work breakdown

1. Module scaffold, protocol types, test vectors.
2. The CRDs (`Connection`, `OnboardingToken`, `ServerKey`) with their CEL rules, selectable fields and printer columns.
3. The token helper; metrics.
4. Connection API: routing under `/v1/connections/`; Registration (token checks before the upgrade, Inner TLS with the token's key, idempotency); orphan expiry.
5. Control Sessions (handshake limits including unknown IDs, Inner TLS server with the requested key, binding to the path's ID, "unknown" inside the Inner TLS after an uncached read, bearer check, compare-and-swap on the bundle).
6. Session server (handshake limits, inner mutual TLS, Hello with `connectionID` and `clusterName`, yamux, registry, re-checking Sessions on a bundle change).
7. Operator proxy: `CONNECT`, proxy credentials, Session choice, Streams, TLS with the serving certificate, kubeconfig `Secret`s; the serving certificate from cert-manager or a self-generated CA.
8. `provider-gateway`'s controllers: `Connection` (finalizer, `Ready`, `lastSeen`, orphan expiry, kubeconfig `Secret`, Events), `OnboardingToken` (garbage collection), `ServerKey` (generation, checks); the `ProviderEndpoint` watch (`connectionID` check, `pendingSecretHash`, `serverKeys`); status.
9. Framing enforcement.
10. The image and the chart: leader election, readiness per port, Services for the Base URL and the operator proxy, RBAC, `ServiceMonitor`.
11. Fake PM-side and test server.
12. Unit tests for the rules above, with envtest and the protocol's test vectors.
13. Documentation for Provider authors, including the kubeconfig `Secret`s, the shared-transport optimisation, the requirements on consumer data and the administration runbooks.

## Releases and protocol versions

- **`provider-gateway` (image and chart) and the Go module share one semver line**, tagged `provider-sdk/vX.Y.Z` through the monorepo's `cmd/release`, released when the SDK changes, not with every PM release.
- **A new protocol version is a new minor version**, supporting the new and the previous protocol version ([01 Protocol, "Versioning"](./xxx-notes-011-v5-impl-01-protocol.md)). For Providers, adopting it is a chart upgrade; their operators don't change. Dropping a protocol version is a breaking change for Providers that haven't upgraded, announced one release ahead.
- **The API version of `provider-sdk.platform-mesh.io` evolves separately**, by Kubernetes' rules (`v1alpha1`, then `v1beta1`, then `v1`, with conversion), and is announced in the same release notes. The resources store the SDK's own representation, not wire messages, so a protocol change rarely needs one.
- **Signals that a Provider should upgrade**:
  - `provider-gateway` logs a warning when a PM-side offers a newer protocol version than it supports;
  - `platform-mesh-operator` sets a **`ProtocolOutdated`** warning condition on the `Provider` while its Control Sessions still negotiate the previous version ([03 Operator, "Health"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)), so the Owner sees it too.

## Decided

- **Delivery**: `provider-gateway`, an image and chart from PM, installed by the Provider next to its unchanged operator; the Go module holds the token helper and test support.
- **Operator integration**: one kubeconfig `Secret` per Connection, with `proxy-url` (a credential per Connection) and `tls-server-name`; standard client-go, multicluster-runtime and kcp-multicluster-provider; a shared transport per Connection is an optional optimisation.
- **One active `provider-gateway` replica** (leader election, standbys answer `503` with `Retry-After: 5` and aren't ready for the operator proxy); horizontal scale is a follow-up needing no PM-side work. The operator scales independently.
- **Kubernetes-API-first**: the SDK's state is `Connection`, `OnboardingToken` and `ServerKey` in `provider-sdk.platform-mesh.io/v1alpha1`, the only source of truth; deleting a `Connection` is the delete; the only `Secret`s hold private keys, the serving certificate or self-generated CA, and the kubeconfigs.
- **Events, not hooks**: Providers watch `Connection`s and their Events. Operational signals are metrics, without `connectionID` labels. Failed authentications change nothing and aren't Events.
- **Onboarding Tokens name their Server Key**; Registration runs inside Inner TLS with it.
- **Server Keys**: generated by `provider-gateway` into `Secret`s, ECDSA P-256; KMS or HSM keys later, without breaking changes; rotated only on demand, with `kubectl` and `retireAt`.
- **No front-proxy URL**: the Provider Workspace is reached through a placeholder host in the kubeconfig.
- **Administration** is `kubectl` on the resources, documented as runbooks.
- **Any language** for the operator; `provider-gateway` and the module are Go. The protocol specification and test vectors exist, but third-party implementations of the Provider-side aren't a supported target.
- **Pinned Bundle**: PEM, parsed strictly, never empty.
- **Recovery**: Connection Secrets change only through the `ProviderEndpoint`; recovery beyond that is out-of-band, through the runbooks.
- **Module**: `go.platform-mesh.io/provider-sdk`; one semver line with `provider-gateway`; upgrade signals as above.
