# v5 implementation plan: overview

Working notes for [011](./011-provider-onboarding-flow-improvements.md). Not the RFC itself. This is the entry point to the implementation plan for the [v5 design](./xxx-notes-011-provider-onboarding-flow-improvements-v5.md), **with inner TLS on sessions as the only mode** (v5, "The Provider's edge: inner TLS on sessions"). The plan is split into one document per component; each says what the component owns, what it relies on, and where the flow continues. It describes principles and responsibilities, not code.

Start here, then follow the walkthrough below, or jump to a component.

## Documents

| Document | What it covers |
|---|---|
| [00 Overview](./xxx-notes-011-v5-impl-00-overview.md) | Decisions, component map, end-to-end walkthrough, cross-cutting principles, ownership, build order |
| [01 Protocol](./xxx-notes-011-v5-impl-01-protocol.md) | The wire contract between PM and the Provider-side: endpoints, credentials, session assertion, inner TLS, multiplexer, status, versioning |
| [02 APIs](./xxx-notes-011-v5-impl-02-apis.md) | kcp-side types and exports: `ProviderOnboardingRequest`, `Provider`, the connection Secret, `ProviderEndpoint`, the connection export and its maximal permission policy |
| [03 providerconnections-operator](./xxx-notes-011-v5-impl-03-providerconnections-operator.md) | The control plane: onboarding, registration, records, health, secret changes, the connector CA trusted set, offboarding |
| [04 Connector CA](./xxx-notes-011-v5-impl-04-connector-ca.md) | The connector CA managed by cert-manager, connector certificates, the trusted set Providers pin, and the automatic rollover |
| [05 Connector](./xxx-notes-011-v5-impl-05-connector.md) | The data plane: the per-shard `providerconnections` virtual workspace, sessions, proxying, policy |
| [06 Provider controller changes](./xxx-notes-011-v5-impl-06-provider-controller.md) | What the RFC 006 Provider controller in platform-mesh-operator changes for remote Providers |
| [07 Provider SDK](./xxx-notes-011-v5-impl-07-provider-sdk.md) | The Provider-side library: connection API, store, session server, transport, lifecycle, portal primitives |
| [08 Provider quickstart](./xxx-notes-011-v5-impl-08-provider-quickstart.md) | The reference Provider in pm-provider-quickstart |
| [09 Deployment](./xxx-notes-011-v5-impl-09-deployment.md) | Charts, kcp bootstrap manifests, kcp-operator and cert-manager resources, network, rollout |
| [10 Testing](./xxx-notes-011-v5-impl-10-testing.md) | Unit, integration, multi-shard kind e2e, security and load tests |

## Decisions taken

