# v5 implementation plan: providerconnections-operator

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). The **control plane**: a new operator at `platform-mesh/operators/providerconnections-operator`. It owns every connection's lifecycle on the PM-side: onboarding, registration, records, health, rotation, rebind, connector CA rollover and offboarding. It's never on the session path.

## Responsibilities

**Owns:**
- reconciling `ProviderOnboardingRequest`s and the connection side of `Provider`s;
- connection Secrets in owners' workspaces;
- every call to a Provider's connection API ([01 Protocol](./xxx-notes-011-v5-impl-01-protocol.md));
- `ProviderEndpoint` records in provider workspaces ([02 APIs](./xxx-notes-011-v5-impl-02-apis.md));
- `Provider.status.connection` and the connection conditions;
- fanning out connector CA rollovers ([04 Connector CA](./xxx-notes-011-v5-impl-04-connector-ca.md)).

**Doesn't own:**
- the provider workspace, SA, binding or connector RBAC ([06 Provider controller](./xxx-notes-011-v5-impl-06-provider-controller.md));
- sessions, proxying or SA tokens ([05 Connector](./xxx-notes-011-v5-impl-05-connector.md));
- issuing connector certificates ([04 Connector CA](./xxx-notes-011-v5-impl-04-connector-ca.md)).

## What it consumes

Exactly two endpoint slices, which is the reason it's a separate operator:

| Endpoint slice | Through it, the operator… |
|---|---|
| **providers export** (owner side) | reads and updates `ProviderOnboardingRequest`s and `Provider`s; reads token Secrets and reads and writes connection Secrets in owners' workspaces, by name and uncached, through the export's existing Secrets claim (see "Secrets") |
| **connection export** (`:root:providers:system`) | writes `ProviderEndpoint`s in provider workspaces |

Plus, outside kcp:
- **read access to `Workspace` objects in `:root:providers`**, to find the provider workspace's logical cluster (see "Finding the provider workspace");
- **the `ConnectorCABundle`**: the connector CA's generations, their states and public certificates; the operator writes its status ([04 Connector CA, "ConnectorCABundle"](./xxx-notes-011-v5-impl-04-connector-ca.md));
- **outbound HTTPS to Providers' base URLs**.

## Principles

- **Write before send** (overview, principle 7): every new secret is in the connection Secret before it's sent.
- **The Provider-side is the source of truth** for what it has pinned and promoted; the operator reads it back with `GET` and converges.
- **Never trust owner-writable fields for links** (overview, principle 4).
- **Bounded, polite egress**: per-Provider rate limits, jitter and backoff; Providers are external services.
- **Aggregate status only** towards owners.

## Secrets

The providers export's Secrets claim is accepted with `matchAll` in PM's bindings, so the operator *could* read every Secret in owners' workspaces. It doesn't cache any of them:
- **It needs exactly two Secrets per connection, both by known name**: the token Secret named in the request's spec, and the connection Secret it creates itself.
- **No Secret informers.** Both are read and written by name with an uncached client, when a reconcile needs them. Nothing about other Secrets ever enters the operator's memory, and a memory dump exposes only Secrets currently being reconciled.
- **No Secret watches are needed.** A token Secret created after its request is picked up by requeueing with backoff; a lost connection Secret is handled by a new request (see "Rebind"); rotation and health are periodic.
- **Rights are unchanged**: the operator's identity can still reach any Secret the claim allows. Narrowing the claim with a label selector in PM's bindings would restrict that too, but affects every consumer of the providers export; it's a PM-wide hardening outside this plan.

## Finding the provider workspace

The operator needs the provider workspace's `clusterID` for registration, and where to write the record. It never takes these from the `Provider`'s spec or status:
1. **Derive the workspace name** the same way the Provider controller does: `<Provider name>-<the Provider's kcp.io/cluster annotation>` under `:root:providers`. The annotation is set by kcp, not the owner. Share this function between the two operators, so they can't drift.
2. **Read the `Workspace` object** in `:root:providers` (PM-owned; owners have no rights there) for its logical cluster name and phase.
3. **Derive the SA's namespace and name** with the Provider controller's shared function.
4. **Wait for `ConnectionPrerequisitesReady`** on the `Provider` as a hint, and confirm it by seeing the provider workspace appear in the connection export's endpoint slice (the binding exists).

## Onboarding

Reconciles each `ProviderOnboardingRequest`:

