# Remote access: overview notes

Private working notes. State as of 2026-11-03.

Working baseline for remote access. Individual RFCs are derived from it.

See [`xxx-notes-011-v6-decisions.md`](./xxx-notes-011-v6-decisions.md) for
related decisions and e2e verification results.

---

## Reframe

"Provider" is not a special role. It is any user who exposes an APIExport,
ContentConfiguration, or similar resource. Today that is limited to workspaces
under `:root:providers:`, but the same need exists for any user with the right
permissions.

The universal primitive is: **a user wants their external runtime cluster to
hold a kubeconfig for a workspace they control on a PM instance.** Their
operators run against that workspace from their own infrastructure.

On top of this primitive, the **access SDK** manages kubeconfig Secrets in
the user-side cluster: SA token renewal, CA bundle rotation, offboarding
detection. It serves the public HTTPS `/access` endpoint that access-operator
dials to deliver credentials (PM-initiated), and mints access tokens on
`AccessIdentity` creation (owned Secret, HMAC secret stored for
verification). Tunnel mode is opt-in (RFC 3): keeps the `/access` connection
alive as a reverse L4 channel when PM's front-proxy is not directly
reachable, and adds session-based reconnect authentication on top of the
baseline handshake.

The access SDK lives in `go.platform-mesh.io/sdk`.

**Trust-boundary / deployment-model note.** "User-side" throughout this
document means "wherever the access SDK runs." The two common deployment
models:

- **Direct**: the Customer (= User = Owner) runs the access SDK on their
  own cluster. One cluster, one identity, one PM instance.
- **Multi-tenant portal**: a Provider runs the access SDK on their own
  cluster and operates it on behalf of many Customers, each with their own
  PM instance. One `AccessIdentity` per Customer PM instance, created by
  the portal (any Kubernetes client works; typed helpers may be factored
  out as a library). The Customer's only footprint is their PM workspace;
  they log into the portal to retrieve an access token and paste it into a
  `ProviderAccessRequest` on their PM. On-the-wire protocol is identical
  to the direct model.

---

## Cast

