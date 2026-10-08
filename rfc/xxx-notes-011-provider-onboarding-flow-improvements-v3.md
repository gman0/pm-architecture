# Notes v3: RFC 011 provider onboarding flow

Working notes for [011](./011-provider-onboarding-flow-improvements.md). Not the RFC itself. A variant of the [v2 notes](./xxx-notes-011-provider-onboarding-flow-improvements-v2.md) that drops the long-lived connection secret: after registration, the only credentials are kcp ServiceAccount tokens. See "Changes from v2".

## Terminology

- **PM-side**: the PM instance (kcp + PM operator). May be behind NAT; dials out.
- **Provider-side**: the service provider (controllers + portal in its runtime cluster). Serves the connection API and reconciles on the PM-side.
- **Connection**: the relationship between one `Provider` object on the PM-side and one connection record on the Provider-side, identified by `connectionID`. Created by registration, ended by deletion on either side. The `connectionID` is a handle, not a secret.
- **Onboarding token**: a single-use, short-lived token from the Provider's portal. It contains the Provider's URL, so the user never enters a URL separately.
- **Access token**: a bound SA token with kcp's audience. The Provider-side holds it and uses it against kcp. The same thing v2 calls the SA token.
- **Call token**: a bound token for the same SA, with the Provider's base URL as audience and a TTL of minutes. It authenticates every call to the connection API after registration. kcp doesn't accept it.
- **Trust anchor**: what the Provider-side pins at registration to verify call tokens: the kcp SA issuer, its public keys, and the SA's identity.
- **Connector**: the PM-side component that mints tokens, delivers them and holds the tunnel. Where it runs is an open item.
- **`ProviderOnboardingRequest`**: a short-lived PM-side resource that carries the token, creates the `Provider` if needed, and runs registration once. Creating one is also how a user re-onboards.

## Problem

- The Provider-side must reconcile resources on the PM-side, for any number of PM instances.
- **The PM-side may be behind NAT.** Every connection is opened by the PM-side; the Provider-side never dials the PM-side network directly.
- The PM-side owner gets a token from the Provider's portal. Authenticating that user is the portal's business and out of scope.
- Both sides must survive restarts, and outages of either side, without manual steps.
- **New in v3: no non-expiring credential anywhere.** v2's connection secret grants no kcp access, but it's the one credential that never expires and needs its own rotation story.

The design keeps three concerns apart, so each stays simple:

1. **Onboarding**: one registration call that turns a token into a connection and pins the trust anchor.
2. **Credentials**: plain HTTPS calls that keep the Provider-side's kcp credentials fresh.
3. **Reachability**: a tunnel that lets the Provider-side reach kcp through NAT.

## Core idea: proof is minting

In v2, the PM-side proves "I own this connection" by presenting the connection secret. In v3, it proves it by **minting a fresh token for the connection's SA**. Only the PM-side's kcp can sign one, and the connector can always mint, because it has TokenRequest permission in its own kcp.

That's why dropping the long-lived secret doesn't create a power-off budget. The connector never has to present an old token to get a new one. After an outage of any length, it mints and presents a fresh token. Compare railgrid's edges provider. There, the agent holds the token but can't mint one: it refreshes by presenting its current token, so an agent offline for longer than the TTL needs a new join token. The RFC avoids that because, here, the dialer is also the minter.

What's needed for this to work: the Provider-side must be able to verify a token it has never seen, **without reaching the PM's kcp**, which may be behind NAT. So it verifies call tokens locally, as JWTs, against keys it pinned at registration.

## Flow overview

1. **Get a token.** In the Provider's portal, the user gets an onboarding token and stores it in a Secret.
2. **Create the request.** The user creates a `ProviderOnboardingRequest` that references the token Secret and names the `Provider`. That's the user's only step.
3. **Create or reuse the `Provider`.** If no `Provider` with that name exists, the request creates one from its `provider` block. The Provider controller creates the workspace, SA and RBAC as usual (RFC 006), and the request waits until they're ready.
4. **Register.** The request calls `POST /v1/connections` with the onboarding token, the addressing, the trust anchor, a call token and a first access token. It gets back a `connectionID` and records it on the `Provider`. The request is then `Ready` and is garbage-collected after a TTL.
5. **Open the tunnel.** The connector opens `/v1/connections/{id}/tunnel` with a fresh call token. The Provider-side reconciles through it.
6. **Refresh credentials.** The connector calls `PUT /v1/connections/{id}/credentials` with a fresh access token, authenticated by a fresh call token, periodically and after every reconnect.
7. **Offboard.** Deleting the `Provider` calls `DELETE /v1/connections/{id}`, authenticated by a call token.

