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

- **remote-access SDK** — gives you a kubeconfig. User-initiated mode works out
  of the box. Tunnel mode is opt-in: configure a public HTTPS `/access` endpoint
  and the component also accepts reverse connections from remote-access-operator,
  routing kcp traffic through them when PM's front-proxy is not directly
  reachable.
- **provider SDK** — you are a service provider with a portal, serving many PM
  instances, each onboarded by a different customer. Exposes a public
  registration endpoint that PM calls once per connection. Builds on the
  remote-access SDK for ongoing credential delivery.

Both live in `go.platform-mesh.io/sdk` and share code at the library level.

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
| **`remote-access-operator`** | PM-side (`platform-mesh/operators/remote-access-operator`, binary: `remote-access-operator`). Reconciles `RemoteAccessToken` objects, serves the `/join` redeem endpoint. Watches `RemoteAccessGrant` and `ProviderAccessRequest` objects; for PM-initiated: calls provider SDK's registration endpoint, dials remote-access SDK's `/access`, delivers credentials, creates `RemoteAccessGrant`. Holds tunnel connections open with leader election. All replicas serve `/join`; leader election governs tunnel management only. Runs in PM's runtime cluster. |
| **`remote-access` SDK** | User-side (`go.platform-mesh.io/sdk/remote-access`, binary: `platform-mesh-access`). Serves the `/access` endpoint (PM-initiated). Watches `RemoteAccessRequest` objects and redeems join tokens (User-initiated). Manages `RemoteAccess` objects. Self-renews SA tokens via TokenRequest. |
| **`provider` SDK** | User-side (`go.platform-mesh.io/sdk/provider`, binary: `platform-mesh-provider`). Go SDK for portal integration: Provider's portal calls the SDK to create per-connection authorized access CRDs; the provider SDK watches and reconciles them. Exposes a public registration endpoint that remote-access-operator calls once per connection. |
| **`RemoteAccessToken`** | PM-side CRD. One-time initiation object, User-initiated. Owner creates in their workspace; remote-access-operator reconciles to mint the token string. Consumed on success. |
| **`ProviderAccessRequest`** | PM-side CRD. One-time initiation object, PM-initiated. Owner creates with the Provider's onboarding token. Consumed on success. |
| **`RemoteAccessGrant`** | PM-side CRD. Ongoing binding. Created on successful initiation (either direction). References the pre-existing SA. PM's view of the live connection. |
| **`RemoteAccessRequest`** | User-side CRD. One-time initiation object, User-initiated. User creates with the join token string; remote-access SDK watches, redeems, and writes the kubeconfig. Consumed on success. |
| **`RemoteAccess`** | User-side CRD. Ongoing binding. Created on successful initiation (either direction). References the kubeconfig Secret. User cluster's view of the live connection. |

---

## Two handshake directions

The mode of a connection is determined by which token flow the Owner initiates.
A user can have User-initiated connections to some PM instances and PM-initiated
connections to others simultaneously, managed by the same remote-access SDK deployment.

### User-initiated — User dials PM

*"Copy PM's token to User."*

**Requirement:** PM's `/join` endpoint must be reachable from the user's
cluster.

```
Owner  ──(1) creates RemoteAccessToken in workspace──►  remote-access-operator
       ◄─(1) token string ──────────────────────────

Owner  ──(2) hands token string (out of band) ──────────────────────────────►  User

                                            User creates RemoteAccessRequest with token string
                                            remote-access  ──(3) calls /join ──►  remote-access-operator
                                                           ◄──(3) kcp URL + CA + SA token ──
                                            remote-access  ──(4) writes kubeconfig Secret
                                                           ──(4) creates RemoteAccess
                                            remote-access-operator ──(4) creates RemoteAccessGrant
                                            operator       ──(5) mounts Secret, talks to kcp directly
```

1. Owner creates a `RemoteAccessToken` in their workspace; `remote-access-operator`
   mints the token string into a status field or delivery Secret.
2. Owner copies the token string and hands it to the User. The only manual
   cross-organisation step.
