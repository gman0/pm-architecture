---
title: Remote access — baseline
---

# RFC 012: Remote access — baseline

| Status  | Proposed   |
|---------|------------|
| Author  | TBD        |
| Created | 2026-11-03 |
| Updated | 2026-11-03 |

## Summary

This RFC defines the baseline for delivering a kcp workspace kubeconfig from a Platform Mesh instance to an external runtime cluster, over a single PM-initiated handshake. It introduces the `access` SDK (user-side), the `access-operator` (PM-side), an access token, and four CRDs (`ProviderAccessRequest`, `AccessGrant`, `AccessIdentity`, `Access`). It also extends the existing `Provider` CRD with an opt-in `spec.remote` struct that switches a Provider from local-kubeconfig mode to remote-delivery mode.

It covers the common network scenario where both sides can reach each other on HTTPS: PM dials the user-side `/access` endpoint, delivers credentials, and PM's kcp is directly reachable from the user-side cluster. Persistent tunnelling, session-based reconnection, and the front-proxy SNI router are deferred to [RFC 013](./013-remote-access-tunnel.md).

## Motivation

The "Provider" role in Platform Mesh is not special — any user who exposes an `APIExport`, `ContentConfiguration`, or similar resource plays it. The universal primitive is: **a user wants their external runtime cluster to hold a kubeconfig for a workspace they control on a PM instance**, so their operators can reconcile against that workspace from their own infrastructure.

Today, this handoff is manual. The Owner waits for a `Provider` to be bootstrapped, extracts the kubeconfig Secret from PM, copies it out of band to the Provider, and the Provider installs it on their cluster. That couples Owner and Provider, requires out-of-band Secret transport, and does not support third-party Providers serving many PM instances.

This RFC makes the handoff programmatic and PM-initiated. The Owner creates a `ProviderAccessRequest` on their PM, carrying a short access token obtained from the user-side cluster. PM dials the user-side `/access` endpoint using the token, delivers kcp URL + CA + a freshly minted SA token, and the user-side access SDK writes the kubeconfig locally. The operator mounts the Secret and reconciles against kcp directly.

**Goals:**

- Programmatic kubeconfig delivery to an external cluster, PM-initiated, no manual Secret transport.
- Support third-party Providers serving many PM instances with one deployment of the access SDK.
- Support both direct (single-tenant) and multi-tenant portal deployment patterns without protocol divergence.
- Keep the access token the only credential on the wire for the handshake.
- Keep PM unaware of any user-side cluster until a `ProviderAccessRequest` arrives carrying an access token for that cluster.

**Non-goals:**

- Persistent tunnels, tunnel reconnection, or session-based auth on `/access` (RFC 013).
- Extending the kcp front-proxy with SNI routing (RFC 013).
- A user-initiated flow where PM mints a token and hands it to the user ("copy PM's token to User"). Design is sketched in the overview and deferred.
- Converting local-mode Providers into remote-mode Providers in-place. Mode is immutable after creation; mode switch is delete-and-recreate.

## Terminology

| Term | Meaning |
|---|---|
| **Owner** | The PM-side workspace owner who wants to onboard a Provider. Creates `Provider` and `ProviderAccessRequest` on PM. |
| **Customer** | The person running operators against a PM workspace. In a direct deployment, same as Owner. In a portal deployment, a customer of the Provider's portal. |
| **Provider** (role) | A user who exposes APIExports, ContentConfigurations, etc. In a portal deployment, operates a shared access SDK on behalf of many Customers. |
| **User-side cluster** | The cluster where the access SDK runs. In a direct deployment, the Customer's own cluster. In a portal deployment, the Provider's cluster. |
| `Provider` (CRD) | Existing PM-side CRD (`providers.platform-mesh.io/v1alpha1`), extended by this RFC. |
| `ProviderAccessRequest` / **PAR** | New PM-side CRD; one-shot auth-material delivery. |
| `AccessGrant` | New PM-side CRD; PM's view of the live connection. |
| `AccessIdentity` | New user-side CRD; standing authorization for a PM instance connection. |
| `Access` | New user-side CRD; ongoing binding. |
| **access token** | Credential minted by the access SDK on `AccessIdentity` creation (wire format: `pma1.…`). |
| **access SDK** | User-side binary (`platform-mesh-access`, `go.platform-mesh.io/sdk/access`). |
| **access-operator** | PM-side binary (`access-operator`). |
| **provider reconciler** | The controller inside `pm-operator` (later `provider-operator`) that reconciles `Provider` and PAR objects. |

