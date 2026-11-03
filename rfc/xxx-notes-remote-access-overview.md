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

On top of this primitive, two components address different points on the
complexity curve:

- **access SDK** — manages kubeconfig Secrets in the user-side cluster: SA
  token renewal, CA bundle rotation, offboarding detection. Serves the public
  HTTPS `/access` endpoint that access-operator dials to deliver credentials
  (PM-initiated). Mints `pma1` tokens on `AccessIdentity` creation (owned
  Secret, HMAC secret stored for verification). Tunnel mode is opt-in (RFC 3):
  keeps the `/access` connection alive as a reverse L4 channel when PM's
  front-proxy is not directly reachable.
- **provider SDK** — Go library a Provider's portal imports when it operates a
  multi-tenant service. Automates `AccessIdentity` creation on the Provider's
  own cluster (same cluster the access SDK runs on) and surfaces `pma1` tokens
  to the portal UI, so the Customer logs into the portal, retrieves the token,
  and pastes it into a `ProviderAccessRequest` on their PM.

Both live in `go.platform-mesh.io/sdk` and share code at the library level.

**Trust-boundary note.** In RFC 1 (direct), User = Customer = Owner; the
user-side cluster is the Customer's own. In RFC 2 (portal), the user-side
cluster is the **Provider's** cluster — it hosts the portal, the access SDK,
and all `AccessIdentity` / `Access` / Secret objects, one group per Customer
PM instance. The Customer's only footprint is their PM workspace; they don't
run access SDK. "User-side" throughout this document means "wherever the
access SDK runs," which is determined by this split.

---

## Cast

