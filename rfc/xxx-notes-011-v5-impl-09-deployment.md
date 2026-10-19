# v5 implementation plan: deployment

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). How the PM-side pieces get into an installation: kcp bootstrap manifests, Helm charts in `pm-helm-charts`, kcp-operator and cert-manager resources, network, and rollout order.

## Principles

- **Declarative and GitOps-friendly**, like the rest of PM's deployment.
- **Per-shard pieces come from an explicit topology in the chart's values**, a deliberate duplicate of the shards in the `infra` service's values, shaped like the per-instance context platform-mesh-deployer will provide later. The two must match (see "Charts").
- **Least privilege per component**: `platform-mesh-operator`, Connectors and cert-manager each get only what they need.

## kcp bootstrap

In `platform-mesh-operator`'s kcp manifests (`manifests/kcp`), next to the existing Providers Export:
- **Providers Export**: add `provideronboardingrequests` and its `APIResourceSchema` ([02 APIs, "ProviderOnboardingRequest"](./xxx-notes-011-v5-impl-02-apis.md)).
- **In `:root:providers:system`** (the `03-providers` group): the `APIResourceSchema` for `providerendpoints` in `providers.platform-mesh.io`, the `APIExport` `connections.providers.platform-mesh.io` with its permission claim and maximal permission policy, its `APIExportEndpointSlice`, and the policy's RBAC ([02 APIs, "Connection Export"](./xxx-notes-011-v5-impl-02-apis.md)).
- **Access** in `:root:providers:system`, for the Connectors' group and `platform-mesh-operator`: `access` on `/`; `get`, `list`, `watch` on the Connection Export's `APIExportEndpointSlice`; `apiexports/content` on the Connection Export, read for the Connectors and read and write for `platform-mesh-operator`; `bind` on it for `platform-mesh-operator` ([02 APIs, "Connection Export"](./xxx-notes-011-v5-impl-02-apis.md)).

## Charts

**One new chart, `providerconnections`**, in `pm-helm-charts`, containing the Connector CA and the per-shard Connector resources. The work for Remote Providers needs no chart of its own: it ships inside `platform-mesh-operator` (see "`platform-mesh-operator`"). `platform-mesh-operator` deploys the `providerconnections` chart as a HelmRelease like PM's other components, with values from the `PlatformMesh` profile.

**How `platform-mesh-operator` deploys it**: the `PlatformMesh` resource points at a profile ConfigMap (`profile.yaml`). For each enabled service in `components.services`, `platform-mesh-operator` renders a Flux `HelmRelease` (or an Argo CD `Application`) with the chart source, the service's `values`, its target namespace and `dependsOn`; Flux or Argo then render and install the chart. Profile values may use the operator's template variables (`{{ .baseDomain }}`, cluster IPs, `{{ .internalFrontProxyUrl }}`), which are scalar strings substituted as text. `providerconnections` becomes one more service, with `dependsOn: infra`. Every change to it, such as adding a shard, is an edit to the profile; Rollovers and distrusting a CA need none.

This targets PM's deployment today. Nothing in the protocol, the APIs other than the Connection Export's home, or the components' logic depends on it; moving to platform-mesh-deployer is a [follow-up](./xxx-notes-011-v5-impl-todo.md).

**Shards: a duplicated list, shaped for the deployer.** Each service's values are separate, and `platform-mesh-operator`'s template variables don't include the shard list, which only exists in the `infra` service's values (`kcp.shards`, `kcp.rootShard`). Rather than changing `platform-mesh-operator` for an interim step, the `providerconnections` service's values carry their own explicit topology:

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
- **The shape matches what platform-mesh-deployer will give each per-shard instance** ([follow-ups](./xxx-notes-011-v5-impl-todo.md)), so the chart's per-shard block reads one entry the way a per-shard component will read its context. On the move, only the source of these values changes.
- **The chart renders one set of Connector resources per entry.** Removing an entry removes that shard's Connector.
- **Must match `infra`'s shards by name** (its root shard and `kcp.shards`). `platform-mesh-operator`'s profile validation will refuse a mismatched profile, a [follow-up](./xxx-notes-011-v5-impl-todo.md). Until then, the runbook for adding or removing a shard changes both in the same profile change; a shard missing from `kcpTopology` has no Connector: `Provider`s hosted on that shard show `Connected=False` with `NoSessions` from Onboarding on, and other Connections do too once consumers bind there.

### `platform-mesh-operator`

