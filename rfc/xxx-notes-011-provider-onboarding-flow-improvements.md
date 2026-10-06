# Notes: RFC 011 provider onboarding flow

Working notes for [011](./011-provider-onboarding-flow-improvements.md). Not the RFC itself.

## Terminology

- **PM-side**: the PM instance (kcp + PM operator). May be behind NAT; dials out.
- **Provider-side**: the service provider (controller + portal in its runtime cluster). Serves `/onboard` and `/connect`, and reconciles on the PM-side.
- **Connection**: the relationship between one `Provider` object on the PM-side and one cluster record on the Provider-side, identified by `connectionID`. It's created by `/onboard`, re-established by every `/connect`, and ended by revocation or offboarding.
- **Connection key**: a per-connection keypair held by the PM-side. The onboarding token is bound to it, and it authenticates the PM-side on `/onboard` and every `/connect` through a signed proof of possession. The Provider-side stores only the public key.
- **Connector**: the PM-side component that runs `/connect`. It mints fresh SA tokens, holds the tunnel, and acts as an allow-listed dial proxy. Where it runs is an open item.
- **`ProviderOnboardingRequest`**: a short-lived PM-side resource that drives `/onboard` and creates the `Provider`.

## Problem

- The Provider-side must reconcile resources on the PM-side, for any number of PM instances.
- The two sides don't necessarily share a network: the PM-side may be behind NAT, so it has to dial out.
- The PM-side owner gets a token from the Provider's portal. Authenticating that user is the portal's business and out of scope.
- Both sides must survive restarts, and outages of either side, without manual steps.

## Flow overview

1. **Create the request.** The user creates a `ProviderOnboardingRequest` with the Provider's base URL. The PM-side generates a connection keypair and shows its fingerprint in the request's status (phase `AwaitingToken`).
2. **Get a token.** In the Provider's portal, the user enters the fingerprint and gets a single-use, expiring token bound to it. The user stores the token in a Secret and references it from the request.
3. **Onboard.** The request calls `/onboard` with the token and a proof signed with the connection key. The Provider-side checks the token and the fingerprint binding, stores the public key on a new cluster record, and returns a `connectionID` and a `connectURL`. No PM-side credentials are sent.
4. **Create the `Provider`.** The request creates the `Provider` (or adopts an existing one) with `spec.connection`. The Provider controller creates the workspace, SA and RBAC (RFC 006).
5. **Connect.** The connector calls `/connect` with a proof signed with the connection key. On every connect it sends a fresh SA token, the CA bundle, the real URLs and the APIExportEndpointSlice name. The Provider-side reconciles through the tunnel (or directly).
6. **Clean up.** Once the `Provider` is Ready, the request is garbage-collected after a TTL. From then on only the `Provider` matters, and every restart is just another `/connect`.

## Design

### Onboarding token

The token has to do three jobs:

1. **Prove authorization.** It contains a secret that only the portal user received.
2. **Let the PM-side verify the Provider-side.** It carries a pin of the Provider-side's onboarding public key, so the PM-side can rule out a MITM or a misconfigured ingress before sending anything, even when TLS terminates at an ingress.
3. **Be single-use and expiring.**

The format follows kubeadm bootstrap tokens:

```
b1.<tokenID>.<tokenSecret>.<sha256 of the Provider-side's onboarding public key>
```

The pin names the onboarding public key rather than the serving CA because, behind an ingress that terminates TLS, the serving CA belongs to the ingress, so a CA pin would only authenticate the ingress. A Provider-side that terminates TLS itself may pin its serving CA instead.

The Provider-side stores only:

```
tokenID → {hash(secret), expiry, owner, connectionKeyFingerprint, usedBy}
```

**The token lives in a Secret**, referenced by `ProviderOnboardingRequest.spec.tokenSecretRef` and owned by the request. It is never inlined in the spec, because:

- **An unspent token is a live credential.** It's bound to the connection key, so a stolen token alone can't be redeemed by anyone else. It's still half of what's needed, so a Secret is defense in depth.
- **Failed requests are kept**, along with their possibly still-valid tokens.
- **RBAC separation.** Users and the UI read requests to see their status, and that shouldn't grant access to the token.
- **Specs leak** into GitOps repos, `kubectl get -o yaml` output and audit logs. Audit policies usually redact Secret bodies but not CR bodies.

