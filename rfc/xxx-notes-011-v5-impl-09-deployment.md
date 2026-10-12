# v5 implementation plan: deployment

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). How the PM-side pieces get into an installation: kcp bootstrap manifests, Helm charts in `pm-helm-charts`, kcp-operator and cert-manager resources, network, and rollout order.

## Principles

- **Declarative and GitOps-friendly**, like the rest of PM's deployment.
- **Per-shard pieces come from an explicit topology in the chart's values**, a deliberate duplicate of the shards in the `infra` service's values, shaped like the per-instance context platform-mesh-deployer will provide later. Drift between the two is detected, not prevented (see "Charts").
- **Least privilege per component**: the operator, connectors and cert-manager each see only what they need.

## kcp bootstrap

In platform-mesh-operator's kcp manifests (`manifests/kcp`), next to the existing providers export:
- **providers export**: add `provideronboardingrequests` and its `APIResourceSchema` ([02 APIs, "ProviderOnboardingRequest"](./xxx-notes-011-v5-impl-02-apis.md)).
- **In `:root:providers:system`** (the `03-providers` group): the `connections.platform-mesh.io` `APIResourceSchema` for `providerendpoints`, the `APIExport` with its permission claim and maximal permission policy, its `APIExportEndpointSlice`, and the policy's RBAC ([02 APIs, "Connection export"](./xxx-notes-011-v5-impl-02-apis.md)).
- **Access**: `apiexports/content` on the connection export for the operator (read and write) and the connectors' group (read); `bind` for the Provider controller's identity.

## Charts

**One new chart, `providerconnections`**, in `pm-helm-charts`, containing the operator and the per-shard connector resources. platform-mesh-operator deploys it as a HelmRelease like PM's other components, with values from the `PlatformMesh` profile.

**How platform-mesh-operator deploys it**: the `PlatformMesh` resource points at a profile ConfigMap (`profile.yaml`). For each enabled service in `components.services`, platform-mesh-operator renders a Flux `HelmRelease` (or an Argo CD `Application`) with the chart source, the service's `values`, its target namespace and `dependsOn`; Flux or Argo then render and install the chart. Profile values may use the operator's template variables (`{{ .baseDomain }}`, cluster IPs, `{{ .internalFrontProxyUrl }}`), which are scalar strings substituted as text. `providerconnections` becomes one more service, with `dependsOn: infra`. Every change, a connector CA rollover step included, is an edit to the profile.

**Shards: a duplicated list, shaped for the deployer.** Each service's values are separate, and platform-mesh-operator's template variables don't include the shard list, which only exists in the `infra` service's values (`kcp.shards`, `kcp.rootShard`). Rather than changing platform-mesh-operator for an interim step, the `providerconnections` service's values carry their own explicit topology:

```yaml
kcpTopology:
  frontProxy:
    host: kcp.api.{{ .baseDomain }}
    internalAddress: frontproxy-front-proxy.kcp-system.svc:8443
  shards:                                      # root shard included
    - name: root
      virtualWorkspaceHost: root.kcp.{{ .baseDomain }}
      internalAddress: root-kcp.kcp-system.svc:6443
    - name: nereus
      virtualWorkspaceHost: nereus.kcp.{{ .baseDomain }}
      internalAddress: nereus-shard-kcp.kcp-system.svc:6443
```

- **Hosts are explicit**, not derived: no shared hostname logic between charts.
- **The shape matches what platform-mesh-deployer will give each per-shard instance** (see "Later: moving to platform-mesh-deployer"), so the chart's per-shard block reads one entry the way a per-shard component will read its context. On the move, only the source of these values changes.
- **The chart renders one set of connector resources per entry.** Removing an entry removes that shard's connector.
- **Mismatches will be refused, not detected**: the shards in `kcpTopology` must match the `infra` service's root shard and `kcp.shards` by name. That's a rule for **platform-mesh-operator's profile validation**, which will refuse a profile that breaks it and report why on the `PlatformMesh`, instead of applying it. Profile validation doesn't exist yet and is **a follow-up, not part of this delivery**; this plan only specifies the rule. No component of this plan checks for drift at runtime.
- **Until the follow-up lands**, a mismatch isn't caught: a shard missing from `kcpTopology` simply has no connector, which shows only as `SessionsConnected=False` on `Provider`s once consumers bind there. The runbook for adding or removing a shard therefore says to change both the `infra` and the `providerconnections` values in the same profile change.

### providerconnections-operator