## Architecture

### Components

**`access` SDK (user-side).** One deployment handles any number of PM instance connections. Reconciles `AccessIdentity` objects: on creation, generates `(tokenID, HMAC secret)` and emits an access token into an owned Secret alongside the identity. Serves `POST /access` publicly over HTTPS. Verifies HMAC on incoming calls, writes kubeconfig Secret from the request body, creates `Access` on acceptance. Manages the delivered kubeconfig's lifecycle (SA token renewal via `TokenRequest`, CA bundle rotation via well-known object polling, offboarding detection via persistent-401 handling).

**`access-operator` (PM-side).** Runs in PM's runtime cluster alongside `pm-operator`. Watches `AccessGrant` objects. For each `Pending` grant: mints an SA token via kcp `TokenRequest` for the SA referenced in the grant, derives a short-lived call credential from the stored access token, and `POST`s `/access` with the HMAC credential and a request body carrying `kcpURL`, `kcpCABundle`, and `saToken`. On `200` marks the grant `Active`. One handshake per grant; re-handshakes only when `AccessGrant.spec.accessToken` changes (recovery via a new PAR).

**Provider reconciler.** Lives inside `pm-operator` (long-term: a dedicated `provider-operator`). Reconciles `Provider` and `ProviderAccessRequest`. On a PAR referencing a remote-mode `Provider`, bootstraps the Provider (Workspace, SA, RBAC — same as local mode), extracts the `/access` URL from the access token, and creates an `AccessGrant` in the Provider Workspace. Reflects `AccessGrant.status` onto `Provider.status.remote`.

### `Provider` extension

```go
type ProviderSpec struct {
    // Existing fields. Populated only in local mode.
    ProviderKubeconfigSecret *KubeconfigSecretSpec `json:"providerKubeconfigSecret,omitempty"`
    HostOverride             string                `json:"hostOverride,omitempty"`

    // New. Presence opts into remote mode.
    Remote *RemoteSpec `json:"remote,omitempty"`
}

type RemoteSpec struct {
    // Optional PEM-encoded CA bundle for verifying the access SDK's /access TLS endpoint.
    // Omitted → access-operator uses system CAs (public web PKI).
    CABundle string `json:"caBundle,omitempty"`
}

type ProviderStatus struct {
    // Existing fields unchanged.
    // ...

    // New. Populated in remote mode.
    Remote *RemoteStatus `json:"remote,omitempty"`
}

type RemoteStatus struct {
    TokenID   string `json:"tokenID,omitempty"`   // from the access token, for OOB verification
    AccessURL string `json:"accessURL,omitempty"` // where PM dials; informational
}
```

Mode is immutable after creation. Mode switch requires deleting and recreating the `Provider`.

In remote mode the reconciler still bootstraps the Workspace, SA, and RBAC. It skips the local kubeconfig Secret and instead manages an `AccessGrant` living in the Provider Workspace.

### `ProviderAccessRequest`

Cluster-scoped. One-shot delivery of an access token to a `Provider`.

```go
type ProviderAccessRequestSpec struct {
    ProviderRef corev1.LocalObjectReference `json:"providerRef"`
    AccessToken string                      `json:"accessToken"`
}

type ProviderAccessRequestStatus struct {
    Phase      string             `json:"phase,omitempty"`      // Pending | Reconciling | Completed | Failed
    Reason     string             `json:"reason,omitempty"`     // on Failed
    Conditions []metav1.Condition `json:"conditions,omitempty"`
}
```

**Reconciliation rules:**

- **Order-independent.** If the referenced `Provider` doesn't exist yet, PAR sits in `Pending` until it does. If the `Provider` exists but lacks `spec.remote`, PAR goes `Failed` with `ProviderNotRemoteMode`.
- **One active PAR per `Provider`.** New PARs arriving while an older non-terminal PAR exists for the same `Provider` go `Failed` immediately with reason `AnotherPARActive: <older-par-name>`. Oldest wins.
- **Terminal states:** `Completed` / `Failed`. Non-terminal PARs are never auto-touched; stuck PARs are force-deleted.
- **Cascades:** deleting the `Provider` removes its PARs.
- **No admission webhooks.** All enforcement is reconciler-side via `status`.

