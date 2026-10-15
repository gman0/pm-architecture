# v5 implementation plan: `platform-mesh-operator`

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). `platform-mesh-operator` (`platform-mesh/operators/platform-mesh-operator`) owns every `Provider`, local or remote, from creation to deletion. For Remote Providers that includes `ProviderOnboardingRequest`s, the Provider Workspace, Registration, `ProviderEndpoint`s, health, Secret Changes, the Trusted Set and offboarding. None of it is on the Session path.

## Responsibilities

**Owns:**
- reconciling `Provider`s, local and remote, and `ProviderOnboardingRequest`s;
- Provider Workspaces and everything PM creates in them: the admin SA, the Connection Export binding, the Connector RBAC, and for Local Providers the scoped kubeconfig;
- `ProviderEndpointRegistration`s and their `Secret`s, in its namespace in PM's runtime cluster ([02 APIs, "ProviderEndpointRegistration"](./xxx-notes-011-v5-impl-02-apis.md));
- every call to a Provider's Connection API ([01 Protocol](./xxx-notes-011-v5-impl-01-protocol.md));
- `ProviderEndpoint`s in Provider Workspaces ([02 APIs](./xxx-notes-011-v5-impl-02-apis.md));
- the whole `Provider` status: conditions, `phase`, `status.connection`;
- the Trusted Set: keeping it and pushing it to every Connection ([04 Connector CA](./xxx-notes-011-v5-impl-04-connector-ca.md)).

**Doesn't own:**
- Sessions, proxying or SA tokens ([05 Connector](./xxx-notes-011-v5-impl-05-connector.md));
- issuing Connector Certificates ([04 Connector CA](./xxx-notes-011-v5-impl-04-connector-ca.md)).

## What it consumes

- **The Providers Export's endpoint slice**: `ProviderOnboardingRequest`s and `Provider`s in Owners' workspaces, and token `Secret`s there, read by name and uncached through the export's `Secret`s claim (see "`Secret`s").
- **The Connection Export's endpoint slice** (`:root:providers:system`): it writes `ProviderEndpoint`s in Provider Workspaces, and sees whether a Provider Workspace's binding works.
- **`Workspace` objects in `:root:providers`**, for Provider Workspaces' logical clusters.
- **The Connector CA's `CertificateRequest`s** in cert-manager's namespace, for the CA's public certificates, and the **`ConnectorCABundle`**, whose status it writes ([04 Connector CA](./xxx-notes-011-v5-impl-04-connector-ca.md)).
- **Outbound HTTPS to Providers' Base URLs.**

## Principles

- **One owner per `Provider`**: one reconciler writes its whole status and runs one finalizer.
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
| 3 | | `ConnectorAccess`: the Connector RBAC (below) |
| 4 | | `Registered`: waits for a Registration, makes the first Control Call, writes the `ProviderEndpoint` (see "First Control Call and ProviderEndpoint") |
| 5 | | `Connected`: health on every poll, plus the Connection's upkeep (see "Health") |

The Provider Workspace is `:root:providers:<Provider name>-<the Provider's kcp.io/cluster annotation>`. The annotation is set by kcp, not the Owner. Its logical cluster name (`clusterName`) is read from the `Workspace` object in `:root:providers`, which is PM-owned; nothing about the workspace is taken from the `Provider`'s spec or status.

### Connection Export binding

- **Created with a fixed name** in the Provider Workspace, referencing the Connection Export in `:root:providers:system` and accepting its one permission claim (read-only `apiexportendpointslices`) ([02 APIs, "The binding in Provider Workspaces"](./xxx-notes-011-v5-impl-02-apis.md)).
- **Recreated if deleted.** The Provider can't delete it through a Session ([05 Connector, "Authorization and policy"](./xxx-notes-011-v5-impl-05-connector.md)), but a PM Administrator's error could.
- **Bind permission**: `platform-mesh-operator` needs `bind` on the Connection Export.

### Connector RBAC