| Topic | Decision |
|---|---|
| Inner TLS on sessions | **Required, the only mode.** Plain sessions aren't implemented. |
| Delivery | **Single delivery**: each document describes its full scope; "Build order" below orders the work. |
| Control plane | A new operator in the platform-mesh monorepo: **`operators/providerconnections-operator`**. It consumes exactly two endpoint slices: the providers export's and the connection export's. |
| Data plane | A new service: **`services/providerconnections-vw`**, built on the virtual workspace framework, one per shard, deployed as a kcp-operator **`VirtualWorkspace`** with `shardRef`. |
| Provider SDK | **`platform-mesh/provider-sdk`**, module `go.platform-mesh.io/provider-sdk`, its own semver line released through `cmd/release` ([07](./xxx-notes-011-v5-impl-07-provider-sdk.md)). |
| Record type | **`connections.platform-mesh.io/ProviderEndpoint`**, served only by the connection export. `ProviderConnection` is removed from `providers.platform-mesh.io`. |
| Connection export home | **`:root:providers:system`**. |
| Multiplexer | **yamux** over the WebSocket. |
| Connector CA | **Managed by cert-manager and rolled over automatically**: 1 year, renewed 90 days before expiry, self-signed by default or from an `issuerRef`. The operator pushes each new CA to every Provider before connectors use it; a status-only **`ConnectorCABundle`** reports progress ([04](./xxx-notes-011-v5-impl-04-connector-ca.md)). |
| Provider-side state | An SDK **storage interface** plus a **Kubernetes reference store** in the Provider's runtime cluster. |
| Portal | **SDK primitives** (token issuing and verification, connection lifecycle, notification hooks) plus a minimal flow in **pm-provider-quickstart**. |
| E2E | **kind, multi-shard**, with the Provider behind a network boundary, extending platform-mesh-operator's `test/e2e/kind`. |
| Naming | `xxx-notes-011-v5-impl-*.md` in `rfc/`. |
| Operator partitioning | **One Deployment in PM's cluster, partitioned by kcp shard** with per-shard Leases; replicas and shards independent ([03](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)). |
| Chart | One new `providerconnections` chart (operator + per-shard connector resources), deployed by platform-mesh-operator as a profile service, with an explicit `kcpTopology` that duplicates `infra`'s shards (must match; profile validation enforcing it is a follow-up) ([09](./xxx-notes-011-v5-impl-09-deployment.md)). |
| Operator Secret access | **Connection Secrets in the operator's namespace** in PM's runtime cluster, never in owners' workspaces; token Secrets read by name, uncached, with **no Secret informers in kcp** ([02](./xxx-notes-011-v5-impl-02-apis.md), [03](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)). |
| Connection secret changes | **Only through the PM-written `ProviderEndpoint`** (`pendingSecretHash`), adopted by the Provider-side over a session; no call changes a secret, so a thief can't lock PM out. Lost or rejected secrets are repaired by the operator; a connection the Provider-side no longer knows is replaced by onboarding the same `Provider` again ([01](./xxx-notes-011-v5-impl-01-protocol.md), [03](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)). |
| Connector CA bundle at Providers | **Compare-and-swap** (`etag`, `If-Match`); a CA PM never trusted is treated as tampering: PM restores the bundle and changes the secret. When the bundle changes, the Provider-side closes sessions that no longer chain to it ([01](./xxx-notes-011-v5-impl-01-protocol.md), [03](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)). |
| Recovery | **Out-of-band, but always possible**: stable `connectionID`s, an SDK import of connection entries, re-pinning a bundle, and administrator input for base URL and server keys on the PM-side. Procedures aren't part of this plan ([01](./xxx-notes-011-v5-impl-01-protocol.md), [07](./xxx-notes-011-v5-impl-07-provider-sdk.md)). |
| Requests | Owners create them (one ClusterRole with `Provider`s), no creator checks; the oldest request per `Provider` wins; no terminal failure after registration; spent token data deleted ([02](./xxx-notes-011-v5-impl-02-apis.md), [03](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)). |
| Migration | None: `spec.remote` is unused today ([06](./xxx-notes-011-v5-impl-06-provider-controller.md)). |
| Protocol | Current and previous version supported; 1 minute clock skew; length-prefixed binary stream preamble; registration is the only plain-HTTPS call ([01](./xxx-notes-011-v5-impl-01-protocol.md)). |
| Connector | Connections move on replica restart; cache-server credentials accepted; child workspaces denied; one identity group for all shards ([05](./xxx-notes-011-v5-impl-05-connector.md)). |
| SDK | One active replica by leader election (standbys answer `503`), owner replicas as a later opt-in; Go only; CRDs plus Secrets as reference store ([07](./xxx-notes-011-v5-impl-07-provider-sdk.md)). |
| Quickstart | Keeps kubeconfig mode alongside connection mode ([08](./xxx-notes-011-v5-impl-08-provider-quickstart.md)). |
| API version | `connections.platform-mesh.io/v1alpha1` ([02](./xxx-notes-011-v5-impl-02-apis.md)). |

## Component map

```
 owner's workspace :root:orgs:<org>:<user>                 PM runtime cluster
 ┌──────────────────────────────────────────┐              ┌──────────────────────────────────────────────┐
 │ ProviderOnboardingRequest                │   providers  │ platform-mesh-operator                       │
 │ Provider (spec.remote, status.connection)│◀── export ──▶│   Provider controller (RFC 006, doc 06)      │
 │ token Secret                             │   (owner     │ providerconnections-operator (doc 03)        │
 └──────────────────────────────────────────┘    side)     │   consumes: providers export,                │
                                                           │             connection export                │
                                                           │   connection Secrets (own namespace)         │
 :root:providers:system                                    │ connector CA (cert-manager, doc 04)            │
 ┌──────────────────────────────────────────┐              └──────────────┬───────────────────────────────┘
 │ connection export connections.pm.io      │                             │ connector certificates
 │   + maximal permission policy RBAC       │                             ▼
 │ providers export (unchanged home)        │              ┌──────────────────────────────────────────────┐
 └──────────────────────────────────────────┘              │ providerconnections-vw @ shard-A, -B, …      │
                                                           │ (doc 05)  reads ProviderEndpoints + slices,   │
 provider workspace :root:providers:<name>-<owner cluster> │   mints SA tokens, proxies to kcp            │
 ┌──────────────────────────────────────────┐              └──────────────┬───────────────────────────────┘
 │ APIBinding to the connection export      │                             │ outbound WebSocket + inner TLS
 │ ProviderEndpoint (PM-written only)       │                             ▼   (protocol, doc 01)
 │ SA (admin), Provider's own exports/slices│              ┌──────────────────────────────────────────────┐
 └──────────────────────────────────────────┘              │ Provider runtime                             │
                                                           │   Provider SDK (doc 07): connection API,     │
                                                           │   session server, transport, store           │
                                                           │   Provider controllers + portal (doc 08)     │
                                                           └──────────────────────────────────────────────┘
```