- **One Deployment in PM's runtime cluster**, several replicas, partitioned internally by kcp shard with per-shard Leases ([03 Operator, "Partitioning by shard"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)). Replica count is independent of the shard count; adding or removing shards needs no deployment change.
- RBAC in its namespace for its Leases.
- A kcp identity (kcp-operator `Kubeconfig`) with the rights listed in [03 Operator, "Identity and permissions"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md).
- In the runtime cluster: read the `ConnectorCABundle` and write its status, read connector `Certificate` objects (not their Secrets) ([04 Connector CA, "ConnectorCABundle"](./xxx-notes-011-v5-impl-04-connector-ca.md)).
- Network policy: outbound to the front-proxy, to every shard's virtual-workspace host (from the providers and connection export endpoint slices), and to the internet (Providers), nothing else.

### Connector (providerconnections-vw)

- **One kcp-operator `VirtualWorkspace` per shard**, with `shardRef` (or the root shard reference), rendered by the `providerconnections` chart, one per `kcpTopology` entry. kcp-operator renders the Deployment, Service, serving certificate and kubeconfig identity (the pattern `kcp-access-vw` uses).
- **`kubeconfigSecretRef` always set**: without it, kcp-operator mounts its privileged logical-cluster-admin credential into the pod.
- **No front-proxy path mapping**: nobody reaches connectors inbound; they only dial out. The resource's `external` field is required, so it's set to the connector's in-cluster address; nothing routes to it.
- **Per-shard configuration**, from its `kcpTopology` entry: shard name, its virtual-workspace host, the front-proxy host, and the in-cluster address for each (see "Network").
- **Connector certificate**: one cert-manager `Certificate` per shard, rendered in the same loop, from the active connector CA issuer (issuer setup is part of the open connector CA configuration), mounted into that shard's connector through the `VirtualWorkspace`'s `extraVolumes` and `extraVolumeMounts` ([04 Connector CA, "Connector certificates"](./xxx-notes-011-v5-impl-04-connector-ca.md)).
- **The connectors' kcp identity**: one group shared by all shards, used in the connector RBAC the Provider controller creates in each provider workspace ([06 Provider controller, "Connector RBAC"](./xxx-notes-011-v5-impl-06-provider-controller.md)).
- **Resources**: requests and limits sized from the shard's expected watch count; anti-affinity from the shard's own pods so a burst can't starve the shard.

### Connector CA

In the `providerconnections` chart ([04 Connector CA](./xxx-notes-011-v5-impl-04-connector-ca.md)):
- **The `ConnectorCABundle` CRD** and its singleton, rendered from values: generations with state and public certificate. Key sources (`caSecretRef`, `issuerRef`) stay in values and never reach the resource.
- **A cert-manager CA issuer per `caSecretRef` generation** in state `Next` or `Active`; `issuerRef` generations use the referenced issuer directly. Key Secrets sit in a namespace only cert-manager reads.
- **Per-shard connector `Certificate`s** from the `Active` generation's issuer.
- **Development only**: a self-signed generation created by cert-manager, behind an explicit flag.

## Observability