`platform-mesh-operator` ([03 Operator](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)) runs as its own Deployment, with leader election (one active replica). For Remote Providers it needs:
- **The `ProviderEndpointRegistration` and `ConnectorCASet` CRDs**, in `providers.platform-mesh.io`, shipped with `platform-mesh-operator`'s chart; `platform-mesh-operator` creates the `ConnectorCASet` object `default` when it's missing ([02 APIs, "ProviderEndpointRegistration"](./xxx-notes-011-v5-impl-02-apis.md)).
- **RBAC in its namespace** for `ProviderEndpointRegistration`s and their `Secret`s. That namespace is separate from cert-manager's, which holds the Connector CA's key. PM Administrators get `update` on `providerendpointregistrations` there, for recovery and Base URL changes, without access to the `Secret`s, and `update` on the `ConnectorCASet`'s `spec`, for `distrust`.
- **kcp rights** listed in [03 Operator, "Identity and permissions"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md): the Connection Export's slice, content and `bind`, and the policies and Connector RBAC in Provider Workspaces.
- **In the runtime cluster**: create the `ConnectorCASet` object `default`, read its `spec` and write its status; a Role in cert-manager's cluster resource namespace with only `get`, `list` and `watch` on `certificaterequests`, for the CA's public certificates, and the same in the Connectors' namespace, for which CA each Connector Certificate chains to ([04 Connector CA, "The Connector CA Set"](./xxx-notes-011-v5-impl-04-connector-ca.md)). No access to `Secret`s there.
- **Egress** to Providers' Base URLs, within whatever the deployment allows (see "Network").
- **Configuration**: its own kcp username, which it writes into the policies' `matchConditions` in Provider Workspaces ([02 APIs, "Policies in Provider Workspaces"](./xxx-notes-011-v5-impl-02-apis.md)).

### Connector