### Mutual verification and the connection key

- **The PM-side verifies the Provider-side** through the pin in the token. The Provider-side signs the `/onboard` response, and later its first message on every `/connect`, with the onboarding key. The PM-side checks the signature against the pin before using or sending anything.
- **The Provider-side verifies the PM-side** through the binding. The token is bound to the connection key's fingerprint, and `/onboard` must carry a proof signed with that key. So the Provider-side knows it's being onboarded by exactly the party the portal user authorized, which covers 011's follow-up. This works without the Provider-side being able to reach the PM-side.

The connection key:

- is generated when the `ProviderOnboardingRequest` is created, before the token exists, so it can be bound;
- is held by the PM-side in a Secret. The request creates the Secret, and ownership moves to the `Provider` once it's created or adopted;
- authenticates `/onboard` and every `/connect` with a signed proof of possession (see below). The Provider-side pins the public key, so it needs neither a CA nor a certificate;
- is per connection, so a leak only affects one connection;
- is revoked by deleting the stored public key on the Provider-side;
- doesn't expire on its own (so there's no `notAfter` to track on the `Provider`);
- is rotated by sending a new public key over the tunnel, signed with the old one (details open).

**Proof of possession, not mTLS.** Public endpoints like `/onboard` and `/connect` usually sit behind an ingress or CDN that terminates TLS, and a client certificate doesn't make it past that. So every request instead carries a short-lived proof signed with the connection key, modelled on DPoP (RFC 9449):

- It's a JWS. On `/onboard` its header carries the public key, and the key's JWK thumbprint (RFC 7638) is the fingerprint the token is bound to. On `/connect` the Provider-side already has the key, so the proof only names the `connectionID`.
- Its claims are the HTTP method and URL, the issue time and a unique ID. On `/onboard` it also includes a hash of the token.
- The Provider-side rejects a proof that is stale, or whose unique ID it has already seen within the acceptance window.

This works the same whether or not TLS terminates in front of the Provider-side. The proof isn't bound to the TLS channel, so confidentiality between the ingress and the Provider-side is up to the ingress, or to the optional HPKE sealing below.

There is deliberately **no PM-instance-wide identity**; see "Rejected alternatives" and "Future layer: known PM instances".

### `/onboard` and `/connect`

Onboarding is two-phase.

- **`/onboard`** is driven by the request and runs once. It registers the connection key and creates the connection. It sends no PM-side credentials, because the provider workspace and SA don't exist yet.
- **`/connect`** is driven by the `Provider` and runs on every (re-)connect, including the first. It's the only way credentials reach the Provider-side.

So "resume" is not a special case: every connection is a resume.

```
POST /onboard
Authorization: DPoP <tokenID>.<tokenSecret>
DPoP: <proof signed with the connection key; its thumbprint must match the token binding>
{
  "onboardingID": "<ProviderOnboardingRequest UID>",  // idempotency key; the provider workspace doesn't exist yet
  "previousConnectionID": "...",                     // optional: re-onboarding an existing Provider
  "connectorVersion": "..."
}
→ 200, body is a JWS signed with the onboarding key:
{
  "connectionID": "...",                             // same as previousConnectionID when adopting
  "caBundle": "<CA for the Provider-side's /connect endpoint>",
  "connectURL": "wss://.../connect"
}

GET /connect   (upgrade to HTTP/2 or WebSocket)
DPoP: <proof signed with the connection key, naming the connectionID>
first message from the Provider-side, signed with the onboarding key:
{
  "connectionID": "...",
  "proofID": "<unique ID of the DPoP proof above>",   // binds the reply to this connect
  "sealKey": "<HPKE public key>"                     // optional, see payload encryption
}
then the PM-side's first message (optionally HPKE-sealed to sealKey):
{
  "connectionID": "...",
  "providerUID": "<UID of the Provider object>",
  "clusterID": "<logical cluster name of the provider workspace>",
  "transport": "tunnel" | "direct",
  "frontProxyURL": "https://...",
  "apiExportEndpointSlice": "<name>",
  "credentials": {
    "caBundle": "<front-proxy CA + shard serving CAs>",
    "token": "<fresh bound SA token>",
    "expirationTimestamp": "..."
  }
}
```