From step 5 on, only the `Provider` and the connector's ability to mint tokens matter. There's no PM-side secret to lose or leak. Every restart or outage is handled by steps 5 and 6 again.

## Design

### Onboarding token

Unchanged from v2:

```
pm1.<base64url(Provider base URL)>.<tokenID>.<tokenSecret>
```

- **The URL is inside the token.** The registration call sends a live access token, so sending it to the wrong host would leak a kcp credential.
- **Single-use and short-lived** (minutes). The Provider-side stores only `tokenID → {hash(secret), expiry, owner, usedBy}`.
- **Stored in a Secret**, referenced from the request, never inlined in its spec. The controller adds the request as the Secret's owner, so the token is garbage-collected with the request.
- **The TTL must cover `Provider` setup.**
- **Trust in the Provider-side** comes from web PKI on the URL, or from the `Provider`'s `spec.remote.caBundle` for private CAs.
- **New role in v3:** the onboarding token is the only thing that authorizes **pinning a trust anchor**. Everything after registration is authorized by tokens that verify against that anchor.

### Tokens

| | Access token | Call token |
|---|---|---|
| Audience | kcp's API audience | The Provider's base URL |
| TTL | Hours | Minutes |
| Minted | At about half the TTL, and after every reconnect | Per call, or cached briefly; one per tunnel open |
| Sent | In the body of `POST` and `PUT` | As `Authorization: Bearer` on every call |
| Accepted by | kcp only | The Provider-side's connection API only |
| Held by the Provider-side | Yes, to reach kcp | No; verified and discarded |

- **Why two tokens.** A single token would mean the credential the Provider-side holds to reach kcp is the same one that authenticates as the PM-side to the Provider's API. That adds three problems:
  - **Logging:** the `Authorization` header shows up in ingress and CDN logs, so the header would expose a kcp credential.
  - **Lifetime:** a stolen header token would be valid for hours instead of minutes.
  - **Replay:** a token given to Provider P would also authenticate against Provider Q, if Q's verification were ever loose.

  Separating the audiences removes all three. Both are standard TokenRequest tokens for the same SA; only the `audiences` field differs.
- **Bound tokens only.** Both are TokenRequest tokens bound to the SA. With `--service-account-lookup=true` (the default), kcp checks on every request that the SA still exists **with the UID in the token**, so deleting the SA cuts off kcp access immediately. kcp carries a patch that honours `--service-account-lookup=false` for bound tokens, for multi-shard setups where the validating shard can't see the SA's workspace. With lookup off, an access token stays valid until it expires even after the SA is deleted.

#### Claims in a kcp bound token

From kcp's Kubernetes fork (`vendor/k8s.io/kubernetes/pkg/serviceaccount/claims.go`):

```json
{
  "iss": "https://kcp.default.svc",
  "sub": "system:serviceaccount:<ns>:<name>",
  "aud": ["..."],
  "exp": 0, "iat": 0, "nbf": 0,
  "jti": "...",
  "kubernetes.io": {
    "clusterName": "<logical cluster>",
    "namespace": "<ns>",
    "serviceaccount": { "name": "<name>", "uid": "<SA UID>" }
  }
}
```

- `clusterName` is kcp's addition; upstream Kubernetes has no such claim. kcp sets it from the SA's logical cluster when minting.
- Legacy Secret-based tokens use a different shape, with a flat `kubernetes.io/serviceaccount/clusterName` claim (`legacy.go`). kcp's own request filter reads both (`pkg/server/filters/serviceaccounts.go`). v3 accepts **only** the bound shape.
- `sub` doesn't include the cluster, so it isn't unique across workspaces. The subject check must compare `clusterName` and the SA UID as well.
- The JWT `kid` header is derived from the signing public key (a hash of its DER encoding, `jwt.go`), so pinned keys can be matched by `kid`.

### Trust anchor and verification