## End-to-end walkthrough

Each step names the document and section that owns it; read them in this order to follow the flow.

1. **The owner gets a token** from the Provider's portal. The token format and its single-use rules: [01 Protocol, "Onboarding token"](./xxx-notes-011-v5-impl-01-protocol.md). Issuing and verifying: [07 SDK, "Onboarding tokens and portal primitives"](./xxx-notes-011-v5-impl-07-provider-sdk.md).
2. **The owner creates a `ProviderOnboardingRequest`** referencing the token Secret: [02 APIs, "ProviderOnboardingRequest"](./xxx-notes-011-v5-impl-02-apis.md).
3. **The operator creates or reuses the `Provider`**: [03 Operator, "Onboarding"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md).
4. **The Provider controller bootstraps the provider workspace** (workspace, SA, connection export binding, connector RBAC), without a kubeconfig for remote Providers: [06 Provider controller](./xxx-notes-011-v5-impl-06-provider-controller.md).
5. **The operator registers the connection** with the Provider-side: generates the connection secret, sends its hash and the connector CA certificates, receives the `connectionID` and server keys: [03 Operator, "Registration"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md); the Provider's side: [07 SDK, "Connection API"](./xxx-notes-011-v5-impl-07-provider-sdk.md); the CA certificates it sends: [04 Connector CA, "ConnectorCABundle"](./xxx-notes-011-v5-impl-04-connector-ca.md).
6. **The operator writes the `ProviderEndpoint`** into the provider workspace through the connection export: [03 Operator, "Records"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md); type and policy: [02 APIs, "ProviderEndpoint" and "Connection export"](./xxx-notes-011-v5-impl-02-apis.md).
7. **Connectors discover the record and open sessions** where needed: [05 Connector, "Discovery" and "Which sessions"](./xxx-notes-011-v5-impl-05-connector.md); the session handshake: [01 Protocol, "Sessions"](./xxx-notes-011-v5-impl-01-protocol.md); the Provider's side: [07 SDK, "Session server"](./xxx-notes-011-v5-impl-07-provider-sdk.md).
8. **The Provider's controllers reconcile through the sessions**: transport and client configs in [07 SDK, "Transport" and "Connections for multicluster-runtime"](./xxx-notes-011-v5-impl-07-provider-sdk.md); the proxy on PM's side in [05 Connector, "Proxying"](./xxx-notes-011-v5-impl-05-connector.md); the reference wiring in [08 Quickstart](./xxx-notes-011-v5-impl-08-provider-quickstart.md).
9. **The operator confirms readiness** from the Provider-side's per-host status: [03 Operator, "Readiness and health"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md); [01 Protocol, "Status"](./xxx-notes-011-v5-impl-01-protocol.md).
10. **Steady state**: the operator polls health and changes the connection secret when needed, through the record; cert-manager renews connector certificates and, once a year, the connector CA, which the operator pushes before connectors switch ([04 Connector CA, "Rollover"](./xxx-notes-011-v5-impl-04-connector-ca.md)); connectors refresh SA tokens: [03 Operator, "Secret changes"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md), [01 Protocol, "Changing the secret"](./xxx-notes-011-v5-impl-01-protocol.md), [07 SDK, "The connection's record"](./xxx-notes-011-v5-impl-07-provider-sdk.md), [04 Connector CA, "Connector certificates"](./xxx-notes-011-v5-impl-04-connector-ca.md), [05 Connector, "Identity towards kcp"](./xxx-notes-011-v5-impl-05-connector.md).
11. **Rare operations**: secret repair and onboarding a `Provider` again ([01 Protocol, "Recovery"](./xxx-notes-011-v5-impl-01-protocol.md), [03 Operator, "Recovery"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)); out-of-band administration ([07 SDK, "Administration"](./xxx-notes-011-v5-impl-07-provider-sdk.md)); server key rotation ([07 SDK, "Server keys"](./xxx-notes-011-v5-impl-07-provider-sdk.md)).
12. **Offboarding**: the operator deletes the record (sessions close), calls `DELETE`, then lets the Provider controller tear down the workspace: [03 Operator, "Offboarding"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md), [06 Provider controller, "Deletion order"](./xxx-notes-011-v5-impl-06-provider-controller.md).