3. User (or automation) creates a `RemoteAccessRequest` in their cluster with
   the token string. The remote-access SDK sees it, calls PM's `/join`, and
   receives the kcp URL, CA bundle, and SA token.
4. The remote-access SDK writes the kubeconfig Secret and creates a `RemoteAccess`.
   `remote-access-operator` creates the corresponding `RemoteAccessGrant` on PM
   side. The `RemoteAccessToken` and `RemoteAccessRequest` are consumed.
5. The User's operator mounts the Secret and talks to kcp directly.

**Kubeconfig:** `server` points directly to kcp's front-proxy. No `proxy-url`.

---

### PM-initiated — PM dials User

*"Copy User's token to PM."*

**Requirements:**
- User's remote-access SDK `/access` endpoint must be publicly reachable from PM.
- Tunnel mode on the remote-access SDK is only needed if PM's kcp front-proxy is
  not directly reachable from the user's cluster.

PM-initiated has two distinct phases.

#### Phase 1: Registration (one-time)

```
Provider's portal      ──(1) SDK creates authorized access CRD ──►  remote-access cluster
provider SDK           ──(1) issues onboarding token ───────────►  Owner (via portal)

Owner  ──(2) creates ProviderAccessRequest with token ──►  remote-access-operator
                                                           ──(3) POST /register, token ──►  provider SDK
                                                           ◄──(3) connection ID, /access URL
```

1. The Provider generates an onboarding token in their portal (provider SDK).
   The SDK also creates a per-connection authorized access CRD in the user
   cluster, keyed to the portal account identity.
2. The Owner puts the onboarding token into a `ProviderAccessRequest` on PM.
3. `remote-access-operator` calls provider SDK's registration endpoint over
   standard HTTPS, presenting the token. The provider SDK authenticates it and
   returns: connection ID and the remote-access SDK `/access` URL.

Registration is complete. No tunnel is needed here; the Provider's endpoint is
public and PM dials out.

#### Phase 2: Credential delivery and ongoing access

```
remote-access-operator  ──(4) dials /access (derived credential) ──────►  remote-access
                        ◄──────────── tunnel established ───────────────
remote-access-operator  ──(5) delivers: kcp URL + CA bundle + SA token ►  remote-access
remote-access           ──(6) writes kubeconfig Secret, creates RemoteAccess
remote-access-operator  ──(6) creates RemoteAccessGrant
ProviderAccessRequest consumed
operator                ──(7) talks to kcp (directly or via tunnel)
```

4. `remote-access-operator` derives a short-lived call credential from the
   long-lived access token (per-connection authorized access CRD) and dials
   remote-access SDK's `/access`. The remote-access SDK verifies against its
   CRD record and accepts.
5. `remote-access-operator` mints an SA token via kcp TokenRequest for the SA
   referenced in the `RemoteAccessGrant` (created automatically by PM from the
   `ProviderAccessRequest`) and delivers the kcp URL, CA bundle, and SA token
   to the remote-access SDK.
6. The remote-access SDK writes the kubeconfig Secret and creates a
   `RemoteAccess`. `remote-access-operator` creates the corresponding
   `RemoteAccessGrant`. The `ProviderAccessRequest` is consumed.