At registration, the PM-side sends, and the Provider-side pins:

- **`issuer`**: the `iss` the kcp shards put in SA tokens. kcp's default is `https://kcp.default.svc`, which isn't a reachable URL. So the keys are **sent**, not discovered from `/.well-known/openid-configuration`.
- **`keys`**: the public keys (JWKS) the PM's kcp signs SA tokens with. They must cover every shard: a call token minted by any shard has to verify.
- **`subject`**: the SA's identity, `system:serviceaccount:<ns>:<name>`, plus its logical cluster (`clusterID`) and its **UID**. Pinning the UID means a deleted and recreated SA with the same name doesn't inherit the connection. Railgrid uses the same pattern, binding agent identity to the edge's UID.

The Provider-side accepts a call token for connection `{id}` if all of the following hold:

1. the signature verifies against the connection's pinned keys;
2. `iss` matches the pinned issuer;
3. `aud` contains the Provider's base URL, and nothing else is accepted in its place;
4. the token isn't expired, its `nbf` has passed, and its TTL isn't above a maximum (e.g. 15 minutes), so a long-lived token can't be passed off as a call token;
5. `kubernetes.io.clusterName`, `kubernetes.io.namespace`, `kubernetes.io.serviceaccount.name` and `kubernetes.io.serviceaccount.uid` match the pinned subject. A token in the legacy shape is refused.

- **Verification is local.** No call to the PM's kcp, so it works when the tunnel is down.
- **Ship it in a provider SDK.** kcp's token shapes have edge cases (see "Claims in a kcp bound token" above, and railgrid's `auth.go`, which had to fix a misclassification). Each Provider shouldn't have to reimplement this.

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
  "transport": "tunnel" | "direct",
  "trustAnchor": {
    "issuer": "https://kcp.default.svc",
    "keys": { "keys": [ /* JWKS */ ] },
    "subject": { "namespace": "...", "name": "...", "uid": "..." }
  },
  "callToken": "<call token>",                   // proves the PM-side can mint for the subject
  "credentials": { "caBundle": "...", "token": "<access token>", "expirationTimestamp": "..." }
}
→ 201 { "connectionID": "..." }

PUT /v1/connections/{id}/credentials
Authorization: Bearer <call token>
{ "caBundle": "...", "token": "<fresh access token>", "expirationTimestamp": "..." }
→ 204

PUT /v1/connections/{id}/keys
Authorization: Bearer <call token signed by a currently pinned key>
{ "keys": { "keys": [ /* full new JWKS */ ] } }
→ 204

GET /v1/connections/{id}/tunnel          (WebSocket upgrade)
Authorization: Bearer <call token>