Deployment of all PM-side pieces: [09 Deployment](./xxx-notes-011-v5-impl-09-deployment.md). How each step is tested: [10 Testing](./xxx-notes-011-v5-impl-10-testing.md).

## Cross-cutting principles

Every component follows these; the component documents refer back here instead of repeating them.

1. **No kcp credential leaves the PM-side.** The Provider-side never receives a token, kubeconfig or kcp CA bundle. Only connectors hold SA tokens, in memory.
2. **Authority only through authenticated, end-to-end encrypted sessions.** The Provider's edge sees opaque bytes on sessions and control sessions. Nothing that grants anything is sent on plain HTTPS except the onboarding token (single-use) and public material.
3. **Records are PM-written.** `ProviderEndpoint` objects are written only by the operator, through the connection export; kcp enforces it with the export's maximal permission policy. Connectors only read them.
4. **Never trust owner-writable fields for links.** The link between a `Provider` and its provider workspace, SA and connection is derived from PM-controlled data (names kcp sets, objects in `:root:providers`), never from fields the owner can edit (`spec`, `status` or annotations of the `Provider`).
5. **The data plane is stateless.** Connectors hold no persistent state, write nothing to kcp except TokenRequests, and learn everything else from their deployment and the connection export.
6. **Control-plane outages don't stop traffic.** Nothing the operator does is on the session path. The one exception: an operator outage of more than about 12 hours right when cert-manager renews the connector CA (once a year) stops new sessions until the operator pushes the new CA; open sessions keep running ([04 Connector CA, "Rollover"](./xxx-notes-011-v5-impl-04-connector-ca.md)).
7. **Write before send.** Every generated secret or key is persisted before it's sent anywhere, so a crash can always resume.
8. **Everything is idempotent and resumable.** Reconcilers converge from any intermediate state; the Provider-side is the source of truth for what it has pinned, and the operator reads it back.
9. **Owners learn nothing about PM's topology.** Owners see aggregate conditions only; shards, hosts and per-shard detail stay in metrics and PM-internal objects.
10. **One mode, versioned protocol.** Inner TLS is always on; the protocol carries an explicit version so SDK and PM can evolve independently ([01 Protocol, "Versioning"](./xxx-notes-011-v5-impl-01-protocol.md)).
11. **Signals go to the audience that can act on them.** Owners get conditions and events on their own objects; the people running PM get metrics, the `ConnectorCABundle` status and documented alert rules; Providers get SDK notifications. PM has no alerting pipeline today, so the plan ships signals and documented rules, not alerts.
12. **Secrets never appear in logs, events, metrics, conditions or audit bodies.** Prefixed formats (`pm1.`, `pmc1_`) make leaks detectable by scanners.

## Who writes what

Clear ownership avoids two controllers fighting over the same fields.

