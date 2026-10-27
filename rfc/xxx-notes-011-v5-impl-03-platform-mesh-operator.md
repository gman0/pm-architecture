# v5 implementation plan: `platform-mesh-operator`

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). `platform-mesh-operator` (`platform-mesh/operators/platform-mesh-operator`) owns every `Provider`, local or remote, from creation to deletion. For Remote Providers that includes `ProviderOnboardingRequest`s, the Provider Workspace, Registration, `ProviderEndpoint`s, health, Secret Changes, the Connector CA Set and offboarding. None of it is on the Session path.

## Responsibilities

**Owns:**
- reconciling `Provider`s, local and remote, and with them their `ProviderOnboardingRequest`s;
- Provider Workspaces and everything PM creates in them: the admin SA, the Connection Export binding, the policies that restrict RBAC subjects and SA-token `Secret`s, the Connector RBAC, and for Local Providers the scoped kubeconfig;
- `ProviderEndpointRegistration`s and their `Secret`s, in its namespace in PM's runtime cluster ([02 APIs, "ProviderEndpointRegistration"](./xxx-notes-011-v5-impl-02-apis.md));
- every call to a Provider's Connection API ([01 Protocol](./xxx-notes-011-v5-impl-01-protocol.md));
- `ProviderEndpoint`s in Provider Workspaces ([02 APIs](./xxx-notes-011-v5-impl-02-apis.md));
- the whole `Provider` status: conditions, `phase`, `status.connection`;
- the Connector CA Set: deriving it, pushing it to every Connection, and the `ConnectorCASet` status ([04 Connector CA](./xxx-notes-011-v5-impl-04-connector-ca.md)).

**Doesn't own:**
- Sessions, proxying or SA tokens ([05 Connector](./xxx-notes-011-v5-impl-05-connector.md));
- issuing Connector Certificates ([04 Connector CA](./xxx-notes-011-v5-impl-04-connector-ca.md)).

## What it consumes

- **The Providers Export's endpoint slice**: `ProviderOnboardingRequest`s and `Provider`s in Owners' workspaces, and token `Secret`s there, read by name and uncached through the export's `Secret`s claim (see "`Secret`s").
- **The Connection Export's endpoint slice** (`:root:providers:system`): it reconciles Provider Workspaces over it, each through its shard's virtual workspace, and writes their `ProviderEndpoint`s.
- **`Workspace` objects in `:root:providers`**, for Provider Workspaces' logical clusters.
- **The Connector CA's `CertificateRequest`s** in cert-manager's namespace, for the CA's public certificates; **the Connector Certificates' `CertificateRequest`s** in the Connectors' namespace, to see which CA each Connector's current certificate chains to; and the **`ConnectorCASet`**, whose `spec.distrust` it reads and whose status it writes ([04 Connector CA](./xxx-notes-011-v5-impl-04-connector-ca.md)).
- **Outbound HTTPS to Providers' Base URLs.**

## Principles

- **One owner per `Provider`**: one reconcile, keyed by the `Provider`, writes its whole status, handles its onboarding requests and runs one finalizer.
- **No kcp credential for Remote Providers** (overview, principle 1): no token `Secret`, no kubeconfig.
- **Write before send** (overview, principle 7): every new Connection Secret is in the `ProviderEndpointRegistration`'s `Secret` before its hash is sent or written into the `ProviderEndpoint`.
- **The Provider-side is the source of truth** for what it has pinned; it's read back with `GET` and converged.
- **Only PM changes a Connection Secret**, through the `ProviderEndpoint` (see "Secret Changes"); no call does.
- **Never trust Owner-writable fields for links** (overview, principle 4).
- **Bounded, polite egress**: per-Provider rate limits, jitter and backoff; Providers are external services.
- **Aggregate status only** towards Owners.

## Reconciling Providers

The `Provider` reconciler runs a fixed sequence of steps per kind. Each step only runs once the ones before it are done, and each reports a condition named after it (see "Conditions and phase").

| Step | Local Providers | Remote Providers |
|---|---|---|
| 1 | `Workspace`: the Provider Workspace and the admin SA | `Workspace`: the Provider Workspace and the admin SA `default/provider` (cluster-admin in the workspace, the SA that Connectors act as) |
| 2 | `Kubeconfig`: an SA-token `Secret`, and a scoped kubeconfig copied to the Owner's workspace (RFC 006) | `ConnectionBinding`: the binding to the Connection Export (below) |
| 3 | | `ConnectorAccess`: the policies in the Provider Workspace, confirmed, then the Connector RBAC (below) |
| 4 | | `Registered`: handles the onboarding request (see "Onboarding requests"), makes the first Control Call, writes the `ProviderEndpoint` (see "First Control Call and ProviderEndpoint") |
| 5 | | `Connected`: health on every poll, plus the Connection's upkeep (see "Health") |

**`ProviderOnboardingRequest`s are reconciled under their `Provider`'s key.** An event for a request enqueues the key of the `Provider` its `spec.providerRef` names (the request's logical cluster plus that name), so everything about one Connection (the request, Registration, the `ProviderEndpointRegistration`, Secret Changes, health) runs in one queue, one reconcile at a time. The `ProviderEndpointRegistration` therefore has one writer in code; PM Administrators' edits to it are updates with `resourceVersion`, like any other write. If the `Provider` doesn't exist, the reconcile sets the condition `ProviderNotFound` on the request and stops there.

The Provider Workspace is `:root:providers:<Provider name>-<the Provider's kcp.io/cluster annotation>`. The annotation is set by kcp, not the Owner. Its logical cluster name (`clusterName`) is read from the `Workspace` object in `:root:providers`, which is PM-owned; nothing about the workspace is taken from the `Provider`'s spec or status.

### Connection Export binding

- **Created with a fixed name** in the Provider Workspace, referencing the Connection Export in `:root:providers:system` and accepting its one permission claim (read-only `apiexportendpointslices`) ([02 APIs, "The binding in Provider Workspaces"](./xxx-notes-011-v5-impl-02-apis.md)).
- **Recreated if deleted.** The Provider can't delete it through a Session ([05 Connector, "Authorization and policy"](./xxx-notes-011-v5-impl-05-connector.md)), but a PM Administrator's error could.
- **Bind permission**: `platform-mesh-operator` needs `bind` on the Connection Export.

### Connector RBAC

