# v5 implementation plan: overview

Working notes for [011](./011-provider-onboarding-flow-improvements.md). Not the RFC itself. This is the entry point to the implementation plan for the [v5 design](./xxx-notes-011-provider-onboarding-flow-improvements-v5.md), **with Inner TLS on Sessions as the only mode** (v5, "The Provider's Edge: Inner TLS on Sessions"). The plan is split into one document per component; each says what the component owns, what it relies on, and where the flow continues. It describes principles and responsibilities, not code.

Start here, then follow the walkthrough below, or jump to a component.

## Documents

| Document | What it covers |
|---|---|
| [00 Overview](./xxx-notes-011-v5-impl-00-overview.md) | Decisions, component map, end-to-end walkthrough, cross-cutting principles, ownership, build order |
| [01 Protocol](./xxx-notes-011-v5-impl-01-protocol.md) | The wire contract between PM and the Provider-side: endpoints, credentials, Inner TLS, multiplexer, status, versioning |
| [02 APIs](./xxx-notes-011-v5-impl-02-apis.md) | kcp-side types and exports: `ProviderOnboardingRequest`, `Provider`, `ProviderEndpointRegistration` and its `Secret`, `ProviderEndpoint`, the Connection Export and its maximal permission policy |
| [03 platform-mesh-operator](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md) | Reconciling local and remote `Provider`s and `ProviderOnboardingRequest`s; Registration, `ProviderEndpoint`s, health, Secret Changes, the Connector CA Set, deletion |
| [04 Connector CA](./xxx-notes-011-v5-impl-04-connector-ca.md) | The Connector CA managed by cert-manager, Connector Certificates, the Connector CA Set that Providers pin, and the automatic Rollover |
| [05 Connector](./xxx-notes-011-v5-impl-05-connector.md) | The per-shard Connector, a standalone proxy; Sessions, proxying, policy |
| [07 SDK](./xxx-notes-011-v5-impl-07-provider-sdk.md) | The Provider-side library: Connection API, its resources (`Connection`, `OnboardingToken`, `ServerKey`), Session server, transport, lifecycle |
| [08 Provider quickstart](./xxx-notes-011-v5-impl-08-provider-quickstart.md) | The reference Provider in pm-provider-quickstart |
| [09 Deployment](./xxx-notes-011-v5-impl-09-deployment.md) | Charts, kcp bootstrap manifests, kcp-operator and cert-manager resources, network, rollout |
| [10 Testing](./xxx-notes-011-v5-impl-10-testing.md) | Unit tests per component, multi-shard kind e2e, security tests; load tests as a follow-up |
| [Follow-ups](./xxx-notes-011-v5-impl-todo.md) | Work that can be done separately: cleanups, larger follow-ups, items outside this plan |

## Decisions taken