### `AccessGrant`

Lives in the Provider Workspace. Created by the provider reconciler.

```go
type AccessGrantSpec struct {
    SAref     corev1.LocalObjectReference `json:"saRef"`      // SA on kcp to mint tokens from
    AccessURL   string                    `json:"accessURL"`   // extracted from the access token
    AccessToken string                    `json:"accessToken"` // full `pma1.…` wire string
    CABundle  string                      `json:"caBundle,omitempty"` // propagated from Provider.spec.remote.caBundle
}

type AccessGrantStatus struct {
    Phase      string             `json:"phase,omitempty"`      // Pending | Active
    Conditions []metav1.Condition `json:"conditions,omitempty"`
}
```

`spec.accessToken` is used for the initial handshake and for re-onboarding when a subsequent PAR delivers a fresh access token. Removed when the Provider Workspace is deleted (which happens on `Provider` deletion).

### `AccessIdentity` (user-side)

Standing authorization for a PM instance connection. Lives on the user-side cluster.

```go
type AccessIdentitySpec struct {
    // How incoming /access calls are handled.
    Accept      AcceptMode `json:"accept"`                // Always (default) | Manual
    MaxPending  int32      `json:"maxPending,omitempty"`  // cap on pending queue size
    PendingTTL  *metav1.Duration `json:"pendingTTL,omitempty"`
}

type AccessIdentityStatus struct {
    // HMAC verification material surfaced for debugging; real HMAC secret in owned Secret.
    TokenID string `json:"tokenID,omitempty"`

    // Pending queue (populated when spec.accept: Manual).
    PendingConnections []PendingConnection `json:"pendingConnections,omitempty"`
}
```

On creation, the access SDK generates `(tokenID, HMAC secret)` and writes an access token into an owned Secret keyed by the identity. One `AccessIdentity` → many `Access` objects over time.

### `Access` (user-side)

Ongoing binding. Created by the access SDK on a successful `/access`. References the kubeconfig Secret. In tunnel mode ([RFC 013](./013-remote-access-tunnel.md)) each `Access` additionally carries its own session-signing key, `jti` tracking state, and a stable local proxy endpoint URL — held in access SDK state, not on the CRD.

```go
type AccessSpec struct {
    IdentityRef corev1.LocalObjectReference `json:"identityRef"`
    KubeconfigSecretRef *corev1.LocalObjectReference `json:"kubeconfigSecretRef,omitempty"`

    // Follow-up: PM-wide identifier the user-side receives in the /access
    // handshake, so the user can see which Access belongs to which PM
    // instance. Shape and semantics TBD.
    // PMInstance PMInstance `json:"pmInstance"`
}

type AccessStatus struct {
    Phase string `json:"phase,omitempty"`   // Active | Revoked
    LastRefreshed *metav1.Time `json:"lastRefreshed,omitempty"`
    Conditions []metav1.Condition `json:"conditions,omitempty"`
}
```

### Access token: wire format

Prefix-tagged dot-separated string:

```
pma1.<base64url(URL)>.<tokenID>.<secret>
```

The `pma1.` prefix identifies this as a Platform Mesh access token, version 1. Clients split on dots and parse by position. Everything else in this document refers to the token as "the access token"; `pma1` appears only on the wire.