DELETE /v1/connections/{id}
Authorization: Bearer <call token>
→ 204
```

- **One-step registration.** As in v2: the request waits in `WaitingForProvider` until the workspace and SA exist, then registers with real addressing and credentials. The request controller mints both tokens itself, so it needs the same TokenRequest permission as the connector.
- **The `POST` call token is consistency, not proof.** For a new connection, the trust anchor is self-asserted: anyone can send keys and a token signed by them. The onboarding token is what authorizes it. The call token only checks that the keys, subject and token agree, which catches misconfiguration before anything is pinned.
- **Re-onboarding is proof.** With `previousConnectionID`, the call token must verify against the connection's **already pinned** anchor, in addition to the usual checks on the token's owner and the addressing. A stolen onboarding token therefore can't touch an existing connection. That's stronger than v2, where it could cause a new connection secret to be issued.
- **No APIExport details at registration.** As in v2: registration names only the workspace (`clusterID`) and the front-proxy.
- **Idempotency.** `POST` is keyed on `(tokenID, idempotencyKey)`. A retry returns the same `connectionID`. Nothing in the response is secret, so there's no "issue a new secret and invalidate the old one" step.
- **Retries outlive the token's expiry**, for a bounded window (e.g. 24h), as in v2: a lost response would otherwise leave a connection record the PM-side doesn't know about. The Provider-side garbage-collects records that never saw a `PUT` or a tunnel.
- **Validate before replacing.** `PUT /credentials` replaces the stored access token only if the new one verifies against the pinned anchor (signature, subject, expiry) and has kcp's audience. A bad or garbage token never overwrites a working one.
- **Addressing is immutable**: `clusterID`, `frontProxyURL`, `issuer` and `subject` are fixed at registration. A stolen call token is valid for minutes and can't redirect a connection to another kcp. Changing any of them means a new connection.
- **The CA bundle and the keys can change**, because the trust anchor is the signing keys, not TLS to kcp. Updating either needs a call token that verifies against the current keys, so it already requires the ability to mint for the SA.
- **Bearer credentials survive ingresses.** Everything works through an ingress or CDN that terminates TLS. No client certificates and no custom signatures.

### Key rotation

kcp verifies tokens against every key in its key files but signs with one signing key. So a rotation naturally has an overlap period:

1. Add the new public key to kcp's verification keys.
2. The connector notices the change and pushes the full key set with `PUT /keys`, authenticated by a call token signed with the **old** key.
3. Switch kcp's signing key.
4. After the longest access-token TTL, drop the old key from kcp and push the reduced set.

- **The connector must push before the switch.** If kcp switches first, every call token fails verification until the keys are pushed. Recovery is re-onboarding. This is the main operational hazard of v3; see "Open items".
- **Where the connector reads keys from**: kcp's issuer discovery documents or the key files' public halves. Which source is authoritative across shards is an open item.

### Credentials and rotation

- **Send parts, not a kubeconfig:** the CA bundle, the token and the real URLs.
- **The PM-side mints, the Provider-side holds.** The connector calls TokenRequest (`serviceaccounts/token`) and `PUT`s the access token at about half its TTL, and immediately after every connector restart and tunnel reconnect.
- **No long-lived proof needed.** An outage longer than the access token's TTL recovers on its own: the connector mints a fresh call token and a fresh access token, and the next `PUT` delivers it. The Provider-side never needs the old token to accept the new one.
- **Credentials don't depend on the tunnel.** `PUT` is a plain outbound HTTPS call, and verification is local.
- **RBAC is owned by the PM-side.** The Provider controller creates the SA and RBAC in the provider workspace (RFC 006). The Provider-side doesn't request permissions.
- **No static kubeconfig for remote `Provider`s.** As in v2: a remote `Provider` only ever uses short-lived TokenRequest tokens, so the controller skips RFC 006's kubeconfig Secret.
  - **Note: no long-lived kcp tokens by default, anywhere.** Unchanged from v2. v3 extends this to the connection itself: there is no long-lived credential of any kind.

### What each credential exposes

| Credential | Lifetime | Stored at | If stolen |
|---|---|---|---|
| Onboarding token | minutes, single-use | PM-side Secret until spent; hash on the Provider-side | The thief can register **their own** kcp under the token owner's portal account (billing, quotas, possibly account-level resources). Unlike v2, they **can't** re-onboard an existing connection: that needs a call token that verifies against the pinned keys. The legitimate registration then fails visibly with "already used". Nothing from the PM-side leaks. |
| Call token | minutes | Nowhere; minted per call | Until it expires: open a tunnel (displacing the real one until the connector reconnects), or delete the connection. A loud denial of service. It can't push credentials (the access token it would push has to verify too), can't redirect, and gives no kcp access. |
| Access token | hours | Provider-side | Access to the provider workspace and the APIExport virtual workspace, within RFC 006 RBAC. Inherent to any design; bounded by TTL and RBAC scope. Not accepted by the connection API (wrong audience). |
| kcp SA signing key | long-lived | PM-side kcp | Every connection, and everything else in that kcp. Not new: whoever holds it already owns the PM instance. |

### Revocation

- **kcp access** ends immediately when the SA is deleted, as long as kcp runs with `--service-account-lookup=true` (the default). With lookup off, it ends when the last access token expires, so the access-token TTL becomes the revocation window.
- **Connection API access** ends when the last call token minted for the SA expires, because the Provider-side verifies locally and can't see that the SA was deleted. That's minutes at most, and the only thing possible in that window is denial of service on a connection that's being removed anyway.
- **`DELETE` remains the primary offboarding path.** Revoking the SA is the backstop.

### Transport

Unchanged from v2, except that the tunnel is authenticated with a call token instead of the connection secret.

- **The connector dials out.** It opens a WebSocket to `/v1/connections/{id}/tunnel`, authenticated with a fresh call token, and keeps it open.
- **The tunnel isn't re-authenticated while it's open.** Everything the Provider-side sends through it to kcp carries the access token, which expires and stops working if the SA is deleted.
- **The tunnel is a dial proxy**, not a pipe to a single apiserver. Every `Dial` on the Provider-side becomes a stream over the one WebSocket, opened with its target `host:port`. The connector checks the target against the allow-list, dials it and pipes bytes.
- **TLS is end to end.** The Provider-side keeps the real URLs; only its `Dial` goes through the tunnel.
- **client-go integration.** The tunnel plugs in as `rest.Config.Dial`.
- **Allow-list.** The connector only dials the front-proxy and the `spec.virtualWorkspaceURL` host of every kcp `Shard`, which it watches. Built only from PM-side data. Each hostname is resolved once and the connector dials those exact addresses. Link-local, multicast and cloud-metadata addresses are always refused.
- **Liveness.** The tunnel runs its own ping/pong with read deadlines.
- **Use an existing library** for the tunnel protocol (e.g. remotedialer or Konnectivity).
- **`direct` mode.** When the Provider-side can reach kcp, `transport: direct` skips the tunnel. Registration, credentials and revocation are identical in both modes.

**ManagedProvider (RFC 006) is unchanged.** The PM operator deploys the provider itself, so trust is implicit and none of this is needed.

### Resources

Both objects live in the same workspace. v3 has no connection Secret.

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
  connection:                                        # the durable connection state; nothing secret
    url: https://provider.example.com/pm
    connectionID: "..."
    registeredTime: "2026-10-06T12:00:00Z"
    keysPushed: "<hash of the last JWKS pushed>"     # lets the connector detect key changes
  conditions:
    - type: Registered          # False with reason AwaitingRequest until a request succeeds
    - type: CredentialsValid    # last PUT accepted
    - type: KeysCurrent         # the Provider-side has the key set kcp currently signs with
    - type: TunnelConnected
```