- **Today in PM**: the `observability` chart runs an OpenTelemetry collector that scrapes `ServiceMonitor`s (kcp, account-operator, security-operator, OpenFGA) into a bundled Prometheus or an external Prometheus-compatible endpoint. There's **no alerting**: Alertmanager and kube-state-metrics are disabled, no chart ships alert rules, and the bundled Prometheus (the community chart, not prometheus-operator) wouldn't read `PrometheusRule`s.
- **The `providerconnections` chart ships**: metrics endpoints on the operator and the connectors, and `ServiceMonitor`s for them, so the existing collector picks them up.
- **Alert rules are documented**, in the runbook, with expressions and severities ([04 Connector CA, "Signals"](./xxx-notes-011-v5-impl-04-connector-ca.md)). The chart can also render them as a `PrometheusRule` behind a values flag, off by default, for installations running prometheus-operator.
- **Delivering alerts** (Alertmanager, routing, on-call) is a platform-wide gap outside this plan (see the overview's external dependencies).

## Network

- **Connectors**: egress to their shard's virtual workspace and the front-proxy in-cluster, and to the internet for Providers. **Map external kcp hosts to in-cluster Services** for the dial while keeping the TLS server name, so traffic doesn't hairpin through PM's load balancer ([05 Connector, "Proxying"](./xxx-notes-011-v5-impl-05-connector.md)).
- **Operator**: egress to the front-proxy and the internet.
- **No new inbound paths** into PM.
- **Multi-cluster installations**: PM's charts render all shards into one cluster today. If shards ever span clusters, each cluster's profile carries its own `kcpTopology`, and the chart renders that cluster's connectors there; the operator stays in PM's cluster ([03 Operator, "Partitioning by shard"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)).

## Dependencies on kcp-operator

Checked against kcp-operator `8994ed7`.

| Need | Status |
|---|---|
| Mount the connector certificate Secret | **Supported**: `extraVolumes` and `extraVolumeMounts` |
| Keep the privileged fallback credential out | **Supported**: not mounted when `kubeconfigSecretRef` is set |
| Omit cache-server credentials | **Not supported**: mounted whenever a cache server is configured. **Accepted** and documented as part of a compromised connector's blast radius |
| Stable replica identities | **Not supported**: Deployment only. **Accepted**: connections move on restart ([05 Connector, "Replicas"](./xxx-notes-011-v5-impl-05-connector.md)) |
| No inbound address | `external` is required; set to the in-cluster address |

## Rollout order

1. kcp bootstrap (schemas, exports, RBAC) and the connector CA's first generation.
2. platform-mesh-operator with the Provider controller changes ([06](./xxx-notes-011-v5-impl-06-provider-controller.md)): bindings and connector RBAC appear in remote Providers' workspaces.
3. The `providerconnections` chart: connectors on every shard, and the operator. Within the chart, connectors come first; the operator tolerates their absence either way.
4. Owners can now create `ProviderOnboardingRequest`s.

Upgrades keep this order. Connectors and the operator tolerate each other's absence: without the operator, no records appear; without connectors, readiness stays false.

## Local development

A kind setup with a multi-shard kcp, cert-manager, both operators, connectors, and the quickstart Provider in a separate network ([10 Testing, "E2E"](./xxx-notes-011-v5-impl-10-testing.md)). The same setup serves developers and CI.

## Later: moving to platform-mesh-deployer

This plan targets how PM is deployed today: platform-mesh-operator (released, used by installations) renders the `pm-helm-charts` charts from the `PlatformMesh` profile. The monorepo also has a newer deployer, `operators/platform-mesh-deployer` (API in `apis/deploy/v1alpha1`): it builds kcp's topology from `deploy.platform-mesh.io/PlatformMesh` (shard groups, one shard per matched cluster, embedded kcp-operator controllers) and deploys components as `OCMModule`s with per-component placements. As of 2026-10-12 it's unreleased, used only for kcp in the monorepo's Tilt dev environment, and no component is deployed as a module yet.

The components here already split along the deployer's placements, so moving later changes packaging, not design:

| Piece | Now (this plan) | On the deployer |
|---|---|---|
| Operator | Deployment in PM's cluster, from the `providerconnections` chart | An `OCMModule` component with `root-shard` placement and a minted front-proxy kubeconfig. Partitioning by shard is unchanged |
| Connectors | kcp-operator `VirtualWorkspace` per shard, from `kcpTopology` | A `per-shard` component: one instance per shard in that shard's cluster, following the deployer's topology. Can be a plain workload (still on the virtual workspace framework) with a minted kubeconfig, which drops kcp-operator's `VirtualWorkspace` and its accepted limitations (cache-server credentials, Deployment only) |
| kcp bootstrap | platform-mesh-operator's kcp manifests; connection export in `:root:providers:system` | Module setup: schemas, connection export, endpoint slice and policy RBAC in the module's workspace `root:modules:providerconnections`. The export's home moves, so records and bindings must be migrated or recreated |
| Shard list | `kcpTopology` in the service's values, duplicating `infra`'s shards, must match `infra`'s (profile validation, a follow-up) | The deployer's topology; nothing to maintain. The per-shard context replaces `kcpTopology` entries field for field |
| Connector CA | Issuers and key sources in PM's cluster, from chart values | Module `values`. Keys and issuers stay on the management cluster; connector certificates are minted there and synced to each shard's cluster, so the CA key never leaves the management cluster |
| Identities | kcp-operator `Kubeconfig`s with the groups and rights this plan lists | Minted users `module:providerconnections:<kubeconfig>`, cluster-admin only in the module workspace. Every right outside it (the providers export's content, `Workspace`s in `:root:providers`, TokenRequest in provider workspaces) is granted elsewhere, e.g. by the Provider controller to that username instead of a group |

**Gaps to close in the deployer first:**
1. **Per-shard instances need their hosts**: the shard's URL and virtual-workspace URL, the front-proxy URL, and in-cluster addresses. The deployer computes them from its exposure templates but doesn't pass them to module instances yet.
2. **Client certificates per instance** from a configurable issuer (the connector CA's active generation), with chosen key usages, as it already mints serving certificates.
3. **Rights outside the module workspace** for minted identities, or a documented place where other components grant them.

Nothing in the protocol, the APIs other than the export's home, the operator's or the connector's logic, or the SDK depends on how PM is deployed.

## Decided

- **Per-shard connector resources** come from a new `providerconnections` chart, deployed as a profile service, from an explicit `kcpTopology` in its values that duplicates `infra`'s shards, shaped like the deployer's per-instance context. The two must match; platform-mesh-operator's profile validation will enforce it as a follow-up, and the shard runbook covers it until then. No watcher of `Shard` resources and no runtime drift detection.
- **Connector CA**: the `ConnectorCABundle` CR rendered by the chart from values; changes go through the `PlatformMesh` profile like any other configuration ([04 Connector CA](./xxx-notes-011-v5-impl-04-connector-ca.md)).