- **Idempotency.** `/onboard` is keyed on `(tokenID, onboardingID)`. If the response gets lost, a retry from the same request returns the same result instead of "token already used".
- **Cluster identity.** `clusterID` is the logical cluster name of the provider workspace (`root:providers:<name>-<suffix>`). It only exists after the `Provider` is created, so it's sent on `/connect`, not on `/onboard`.
- **Credentials go in the first message, not in headers.** Ingresses and proxies log upgrade headers but not what flows over the channel afterwards.
- **Binding to the `Provider` object.** The Provider-side records `providerUID` on the cluster record at the first `/connect`, and rejects a later `/connect` with a different UID until the connection is re-onboarded. So a `Provider` that is deleted and recreated under the same name can't continue the old connection. This is defense in depth, since the connection key Secret is garbage-collected with the `Provider` anyway.
- **The PM-side waits for the signed reply.** It sends credentials only after checking the Provider-side's first message against the pinned onboarding key and the proof it just sent, so whatever answers in front of the Provider-side can't receive them.
- **Payload encryption (optional).** When the Provider-side terminates TLS itself, TLS already gives confidentiality. If TLS terminates at an ingress, the credentials can be sealed with HPKE (RFC 9180) to `sealKey`, with `connectionID` as associated data. `sealKey` is a separate encryption key, vouched for by the onboarding key's signature on the reply. A signing key shouldn't double as an encryption key.

### Transport

**The tunnel is a dial proxy** (Konnectivity / remotedialer style), not a pipe to a single apiserver.

- The Provider-side keeps the **real URLs**: the front-proxy, plus the shard virtual-workspace URLs from the APIExportEndpointSlice. Only its `Dial` goes through the tunnel; the connector opens the actual TCP connection.
- **One multiplexed connection.** Every `Dial` becomes a stream over the `/connect` connection, opened with its target `host:port`. The connector checks the target against the allow-list, dials it and pipes bytes. The alternative, a fresh outbound connection from the connector for each dial, costs a full TCP, TLS and WebSocket handshake every time and keeps one connection open per watch.
- **TLS is end to end.** `ServerName` is the real URL's host, so no URLs are rewritten. `caBundle` is the front-proxy CA plus the shard serving CAs. The connector is a dumb pipe that can't read or alter traffic.
- **Allow-list.** The connector only dials the front-proxy and the hosts in the APIExportEndpointSlice, which it watches. Shard hostnames can be internal-only.
  - The list is built only from PM-side objects, never from anything the Provider-side sends. An empty list refuses every dial.
  - Each hostname is resolved once, every resulting address is checked, and the connector then dials those exact addresses. Otherwise DNS could be re-pointed between the check and the dial (DNS rebinding).
  - Link-local, multicast and cloud-metadata addresses are always refused.
- **Liveness.** The control channel runs its own ping/pong with read deadlines. Proxies and CDNs drop idle connections without telling either end, and TCP keepalive doesn't notice.
- **multicluster-runtime** per-endpoint clients are derived from the base `rest.Config`, so they all inherit the `Dial`.

**`direct` mode.** `/connect` negotiates `transport: tunnel | direct`. In `direct` mode the Provider-side dials the real URLs itself. Onboarding, credentials, rotation and revocation are the same in both modes.

**ManagedProvider (RFC 006) is unchanged.** The PM operator deploys the provider itself, so trust is implicit and no handshake is needed. Externally run Providers always go through `ProviderOnboardingRequest`.

### Credentials, rotation and RBAC

