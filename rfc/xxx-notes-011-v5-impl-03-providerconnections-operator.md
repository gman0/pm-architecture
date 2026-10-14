# v5 implementation plan: connection handling in platform-mesh-operator

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). The **control plane** of remote Providers lives in **platform-mesh-operator**, next to its Provider controller: onboarding requests, registration, records, health, secret changes, the connector CA trusted set and offboarding. The Provider controller's lifecycle, its subroutines, conditions and `phase` are in [06 Provider controller](./xxx-notes-011-v5-impl-06-provider-controller.md); this document covers what the connection-specific parts do. None of it is on the session path.

The `Provider` part of platform-mesh-operator is expected to move into its own operator later, taking all of this with it (see "Interim: internet-facing code in platform-mesh-operator").

## Responsibilities

**Owns:**
- reconciling `ProviderOnboardingRequest`s;
- the connection side of remote `Provider`s, through the `Registered` and `Connected` subroutines ([06 Provider controller, "Subroutines"](./xxx-notes-011-v5-impl-06-provider-controller.md));
- connection Secrets, in platform-mesh-operator's namespace in PM's runtime cluster;
- every call to a Provider's connection API ([01 Protocol](./xxx-notes-011-v5-impl-01-protocol.md));
- `ProviderEndpoint` records in provider workspaces ([02 APIs](./xxx-notes-011-v5-impl-02-apis.md));
- the connector CA trusted set: keeping it and pushing it to every connection ([04 Connector CA](./xxx-notes-011-v5-impl-04-connector-ca.md)).

**Doesn't own:**
- sessions, proxying or SA tokens ([05 Connector](./xxx-notes-011-v5-impl-05-connector.md));
- issuing connector certificates ([04 Connector CA](./xxx-notes-011-v5-impl-04-connector-ca.md)).

## What it consumes

- **The providers export's endpoint slice**, as platform-mesh-operator already does: `ProviderOnboardingRequest`s and `Provider`s, and token Secrets in owners' workspaces, read by name and uncached through the export's existing Secrets claim (see "Secrets").
- **The connection export's endpoint slice** (`:root:providers:system`): it writes `ProviderEndpoint`s in provider workspaces, and sees whether a provider workspace's binding works.
- **`Workspace` objects in `:root:providers`**, to find a provider workspace's logical cluster.
- **The connector CA's `CertificateRequest`s** in cert-manager's namespace, for the CA's public certificates, and the **`ConnectorCABundle`**, whose status it writes ([04 Connector CA](./xxx-notes-011-v5-impl-04-connector-ca.md)).
- **Outbound HTTPS to Providers' base URLs.**

## Principles

- **Write before send** (overview, principle 7): every new secret is in the connection Secret before its hash is sent or written into the record.
- **The Provider-side is the source of truth** for what it has pinned; it's read back with `GET` and converged.
- **Only PM changes a connection's secret**, through the record (see "Secret changes"); no call does.
- **Never trust owner-writable fields for links** (overview, principle 4).
- **Bounded, polite egress**: per-Provider rate limits, jitter and backoff; Providers are external services.
- **Aggregate status only** towards owners.

## Interim: internet-facing code in platform-mesh-operator

The connection handling makes outbound calls to Providers on the internet and parses their responses. Until the `Provider` part moves into its own operator, that code runs in platform-mesh-operator, PM's most privileged component. This is an **accepted interim risk**; the split is on the [follow-ups list](./xxx-notes-011-v5-impl-todo.md). Meanwhile:
- **The connection logic and the protocol client sit in their own packages**, with no dependency on the rest of platform-mesh-operator beyond the Provider controller's lifecycle, so the split moves them as they are.
- **Everything parsed from Providers has size limits and fuzz tests**: registration responses, control-call responses, bundles, status.

## Secrets

Two kinds, in two places:
- **Connection Secrets live in platform-mesh-operator's namespace** in PM's runtime cluster, one per connection ([02 APIs, "The connection Secret"](./xxx-notes-011-v5-impl-02-apis.md)). Nothing in kcp holds them, so neither owners nor any Provider with a Secrets claim on owners' workspaces can read them. They may be cached with an ordinary informer on that namespace.
- **Token Secrets stay in owners' workspaces**, where owners create them. The providers export's Secrets claim is accepted with `matchAll` in PM's bindings, so platform-mesh-operator *could* read every Secret there. It doesn't cache any of them: **no Secret informers in kcp.** It reads the token Secret named in the request's spec by name, with an uncached client, and later deletes its token data. Nothing about other Secrets in owners' workspaces ever enters its memory. A token Secret created after its request is picked up by requeueing with backoff.
- **Rights in kcp are unchanged**: the identity can still reach any Secret the claim allows. Narrowing the claim with a label selector in PM's bindings is a PM-wide hardening outside this plan.

