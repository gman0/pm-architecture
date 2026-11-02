# Remote access: overview notes

Private working notes. State as of 2026-11-02.

RFC 011 (v5) was refused. v6 was a redesign. This document reframes from first
principles and is the new working baseline. Individual RFCs will be derived from
it.

See [`xxx-notes-011-v6-decisions.md`](./xxx-notes-011-v6-decisions.md) for
v6-specific decisions and e2e verification results that still apply.

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

- **pm-access** — gives you a kubeconfig. User-initiated mode works out of the
  box. Tunnel mode is opt-in: configure a public HTTPS `/access` endpoint and
  the component also accepts reverse connections from pm-gateway, routing kcp
  traffic through them when PM's front-proxy is not directly reachable.
- **pm-provider** — you are a service provider with a portal, serving many PM
  instances, each onboarded by a different customer. Exposes a public
  registration endpoint that PM calls once per connection. Builds on pm-access
  for ongoing credential delivery.

Both live in one repo and share code at the library level.

---

## Cast

| Term | Who or what |
|---|---|
| **User** | Any person controlling a workspace in PM. May also be a Provider. |
| **Provider** | A user who exposes APIExports, ContentConfigurations, etc. |
| **Owner** | The PM-side workspace owner who wants a Provider's service. Often the same person as the User. |
| **Provider Workspace** | The kcp workspace containing the Provider's APIExports, created by PM for the Provider. |
| **PM Instance** | One deployment of Platform Mesh (kcp + pm-operator). May be public or private. |
| **Provider's portal** | The Provider's website/system for managing customers and issuing tokens. PM has no opinion on its internals. |
| **`pm-gateway`** | PM-side. Single binary combining `/join` serving and PM-initiated connection management. Reconciles `JoinToken` objects, serves the `/join` redeem endpoint. Watches `WorkspaceGrant` and `ProviderOnboardingRequest` objects; for PM-initiated: calls pm-provider's registration endpoint, dials pm-access's `/access`, delivers credentials, creates `WorkspaceGrant`. Holds tunnel connections open with leader election. All replicas serve `/join`; leader election governs tunnel management only. Runs in PM's runtime cluster. |
| **`pm-access`** | User-side. Serves the `/access` endpoint (PM-initiated). Watches `JoinRequest` objects and redeems join tokens (User-initiated). Manages `WorkspaceAccess` objects. Self-renews SA tokens via TokenRequest. |
| **`pm-provider`** | User-side. Go SDK for portal integration: Provider's portal calls the SDK to create per-connection authorized access CRDs; pm-provider watches and reconciles them. Exposes a public registration endpoint that pm-gateway calls once per connection. |
| **`JoinToken`** | PM-side CRD. One-time initiation object, User-initiated. Owner creates in their workspace; pm-gateway reconciles to mint the token string. Consumed on success. |
| **`ProviderOnboardingRequest`** | PM-side CRD. One-time initiation object, PM-initiated. Owner creates with the Provider's onboarding token. Consumed on success. |
| **`WorkspaceGrant`** | PM-side CRD. Ongoing binding. Created on successful initiation (either direction). References the pre-existing SA. PM's view of the live connection. |
| **`JoinRequest`** | User-side CRD. One-time initiation object, User-initiated. User creates with the join token string; pm-access watches, redeems, and writes the kubeconfig. Consumed on success. |
| **`WorkspaceAccess`** | User-side CRD. Ongoing binding. Created on successful initiation (either direction). References the kubeconfig Secret. User cluster's view of the live connection. |

---

## Two handshake directions

The mode of a connection is determined by which token flow the Owner initiates.
A user can have User-initiated connections to some PM instances and PM-initiated
connections to others simultaneously, managed by the same pm-access deployment.

### User-initiated — User dials PM

*"Copy PM's token to User."*

**Requirement:** PM's `/join` endpoint must be reachable from the user's
cluster.