| Topic | Decision |
|---|---|
| Inner TLS on Sessions | **Required, the only mode.** Plain Sessions aren't implemented. |
| Delivery | **Single delivery**: each document describes its full scope; "Build order" below orders the work. |
| `platform-mesh-operator` | **`platform-mesh-operator`** owns each `Provider`, local or remote, end to end. It moves with the `Provider` part when that becomes its own operator; until then, internet-facing code in PM's most privileged component is an accepted interim risk ([03](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)). |
| `providerconnections-connector` | A new service: **`services/providerconnections-connector`**, a **standalone proxy server modelled on kcp's front-proxy**, one per shard, deployed as a plain Deployment with its identity from a kcp-operator `Kubeconfig`. |
| the SDK | **`platform-mesh/provider-sdk`**, module `go.platform-mesh.io/provider-sdk`, its own semver line released through `cmd/release` ([07](./xxx-notes-011-v5-impl-07-provider-sdk.md)). |
| Connection object type | **`providers.platform-mesh.io/ProviderEndpoint`**, served only by the Connection Export. `ProviderConnection` is removed. |
| Connection Export home | **`:root:providers:system`**. |
| Multiplexer | **yamux** over the WebSocket. |
| Connector CA | **Managed by cert-manager and rolled over automatically**: 1 year, renewed 90 days before expiry, self-signed by default or from an `issuerRef`. `platform-mesh-operator` pushes each new CA to every Provider before Connectors use it; a status-only **`ConnectorCASet`** reports progress ([04](./xxx-notes-011-v5-impl-04-connector-ca.md)). |
| Provider-side state | **Custom resources in the Provider's cluster** (`Connection`, `OnboardingToken`, `ServerKey` in `provider-sdk.platform-mesh.io`), the only source of truth; the only `Secret`s hold private keys, unless they're in a KMS or HSM. |
| Portal | **The Kubernetes API on the SDK's resources**, plus a token helper; the quickstart has its own small portal. |
| E2E | **kind, multi-shard**, with the Provider behind a network boundary, extending `platform-mesh-operator`'s `test/e2e/kind`. |
| Naming | `xxx-notes-011-v5-impl-*.md` in `rfc/`. |
| `platform-mesh-operator` replicas | **One active replica** by `platform-mesh-operator`'s leader election, with one multicluster-runtime provider per export; no partitioning by shard ([03](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)). |
| Chart | One new `providerconnections` chart (the Connector CA and per-shard Connector resources; the `ConnectorCASet` CRD ships with `platform-mesh-operator`), deployed by `platform-mesh-operator` as a profile service, with an explicit `kcpTopology` that duplicates `infra`'s shards (must match; profile validation enforcing it is a follow-up) ([09](./xxx-notes-011-v5-impl-09-deployment.md)). |
| Connection state | **A `ProviderEndpointRegistration` per Connection in `platform-mesh-operator`'s namespace** in PM's runtime cluster, authoritative, with a `Secret` holding only the Connection Secret; `ProviderEndpoint`s are written from them ([02](./xxx-notes-011-v5-impl-02-apis.md)). |
| `Secret` access | **Connection Secrets in `platform-mesh-operator`'s namespace** in PM's runtime cluster, never in Owners' workspaces; token `Secret`s read by name, uncached, with **no `Secret` informers in kcp** ([02](./xxx-notes-011-v5-impl-02-apis.md), [03](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)). |
| Secret Changes | **Only through the PM-written `ProviderEndpoint`** (`pendingSecretHash`), adopted by the Provider-side over a Session; no call changes a Connection Secret, so a thief can't lock PM out. Lost or rejected Connection Secrets are repaired automatically; changes happen on demand (an annotation) or after tampering, never periodically; a Connection the Provider-side no longer knows is replaced by Onboarding the same `Provider` again ([01](./xxx-notes-011-v5-impl-01-protocol.md), [03](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)). |
| Pinned Bundles | **Compare-and-swap** (`etag`, `If-Match`); an unexpired pinned CA that isn't one of PM's unexpired CAs is treated as tampering: PM restores the bundle and changes the Connection Secret. When the bundle changes, the Provider-side closes Sessions that no longer chain to it ([01](./xxx-notes-011-v5-impl-01-protocol.md), [03](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)). |
| Recovery | **Out-of-band, but always possible**: stable `connectionID`s, importing and re-pinning `Connection`s with `kubectl`, and administrator input for Base URL and Server Keys on the PM-side. Procedures aren't part of this plan ([01](./xxx-notes-011-v5-impl-01-protocol.md), [07](./xxx-notes-011-v5-impl-07-provider-sdk.md)). |
| Requests | Owners create them (one ClusterRole with `Provider`s), no creator checks; the oldest request per `Provider` wins; a request ends at Registration (`Pending → Registering → Registered`, or `Failed` before Registration succeeds), and everything after is reported on the `Provider`; spent token data deleted ([02](./xxx-notes-011-v5-impl-02-apis.md), [03](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)). |
| `Provider` status | One writer; one condition per reconcile step, with labelled messages; `phase` for both kinds: `Pending → Provisioning → Ready ⇄ Degraded`, and `Deleting`; CEL rejects local fields on Remote Providers ([02](./xxx-notes-011-v5-impl-02-apis.md), [03](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)). |
| Migration | None: no Remote Providers exist yet ([03](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)). |
| Protocol | Current and previous version supported; Sessions authenticated by the inner mutual TLS alone, with a Hello message; Streams carry HTTP from the first byte; Control Calls over HTTP/1.1; Registration is the only plain-HTTPS call ([01](./xxx-notes-011-v5-impl-01-protocol.md)). |
| Connector | Every replica serves every Connection its shard needs; no request timeouts of its own, limits per Connection; child workspaces denied; one identity group for all shards ([05](./xxx-notes-011-v5-impl-05-connector.md)). |
| SDK | One active replica by leader election (standbys answer `503` with `Retry-After: 5`), horizontal scale as a later opt-in; Kubernetes-API-first resources, Events instead of hooks, metrics for operators; Server Keys rotated only on demand, through `kubectl`; Go only; Pinned Bundles in PEM ([07](./xxx-notes-011-v5-impl-07-provider-sdk.md)). |
| Quickstart | Keeps kubeconfig mode alongside Connection mode ([08](./xxx-notes-011-v5-impl-08-provider-quickstart.md)). |
| API groups | **Two**: `providers.platform-mesh.io` for every PM-side kind (in kcp, through the Providers Export and the Connection Export; in PM's runtime cluster, as CRDs), and `provider-sdk.platform-mesh.io` for the SDK's kinds; `v1alpha1` ([02](./xxx-notes-011-v5-impl-02-apis.md)). |

## Component map

```
 owner's workspace :root:orgs:<org>:<user>                 PM runtime cluster
 ┌──────────────────────────────────────────┐              ┌──────────────────────────────────────────────┐
 │ ProviderOnboardingRequest                │   providers  │ platform-mesh-operator                       │
 │ Provider (spec.remote, status.connection)│◀── export ──▶│   reconciles Providers and onboarding        │
 │ token Secret                             │   (owner     │   requests (doc 03)                          │
 └──────────────────────────────────────────┘    side)     │   consumes: providers export,                │
                                                           │             connection export                │
                                                           │   ProviderEndpointRegistrations (own ns)     │
 :root:providers:system                                    │ connector CA (cert-manager, doc 04)          │
 ┌──────────────────────────────────────────┐              └──────────────┬───────────────────────────────┘
 │ connection export                        │                             │ connector certificates
 │   connections.providers.platform-mesh.io │                             │
 │   + maximal permission policy RBAC       │                             ▼
 │ providers export (unchanged home)        │              ┌──────────────────────────────────────────────┐
 └──────────────────────────────────────────┘              │ connector @ shard-A, -B, …                   │
                                                           │ (doc 05)  reads ProviderEndpoints + slices,  │
 provider workspace :root:providers:<name>-<owner cluster> │   mints SA tokens, proxies to kcp            │
 ┌──────────────────────────────────────────┐              └──────────────┬───────────────────────────────┘
 │ APIBinding to the connection export      │                             │ outbound WebSocket + inner TLS
 │ ProviderEndpoint (PM-written only)       │                             ▼   (protocol, doc 01)
 │ SA (admin), Provider's own exports/slices│              ┌──────────────────────────────────────────────┐
 └──────────────────────────────────────────┘              │ Provider runtime                             │
                                                           │   the SDK (doc 07): connection API,          │
                                                           │   session server, transport, its CRDs        │
                                                           │   Provider controllers + portal (doc 08)     │
                                                           └──────────────────────────────────────────────┘
```

## End-to-end walkthrough

Each step names the document and section that owns it; read them in this order to follow the flow.

1. **The Owner gets a token** from the Provider's portal. The token format and its single-use rules: [01 Protocol, "Onboarding Token"](./xxx-notes-011-v5-impl-01-protocol.md). Issuing and verifying: [07 SDK, "OnboardingToken"](./xxx-notes-011-v5-impl-07-provider-sdk.md).
2. **The Owner creates a remote `Provider` and a `ProviderOnboardingRequest`** referencing it and the token `Secret`, in either order: [02 APIs, "ProviderOnboardingRequest"](./xxx-notes-011-v5-impl-02-apis.md).
3. **`platform-mesh-operator` finds the `Provider`**, waiting until it exists: [03 Operator, "Onboarding requests"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md).
4. **`platform-mesh-operator` bootstraps the Provider Workspace** (workspace, SA, Connection Export binding, Connector RBAC), without a kubeconfig for Remote Providers: [03 Operator, "Reconciling Providers"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md).
5. **The request registers the Connection** with the Provider-side: generates the Connection Secret, sends its hash and the Connector CA certificates, receives the `connectionID` and Server Keys, and is done (`Registered`): [03 Operator, "Registration"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md); the Provider's side: [07 SDK, "Connection API"](./xxx-notes-011-v5-impl-07-provider-sdk.md); the CA certificates it sends: [04 Connector CA, "ConnectorCASet"](./xxx-notes-011-v5-impl-04-connector-ca.md).
6. **`platform-mesh-operator` makes a first Control Call and writes the `ProviderEndpoint`** into the Provider Workspace through the Connection Export: [03 Operator, "First Control Call and ProviderEndpoint" and "Writing ProviderEndpoints"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md); type and policy: [02 APIs, "ProviderEndpoint" and "Connection Export"](./xxx-notes-011-v5-impl-02-apis.md).
7. **Connectors discover the `ProviderEndpoint` and open Sessions** where needed: [05 Connector, "Discovery" and "Which Sessions"](./xxx-notes-011-v5-impl-05-connector.md); the Session handshake: [01 Protocol, "Sessions"](./xxx-notes-011-v5-impl-01-protocol.md); the Provider's side: [07 SDK, "Session server"](./xxx-notes-011-v5-impl-07-provider-sdk.md).
8. **The Provider's controllers reconcile through the Sessions**: transport and client configs in [07 SDK, "Transport" and "Connections for multicluster-runtime"](./xxx-notes-011-v5-impl-07-provider-sdk.md); the proxy on PM's side in [05 Connector, "Request pipeline"](./xxx-notes-011-v5-impl-05-connector.md); the reference wiring in [08 Quickstart](./xxx-notes-011-v5-impl-08-provider-quickstart.md).
9. **`platform-mesh-operator` confirms readiness** from the Provider-side's per-host status, and the `Provider` becomes `Ready`: [03 Operator, "Health"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md); [01 Protocol, "Status"](./xxx-notes-011-v5-impl-01-protocol.md).
10. **Steady state**: `platform-mesh-operator` polls health and changes the Connection Secret when needed, through the `ProviderEndpoint`; cert-manager renews Connector Certificates and, once a year, the Connector CA, which `platform-mesh-operator` pushes before Connectors switch ([04 Connector CA, "Rollover"](./xxx-notes-011-v5-impl-04-connector-ca.md)); Connectors refresh SA tokens: [03 Operator, "Secret Changes"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md), [01 Protocol, "Changing the Connection Secret"](./xxx-notes-011-v5-impl-01-protocol.md), [07 SDK, "The Connection's ProviderEndpoint"](./xxx-notes-011-v5-impl-07-provider-sdk.md), [04 Connector CA, "Connector Certificates"](./xxx-notes-011-v5-impl-04-connector-ca.md), [05 Connector, "Identity towards kcp"](./xxx-notes-011-v5-impl-05-connector.md).
11. **Rare operations**: Connection Secret repair and Onboarding a `Provider` again ([01 Protocol, "Recovery"](./xxx-notes-011-v5-impl-01-protocol.md), [03 Operator, "Recovery"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)); out-of-band administration ([07 SDK, "Administration"](./xxx-notes-011-v5-impl-07-provider-sdk.md)); Server Key rotation ([07 SDK, "Server Keys"](./xxx-notes-011-v5-impl-07-provider-sdk.md)).
12. **Offboarding**: one finalizer deletes the `ProviderEndpoint` (Sessions close), calls `DELETE`, deletes the `ProviderEndpointRegistration` and its `Secret`, then tears down the Connector RBAC, binding, SA and workspace: [03 Operator, "Deletion"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md).

Deployment of all PM-side pieces: [09 Deployment](./xxx-notes-011-v5-impl-09-deployment.md). How each step is tested: [10 Testing](./xxx-notes-011-v5-impl-10-testing.md).

## Cross-cutting principles

Every component follows these; the component documents refer back here instead of repeating them.

1. **No kcp credential leaves the PM-side.** The Provider-side never receives a token, kubeconfig or kcp CA bundle. Only Connectors hold SA tokens, in memory.
2. **Authority only through authenticated, end-to-end encrypted Sessions.** The Provider's Edge sees opaque bytes on Sessions and Control Sessions. Nothing that grants anything is sent on plain HTTPS except the Onboarding Token (single-use) and public material.
3. **`ProviderEndpoint`s are PM-written.** `ProviderEndpoint` objects are written only by `platform-mesh-operator`, through the Connection Export; kcp enforces it with the export's maximal permission policy. Connectors only read them.
4. **Never trust Owner-writable fields for links.** The link between a `Provider` and its Provider Workspace, SA and Connection is derived from PM-controlled data (names kcp sets, objects in `:root:providers`), never from fields the Owner can edit (`spec`, `status` or annotations of the `Provider`).
5. **Connectors are stateless.** They hold no persistent state, write nothing to kcp except TokenRequests, and learn everything else from their deployment and the Connection Export.
6. **`platform-mesh-operator` outages don't stop traffic.** Nothing `platform-mesh-operator` does is on the Session path. The one exception: a `platform-mesh-operator` outage of more than about 12 hours right when cert-manager renews the Connector CA (once a year) stops new Sessions until `platform-mesh-operator` pushes the new CA; open Sessions keep running ([04 Connector CA, "Rollover"](./xxx-notes-011-v5-impl-04-connector-ca.md)).
7. **Write before send.** Every generated secret or key is persisted before it's sent anywhere, so a crash can always resume.
8. **Everything is idempotent and resumable.** Reconcilers converge from any intermediate state; the Provider-side is the source of truth for what it has pinned, and `platform-mesh-operator` reads it back.
9. **Owners learn nothing about PM's topology.** Owners see aggregate conditions only; shards, hosts and per-shard detail stay in metrics and PM-internal objects.
10. **One mode, versioned protocol.** Inner TLS is always on; the protocol carries an explicit version so SDK and PM can evolve independently ([01 Protocol, "Versioning"](./xxx-notes-011-v5-impl-01-protocol.md)).
11. **Signals go to the audience that can act on them.** Owners get conditions and events on their own objects; the PM Administrators get metrics, the `ConnectorCASet` status and documented alert rules; Providers get SDK notifications. PM has no alerting pipeline today, so the plan ships signals and documented rules, not alerts.
12. **Secrets never appear in logs, events, metrics, conditions or audit bodies.** Prefixed formats (`pm1.`, `pmc1_`) make leaks detectable by scanners.

## Who writes what

Clear ownership avoids two controllers fighting over the same fields.

| Object or field | Written by | Read by |
|---|---|---|
| `ProviderOnboardingRequest.spec` | Owner | `platform-mesh-operator` |
| token `Secret` | Owner; `platform-mesh-operator` deletes its token data once spent | `platform-mesh-operator` |
| `ProviderOnboardingRequest.status` | `platform-mesh-operator` | Owner |
| `Provider.spec`, including `caBundle` | Owner; the rotation annotation, Owner or PM Administrators | `platform-mesh-operator` |
| `Provider.status`, all of it | `platform-mesh-operator` | Owner |
| `ProviderEndpointRegistration` (`platform-mesh-operator`'s namespace, runtime cluster) | `platform-mesh-operator`; PM Administrators for recovery | `platform-mesh-operator`, PM Administrators |
| its `Secret` (Connection Secret) | `platform-mesh-operator` | `platform-mesh-operator` only |
| Provider Workspace, SA, binding, Connector RBAC | `platform-mesh-operator` | Connectors |
| `ProviderEndpoint`, including `pendingSecretHash` | `platform-mesh-operator` | Connectors, the SDK (over Sessions) |
| Connector CA, Connector Certificates | cert-manager | `platform-mesh-operator` (public certificates, from `CertificateRequest`s), Connectors (their own Connector Certificate) |
| Connector CA Set, `ConnectorCASet` status | `platform-mesh-operator` | Connectors (switch rule), PM Administrators |
| SA tokens | Connectors (TokenRequest) | Connectors only |
| a Connection's Provider-side state | the SDK | the SDK, portal |


## Build order

Single delivery, but the pieces depend on each other. Suggested order, with what each step unblocks:

1. **Protocol and APIs** ([01](./xxx-notes-011-v5-impl-01-protocol.md), [02](./xxx-notes-011-v5-impl-02-apis.md)): the contract everyone builds against. Includes protocol test vectors.
2. **SDK core** ([07](./xxx-notes-011-v5-impl-07-provider-sdk.md)): Connection API, its resources and controllers, Session server, transport. Can be tested against a fake PM-side.
3. **Connector CA** ([04](./xxx-notes-011-v5-impl-04-connector-ca.md)) and **kcp bootstrap** of the Connection Export ([09](./xxx-notes-011-v5-impl-09-deployment.md)).
4. **`platform-mesh-operator`** ([03](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)) and **Connector** ([05](./xxx-notes-011-v5-impl-05-connector.md)), in parallel; they meet only at the `ProviderEndpoint` and the protocol.
5. **Quickstart** ([08](./xxx-notes-011-v5-impl-08-provider-quickstart.md)).
6. **Deployment** ([09](./xxx-notes-011-v5-impl-09-deployment.md)) and **e2e and security tests** ([10](./xxx-notes-011-v5-impl-10-testing.md)).

## Deployment target

The plan targets PM's current deployment: `platform-mesh-operator` rendering `pm-helm-charts` from the `PlatformMesh` profile. The newer, not yet released platform-mesh-deployer (`OCMModule`s with `root-shard` and `per-shard` placements) is a later move; the components map onto it without design changes, but the deployer needs three extensions first ([follow-ups](./xxx-notes-011-v5-impl-todo.md)).

## External dependencies

| Dependency | Needed for | Status |
|---|---|---|
| Alerting in PM | Turning the documented alert rules (Connector CA renewal, Connector Certificates, convergence) into notifications for the PM Administrators | **Outside this plan.** PM collects metrics but has no alerting pipeline (Alertmanager disabled, no rules shipped). This plan ships metrics, `ServiceMonitor`s, documented rules and an optional `PrometheusRule` ([09](./xxx-notes-011-v5-impl-09-deployment.md)) |
| `platform-mesh-operator`: profile validation | Refusing a profile whose `providerconnections` shard list doesn't match `infra`'s, instead of applying it | **Follow-up, not part of this delivery.** Doesn't exist yet and is broader than this plan, which only specifies the rule; until then a shard runbook covers it. Admission-time (needs a webhook, since CEL can't parse the YAML inside the profile ConfigMap) or reconcile-time with the reason on the `PlatformMesh` ([09](./xxx-notes-011-v5-impl-09-deployment.md)) |
| kcp-operator: `Kubeconfig` | The Connectors' kcp identity | **Supported** ([09](./xxx-notes-011-v5-impl-09-deployment.md)) |
| kcp: maximal permission policy on writes through the APIExport virtual workspace | `platform-mesh-operator` writing `ProviderEndpoint`s | Read from code, not tested (v5 observation 8); see [10 Testing](./xxx-notes-011-v5-impl-10-testing.md) |
| kcp-operator: `--service-account-lookup=false` | Not needed by v5 for Remote Providers | Unchanged |

## Open questions across documents

Nothing is open. Work that can be done separately, before, alongside or after this plan is listed in the [follow-ups](./xxx-notes-011-v5-impl-todo.md).