1. **Validate the request**: it's the oldest unfinished request for its `Provider` (otherwise fail it, naming the active one); the token Secret exists and parses; the base URL inside passes the egress rules (see "Egress"); `spec.provider` is a remote Provider. The operator doesn't check who created the request: `create` on requests is granted together with managing `Provider`s ([02 APIs, "ProviderOnboardingRequest"](./xxx-notes-011-v5-impl-02-apis.md)).
2. **Create or reuse the `Provider`**:

   | The `Provider`… | The operator… |
   |---|---|
   | doesn't exist | creates it with `spec.remote` from the request |
   | exists, not registered | reuses it |
   | exists, registered, connection Secret present | fails the request: changes to a live connection go through the operator's own reconciliation |
   | exists, registered, connection Secret missing | starts a rebind (see "Rebind") |
   | exists, not remote (e.g. a ManagedProvider's) | fails the request |

3. **Wait for the provider workspace** (see "Finding the provider workspace"). Phase `WaitingForProvider`.
4. **Register** (see "Registration"). Phase `Registering`.
5. **Write the record** (see "Records").
6. **Verify readiness** (see "Readiness and health"). Phase `VerifyingKcp`, then `Ready`. **No terminal failure after registration**: past a timeout (e.g. 15 minutes), the request gets a warning condition with the Provider-side's last error and keeps reconciling, so it becomes `Ready` once the problem is fixed. The connection stays registered and is reconciled like any other.
7. **Delete the token Secret's content** ([02 APIs, "ProviderOnboardingRequest"](./xxx-notes-011-v5-impl-02-apis.md)). Only after the registration's result (the `connectionID`) is persisted in the connection Secret and the request's status: until then, a lost response may still need a retry with the same token. The same applies to a rebind's token once the `POST /rebind` call has succeeded. The operator doesn't delete the Secret object itself, which the owner may manage through GitOps, only its token data.

Every step is resumable: the request's status and the connection Secret record how far it got.

## Registration

1. **Generate the connection secret** and write it to the connection Secret (write before send).
2. **Call `POST /v1/connections`** with the token, the request's UID as idempotency key, the `clusterID`, the front-proxy URL, the secret's hash and the current connector CA certificates ([01 Protocol, "Registration"](./xxx-notes-011-v5-impl-01-protocol.md)).
3. **On success**, record the `connectionID` in the connection Secret, the request's status and `Provider.status.connection`, and keep the returned server keys for the record.
4. **On a lost response**, retry with the same idempotency key and hash; the same `connectionID` comes back.
5. **On refusal** (token spent elsewhere, expired, wrong owner), fail the request with a reason the owner can act on. This is the last point at which a request can fail.

## Records

The operator writes one `ProviderEndpoint` per provider workspace through the connection export ([02 APIs, "ProviderEndpoint"](./xxx-notes-011-v5-impl-02-apis.md)):
- **Contents**: `connectionID`, base URL, SA reference, server keys.
- **Drift repair**: if the record is missing (e.g. the binding was recreated) or differs, the operator rewrites it. Connectors react to the change ([05 Connector, "Discovery"](./xxx-notes-011-v5-impl-05-connector.md)).
- **Server key updates**: when `GET` shows a new server key, the operator adds it to the record before the Provider-side starts using it, and removes retired keys after the overlap.
- **Deletion first on offboarding** (see "Offboarding").

## Readiness and health

- **Polls `GET /v1/connections/{id}`** over a control session, every few minutes per connection with jitter; more often while `VerifyingKcp`.
- **Turns per-host status into aggregate conditions**:
  - `SessionsConnected`: every required host has at least one session;
  - `KcpReachable`: every required host reaches kcp;
  - `SecretCurrent`: no rotation pending;
  - `Rebinding`: a rebind is pending;
  - `ProtocolOutdated` (warning): the connection still negotiates the previous protocol version, so the Provider should upgrade its SDK ([07 SDK, "Releases and protocol versions"](./xxx-notes-011-v5-impl-07-provider-sdk.md)).
- **Readiness for onboarding**: at least the front-proxy host is covered and reachable (before any consumer binds, that's the only required host).
- **Edge diagnosis**: on the first control session after registration and on every failed reconnect, the operator classifies failures as described in [01 Protocol, "Diagnosing the edge"](./xxx-notes-011-v5-impl-01-protocol.md) and reports them with distinct reasons on the request's warning condition (while `VerifyingKcp`) and on the `Provider`'s connection conditions: `EdgeUpgradeRefused` (non-`101` response), `EdgeBotChallenge` (an HTML page instead of an upgrade), `EdgePayloadBlocked` (closed right after a binary message), `EdgeTimeout` (closed at a regular idle time or age). Each reason's message points the owner at the Provider's edge checklist and self-test ([08 Quickstart, "Connection server"](./xxx-notes-011-v5-impl-08-provider-quickstart.md)). The operator only observes its own control sessions; connectors' session failures go to their metrics, where the same classification applies.
- **Every successful poll also cancels any pending rebind** against a connection whose secret still exists ([01 Protocol, "Rebind"](./xxx-notes-011-v5-impl-01-protocol.md)).
- **Per-shard and per-host detail goes to metrics**, never to conditions (overview, principle 9).

## Secret rotation

Automatic, periodic (e.g. every 30 days), on demand (an annotation on the `Provider`) and after a suspected leak:
1. Generate a new secret; write it as `pendingSecret`.
2. `PUT /secret` with the new hash, authenticated with the current secret.
3. Authenticate once with the new secret (a `GET`); the Provider-side promotes it.
4. Move `pendingSecret` to `secret`.

On restart mid-rotation, try `pendingSecret` first, then `secret`, and resume. Sessions are unaffected.

## Rebind

When a registered `Provider`'s connection Secret is missing and the owner creates a new request:
1. Generate a new secret, write it as `pendingSecret` in a new connection Secret.
2. Call `POST /rebind` with the new token and hash. Phase `Rebinding`; record `rebindCompletesAt`.
3. Wait. Sessions keep working throughout.
4. After the waiting period, authenticate with the new secret; on success, promote it to `secret`, phase `Ready`.

If the old secret still exists somewhere the operator can use, it never rebinds; it rotates. A rebind cancelled by someone else's successful authentication fails the request with a reason pointing at a possible leak.

## Connector CA rollover

Driven by the `ConnectorCABundle` resource ([04 Connector CA, "ConnectorCABundle"](./xxx-notes-011-v5-impl-04-connector-ca.md)); people change generation states, the operator only follows:
- **Every shard worker watches the resource's spec**, validates the certificates, derives the sequence and builds the bundle. All workers compute the same bundle for the same spec, so a late call from a previous Lease holder carries a lower sequence and is refused instead of reverting a newer set.
- **Push on change**: when the sequence increases, each worker sends `PUT /connector-ca-bundle` to its connections whose pinned sequence (read back from `GET`) is lower, rate-limited ([01 Protocol, "Connector CA bundle"](./xxx-notes-011-v5-impl-01-protocol.md)). Registration always sends the current bundle.
- **Report per shard**: each worker writes its own entry in the resource's `status.shards` (connections, pinned current, unreachable) with a per-shard field manager.
- **Aggregate**: one aggregator, under its own Lease separate from the shard Leases, computes the sequence, the published generations, totals and the conditions (`Valid`, `BundleConverged`, `ReadyToActivate`, `ReadyToRetire`) from the shard entries and the connector `Certificate` objects.
- **Unreachable connections** are listed, not blocked on: conditions count reachable connections only.
- **Catch up as soon as a connection is back.** A Provider that was down during a rollover still pins the old bundle; once connector certificates come from the new generation, it rejects sessions. Its control channel doesn't depend on the connector CA, so on the **first successful control call after a connection was unreachable**, the worker compares the pinned sequence with the current one and pushes the bundle right away, instead of waiting for the next regular poll. The gap after the Provider's return shrinks from a poll interval to seconds.

## Offboarding

A finalizer on remote `Provider`s:
1. **Delete the `ProviderEndpoint`**, so every connector closes its sessions. kcp access ends here.
2. **Call `DELETE`** in a control session, with bounded retries; an unreachable Provider-side doesn't block deletion beyond a timeout.
3. **Remove the finalizer**; the Provider controller then deletes the SA, binding and workspace ([06 Provider controller, "Deletion order"](./xxx-notes-011-v5-impl-06-provider-controller.md)).

The connection Secret goes with the `Provider` through its owner reference.

## Egress

- **Base URL rules**, checked before registration, before writing a record and on every dial: `https` only, public addresses only, resolve once and dial the checked addresses, no redirects to other hosts.
- **TLS** to plain-HTTPS endpoints: web PKI, or the `Provider`'s `spec.remote.caBundle`.
- **Per-Provider limits**: concurrent calls, rate, backoff; jitter on periodic work. With 1,000 connections and a 5-minute poll, steady state is about 3–4 calls per second in total.

## Identity and permissions

The operator's kcp identity (a kcp-operator `Kubeconfig`) needs:
- `apiexports/content` on the providers export (as the platform-mesh-operator already has) and on the connection export;
- writes to `providerendpoints` through the connection export, granted by the export's maximal permission policy ([02 APIs, "Connection export"](./xxx-notes-011-v5-impl-02-apis.md));
- `get`, `list`, `watch` on `workspaces` in `:root:providers`.

In the runtime cluster, it needs: read access to the `ConnectorCABundle` and write access to its status; read access to the connector `Certificate` objects (not their Secrets); Leases in its namespace. Nothing else, and in particular no access to CA keys or issuers.

## Partitioning by shard

One Deployment with several replicas in **PM's runtime cluster** (next to platform-mesh-operator), partitioned internally by kcp shard:

- **The shard list comes from kcp**: the providers export's endpoint slice lists one virtual-workspace URL per shard with owner workspaces bound to it. No deployment change when shards are added or removed.
- **Replicas and shards are independent**: N replicas share M shards in any combination. Each shard has exactly one owner replica at a time (one Lease per shard); a replica can own any number of shards, about M/N. With more replicas than shards, the extra replicas stand by.
- **Per-shard Leases**: each replica runs a shard manager that takes Leases for the shards rendezvous hashing assigns to it, and for any shard whose Lease is free and whose preferred replica is gone. Leases live in the operator's namespace in PM's cluster.
- **Why one owner per shard**: every object needs a single writer at a time (rotations, rebinds and rollovers must not race), and the shard is the coarsest unit that gives that cheaply. Work within a shard is mostly waiting on Provider calls, so it scales with reconcile concurrency (configurable per worker), not replicas. If a single shard ever grows too large, it can be split into buckets with one Lease per (shard, bucket); buckets spread work but not memory, since each bucket's worker still caches the whole shard.
- **One worker per held shard**: a multicluster-runtime provider restricted to that shard's endpoint of the providers export, with its own caches and work queues. It reconciles requests, `Provider`s and connections of owners' workspaces hosted on that shard. Caches therefore hold only the shards a replica currently holds, and a stuck shard doesn't hold up others.
- **Shared across workers**: the connection export's full endpoint slice (records may live on any shard), the protocol client with its per-Provider rate limits, and the `ConnectorCABundle`.
- **Shard added**: kcp adds its endpoint to the slice; a replica takes its Lease and starts a worker. **Removed**: the endpoint disappears; the holder stops the worker and releases the Lease. **Replica lost**: its Leases expire; others take its shards over.
- **Rollover fan-out** runs per worker over its own connections; convergence is reported per shard and summed.
- **Isolation is of memory and failures, not of access**: all replicas share one kcp identity, and `apiexports/content` isn't shard-scoped. Per-shard access isolation would need per-shard identities, out of scope here.
- **Remote shards in other clusters** are reached through their virtual-workspace URLs, like any multi-shard consumer. If replicas ever have to span clusters, the Leases and the `ConnectorCABundle` move into a PM workspace in kcp.

## Availability

- **Outages delay** onboarding, rotations, rebinds, rollovers and offboarding for the affected shards; they never stop traffic.

## Observability

- **Events** on `ProviderOnboardingRequest` and `Provider` for every phase change and failure, without secrets.
- **Metrics**: calls per Provider and outcome, poll latency, connections per condition state, rotations and rebinds in progress, egress refusals, and the connector CA signals (generation expiry, `Valid`, rollover convergence; [04 Connector CA, "Signals"](./xxx-notes-011-v5-impl-04-connector-ca.md)). Scraped through a `ServiceMonitor` ([09 Deployment, "Observability"](./xxx-notes-011-v5-impl-09-deployment.md)).
- **Audiences**: owners act on events and conditions on their `ProviderOnboardingRequest`s and `Provider`s; the people running PM act on metrics, the `ConnectorCABundle` status and the documented alert rules. Owners never see PM-wide signals, and PM-wide signals never name owners' data beyond connection IDs.
- **Logs** never contain the token, the secret or hashes.

## Work breakdown

1. Scaffold the operator in the monorepo: the shard manager (per-shard Leases, rendezvous preference), per-shard workers on the providers export, the shared connection-export provider, config.
2. Shared naming functions with platform-mesh-operator (workspace, SA).
3. Onboarding reconciler and its phases.
4. Protocol client: plain-HTTPS calls, control sessions (WebSocket, inner TLS with pinned keys), retries, egress rules. Shared with nothing else on the PM-side; built against the SDK's test server ([07 SDK, "Testing support"](./xxx-notes-011-v5-impl-07-provider-sdk.md)).
5. Records writer with drift repair and server-key updates.
6. Health poller and condition aggregation.
7. Secret rotation, rebind, connector CA rollover.
8. Offboarding finalizer, coordinated with the Provider controller.
9. Metrics, events, docs.

## Decided

- **Request authorization**: no creator checks; RBAC on requests is granted with managing `Provider`s.
- **Concurrent requests**: the oldest wins; newer ones fail.
- **After registration, no terminal failure**: a warning condition after the readiness timeout, and continued reconciliation.
- **Partitioning**: one Deployment in PM's cluster, partitioned by kcp shard with per-shard Leases (see "Partitioning by shard").

- **Secret scope**: no Secret informers (see "Secrets").