```
Owner  ──(1) creates JoinToken in workspace──►  pm-gateway
       ◄─(1) token string ──────────────────

Owner  ──(2) hands token string (out of band) ─────────────────►  User

                                                   User creates JoinRequest with token string
                                                   pm-access  ──(3) calls /join ──►  pm-gateway
                                                              ◄──(3) kcp URL + CA + SA token ──
                                                   pm-access  ──(4) writes kubeconfig Secret
                                                              ──(4) creates WorkspaceAccess
                                                   pm-gateway ──(4) creates WorkspaceGrant
                                                   operator   ──(5) mounts Secret, talks to kcp directly
```

1. Owner creates a `JoinToken` in their workspace; `pm-gateway` mints the token
   string into a status field or delivery Secret.
2. Owner copies the token string and hands it to the User. The only manual
   cross-organisation step.
3. User (or automation) creates a `JoinRequest` in their cluster with the token
   string. `pm-access` sees it, calls PM's `/join`, and receives the kcp URL,
   CA bundle, and SA token.
4. `pm-access` writes the kubeconfig Secret and creates a `WorkspaceAccess`.
   `pm-gateway` creates the corresponding `WorkspaceGrant` on PM side. The
   `JoinToken` and `JoinRequest` are consumed.
5. The User's operator mounts the Secret and talks to kcp directly.

**Kubeconfig:** `server` points directly to kcp's front-proxy. No `proxy-url`.

---

### PM-initiated — PM dials User

*"Copy User's token to PM."*

**Requirements:**
- User's pm-access `/access` endpoint must be publicly reachable from PM.
- Tunnel mode on pm-access is only needed if PM's kcp front-proxy is not
  directly reachable from the user's cluster.

PM-initiated has two distinct phases.

#### Phase 1: Registration (one-time)

```
Provider's portal  ──(1) SDK creates authorized access CRD ──►  pm-access cluster
pm-provider        ──(1) issues onboarding token ───────────►  Owner (via portal)

Owner  ──(2) creates ProviderOnboardingRequest with token ──►  pm-gateway
                                                               ──(3) POST /register, token ──►  pm-provider
                                                               ◄──(3) connection ID, /access URL
```

1. The Provider generates an onboarding token in their portal (`pm-provider`
   SDK). The SDK also creates a per-connection authorized access CRD in the
   user cluster, keyed to the portal account identity.
2. The Owner puts the onboarding token into a `ProviderOnboardingRequest` on PM.
3. `pm-gateway` calls pm-provider's registration endpoint over standard HTTPS,
   presenting the token. pm-provider authenticates it and returns: connection ID
   and the pm-access `/access` URL.

Registration is complete. No tunnel is needed here; the Provider's endpoint is
public and PM dials out.

#### Phase 2: Credential delivery and ongoing access

```
pm-gateway  ──(4) dials /access (derived credential) ──────►  pm-access
            ◄──────────── tunnel established ───────────────
pm-gateway  ──(5) delivers: kcp URL + CA bundle + SA token ►  pm-access
pm-access   ──(6) writes kubeconfig Secret, creates WorkspaceAccess
pm-gateway  ──(6) creates WorkspaceGrant
ProviderOnboardingRequest consumed
operator    ──(7) talks to kcp (directly or via tunnel)
```

4. `pm-gateway` derives a short-lived call credential from the long-lived
   access token (per-connection authorized access CRD) and dials pm-access's
   `/access`. pm-access verifies against its CRD record and accepts.
5. `pm-gateway` mints an SA token via kcp TokenRequest for the SA referenced in
   the `WorkspaceGrant` (created automatically by PM from the
   `ProviderOnboardingRequest`) and delivers the kcp URL, CA bundle, and SA
   token to pm-access.
6. `pm-access` writes the kubeconfig Secret and creates a `WorkspaceAccess`.
   `pm-gateway` creates the corresponding `WorkspaceGrant`. The
   `ProviderOnboardingRequest` is consumed.