In each Remote Provider's workspace, a role and binding for the **Connectors' identity** (one group shared by all shards' Connectors):
- `access` on `/` (workspace content);
- `create` on `serviceaccounts/token` with `resourceNames: [provider]`, in namespace `default`.

Nothing else. Connectors read `ProviderEndpoint`s through the Connection Export, not through this RBAC ([05 Connector, "Identity towards kcp"](./xxx-notes-011-v5-impl-05-connector.md)).

## Spec validation

A CEL rule on `ProviderSpec`: `spec.remote` excludes `providerKubeconfigSecret` and `hostOverride`, which only apply to Local Providers. An Owner who sets both gets an error at apply time. The remote sequence never creates a kubeconfig either way, as defense in depth. Switching an existing `Provider` between local and remote isn't specified.

## Conditions and phase

**Conditions**, for both kinds:
- **one per step, named after it**: `True` when it's done, `False` when it isn't, with a message saying why. Steps after one that isn't done aren't run;
- **`Ready`**: `True` when every step is done. The printer column shows it.

For Remote Providers that's `Workspace`, `ConnectionBinding`, `ConnectorAccess`, `Registered`, `Connected` and `Ready`, plus the warning **`ProtocolOutdated`** (see "Health"), which doesn't affect `Ready`.

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

**`status.connection { url, connectionID }`** on Remote Providers is for the Owner's information: the Base URL and the Connection ID in use, updated when the `Provider` is onboarded again, never used as a link.

## Onboarding requests

The `ProviderOnboardingRequest` reconciler. A request is one action: spend a single-use token to register the `Provider`'s Connection. Everything after Registration is reported on the `Provider`.

1. **Validate the request**: it's the oldest unfinished request for its `Provider` (otherwise fail it, naming the active one); the token `Secret` exists and parses; the Base URL inside passes the egress rules (see "Egress"). The creator isn't checked: `create` on requests is granted together with managing `Provider`s ([02 APIs, "ProviderOnboardingRequest"](./xxx-notes-011-v5-impl-02-apis.md)). Phase `Pending`.
2. **Find the `Provider`** named in `spec.providerRef`. The Owner creates it; the request reconciler never writes `Provider`s.

   | The `Provider`… | The reconciler… |
   |---|---|
   | doesn't exist (yet) | waits, phase `Pending`, with `waiting for Provider <name>` |
   | exists, not registered | uses it; this includes a `Provider` left behind by a failed Onboarding |
   | exists, registered, and the Provider-side knows the Connection | fails the request: a live Connection needs no new Registration, and a lost or rejected Connection Secret is repaired without the Owner (see "Secret Changes") |
   | exists, registered, and the Provider-side reports the Connection unknown or deleted | reuses it and registers a **new** Connection, with the token's Base URL |
   | exists, registered, and the Provider-side doesn't answer | waits: only a definite "unknown" allows a new Registration |
   | exists, not remote (e.g. a ManagedProvider's) | fails the request |

   For a registered `Provider`, the reconciler asks the Provider-side itself (a `GET` with the Connection Secret) instead of reading the `Provider`'s conditions, which the Owner could edit.

   **Onboarding a registered `Provider` again** keeps the Provider Workspace, and with it the APIExport's identity and consumers' bindings; deleting the `Provider` and creating a new one would break every binding for good. It's meant for two cases:
   - **switching Providers**: the old Provider-side deletes the Connection, and a token from the new Provider names its own Base URL;
   - **a Connection the Provider-side deleted and won't restore**.

   **The new Connection has a new `connectionID`.** Provider-side data carries over only where the Provider can find it from kcp objects (an ID in an object's `status`, see [07 SDK, "Requirements on Providers"](./xxx-notes-011-v5-impl-07-provider-sdk.md)); data keyed only by the old `connectionID` is orphaned. When the same Provider still has the data, or can recover it, the preferred path is Provider Administrators importing the old entry, which keeps the `connectionID` ([07 SDK, "Administration"](./xxx-notes-011-v5-impl-07-provider-sdk.md)). The `Registered` condition's `ConnectionUnknown` message says so.

   These rows never create a second Connection next to a live one.
3. **Wait for the Provider Workspace's `clusterName`.** Phase `Pending`.
4. **Register** (see "Registration"). Phase `Registering`.
5. **Delete the token `Secret`'s content**, only after the Registration's result is persisted. The `Secret` object stays, since the Owner may manage it through GitOps; only its token data goes.
6. **Done**: phase `Registered`, with the `connectionID` in status, and a message pointing to the `Provider`. A request never changes again once `Registered` or `Failed`; Onboarding again needs a new request.

`Failed` is only possible before Registration succeeds: the token refused, validation, or another active request. After Registration, the `Provider` waits for the first Control Call, the `ProviderEndpoint`, Sessions and kcp for as long as it takes.

## Registration

1. **Create the `ProviderEndpointRegistration`** for the Provider Workspace, if it doesn't exist, and **generate the Connection Secret** into its `Secret` (write before send). When a registered `Provider` is onboarded again, this replaces the old Connection Secret, and step 3 replaces its spec.
2. **Call `POST /v1/connections`** with the token, the request's UID as idempotency key, the `clusterName`, the front-proxy URL, the Connection Secret's hash and the current Trusted Set ([01 Protocol, "Registration"](./xxx-notes-011-v5-impl-01-protocol.md)).
3. **On success**, persist the `connectionID`, **the Base URL from the token and the returned Server Keys** in the `ProviderEndpointRegistration`'s spec, in one write; then record the `connectionID` in the request's status. The token's data is deleted afterwards, so the `ProviderEndpointRegistration` is the only place the Base URL survives until the `ProviderEndpoint` is written. The Server Keys are trust anchors for the Control Session and the `ProviderEndpoint`, so they're never kept in Owner-writable status, and never only in memory.
4. **On a lost response, or a crash before step 3**, retry with the same idempotency key and hash; the same `connectionID` and Server Keys come back. The Provider-side honours such retries for 7 days by default ([01 Protocol, "Registration"](./xxx-notes-011-v5-impl-01-protocol.md)), far longer than any expected outage.
5. **On refusal** (token spent elsewhere, expired, wrong Owner, or a retry after the window), fail the request with a reason the Owner can act on. A retry refused after the window leaves an orphaned Connection on the Provider-side, which the SDK expires because it never saw a Control Call or Session ([07 SDK, "Connection API"](./xxx-notes-011-v5-impl-07-provider-sdk.md)).

Every step resumes from persisted state: the `Secret` holds the Connection Secret, the `ProviderEndpointRegistration` the `connectionID`, Base URL and Server Keys; the request's status holds the phase.

## First Control Call and ProviderEndpoint

The `Registered` step of a remote `Provider`:
1. **Wait for a Registration**: until the `ProviderEndpointRegistration` holds a `connectionID`, Base URL and Server Keys, `Registered=False` with `NotRegistered: waiting for an onboarding request`.
2. **Make a first Control Call** (`GET` over a Control Session). The `ProviderEndpoint` isn't written until it succeeds, so a control path that's broken from the start shows up here, with its edge-diagnosis label (see "Edge diagnosis"), before any Session opens. Every live Connection has therefore had a working control path at least once.
3. **Wait for the binding to work**: the Provider Workspace must appear in the Connection Export's endpoint slice before its `ProviderEndpoint` can be written. That's kcp's own signal that the binding is in effect.
4. **Write the `ProviderEndpoint`** (see "Writing ProviderEndpoints").
5. **Report the Provider-side's view**: `ConnectionUnknown` (it no longer knows the Connection; the Owner can onboard the `Provider` again, or the Provider can import the old entry).

## Writing ProviderEndpoints

One `ProviderEndpoint` per Provider Workspace, written through the Connection Export ([02 APIs, "ProviderEndpoint"](./xxx-notes-011-v5-impl-02-apis.md)):
- **Written from the `ProviderEndpointRegistration`**, which is authoritative ([02 APIs, "ProviderEndpointRegistration"](./xxx-notes-011-v5-impl-02-apis.md)): `connectionID`, Base URL and Server Keys are copied; `pendingSecretHash` is the hash of its `Secret`'s `pendingSecret` while the Connection Secret changes; `caBundle` is copied from the `Provider`'s `spec.remote.caBundle`, so an Owner's update reaches Connectors' new Sessions. Every change goes into the `ProviderEndpointRegistration` or its `Secret` first; the `ProviderEndpoint` follows.
- **Drift repair**: if the `ProviderEndpoint` is missing (e.g. the binding was recreated) or differs, it's rewritten from them. Connectors react to the change ([05 Connector, "Discovery"](./xxx-notes-011-v5-impl-05-connector.md)).
- **Read as input only when the `ProviderEndpointRegistration` is lost**: it's rebuilt from the `ProviderEndpoint` (`connectionID`, Base URL, Server Keys), and the Connection Secret is repaired, since the `Secret` goes with it (see "Secret Changes").
- **Server Key updates**: when `GET` shows a new Server Key, it's added to the `ProviderEndpointRegistration` and then the `ProviderEndpoint`; the Provider-side presents it on a Connection only once that Connection's `ProviderEndpoint` lists it. Retired keys are removed once `GET` no longer lists them ([01 Protocol, "Server Keys"](./xxx-notes-011-v5-impl-01-protocol.md)).
- **Administrator input for recovery**: a PM Administrator edits a Connection's Base URL or Server Keys in its `ProviderEndpointRegistration`, for example after a Provider rebuilt elsewhere lost its keys. The input lives in PM's runtime cluster, never in an Owner-writable field; the `ProviderEndpoint` follows. Server Keys are never fetched over plain HTTPS. Runbooks for it are a [follow-up](./xxx-notes-011-v5-impl-todo.md).
- **Deleted first** when the `Provider` is deleted (see "Deletion").

## Health

The `Connected` step runs on every reconcile of a registered remote `Provider` and requeues at the poll interval (every few minutes per Connection, with jitter):
- **Polls `GET /v1/connections/{id}`** over a Control Session, reusing an open one where possible.
- **Never gives up on a Connection it can't reach.** Failed polls back off, capped at about the normal poll interval, so a Provider that returns after any outage is caught up within minutes (see "Trusted Set").
- **Reports the Connection's state** as the `Connected` condition, from the first failing check:
  1. **Control path**: the edge-diagnosis labels (see "Edge diagnosis"), or `ControlUnreachable` (DNS, connection refused, timeout);
  2. **Sessions**: `NoSessions`, when a host the Provider-side needs has none;
  3. **kcp**: `KcpUnreachable`, when a host's requests don't reach kcp.

  The message adds counts and the last error, never shard names (overview, principle 9). If the control path fails, the status can't be read, so the control-path label wins and the message mentions the last known Session state.
- **Sets the warning condition `ProtocolOutdated`** while its Control Sessions still negotiate the previous protocol version through ALPN, so the Owner can ask the Provider to upgrade its SDK ([07 SDK, "Releases and protocol versions"](./xxx-notes-011-v5-impl-07-provider-sdk.md)).
- **Does the Connection's upkeep** on the same polls: Server Key updates in the `ProviderEndpoint`, Secret Changes (see "Secret Changes"), and trusted-set pushes and the tamper check (see "Trusted Set").
- **Per-shard and per-host detail goes to metrics**, never to conditions (overview, principle 9).

**Readiness for Onboarding**: at least the front-proxy host is covered and reachable; before any consumer binds, that's the only required host.

### Edge diagnosis

On the first Control Session after Registration and on every failed reconnect, Control Session failures are classified as described in [01 Protocol, "Diagnosing the Edge"](./xxx-notes-011-v5-impl-01-protocol.md). The label starts the message of `Registered` (during the first Control Call) or `Connected` (afterwards):
- `EdgeUpgradeRefused`: a non-`101` response;
- `EdgeBotChallenge`: an HTML page instead of an upgrade;
- `EdgePayloadBlocked`: closed right after a binary message;
- `EdgeTimeout`: closed at a regular idle time or age;
- `EdgeTLS`: the server certificate isn't trusted by web PKI or the `Provider`'s `caBundle`, typically a private CA the Provider rotated before the Owner updated `caBundle` ([02 APIs, "Provider"](./xxx-notes-011-v5-impl-02-apis.md)).

Each message points the Owner at the Provider's Edge checklist ([08 Quickstart, "Connection API server"](./xxx-notes-011-v5-impl-08-provider-quickstart.md)). Only Control Sessions are observed here; Connectors' Session failures go to their metrics, where the same classification applies.

## Secret Changes

The only way a Connection Secret changes ([01 Protocol, "Changing the Connection Secret"](./xxx-notes-011-v5-impl-01-protocol.md)):
1. Generate a new Connection Secret; write it as `pendingSecret` in the `ProviderEndpointRegistration`'s `Secret`.
2. Write its hash into the `ProviderEndpoint`'s `pendingSecretHash`.
3. Wait for the Provider-side to adopt it, which it does when it reads the `ProviderEndpoint` over a Session: authenticate with `pendingSecret` (a `GET`) until it succeeds.
4. Move `pendingSecret` to `secret`, then clear `pendingSecretHash`.

**Triggers:**
- **On demand**: the annotation `providers.platform-mesh.io/rotate-connection-secret` on the `Provider`, set by the Owner or by PM Administrators, e.g. `kubectl annotate provider my-provider providers.platform-mesh.io/rotate-connection-secret="$(date -u +%FT%TZ)" --overwrite`. Any new value triggers one change; the value last acted on is kept in the `ProviderEndpointRegistration` (`rotationRequest`), so re-applying the same value (e.g. through GitOps) does nothing. Trusting it is harmless: it only makes PM change a Connection Secret that PM controls, and it's never used as a link.
- **After tampering** with a Pinned Bundle (see "Trusted Set").
- **Repair**: when the `Secret` is missing, or the Provider-side rejects both `secret` and `pendingSecret` while the Inner TLS against its pinned Server Key succeeds (e.g. its store was restored from an older backup). A missing `ProviderEndpointRegistration` is first rebuilt from the `ProviderEndpoint` (`connectionID`, Base URL, Server Keys). Repairs are rate-limited per Connection (e.g. at most one per hour), so an Edge or Provider bug that breaks authentication can't cause a loop of changes.

**The code carries a comment at the list of triggers** about periodic rotation: what it would need (a last-change timestamp, a configured interval, due times spread by `connectionID`), why it might be wanted (compliance), and why it isn't needed (see "Declined").

**Properties:**
- **Sessions are unaffected**: they never use the Connection Secret. A change needs one, though; without Sessions it waits, and the Provider-side keeps its current hash.
- **On restart mid-change**, try `pendingSecret` first, then `secret`, and resume; the `ProviderEndpoint` still names the pending hash.
- **Nobody else can change the Connection Secret**: not someone holding it, an Onboarding Token or a portal account. There's no lock-out path.

## Recovery

Each case is covered elsewhere ([01 Protocol, "Recovery"](./xxx-notes-011-v5-impl-01-protocol.md)):
- **A lost or rejected Connection Secret**: repaired automatically (see "Secret Changes").
- **A Connection the Provider-side no longer knows**: `Registered=False` with `ConnectionUnknown`; the Owner onboards the same `Provider` again, which registers a new Connection (see "Onboarding requests").
- **No Session verifies**: `Connected=False` with `NoSessions`; Provider Administrators re-pin the Pinned Bundle out-of-band.
- **Disasters on either side**: out-of-band, using the administrator input in "Writing ProviderEndpoints" and the Provider Administrators' runbooks ([07 SDK, "Administration"](./xxx-notes-011-v5-impl-07-provider-sdk.md)).

## Trusted Set

cert-manager renews the Connector CA; `platform-mesh-operator` makes sure every Provider trusts the new CA before Connectors use it ([04 Connector CA, "Rollover"](./xxx-notes-011-v5-impl-04-connector-ca.md)):
- **Derived, not stored**: the CA's `CertificateRequest`s, which cert-manager creates per issuance and keeps the last three of, give the current CA, the **Trusted Set**, each Connection's drop time for the previous CA, and PM's unexpired CAs ([04 Connector CA, "The Trusted Set"](./xxx-notes-011-v5-impl-04-connector-ca.md)). `platform-mesh-operator` watches them and reads only their status, which holds public certificates. A restart or failover derives the same result.
- **Why `CertificateRequest`s**: they're the only place the CA certificates of past issuances are kept without the key. The CA's `Secret` holds the key and only the current certificate, and the CA `Certificate` object holds neither.
- **Push on difference**: on each poll, the Connection's pinned fingerprints (from `GET`) are compared with the bundle this Connection should pin (the Trusted Set's certificates as PEM, without the previous CA once the Connection's drop time has passed), and `PUT /connector-ca-bundle` is sent with `If-Match: <the etag just read>` where they differ, rate-limited ([01 Protocol, "Pinned Bundle"](./xxx-notes-011-v5-impl-01-protocol.md)). A `412` means the bundle changed since: read it again and decide anew. When a new CA appears, every Connection is requeued, so pushes go out within minutes; drops happen on the regular polls after each Connection's drop time. Registration always sends the current set.
- **Catch up as soon as a Connection is back.** A Provider that was unreachable during a Rollover, however long, still pins an old set; its control channel doesn't depend on the Connector CA, so the **first successful Control Call after a Connection was unreachable** pushes the current set right away. The SDK accepts the push even if every CA it pins has expired.
- **Tampering**: a pinned certificate that's unexpired and isn't one of PM's unexpired CAs can only have come from someone holding the Connection Secret. Then:
  1. restore the Trusted Set with the current Connection Secret (with `If-Match`), so PM's Sessions verify again; the Provider-side closes every Session that doesn't chain to it, including any the thief opened ([01 Protocol, "Pinned Bundle"](./xxx-notes-011-v5-impl-01-protocol.md));
  2. change the Connection Secret through the `ProviderEndpoint` (see "Secret Changes"), which makes the stolen Connection Secret useless;
  3. record an **event** on the `Provider`, so the Owner sees it happened, and count it in a metric with a critical alert rule for the PM Administrators ([04 Connector CA, "Signals"](./xxx-notes-011-v5-impl-04-connector-ca.md)).

  If the thief pushes again before the change completes, step 1 repeats; the change completes on the Provider-side's first read of the `ProviderEndpoint` over a Session. Expired pinned CAs verify nothing, so they're never tampering, whoever issued them; neither is a bundle merely missing some of PM's CAs. The push above converges both.
- **Convergence**: a CA has converged when every Connection whose Control Calls succeed pins it. A Connection whose Control Calls fail counts as **not reached**, whatever the cause, and is listed with its last error; Owners see the cause on their `Provider`. Nothing waits for Connections that aren't reached. `platform-mesh-operator` writes the whole `ConnectorCABundle` status: the trusted CAs with their `converged` flags, counts per shard, and the conditions `Valid` and `Converged`. Connectors read the flags to decide when to switch certificates ([05 Connector, "Connector Certificate"](./xxx-notes-011-v5-impl-05-connector.md)).

## Deletion

One finalizer per `Provider`. For Remote Providers, in order:
1. **Delete the `ProviderEndpoint`**, so every Connector closes its Sessions. kcp access ends here.
2. **Call `DELETE`** in a Control Session, with bounded retries; an unreachable Provider-side doesn't block deletion beyond a timeout.
3. **Delete the `ProviderEndpointRegistration`**; its `Secret` follows by owner reference.
4. **Delete the Connector RBAC, then the binding**, once nothing uses them anymore.
5. **Delete the SA and the workspace.**

For Local Providers: the SA-token `Secret`, the kubeconfig in the Owner's workspace, the SA and the workspace.

## Egress

- **Base URL rules**, checked before Registration, before writing a `ProviderEndpoint` and on every dial: `https` only, public addresses only, resolve once and dial the checked addresses, no redirects to other hosts.
- **TLS** to the Base URL, for Registration and the outer TLS of Control Sessions: web PKI, or the `Provider`'s `spec.remote.caBundle` when set. Connectors verify their Sessions the same way, from the `ProviderEndpoint`'s copy.
- **Per-Provider limits**: concurrent calls, rate, backoff; jitter on periodic work. With 1,000 Connections and a 5-minute poll, steady state is about 3–4 calls per second in total.

## `Secret`s

Two kinds, in two places:
- **Connection Secrets live in the `ProviderEndpointRegistration`s' `Secret`s**, in `platform-mesh-operator`'s namespace in PM's runtime cluster, one per Connection ([02 APIs, "Its `Secret`"](./xxx-notes-011-v5-impl-02-apis.md)). Nothing in kcp holds them, so neither Owners nor any Provider with a `Secret`s claim on Owners' workspaces can read them. They may be cached with an ordinary informer on that namespace.
- **Token `Secret`s stay in Owners' workspaces**, where Owners create them. The Providers Export's `Secret`s claim is accepted with `matchAll` in PM's bindings, so `platform-mesh-operator` *could* read every `Secret` there. It doesn't cache any of them: **no `Secret` informers in kcp.** It reads the token `Secret` named in the request's spec by name, with an uncached client, and later deletes its token data. Nothing about other `Secret`s in Owners' workspaces enters its memory. A token `Secret` created after its request is picked up by requeueing with backoff.
- **Rights in kcp**: the identity can reach any `Secret` the claim allows. Narrowing the claim with a label selector in PM's bindings is a PM-wide hardening outside this plan.

## Internet-facing code

The code for Remote Providers makes outbound calls to Providers on the internet and parses their responses, inside PM's most privileged component. That's accepted while the `Provider` part of `platform-mesh-operator` is still part of it; moving it into its own operator is on the [follow-ups list](./xxx-notes-011-v5-impl-todo.md). Meanwhile:
- **The Connection code and the protocol client sit in their own packages**, depending on the rest of `platform-mesh-operator` only where they plug into the `Provider` reconciler, so they can move as they are.
- **Everything parsed from Providers has size limits and fuzz tests**: Registration responses, Control Call responses, bundles, status.

## Identity and permissions

Beyond what `platform-mesh-operator` needs for Local Providers:
- **In kcp**: `apiexports/content` on the Connection Export, including writes to `providerendpoints`, granted by the export's maximal permission policy ([02 APIs, "Connection Export"](./xxx-notes-011-v5-impl-02-apis.md)); `bind` on the Connection Export; the rights to create the Connector RBAC in Provider Workspaces.
- **In the runtime cluster**: `providerendpointregistrations` and `Secret`s in its own namespace; a Role in cert-manager's cluster resource namespace allowing only `get`, `list` and `watch` on `certificaterequests`, whose status holds public certificates; write access to the `ConnectorCABundle`'s status. No access to the CA's key `Secret` or issuers, which only cert-manager reads.
- **Network**: outbound HTTPS to the internet, for Providers' Base URLs.

## Replicas and availability

- **One active replica**, by leader election, with standbys that take over within seconds. It runs one multicluster-runtime provider over all shards of the Providers Export and one over the Connection Export; each shard's endpoint gets its own caches, so a shard whose virtual workspace is down only stalls its own objects. The work is mostly waiting on Providers, so it scales with reconcile concurrency, not replicas.
- **After a failover**, the new leader rebuilds its caches and reconciles every Connection once, within the per-Provider rate limits.
- **Outages delay** Onboarding, Secret Changes, Rollovers and offboarding; they don't stop traffic. The one exception: an outage of more than about 12 hours right when cert-manager renews the Connector CA. The new CA then never converges, Connectors fall back to it as their old certificates expire, and new Sessions fail at Providers that don't pin it, until `platform-mesh-operator` is back and pushes it. Open Sessions keep running.

## Observability

- **Events** on `ProviderOnboardingRequest`s and `Provider`s for every phase change and failure, and for tampering, without secrets.
- **Metrics**: calls per Provider and outcome, poll latency, Connections per condition state, Secret Changes in progress (by trigger), repairs refused by the rate limit, tampering detected, egress refusals, and the Connector CA signals (CA expiry, `Valid`, convergence; [04 Connector CA, "Signals"](./xxx-notes-011-v5-impl-04-connector-ca.md)). Scraped through `platform-mesh-operator`'s `ServiceMonitor` ([09 Deployment, "Observability"](./xxx-notes-011-v5-impl-09-deployment.md)).
- **Audiences**: Owners act on events and conditions on their `ProviderOnboardingRequest`s and `Provider`s; the PM Administrators act on metrics, the `ConnectorCABundle` status and the documented alert rules. Owners never see PM-wide signals, and PM-wide signals never name Owners' data beyond Connection IDs.
- **Logs** never contain the token, the Connection Secret or hashes.

## Work breakdown

1. The `Provider` reconciler's remote sequence, next to the local one; `phase` computed from the conditions for both kinds, with `status.lastReadyTime`.
2. The CEL rule in `platform-mesh/apis`.
3. The remote bootstrap steps: workspace, Connection Export binding, Connector RBAC.
4. The `ProviderOnboardingRequest` reconciler and its phases.
5. The protocol client, in its own package: plain-HTTPS calls, Control Sessions (WebSocket, Inner TLS with pinned keys, HTTP/1.1), retries, egress rules, input limits and fuzz tests. Built against the SDK's test server ([07 SDK, "Testing support"](./xxx-notes-011-v5-impl-07-provider-sdk.md)).
6. The `ProviderEndpointRegistration` CRD; Registration, the first Control Call and the `ProviderEndpoint`s writer, with drift repair, server-key updates, `caBundle` and `pendingSecretHash`; rebuilding a lost `ProviderEndpointRegistration`.
7. Health: polling, the `Connected` condition, Edge diagnosis.
8. Secret Changes (on demand, repair, after tampering), with the comment on periodic rotation.
9. The Trusted Set, its pushes with compare-and-swap, the tamper response, and the `ConnectorCABundle` status.
10. The remote finalizer.
11. Metrics, events, docs.
12. Tests: unit for each step and for `phase`, the CEL rule against kcp, kind e2e for a Remote Provider from Onboarding to deletion ([10 Testing](./xxx-notes-011-v5-impl-10-testing.md)).

## Decided

- **`platform-mesh-operator` owns each `Provider`**, local and remote, end to end.
- **One active replica** by leader election.
- **Request authorization**: no creator checks; RBAC on requests is granted with managing `Provider`s.
- **Concurrent requests**: the oldest wins; newer ones fail.
- **Requests end at Registration**: `Pending → Registering → Registered`, or `Failed` only before Registration succeeds.
- **`phase` for both kinds**: `Pending → Provisioning → Ready ⇄ Degraded`, and `Deleting`.
- **Local fields are rejected on Remote Providers** by CEL.
- **`Secret` scope**: Connection Secrets in `platform-mesh-operator`'s namespace; token `Secret`s read by name, with no `Secret` informers in kcp.
- **Requests reference an existing `Provider`**, which the Owner creates; a request for a missing one waits.
- **Onboarding order**: register, first Control Call, then the `ProviderEndpoint`; `connectionID`, Base URL and Server Keys persisted together.
- **The `ProviderEndpointRegistration` is authoritative**, with a `Secret` holding only the credentials; `ProviderEndpoint`s are written from them and read back only to rebuild a lost `ProviderEndpointRegistration`.
- **The `caBundle`** verifies all outer TLS to a Base URL; it stays on the `Provider`, and rotating it is out-of-band.
- **Remote Providers' admin SA** is `default/provider`.
- **Secret Changes**: on demand (annotation), after tampering, and repair.
- **Connector CA**: the Trusted Set is derived from the CA's `CertificateRequest`s, with nothing stored, and pushed here; the previous CA is dropped per Connection, spread over 6 hours after the 48-hour mark; the tamper check only considers unexpired CAs; only Connections that are reached count for convergence.
- **Connector identity**: one group for all shards' Connectors.
- **The admin SA has cluster-admin** in the Provider Workspace, as in RFC 006; the Connector's Denylist and the `ProviderEndpoint`'s maximal permission policy narrow it where PM needs to.
- **No migration**: no Remote Providers exist yet.

## Declined

- **A separate operator for Remote Providers' Connections.** Two controllers would act on every remote `Provider`: two status writers, a readiness hint passed between them, a finalizer handshake, and naming functions shared as a contract. One owner per `Provider` avoids all of it; the price is the internet-facing code above.
- **Partitioning the work by kcp shard**, with a Lease per shard and a worker per held shard. The load (a few calls per second for 1,000 Connections, mostly waiting) doesn't need horizontal scale; leader election already gives one writer per object, and the per-shard caches already isolate a stuck shard.
- **Folding `ProviderOnboardingRequest` into `Provider`**, with a token reference in `spec.remote`. Onboarding is a one-off event spending a single-use token; in a spec, the reconciler would have to rebuild that event by remembering the last token used, and notice new tokens in `Secret`s it doesn't watch. The states wouldn't get fewer, only move.
- **The request creating the `Provider`.** Before Registration, it needs a copy of `spec.remote` in the request, and a rule for when the copy and an existing `Provider` disagree. After Registration, it would also need registering without a `clusterName` (the workspace doesn't exist yet) and handling a spent token with no `Provider` to show for it. A `Provider` left behind by a failed Onboarding is harmless: it waits, and the next request uses it.
- **Inlining the token in the request** instead of referencing a `Secret`. An unspent token in a custom resource reaches more readers and more copies (`last-applied-configuration`, `kubectl get -o yaml`, audit bodies) than a `Secret`, and it couldn't come through GitOps or a secret store, which keep the request in Git and supply the token later.
- **Periodic Connection Secret rotation.** Everything harmful someone could do with a stolen Connection Secret is either detected and undone (pushing a bundle) or impossible (changing the Connection Secret), so rotation would only shorten how long they could read status. If compliance requires it, it's a small addition to the existing Secret Change ([follow-ups](./xxx-notes-011-v5-impl-todo.md)).
