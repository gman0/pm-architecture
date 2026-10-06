# Notes v2: RFC 011 provider onboarding flow

Working notes for [011](./011-provider-onboarding-flow-improvements.md). Not the RFC itself. A streamlined redesign of the [v1 notes](./xxx-notes-011-provider-onboarding-flow-improvements.md); see "Changes from v1".

## Terminology

- **PM-side**: the PM instance (kcp + PM operator). May be behind NAT; dials out.
- **Provider-side**: the service provider (controllers + portal in its runtime cluster). Serves the connection API and reconciles on the PM-side.
- **Connection**: the relationship between one `Provider` object on the PM-side and one connection record on the Provider-side, identified by `connectionID`. Created by registration, ended by deletion on either side.
- **Onboarding token**: a single-use, short-lived token from the Provider's portal. It contains the Provider's URL, so the user never enters a URL separately.
- **Connection secret**: a long-lived bearer credential issued at registration. It authenticates the PM-side for everything after registration. The Provider-side stores only its hash.
- **Connector**: the PM-side component that delivers credentials and holds the tunnel. Where it runs is an open item.
- **`ProviderOnboardingRequest`**: a short-lived PM-side resource that carries the token, creates the `Provider` if needed, and runs registration once. Creating one is also how a user re-onboards.

## Problem

- The Provider-side must reconcile resources on the PM-side, for any number of PM instances.
- **The PM-side may be behind NAT.** Every connection is opened by the PM-side; the Provider-side never dials the PM-side network directly.
- The PM-side owner gets a token from the Provider's portal. Authenticating that user is the portal's business and out of scope.
- Both sides must survive restarts, and outages of either side, without manual steps.

The design keeps three concerns apart, so each stays simple:

1. **Onboarding**: one registration call that turns a token into a connection.
2. **Credentials**: plain HTTPS calls that keep the Provider-side's kcp credentials fresh.
3. **Reachability**: a tunnel that lets the Provider-side reach kcp through NAT.

## Flow overview

1. **Get a token.** In the Provider's portal, the user gets an onboarding token and stores it in a Secret.
2. **Create the request.** The user creates a `ProviderOnboardingRequest` that references the token Secret and names the `Provider`. That's the user's only step.
3. **Create or reuse the `Provider`.** If no `Provider` with that name exists, the request creates one from its `provider` block. The Provider controller creates the workspace, SA and RBAC as usual (RFC 006), and the request waits until they're ready.
4. **Register.** The request calls `POST /v1/connections` with the token, the addressing and a first set of credentials. It gets back a `connectionID` and a connection secret, and stores them in a Secret owned by the `Provider`. The request is then `Ready` and is garbage-collected after a TTL.
5. **Open the tunnel.** The connector opens `/v1/connections/{id}/tunnel` with the connection secret. The Provider-side reconciles through it.
6. **Refresh credentials.** The connector calls `PUT /v1/connections/{id}/credentials` with a fresh SA token, periodically and after every reconnect.
7. **Offboard.** Deleting the `Provider` calls `DELETE /v1/connections/{id}`.

From step 5 on, only the `Provider` and its connection Secret matter. Every restart or outage is handled by steps 5 and 6 again.

## Design

### Onboarding token

The token is one opaque string that the user pastes as-is:

```
pm1.<base64url(Provider base URL)>.<tokenID>.<tokenSecret>
```

- **The URL is inside the token.** The registration call already sends a live SA token, so sending it to the wrong host would leak a kcp credential. With the URL in the token, a mistyped or phished URL is impossible short of a forged token.
- **Single-use and short-lived** (minutes). The Provider-side stores only `tokenID → {hash(secret), expiry, owner, usedBy}`.
- **Stored in a Secret**, referenced from the request, never inlined in its spec. Specs leak into GitOps repos, `kubectl get -o yaml` and audit logs. The controller adds the request as the Secret's owner, so the token is garbage-collected with the request.
- **The TTL must cover `Provider` setup.** When the request creates the `Provider`, registration waits for the workspace and SA. That normally takes seconds, but a TTL in the tens of minutes leaves room for a slow kcp.
- **Trust in the Provider-side** comes from web PKI on the URL, or from `spec.remote.caBundle` for private CAs.

### Connection API

All paths are relative to the base URL from the token and are versioned.