| Term | Who or what |
|---|---|
| **User** | Any person controlling a workspace in PM. May also be a Provider. |
| **Provider** | A user who exposes APIExports, ContentConfigurations, etc. |
| **Owner** | The PM-side workspace owner who wants a Provider's service. Often the same person as the User. |
| **Provider Workspace** | The kcp workspace containing the Provider's APIExports, created by PM for the Provider. |
| **`Provider`** | PM-side CRD (`providers.platform-mesh.io/v1alpha1`). Cluster-scoped. Pre-existing resource for kcp-side bootstrap (Workspace, SA, RBAC); extended with `spec.remote` (nested struct, presence-based opt-in) to switch from local mode (kubeconfig Secret stored in PM) to remote mode (kubeconfig delivered to a user-side cluster via `AccessGrant`). `spec.remote.caBundle` (optional, PEM) carries the trust bundle for the access SDK's `/access` TLS endpoint; omitted → system CAs (public web PKI). Mode is immutable after creation. Local kubeconfig Secret is skipped in remote mode. Persistent lifecycle handle: owns `AccessGrant` and PARs; `status` follows the existing `phase` + `conditions` convention (phase computed from conditions; `Pending → Provisioning → Ready ⇄ Degraded`, `Deleting`). Remote-specific conditions: `Connected` (with reasons like `Reconnecting`, `NeedsReonboarding`). `status.remote` sub-struct carries informational fields only (`tokenID`, `accessURL`). Deleting the `Provider` is the authoritative teardown signal. |
| **PM Instance** | One deployment of Platform Mesh (kcp + pm-operator). May be public or private. |
| **Provider's portal** | In multi-tenant deployments, the Provider's website/system for managing customers. Creates `AccessIdentity` objects on the Provider's cluster (any Kubernetes client) and surfaces the minted access token to the Customer. PM has no opinion on its internals. |
| **`access-operator`** | PM-side (`platform-mesh/operators/access-operator`, binary: `access-operator`). Watches `AccessGrant` objects in `Pending` state (created and pre-populated by the provider reconciler). For each: mints an SA token via kcp TokenRequest, derives a short-lived call credential from `AccessGrant.spec.accessToken`, and `POST`s `/access` (verifying the server cert against `AccessGrant.spec.caBundle` if set, otherwise system CAs) with the HMAC credential and a request body carrying `kcpURL`, `kcpCABundle`, `saToken`. Retries on `202` until `200` (ack) or `403`. On `200`: marks `AccessGrant` `Active`. Holds tunnel connections open with leader election (RFC 3); reconnects use session tokens (RFC 3 only). Runs in PM's runtime cluster. |
| **`access` SDK** | User-side (`go.platform-mesh.io/sdk/access`, binary: `platform-mesh-access`). Reconciles `AccessIdentity` objects: on creation, generates the HMAC secret and emits an access token via an owned Secret. Serves the `/access` endpoint: verifies HMAC against the identity, enforces `accept` mode, manages the pending queue. Manages `Access` objects (one per live connection). Self-renews SA tokens via TokenRequest. Handles CA bundle rotation and offboarding detection. In tunnel mode (RFC 3), additionally issues session tokens for reconnect authentication. |
| **`ProviderAccessRequest`** | PM-side CRD. One-shot auth-material delivery object. Owner creates it with `spec.providerRef.name` and an access token (obtained from the user-side cluster — in a portal deployment, from the Provider's portal UI). Reconciled by the provider reconciler (inside `pm-operator`; later `provider-operator`): waits in `Pending` until the referenced `Provider` exists with `spec.remote` set, then propagates the access token into the Provider's `AccessGrant`; terminal states `Completed` / `Failed`. One active PAR per `Provider`: new PARs arriving while an older non-terminal PAR exists go `Failed` with reason `AnotherPARActive`. Deleting the `Provider` cascades to its PARs. |
| **`AccessGrant`** | PM-side CRD living in the Provider Workspace. Created by provider reconciler in `Pending` state with SA ref, `/access` URL, access token credentials, and the `/access` TLS CA bundle propagated from `Provider.spec.remote.caBundle` (if set). access-operator drives it to `Active` by completing the initial `/access` handshake. `spec.accessToken` is used for initial handshake (first bind, and re-onboarding after a subsequent PAR delivers a fresh access token). In tunnel mode (RFC 3), gains reconnection-credential status fields. PM's view of the live connection. Removed when the Provider Workspace is deleted (which happens on `Provider` deletion). |
| **`AccessIdentity`** | User-side CRD. Standing authorization for a PM instance connection. Created by the Customer directly (direct deployment) or by the Provider's portal (multi-tenant deployment). On creation, the access SDK generates `(tokenID, HMAC secret)` for verifying `/access` handshakes and emits the full access token into an owned Secret. `spec.accept`: `Always` (auto) or `Manual` (pending queue). `spec.maxPending` caps simultaneous pending entries. One `AccessIdentity` → many `Access` objects over time. Rotate the HMAC secret (new access token) to block future handshakes under this identity — existing `Access` objects keep working on their delivered SA tokens. Delete to end the standing authorization. Per-PM revocation: delete the specific `Access`. (RFC 3 adds per-`Access` session-signing keys and session tokens for tunnel reconnects; no new crypto on `AccessIdentity`.) |
| **`Access`** | User-side CRD. Ongoing binding. Created by access SDK on successful connection. References the kubeconfig Secret. User cluster's view of the live connection. |

---

## Two handshake directions

RFC 1 covers PM-initiated only; see [Deferred: User-initiated flow](#deferred-user-initiated-flow).

### PM-initiated — PM dials User

*"Copy User's token to PM."*

**Requirements:**
- User's access SDK `/access` endpoint must be publicly reachable from PM.
- Tunnel mode on the access SDK is only needed if PM's kcp front-proxy is
  not directly reachable from the user's cluster.

PM-initiated has two distinct phases.

#### Phase 1: Registration (one-time)

```
<creator>  ──(1) creates AccessIdentity in user-side cluster ──►  access SDK
                                                                  [mints access token → owned Secret]
Owner      ──(2) creates Provider (spec.remote: {}) on PM
Owner      ──(2) obtains access token, creates ProviderAccessRequest
                 (providerRef: <provider>, accessToken) on PM  ──►  provider reconciler
                                                                    [bootstraps Provider: Workspace, SA, RBAC]
                                                                    [extracts /access URL from access token]
                                                                    [creates AccessGrant (Pending) in Provider Workspace]
                                                                    [marks PAR Completed]
```

1. An `AccessIdentity` is created in the user-side cluster. The access SDK
   generates the HMAC secret and emits the access token via an owned Secret.
   Who creates it and how the access token reaches the Owner depends on the
   deployment model (see Reframe above): the Customer applies it directly on
   their own cluster, or a Provider's portal applies it on the Provider's
   cluster and surfaces the access token in its UI.
2. The Owner creates a `Provider` with `spec.remote: {}` and a
   `ProviderAccessRequest` referencing it by name (`spec.providerRef.name`),
   carrying the access token. Order-independent: whichever is created first waits
   for the other.
3. The provider reconciler bootstraps the `Provider` (Workspace, SA, RBAC —
   same as local mode but skips the local kubeconfig Secret), extracts the
   `/access` URL from the access token, and creates an `AccessGrant` in `Pending`
   state pre-populated with the SA ref, `/access` URL, access token credentials,
   and the `/access` TLS CA bundle (propagated from
   `Provider.spec.remote.caBundle`, if set). The `AccessGrant` lives in the
   Provider Workspace. The PAR is marked `Completed`.

#### Phase 2: Credential delivery and ongoing access

```
access-operator  ──(4) picks up AccessGrant (Pending)
access-operator  ──(4) mints SA token via kcp TokenRequest
access-operator  ──(4) POST /access ──────────────────────────────────►  access SDK
                       Authorization: Bearer <JWT signed with access token HMAC>
                       [JWT: sub=tokenID, body_sha256, jti, iat, exp]
                       body: { kcpURL, kcpCABundle, saToken }
                 ◄──── 202, tokenID (accept: Manual) ────────────────
                       [access-operator retries until approved or TTL]
                 ◄──── 200 (ack; empty body) ──────────────────────────
access-operator  ──(5) marks AccessGrant Active
access SDK              ──(5) writes kubeconfig Secret from request body,
                               creates Access
provider reconciler ──(5) sets Provider.status.conditions[Connected]=True; phase → Ready
operator                ──(6) talks to kcp directly
```

4. access-operator picks up the `AccessGrant` in `Pending` state. It mints an
   SA token via kcp TokenRequest for the SA referenced in the `AccessGrant`,
   derives a short-lived call credential from the access token in `AccessGrant.spec`,
   and calls `POST /access` on the access SDK — verifying the TLS server cert
   against `AccessGrant.spec.caBundle` (propagated from
   `Provider.spec.remote.caBundle`) if set, otherwise using system CAs. The
   request body carries `kcpURL`, `kcpCABundle`, and the freshly minted
   `saToken`. The access SDK verifies the HMAC credential against the
   `AccessIdentity`.
   - `spec.accept: Always` — accepts and proceeds to step 5.
   - `spec.accept: Manual` — creates a pending entry in
     `AccessIdentity.status.pendingConnections` with the `tokenID` and responds
     `202`. access-operator retries periodically. The Provider approves by
     patching the `AccessIdentity`; the next retry proceeds. Denial or TTL
     expiry returns `403`.
5. access SDK writes the kubeconfig Secret using the credentials from the
   request body, creates the `Access` object, and responds `200`.
   access-operator marks the `AccessGrant` `Active`. The provider reconciler
   sets `Provider.status.conditions[Connected]=True` and flips `phase` to `Ready`.
6. The operator mounts the Secret and talks to kcp directly.

`/access` is a one-shot call per connection: once step 5 completes, PM does
not dial `/access` again for this `AccessGrant`. Re-onboarding (e.g., after
SA revocation PM-side, or deliberate rotation) is a new PAR against the same
`Provider`, carrying a fresh access token.

**Tunnel (RFC 3):** after credential delivery, if PM's kcp is not directly
reachable, the same `/access` connection is upgraded to a persistent reverse L4
tunnel. The kubeconfig uses `proxy-url` pointing to a local access SDK endpoint
that routes traffic to PM's front-proxy SNI router.

PM's kcp front-proxy is extended with an L4 SNI router (new work).
A single tunnel connection per remote cluster carries all kcp traffic — both
front-proxy calls and per-shard VW calls — routed by the TLS SNI hostname
without terminating TLS. access-operator holds the tunnel open with
leader election and reconnects automatically on drop.

RFC 3 adds a tunnel-request flag to the `/access` request (set by
access-operator when PM's front-proxy is non-public) and a tunnel-policy
field to `AccessIdentity.spec` (`Allow` default, `Deny` opt-out). On an
accepted tunnel upgrade, access SDK writes the kubeconfig with a `proxy-url`
pointing at its own local tunnel endpoint; PM never sees or references that
URL. On policy mismatch (PM requires, user-side denies), `/access` returns
`403 TunnelDenied`.

Because tunnels drop and reconnect, RFC 3 also adds a
**reconnect-authentication plane** disjoint from the initial-handshake HMAC:
on the 200 response, access SDK issues a short-lived `sessionToken` (JWT
signed by a **per-`Access`** session-signing key, held in a Secret owned by
the `Access`); access-operator stores it on `AccessGrant.status` and
presents it on subsequent `/access` dials, with a fresh `sessionToken`
returned on each reply. HMAC and session-signing-key verification are
performed against disjoint keys and never fall back to each other. The SDK
tracks `current_jti`/`previous_jti` per `Access` and rejects stale `jti`s;
a short grace window (default 60 s) accepts `previous_jti` to tolerate lost
responses. If a reconnect's session token is rejected, `AccessGrant`
surfaces `NeedsReonboarding` and access-operator stops dialing; recovery is
a new PAR against the same `Provider`. The scope split (HMAC Identity-wide,
session key per-`Access`) keeps HMAC rotation from disturbing running
tunnels, and keeps one `Access`'s session-key rotation from disturbing
sibling `Access`es under the same Identity. Local proxy endpoint routing:
default is Proxy-Authorization carrying the `Access` UID (single hostname,
single cert); wildcard-DNS SNI routing and sidecar-per-operator loopback are
alternative deployment patterns.

**Kubeconfig:** direct (`server` → kcp front-proxy) when PM's kcp is reachable;
`proxy-url` through the access SDK's local tunnel endpoint when it is not.

---

## Network mode coverage

| PM reachable from User | User endpoint reachable from PM | Works | Notes |
|---|---|---|---|
| Yes | Yes | PM-initiated (RFC 1) | Direct kubeconfig; User-initiated deferred |
| Yes | No | — | PM cannot reach `/access`; User-initiated deferred |
| No | Yes | PM-initiated + RFC 3 | Credential delivery works; operator needs tunnel for kcp |
| No | No | — | No supported path |

"User endpoint reachable from PM" means the access SDK's `/access` endpoint is
reachable from PM. "PM reachable from User" determines whether the delivered
kubeconfig can be direct or needs `proxy-url` via tunnel (RFC 3).

---

## Component details

### `access` SDK (User-side, `go.platform-mesh.io/sdk/access`, binary: `platform-mesh-access`)

One deployment handles any number of PM instance connections.

**`AccessIdentity` management:**

- Watches `AccessIdentity` objects. Each represents a standing authorization
  for one PM instance connection.
- `spec.accept: Always` (default): incoming `/access` calls with a valid HMAC
  credential are accepted immediately.
- `spec.accept: Manual`: incoming `/access` calls create a pending entry in
  `AccessIdentity.status.pendingConnections` (with `tokenID`) and return
  `202`. `spec.maxPending` caps the queue; entries expire after the pending
  TTL. Provider approves or denies by patching the `AccessIdentity`.
- Deleting an `AccessIdentity` stops accepting new handshakes under its HMAC
  secret. Existing `Access` objects and kubeconfig Secrets continue to
  function on their delivered SA tokens; the user-side operator (Customer in
  a direct deployment, Provider's portal in a multi-tenant deployment)
  deletes individual `Access` objects to cut specific PM connections.

**`/access` endpoint (PM-initiated):**

- Serves `POST /access`. access-operator presents a short-lived JWT in
  `Authorization: Bearer …` (HS256-signed with the access token's HMAC
  secret; claims include `sub=tokenID`, `body_sha256`, `jti`, `iat`, `exp`)
  and a request body carrying `kcpURL`, `kcpCABundle`, and the SA token
  access-operator minted.
- Looks up `AccessIdentity` by `tokenID`, verifies HMAC against stored secret.
- Writes the kubeconfig Secret from the request body and creates `Access`
  on acceptance.
- Returns `200` (ack; empty body in the baseline), `202` (pending), or
  `403` (denied/unknown/TTL).
- **Tunnel mode (RFC 3, opt-in):** after credential delivery, if PM's kcp is
  not directly reachable, the connection is upgraded to a persistent reverse L4
  tunnel. Exposes a per-connection local HTTPS proxy endpoint; the kubeconfig
  `proxy-url` points here. RFC 3 also adds session-token issuance on the 200
  response and session-token verification on reconnect dials (see Decision 2).

**Lifecycle:**

- Creates and manages `Access` objects (one per live connection).
- Self-renews SA token via TokenRequest periodically.
- CA bundle rotation: polls the well-known object in the workspace; rewrites
  kubeconfig when it changes.
- Offboarding detection:
  - 401 after JWT `exp` → expired.
  - Persistent 401 before `exp` (including failed renewal) → revoked; deletes
    the Secret and `Access`, emits a Pod Event, reports not-ready.
  - 403, 5xx, network errors: not counted as offboarding.
- Re-onboarding: create a new `ProviderAccessRequest` (PM-initiated).

### `access-operator` (PM-side, `platform-mesh/operators/access-operator`, binary: `access-operator`)

- Watches `AccessGrant` objects in `Pending` state (created and pre-populated
  by the provider reconciler with SA ref, `/access` URL, access token credentials,
  and `/access` TLS CA bundle).
- Mints an SA token via kcp TokenRequest for the SA in `AccessGrant`, derives
  a short-lived call credential from `AccessGrant.spec.accessToken`, and `POST`s
  `/access` (TLS verified against `AccessGrant.spec.caBundle` if set,
  otherwise system CAs) with the HMAC credential and a body carrying
  `kcpURL`, `kcpCABundle`, and the SA token. Retries on `202` (`accept:
  Manual`, pending approval) until `200` (ack) or `403` (denied/TTL
  expired). On `200`: marks `AccessGrant` `Active`. One handshake per
  `AccessGrant` — re-triggered only when `AccessGrant.spec.accessToken` changes
  (new PAR against the same `Provider`).
- RFC 3 adds tunnel-reconnection: session-token issuance on initial handshake,
  session-token presentation on reconnect dials, `NeedsReonboarding` on
  session-token rejection. See Decision 2.
- Stops when `AccessGrant` is deleted (which happens when the `Provider`
  is deleted and its Workspace is torn down).
- Holds tunnel connections open between PM and remote clusters (RFC 3). Leader
  election governs tunnel management only; credential delivery is stateless.
- Maintains CA bundle well-known object per workspace (Decision 4).
- Runs in PM's runtime cluster alongside pm-operator. Minimal privileges;
  separate from pm-operator.

---

## Token formats

| Token | Minted by | Lifetime | Encodes |
|---|---|---|---|
| Access token `pma1.…` | `access` SDK (on `AccessIdentity` creation) | Long-lived (per `AccessIdentity`) | access SDK `/access` URL, token ID, HMAC secret |

`pma1.<base64url(URL)>.<tokenID>.<secret>` — the URL pins the recipient so the
token cannot be redirected; the token ID is safe to log; the secret is the
HMAC credential. The access SDK generates `(tokenID, secret)` when it
reconciles a new `AccessIdentity` and writes the full access token string into an
owned Secret alongside the identity. access-operator derives a short-lived
call credential from the HMAC secret for each `/access` dial.

RFC 3 adds a second token type — a short-lived JWT session token issued by
access SDK on the `/access` 200 response and used by access-operator to
authenticate tunnel reconnects. See Decision 2.

---

## Protocol versioning

`/access` is served under a TLS connection negotiated via ALPN. The baseline
uses the ALPN token `pm-access/1`; future protocol shapes get new tokens
(`pm-access/2`, etc.). access-operator advertises its supported tokens in
`ClientHello`; access SDK picks the highest mutually-supported one. No
overlap → TLS handshake fails with an ALPN alert (clean, specific error).

Feature flags within a version (e.g., tunnel support, future OIDC credential
types) are negotiated in the request/response body — additive within an
ALPN-selected protocol shape.

The access token URL pins only the host, not the ALPN version, so a long-lived
access token keeps working as the SDK rolls through protocol versions. There is no
separate discovery endpoint: ALPN is the version-selection primitive, and
feature detection lives in the message body.

---

## Decisions

**1. SA token minting.** `access-operator` is a single binary. Owns
TokenRequest for PM-initiated connections. Leader election governs tunnel
management (RFC 3) only; credential delivery is stateless.

**2. Reconnection (RFC 3).** In the baseline (no tunnel), `/access` is called
once per connection; there are no reconnects. Tunnel mode adds a persistent
`/access`-rooted channel that drops and reconnects, so RFC 3 introduces a
reconnect-authentication plane disjoint from the initial handshake:

- On `Access` creation, the access SDK generates a **per-`Access`**
  session-signing key (held with `jti` tracking state in a Secret owned by
  the `Access`).
- On `/access` 200, access SDK issues a `sessionToken` (JWT signed by that
  key, short TTL). JWT body: `sub` = `Access` UID, `iat`, `exp`, `jti`.
  Header: `alg`, `kid` = SHA-256 fingerprint of the signing public key.
  access-operator stores it on `AccessGrant.status.sessionToken` +
  `.sessionTokenExpiresAt`.
- Reconnect dials present `sessionToken` only; access SDK verifies the JWT
  (resolve Access by `sub` → resolve key by `kid` → signature check → `jti`
  check). A fresh `sessionToken` is returned on each successful response and
  rolled back onto `AccessGrant.status`.
- **Replay tracking.** SDK tracks `current_jti` / `previous_jti` per
  `Access`. A grace window (default 60 s) accepts `previous_jti` to
  tolerate lost responses; outside that window → reject.
- No fallback: on session-token rejection access-operator stops dialing;
  the provider reconciler surfaces this as
  `Provider.status.conditions[Connected]=False` with
  `reason=NeedsReonboarding` and `phase=Degraded`. Recovery is a new PAR
  against the same `Provider`, which propagates the fresh access token into
  `AccessGrant.spec.accessToken` and triggers a fresh initial handshake
  (creating a new `Access` with a new signing key).
- HMAC and session-signing-key verification are performed against disjoint
  keys and never substitute for each other. Rotating the HMAC secret on
  `AccessIdentity` (new access token) blocks future initial handshakes under this
  identity but has zero effect on running tunnels. Rotating one `Access`'s
  session-signing key invalidates that one `Access`'s outstanding session
  tokens and drives its tunnel into `NeedsReonboarding`; sibling `Access`es
  under the same Identity are unaffected.
- **Local proxy endpoint URL.** Default: the kubeconfig `proxy-url` is
  `https://<access-uid>:@access-tunnel.svc:443` — Go sends `Access` UID in
  `Proxy-Authorization: Basic …` on CONNECT; SDK routes the CONNECT to the
  right per-`Access` tunnel backend. Alternatives: wildcard-DNS SNI routing
  or sidecar-per-operator loopback. The JWT `sub` is the authoritative
  identifier; the connection-level identifier is routing metadata.

Teardown is independent of tunnel mode: deleting the `Provider` tears
down the Provider Workspace, taking the `AccessGrant` with it;
access-operator observes the deletion and stops re-dialing. On the user side, `AccessIdentity` deletion
stops accepting new handshakes under that identity; existing `Access`
objects and kubeconfig Secrets remain. The user-side operator (Customer in
a direct deployment, Provider's portal in a multi-tenant deployment)
deletes individual `Access` objects to cut specific PM connections.

**3. Offboarding / revocation.** Deleting the `Provider` is the
authoritative teardown signal; it tears down the Provider Workspace
(taking the `AccessGrant` with it), which access-operator observes and
stops tracking (in tunnel mode, this also closes the tunnel). On the user side: persistent 401
against kcp → access SDK deletes the `Access` and kubeconfig Secret;
operator loses credentials and stops reconciling. In tunnel mode the same
401 chain propagates through the tunnel and additionally triggers tunnel
closure.

**4. CA bundle rotation.** PM publishes the current CA bundle and server URLs in
a well-known object in each workspace where a `AccessGrant` exists.
access-operator maintains this object (watches the CA bundle source in
PM's cluster, handles the old/new overlap window). access SDK reads it
periodically using its SA token and rewrites the kubeconfig when it changes.

**5. `Provider` and `ProviderAccessRequest` roles.** The persistent
lifecycle handle for a remote-access binding is the existing `Provider`
CRD, extended with an optional nested `spec.remote` struct. Presence of
`spec.remote` opts the Provider into remote mode: the reconciler still
bootstraps the Workspace, SA, and RBAC (as in local mode), but skips the
local kubeconfig Secret and instead manages an `AccessGrant` owned by the
`Provider`. Mode is immutable after creation; mode switch = delete and
recreate. `status` follows the existing `phase` + `conditions` convention
(`Pending → Provisioning → Ready ⇄ Degraded`, `Deleting`; phase computed
from conditions). Remote mode adds a `Connected` condition with reasons
(`Reconnecting`, `NeedsReonboarding`, etc.). `status.remote` sub-struct
holds informational fields only: `tokenID` and `accessURL`.

`Provider.spec.remote.caBundle` (optional, PEM-encoded) carries the CA
material access-operator uses to verify the TLS server cert on `/access`.
Omitted → access-operator uses system CAs (public web PKI). Set →
access-operator uses only this bundle. Rotation of the CA is a
`Provider.spec` edit; access token and caBundle are independent concerns and
rotate independently.

`ProviderAccessRequest` is a one-shot auth-material delivery object with
`spec.providerRef.name` and `spec.accessToken`. Reconciled by the provider
reconciler (inside `pm-operator`; later `provider-operator`), not by
`access-operator`. Rules:

- **Order-independent.** If the referenced `Provider` doesn't exist yet,
  PAR sits in `Pending` until it does. If the `Provider` exists but lacks
  `spec.remote`, PAR goes `Failed` with `ProviderNotRemoteMode`.
- **One active PAR per `Provider`.** New PARs arriving while an older
  non-terminal PAR exists for the same `Provider` go `Failed` immediately
  with `AnotherPARActive: <older-par-name>`. Oldest wins.
- **Terminal states.** `Completed` / `Failed`.
- **Deleting the `Provider`** cascades to all its PARs.
- **Enforcement is reconciler-side**, via `status`.

First bind: Owner creates `Provider` + PAR (either order). Reconciler
bootstraps and creates `AccessGrant`.

Re-onboarding / rotation: Owner creates a **new PAR** against the same
`Provider` with a current access token; reconciler propagates the access token into
`AccessGrant.spec.accessToken`; access-operator runs a fresh initial handshake.
(RFC 3 formalises the trigger via a `NeedsReonboarding` condition — see
Decision 2.)

Teardown: delete the `Provider`. The Provider Workspace is torn down
(taking `AccessGrant` with it); access-operator closes the connection.
PARs are removed. Provider's own teardown logic handles SA, Workspace,
RBAC.

**6. RFC split.** Two RFCs plus one deferred:
- *RFC 1*: PM-initiated flow, public PM, no tunnel. access SDK and
  access-operator fully specified. Four CRDs (`ProviderAccessRequest`,
  `AccessGrant`, `AccessIdentity`, `Access`) plus the `Provider` extension
  (`spec.remote`). One token (access token), minted by the access SDK on
  `AccessIdentity` creation. Covers both direct and multi-tenant portal
  deployment models — protocol is identical; the portal case is a
  deployment pattern, not a separate RFC.
- *RFC 3*: tunnel mode. Front-proxy SNI router (new kcp work), access SDK
  tunnel mode, and the reconnect-authentication plane (session tokens,
  per-`Access` session-signing key, `jti` replay tracking with grace
  window, `NeedsReonboarding` recovery via new PAR).
- *Deferred*: User-initiated flow. See below.

**7. `AccessIdentity`.** The user-side standing authorization for a PM instance
connection. Created by the Customer directly in a direct deployment, or by
the Provider's portal in a multi-tenant deployment. On creation, the access
SDK mints the access token and surfaces it via an owned Secret.
`spec.accept: Always` auto-accepts incoming `/access` calls; `spec.accept:
Manual` holds them in a pending queue in `AccessIdentity.status.pendingConnections`.
Each pending entry surfaces the `tokenID` so the Provider can cross-check it
against `Provider.status.remote.tokenID` out-of-band before approving. `spec.maxPending`
caps the queue (prevents flooding); a configurable TTL expires unanswered entries.
Revocation granularities (user-side — Customer in a direct deployment, Provider's portal in a multi-tenant deployment):
- Delete an `Access` → cut one PM instance (operator loses that kubeconfig;
  other `Access` objects and the `AccessIdentity` unaffected).
- Rotate `AccessIdentity` HMAC secret (new access token) → blocks future
  handshakes under this identity. Existing `Access` objects continue
  working on their delivered SA tokens.
- Delete `AccessIdentity` → stops accepting new handshakes. Existing
  `Access` objects keep working until their delivered SA token is revoked
  PM-side or their `Access` is deleted.

Response to a leaked access token: rotate the HMAC secret. The attacker cannot
open a new handshake with the stale token; existing tunnels (if the
connection is in RFC 3 tunnel mode) are unaffected by the HMAC rotation —
see Decision 2 for the RFC 3 incident-response path that additionally
forces re-authentication of currently-active tunnels.

**OIDC (future).** Using kcp-issued audience-bound JWTs as the `/access` call
credential — and storing `(issuer, subject)` in `AccessIdentity` instead of
`(tokenID, secret)` — is a viable follow-up, particularly in multi-tenant
portal deployments where PM's kcp is publicly reachable. Does not affect the
current design.

---

## Deferred: User-initiated flow

*Design complete; implementation deferred until cross-org delegation becomes a
priority. Does not require changes to the PM-initiated design.*

"Copy PM's token to User." The Owner mints a single-use `pmj1` join token via
an `AccessToken` object; `access-operator` reconciles it. The User creates an
`AccessRequest` with the token string; the access SDK redeems it at PM's `/join`
endpoint and receives the kubeconfig directly. No access SDK deployment required
on the User side; simpler than PM-initiated.

**Network requirement:** PM's `/join` endpoint must be reachable from the User's
cluster.

**Objects:** `AccessToken` (PM-side; Owner creates; same workspace as the SA it
references), `AccessRequest` (User-side; User creates with the token string;
consumed on redeem).

**Token:** `pmj1.<base64url(/join URL)>.<tokenID>.<secret>` — single-use, minted
by `access-operator`. Configurable lifetime; default 1 h.

```
Owner  ──(1) creates AccessToken in workspace ──►  access-operator
       ◄─(1) token string ────────────────────────

Owner  ──(2) hands token string (out of band) ─────────────────────────────►  User

                                    User creates AccessRequest with token string
                                    access SDK  ──(3) calls /join ──►  access-operator
                                                ◄──(3) kcp URL + CA + SA token ──
                                    access SDK  ──(4) writes kubeconfig Secret, creates Access
                                    access-operator  ──(4) creates AccessGrant
                                    operator    ──(5) mounts Secret, talks to kcp directly
```

The access SDK's lifecycle management (SA token renewal, CA bundle rotation,
offboarding detection) applies to the delivered kubeconfig unchanged.
