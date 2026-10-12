# v5 implementation plan: deployment

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). How the PM-side pieces get into an installation: kcp bootstrap manifests, Helm charts in `pm-helm-charts`, kcp-operator and cert-manager resources, network, and rollout order.

## Principles

- **Declarative and GitOps-friendly**, like the rest of PM's deployment.
- **Per-shard pieces come from the one shard list PM already has**: the `PlatformMesh` profile's `kcp.shards` and root shard values, which the `infra` chart renders shards from. Adding a shard adds its connector and shard certificate, with no second list and no new trigger.
- **Least privilege per component**: the operator, connectors and cert-manager each see only what they need.

## kcp bootstrap

In platform-mesh-operator's kcp manifests (`manifests/kcp`), next to the existing providers export:
- **providers export**: add `provideronboardingrequests` and its `APIResourceSchema` ([02 APIs, "ProviderOnboardingRequest"](./xxx-notes-011-v5-impl-02-apis.md)).
- **In `:root:providers:system`** (the `03-providers` group): the `connections.platform-mesh.io` `APIResourceSchema` for `providerendpoints`, the `APIExport` with its permission claim and maximal permission policy, its `APIExportEndpointSlice`, and the policy's RBAC ([02 APIs, "Connection export"](./xxx-notes-011-v5-impl-02-apis.md)).
- **Access**: `apiexports/content` on the connection export for the operator (read and write) and the connectors' group (read); `bind` for the Provider controller's identity.

## Charts

**One new chart, `providerconnections`**, in `pm-helm-charts`, containing the operator and the per-shard connector resources. platform-mesh-operator deploys it as a HelmRelease like PM's other components, with values from the `PlatformMesh` profile.

**Shards**: the profile passes the chart the same `kcp.shards` list and root shard values it passes to `infra`. The chart renders one set of connector resources per entry, root shard included. Removing a shard from the list removes its connector with it.

**Derived hosts**: a shard's external hostname (`<name>.kcp.<external hostname>` by default, or its override) and in-cluster Service name are derived today inside the `infra` chart. That derivation moves into the existing `common` library chart, and both `infra` and `providerconnections` use it, so a shard's connector always targets the host its `Shard` was created with.

### providerconnections-operator

- **One Deployment in PM's runtime cluster**, several replicas, partitioned internally by kcp shard with per-shard Leases ([03 Operator, "Partitioning by shard"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)). Replica count is independent of the shard count; adding or removing shards needs no deployment change.
- RBAC in its namespace for its Leases.
- A kcp identity (kcp-operator `Kubeconfig`) with the rights listed in [03 Operator, "Identity and permissions"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md).
- Read access in the runtime cluster to the session CA generation declaration and public certificates ([04 Session CA, "Publishing generations"](./xxx-notes-011-v5-impl-04-session-ca.md)).
- Network policy: outbound to the front-proxy, to every shard's virtual-workspace host (from the providers and connection export endpoint slices), and to the internet (Providers), nothing else.

### Connector (providerconnections-vw)

- **One kcp-operator `VirtualWorkspace` per shard**, with `shardRef` (or the root shard reference), rendered by the `providerconnections` chart from the shard list. kcp-operator renders the Deployment, Service, serving certificate and kubeconfig identity (the pattern `kcp-access-vw` uses).
- **`kubeconfigSecretRef` always set**: without it, kcp-operator mounts its privileged logical-cluster-admin credential into the pod.
- **No front-proxy path mapping**: nobody reaches connectors inbound; they only dial out. The resource's `external` field is required, so it's set to the connector's in-cluster address; nothing routes to it.
- **Per-shard configuration**, all derived from the shard list through the shared helper: shard name, its virtual-workspace host, the front-proxy host, and the in-cluster address for each (see "Network").
- **Shard certificate**: one cert-manager `Certificate` per shard, rendered in the same loop, from the active session CA issuer (issuer setup is part of the open session CA configuration), mounted into that shard's connector through the `VirtualWorkspace`'s `extraVolumes` and `extraVolumeMounts` ([04 Session CA, "Shard certificates"](./xxx-notes-011-v5-impl-04-session-ca.md)).
- **The connectors' kcp identity**: one group shared by all shards, used in the connector RBAC the Provider controller creates in each provider workspace ([06 Provider controller, "Connector RBAC"](./xxx-notes-011-v5-impl-06-provider-controller.md)).
- **Resources**: requests and limits sized from the shard's expected watch count; anti-affinity from the shard's own pods so a burst can't starve the shard.

### Session CA

Templates for generations (`Certificate` with `rotationPolicy: Never` and `Issuer` per generation), the generation declaration, and per-shard `Certificate`s ([04 Session CA](./xxx-notes-011-v5-impl-04-session-ca.md)). Placed in a namespace whose Secrets only cert-manager reads.

## Network

- **Connectors**: egress to their shard's virtual workspace and the front-proxy in-cluster, and to the internet for Providers. **Map external kcp hosts to in-cluster Services** for the dial while keeping the TLS server name, so traffic doesn't hairpin through PM's load balancer ([05 Connector, "Proxying"](./xxx-notes-011-v5-impl-05-connector.md)).
- **Operator**: egress to the front-proxy and the internet.
- **No new inbound paths** into PM.
- **Multi-cluster installations**: PM's charts render all shards into one cluster today. If shards ever span clusters, each cluster's profile carries its own shard list, and the chart renders that cluster's connectors there; the operator stays in PM's cluster ([03 Operator, "Partitioning by shard"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)).

## Dependencies on kcp-operator

Checked against kcp-operator `8994ed7`.

| Need | Status |
|---|---|
| Mount the shard certificate Secret | **Supported**: `extraVolumes` and `extraVolumeMounts` |
| Keep the privileged fallback credential out | **Supported**: not mounted when `kubeconfigSecretRef` is set |
| Omit cache-server credentials | **Not supported**: mounted whenever a cache server is configured. **Accepted** and documented as part of a compromised connector's blast radius |
| Stable replica identities | **Not supported**: Deployment only. **Accepted**: connections move on restart ([05 Connector, "Replicas"](./xxx-notes-011-v5-impl-05-connector.md)) |
| No inbound address | `external` is required; set to the in-cluster address |

## Rollout order

1. kcp bootstrap (schemas, exports, RBAC) and the session CA's first generation.
2. platform-mesh-operator with the Provider controller changes ([06](./xxx-notes-011-v5-impl-06-provider-controller.md)): bindings and connector RBAC appear in remote Providers' workspaces.
3. The `providerconnections` chart: connectors on every shard, and the operator. Within the chart, connectors come first; the operator tolerates their absence either way.
4. Owners can now create `ProviderOnboardingRequest`s.

Upgrades keep this order. Connectors and the operator tolerate each other's absence: without the operator, no records appear; without connectors, readiness stays false.

## Local development

A kind setup with a multi-shard kcp, cert-manager, both operators, connectors, and the quickstart Provider in a separate network ([10 Testing, "E2E"](./xxx-notes-011-v5-impl-10-testing.md)). The same setup serves developers and CI.

## Decided

- **Per-shard connector resources** come from a new `providerconnections` chart that reads the same profile shard list as `infra`, with hostname derivation shared through the `common` library chart. No watcher of `Shard` resources.

## Open questions

To be revisited separately:

- **Where the session CA generation declaration lives** and who may change it ([04 Session CA, "Open questions"](./xxx-notes-011-v5-impl-04-session-ca.md)).