## Finding the provider workspace

The registration needs the provider workspace's `clusterName`, and the record needs to know where to go. Neither is taken from the `Provider`'s spec or status:
1. **Derive the workspace name** the same way the workspace subroutine does: `<Provider name>-<the Provider's kcp.io/cluster annotation>` under `:root:providers`. The annotation is set by kcp, not the owner.
2. **Read the `Workspace` object** in `:root:providers` (PM-owned; owners have no rights there) for its logical cluster name and phase.

## Onboarding requests

A reconciler for `ProviderOnboardingRequest`s. A request is one action: spend a single-use token to register the `Provider`'s connection. Everything after registration is reported on the `Provider` ([06 Provider controller, "Conditions and phase"](./xxx-notes-011-v5-impl-06-provider-controller.md)).

1. **Validate the request**: it's the oldest unfinished request for its `Provider` (otherwise fail it, naming the active one); the token Secret exists and parses; the base URL inside passes the egress rules (see "Egress"); `spec.provider` is a remote Provider. No check of who created the request: `create` on requests is granted together with managing `Provider`s ([02 APIs, "ProviderOnboardingRequest"](./xxx-notes-011-v5-impl-02-apis.md)). Phase `Pending`.
2. **Create or reuse the `Provider`**:

   | The `Provider`… | The reconciler… |
   |---|---|
   | doesn't exist | creates it with `spec.remote` from the request |
   | exists, not registered | reuses it |
   | exists, registered, and the Provider-side knows the connection (frozen included) | fails the request: a live connection needs no new registration, and a lost or rejected secret is repaired without the owner (see "Secret changes") |
   | exists, registered, and the Provider-side reports the connection unknown or deleted | reuses it and registers a **new** connection, with the token's base URL |
   | exists, registered, and the Provider-side doesn't answer | waits: only a definite "unknown" allows a new registration |
   | exists, not remote (e.g. a ManagedProvider's) | fails the request |

   For a registered `Provider`, the reconciler asks the Provider-side itself (a `GET` with the connection's secret) instead of reading the `Provider`'s conditions, which the owner could edit.

   **Onboarding a registered `Provider` again** keeps the provider workspace, and with it the APIExport's identity and consumers' bindings. The alternative an owner could do alone, deleting the `Provider` and creating a new one, breaks every binding for good. It's meant for two cases:
   - **switching Providers**: the old Provider-side deletes the connection, and a token from the new Provider names its own base URL;
   - **a connection the Provider-side deleted and won't restore**.

   **The new connection has a new `connectionID`.** Provider-side data carries over only where the Provider can find it from kcp objects (an ID in an object's `status`, see [07 SDK, "Requirements on Providers"](./xxx-notes-011-v5-impl-07-provider-sdk.md)); data keyed only by the old `connectionID` is orphaned. So when the same Provider still has the data, or can recover it, the preferred path is the Provider's administrators importing the old entry, which keeps the `connectionID` ([07 SDK, "Administration"](./xxx-notes-011-v5-impl-07-provider-sdk.md)). The `Registered` condition's `ConnectionUnknown` message says so.

   These rows never create a second connection next to a live one.
3. **Wait for the provider workspace** (see "Finding the provider workspace"). Phase `Pending`.
4. **Register** (see "Registration"). Phase `Registering`.
5. **Delete the token Secret's content**, only after the registration's result is persisted. The Secret object itself stays, since the owner may manage it through GitOps; only its token data goes ([02 APIs, "ProviderOnboardingRequest"](./xxx-notes-011-v5-impl-02-apis.md)).
6. **Done**: phase `Registered`, with `providerRef` and `connectionID` in status, and a message pointing to the `Provider` for everything after. A request never changes again once `Registered` or `Failed`; onboarding again needs a new request.

`Failed` is only possible before registration succeeds: the token refused, validation, or another active request. After registration, waiting for the first control call, the record, sessions and kcp is the `Provider`'s business, without a timeout.

## Registration

1. **Generate the connection secret** and write it to the connection Secret in platform-mesh-operator's namespace (write before send). When onboarding a registered `Provider` again, this replaces the old connection's Secret.
2. **Call `POST /v1/connections`** with the token, the request's UID as idempotency key, the `clusterName`, the front-proxy URL, the secret's hash and the current trusted set ([01 Protocol, "Registration"](./xxx-notes-011-v5-impl-01-protocol.md)).
3. **On success**, persist the `connectionID` **and the returned server keys** in the connection Secret, in one write; then record the `connectionID` in the request's status. The server keys are trust anchors for the control session and the record, so they're never kept in owner-writable status, and never only in memory.
4. **On a lost response, or a crash before step 3**, retry with the same idempotency key and hash; the same `connectionID` and server keys come back. The Provider-side honours such retries for 7 days by default ([01 Protocol, "Registration"](./xxx-notes-011-v5-impl-01-protocol.md)), far longer than any expected outage.
5. **On refusal** (token spent elsewhere, expired, wrong owner, or a retry after the window), fail the request with a reason the owner can act on. A retry refused after the window leaves an orphaned connection on the Provider-side, which the SDK expires because it never saw a control call or session ([07 SDK, "Connection API"](./xxx-notes-011-v5-impl-07-provider-sdk.md)).

Every step resumes from persisted state: the connection Secret holds the secret, then the `connectionID` and server keys; the request's status holds the phase.

## The `Registered` subroutine

Runs in the Provider controller's lifecycle for remote Providers, after the bootstrap subroutines ([06 Provider controller, "Subroutines"](./xxx-notes-011-v5-impl-06-provider-controller.md)):
1. **Wait for a registration**: until the connection Secret holds a `connectionID` and server keys, `Registered=False` with `NotRegistered: waiting for an onboarding request`.
2. **Make a first control call** (`GET` over a control session). The record isn't written until it succeeds, so a control path that's broken from the start shows up here, with its edge-diagnosis label (see "Edge diagnosis"), before any session opens. Every live connection has therefore had a working control path at least once.
3. **Wait for the binding to work**: the provider workspace must appear in the connection export's endpoint slice before its record can be written. That's kcp's own signal that the binding the `ConnectionBinding` subroutine created is in effect.
4. **Write the record** (see "Records").
5. **Report the Provider-side's view**: `ConnectionUnknown` (it no longer knows the connection; the owner can onboard the `Provider` again, or the Provider can import the old entry) or `ConnectionFrozen`.

Failures are `Stopped` with a labelled message, as described in [06 Provider controller, "Conditions and phase"](./xxx-notes-011-v5-impl-06-provider-controller.md).

## Records

One `ProviderEndpoint` per provider workspace, written through the connection export ([02 APIs, "ProviderEndpoint"](./xxx-notes-011-v5-impl-02-apis.md)):
- **Contents**: `connectionID`, base URL, SA reference, server keys, and `pendingSecretHash` while the secret changes.
- **Drift repair**: if the record is missing (e.g. the binding was recreated) or differs, it's rewritten. Connectors react to the change ([05 Connector, "Discovery"](./xxx-notes-011-v5-impl-05-connector.md)).
- **Server key updates**: when `GET` shows a new server key, it's added to the record; the Provider-side presents it on a connection only once that connection's record lists it. Retired keys are removed once `GET` no longer lists them ([01 Protocol, "Server keys"](./xxx-notes-011-v5-impl-01-protocol.md)).
- **Administrator input for recovery**: a PM administrator can supply a connection's base URL and server keys, for example after a Provider rebuilt elsewhere lost its keys. The input lives in PM's runtime cluster, never in an owner-writable field, and is applied to the record. Server keys are never fetched over plain HTTPS. The tooling is a [follow-up](./xxx-notes-011-v5-impl-todo.md); the record's design already allows it.
- **Deleted first on offboarding** (see "Offboarding").

## The `Connected` subroutine

Runs on every reconcile of a registered remote Provider, after `Registered`, and requeues at the poll interval (every few minutes per connection, with jitter):
- **Polls `GET /v1/connections/{id}`** over a control session, reusing an open one where possible.
- **Never gives up on a connection it can't reach.** Failed polls back off, capped at about the normal poll interval, so a Provider that returns after any outage is caught up within minutes (see "Connector CA trusted set").
- **Reports the connection's state** as the `Connected` condition, from the first failing check:
  1. **Control path**: the edge-diagnosis labels (see "Edge diagnosis"), or `ControlUnreachable` (DNS, connection refused, timeout);
  2. **Sessions**: `NoSessions`, when a host the Provider-side needs has none;
  3. **kcp**: `KcpUnreachable`, when a host's requests don't reach kcp.

  The message adds counts and the last error, never shard names (overview, principle 9). If the control path fails, the status can't be read, so the control-path label wins and the message mentions the last known session state.
- **Writes the warning condition `ProtocolOutdated`** while the connection still negotiates the previous protocol version, so the owner can ask the Provider to upgrade its SDK ([07 SDK, "Releases and protocol versions"](./xxx-notes-011-v5-impl-07-provider-sdk.md)). It's informational and doesn't affect `Ready`.
- **Does the connection's upkeep** on the same polls: server key updates in the record, secret changes (see "Secret changes"), and trusted-set pushes and the tamper check (see "Connector CA trusted set").
- **Per-shard and per-host detail goes to metrics**, never to conditions (overview, principle 9).

**Readiness for onboarding**: at least the front-proxy host is covered and reachable; before any consumer binds, that's the only required host.

### Edge diagnosis

On the first control session after registration and on every failed reconnect, control-session failures are classified as described in [01 Protocol, "Diagnosing the edge"](./xxx-notes-011-v5-impl-01-protocol.md). The label starts the message of `Registered` (during the first control call) or `Connected` (afterwards):
- `EdgeUpgradeRefused`: a non-`101` response;
- `EdgeBotChallenge`: an HTML page instead of an upgrade;
- `EdgePayloadBlocked`: closed right after a binary message;
- `EdgeTimeout`: closed at a regular idle time or age.

Each message points the owner at the Provider's edge checklist and self-test ([08 Quickstart, "Connection server"](./xxx-notes-011-v5-impl-08-provider-quickstart.md)). Labels are stable, for people and tests, not a machine interface: nothing on the PM-side can fix a Provider's edge. Only the control sessions are observed here; connectors' session failures go to their metrics, where the same classification applies.

## Secret changes

The only way a connection's secret changes ([01 Protocol, "Changing the secret"](./xxx-notes-011-v5-impl-01-protocol.md)):
1. Generate a new secret; write it as `pendingSecret` in the connection Secret.
2. Write its hash into the record's `pendingSecretHash`.
3. Wait for the Provider-side to adopt it, which it does when it reads the record over a session: authenticate with `pendingSecret` (a `GET`) until it succeeds.
4. Move `pendingSecret` to `secret`, then clear `pendingSecretHash`.

**Triggers:**
- **On demand**: the annotation `providers.platform-mesh.io/rotate-connection-secret` on the `Provider`, set by the owner or by PM's operators, e.g. `kubectl annotate provider my-provider providers.platform-mesh.io/rotate-connection-secret="$(date -u +%FT%TZ)" --overwrite`. Any new value triggers one change; the value last acted on is kept in the connection Secret, so re-applying the same value (e.g. through GitOps) does nothing. Trusting it is harmless: it only makes PM change a secret PM controls, and it's never used as a link.
- **After tampering** with the connector CA bundle (see "Connector CA trusted set").
- **Repair**: when the connection Secret is missing, or the Provider-side rejects both `secret` and `pendingSecret` while the inner TLS against its pinned server key succeeds (e.g. its store was restored from an older backup). The `connectionID` comes from the record. Repairs are rate-limited per connection (e.g. at most one per hour), so an edge or Provider bug that breaks authentication can't cause a loop of changes.

**No periodic rotation.** Everything harmful someone could do with a stolen secret is either detected and undone (pushing a bundle) or impossible (changing the secret), so rotation would only shorten how long they could read status. If compliance ever requires it, it's a small addition, listed in the [follow-ups](./xxx-notes-011-v5-impl-todo.md). **The code carries a comment at the list of triggers** saying what periodic rotation would need (a last-change timestamp, a configured interval, due times spread by `connectionID`), why it might be wanted (compliance), and why it isn't needed now.

**Properties:**
- **Sessions are unaffected**: they never use the connection secret. A change needs one, though; without sessions it waits, and the Provider-side keeps its current hash.
- **On restart mid-change**, try `pendingSecret` first, then `secret`, and resume; the record still names the pending hash.
- **Nobody else can change the secret**: not someone holding it, an onboarding token or a portal account. There's no lock-out path.

## Recovery

Each case is covered elsewhere ([01 Protocol, "Recovery"](./xxx-notes-011-v5-impl-01-protocol.md)):
- **A lost or rejected secret**: repaired automatically (see "Secret changes").
- **A connection the Provider-side no longer knows**: `Registered=False` with `ConnectionUnknown`; the owner onboards the same `Provider` again, which registers a new connection (see "Onboarding requests").
- **No session verifies**: `Connected=False` with `NoSessions`; the Provider's administrators re-pin the connector CA bundle out-of-band.
- **Disasters on either side**: out-of-band, using the administrator input in "Records" and the SDK's administration commands ([07 SDK, "Administration"](./xxx-notes-011-v5-impl-07-provider-sdk.md)).

## Connector CA trusted set

cert-manager renews the connector CA; platform-mesh-operator makes sure every Provider trusts the new CA before connectors use it ([04 Connector CA, "Rollover"](./xxx-notes-011-v5-impl-04-connector-ca.md)):
- **New CA certificates** come from the CA's `CertificateRequest`s (public status, no key access). Each is validated and added to the **trusted set**: the current CA, plus the previous one until 48 hours after the current one was first seen, or until it's listed in the `distrust` configuration, whichever is first.
- **History**: the fingerprints of every CA ever trusted, kept with the trusted set in a ConfigMap in platform-mesh-operator's namespace. The tamper check below compares against it.
- **Push on difference**: on each poll, the `Connected` subroutine compares the connection's pinned fingerprints (from `GET`) with the trusted set, and sends `PUT /connector-ca-bundle` with `If-Match: <the etag it just read>` where they differ, rate-limited ([01 Protocol, "Connector CA bundle"](./xxx-notes-011-v5-impl-01-protocol.md)). A `412` means the bundle changed since: read it again and decide anew. When the trusted set changes, every connection is requeued, so pushes go out within minutes. Registration always sends the current set.
- **Catch up as soon as a connection is back.** A Provider that was unreachable during a rollover, however long, still pins an old set; its control channel doesn't depend on the connector CA, so the **first successful control call after a connection was unreachable** pushes the current set right away. The SDK accepts the push even if every CA it pins has expired.
- **Tampering**: a pinned certificate that isn't in the history can only have come from someone holding the connection secret. Then:
  1. restore the trusted set with the current secret (with `If-Match`), so PM's sessions verify again; the Provider-side closes every session that doesn't chain to it, including any the thief opened ([01 Protocol, "Connector CA bundle"](./xxx-notes-011-v5-impl-01-protocol.md));
  2. change the secret through the record (see "Secret changes"), which makes the stolen secret useless;
  3. record an **event** on the `Provider`, so the owner sees it happened, and count it in a metric with a critical alert rule for the people running PM ([04 Connector CA, "Signals"](./xxx-notes-011-v5-impl-04-connector-ca.md)).

  If the thief pushes again before the change completes, step 1 repeats; the change completes on the Provider-side's first read of the record over a session. A bundle that holds only CAs from the history, old or missing some, isn't tampering; the push above converges it.
- **Convergence**: a CA has converged when every connection whose control calls succeed pins it. A connection whose control calls fail counts as **not reached**, whatever the cause, and is listed with its last error; owners see the cause on their `Provider`. Nothing waits for connections that aren't reached. The active replica computes convergence and writes the whole `ConnectorCABundle` status: the trusted CAs with their `converged` flags, counts per shard, and the conditions `Valid` and `Converged`. Connectors read the flags to decide when to switch certificates ([05 Connector, "Connector certificate"](./xxx-notes-011-v5-impl-05-connector.md)).

## Offboarding

The connection-specific steps of the remote `Provider`'s finalizer, in order; the full order, including the bootstrap objects, is in [06 Provider controller, "Deletion"](./xxx-notes-011-v5-impl-06-provider-controller.md):
1. **Delete the `ProviderEndpoint`**, so every connector closes its sessions. kcp access ends here.
2. **Call `DELETE`** in a control session, with bounded retries; an unreachable Provider-side doesn't block deletion beyond a timeout.
3. **Delete the connection Secret** in platform-mesh-operator's namespace.

## Egress

- **Base URL rules**, checked before registration, before writing a record and on every dial: `https` only, public addresses only, resolve once and dial the checked addresses, no redirects to other hosts.
- **TLS** to the registration endpoint: web PKI, or the `Provider`'s `spec.remote.caBundle`.
- **Per-Provider limits**: concurrent calls, rate, backoff; jitter on periodic work. With 1,000 connections and a 5-minute poll, steady state is about 3–4 calls per second in total.

## Identity and permissions

What platform-mesh-operator needs **in addition** for this:
- **In kcp**: `apiexports/content` on the connection export, including writes to `providerendpoints`, granted by the export's maximal permission policy ([02 APIs, "Connection export"](./xxx-notes-011-v5-impl-02-apis.md)); `bind` on the connection export, and the rights to create the connector RBAC in provider workspaces ([06 Provider controller](./xxx-notes-011-v5-impl-06-provider-controller.md)). It already reads `Workspace`s in `:root:providers` and has `apiexports/content` on the providers export.
- **In the runtime cluster**: Secrets (connection Secrets) and the trusted-set ConfigMap in its own namespace; read access to `CertificateRequest`s in cert-manager's cluster resource namespace (public certificates only); write access to the `ConnectorCABundle`'s status. No access to the CA's key Secret or issuers, which only cert-manager reads.
- **Network**: outbound HTTPS to the internet, for Providers' base URLs.

## Replicas and availability

- **One active replica**, by platform-mesh-operator's leader election, with standbys that take over within seconds. It runs one multicluster-runtime provider over all shards of the providers export and one over the connection export; each shard's endpoint gets its own caches, so a shard whose virtual workspace is down only stalls its own objects. The work is mostly waiting on Providers, so it scales with reconcile concurrency, not replicas.
- **After a failover**, the new leader rebuilds its caches and reconciles every connection once, within the per-Provider rate limits.
- **Outages delay** onboarding, secret changes, rollovers and offboarding; they don't stop traffic. The one exception: an outage of more than about 12 hours right when cert-manager renews the connector CA. The new CA then never converges, connectors fall back to it as their old certificates expire, and new sessions fail at Providers that don't pin it, until platform-mesh-operator is back and pushes it. Open sessions keep running.

## Observability

- **Events** on `ProviderOnboardingRequest`s and `Provider`s for every phase change and failure, and for tampering, without secrets.
- **Metrics**: calls per Provider and outcome, poll latency, connections per condition state, secret changes in progress (by trigger), repairs refused by the rate limit, tampering detected, egress refusals, and the connector CA signals (CA expiry, `Valid`, convergence; [04 Connector CA, "Signals"](./xxx-notes-011-v5-impl-04-connector-ca.md)). Scraped through platform-mesh-operator's `ServiceMonitor` ([09 Deployment, "Observability"](./xxx-notes-011-v5-impl-09-deployment.md)).
- **Audiences**: owners act on events and conditions on their `ProviderOnboardingRequest`s and `Provider`s; the people running PM act on metrics, the `ConnectorCABundle` status and the documented alert rules. Owners never see PM-wide signals, and PM-wide signals never name owners' data beyond connection IDs.
- **Logs** never contain the token, the secret or hashes.

## Work breakdown

1. Packages in platform-mesh-operator for the connection handling and the protocol client, separate from the rest of the operator; the connection export's multicluster-runtime provider.
2. The `ProviderOnboardingRequest` reconciler and its phases.
3. Protocol client: plain-HTTPS calls, control sessions (WebSocket, inner TLS with pinned keys, HTTP/1.1), retries, egress rules, input limits and fuzz tests. Built against the SDK's test server ([07 SDK, "Testing support"](./xxx-notes-011-v5-impl-07-provider-sdk.md)).
4. The `Registered` subroutine and the records writer with drift repair, server-key updates and `pendingSecretHash`.
5. The `Connected` subroutine: polling, conditions, edge diagnosis.
6. Secret changes (on demand, repair, after tampering), with the comment on periodic rotation.
7. The connector CA trusted set, its pushes with compare-and-swap, the tamper response, and the `ConnectorCABundle` status.
8. The connection steps of offboarding.
9. Metrics, events, docs.

## Decided

- **The connection handling lives in platform-mesh-operator**, as subroutines of its Provider controller, until the `Provider` part becomes its own operator; the internet-facing code in PM's most privileged component is an accepted interim risk.
- **One active replica** by leader election; no partitioning by shard.
- **Request authorization**: no creator checks; RBAC on requests is granted with managing `Provider`s.
- **Concurrent requests**: the oldest wins; newer ones fail.
- **Requests end at registration**: `Pending → Registering → Registered`, or `Failed` only before registration succeeds. Everything after is reported on the `Provider`.
- **Secret scope**: connection Secrets in platform-mesh-operator's namespace; token Secrets read by name, with no Secret informers in kcp.
- **Onboarding order**: register, first control call, then the record; `connectionID` and server keys persisted together.
- **Secret changes**: on demand (annotation), after tampering, and repair; no periodic rotation.
- **Connector CA**: the trusted set derived from cert-manager's CA is kept and pushed here; only connections that are reached count for convergence.
- **Recovery**: the secret changes only through the record; an owner onboards a registered `Provider` again only after the Provider-side reports its connection unknown.