```
POST /v1/connections
Authorization: Bearer <tokenID>.<tokenSecret>
{
  "idempotencyKey": "<ProviderOnboardingRequest UID>",
  "previousConnectionID": "...",                 // optional: re-onboarding this Provider
  "clusterID": "<logical cluster name of the provider workspace>",
  "frontProxyURL": "https://...",
  "apiExportEndpointSlice": "<name>",
  "transport": "tunnel" | "direct",
  "credentials": { "caBundle": "...", "token": "<bound SA token>", "expirationTimestamp": "..." }
}
→ 201 { "connectionID": "...", "connectionSecret": "..." }

PUT /v1/connections/{id}/credentials
Authorization: Bearer <connectionSecret>
{ "caBundle": "...", "token": "<fresh bound SA token>", "expirationTimestamp": "..." }
→ 204

GET /v1/connections/{id}/tunnel          (WebSocket upgrade)
Authorization: Bearer <connectionSecret>

DELETE /v1/connections/{id}
Authorization: Bearer <connectionSecret>
→ 204
```

- **One-step registration.** The workspace and SA exist before the request registers, so the first call carries real addressing and credentials. There's no separate "onboard, then connect" phase.
- **Idempotency.** `POST` is keyed on `(tokenID, idempotencyKey)`. If the response is lost, a retry from the same request returns the same `connectionID` and **issues a new connection secret**, which invalidates the previous one. The Provider-side stores only hashes, so it can't return the old secret, and the old one never reached anyone.
- **Addressing is immutable.** `clusterID`, `frontProxyURL` and `apiExportEndpointSlice` are fixed at registration. `PUT` only replaces the token and CA bundle. So a stolen connection secret can't redirect the Provider-side to another kcp. Changing addressing means re-onboarding.
- **Responses carry nothing sensitive** beyond the connection secret on `POST`.
- **Bearer credentials survive ingresses.** Everything works through an ingress or CDN that terminates TLS. No client certificates and no custom signatures.

### Credentials and rotation

- **Send parts, not a kubeconfig:** the CA bundle, the token and the real URLs.
- **The PM-side mints, the Provider-side holds.** The connector calls TokenRequest (`serviceaccounts/token`) and `PUT`s the result at about half the token's TTL, and immediately after every reconnect.
- **Long-lived proof, short-lived authorization.** The connection secret is long-lived and the SA token is short-lived. An outage longer than the SA token's TTL therefore recovers on its own: the next `PUT` delivers a fresh token. If a credential could only be refreshed by presenting itself, its TTL would become a power-off budget.
- **Credentials don't depend on the tunnel.** `PUT` is a plain outbound HTTPS call. A broken tunnel doesn't block credential delivery, and the other way round.
- **RBAC is owned by the PM-side.** The Provider controller creates the SA and RBAC in the provider workspace (RFC 006). The Provider-side doesn't request permissions.

### What each credential exposes

| Credential | Lifetime | Stored at | If stolen |
|---|---|---|---|
| Onboarding token | minutes, single-use | PM-side Secret until spent; hash on the Provider-side | The thief can register **their own** kcp under the token owner's portal account (billing, quotas, possibly account-level resources). The legitimate registration then fails visibly with "already used". Nothing from the PM-side leaks. |
| Connection secret | long-lived | PM-side Secret; hash on the Provider-side | The thief can push a broken token or delete the connection: loud denial of service. They can't redirect it, because addressing is immutable. |
| SA token | hours | Provider-side | Access to the provider workspace and the APIExport virtual workspace, within RFC 006 RBAC. Inherent to any design; bounded by TTL and RBAC scope. |

### Transport

NAT is a requirement, so the tunnel is the default transport.