- The URL pins the recipient (`https://access.example/access`), so the token cannot be redirected. The URL carries host only, not a protocol version — see [Protocol versioning](#protocol-versioning).
- `tokenID` is safe to log. It is also surfaced in `Provider.status.remote.tokenID` for out-of-band verification.
- `secret` is the HMAC credential. access-operator derives a short-lived call credential from it per `/access` dial.

The access SDK generates `(tokenID, secret)` on `AccessIdentity` creation and writes the full access token string into an owned Secret. The access token is long-lived per `AccessIdentity`; rotating it requires minting a new `AccessIdentity` and issuing a new PAR (see [Rotation and revocation](#rotation-and-revocation)).

### Call credential: per-`/access` authentication

Each `POST /access` dial carries an authentication credential in the `Authorization` header. The credential is a short-lived JWT, signed with the access token's HMAC secret, that binds authentication to this specific request body.

**Wire shape:**

```
Authorization: Bearer <jwt>
```

JWT header:
- `alg` — `HS256`.

JWT body:
- `sub` — `tokenID` (from the access token). Identifies which `AccessIdentity` this dial is for.
- `iat`, `exp` — standard. `exp - iat` is short; default **60 s**.
- `jti` — random 128-bit, unique per issuance. Enables replay tracking on the SDK side.
- `body_sha256` — base64url-encoded SHA-256 of the raw request body. Binds the credential to this exact payload.

**access-operator (signer):**

1. Compute `sha256(body)` of the serialized `/access` request body.
2. Build the claims above using `AccessIdentity.tokenID` from the stored access token.
3. Sign with `HS256` using the access token's HMAC secret.
4. Send on `Authorization: Bearer <jwt>`.

**access SDK (verifier):**

1. Parse the JWT without verifying the signature yet; extract `sub`.
2. Look up `AccessIdentity` by `tokenID = sub`. Unknown → `403 UnknownIdentity`.
3. Verify signature using the stored HMAC secret and standard JWT claim checks (`exp`, `iat`, `alg == HS256`). Failure → `403 InvalidCredential`.
4. Recompute `sha256(body)` from the incoming request body. Compare to `body_sha256`. Mismatch → `403 BodyMismatch`.
5. Check `jti` against a short-lived replay cache keyed by `(tokenID, jti)`, TTL ≥ `exp - iat` plus clock-skew margin. Present → `403 Replay`. Otherwise record it.
6. Proceed with the handshake (accept mode, pending queue, etc.).

**Why this shape:**

- JWT is widely supported; standard claim checks (`exp`, `iat`, signature) come free from any mature JWT library.
- `body_sha256` prevents a captured credential from being replayed against a different payload.
- `jti` replay cache closes the remaining window within `exp`. Short `exp` (60 s) bounds the cache size.
- The same JWT primitive is reused in RFC 013 for session tokens on the reconnect plane (signed differently — per-`Access` asymmetric key instead of the Identity's HMAC secret). One mental model across both authentication planes.

**Clock skew.** JWT verifiers accept a configurable tolerance on `exp` / `iat`; default ±30 s. In NTP-synchronized environments this is comfortable.

**Algorithm discipline.** The verifier rejects any token whose header `alg` is not `HS256`. This prevents algorithm-confusion attacks (a classic JWT pitfall) where a token signed with a different algorithm could be confused with an HMAC-signed one.

**Distinguishing from the session token (RFC 013).** The call credential is `HS256`, signed with the access token's HMAC secret. The RFC 013 session token is asymmetric (`EdDSA` or `ES256`), signed with a per-`Access` private key. Access SDK routes verification by `alg` in the header and by the handshake context. No overlap.

### Protocol versioning

`/access` is served under a TLS connection negotiated via ALPN. The baseline uses the ALPN token `pm-access/1`; future protocol shapes get new tokens. access-operator advertises its supported tokens in `ClientHello`; access SDK picks the highest mutually-supported. No overlap → TLS handshake fails with an ALPN alert.

Feature flags within a version (e.g., opting into RFC 013 tunnel mode on a handshake) are negotiated in the request/response body — additive within an ALPN-selected shape.

The URL inside the access token pins only the host; a long-lived access token keeps working as the SDK rolls through protocol versions.

There is no separate discovery endpoint. ALPN is the version-selection primitive; feature detection lives in the message body.

### TLS trust

access-operator verifies the `/access` server cert against `AccessGrant.spec.caBundle` if set, otherwise system CAs. The bundle is propagated from `Provider.spec.remote.caBundle` by the provider reconciler at `AccessGrant` creation time.

Rotation of the CA is a `Provider.spec` edit. The provider reconciler re-propagates to the existing `AccessGrant` and subsequent `/access` dials use the new bundle. The access token and the `caBundle` rotate independently.

## Flows

### Phase 1: Registration (one-time)

```
<creator>  ──(1) creates AccessIdentity in user-side cluster ──►  access SDK
                                                                  [mints access token → owned Secret]
Owner      ──(2) creates Provider (spec.remote: {}) on PM
Owner      ──(2) obtains access token, creates ProviderAccessRequest (providerRef, accessToken) on PM
                                                                  ──►  provider reconciler
                                                                       [bootstraps Provider: Workspace, SA, RBAC]
                                                                       [extracts /access URL from access token]
                                                                       [creates AccessGrant (Pending) in Provider Workspace,
                                                                        pre-populated with SA ref, /access URL, accessToken, caBundle]
                                                                       [marks PAR Completed]
```

Who creates the `AccessIdentity` depends on the deployment model. In a direct deployment, the Customer applies it to their own cluster. In a multi-tenant portal deployment, the Provider's portal applies it on the Provider's cluster and surfaces the minted access token in its UI for the Customer to copy. Protocol-wise, the two modes are identical.

`Provider` and PAR creation on PM are order-independent. Whichever arrives first waits for the other.

### Phase 2: Credential delivery

```
access-operator  ──(4) picks up AccessGrant (Pending)
access-operator  ──(4) mints SA token via kcp TokenRequest
access-operator  ──(4) POST /access ──────────────────────────────────►  access SDK
                       Authorization: Bearer <JWT signed with access token HMAC>
                       body: { kcpURL, kcpCABundle, saToken }
                       [JWT: sub=tokenID, body_sha256, jti, iat, exp;
                        see "Call credential"]
                 ◄──── 202, tokenID (accept: Manual) ────────────────
                       [access-operator retries until approved or TTL]
                 ◄──── 200 (ack; empty body) ──────────────────────────
access-operator      ──(5) marks AccessGrant Active
access SDK           ──(5) writes kubeconfig Secret from request body, creates Access
provider reconciler  ──(5) sets Provider.status.conditions[Connected]=True; phase → Ready
operator             ──(6) talks to kcp directly
```

Request body credentials flow one-way: PM → user-side. The response is an acknowledgement (empty body in the baseline). access-operator verifies the TLS server cert against `AccessGrant.spec.caBundle` or system CAs before sending the request.

**Accept modes:**

- `spec.accept: Always` (default) — access SDK accepts immediately on valid HMAC.
- `spec.accept: Manual` — access SDK adds a pending entry to `AccessIdentity.status.pendingConnections` (keyed by `tokenID`) and returns `202`. The Provider approves by patching the identity. access-operator retries periodically. Denial or TTL returns `403`.

`spec.accept` applies to the one-shot `/access` call only.

`/access` is a one-shot call per connection: once step 5 completes, PM does not dial `/access` again for this `AccessGrant`. Re-onboarding (after SA revocation PM-side, or deliberate rotation) is a new PAR against the same `Provider`, carrying a fresh access token.

## Lifecycle on the user-side cluster

After step 5, the access SDK owns the delivered kubeconfig:

- **SA token renewal:** access SDK periodically renews via `TokenRequest` against kcp. The operator never sees the handshake; it just reads the Secret.
- **CA bundle rotation:** access SDK polls a well-known object in the workspace maintained by access-operator (Decision 4 in the overview; see [Operations](#operations)). Rewrites the kubeconfig when the bundle changes.
- **Offboarding detection:**
  - `401` after the delivered JWT's `exp` → token expired; access SDK renews.
  - Persistent `401` before `exp` (including repeated renewal failures) → SA revoked PM-side. Access SDK deletes the Secret and `Access`, emits a Pod Event, reports not-ready. Operator loses credentials and stops reconciling.
  - `403`, 5xx, network errors are not treated as offboarding.
- **Re-onboarding:** create a new PAR on PM. No user-side state machine needed for recovery — the user-side is passive.

## Rotation and revocation

### User-side levers

- **Delete `Access`** → cut one PM instance (operator loses that kubeconfig; other `Access` objects and the `AccessIdentity` unaffected).
- **Rotate `AccessIdentity` HMAC secret** (new access token) → blocks future handshakes under this identity. Existing `Access` objects continue working on their delivered SA tokens.
- **Delete `AccessIdentity`** → stops accepting new handshakes. Existing `Access` objects keep working until their delivered SA token is revoked PM-side or their `Access` is deleted.

### PM-side levers

- **Delete the `Provider`** (authoritative teardown): tears down the Provider Workspace, taking the `AccessGrant` with it; access-operator observes the deletion and stops. Provider's own teardown logic handles SA, Workspace, and RBAC.
- **Revoke the SA on kcp**: operator starts getting `401` against kcp. Access SDK propagates this to kubeconfig deletion via the offboarding-detection path.

### Rotation after a leaked access token

Rotate the HMAC secret on the `AccessIdentity` (mint a new access token). The attacker cannot open a new handshake under the stale token. Existing connections are unaffected because the baseline only uses the access token once, at initial handshake; no further verification of the stale token ever happens. The Owner creates a fresh PAR with the new access token to replace the stored value in `AccessGrant.spec`.

A leaked access token cannot be used by the attacker against a `Provider` that already has an active `AccessGrant` for a different access token — the attacker's call would be verified against the current HMAC secret, which they do not have.

## Operations

### CA-bundle well-known object

PM publishes its current CA bundle and kcp server URLs in a well-known object in each Provider Workspace. access-operator maintains the object (watching the CA source in PM's cluster, handling the old/new overlap window). access SDK polls the object using its SA token and rewrites the delivered kubeconfig when it changes. (See the overview's Decision 4 for the rationale; a dedicated ADR may follow.)

### Status surface

Follows the existing `Provider` convention: `status.phase` plus `status.conditions`, with phase computed from conditions. Adds remote-specific conditions on top of the local-mode set:

- `WorkspaceReady` — Provider Workspace bootstrapped.
- `Connected` — handshake completed; `AccessGrant` `Active`. Reasons on `Connected=False` include `Reconnecting`, `TunnelDown` (RFC 013), `SessionExpired` (RFC 013), `NeedsReonboarding` (RFC 013), `SARevoked`.
- `Ready` — aggregate.

Phase transitions: `Pending → Provisioning → Ready ⇄ Degraded`, and `Deleting`. `Degraded` means was-`Ready` before (`status.lastReadyTime`) and currently not; conditions say why. `status.remote` (sub-struct) carries informational fields only (`tokenID`, `accessURL`).

`AccessGrant.status` is PM-side infrastructure state; Customers should not rely on it directly. PAR status is per-reconciliation: `Pending` / `Reconciling` / `Completed` / `Failed` with reasons.

## Deployment models

### Direct

Customer = User = Owner. One cluster, one `AccessIdentity`, one PM instance. The Customer creates the `AccessIdentity` on their own cluster with any kube client, retrieves the minted access token from the owned Secret, and pastes it into a `ProviderAccessRequest` on their PM.

### Multi-tenant portal

A Provider runs the access SDK on their own cluster and operates it on behalf of many Customers. One `AccessIdentity` per Customer PM instance, created by the portal (any Kubernetes client; typed helpers may be factored out of the access SDK as a library). The Customer's only footprint is their PM workspace; they log into the portal to retrieve an access token and paste it into a `ProviderAccessRequest` on their PM.

On-the-wire protocol is identical. This RFC does not specify portal internals.

## Security considerations

- **Credentials never flow in `/access` responses.** The 200 is an acknowledgement. Attackers intercepting the response learn nothing useful.
- **The access token's HMAC secret is long-lived.** Treat it like any shared-secret bearer token: transport out of band, do not log, rotate on suspicion of leak. HMAC-derived per-dial credentials limit replay windows.
- **`tokenID` is public.** Surfaced in logs and in `Provider.status.remote.tokenID`. Only the `secret` component is sensitive.
- **Trust boundary for TLS to `/access`.** If a `caBundle` is pinned on `Provider.spec.remote`, it is the only trust material used; system CAs are ignored. Operators running access SDK with private PKI should keep the TLS key across cert renewals so trust doesn't need to be rotated on every renewal.
- **SA token exposure.** The SA token lives in the kubeconfig Secret on the user-side cluster. Treated as any kube Secret: RBAC-restricted, etcd-encrypted per cluster policy.
- **Replay protection.** Each `/access` dial uses a fresh derived credential from the access token's HMAC secret with a timestamp or nonce; the access SDK rejects replays. (Exact mechanism TBD; HMAC-over-nonce is the straightforward choice.)
- **Abuse by a stolen `AccessGrant`.** Access-operator acts on `AccessGrant` in its own cluster; the PM runtime cluster must be secured as the primary trust boundary. A compromised PM runtime cluster can mint SA tokens and dial out, but this RFC does not re-secure against PM-side compromise.

## Future work

- **RFC 013 (tunnel mode).** Adds persistent tunnel and session-based reconnect for the case where PM's kcp is not publicly reachable from the user-side cluster.
- **OIDC call credentials.** Using kcp-issued audience-bound JWTs as the `/access` call credential, replacing `(tokenID, secret)` with `(issuer, subject)` in `AccessIdentity`. Particularly interesting in multi-tenant portal deployments where PM's kcp is publicly reachable.
- **User-initiated flow.** Design is sketched in the overview as "copy PM's token to User." Implementation deferred until cross-org delegation becomes a priority.