7. The operator talks to kcp. If a tunnel was established, the kubeconfig uses
   `proxy-url` pointing to a local pm-access endpoint (which routes through the
   tunnel to PM's front-proxy SNI router). If PM's kcp is publicly reachable,
   the kubeconfig points directly and no `proxy-url` is needed.

**Tunnel:** PM's kcp front-proxy is extended with an L4 SNI router (new work).
A single tunnel connection per remote cluster carries all kcp traffic — both
front-proxy calls and per-shard VW calls — routed by the TLS SNI hostname
without terminating TLS. pm-gateway holds the tunnel open with leader election
and reconnects automatically on drop.

**Kubeconfig:** direct (`server` → kcp front-proxy) when PM's kcp is reachable;
`proxy-url` through pm-access's local tunnel endpoint when it is not.

---

## Network mode coverage

| PM reachable from User | User endpoint reachable from PM | Works | Notes |
|---|---|---|---|
| Yes | Yes | Either | User-initiated is simpler; PM-initiated gives self-service onboarding UX |
| Yes | No | User-initiated only | User can dial PM; PM cannot reach pm-access |
| No | Yes | PM-initiated only | PM dials pm-provider (registration) and pm-access `/access` |
| No | No | Out-of-band VPN | Admins arrange network path; after which the relevant direction applies |

"PM reachable from User" means PM's kcp front-proxy and `pm-gateway` `/join`
endpoint are reachable from the user's cluster. "User endpoint reachable from
PM" means pm-provider's registration endpoint and pm-access's `/access`
endpoint are reachable from PM.

---

## Component details

### `pm-access` (User-side)

One deployment handles any number of PM instance connections, in either
direction.

**User-initiated (always on):**

- Watches `JoinRequest` objects. Each `JoinRequest` carries the token string
  and references a Secret where the kubeconfig should be written.
- On a new `JoinRequest`: calls PM's `/join`, receives kcp URL + CA bundle +
  SA token, writes `kubeconfig` and `token` into the referenced Secret, creates
  a `WorkspaceAccess`, updates `JoinRequest` status.
- Self-renews SA token via TokenRequest periodically (`tokenFile: token`;
  client-go re-reads automatically).

**PM-initiated (requires configuring a public HTTPS `/access` endpoint):**

- Accepts incoming connections from pm-gateway, authenticated by a short-lived
  credential derived from the per-connection authorized access CRD.
- Receives kubeconfig delivery (kcp URL + CA bundle + SA token) from pm-gateway.
- Creates a `WorkspaceAccess` and writes the kubeconfig Secret.
- **Tunnel mode (opt-in):** if PM's kcp is not directly reachable from the
  user's cluster, keeps the connection alive as a reverse L4 channel. Exposes
  a per-connection local HTTPS proxy endpoint; the kubeconfig `proxy-url`
  points here. Routes traffic to PM's front-proxy SNI router port. When PM's
  kcp is publicly reachable, the connection is closed after credential delivery
  and no `proxy-url` is needed.
- Manages many simultaneous connections.

**Shared lifecycle (both directions):**

- Creates and manages `WorkspaceAccess` objects (one per live connection).
- Offboarding detection:
  - 401 after JWT `exp` → expired.
  - Persistent 401 before `exp` (including failed renewal) → revoked; deletes
    the Secret and `WorkspaceAccess`, emits a Pod Event, reports not-ready.
  - 403, 5xx, network errors: not counted as offboarding.
- Re-onboarding: create a new `JoinRequest` (User-initiated) or a new
  `ProviderOnboardingRequest` (PM-initiated).

### `pm-provider` (User-side)

- **Go SDK** for portal integration: the Provider's portal code calls the SDK
  to create per-connection authorized access CRDs, keyed to portal account
  identity. pm-provider watches and reconciles them. The Provider controls
  timing (eager at account creation, lazy at first registration, or any policy).
- Exposes a public HTTPS registration endpoint that pm-gateway calls once per
  connection, presenting the onboarding token.
- On a valid registration: authenticates the token against the authorized access
  CRD, returns the connection ID and pm-access `/access` URL.
- Issues long-lived reusable onboarding tokens, one per portal account. One
  account → many PM instance connections; each identified by clusterName.
  Idempotent: re-registering the same clusterName is a no-op.
- Provider can revoke a single connection without invalidating the onboarding
  token for other PM instances.
- GitOps-safe: applying the same `ProviderOnboardingRequest` N times produces
  the same result.

### `pm-gateway` (PM-side)

- Reconciles `JoinToken` objects; mints the join token string.
- Serves the `/join` redeem endpoint (User-initiated, all replicas).
- On User-initiated redemption: mints SA token via kcp TokenRequest for the SA
  referenced in the binding, returns kcp URL + CA bundle + SA token. Creates
  `WorkspaceGrant`.
- Watches `WorkspaceGrant` objects and `ProviderOnboardingRequest` objects.
- PM-initiated: calls pm-provider's registration endpoint, receives `/access`
  URL and connection ID. Derives a short-lived call credential from the
  long-lived access token. Dials pm-access's `/access`. Delivers kcp URL + CA
  bundle + SA token. Creates `WorkspaceGrant` on success.
- Holds tunnel connections open between PM and remote clusters. Forwards traffic
  between the tunnel and PM's front-proxy SNI router port.
- Multiple replicas with leader election: leader election governs tunnel
  management only; all replicas serve `/join`.
- Maintains CA bundle well-known object per workspace (Decision 4).
- Runs in PM's runtime cluster alongside pm-operator. Minimal privileges;
  separate from pm-operator.

---

## Token formats

| Token | Minted by | Reusable | Encodes |
|---|---|---|---|
| Join token `pmj1.…` | `pm-gateway` | No (single-use) | PM `/join` URL, token ID, secret |
| Onboarding token `pmop1.…` | `pm-provider` | Yes (per portal account) | pm-provider registration URL, token ID, secret |
| Access token `pma1.…` | `pm-access` | Yes (per connection) | pm-access `/access` URL, token ID, secret |

`<prefix>.<base64url(URL)>.<tokenID>.<secret>` — the URL pins the recipient so
the token cannot be redirected; the token ID is safe to log; the secret is the
credential. pm-gateway derives a short-lived call credential from the access
token secret for each dial to `/access`.

Join token lifetime: open (see below).

---

## Decisions

**1. SA token minting.** `pm-gateway` is a single binary (merging the former
pm-join and pm-connect roles). It owns TokenRequest for both User-initiated and
PM-initiated. All replicas serve `/join`; leader election governs tunnel
management only.

**2. Reconnection.** On tunnel drop, pm-gateway re-dials using the existing
`WorkspaceGrant` (which holds the `/access` URL and access token). It always
delivers a fresh SA token on every dial. `WorkspaceGrant` has the SA as an
ownerReference — SA deletion triggers GC deletion of `WorkspaceGrant`, which
pm-gateway observes and stops re-dialing.

**3. Offboarding / revocation.** `WorkspaceGrant` deletion is the authoritative
teardown signal: pm-gateway closes the tunnel; pm-access detects closure and
deletes `WorkspaceAccess` + Secret. From the user side: persistent 401 →
pm-access closes tunnel → pm-gateway reconnects → if SA gone, `WorkspaceGrant`
already GC'd → stop; if SA present → reconnect normally.

**4. CA bundle rotation.** PM publishes the current CA bundle and server URLs in
a well-known object in each workspace where a `WorkspaceGrant` exists.
pm-gateway maintains this object (watches the CA bundle source in PM's cluster,
handles the old/new overlap window). pm-access reads it periodically using its
SA token and rewrites the kubeconfig when it changes.

**5. `ProviderOnboardingRequest` shape.** `Provider` object is kept.
`spec.remote` removed. `serviceAccountRef` is not on the POR — the Owner
creates the POR with only `spec.providerRef` and `spec.onboardingToken`. PM
automatically creates Provider → Provider Workspace → dedicated SA →
`WorkspaceGrant` (with SA ownerReference). pm-gateway finds the pending
`WorkspaceGrant` and completes the PM-initiated connection.

**6. `JoinToken` location.** Always the same workspace as the SA it references.
Provider case: Provider Workspace. Plain user case: their own workspace.

**7. RFC split.** Three RFCs:
- *RFC 1*: pm-access + pm-gateway, User-initiated only. Core primitive, no tunnel, public PM.
- *RFC 2*: pm-provider, `ProviderOnboardingRequest`, PM-initiated without tunnel. Provider/portal use case, public PM.
- *RFC 3*: tunnel mode. Front-proxy SNI router (new kcp work), pm-access tunnel mode. Extends both directions to private PM instances.

**8. Join token lifetime.** Configurable; default 1 h.