- **The connector dials out.** It opens a WebSocket to `/v1/connections/{id}/tunnel`, authenticated with the connection secret, and keeps it open.
- **The tunnel is a dial proxy**, not a pipe to a single apiserver. Every `Dial` on the Provider-side becomes a stream over the one WebSocket, opened with its target `host:port`. The connector checks the target against the allow-list, dials it and pipes bytes.
- **TLS is end to end.** The Provider-side keeps the real URLs (the front-proxy and the shard virtual-workspace URLs from the APIExportEndpointSlice). Only its `Dial` goes through the tunnel; `ServerName` is the real host, so no URLs are rewritten. The connector can't read or alter traffic.
- **client-go integration.** The tunnel plugs in as `rest.Config.Dial`. multicluster-runtime per-endpoint clients are derived from the base `rest.Config`, so they all inherit it.
- **Allow-list.** The connector only dials the front-proxy and the hosts in the APIExportEndpointSlice, which it watches. Shard hostnames can be internal-only.
  - The list is built only from PM-side objects, never from anything the Provider-side sends. An empty list refuses every dial.
  - Each hostname is resolved once, every resulting address is checked, and the connector dials those exact addresses, so DNS can't be re-pointed between the check and the dial.
  - Link-local, multicast and cloud-metadata addresses are always refused.
- **Liveness.** The tunnel runs its own ping/pong with read deadlines. Proxies and CDNs drop idle connections without telling either end, and TCP keepalive doesn't notice.
- **Use an existing library** for the tunnel protocol (e.g. remotedialer or Konnectivity) rather than designing one.
- **`direct` mode.** When the Provider-side can reach kcp, `transport: direct` skips the tunnel: the Provider-side dials the real URLs itself. Registration, credentials and revocation are identical in both modes.

**ManagedProvider (RFC 006) is unchanged.** The PM operator deploys the provider itself, so trust is implicit and none of this is needed.

### Resources

Both objects live in the same workspace.

```yaml
apiVersion: providers.platform-mesh.io/v1alpha1
kind: ProviderOnboardingRequest
metadata:
  name: my-provider-2026-10-06
spec:
  tokenSecretRef: { name: my-provider-token, key: token }
  provider:
    name: my-provider                                # created if missing, reused if present
    spec:                                            # applied only when the request creates the Provider
      remote:
        caBundle: ""                                 # optional, for private CAs
        transport: Tunnel                            # Tunnel (default) | Direct
  ttlSecondsAfterFinished: 3600                      # optional, default 1h; applies to Ready requests only
status:
  phase: Ready                                       # WaitingForProvider → Registering → Ready | Failed
  providerRef: { name: my-provider }
  connectionID: "..."
  finishedTime: "2026-10-06T12:00:00Z"
  conditions:
    - type: ProviderCreated     # False when an existing Provider was reused
    - type: Registered
    - type: ProviderSpecIgnored # the Provider existed and differs from spec.provider.spec
---
apiVersion: providers.platform-mesh.io/v1alpha1
kind: Provider
metadata:
  name: my-provider                                  # no ownerReferences: independent once created
spec:
  providerKubeconfigSecret: { ... }                  # existing fields (RFC 006), largely unchanged
  remote:                                            # new: an externally run Provider
    caBundle: ""
    transport: Tunnel
status:
  phase: Ready
  connection:                                        # mirrors the connection Secret, for display
    url: https://provider.example.com/pm
    connectionID: "..."
    registeredTime: "2026-10-06T12:00:00Z"
  conditions:
    - type: Registered          # False with reason AwaitingRequest until a request succeeds
    - type: CredentialsValid    # last PUT accepted
    - type: TunnelConnected
---
apiVersion: v1
kind: Secret
metadata:
  name: my-provider-connection                       # owned by the Provider
type: Opaque
stringData:
  url: https://provider.example.com/pm
  connectionID: "..."
  connectionSecret: "..."
```

**`ProviderOnboardingRequest`**
- **The user creates only the request** (and the token Secret). The `Provider` is created as a side effect if it doesn't exist yet.
- **Runs once.** A request registers at most once. A finished request never acts again, even before it's garbage-collected.
- **One active request per `Provider`.** A second request for the same `Provider` while one is running is refused with a condition until the first finishes, so two registrations can't race on one connection.
- **Short-lived.** A `Ready` request is deleted after `ttlSecondsAfterFinished`, together with the token Secret it owns. Failed requests are kept until someone deletes them, as a record of the attempt.
- **Separate permission.** Registering sends a live SA token to an external party. RBAC on requests is therefore "who may connect this PM instance to an external Provider", separate from managing `Provider` objects. Creating a request implies creating the `Provider`, which the controller does on the user's behalf.

**How the request treats `provider.name`**