**`ProviderOnboardingRequest`**: unchanged from v2.
- **The user creates only the request** (and the token Secret).
- **Runs once.** A finished request never acts again.
- **One active request per `Provider`.**
- **Short-lived.** A `Ready` request is deleted after `ttlSecondsAfterFinished`, together with the token Secret it owns. Failed requests are kept.
- **Separate permission.** Registering sends a live access token to an external party, so RBAC on requests is separate from managing `Provider` objects.

**How the request treats `provider.name`**

| The `Provider`... | The request... |
|---|---|
| doesn't exist | creates it from `provider.spec`, waits for the workspace and SA, then registers |
| exists, not registered (a leftover from a failed attempt, or created by hand) | reuses it and registers |
| exists and is registered | re-onboards: `POST` with `previousConnectionID`. The Provider-side restores the same connection record only if the call token verifies against the **pinned** anchor, and the token's owner and the addressing match; otherwise it refuses and the existing connection is untouched. Before sending anything, the request refuses if the token's URL differs from the existing connection's URL |
| exists, but isn't a remote `Provider` (e.g. a ManagedProvider) | refuses |

- **`provider.spec` applies only on creation.** As in v2.
- **The `Provider` is never deleted on failure.** As in v2.
- **Moving to another portal account** isn't re-onboarding: delete the `Provider` and onboard again.

**`Provider`**
- Kept largely as-is (RFC 006). It gains `spec.remote` and handles credentials, keys and the tunnel.
- **Independent once created**: no `ownerReferences` to the request.
- **`status.connection` is the durable state.** It holds nothing secret. The connector needs only it and TokenRequest permission. Whether status is the right home for state that can't be rebuilt from observation is an open item.

**Re-onboarding** is creating a new request with the same `provider.name`. Fewer reasons than in v2, because there's no secret to leak or lose:
- the connection was deleted on the Provider-side (`CredentialsValid=False`);
- `status.connection` was lost;
- the Provider-side's pinned keys are out of date and `PUT /keys` can no longer authenticate (`KeysCurrent=False`).

**Multiple PM instances per Provider** are supported, as in v2.

### Restart and failure semantics

- **Leases** are used on both sides, as in v2: connector leader election on the PM-side, tunnel ownership on the Provider-side.