- **One Deployment per shard**, rendered by the `providerconnections` chart, one per `kcpTopology` entry, running the `providerconnections-connector` image ([05 Connector](./xxx-notes-011-v5-impl-05-connector.md)). No Service for traffic and no serving certificate: nothing reaches Connectors inbound; they only dial out. The ops endpoints (`/metrics`, health) are cluster-internal, behind a Service for the ops port only, which the `ServiceMonitor` selects.
- **Its kcp identity from a kcp-operator `Kubeconfig`** per shard, rendered in the same loop: a client certificate for the Connectors' group, targeting the front-proxy (`frontProxyRef`), written to a `Secret` in the namespace of the kcp-operator objects, where the Connector runs. kcp-operator regenerates it before it expires; the Connector reloads it when the `Secret` changes. The group is the one `platform-mesh-operator` uses in the Connector RBAC in each Provider Workspace ([03 Operator, "Connector RBAC"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)). **Assumption**: the certificate is issued by the RootShard's client CA (kcp-operator does so for a `frontProxyRef` target, `pkg/controller/kubeconfig/controller.go`), and every shard trusts that CA (`internal/resources/shard/ca_bundle.go`), so it's also accepted directly at each shard's virtual-workspace URL. A deployment that gives the front-proxy a separate client CA (`clientCABundleRef`) breaks this; the e2e checks it ([10 Testing](./xxx-notes-011-v5-impl-10-testing.md)).
- **Per-shard configuration**, from its `kcpTopology` entry: shard name, its virtual-workspace host, the front-proxy host, and the in-cluster address for each (see "Network"). The virtual-workspace host must be the one kcp writes into endpoint slices (the shard's `virtualWorkspaceURL`): the Connector recognises Provider Workspaces hosted on its shard by it ([05 Connector, "Discovery"](./xxx-notes-011-v5-impl-05-connector.md)).
- **Connector Certificate**: one cert-manager `Certificate` per shard, rendered in the same loop, from the Connector CA's `ClusterIssuer` (see "Connector CA" below), with a common label for renewing them all at once, mounted into that shard's Connector as an ordinary volume ([04 Connector CA, "Connector Certificates"](./xxx-notes-011-v5-impl-04-connector-ca.md)).
- **Replicas, resources, anti-affinity** set directly on the Deployment: requests and limits sized from the shard's expected watch count; anti-affinity from the shard's own pods so a burst can't starve the shard; a PodDisruptionBudget so rollouts keep at least one replica.

### Connector CA

In the `providerconnections` chart ([04 Connector CA](./xxx-notes-011-v5-impl-04-connector-ca.md)):
- **The CA `Certificate`** (`isCA: true`, 1 year, renewed 90 days before expiry, `rotationPolicy: Always`, `revisionHistoryLimit: 10`) in cert-manager's cluster resource namespace, taken from values (default `cert-manager`), issued by a self-signed issuer by default or by the `issuerRef` from values (e.g. Vault or a cloud CA).
- **A `ClusterIssuer` of type CA** on the CA's `Secret`, issuing the Connector Certificates.
- **Per-shard Connector `Certificate`s** from that `ClusterIssuer`, with a common label.
- **Not the `ConnectorCASet` CRD**: it ships with `platform-mesh-operator`, which writes its status (see "`platform-mesh-operator`"). Distrusting a CA after a compromise is an edit to the `ConnectorCASet`'s `spec.distrust` by PM Administrators, not a chart value ([04 Connector CA, "Compromise response"](./xxx-notes-011-v5-impl-04-connector-ca.md)).

The same templates serve development and production; nothing about the CA has to be prepared outside the chart.

## Observability

- **Today in PM**: the `observability` chart runs an OpenTelemetry collector that scrapes `ServiceMonitor`s (kcp, account-operator, security-operator, OpenFGA) into a bundled Prometheus or an external Prometheus-compatible endpoint. There's **no alerting**: Alertmanager and kube-state-metrics are disabled, no chart ships alert rules, and the bundled Prometheus (the community chart, not prometheus-operator) wouldn't read `PrometheusRule`s.
- **The `providerconnections` chart ships**: metrics endpoints on the Connectors' ops listener, and `ServiceMonitor`s for them, so the existing collector picks them up. `platform-mesh-operator`'s metrics come from its own metrics endpoint.
- **Alert rules are documented**, in the runbook, with expressions and severities ([04 Connector CA, "Signals"](./xxx-notes-011-v5-impl-04-connector-ca.md)). The chart can also render them as a `PrometheusRule` behind a values flag, off by default, for installations running prometheus-operator.
- **Delivering alerts** (Alertmanager, routing, on-call) is a platform-wide gap outside this plan (see the overview's external dependencies).

## Network

- **Connectors**: egress to their shard's virtual workspace and the front-proxy in-cluster, and to Providers. **Dial kcp at its in-cluster Services**, sending the upstream's real host and TLS server name from configuration, so traffic doesn't hairpin through PM's load balancer ([05 Connector, "Request pipeline"](./xxx-notes-011-v5-impl-05-connector.md)).
- **`platform-mesh-operator`**: additionally, egress to Providers.
- **Direct egress to Providers is assumed.** Egress through an HTTP proxy isn't supported; every transport in the Connectors and `platform-mesh-operator` sets `Proxy` explicitly, so `HTTPS_PROXY` must not be set on these pods (and has no effect if it is).
- **Which addresses Providers may have is the deployment's decision.** The code only refuses loopback, link-local (including cloud metadata), unspecified and multicast addresses ([03 Operator, "Egress"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)); on-prem and private-network Providers work. The `providerconnections` chart ships an optional NetworkPolicy for the Connectors, and `platform-mesh-operator`'s chart one for the operator, whose egress ranges for Providers are values; both are off by default.
- **Residual**: an Owner chooses the Base URL, so an Owner can make `platform-mesh-operator` and the Connectors send unauthenticated WebSocket upgrade `GET`s to anything those pods can reach. No credentials or secrets are sent (Registration's travel inside the Inner TLS, and Connectors send nothing on the upgrade), and no response body reaches the Owner. Deployments that care restrict it at the network layer, with the NetworkPolicies above or an egress gateway. NetworkPolicy works per pod, so it can't tell a Provider dial from a kcp dial; that's why the fixed refusals stay in code.
- **No new inbound paths** into PM.

## Dependencies on kcp-operator

Checked against kcp-operator `8994ed7`: the Connectors' identity comes from a `Kubeconfig` (`operator.kcp.io/v1alpha1`), with a username, groups, a validity and a target of the root shard, a shard or the front-proxy. Nothing else from kcp-operator is needed.

## Rollout order

One upgrade of `platform-mesh-operator` ([03](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)), which then does the rest itself:
1. it applies the kcp bootstrap (schemas, exports, RBAC) from its own manifests;
2. it renders the `providerconnections` HelmRelease: the Connector CA, and Connectors with their `Kubeconfig`s on every shard;
3. it processes `ProviderOnboardingRequest`s only once both are in place (the bootstrap applied, the HelmRelease Ready). Until then, requests stay `Pending` with `RemoteProvidersNotReady`.

Upgrades follow the same path. Connectors and `platform-mesh-operator` tolerate each other's absence: without `platform-mesh-operator`, no `ProviderEndpoint`s appear, and until the `ConnectorCASet` exists Connectors switch to a renewed certificate only when the current one has less than an hour left; without Connectors, `Connected` stays false.

## Local development

A kind setup with a multi-shard kcp, cert-manager, `platform-mesh-operator`, Connectors, and the quickstart Provider with `provider-gateway`, installed from its chart, in a separate network ([10 Testing, "E2E (kind, multi-shard)"](./xxx-notes-011-v5-impl-10-testing.md)). The same setup serves developers and CI.

## Decided

- **Per-shard Connector resources** come from a new `providerconnections` chart, deployed as a profile service, from an explicit `kcpTopology` in its values that duplicates `infra`'s shards, shaped like the deployer's per-instance context. The two must match; `platform-mesh-operator`'s profile validation will enforce it as a follow-up, and the shard runbook covers it until then. No watcher of `Shard` resources and no runtime drift detection.
- **Connector CA**: a cert-manager `Certificate` rendered by the chart, renewed and rolled over automatically; only the optional `issuerRef` comes from values; `distrust` is in the `ConnectorCASet`'s spec ([04 Connector CA](./xxx-notes-011-v5-impl-04-connector-ca.md)).
- **Egress**: direct, with the reachable ranges up to the deployment; optional NetworkPolicies, off by default.
- **Rollout**: driven by `platform-mesh-operator`, which gates Onboarding on the bootstrap and the `providerconnections` HelmRelease.