| The `Provider`... | The request... |
|---|---|
| doesn't exist | creates it from `provider.spec`, waits for the workspace and SA, then registers |
| exists, not registered (a leftover from a failed attempt, or created by hand) | reuses it and registers |
| exists and is registered | re-onboards: `POST` with `previousConnectionID`. The Provider-side keeps the same connection record and issues a new connection secret only if the token's owner and the `clusterID` match; otherwise it refuses and the existing connection is untouched |
| exists, but isn't a remote `Provider` (e.g. a ManagedProvider) | refuses |

- **`provider.spec` applies only on creation.** If the `Provider` already exists, the request doesn't rewrite it. When the two differ, it sets `ProviderSpecIgnored`. Later changes are made on the `Provider` directly.
- **The `Provider` is never deleted on failure.** A failed registration leaves it with `Registered=False`. A later request with the same name picks it up, or the user deletes it. No cleanup can fail, so nothing is orphaned.
- **The connection Secret is replaced only after a successful `POST`.** A failed re-onboarding leaves a working connection working.
- **Moving to another portal account** isn't re-onboarding: the Provider-side refuses a token from a different owner. Delete the `Provider` (which offboards it) and onboard again.

**`Provider`**
- Kept largely as-is (RFC 006). It gains `spec.remote` and handles credentials and the tunnel.
- **Independent once created**: no `ownerReferences` to the request, so it survives the request's garbage collection.
- **The connection Secret is the durable state**, not the status. After a restart the connector needs only that Secret.

**Re-onboarding** is creating a new request with the same `provider.name`. Typical reasons: the connection secret leaked, the connection was deleted on the Provider-side (`CredentialsValid=False`), or the connection Secret was lost.

**Multiple PM instances per Provider** are supported. The Provider-side keeps one record per connection and doesn't need to know which connections share a PM instance.

### Restart and failure semantics

- **Leases** are used on both sides, for different things. On the PM-side they're only for connector leader election, so that one replica holds a connection's tunnel. On the Provider-side they can record which replica holds which tunnel (see the table).

| Scenario | Behavior |
|---|---|
| PM-side restarts | The connector reads the connection Secret, reopens the tunnel and `PUT`s fresh credentials. |
| Provider-side restarts | Tunnels drop and connectors reconnect and `PUT`. The Provider-side rebuilds clients from the connection record and the latest token. |
| Outage longer than the SA token TTL | The next `PUT` after recovery delivers a fresh token. |
| Crash after the request created the `Provider`, before `POST` | The request reuses the `Provider` it created on the next reconcile; nothing is created twice. |
| `POST` response lost, or crash before the connection Secret is written | The same request retries with the same token and idempotency key: same `connectionID`, new connection secret. If the token has expired in the meantime, the request fails and the user creates a new one with a new token; the `Provider` is reused. |
| Registration fails (bad or expired token, Provider-side refuses) | The request is `Failed` and kept. The `Provider` stays with `Registered=False`, or, when re-onboarding, keeps its existing connection. |
| Connection deleted on the Provider-side | `PUT` and the tunnel get 401. The `Provider` gets `CredentialsValid=False`. Recovery is a new request with a new token. |
| Request rejected by the Provider-side vs. something in between | A rejection by the Provider-side itself means `CredentialsValid=False` and a long backoff. An ingress error or 5xx is transient. The Provider-side marks its own rejections so the connector can tell them apart. |
| Connector reconnects while the old tunnel is still open | The new tunnel replaces the old one, which is closed. Cleaning up the old tunnel must only remove its own registration (and Lease), never the one that replaced it. Otherwise the connection becomes unroutable while both ends look healthy. |
| Provider-side runs multiple replicas | A tunnel lands on one pod. Either shard so that pod owns that connection's reconciliation, or add a routing layer: the pod holding the tunnel claims a Lease per connection naming its internal address and renews it while the tunnel lives, and other pods relay through it. The Leases are then the only input to the connection record's `Connected` status. |

### Offboarding and revocation

- **Deleting the `Provider`** triggers a finalizer that calls `DELETE /v1/connections/{id}`. The Provider controller then cleans up the SA and RBAC (RFC 006), and the connection Secret is garbage-collected with the `Provider`.
- **Deleting the connection on the Provider-side** invalidates the connection secret and closes the tunnel. The PM-side surfaces it through `CredentialsValid=False`.

## Changes from v1