Before granting any access, `platform-mesh-operator` creates in the Provider Workspace the two `ValidatingAdmissionPolicy`s, their bindings and the ClusterRole `providers.platform-mesh.io:apiexport-bind` ([02 APIs, "Policies in Provider Workspaces"](./xxx-notes-011-v5-impl-02-apis.md)), with its own kcp username, from its configuration, in their `matchConditions`. kcp enforces new policies asynchronously, so it then **confirms enforcement**: a dry-run create (`dryRun=All`) of a `RoleBinding` that the policy must refuse (a `User` subject outside `apis.kcp.io:binding:`), impersonating `default/provider`. Until the policy refuses it, `ConnectorAccess=False` with `PolicyNotEnforced`, and nothing later runs; no `ProviderEndpoint` is written before the policies are in force.

Then, for the **Connectors' identity** (one group shared by all shards' Connectors), a role and binding:
- `access` on `/` (workspace content);
- `create` on `serviceaccounts/token` with `resourceNames: [provider]`, in namespace `default`.

Nothing else. Connectors read `ProviderEndpoint`s through the Connection Export, not through this RBAC ([05 Connector, "Identity towards kcp"](./xxx-notes-011-v5-impl-05-connector.md)). The policies, their bindings and the ClusterRole are recreated if they're deleted or differ; the Provider can't change them through a Session ([05 Connector, "Authorization and policy"](./xxx-notes-011-v5-impl-05-connector.md)).

## Spec validation

A CEL rule on `ProviderSpec`: `spec.remote` excludes `providerKubeconfigSecret` and `hostOverride`, which only apply to Local Providers. An Owner who sets both gets an error at apply time. The remote sequence never creates a kubeconfig either way, as defense in depth. Switching an existing `Provider` between local and remote isn't supported for now.

## Conditions and phase

**Conditions**, for both kinds:
- **one per step, named after it**: `True` when it's done, `False` when it isn't, with a message saying why. Steps after one that isn't done aren't run;
- **`Ready`**: `True` when every step is done. The printer column shows it.

For Remote Providers that's `Workspace`, `ConnectionBinding`, `ConnectorAccess`, `Registered`, `Connected` and `Ready`, plus the warnings **`ProtocolOutdated`** and **`TrustOutdated`** (see "Health"), which don't affect `Ready`.

**Messages start with a stable label** and say what to fix, for example `EdgeBotChallenge: the Provider's edge answered the WebSocket upgrade on /control with an HTML page. Ask the Provider to exempt its connection paths from bot protection; see its edge checklist.` People read these; the labels keep them consistent and testable. The labels are listed with the steps that use them below.

**`phase`**, for both kinds:

| Phase | Means |
|---|---|
| `Pending` | Nothing done yet |
| `Provisioning` | Not yet usable for the first time; the conditions say which step is pending |
| `Ready` | Usable: `Ready` is true |
| `Degraded` | Was `Ready` once, and `Ready` is false now; the conditions say why. For Local Providers this usually means the reconciler can't currently confirm or repair something (kcp unavailable, the token `Secret` recreated), while the existing kubeconfig keeps working |
| `Deleting` | Being deleted |

- **Computed in one place**, after each reconcile, from the conditions. `Ready` and `Degraded` always mirror the `Ready` condition.
- **`Degraded` needs a `Provider` that was `Ready` before**, remembered in `status.lastReadyTime`, so a restart doesn't turn it back into `Provisioning`.
- **ManagedProviders** wait for their `Provider`'s `phase` to be `Ready`; how they react to `Degraded` is a [follow-up](./xxx-notes-011-v5-impl-todo.md).

**`status.connection { url, connectionID }`** on Remote Providers is for the Owner's information: the Base URL and the Connection ID in use, updated when the `Provider` is onboarded again or its Base URL changes, never used as a link.

## Onboarding requests

Part of the `Provider` reconcile (see "Reconciling Providers"). A request is one action: spend a single-use token to register the `Provider`'s Connection. Everything after Registration is reported on the `Provider`.