| Scenario | Behavior |
|---|---|
| PM-side restarts | The connector reads `status.connection`, mints a call token, reopens the tunnel and `PUT`s a fresh access token. |
| Provider-side restarts | Tunnels drop and connectors reconnect and `PUT`. The Provider-side rebuilds clients from the connection record and the latest access token. |
| Outage longer than the access token's TTL | The connector mints fresh tokens; the next `PUT` delivers one. Same as v2, without a long-lived secret. |
| Crash after the request created the `Provider`, before `POST` | The request reuses the `Provider` it created on the next reconcile. |
| `POST` response lost, or crash before `status.connection` is written | The same request retries with the same token and idempotency key and gets the same `connectionID`. Nothing is invalidated. The retry works even if the token has expired since, within the bounded retry window. |
| Registration fails (bad or expired token, Provider-side refuses) | The request is `Failed` and kept. The `Provider` stays with `Registered=False`, or, when re-onboarding, keeps its existing connection. |
| Connection deleted on the Provider-side | `PUT` and the tunnel get 401 (or 404). The `Provider` gets `CredentialsValid=False`. Recovery is a new request with a new token. |
| kcp signing key switched before the keys were pushed | Every call token fails verification, including the one `PUT /keys` needs, because kcp only signs with its current key. The `Provider` gets `KeysCurrent=False`. Recovery is re-onboarding. Prevented by pushing before switching (see "Key rotation"). |
| SA deleted and recreated (e.g. RBAC repair) | The new SA has a new UID, so its tokens don't match the pinned subject. Recovery is re-onboarding. |
| `PUT` or tunnel connect rejected by the Provider-side vs. by something in between | As in v2: the Provider-side marks its own rejections so the connector can tell them from transient ingress errors. |
| Connector reconnects while the old tunnel is still open | As in v2: the new tunnel replaces the old one, and the old tunnel's cleanup only removes its own registration. |
| Provider-side runs multiple replicas | As in v2: shard connections across pods, or route through per-connection Leases. Railgrid's edges provider is a working example of the Lease-routing option (`internal/tunnel/registry.go`, `remote.go`, `connman.go`). |

### Offboarding and revocation

- **Deleting the `Provider`** triggers a finalizer that calls `DELETE /v1/connections/{id}` with a call token. The Provider controller then cleans up the SA and RBAC (RFC 006). Deleting the SA cuts off kcp access even if `DELETE` never arrives, and the connection API within minutes.
- **An unreachable Provider-side doesn't block deletion forever.** As in v2: the finalizer retries `DELETE` and gives up after a timeout, or immediately if annotated to skip it.
- **Deleting the connection on the Provider-side** closes the tunnel and refuses further calls. The PM-side surfaces it through `CredentialsValid=False`.

## Changes from v2

| v2 | v3 | Why |
|---|---|---|
| Long-lived connection secret, stored as a hash on the Provider-side | No connection secret; short-lived call tokens verified against a pinned trust anchor | No non-expiring credential anywhere; the connector can always mint, so there's no power-off budget |
| `POST` returns `connectionID` and `connectionSecret` | `POST` returns `connectionID`; the request pins `issuer`, `keys` and `subject` | Nothing secret in the response |
| Connection Secret owned by the `Provider` is the durable state | `Provider.status.connection` is the durable state | Nothing secret to store |
| Lost response: same `connectionID`, new secret, old one invalidated | Lost response: same `connectionID`, nothing invalidated | No secret to reissue |
| One SA token, sent in request bodies | Two audiences: access token (kcp) and call token (Provider) | A header token visible to ingresses gives no kcp access and expires in minutes |
| Re-onboarding needs only the onboarding token, owner match and addressing match | Re-onboarding also needs a call token that verifies against the pinned anchor | A stolen onboarding token can't touch existing connections |
| Re-onboard when the secret leaks or is lost | Re-onboard only when the connection is gone, its state is lost, or the keys are out of date | Fewer failure modes |
| `PUT` replaces the token on a valid secret | `PUT` replaces the token only if the new one verifies | A bad push never breaks a working connection |
| No key management | `PUT /keys` and an ordered rotation procedure | The cost of local verification |
| Revocation: `DELETE`, or revoke the SA | Same, plus up to one call-token TTL of connection-API access after the SA is deleted | Local verification can't see SA deletion |

## Rejected alternatives