| v1 | v2 | Why |
|---|---|---|
| `ProviderOnboardingRequest` created before the token (`AwaitingToken`), creates the `Provider` after `/onboard` | `ProviderOnboardingRequest` created with the token; creates the `Provider` first, then registers | No pre-token phase without fingerprint binding; registering after the workspace exists makes it one call |
| `/onboard` then `/connect`, credentials in the first message on the channel | One `POST`, then `PUT` for credentials, tunnel separate | Credentials no longer depend on the tunnel |
| Per-connection keypair, fingerprint entered in the portal, token bound to it | Long-lived connection secret, stored as a hash on the Provider-side | Binding only protected against a stolen, unspent token; see the credentials table |
| mTLS, then DPoP-style proofs | Bearer credentials | Work through any ingress, no custom signatures |
| Pin of the Provider-side's key, signed replies, HPKE sealing | Web PKI, URL embedded in the token | The realistic risk was a wrong URL, which the token now rules out |
| Adoption via `previousConnectionID` with `ProviderConflict` | Same mechanism; the Provider-side checks the token's owner and the `clusterID`, and a refusal fails the request | Reuse by name is safe because the Provider-side, not the name, decides |

## Rejected alternatives

- **Onboarding fields on the `Provider`, no request object**: one object fewer, but re-onboarding has no explicit signal (it would mean swapping the token Secret and having the controller detect a new `tokenID`), the token reference goes stale on a long-lived spec, there's no separate permission for connecting to an external party, and failed attempts leave no record.
- **A request that requires an existing `Provider`** (`providerRef` only): the user creates two objects, and gains nothing that create-or-reuse by name doesn't cover.
- **Deleting the `Provider` when registration fails**: cleanup can fail and leave orphans. Leaving it unregistered for the next request to reuse is simpler.
- **Binding the token to a PM-side key** (fingerprint in the portal, proofs of possession): closes the stolen-unspent-token case at the cost of a key lifecycle, a portal UX step and custom signatures. That risk is short-lived, bounded to the owner's account, and visible. Could come back as opt-in hardening.
- **Pinning the Provider-side's key**: protects against misissued certificates, at the cost of signed replies and key rotation problems for every connected PM-side. Web PKI is what the PM-side trusts for any other API it calls.
- **Credentials delivered over the tunnel**: couples credential freshness to reachability.
- **Mutable addressing on `PUT`**: a stolen connection secret could redirect the connection to another kcp.
- **A URL entered separately from the token**: a mistyped or phished URL would receive a live SA token at registration.
- **A fresh outbound connection per dial**, instead of streams over one tunnel: a full handshake per dial and one connection held per watch.
- **Pull mode** (a PM-side agent pushes specs to the Provider-side and pulls status back): needs no tunnel at all, but changes the provider programming model from reconciling the kcp virtual workspace to reconciling local objects.

## Open items

- **Connector placement.** Inside the PM operator's Provider controller, a separate deployment per PM instance, or one per connection. Considerations: one long-lived tunnel per connection, spreading tunnels across replicas, network reach to the front-proxy and every shard, TokenRequest permissions in every provider workspace, blast radius (it holds connection secrets and tokens), coupling to PM operator upgrades.
- **Tunnel library.** remotedialer or Konnectivity, and how its protocol is versioned alongside `/v1`.
- **Provider-side multi-replica.** Sharding vs. the Lease-based routing layer.
- **Connection secret rotation.** Whether `PUT` should occasionally return a new secret, and how to avoid losing it if the response is lost.
- **`direct` mode.** Whether the user chooses it, or the connector detects reachability.
- **Addressing changes.** Whether re-onboarding is acceptable when the front-proxy URL changes, or whether a dedicated, re-authorized update is needed.
- **Request permissions.** Whether creating a request should require create permission on `Provider` too, or whether the request's permission alone is enough for the controller to create it.
- **Token format and portal UX.** Encoding, length, and whether the portal offers a "copy as Secret manifest" button.

## Prior art

- **kubeadm bootstrap tokens:** `tokenID.tokenSecret`, stored hashed, short-lived.
- **Rancher `remotedialer`** and **Konnectivity (apiserver-network-proxy):** multiplexed reverse tunnels with dial callbacks.
- **Open Cluster Management:** bootstrap kubeconfig plus registration; cluster-proxy for the reverse tunnel.
- **Karmada pull mode:** the pull-mode alternative.