- **Send parts, not a kubeconfig:** the CA bundle, the token and the real URLs, on `/connect`.
- **Baseline: a fresh SA token on every `/connect`.** The connector calls TokenRequest (`serviceaccounts/token`), which kcp supports. Two-phase onboarding requires this, since `/connect` is the only way credentials are delivered. It also covers Provider-side outages longer than the token's TTL: the Provider-side just gets a new token on reconnect.
- **Why a long-lived key plus short-lived tokens.** If a credential could only be refreshed by presenting it, its TTL would become a power-off budget: any outage longer than the TTL would need manual re-onboarding. Keeping the long-lived proof (the connection key) separate from the short-lived authorization (the SA token) avoids that.
- **Optional: Provider-side self-rotation** for long-lived connections, so the tunnel doesn't have to be re-established when the token nears expiry. It's an optimization only; correctness doesn't depend on it. Scope it to `create` on `serviceaccounts/token` with `resourceNames: [its-sa]`.
- **RBAC is owned by the PM-side.** The Provider controller creates the SA and RBAC in the provider workspace (RFC 006). The Provider-side doesn't request permissions.

### Resources

Both objects live in the same workspace.

```yaml
apiVersion: providers.platform-mesh.io/v1alpha1
kind: ProviderOnboardingRequest
metadata:
  name: my-provider
spec:
  address: https://provider.example.com/pm           # base URL of the Provider's onboarding server
  tokenSecretRef: { name: my-provider-token, key: token }   # added once the user has the token
  provider:
    name: my-provider                                # Provider to create (or adopt) on success
  ttlSecondsAfterReady: 3600                         # optional, default 1h
status:
  phase: Ready                                       # AwaitingToken → Onboarding → Ready
  connectionKeyFingerprint: "SHA256:..."             # enter in the Provider's portal to get a bound token
  connectionKeySecretRef: { name: my-provider-connection-key }   # ownership moves to the Provider
  providerRef: { name: my-provider }
  readyTime: "2026-10-06T12:00:00Z"                  # TTL counts from here
  conditions:
    - type: Onboarded         # token redeemed, connection key registered
    - type: ProviderCreated   # or adopted
    - type: ProviderConflict  # adoption rejected by the Provider-side
---
apiVersion: providers.platform-mesh.io/v1alpha1
kind: Provider
metadata:
  name: my-provider                                  # no ownerReferences: independent once created
spec:
  providerKubeconfigSecret: { ... }                  # existing fields (RFC 006), largely unchanged
  connection:                                        # new, set by the request at creation/adoption
    connectionID: "..."
    connectURL: wss://provider.example.com/pm/v1/connect
    caBundle: "..."                                  # CA for the Provider-side's /connect endpoint
    providerKeyPin: "SHA256:..."                     # onboarding key pin from the token; checked on every /connect
    keySecretRef: { name: my-provider-connection-key }
status:
  phase: Ready
  onboardedTime: "2026-10-06T11:59:00Z"              # survives deletion of the request
  clusterName: 2cyb4oxml4sv8o3r
  transport: tunnel
  conditions:
    - type: Connected         # control channel up
    - type: CredentialsValid
```

**`Provider`**
- Kept largely as-is (RFC 006). It gains `spec.connection` and handles (re-)connections.
- Independent once created: it has no `ownerReferences` to the request.

**`ProviderOnboardingRequest` lifecycle**
- **Short-lived.** Once `phase=Ready` and `ttlSecondsAfterReady` has passed, it's deleted automatically, like a Job's `ttlSecondsAfterFinished`. The token Secret, which the request owns, goes with it.
- **Failed requests are kept.** Garbage collection only applies to `Ready` requests. Requests stuck in `AwaitingToken`, or failed on a bad or mismatched token, an unreachable address or `ProviderConflict`, stay until someone deletes them.
- **Nothing needs it after success:**
  - the address is only used for `/onboard`;
  - the token is spent;
  - the connection key Secret is owned by the `Provider`;
  - the audit trail is on the `Provider` (`status.onboardedTime`, events);
  - idempotency only matters before `Ready`.

**Re-onboarding adopts the existing `Provider`**, never by name alone:
- If `spec.provider.name` exists, the request sends its `connectionID` as `previousConnectionID`.
- The Provider-side re-registers the new connection key on the **same cluster record**, but only if that record exists and belongs to the token's owner.
- On success, the request updates `Provider.spec.connection`. On rejection, it fails with `ProviderConflict`.