All of v2's rejected alternatives still apply. In addition:

- **Keep the connection secret (v2).** Simpler: no keys to pin or rotate. But it's the one non-expiring credential, and it needs its own rotation story (v2 open item). v3 trades that for key management, which kcp operators already have to handle.
- **Verify tokens through the tunnel.** The Provider-side checks each presented token with a `SelfSubjectReview` against the pinned front-proxy, through the tunnel, over TLS with the pinned CA. Gives immediate revocation and needs no keys. But credential refresh, tunnel replacement and `DELETE` would all depend on reachability, which v2 separated on purpose. Opening a tunnel would also mean accepting it before it can be verified. Viable in `direct` mode only.
- **One token for both kcp access and calls.** Fewer tokens to mint. But the header token would be a kcp credential valid for hours, visible to every ingress and CDN, and replayable against any Provider whose verification were loose.
- **Fetching keys from kcp's issuer discovery.** Needs kcp's discovery endpoint to be reachable from the Provider-side, which NAT rules out, and kcp's default issuer isn't a reachable URL. Pushing keys works in every topology.
- **Unpinned subject (any SA in `clusterID`).** Any SA in the provider workspace could then act on the connection. Pinning the SA, including its UID, limits it to the one the Provider controller created.

## Open items

- **kcp signing keys across shards.** Call tokens can be minted by any shard, so the pinned keys must cover every shard. kcp's sharded test server gives every shard the same key files (`cmd/sharded-test-server/shard.go`), but production must guarantee it too. Which source the connector reads keys from (discovery, key files, operator config) is undecided.
- **SA lookup in sharded kcp.** With `--service-account-lookup=false`, deleting an SA doesn't revoke its bound tokens until they expire. Find out whether PM's sharded deployments run with lookup off. If they do, shorten the access-token TTL or accept the longer revocation window.
- **Key rotation safety.** Whether the connector should refuse, or loudly warn, when kcp's current signing key isn't in the set the Provider-side has pinned, and whether the PM operator should gate signing-key switches on every connection reporting `KeysCurrent=True`.
- **Call-token TTL and the revocation window.** Shorter means faster revocation and more TokenRequest calls. Whether to cap the window by checking SA existence some other way.
- **Where `connectionID` lives.** Status is conventionally rebuildable from observation, and a `connectionID` isn't. Options: keep it in status, use a non-secret ConfigMap owned by the `Provider`, or let the PM-side recover it with a call token (`GET /v1/connections/self`), which would also remove the need for the retry window after token expiry.
- **Connector placement.** As in v2. Without connection secrets, its blast radius is TokenRequest permission plus the access tokens it mints.
- **Tunnel library.** remotedialer or Konnectivity, and how its protocol is versioned alongside `/v1`.
- **Provider-side multi-replica.** Sharding vs. the Lease-based routing layer.
- **Provider SDK.** Token verification, key pinning and rotation handling should be shipped as a library, not left to each Provider.
- **`direct` mode.** Whether the user chooses it, or the connector detects reachability.
- **Addressing changes.** A new front-proxy URL, or a new issuer, means a new connection. Is a dedicated update needed, authorized by a new onboarding token **and** a call token that verifies against the pinned anchor?
- **Request permissions.** Whether creating a request should also require create permission on `Provider`.
- **Token format and portal UX.** Encoding, length, and whether the portal offers a "copy as Secret manifest" button.

## Prior art

- **kubeadm bootstrap tokens:** `tokenID.tokenSecret`, stored hashed, short-lived.
- **Kubernetes ServiceAccount issuer discovery** and **workload identity federation** (e.g. cloud providers trusting a cluster's SA tokens): an external party trusts a cluster's SA tokens by pinning its issuer and keys.
- **Rancher `remotedialer`** and **Konnectivity (apiserver-network-proxy):** multiplexed reverse tunnels with dial callbacks.
- **Railgrid edges provider:** bound SA tokens as the only post-enrolment credential, UID-bound identities, and Lease-based tunnel routing across replicas. Also shows the limit v3 avoids: a holder that can't mint its own tokens runs out of time after an outage.
- **Open Cluster Management:** bootstrap kubeconfig plus registration; cluster-proxy for the reverse tunnel.
- **Karmada pull mode:** the pull-mode alternative.