7. The operator talks to kcp. If a tunnel was established, the kubeconfig uses
   `proxy-url` pointing to a local remote-access SDK endpoint (which routes
   through the tunnel to PM's front-proxy SNI router). If PM's kcp is publicly
   reachable, the kubeconfig points directly and no `proxy-url` is needed.

**Tunnel:** PM's kcp front-proxy is extended with an L4 SNI router (new work).
A single tunnel connection per remote cluster carries all kcp traffic — both
front-proxy calls and per-shard VW calls — routed by the TLS SNI hostname
without terminating TLS. remote-access-operator holds the tunnel open with
leader election and reconnects automatically on drop.

**Kubeconfig:** direct (`server` → kcp front-proxy) when PM's kcp is reachable;
`proxy-url` through the remote-access SDK's local tunnel endpoint when it is not.

---

## Network mode coverage

| PM reachable from User | User endpoint reachable from PM | Works | Notes |
|---|---|---|---|
| Yes | Yes | Either | User-initiated is simpler; PM-initiated gives self-service onboarding UX |
| Yes | No | User-initiated only | User can dial PM; PM cannot reach remote-access SDK |
| No | Yes | PM-initiated only | PM dials provider SDK (registration) and remote-access SDK `/access` |
| No | No | Out-of-band VPN | Admins arrange network path; after which the relevant direction applies |

"PM reachable from User" means PM's kcp front-proxy and `remote-access-operator`
`/join` endpoint are reachable from the user's cluster. "User endpoint reachable
from PM" means provider SDK's registration endpoint and remote-access SDK's
`/access` endpoint are reachable from PM.

---

## Component details

### `remote-access` SDK (User-side, `go.platform-mesh.io/sdk/remote-access`, binary: `platform-mesh-access`)

One deployment handles any number of PM instance connections, in either
direction.

**User-initiated (always on):**

- Watches `RemoteAccessRequest` objects. Each `RemoteAccessRequest` carries the
  token string and references a Secret where the kubeconfig should be written.
- On a new `RemoteAccessRequest`: calls PM's `/join`, receives kcp URL + CA
  bundle + SA token, writes `kubeconfig` and `token` into the referenced Secret,
  creates a `RemoteAccess`, updates `RemoteAccessRequest` status.
- Self-renews SA token via TokenRequest periodically (`tokenFile: token`;
  client-go re-reads automatically).

**PM-initiated (requires configuring a public HTTPS `/access` endpoint):**

- Accepts incoming connections from remote-access-operator, authenticated by a
  short-lived credential derived from the per-connection authorized access CRD.
- Receives kubeconfig delivery (kcp URL + CA bundle + SA token) from
  remote-access-operator.
- Creates a `RemoteAccess` and writes the kubeconfig Secret.
- **Tunnel mode (opt-in):** if PM's kcp is not directly reachable from the
  user's cluster, keeps the connection alive as a reverse L4 channel. Exposes
  a per-connection local HTTPS proxy endpoint; the kubeconfig `proxy-url`
  points here. Routes traffic to PM's front-proxy SNI router port. When PM's
  kcp is publicly reachable, the connection is closed after credential delivery
  and no `proxy-url` is needed.
- Manages many simultaneous connections.

**Shared lifecycle (both directions):**

- Creates and manages `RemoteAccess` objects (one per live connection).
- Offboarding detection:
  - 401 after JWT `exp` → expired.
  - Persistent 401 before `exp` (including failed renewal) → revoked; deletes
    the Secret and `RemoteAccess`, emits a Pod Event, reports not-ready.
  - 403, 5xx, network errors: not counted as offboarding.
- Re-onboarding: create a new `RemoteAccessRequest` (User-initiated) or a new
  `ProviderAccessRequest` (PM-initiated).

### `provider` SDK (User-side, `go.platform-mesh.io/sdk/provider`, binary: `platform-mesh-provider`)

- **Go SDK** for portal integration: the Provider's portal code calls the SDK
  to create per-connection authorized access CRDs, keyed to portal account
  identity. The provider SDK watches and reconciles them. The Provider controls
  timing (eager at account creation, lazy at first registration, or any policy).
- Exposes a public HTTPS registration endpoint that remote-access-operator calls
  once per connection, presenting the onboarding token.
- On a valid registration: authenticates the token against the authorized access
  CRD, returns the connection ID and remote-access SDK `/access` URL.
- Issues long-lived reusable onboarding tokens, one per portal account. One
  account → many PM instance connections; each identified by clusterName.
  Idempotent: re-registering the same clusterName is a no-op.
- Provider can revoke a single connection without invalidating the onboarding
  token for other PM instances.
- GitOps-safe: applying the same `ProviderAccessRequest` N times produces the
  same result.

### `remote-access-operator` (PM-side, `platform-mesh/operators/remote-access-operator`, binary: `remote-access-operator`)

- Reconciles `RemoteAccessToken` objects; mints the join token string.
- Serves the `/join` redeem endpoint (User-initiated, all replicas).
- On User-initiated redemption: mints SA token via kcp TokenRequest for the SA
  referenced in the binding, returns kcp URL + CA bundle + SA token. Creates
  `RemoteAccessGrant`.
- Watches `RemoteAccessGrant` objects and `ProviderAccessRequest` objects.
- PM-initiated: calls provider SDK's registration endpoint, receives `/access`
  URL and connection ID. Derives a short-lived call credential from the
  long-lived access token. Dials remote-access SDK's `/access`. Delivers kcp
  URL + CA bundle + SA token. Creates `RemoteAccessGrant` on success.
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
| Join token `pmj1.…` | `remote-access-operator` | No (single-use) | PM `/join` URL, token ID, secret |
| Onboarding token `pmop1.…` | `provider` SDK | Yes (per portal account) | provider SDK registration URL, token ID, secret |
| Access token `pma1.…` | `remote-access` SDK | Yes (per connection) | remote-access SDK `/access` URL, token ID, secret |

`<prefix>.<base64url(URL)>.<tokenID>.<secret>` — the URL pins the recipient so
the token cannot be redirected; the token ID is safe to log; the secret is the
credential. remote-access-operator derives a short-lived call credential from
the access token secret for each dial to `/access`.

Join token lifetime: open (see below).

---

## Decisions

**1. SA token minting.** `remote-access-operator` is a single binary (merging
the former pm-join and pm-connect roles). It owns TokenRequest for both
User-initiated and PM-initiated. All replicas serve `/join`; leader election
governs tunnel management only.

**2. Reconnection.** On tunnel drop, remote-access-operator re-dials using the
existing `RemoteAccessGrant` (which holds the `/access` URL and access token).
It always delivers a fresh SA token on every dial. `RemoteAccessGrant` has the
SA as an ownerReference — SA deletion triggers GC deletion of
`RemoteAccessGrant`, which remote-access-operator observes and stops re-dialing.

**3. Offboarding / revocation.** `RemoteAccessGrant` deletion is the
authoritative teardown signal: remote-access-operator closes the tunnel;
remote-access SDK detects closure and deletes `RemoteAccess` + Secret. From the
user side: persistent 401 → remote-access SDK closes tunnel →
remote-access-operator reconnects → if SA gone, `RemoteAccessGrant` already
GC'd → stop; if SA present → reconnect normally.

**4. CA bundle rotation.** PM publishes the current CA bundle and server URLs in
a well-known object in each workspace where a `RemoteAccessGrant` exists.
remote-access-operator maintains this object (watches the CA bundle source in
PM's cluster, handles the old/new overlap window). remote-access SDK reads it
periodically using its SA token and rewrites the kubeconfig when it changes.

**5. `ProviderAccessRequest` shape.** `Provider` object is kept. `spec.remote`
removed. `serviceAccountRef` is not on the PAR — the Owner creates the PAR with
only `spec.providerRef` and `spec.onboardingToken`. PM automatically creates
Provider → Provider Workspace → dedicated SA → `RemoteAccessGrant` (with SA
ownerReference). remote-access-operator finds the pending `RemoteAccessGrant`
and completes the PM-initiated connection.

**6. `RemoteAccessToken` location.** Always the same workspace as the SA it
references. Provider case: Provider Workspace. Plain user case: their own
workspace.

**7. RFC split.** Three RFCs:
- *RFC 1*: remote-access SDK + remote-access-operator, User-initiated only. Core primitive, no tunnel, public PM.
- *RFC 2*: provider SDK, `ProviderAccessRequest`, PM-initiated without tunnel. Provider/portal use case, public PM.
- *RFC 3*: tunnel mode. Front-proxy SNI router (new kcp work), remote-access SDK tunnel mode. Extends both directions to private PM instances.

**8. Join token lifetime.** Configurable; default 1 h.