**Multiple PM instances per Provider** are supported. The Provider-side keeps **one cluster record per connection**, not per PM instance. A `Provider` can be created in any workspace that has bound the providers APIExport, so several users or orgs in the same PM instance can each onboard the same Provider, and the Provider-side doesn't need to know which connections share a PM instance.

### Restart and failure semantics

- **State:** the request's status is the source of truth only while it exists. After that it's the `Provider` (`spec.connection`, status) plus the connection key Secret.
- **Leases** are used on both sides, for different things. On the PM-side they're only for connector leader election, so that one replica holds a tunnel. On the Provider-side they can record which replica holds which tunnel (see the table). Neither records whether onboarding succeeded.

| Scenario | Behavior |
|---|---|
| PM-side restarts | The `Provider` has `spec.connection` and the key Secret, so it goes straight to `/connect`. |
| Provider-side restarts | Tunnels drop and connectors reconnect. The Provider-side rebuilds REST configs from its cluster record and the fresh token. |
| Both down past the token TTL | Covered by the fresh token on `/connect`. |
| Connection key revoked or lost | `CredentialsValid=False` on the `Provider`. Recovery needs a new request and token; the new request adopts the existing `Provider`. |
| Re-onboarding hits a `Provider` connected elsewhere | The Provider-side rejects `previousConnectionID`. The request fails with `ProviderConflict` and is kept. |
| Token bound to a different connection key | The Provider-side rejects `/onboard`. The request fails and is kept; the user needs a new token for the shown fingerprint. |
| Crash during `/onboard` | Retry is safe thanks to `(tokenID, onboardingID)` idempotency. |
| Crash after `/onboard`, before the `Provider` is created | The key Secret already exists and the `connectionID` is persisted first. The `Provider` is created idempotently on the next reconcile, without a second `/onboard`. |
| `/connect` rejected | If the Provider-side itself rejects it (bad proof, revoked key, UID mismatch), the `Provider` gets `CredentialsValid=False` and the connector backs off for a long time. A failure in anything in between, such as an ingress error or a 5xx, is transient. The Provider-side marks its own rejections so the connector can tell the two apart. |
| Connector reconnects while the old tunnel is still open | Both tunnels have the same `connectionID`, so the new one replaces the old one, which is then closed. Cleaning up the old tunnel must only remove its own registration (and Lease), never the one that replaced it. Otherwise the connection becomes unroutable while both ends look healthy. |
| Provider-side runs multiple replicas | A tunnel lands on one pod. Either shard so that pod owns that connection's reconciliation, or add a routing layer. For the routing layer, the pod holding the tunnel claims a Lease per connection naming its internal address and renews it while the tunnel lives, and other pods relay through that pod. The Leases are then the only input to the cluster record's `Connected` status, which a single reconciler writes. |

### Offboarding and revocation

- **Deleting the `Provider` on the PM-side** triggers a finalizer. It notifies the Provider-side over the tunnel, then deletes the SA, RBAC and connection key Secret. The Provider controller already cleans up the first two (RFC 006).
- **Deleting the cluster record on the Provider-side** deletes the stored public key, which revokes it. It also closes the tunnel and tells the connector, so the `Provider` can surface the status.

## Rejected alternatives

- **The original draft's flow:**
  - a public key as the token: not a secret, so it authenticates nothing;
  - a kubeconfig sent on `/onboard`: the workspace doesn't exist yet, and the URL is meaningless through a tunnel;
  - a separate `/resume` endpoint: replaced by `/connect`;
  - "the Lease exists" as the onboarding signal.
- **Inlining the token in the request spec**, or inlining it and having the controller move it into a Secret: the token still ends up in audit logs and GitOps, and the controller would rewrite user-authored spec.
- **A PM instance key** for mutual verification. The Provider-side would have to learn it:
  - out of band, which is the same as the connection key but with a bigger blast radius;
  - from a PM-hosted JWKS, ruled out by NAT;
  - via a CA, which needs a central authority.
  
  Rotations would also have to reach every Provider-side. See "Future layer: known PM instances" for when it would make sense.
- **Other durable identities:**
  - client cert via CSR: every Provider would need to run a CA;
  - shared secret: a bearer credential stored on both sides;
  - workload identity federation: we can't assume cross-domain trust.