1. **Validate the request**: it's the oldest unfinished request for its `Provider` (otherwise fail it, naming the active one); the token `Secret` exists and parses, and its `tokenID` is the request's `spec.tokenID`; the Base URL inside passes the egress rules (see "Egress"). The creator isn't checked: `create` on requests is granted together with managing `Provider`s ([02 APIs, "ProviderOnboardingRequest"](./xxx-notes-011-v5-impl-02-apis.md)). Phase `Pending`. Requests are only processed once Remote Providers are installed: the kcp bootstrap applied and the `providerconnections` HelmRelease Ready ([09 Deployment, "Rollout order"](./xxx-notes-011-v5-impl-09-deployment.md)); until then they stay `Pending` with `RemoteProvidersNotReady`.
2. **Check the `Provider`** named in `spec.providerRef`. The Owner creates it; this step never writes `Provider`s.

   | The `Provider`… | The reconcile… |
   |---|---|
   | doesn't exist (yet) | sets `ProviderNotFound` on the request |
   | exists, not registered | uses it; this includes a `Provider` left behind by a failed Onboarding, and one whose Connection the Owner abandoned (see "Abandoning a Connection") |
   | exists, registered, and the Provider-side knows the Connection | fails the request: a live Connection needs no new Registration, and a lost or rejected Connection Secret is repaired without the Owner (see "Secret Changes") |
   | exists, registered, and the Provider-side reports the Connection unknown or deleted | reuses it and registers a **new** Connection, with the token's Base URL |
   | exists, registered, and the Provider-side doesn't answer | waits: only a definite "unknown" allows a new Registration. The message says how to give up on the Connection instead, with a ready-to-paste `kubectl annotate` line (see "Abandoning a Connection") |
   | exists, not remote (e.g. a ManagedProvider's) | fails the request |

   For a registered `Provider`, the reconcile asks the Provider-side itself (a `GET` over a Control Session) instead of reading the `Provider`'s conditions, which the Owner could edit. Only an answer inside the Inner TLS, against the pinned Server Key, counts as "unknown" ([01 Protocol, "Errors and retries"](./xxx-notes-011-v5-impl-01-protocol.md)).

   **Onboarding a registered `Provider` again** keeps the Provider Workspace, and with it the APIExport's identity and consumers' bindings; deleting the `Provider` and creating a new one would break every binding for good. It's meant for two cases:
   - **switching Providers**: the old Provider-side deletes the Connection, or the Owner abandons it, and a token from the new Provider names its own Base URL;
   - **a Connection the Provider-side deleted and won't restore**.

   **The new Connection may be backed by a different vendor.** That vendor then gets everything the export serves and claims in consumer workspaces. That's the Owner's decision, as it already is for Local Providers, whose Owners hold an admin kubeconfig they can hand to anyone: consumers trust the Owner, not a particular vendor. When a new Connection replaces an earlier one, an Event on the `Provider` names the old and the new Base URL host and `connectionID`.

   **The new Connection has a new `connectionID`.** Provider-side data carries over only where the Provider can find it from kcp objects (an ID in an object's `status`, see [07 SDK, "Requirements on Providers"](./xxx-notes-011-v5-impl-07-provider-sdk.md)); data keyed only by the old `connectionID` is orphaned. When the same Provider still has the data, or can recover it, the preferred path is Provider Administrators importing the old entry, which keeps the `connectionID` ([07 SDK, "Administration"](./xxx-notes-011-v5-impl-07-provider-sdk.md)). The `Registered` condition's `ConnectionUnknown` message says so.

   These rows never create a second Connection next to a live one.
3. **Wait for the Provider Workspace's `clusterName`.** Phase `Pending`.
4. **Register** (see "Registration"). Phase `Registering`.
5. **Done**: phase `Registered`, with the `connectionID` in status, and a message pointing to the `Provider`. A request never changes again once `Registered` or `Failed`; Onboarding again needs a new request.

`Failed` is only possible before Registration succeeds: the token refused, validation, or another active request. After Registration, the `Provider` waits for the first Control Call, the `ProviderEndpoint`, Sessions and kcp for as long as it takes.

## Registration

1. **Create the `ProviderEndpointRegistration`** for the Provider Workspace, if it doesn't exist, and **generate the Connection Secret** into its `Secret` (write before send). When a registered `Provider` is onboarded again, this replaces the old Connection Secret, and step 3 replaces its spec.
2. **Register** over `GET /v1/connections/register/{tokenID}`: a WebSocket upgrade, then Inner TLS accepting only the Server Key whose `key-sha256` is the token's `keyfp`, then one request carrying `tokenSecret`, the request's UID as idempotency key, the `clusterName`, the Connection Secret's hash and the current Connector CA Set ([01 Protocol, "Registration"](./xxx-notes-011-v5-impl-01-protocol.md)). The Provider's Edge sees none of it.
3. **On success**, persist the `connectionID`, **the Base URL from the token, the returned Server Keys and `unusedExpiresAt`** in the `ProviderEndpointRegistration`'s spec, in one write; then record the `connectionID` in the request's status. The token's data is deleted afterwards, so the `ProviderEndpointRegistration` is the only place the Base URL survives until the `ProviderEndpoint` is written. The Server Keys are trust anchors for the Control Session and the `ProviderEndpoint`, so they're never kept in Owner-writable status, and never only in memory.
4. **On a lost response, or a crash before step 3**, retry with the same token, idempotency key, hash and `clusterName`; the same response comes back. A changed Connector CA Set doesn't matter: the first Control Call pushes the current one. The Provider-side honours such retries for 7 days by default ([01 Protocol, "Registration"](./xxx-notes-011-v5-impl-01-protocol.md)), far longer than any expected outage.
5. **On refusal**, fail the request with a reason the Owner can act on: `TokenRefused` when the Provider-side refuses the token: before the upgrade if it's unknown, expired, past its retry window, or its Server Key is unusable (e.g. retired since the token was issued), and inside the Inner TLS if another Registration spent it. The Owner asks the Provider's portal for a new token. An `IdempotencyConflict` (the same request UID with another `secretHash` or `clusterName`, which `platform-mesh-operator` never sends, since both are persisted before the first attempt) also fails the request, and the Edge labels when the Edge is in the way (see "Edge diagnosis"). A retry refused after the window leaves an orphaned Connection on the Provider-side, which the SDK expires because it never saw an authenticated Control Call or Session ([07 SDK, "Connection API"](./xxx-notes-011-v5-impl-07-provider-sdk.md)).

Every step resumes from persisted state: the `Secret` holds the Connection Secret, the `ProviderEndpointRegistration` the `connectionID`, Base URL, Server Keys and `unusedExpiresAt`; the request's status holds the phase.

## First Control Call and ProviderEndpoint

The `Registered` step of a remote `Provider`. Its condition is derived from state only: `Registered=True` once the Registration's result is persisted and the `ProviderEndpoint` exists, which is only written after the Inner TLS against the pinned Server Key succeeded and the Provider-side knew the Connection.
1. **Wait for a Registration**: until the `ProviderEndpointRegistration` holds a `connectionID`, Base URL and Server Keys, `Registered=False` with `NotRegistered: waiting for an onboarding request`.
2. **Make a first Control Call** (`GET` over a Control Session). The `ProviderEndpoint` isn't written before the Inner TLS against the pinned Server Key succeeds, so a control path that's broken from the start shows up here, with its edge-diagnosis label (see "Edge diagnosis"), before any Session opens. The outcome decides what happens next:
   - **Authenticated**: push the current Connector CA Set if the pinned one differs, which it may after a retried Registration or a Rollover since (see "Connector CA Set"), then write the `ProviderEndpoint`.
   - **`SecretRejected`**: the Inner TLS succeeded and the answer concerns this Connection, but the Connection Secret is refused (e.g. the Provider-side lost or garbled its hash). The control path works, so don't wait: write `pendingSecret` first, reusing one that's already there, then create the `ProviderEndpoint` with its `pendingSecretHash` in one write, and continue the Secret Change on the following polls (see "Secret Changes"). This first repair of a new Connection isn't rate-limited: it's recognised by the `ProviderEndpoint` not existing yet.
   - **"Unknown"**, answered inside the Inner TLS: `ConnectionUnknown` (it no longer knows the Connection; the Owner can onboard the `Provider` again, or the Provider can import the old entry).
   - **No answer** (an Edge label, `ControlUnreachable`, or an Inner TLS handshake that fails against the pinned keys): `Registered=False` with the label, retried with backoff. While no authenticated contact has succeeded, the message adds the Provider-side's deadline from `unusedExpiresAt`: `The Provider will remove this Connection at <time> unless it's reached by then.` After it, the Provider-side removes the Connection, the next Control Call gets "unknown", and the Owner onboards again; nothing is lost, since the Connection never had a Session.
3. **Write the `ProviderEndpoint`** through the Connection Export (see "Writing ProviderEndpoints"). `platform-mesh-operator` reconciles Provider Workspaces over the export's endpoint slice, each through its shard's virtual workspace; a write that fails because the workspace isn't available there yet is retried like any other reconcile error.

Every live Connection has therefore had a working control path at least once.

## Writing ProviderEndpoints

One `ProviderEndpoint` per Provider Workspace, written through the Connection Export ([02 APIs, "ProviderEndpoint"](./xxx-notes-011-v5-impl-02-apis.md)):
- **Written from the `ProviderEndpointRegistration`**, which is authoritative ([02 APIs, "ProviderEndpointRegistration"](./xxx-notes-011-v5-impl-02-apis.md)): `connectionID`, Base URL and Server Keys are copied; `pendingSecretHash` is the hash of its `Secret`'s `pendingSecret` while the Connection Secret changes; `caBundle` is copied from the `Provider`'s `spec.remote.caBundle`, so an Owner's update reaches Connectors' new Sessions. Every change goes into the `ProviderEndpointRegistration` or its `Secret` first; the `ProviderEndpoint` follows.
- **Drift repair**: if the `ProviderEndpoint` is missing (e.g. the binding was recreated) or differs, it's rewritten from them. Connectors react to the change ([05 Connector, "Discovery"](./xxx-notes-011-v5-impl-05-connector.md)).
- **Read as input only when the `ProviderEndpointRegistration` is lost**: it's rebuilt from the `ProviderEndpoint` (`connectionID`, Base URL, Server Keys), and the Connection Secret is repaired, since the `Secret` goes with it (see "Secret Changes"). If both are lost, the `connectionID` can't be recovered by PM: `Registered=False` with `ConnectionIDMissing`, until a PM Administrator re-enters it in a new `ProviderEndpointRegistration`, from the Provider's portal or the Owner's records.
- **Server Key updates**: when `GET` shows a new Server Key, it's added to the `ProviderEndpointRegistration` and then the `ProviderEndpoint`; the Provider-side presents it on a Connection only once that Connection's `ProviderEndpoint` lists it. Retired keys are removed once `GET` no longer lists them ([01 Protocol, "Server Keys"](./xxx-notes-011-v5-impl-01-protocol.md)).
- **A new Base URL**: a Provider that moves to a new Base URL asks PM Administrators, who edit `url` in the `ProviderEndpointRegistration`; the `connectionID`, Server Keys and all Provider-side state stay. The new URL passes the egress rules before it's written, and Connectors replace their Sessions when the `ProviderEndpoint`'s `url` changes. If the new hostname's certificate comes from a different private CA, the Owner adds that CA to `caBundle` first, or the new URL fails with `EdgeTLS`. Abandoning the Connection and onboarding again also works, but gives a new `connectionID` (see "Abandoning a Connection").
- **Administrator input for recovery**: a PM Administrator edits a Connection's Base URL or Server Keys in its `ProviderEndpointRegistration`, for example after a Provider rebuilt elsewhere lost its keys. The input lives in PM's runtime cluster, never in an Owner-writable field; the `ProviderEndpoint` follows. Server Keys are never fetched over plain HTTPS. Runbooks for it are a [follow-up](./xxx-notes-011-v5-impl-todo.md).
- **Deleted first** when the `Provider` is deleted (see "Deletion") or its Connection abandoned.

## Abandoning a Connection

For a Provider that's gone for good: the Owner, or a PM Administrator, sets `providers.platform-mesh.io/abandon-connection=<connectionID>` on the `Provider` ([02 APIs, "Provider"](./xxx-notes-011-v5-impl-02-apis.md)). It acts only if the value is the `connectionID` in the `ProviderEndpointRegistration`, so a stale annotation never hits a later Connection. In order:
1. **Delete the `ProviderEndpoint`**, so every Connector closes its Sessions.
2. **Call `DELETE`** in a Control Session, best effort, with a short timeout.
3. **Delete the `ProviderEndpointRegistration`**; its `Secret` follows by owner reference.

The Provider Workspace, its admin SA, the Connection Export binding, the policies and the Connector RBAC stay, and with them the APIExport and consumers' bindings. The `Provider` shows `Registered=False` with `ConnectionAbandoned`, plus an Event; it's no longer registered, so a normal request onboards it anew, with a new `connectionID`.

The messages for a request waiting on a Provider-side that doesn't answer, and for a `Connected=False` that has lasted long, include a ready-to-paste line: `kubectl annotate provider <name> providers.platform-mesh.io/abandon-connection=<connectionID>`.

## Health

The `Connected` step runs on every reconcile of a registered remote `Provider` and requeues at the poll interval (every few minutes per Connection, with jitter):
- **Polls `GET /v1/connections/{id}`** over a Control Session, reusing an open one where possible.
- **Never gives up on a Connection it can't reach.** Failed polls back off, capped at about the normal poll interval, so a Provider that returns after any outage is caught up within minutes (see "Connector CA Set").
- **Reports the Connection's state** as the `Connected` condition, from the first failing check:
  1. **Control path**: the edge-diagnosis labels (see "Edge diagnosis"), `ControlUnreachable` (DNS, connection refused, timeout, or an Inner TLS handshake that fails against the pinned keys), or `SecretRejected` (the Inner TLS succeeds and the answer concerns this Connection, but the Connection Secret is refused, which starts a repair, see "Secret Changes");
  2. **Sessions**: `NoSessions`, when the status's `workspace` or a `virtualWorkspaces` host has none ([01 Protocol, "Status"](./xxx-notes-011-v5-impl-01-protocol.md));
  3. **kcp**: `KcpUnreachable`, when its requests don't reach kcp.

  The message adds counts and the last error, never shard names (overview, principle 9). If the control path fails, the status can't be read, so the control-path label wins and the message mentions the last known Session state. While no authenticated contact has succeeded since Registration, it also carries the `unusedExpiresAt` warning (see "First Control Call and ProviderEndpoint"). A failure that has lasted long carries the abandon line (see "Abandoning a Connection").
- **Sets the warning condition `ProtocolOutdated`** while its Control Sessions still negotiate the previous protocol version through ALPN, so the Owner can ask the Provider to upgrade its `provider-gateway` ([07 SDK, "Releases and protocol versions"](./xxx-notes-011-v5-impl-07-provider-sdk.md)).
- **Sets the warning condition `TrustOutdated`** while the Connection can't be reached and its last known Pinned Bundle still holds a CA that is `Distrusted` or `DistrustPending` ([04 Connector CA, "Compromise response"](./xxx-notes-011-v5-impl-04-connector-ca.md)), so the Owner knows that Provider still trusts a CA PM gave up. It clears with the first successful push.
- **Does the Connection's upkeep** on the same polls: Server Key updates in the `ProviderEndpoint`, Secret Changes (see "Secret Changes"), and Connector CA Set pushes and the tamper check (see "Connector CA Set").
- **Per-shard and per-host detail goes to metrics**, never to conditions (overview, principle 9).

**Readiness for Onboarding**: the status's `workspace` has Sessions and reaches kcp; before any consumer binds, that's all that's required.

### Edge diagnosis

On Registration, on the first Control Session after Registration and on every failed reconnect, failures are classified as described in [01 Protocol, "Diagnosing the Edge"](./xxx-notes-011-v5-impl-01-protocol.md). The label starts the message of the request (during Registration), `Registered` (during the first Control Call) or `Connected` (afterwards):
- `EdgeUpgradeRefused`: a non-`101` response; any response before the upgrade is the Edge's;
- `EdgeBotChallenge`: an HTML page instead of an upgrade;
- `EdgePayloadBlocked`: closed right after a binary message;
- `EdgeTimeout`: closed at a regular idle time or age;
- `EdgeTLS`: the server certificate isn't trusted by web PKI or the `Provider`'s `caBundle`, typically a private CA the Provider rotated before the Owner updated `caBundle` ([02 APIs, "Provider"](./xxx-notes-011-v5-impl-02-apis.md));
- `EdgePathRewritten`: the Provider-side answered `ConnectionMismatch`, so the Edge sent the Control Session to another Connection. It's never treated as `SecretRejected` and never starts a repair.

Each message points the Owner at the Provider's Edge checklist ([08 Quickstart, "Connection API server"](./xxx-notes-011-v5-impl-08-provider-quickstart.md)). Only Control Sessions are observed here; Connectors' Session failures go to their metrics, where the same classification applies.

## Secret Changes

The only way a Connection Secret changes ([01 Protocol, "Changing the Connection Secret"](./xxx-notes-011-v5-impl-01-protocol.md)):
1. Generate a new Connection Secret; write it as `pendingSecret` in the `ProviderEndpointRegistration`'s `Secret`.
2. Write its hash into the `ProviderEndpoint`'s `pendingSecretHash`.
3. Wait for the Provider-side to adopt it, which it does when it reads the `ProviderEndpoint` over a Session: authenticate with `pendingSecret` (a `GET`) until it succeeds.
4. Move `pendingSecret` to `secret`, then clear `pendingSecretHash`.

**One at a time** per Connection. A trigger that arrives while a change runs is acted on after it completes; several on-demand requests in the meantime are coalesced into one.

**Triggers:**
- **On demand**: the annotation `providers.platform-mesh.io/rotate-connection-secret` on the `Provider`, set by the Owner or by PM Administrators, e.g. `kubectl annotate provider my-provider providers.platform-mesh.io/rotate-connection-secret="$(date -u +%FT%TZ)" --overwrite`. Any new value triggers one change; the value last acted on is kept in the `ProviderEndpointRegistration` (`rotationRequest`), so re-applying the same value (e.g. through GitOps) does nothing. On-demand changes are rate-limited per Connection (e.g. at most one per 10 minutes), so a script or a GitOps loop writing timestamps can't keep a Connection changing; a deferred one is noted in the `Connected` message and counted. Trusting it is harmless: it only makes PM change a Connection Secret that PM controls, and it's never used as a link.
- **After tampering** with a Pinned Bundle, and when a **distrusted CA is still pinned** (see "Connector CA Set"). Not rate-limited.
- **Repair**: when the `Secret` is missing, or the Provider-side rejects both `secret` and `pendingSecret` while the Inner TLS against its pinned Server Key succeeds and the answer concerns this Connection (e.g. its store was restored from an older backup). A missing `ProviderEndpointRegistration` is first rebuilt from the `ProviderEndpoint` (`connectionID`, Base URL, Server Keys). Repairs are rate-limited per Connection (e.g. at most one per hour), so an Edge or Provider bug that breaks authentication can't cause a loop of changes; the first repair of a new Connection, before its `ProviderEndpoint` exists, is exempt (see "First Control Call and ProviderEndpoint").

**The code carries a comment at the list of triggers** about periodic rotation: what it would need (a last-change timestamp, a configured interval, due times spread by `connectionID`), why it might be wanted (compliance), and why it isn't needed (see "Declined").

**Properties:**
- **Sessions are unaffected**: they never use the Connection Secret. A change needs one, though; without Sessions it waits, and the Provider-side keeps its current hash.
- **On restart mid-change**, try `pendingSecret` first, then `secret`, and resume; the `ProviderEndpoint` still names the pending hash.
- **No call, Onboarding Token or portal account can change the Connection Secret.** A leaked Connection Secret can: it lets its holder push a Pinned Bundle with a CA of their own, and through a Session under it change the Connection Secret, which locks PM out of that Connection ([01 Protocol, "Changing the Connection Secret"](./xxx-notes-011-v5-impl-01-protocol.md)). If the tamper check sees the bundle first, it's restored and the Connection Secret changed (see "Connector CA Set"). Otherwise the `Provider` shows `SecretRejected`, repairs don't hold, and recovery is out-of-band: the Provider Administrators re-pin the bundle and PM repairs the Connection Secret, or the Owner abandons the Connection and onboards again. A leaked Connection Secret means the PM-side leaked, so it's handled as an incident like any other PM-side secret leak: find the leak, rotate PM's credentials. The runbook covers it.

## Recovery

Each case is covered elsewhere ([01 Protocol, "Recovery"](./xxx-notes-011-v5-impl-01-protocol.md)):
- **A lost or rejected Connection Secret**: repaired automatically (see "Secret Changes").
- **A Connection the Provider-side no longer knows**: `Registered=False` with `ConnectionUnknown`; the Owner onboards the same `Provider` again, which registers a new Connection (see "Onboarding requests"). This includes a Connection the Provider-side expired because nothing reached it within `unusedExpiresAt`.
- **A Provider that won't come back**: the Owner abandons the Connection (see "Abandoning a Connection").
- **A Provider's new Base URL**: a PM Administrator edits it (see "Writing ProviderEndpoints").
- **No Session verifies**: `Connected=False` with `SecretRejected`, its message noting that no Session verifies; Provider Administrators re-pin the Pinned Bundle out-of-band.
- **Disasters on either side**: out-of-band, using the administrator input in "Writing ProviderEndpoints" and the Provider Administrators' runbooks ([07 SDK, "Administration"](./xxx-notes-011-v5-impl-07-provider-sdk.md)).

## Connector CA Set

cert-manager renews the Connector CA; `platform-mesh-operator` makes sure every Provider trusts the new CA before Connectors use it ([04 Connector CA, "Rollover"](./xxx-notes-011-v5-impl-04-connector-ca.md)):
- **Derived, not stored**: the CA's `CertificateRequest`s, which cert-manager creates per issuance and keeps the last ten of, give the current CA, the **Connector CA Set**, each CA's drop time per Connection, and PM's unexpired CAs ([04 Connector CA, "The Connector CA Set"](./xxx-notes-011-v5-impl-04-connector-ca.md)). Requests are ordered by revision; the current CA is the newest request that is `Ready`, holds a certificate and passes validation. `platform-mesh-operator` watches them and reads only their status, which holds public certificates. Together with `spec.distrust` and the Connectors' current certificates (from their `CertificateRequest`s), a restart or failover derives the same result.
- **Why `CertificateRequest`s**: they're the only place the CA certificates of past issuances are kept without the key. The CA's `Secret` holds the key and only the current certificate, and the CA `Certificate` object holds neither.
- **Push on difference**: on each poll, the Connection's pinned fingerprints (from `GET`) are compared with the bundle this Connection should pin (the Connector CA Set's certificates as PEM, without each previous CA once its drop time for this Connection has passed), and `PUT /connector-ca-set` is sent with `If-Match: <the etag just read>` where they differ, rate-limited ([01 Protocol, "Pinned Bundle"](./xxx-notes-011-v5-impl-01-protocol.md)). A `412` means the bundle changed since: read it again and decide anew. When a new CA appears, every Connection is requeued, so pushes go out within minutes; drops happen on the regular polls after each Connection's drop time. Registration always sends the current set, and the first Control Call pushes it again if it changed meanwhile. The set is never empty: if deriving it would leave no CA, nothing is pushed, `Valid=False`, and the alert fires.
- **Catch up as soon as a Connection is back.** A Provider that was unreachable during a Rollover, however long, still pins an old set; its control channel doesn't depend on the Connector CA, so the **first successful Control Call after a Connection was unreachable** pushes the current set right away. The SDK accepts the push even if every CA it pins has expired.
- **Tampering**: a pinned certificate that's unexpired and isn't one of PM's unexpired CAs (distrusted ones don't count as PM's) can only have come from someone holding the Connection Secret. Then:
  1. restore the Connector CA Set with the current Connection Secret (with `If-Match`), so PM's Sessions verify again; the Provider-side closes every Session that doesn't chain to it, including any the thief opened ([01 Protocol, "Pinned Bundle"](./xxx-notes-011-v5-impl-01-protocol.md));
  2. change the Connection Secret through the `ProviderEndpoint` (see "Secret Changes"), which makes the stolen Connection Secret useless;
  3. record an **event** on the `Provider`, so the Owner sees it happened, and count it in a metric with a critical alert rule for the PM Administrators ([04 Connector CA, "Signals"](./xxx-notes-011-v5-impl-04-connector-ca.md)).

  If the thief pushes again before the change completes, step 1 repeats; the change completes on the Provider-side's first read of the `ProviderEndpoint` over a Session. Expired pinned CAs verify nothing, so they're never tampering, whoever issued them; neither is a bundle merely missing some of PM's CAs. The push above converges both.
- **A distrusted CA still pinned** is different: a Connection that was unreachable while the distrust took effect still holds it legitimately, and PM can't tell that from someone who re-pinned it with the Connection Secret. So, always, and in this order: push the corrected set, which closes every Session under the distrusted CA; then change the Connection Secret, so the new hash is only read over Sessions under CAs PM still trusts. It's reported as `DistrustedCAPinned`, an Event on the `Provider` and a metric reason, not as tampering, so the critical tamper alert stays for CAs that were never PM's. `DistrustPending` CAs are still trusted and aren't foreign.
- **Convergence**: each registered Connection (from the `ProviderEndpointRegistration`s) is classified against the current CA: **pinned** (a Control Call that succeeded after the CA was issued confirmed it), **not reached** (an attempt after the CA was issued failed, whatever the cause) or **unknown** (not tried since the CA was issued, or not since this process started). The CA has converged when no Connection is unknown and none is reached but not pinning it; nothing waits for Connections that aren't reached, and Owners see their cause on their `Provider`. After a restart, `Converged` stays `False` until every Connection has been polled once. The counts go to the metric `providerconnections_connector_ca_connections{state,shard}`, with a `reason` label for Connections not reached, never into status. `platform-mesh-operator` writes the whole `ConnectorCASet` status: every unexpired known CA with its fingerprints and `state`, the distrust results, and the conditions `Valid` and `Converged`; the `Converged` message is written only when the condition changes. Connectors read it to decide when to switch certificates ([05 Connector, "Connector Certificate"](./xxx-notes-011-v5-impl-05-connector.md)).
- **Distrust**: PM Administrators list a CA's key in `ConnectorCASet.spec.distrust`; `platform-mesh-operator` drops it once its gate is met, and never trusts that key again ([04 Connector CA, "Compromise response"](./xxx-notes-011-v5-impl-04-connector-ca.md)).

## Deletion

One finalizer per `Provider`. For Remote Providers, in order:
1. **Delete the `ProviderEndpoint`**, so every Connector closes its Sessions. kcp access ends here.
2. **Call `DELETE`** in a Control Session, with bounded retries; an unreachable Provider-side doesn't block deletion beyond a timeout. If the `ProviderEndpointRegistration` is already gone (deleted mid-deletion), there's nothing to call with, and the step is skipped. That's harmless: the `ProviderEndpoint` is already gone, so the Connection has no Sessions; the Provider-side keeps an entry its portal shows with an old `lastSeen`, which its administrators or the Owner can delete.
3. **Delete the `ProviderEndpointRegistration`**; its `Secret` follows by owner reference.
4. **Delete the Connector RBAC, then the policies, then the binding**, once nothing uses them anymore.
5. **Delete the SA and the workspace.**

For Local Providers: the SA-token `Secret`, the kubeconfig in the Owner's workspace, the SA and the workspace.

## Egress

- **Which addresses PM may reach is the deployment's decision**, through NetworkPolicy, an egress gateway or firewalls ([09 Deployment, "Network"](./xxx-notes-011-v5-impl-09-deployment.md)). On-prem and private-network Providers work.
- **Fixed rules in code**, checked before Registration, before writing a `ProviderEndpoint` and on every dial: `https` only; resolve once, check every address and dial those exact addresses; refuse loopback, link-local (which includes cloud metadata addresses), unspecified and multicast addresses; no redirects. They aren't configurable: no Provider legitimately sits there, and a pod's own loopback is the one place a NetworkPolicy never reaches.
- **No proxy from the environment**: every HTTP transport in `platform-mesh-operator` sets `Proxy` explicitly (`nil`), instead of inheriting `ProxyFromEnvironment`, so a stray `HTTPS_PROXY` on the pod can't send kcp traffic through a corporate proxy.
- **TLS** to the Base URL, for Registration and the outer TLS of Control Sessions: web PKI, or the `Provider`'s `spec.remote.caBundle` when set. Connectors verify their Sessions the same way, from the `ProviderEndpoint`'s copy.
- **Per-Provider limits**: concurrent calls, rate, backoff; jitter on periodic work. With 1,000 Connections and a 5-minute poll, steady state is about 3–4 calls per second in total.

## `Secret`s

Two kinds, in two places:
- **Connection Secrets live in the `ProviderEndpointRegistration`s' `Secret`s**, in `platform-mesh-operator`'s namespace in PM's runtime cluster, one per Connection ([02 APIs, "Its `Secret`"](./xxx-notes-011-v5-impl-02-apis.md)). Nothing in kcp holds them, so neither Owners nor any Provider with a `Secret`s claim on Owners' workspaces can read them. They may be cached with an ordinary informer on that namespace.
- **Token `Secret`s stay in Owners' workspaces**, where Owners create them. The Providers Export's `Secret`s claim is accepted with `matchAll` in PM's bindings, so `platform-mesh-operator` *could* read every `Secret` there. It doesn't cache any of them: **no `Secret` informers in kcp.** It reads the token `Secret` named in the request's spec by name, with an uncached client, and never writes to it: once spent, the token is useless (see "Declined"), and the Owner may delete the `Secret` when the request is `Registered`. Nothing about other `Secret`s in Owners' workspaces enters its memory. A token `Secret` created after its request is picked up by requeueing with backoff.
- **Rights in kcp**: the identity can reach any `Secret` the claim allows. Narrowing the claim with a label selector in PM's bindings is a PM-wide hardening outside this plan.

## Internet-facing code

The code for Remote Providers makes outbound calls to Providers on the internet and parses their responses, inside PM's most privileged component. That's accepted while the `Provider` part of `platform-mesh-operator` is still part of it; moving it into its own operator is on the [follow-ups list](./xxx-notes-011-v5-impl-todo.md). Meanwhile:
- **The Connection code and the protocol client sit in their own packages**, depending on the rest of `platform-mesh-operator` only where they plug into the `Provider` reconciler, so they can move as they are.
- **Everything parsed from Providers has size limits and fuzz tests**: Registration responses, Control Call responses, bundles, status.

## Identity and permissions

Beyond what `platform-mesh-operator` needs for Local Providers:
- **In kcp**:
  - in `:root:providers:system`: `access` on `/`, read on the Connection Export's `APIExportEndpointSlice`, and `apiexports/content` on the Connection Export, including writes to `providerendpoints`, granted by the export's maximal permission policy ([02 APIs, "Connection Export"](./xxx-notes-011-v5-impl-02-apis.md)); `bind` on the Connection Export;
  - in Provider Workspaces: creating the policies, their bindings, the ClusterRole and the Connector RBAC, and impersonating `default/provider` for the dry-run check.

  Its current identity, `kcp-admin`, has all of this; the grants are listed so that a dedicated identity later doesn't silently lose any.
- **In the runtime cluster**: `providerendpointregistrations` and `Secret`s in its own namespace; the cluster-scoped `connectorcasets` (create `default`, read `spec`, write its status), all in `providers.platform-mesh.io`; a Role in cert-manager's cluster resource namespace allowing only `get`, `list` and `watch` on `certificaterequests`, whose status holds public certificates; the same, read-only, on `certificaterequests` in the Connectors' namespace. No access to the CA's key `Secret` or issuers, which only cert-manager reads.
- **Network**: outbound HTTPS to Providers' Base URLs, within whatever the deployment allows.

## Replicas and availability

- **One active replica**, by leader election, with standbys that take over within seconds. It runs one multicluster-runtime provider over all shards of the Providers Export and one over the Connection Export; each shard's endpoint gets its own caches, so a shard whose virtual workspace is down only stalls its own objects. The work is mostly waiting on Providers, so it scales with reconcile concurrency, not replicas.
- **After a failover**, the new leader rebuilds its caches and reconciles every Connection once, within the per-Provider rate limits. Until it has, a new CA's convergence counts those Connections as unknown.
- **Outages delay** Onboarding, Secret Changes, Rollovers and offboarding; they don't stop traffic. The one exception: an outage of more than about 12 hours right when cert-manager renews the Connector CA. The new CA then never converges, Connectors fall back to it as their old certificates expire, and new Sessions fail at Providers that don't pin it, until `platform-mesh-operator` is back and pushes it. Open Sessions keep running.

## Observability

- **Events** on `ProviderOnboardingRequest`s and `Provider`s for every phase change and failure, for tampering, `DistrustedCAPinned`, an abandoned Connection and a Connection replaced by a new one, without secrets.
- **Metrics**: calls per Provider and outcome, poll latency, Connections per condition state, Secret Changes in progress (by trigger) and deferred, repairs refused by the rate limit, tampering detected, egress refusals, and the Connector CA signals (CA expiry, `Valid`, Connections per convergence state; [04 Connector CA, "Signals"](./xxx-notes-011-v5-impl-04-connector-ca.md)). Scraped through `platform-mesh-operator`'s `ServiceMonitor` ([09 Deployment, "Observability"](./xxx-notes-011-v5-impl-09-deployment.md)).
- **Audiences**: Owners act on events and conditions on their `ProviderOnboardingRequest`s and `Provider`s; the PM Administrators act on metrics, the `ConnectorCASet` status and the documented alert rules. Owners never see PM-wide signals, and PM-wide signals never name Owners' data beyond Connection IDs.
- **Logs** never contain the token, the Connection Secret or hashes.

## Work breakdown

1. The `Provider` reconciler's remote sequence, next to the local one, with onboarding requests mapped to their `Provider`'s key; `phase` computed from the conditions for both kinds, with `status.lastReadyTime`.
2. The CEL rules in `platform-mesh/apis`: local fields on Remote Providers; the request's immutable spec.
3. The remote bootstrap steps: workspace, Connection Export binding, the policies and their dry-run check, Connector RBAC.
4. Onboarding requests and their phases, including the gate on Remote Providers being installed.
5. The protocol client, in its own package: Registration and Control Sessions (WebSocket, Inner TLS with pinned keys, HTTP/1.1), retries, egress rules, input limits and fuzz tests. Built against the SDK's test server ([07 SDK, "Testing support"](./xxx-notes-011-v5-impl-07-provider-sdk.md)).
6. The `ProviderEndpointRegistration` CRD; Registration, the first Control Call and the `ProviderEndpoint`s writer, with drift repair, server-key updates, Base URL changes, `caBundle` and `pendingSecretHash`; rebuilding a lost `ProviderEndpointRegistration`.
7. Abandoning a Connection.
8. Health: polling, the `Connected` condition, Edge diagnosis, the `unusedExpiresAt` warning, `TrustOutdated`.
9. Secret Changes (on demand, repair, after tampering, distrusted CA pinned), one at a time, with coalescing, rate limits and the comment on periodic rotation.
10. The Connector CA Set, its pushes with compare-and-swap, distrust, the tamper response, convergence, and the `ConnectorCASet` status.
11. The remote finalizer.
12. Metrics, events, docs.
13. Tests: unit for each step and for `phase`, the CEL rules against kcp, kind e2e for a Remote Provider from Onboarding to deletion ([10 Testing](./xxx-notes-011-v5-impl-10-testing.md)).

## Decided

- **`platform-mesh-operator` owns each `Provider`**, local and remote, end to end, with its onboarding requests reconciled under the `Provider`'s key.
- **One active replica** by leader election.
- **Request authorization**: no creator checks; RBAC on requests is granted with managing `Provider`s.
- **Concurrent requests**: the oldest wins; newer ones fail.
- **Requests end at Registration**: `Pending → Registering → Registered`, or `Failed` only before Registration succeeds.
- **`phase` for both kinds**: `Pending → Provisioning → Ready ⇄ Degraded`, and `Deleting`.
- **Local fields are rejected on Remote Providers** by CEL.
- **`Secret` scope**: Connection Secrets in `platform-mesh-operator`'s namespace; token `Secret`s read by name, with no `Secret` informers in kcp.
- **Requests reference an existing `Provider`**, which the Owner creates; a request for a missing one has `ProviderNotFound`.
- **Registration runs inside Inner TLS**, pinned to the Server Key the token names.
- **Onboarding order**: register, first Control Call (pushing the current Connector CA Set), then the `ProviderEndpoint`; `connectionID`, Base URL, Server Keys and `unusedExpiresAt` persisted together. A rejected Connection Secret on the first call doesn't block the `ProviderEndpoint`.
- **The `ProviderEndpointRegistration` is authoritative**, with a `Secret` holding only the credentials; `ProviderEndpoint`s are written from them and read back only to rebuild a lost `ProviderEndpointRegistration`. Its Base URL may change.
- **The `caBundle`** verifies all outer TLS to a Base URL; it stays on the `Provider`, and rotating it is out-of-band.
- **Remote Providers' admin SA** is `default/provider`.
- **Policies in Provider Workspaces** restrict RBAC subjects and SA-token `Secret`s; their enforcement is confirmed before any access is granted.
- **Abandoning a Connection** is an annotation on the `Provider`; a Base URL change is an administrator's edit.
- **Switching Providers** may hand the export to another vendor; that's the Owner's decision.
- **Secret Changes**: on demand (annotation, rate-limited), after tampering or a distrusted CA pinned, and repair; one at a time.
- **Connector CA**: the Connector CA Set is derived from the CA's `CertificateRequest`s, with nothing stored, and pushed here; each previous CA is dropped per Connection, spread over 6 hours after the 48-hour mark, and not while a Connector's certificate still chains to it; distrust follows its gate; the tamper check only considers unexpired, non-distrusted CAs; only Connections that are reached count for convergence, and unknown ones block it.
- **Egress**: fixed refusals in code, everything else up to the deployment; no proxy from the environment.
- **Connector identity**: one group for all shards' Connectors.
- **The admin SA has cluster-admin** in the Provider Workspace, as in RFC 006; the policies, the Connector's Denylist and the `ProviderEndpoint`'s maximal permission policy narrow it where PM needs to.
- **No migration**: no Remote Providers exist yet.

## Declined

- **A separate operator for Remote Providers' Connections.** Two controllers would act on every remote `Provider`: two status writers, a readiness hint passed between them, a finalizer handshake, and naming functions shared as a contract. One owner per `Provider` avoids all of it; the price is the internet-facing code above.
- **Partitioning the work by kcp shard**, with a Lease per shard and a worker per held shard. The load (a few calls per second for 1,000 Connections, mostly waiting) doesn't need horizontal scale; leader election already gives one writer per object, and the per-shard caches already isolate a stuck shard.
- **Folding `ProviderOnboardingRequest` into `Provider`**, with a token reference in `spec.remote`. Onboarding is a one-off event spending a single-use token; in a spec, the reconciler would have to rebuild that event by remembering the last token used, and notice new tokens in `Secret`s it doesn't watch. The states wouldn't get fewer, only move.
- **The request creating the `Provider`.** Before Registration, it needs a copy of `spec.remote` in the request, and a rule for when the copy and an existing `Provider` disagree. After Registration, it would also need registering without a `clusterName` (the workspace doesn't exist yet) and handling a spent token with no `Provider` to show for it. A `Provider` left behind by a failed Onboarding is harmless: it waits, and the next request uses it.
- **Inlining the token in the request** instead of referencing a `Secret`. An unspent token in a custom resource reaches more readers and more copies (`last-applied-configuration`, `kubectl get -o yaml`, audit bodies) than a `Secret`, and it couldn't come through GitOps or a secret store, which keep the request in Git and supply the token later.
- **Removing the spent token from the Owner's `Secret`**, or deleting the `Secret` and the request after Registration. A spent token is useless: `provider-gateway` refuses it except for PM's own retries of the same Registration, which must match PM's idempotency key, the Connection Secret's hash and the `clusterName`, and get back nothing secret. Writing to the Owner's objects would fight GitOps and secret stores, which re-create them, and the request is the record of which token produced which `connectionID`. The cost is that secret scanners still flag a spent `pm1.` token in Git or backups. Cleaning up completed requests after an Owner-set TTL is a possible follow-up.
- **A `replaceConnection` field** for switching Providers. Abandoning the Connection, then a normal request, does the same with less state.
- **Periodic Connection Secret rotation.** A stolen Connection Secret is used at once or not at all: a pushed bundle is detected and undone if PM sees it first, and a lock-out of the Connection is recovered out-of-band. Rotation wouldn't prevent either; it would only shorten how long the holder could read status. If compliance requires it, it's a small addition to the existing Secret Change ([follow-ups](./xxx-notes-011-v5-impl-todo.md)).