| Object or field | Written by | Read by |
|---|---|---|
| `ProviderOnboardingRequest.spec` | owner | operator |
| token Secret | owner; the operator deletes its token data once spent | operator |
| `ProviderOnboardingRequest.status` | operator | owner |
| `Provider.spec` | owner (or operator, when the request creates it) | Provider controller, operator |
| `Provider.status` bootstrap fields and conditions | Provider controller | owner, operator (as a hint only, principle 4) |
| `Provider.status.connection` and connection conditions | operator | owner |
| connection Secret (operator's namespace, runtime cluster) | operator | operator only |
| `ProviderEndpoint.pendingSecretHash` | operator | Provider SDK, over sessions |
| provider workspace, SA, binding, connector RBAC | Provider controller | operator, connectors |
| `ProviderEndpoint` | operator | connectors (and readable by the Provider) |
| connector CA, connector certificates | cert-manager | operator (public certificates, from `CertificateRequest`s), connectors (their own connector certificate) |
| trusted set, `ConnectorCABundle` status | operator | connectors (switch rule), people running PM |
| SA tokens | connectors (TokenRequest) | connectors only |
| Provider-side connection state | Provider SDK | Provider SDK, portal |

Status writes use server-side apply with distinct field managers, so the Provider controller and the operator never overwrite each other's conditions ([02 APIs, "Status ownership"](./xxx-notes-011-v5-impl-02-apis.md)).

## Build order

Single delivery, but the pieces depend on each other. Suggested order, with what each step unblocks:

1. **Protocol and APIs** ([01](./xxx-notes-011-v5-impl-01-protocol.md), [02](./xxx-notes-011-v5-impl-02-apis.md)): the contract everyone builds against. Includes protocol test vectors.
2. **SDK core** ([07](./xxx-notes-011-v5-impl-07-provider-sdk.md)): connection API, store, session server, transport. Can be tested against a fake PM side.
3. **Connector CA** ([04](./xxx-notes-011-v5-impl-04-connector-ca.md)) and **kcp bootstrap** of the connection export ([09](./xxx-notes-011-v5-impl-09-deployment.md)).
4. **Provider controller changes** ([06](./xxx-notes-011-v5-impl-06-provider-controller.md)): binding, connector RBAC, no kubeconfig for remote Providers.
5. **Operator** ([03](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)) and **connector** ([05](./xxx-notes-011-v5-impl-05-connector.md)), in parallel; they meet only at the `ProviderEndpoint` and the protocol.
6. **Quickstart** ([08](./xxx-notes-011-v5-impl-08-provider-quickstart.md)).
7. **Deployment** ([09](./xxx-notes-011-v5-impl-09-deployment.md)) and **e2e, security and load tests** ([10](./xxx-notes-011-v5-impl-10-testing.md)).

## Deployment target

The plan targets PM's current deployment: platform-mesh-operator rendering `pm-helm-charts` from the `PlatformMesh` profile. The newer, not yet released platform-mesh-deployer (`OCMModule`s with `root-shard` and `per-shard` placements) is a later move; the components map onto it without design changes, but the deployer needs three extensions first ([09 Deployment, "Later: moving to platform-mesh-deployer"](./xxx-notes-011-v5-impl-09-deployment.md)).

## External dependencies

| Dependency | Needed for | Status |
|---|---|---|
| Alerting in PM | Turning the documented alert rules (connector CA renewal, connector certificates, convergence) into notifications for the people running PM | **Outside this plan.** PM collects metrics but has no alerting pipeline (Alertmanager disabled, no rules shipped). This plan ships metrics, `ServiceMonitor`s, documented rules and an optional `PrometheusRule` ([09](./xxx-notes-011-v5-impl-09-deployment.md)) |
| platform-mesh-operator: profile validation | Refusing a profile whose `providerconnections` shard list doesn't match `infra`'s, instead of applying it | **Follow-up, not part of this delivery.** Doesn't exist yet and is broader than this plan, which only specifies the rule; until then a shard runbook covers it. Admission-time (needs a webhook, since CEL can't parse the YAML inside the profile ConfigMap) or reconcile-time with the reason on the `PlatformMesh` ([09](./xxx-notes-011-v5-impl-09-deployment.md)) |
| kcp-operator: cache-server credentials in a `VirtualWorkspace` | Connectors don't need them (v5 observation 7) | Mounted regardless; **accepted** and documented ([09](./xxx-notes-011-v5-impl-09-deployment.md)) |
| kcp-operator: mount the connector certificate | Connectors sign and handshake with it | **Supported** (`extraVolumes`) |
| kcp-operator: stable replica identities | Rendezvous hashing keeps connections across restarts | Deployment only; **accepted**: connections move on restart ([05](./xxx-notes-011-v5-impl-05-connector.md)) |
| kcp: maximal permission policy on writes through the APIExport virtual workspace | The operator writing records | Read from code, not tested (v5 observation 8); see [10 Testing](./xxx-notes-011-v5-impl-10-testing.md) |
| kcp-operator: `--service-account-lookup=false` | Not needed by v5 for remote Providers | Unchanged |

## Open questions across documents

Open:

- **Periodic secret rotation** (e.g. every 30 days). With the secret out of kcp and changeable only by PM, changes on demand and after tampering may be enough ([03](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)).

Deferred to later changes:

- **Owner replicas for Providers**: an opt-in SDK mode for horizontal scale, needing no PM-side work; this delivery runs one active replica ([07](./xxx-notes-011-v5-impl-07-provider-sdk.md)).
- **SDK administration commands and the PM-side administrator input** for recovery; the design supports them from the start ([03](./xxx-notes-011-v5-impl-03-providerconnections-operator.md), [07](./xxx-notes-011-v5-impl-07-provider-sdk.md)).
- **Deletion protection for `Provider`s with consumer bindings.** Deleting a `Provider` deletes its workspace and the APIExport's identity, which kcp can't re-pair with existing bindings; it's the one owner mistake recovery can't undo.