- **mTLS with the connection key as a client cert:** the cert doesn't reach the Provider-side through an ingress or CDN that terminates TLS, which is the common setup for a public endpoint. Replaced by the signed proof of possession.
- **A fresh outbound connection per dial**, instead of streams over the `/connect` connection: a full handshake per dial, one connection held open per watch, and each extra connection has to be authenticated and bound to its connection separately.
- **Request creates the `Provider` before `/onboard` and deletes it on failure:** orphans if cleanup fails.
- **Request creates the workspace and SA itself:** duplicates the Provider controller and blurs the RFC 006 split.

## To carry into the final RFC

### Future layer: known PM instances

An optional layer on top of the baseline, for Providers that need **per-PM-instance policy**: allow-lists of trusted PM instances, billing or quotas per instance, or "connect my PM once" across several onboardings.

- Each PM instance gets an identity **vouched for by a CA the Provider trusts**, typically where the PM operator and the Provider share a trust domain.
  - Distribution: the Provider-side trusts the CA, not individual keys, so nothing has to reach behind NAT.
  - Rotation: a new key just gets a new certificate, and Provider-sides don't need to be told.
- During `/onboard`, the PM-side presents the instance certificate. For example, the instance key could sign the connection key, giving a chain from CA to instance to connection key.
- The Provider-side records the instance identity on the cluster record, so it can group connections by PM instance and apply policy.
- Baseline guarantees are unchanged; the layer only adds information.

### Endpoint paths and versioning

- `spec.address` is the **base URL** of the Provider's onboarding server and may include a path prefix. Routing is up to the Provider (ingress, sub-path mount).
- The Provider-side SDK serves **fixed paths relative to it**.
- The paths are **versioned**: `v1/onboard` and `v1/connect`. A newer SDK can serve several versions side by side, and the PM-side picks one. These notes use `/onboard` and `/connect` for short.
- `connectorVersion` is for diagnostics and non-breaking feature negotiation only.

## Open items

- **Connector placement.** Options:
  - inside the PM operator's Provider controller (currently assumed);
  - a separate deployment per PM instance;
  - one per connection.

  Considerations:
  - scale: one long-lived tunnel per connection;
  - spreading connections across replicas: a Lease per connection, or sharding;
  - network reach to the front-proxy and every shard;
  - TokenRequest permissions in every provider workspace;
  - blast radius: it holds connection keys and tokens;
  - coupling to PM operator upgrades and restarts.
- **Fingerprint UX.** How the user gets the fingerprint from the request's status into the Provider's portal, once per onboarding: copy and paste, QR code, or a "connect to PM" link?
- **Connection key rotation protocol.** Message format, triggers (periodic or on demand), and the overlap or grace period.
- **Onboarding key rotation.** Every `Provider` stores the pin in `spec.connection.providerKeyPin`, so rotating the Provider-side's onboarding key would break every existing connection. Options:
  - the Provider-side announces the next key over the tunnel, signed with the current one, and the PM-side updates the pin;
  - the pin covers a small, long-lived root key that signs short-lived onboarding keys.
- **Connection key algorithm.** e.g. Ed25519 or ECDSA P-256. It must be supported for JWS signatures (EdDSA or ES256) in the Provider-side SDK.
- **Old connection key Secret after adoption.** Who deletes it (the request or the Provider controller), and when: before or after the first successful `/connect` with the new key?
- **Leaked but not revoked connection key.** Only the Provider-side can revoke a key; the PM-side's only option is deleting the `Provider`, which offboards entirely. Options:
  - an emergency rotation over the tunnel, which doesn't help if the attacker can also connect;
  - a PM-side "revoke and re-onboard" action that notifies the Provider-side;
  - declare it out of scope, so recovery goes through the Provider's portal.
- **`direct` transport.** Who decides the mode, how reachability is detected, and whether `/connect` is still needed as a control channel in direct mode.

## Prior art

- **Open Cluster Management:** the klusterlet's bootstrap kubeconfig plus CSR-based registration, and cluster-proxy using apiserver-network-proxy (Konnectivity) for the reverse tunnel.
- **Rancher `remotedialer`:** a small, battle-tested Go library for the tunnel half.
- **Karmada pull mode.**