| Term | Who or what |
|---|---|
| **User** | Any person controlling a workspace in PM. May also be a Provider. |
| **Provider** | A user who exposes APIExports, ContentConfigurations, etc. |
| **Owner** | The PM-side workspace owner who wants a Provider's service. Often the same person as the User. |
| **Provider Workspace** | The kcp workspace containing the Provider's APIExports, created by PM for the Provider. |
| **`Provider`** | PM-side CRD (`providers.platform-mesh.io/v1alpha1`). Cluster-scoped. Pre-existing resource for kcp-side bootstrap (Workspace, SA, RBAC); extended with `spec.remote` (nested struct, presence-based opt-in) to switch from local mode (kubeconfig Secret stored in PM) to remote mode (kubeconfig delivered to a user-side cluster via `AccessGrant`). Mode is immutable after creation. Local kubeconfig Secret is skipped in remote mode. Persistent lifecycle handle: owns `AccessGrant` and PARs via ownerReferences; `status.remote` surfaces the live connection state (`connectionState`, `tokenID`, `accessGrantRef`, `lastConnected`). Deleting the `Provider` is the authoritative teardown signal. |
| **PM Instance** | One deployment of Platform Mesh (kcp + pm-operator). May be public or private. |
| **Provider's portal** | The Provider's website/system for managing customers. Imports the provider SDK library to create `AccessIdentity` objects on the Provider's cluster and surface the minted `pma1` to the Customer. PM has no opinion on its internals. |
| **`access-operator`** | PM-side (`platform-mesh/operators/access-operator`, binary: `access-operator`). Watches `AccessGrant` objects in `Pending` state (created and pre-populated by the provider reconciler). Runs an initial-handshake reconciliation loop against access SDK's `/access` using the `pma1` from `AccessGrant.spec`: retries on `202` until `200` (credentials + session token) or `403`. On `200`: mints SA token via kcp TokenRequest, stores `sessionToken` on `AccessGrant.status`, marks `AccessGrant` `Active`. Subsequent `/access` calls (tunnel reconnects) present `sessionToken` only; rejection sets `NeedsReonboarding` and dialing stops. Holds tunnel connections open with leader election (RFC 3). Runs in PM's runtime cluster. |
| **`access` SDK** | User-side (`go.platform-mesh.io/sdk/access`, binary: `platform-mesh-access`). Reconciles `AccessIdentity` objects: on creation, generates the HMAC secret, the per-identity session-signing key, and emits a `pma1` token via an owned Secret. Serves the `/access` endpoint: verifies HMAC (initial handshake) or session token (reconnect), enforces `accept` mode, manages the pending queue. Issues a `sessionToken` (JWT, signed by the per-`AccessIdentity` signing key, short TTL) on every successful `/access` 200 response. Manages `Access` objects (one per live connection). Self-renews SA tokens via TokenRequest. Handles CA bundle rotation and offboarding detection. |
| **`provider` SDK** | Go library (`go.platform-mesh.io/sdk/provider`). Portal integration helper for RFC 2: the Provider's portal imports it to create `AccessIdentity` objects on the Provider's own cluster, read the minted `pma1` from the owned Secret, and manipulate `AccessIdentity.spec` (`accept` mode, pending-queue approvals, blocklists) to enforce portal policy. |
| **`ProviderAccessRequest`** | PM-side CRD. One-shot auth-material delivery object. Owner creates it with `spec.providerRef.name` and a `pma1` access token (from the user-side cluster in RFC 1, from the Provider's portal UI in RFC 2). Reconciled by the provider reconciler (inside `pm-operator`; later `provider-operator`): waits in `Pending` until the referenced `Provider` exists with `spec.remote` set, then propagates `pma1` into the Provider's `AccessGrant`; terminal states `Completed` / `Failed`. One active PAR per `Provider`: new PARs arriving while an older non-terminal PAR exists go `Failed` with reason `AnotherPARActive`. OwnerReference → `Provider` (cascade on Provider deletion). |
| **`AccessGrant`** | PM-side CRD. Created by provider reconciler in `Pending` state with SA ref, `/access` URL, and `pma1` credentials. OwnerReference → `Provider` (cascade on Provider deletion). access-operator drives it to `Active` by completing the initial `/access` handshake. `spec.pma1` is used only for initial handshake (first bind, and recovery after a subsequent PAR delivers fresh `pma1`). `.status.sessionToken` + `.status.sessionTokenExpiresAt` is the sole reconnection credential, refreshed on each successful dial. PM's view of the live connection. |
| **`AccessIdentity`** | User-side CRD. Standing authorization for a PM instance connection. On creation, the access SDK generates `(tokenID, HMAC secret)` for verifying initial `/access` handshakes, a per-identity session-signing key for signing session tokens, and emits the full `pma1` into an owned Secret. The two credential planes are disjoint: `pma1` authorizes initial handshake only; session tokens authorize all post-handshake dials. `spec.accept`: `Always` (auto) or `Manual` (pending queue). `spec.maxPending` caps simultaneous pending entries. One `AccessIdentity` → many `Access` objects over time. Rotate the HMAC secret (new `pma1`) to block future initial handshakes under this identity — zero effect on existing `Access` objects in either mode. Rotate the session-signing key to invalidate every outstanding session token (forces all current tunnels into re-onboarding; no effect in direct mode). Delete to do both at once. Per-PM revocation: delete the specific `Access`. |
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
RFC 1 (direct):
  Customer  ──(1) creates AccessIdentity in own cluster ──►  access SDK
                                                             [mints pma1 → owned Secret]
  Owner     ──(2) creates Provider (spec.remote: {}) on PM
  Owner     ──(2) reads pma1, creates ProviderAccessRequest (providerRef: <provider>, pma1) on PM
                                                             ──►  provider reconciler
                                                                  [bootstraps Provider: Workspace, SA, RBAC]
                                                                  [extracts /access URL from pma1]
                                                                  [creates AccessGrant (Pending), ownerRef → Provider]
                                                                  [marks PAR Completed]

RFC 2 (portal):
  Customer  ──(1) logs into Provider's portal
  Portal    ──(1) provider SDK creates AccessIdentity on Provider's cluster ──►  access SDK
                                                                                 [mints pma1 → owned Secret]
  Portal    ──(1) reads pma1 from owned Secret, shows in UI
  Owner     ──(2) creates Provider (spec.remote: {}) on PM
  Owner     ──(2) copies pma1 from portal, creates ProviderAccessRequest on PM
                                                                              ──►  provider reconciler
                                                                                   [same as RFC 1 from here]
```

1. An `AccessIdentity` is created in the user-side cluster — by the Customer
   directly (RFC 1) or by the Provider's portal via the provider SDK (RFC 2).
   The access SDK generates the HMAC secret and emits the `pma1` token via an
   owned Secret. In RFC 2 the portal reads the Secret and surfaces the token
   in the Provider's UI.
2. The Owner creates a `Provider` with `spec.remote: {}` and a
   `ProviderAccessRequest` referencing it by name (`spec.providerRef.name`),
   carrying the `pma1`. Order-independent: whichever is created first waits
   for the other.
3. The provider reconciler bootstraps the `Provider` (Workspace, SA, RBAC —
   same as local mode but skips the local kubeconfig Secret), extracts the
   `/access` URL from the `pma1`, and creates an `AccessGrant` in `Pending`
   state pre-populated with the SA ref, `/access` URL, and `pma1` credentials.
   The `AccessGrant` has `ownerReference` → `Provider`. The PAR is marked
   `Completed`.

The two modes differ only in who creates the `AccessIdentity` and where the
Customer gets the `pma1` from. The PM-side flow is identical.

#### Phase 2: Credential delivery and ongoing access

```
access-operator  ──(4) picks up AccessGrant (Pending)
access-operator  ──(4) POST /access (derived credential from pma1) ──►  access SDK
                 ◄──── 202, tokenID (accept: Manual) ────────────────
                       [access-operator retries until approved or TTL]
                 ◄──── 200 + kcp URL + CA + SA token + sessionToken ─
access-operator  ──(5) mints SA token, stores sessionToken on
                       AccessGrant.status, marks AccessGrant Active
access SDK              ──(5) writes kubeconfig Secret, creates Access
provider reconciler ──(5) updates Provider.status.remote.connectionState → Connected
operator                ──(6) talks to kcp directly

On tunnel drop (RFC 3):
access-operator  ──(R) POST /access (sessionToken from AccessGrant.status) ──►  access SDK
                 ◄──── 200 + fresh sessionToken ─────────────────────────────
                       [tunnel re-established; AccessGrant.status.sessionToken rolled]
```

4. access-operator picks up the `AccessGrant` in `Pending` state. It derives a
   short-lived call credential from the `pma1` credentials in the `AccessGrant`
   and calls `POST /access` on the access SDK. The access SDK verifies the
   credential against the `AccessIdentity`.
   - `spec.accept: Always` — responds immediately (`200`) with kcp URL + CA
     bundle + SA token + `sessionToken` (JWT signed by the per-`AccessIdentity`
     session-signing key, short TTL).
   - `spec.accept: Manual` — creates a pending entry in
     `AccessIdentity.status.pendingConnections` with the `tokenID` and responds
     `202`. access-operator retries periodically. The Provider approves by
     patching the `AccessIdentity`; the next retry receives `200` + credentials.
     Denial or TTL expiry returns `403`.
5. access-operator mints an SA token via kcp TokenRequest for the SA referenced
   in the `AccessGrant`, stores the `sessionToken` + expiry on
   `AccessGrant.status`, and marks it `Active`. The access SDK writes the
   kubeconfig Secret and creates `Access`. The provider reconciler reflects the
   `Active` state onto `Provider.status.remote.connectionState`.
6. The operator mounts the Secret and talks to kcp directly.

**Reconnection credential.** `pma1` is used only on initial handshake.
Subsequent `/access` dials (tunnel reconnects, scheduled refreshes) present
the stored `sessionToken` and nothing else; access SDK verifies the JWT
signature against the per-`AccessIdentity` signing key and issues a fresh
`sessionToken` on each successful reply. The two credential planes are
disjoint: HMAC-secret state and session-signing-key state do not depend on
each other at verification time. If the session token is rejected (expired,
signing key rotated, access SDK state loss), the `AccessGrant` surfaces
`NeedsReonboarding` (reflected onto `Provider.status.remote.connectionState`)
and access-operator stops dialing. Recovery is a **new PAR** referencing the
same `Provider` with a current `pma1`; the provider reconciler propagates it
into `AccessGrant.spec.pma1`, access-operator sees the change and
re-handshakes from scratch.

**Tunnel (RFC 3):** after credential delivery, if PM's kcp is not directly
reachable, the same `/access` connection is upgraded to a persistent reverse L4
tunnel. The kubeconfig uses `proxy-url` pointing to a local access SDK endpoint
that routes traffic to PM's front-proxy SNI router.

**Tunnel:** PM's kcp front-proxy is extended with an L4 SNI router (new work).
A single tunnel connection per remote cluster carries all kcp traffic — both
front-proxy calls and per-shard VW calls — routed by the TLS SNI hostname
without terminating TLS. access-operator holds the tunnel open with
leader election and reconnects automatically on drop.

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
- `spec.accept` governs **initial handshake only**. Reconnects with valid
  session tokens are always accepted; they do not enter the pending queue.
- `spec.accept: Always` (default): incoming initial-handshake calls with a
  valid HMAC credential are accepted immediately.
- `spec.accept: Manual`: incoming initial-handshake calls create a pending
  entry in `AccessIdentity.status.pendingConnections` (with `tokenID`) and
  return `202`. `spec.maxPending` caps the queue; entries expire after the
  pending TTL. Provider approves or denies by patching the `AccessIdentity`.
- Deleting an `AccessIdentity` invalidates both credential planes for this
  identity simultaneously: HMAC secret is gone (future initial handshakes
  fail `403`), session-signing key is gone (session-token verification on
  reconnects fails `403`). Existing `Access` objects and kubeconfig Secrets
  continue to function on their delivered SA tokens; the user-side operator
  (Customer in RFC 1, Provider's portal in RFC 2) deletes individual `Access`
  objects to cut specific PM connections.

**`/access` endpoint (PM-initiated):**

- Serves `POST /access`. access-operator presents either
  `Bearer <tokenID>.<derived-credential>` (initial handshake) or
  `Bearer <sessionToken>` (reconnect). The two credential types are
  verified against disjoint keys and never substitute for each other.
- Initial handshake: looks up `AccessIdentity` by `tokenID`, verifies HMAC
  against stored HMAC secret.
- Reconnect: verifies the `sessionToken` JWT signature against the
  per-`AccessIdentity` signing key; rejects on signature failure, expiry, or
  signing-key rotation. Does not fall back to HMAC verification.
- Returns `200` + credentials + fresh `sessionToken`, `202` (pending), or
  `403` (denied/unknown/TTL/expired-session).
- Creates `Access` and writes kubeconfig Secret on `200` (initial handshake
  only; reconnects reuse the existing `Access`).
- **Tunnel mode (RFC 3, opt-in):** after credential delivery, if PM's kcp is
  not directly reachable, the connection is upgraded to a persistent reverse L4
  tunnel. Exposes a per-connection local HTTPS proxy endpoint; the kubeconfig
  `proxy-url` points here.

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

### `provider` SDK (Go library, `go.platform-mesh.io/sdk/provider`)

Imported by the Provider's portal. Operates against the Provider's own cluster
— the same cluster the access SDK runs on, co-located with the portal.

- **`AccessIdentity` lifecycle helpers.** Create / update / delete
  `AccessIdentity` objects on the Provider's cluster, keyed to portal account
  identity + Customer PM instance. Portal controls timing (eager at account
  creation, lazy at first bind, or any policy).
- **`pma1` retrieval.** Read the minted `pma1` from the owned Secret the
  access SDK writes alongside each `AccessIdentity`. Portal surfaces it in the
  UI for the Customer to copy.
- **Policy enforcement via `spec`.** Helpers to flip `accept: Manual`,
  approve/deny `pendingConnections`, maintain blocklists, revoke per PM by
  deleting a specific `Access`, revoke per account by deleting the
  `AccessIdentity`.
- **GitOps-safe.** All operations are declarative kube operations on the
  Provider's cluster.

### `access-operator` (PM-side, `platform-mesh/operators/access-operator`, binary: `access-operator`)

- Watches `AccessGrant` objects in `Pending` state (created and pre-populated
  by the provider reconciler with SA ref, `/access` URL, and `pma1` credentials).
- Initial handshake: derives a short-lived call credential from the `pma1` in
  `AccessGrant.spec`. Runs a reconciliation loop against `POST /access`:
  retries on `202` (`accept: Manual`, pending approval) until `200`
  (credentials + session token) or `403` (denied/TTL expired). On `200`:
  mints SA token via kcp TokenRequest, stores `sessionToken` +
  `sessionTokenExpiresAt` on `AccessGrant.status`, marks `AccessGrant` `Active`.
  Triggered when `AccessGrant` is `Pending` (first bind) or when
  `AccessGrant.spec.pma1` changes while in `NeedsReonboarding` (recovery
  after the Owner creates a fresh PAR against the same `Provider`).
- Reconnect: presents `AccessGrant.status.sessionToken` and nothing else;
  rolls the stored value on each successful response. If the session token
  is rejected, sets `AccessGrant` status condition `NeedsReonboarding` and
  stops dialing. Dialing resumes only after a new PAR delivers a fresh
  `pma1` into `AccessGrant.spec.pma1`, at which point access-operator runs
  a fresh initial handshake.
- Reconnects on tunnel drop (RFC 3) using the session token from the same
  `AccessGrant`. Stops when `AccessGrant` is deleted (cascaded from
  `Provider` deletion via ownerReference).
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
| Session token (JWT) | `access` SDK (on `/access` 200 response) | Short-lived (hours), rolled on each reconnect | `AccessIdentity` ref, `Access` ref, expiry; signed by per-`AccessIdentity` signing key |

`pma1.<base64url(URL)>.<tokenID>.<secret>` — the URL pins the recipient so the
token cannot be redirected; the token ID is safe to log; the secret is the
HMAC credential. The access SDK generates `(tokenID, secret)` when it
reconciles a new `AccessIdentity` and writes the full `pma1` string into an
owned Secret alongside the identity. access-operator derives a short-lived
call credential from the HMAC secret for initial handshake only (first bind,
and recovery after `NeedsReonboarding` via a new PAR against the same
`Provider`).

The session token is a standard JWT. access SDK issues one on every
successful `/access` 200 response; access-operator stores it on
`AccessGrant.status.sessionToken` and presents it on subsequent dials. Signed
by a per-`AccessIdentity` signing key held in access SDK state — rotating
that key invalidates every outstanding session token for the identity. In
tunnel mode this drops every current tunnel into `NeedsReonboarding`;
recovery is a fresh initial handshake triggered by a new PAR against the
same `Provider`, same as any other `NeedsReonboarding` cause.

---

## Decisions

**1. SA token minting.** `access-operator` is a single binary. Owns
TokenRequest for PM-initiated connections. Leader election governs tunnel
management (RFC 3) only; credential delivery is stateless.

**2. Reconnection.** The two credential planes are disjoint. `pma1`
authorizes initial handshake only; session tokens authorize all subsequent
dials. On tunnel drop, access-operator re-dials using the `sessionToken`
stored on `AccessGrant.status`; access SDK verifies the JWT and issues a
fresh `sessionToken` in the 200 response, which access-operator rolls back
onto `AccessGrant.status`. No fallback: session-token rejection sets
`AccessGrant` condition `NeedsReonboarding` and stops dialing. Recovery
requires a fresh initial handshake, triggered by `AccessGrant.spec.pma1`
changing — which happens when the Owner creates a new PAR referencing the
same `Provider`. access-operator always delivers a fresh SA token on every
successful dial. `AccessGrant` has `ownerReference` → `Provider`; deleting
the `Provider` cascades to `AccessGrant` deletion, which access-operator
observes and stops re-dialing (the Provider's own teardown logic handles SA
and Workspace cleanup). On the user side, `AccessIdentity` deletion
invalidates both planes for this identity; existing `Access` objects and
kubeconfig Secrets remain. The user-side operator (Customer in RFC 1,
Provider's portal in RFC 2) deletes individual `Access` objects to cut
specific PM connections.

**3. Offboarding / revocation.** Deleting the `Provider` is the
authoritative teardown signal; it cascades to `AccessGrant` deletion
(ownerReference), which access-operator observes to close the tunnel.
access SDK detects closure and deletes `Access` + Secret. From the
user side: persistent 401 → access SDK closes tunnel →
access-operator reconnects → if the SA has been revoked on kcp,
the handshake fails and access-operator stops; otherwise reconnect
proceeds normally.

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
recreate. `Provider.status.remote` carries live connection state
(`connectionState`, `tokenID`, `accessGrantRef`, `lastConnected`), fed by
the provider reconciler watching `AccessGrant.status`.

`ProviderAccessRequest` is a one-shot auth-material delivery object with
`spec.providerRef.name` and `spec.pma1`. Reconciled by the provider
reconciler (inside `pm-operator`; later `provider-operator`), not by
`access-operator`. Rules:

- **Order-independent.** If the referenced `Provider` doesn't exist yet,
  PAR sits in `Pending` until it does. If the `Provider` exists but lacks
  `spec.remote`, PAR goes `Failed` with `ProviderNotRemoteMode`.
- **One active PAR per `Provider`.** New PARs arriving while an older
  non-terminal PAR exists for the same `Provider` go `Failed` immediately
  with `AnotherPARActive: <older-par-name>`. Oldest wins.
- **Terminal states.** `Completed` / `Failed`.
- **OwnerReference → `Provider`** set once the referenced `Provider`
  exists. Deleting the `Provider` cascades to all its PARs.
- **Enforcement is reconciler-side**, via `status`.

First bind: Owner creates `Provider` + PAR (either order). Reconciler
bootstraps and creates `AccessGrant`.

Recovery / rotation: `AccessGrant` enters `NeedsReonboarding`; Owner
creates a **new PAR** against the same `Provider` with a current `pma1`;
reconciler propagates `pma1` into `AccessGrant.spec.pma1`; access-operator
runs a fresh initial handshake.

Teardown: delete the `Provider`. Cascades: AccessGrant → access-operator
closes tunnel; PARs → GC'd; Provider's own teardown logic handles SA,
Workspace, RBAC.

**6. RFC split.** Three RFCs plus one deferred:
- *RFC 1*: PM-initiated flow, public PM, no tunnel. access SDK and
  access-operator fully specified. Four CRDs: `ProviderAccessRequest`,
  `AccessGrant`, `AccessIdentity`, `Access`. One token (`pma1`), minted by the
  access SDK on `AccessIdentity` creation. No portal integration — Customer
  creates `AccessIdentity` on their own cluster, reads the `pma1` from the
  owned Secret, pastes into the PAR.
- *RFC 2*: provider SDK — Go library the Provider's portal imports to
  automate `AccessIdentity` creation on the Provider's own cluster (same
  cluster the access SDK runs on) and surface the minted `pma1` to the portal
  UI. Enables multi-tenant Provider portals. Protocol-identical to RFC 1 from
  PM's perspective.
- *RFC 3*: tunnel mode. Front-proxy SNI router (new kcp work), access SDK
  tunnel mode.
- *Deferred*: User-initiated flow. See below.

**7. `AccessIdentity`.** The user-side standing authorization for a PM instance
connection. Created by the Customer directly (RFC 1) or by the Provider's
portal via the provider SDK library (RFC 2). On creation, the access SDK mints
the `pma1` and surfaces it via an owned Secret.
`spec.accept: Always` auto-accepts incoming `/access` calls; `spec.accept:
Manual` holds them in a pending queue in `AccessIdentity.status.pendingConnections`.
Each pending entry surfaces the `tokenID` so the Provider can cross-check it
against `Provider.status.remote.tokenID` out-of-band before approving. `spec.maxPending`
caps the queue (prevents flooding); a configurable TTL expires unanswered entries.
Revocation granularities (user-side — Customer in RFC 1, Provider's portal in RFC 2 — all independent):
- Delete an `Access` → cut one PM instance (operator loses that kubeconfig;
  other `Access` objects and the `AccessIdentity` unaffected).
- Rotate `AccessIdentity` HMAC secret (new `pma1`) → blocks future initial
  handshakes under this identity. **Zero effect on existing `Access` objects
  in either mode** — the two credential planes are disjoint, so running
  tunnels (which authenticate reconnects via session tokens) do not depend
  on the current HMAC secret, and direct-mode connections never consult
  `/access` after initial handshake either way.
- Rotate `AccessIdentity` session-signing key → invalidates every outstanding
  session token for this identity. In tunnel mode: all current tunnels enter
  `NeedsReonboarding` on next drop (or immediately, if access SDK proactively
  tears tunnels down on key rotation). In direct mode: no effect, because
  direct connections never consult session tokens.
- Delete `AccessIdentity` → invalidates both planes simultaneously. Running
  tunnels enter `NeedsReonboarding` on next drop; direct-mode connections
  continue until their delivered SA token is revoked PM-side or their
  `Access` is deleted.

Response to a leaked `pma1`: rotate the HMAC secret (locks the attacker out
of opening a new tunnel). The attacker cannot leverage the leaked `pma1`
against existing tunnels, because `pma1` is not valid currency on the
reconnect path. If there is reason to believe the attacker opened a tunnel
*before* the leak was detected, additionally rotate the session-signing key
to force all current tunnels into re-onboarding; the Owner creates a new
PAR against the same `Provider` with the new `pma1` and the legitimate
tunnel is restored.

**Rotation by mode.** HMAC-secret rotation is symmetric: it blocks future
initial handshakes and nothing more, in both modes. Session-signing-key
rotation is tunnel-mode-only in effect (direct mode has no ongoing
authenticated channel to invalidate); forcing re-authentication of a
direct-mode connection requires PM-side action — delete the SA on kcp, or
delete the `AccessGrant`.

**OIDC (future).** Using kcp-issued audience-bound JWTs as the `/access` call
credential — and storing `(issuer, subject)` in `AccessIdentity` instead of
`(tokenID, secret)` — is a viable follow-up, particularly for the provider SDK
(RFC 2) case where PM's kcp is publicly reachable. Does not affect the current
design.

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
