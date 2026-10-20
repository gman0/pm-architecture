# v5 implementation plan: Provider quickstart

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). `pm-provider-quickstart` is the reference Provider: it shows Provider authors how to run the [SDK](./xxx-notes-011-v5-impl-07-provider-sdk.md) (`provider-gateway`) next to their operator, and it's the Provider in PM's [e2e tests](./xxx-notes-011-v5-impl-10-testing.md). Today it exposes the Wild West `Cowboys` API through an APIExport, with `ProviderMetadata` and `ContentConfiguration`, and assumes a kubeconfig for PM's kcp.

## What changes

| Today | With v5 |
|---|---|
| One kubeconfig, one PM Instance | Still the operator's input: in addition to the single kubeconfig (see "Two modes"), any number of kubeconfigs, one per Connection, which `provider-gateway` writes as `Secret`s |
| Operator watches one endpoint slice | Operator runs kcp-multicluster-provider's APIExport provider per kubeconfig, cluster names prefixed by `connectionID` |
| No Connection API server | `provider-gateway`, installed from PM's chart, serving the Connection API under the Base URL behind an ingress with WebSocket support |
| No portal flow for Onboarding | Its own small portal: sign in, get the Onboarding command and manifests, list and delete Connections |

## Principles

- **Show the intended way, nothing more.** The quickstart installs `provider-gateway` with its chart's defaults, uses the SDK's resources through the Kubernetes API, and keeps its operator on standard client-go, multicluster-runtime and kcp-multicluster-provider; anything Provider-specific is clearly marked as such.

## Pieces

### Connection API server

- `provider-gateway`, installed from PM's chart into the quickstart's namespace, with the Base URL path the token names (e.g. `https://wildwest.example.com/pm`) routed to its Service. The kind setup includes cert-manager, the chart's default for the serving certificate; the README also shows the value for running without it.
- Behind an ingress that allows WebSocket upgrades with suitable idle timeouts and maximum connection age. The manifests ship a tested **Gateway API `HTTPRoute` on traefik**, the setup PM itself uses and the e2e runs, plus a **generic checklist** for any other ingress, cloud load balancer, CDN or WAF:
  - **WebSocket upgrades allowed** on the Base URL's `/v1/connections/register/*`, `/v1/connections/*/session` and `/v1/connections/*/control` paths;
  - **no bot protection on those paths**: the Connectors and `platform-mesh-operator` aren't browsers, so JavaScript challenges and CAPTCHAs (e.g. a CDN's bot mode) block them. Exempt all three;
  - **binary WebSocket messages up to the protocol's maximum (32 KB) passed through untouched**: WAFs (web application firewalls, usually a feature of the CDN, load balancer or ingress) typically inspect only the upgrade request, which is fine; WAFs that inspect or block WebSocket payloads need a rule exempting the three paths. The payload is TLS ciphertext, so there's nothing to inspect;
  - **paths passed through unchanged**: the Edge must not rewrite one Connection's path into another's. `platform-mesh-operator` reports it as `EdgePathRewritten` if it does;
  - **idle timeout above the outer ping interval** ([01 Protocol, "WebSocket framing"](./xxx-notes-011-v5-impl-01-protocol.md); some load balancers default to 60 seconds);
  - **a maximum connection age long enough** for Connectors to replace Sessions gracefully (hours, not minutes);
  - **no request-body buffering or size limits** that break long-lived Streams;
  - **no client-IP affinity or sticky sessions** on those paths, so a Connector that reaches a standby replica can reach the leader on its next attempt;
  - **enough concurrent connections**, also per source IP: every replica of PM's Connector for a shard keeps its own Session, so expect up to R Sessions per Connection and shard (R typically 2–3), all from a PM Instance's few egress addresses. Streams, and so watches and requests, aren't multiplied ([01 Protocol, "Sessions and Streams"](./xxx-notes-011-v5-impl-01-protocol.md));
  - **a plain load balancer or ingress, not a message-based WebSocket API gateway**: gateways that route individual messages to backends, cap message sizes and limit connection lifetimes (such as cloud "WebSocket API" products) don't work;
  - **a server certificate from a public CA**, or, with a private CA, Owners set it as their `Provider`'s `caBundle`, which may hold several CAs. Before switching to a new private CA, or to a new hostname whose certificate comes from one, tell Owners, so they add it next to the old one; otherwise their new Sessions fail with `EdgeTLS`.

  This checklist is the one list of Edge requirements; the SDK's requirements and the v5 notes refer to it. The quickstart's README walks through checking each point on the deployed Base URL; `platform-mesh-operator` reports violations it sees on the `Provider` with a labelled message ([03 Operator, "Edge diagnosis"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
- Replicas: `provider-gateway` with **leader election**, one active replica; standbys answer upgrades with `503` and `Retry-After: 5` ([07 SDK, "Replicas"](./xxx-notes-011-v5-impl-07-provider-sdk.md)). The Gateway API example routes the `…/register/…`, `…/session` and `…/control` paths without affinity. The operator runs and scales separately.
- State: the SDK's resources (`Connection`, `OnboardingToken`, `ServerKey`), and the kubeconfig `Secret`s, in the quickstart's namespace, which the operator reads through a Role the quickstart's manifests add.
- Metrics: `provider-gateway`'s collectors, scraped by the `ServiceMonitor` in its chart.

### Bootstrap per Connection

Bootstrap of the Provider's own APIs stays the Provider's job. When a Connection appears, the quickstart's bootstrap (today `pkg/bootstrap`) creates its APIExport, schemas, `ProviderMetadata`, `ContentConfiguration` and endpoint slice in that Connection's Provider Workspace, through the Connection's kubeconfig. Its Service endpoint slices stay in that workspace, since Connectors open Sessions on a shard only if such a slice lists it ([05 Connector, "Which Sessions"](./xxx-notes-011-v5-impl-05-connector.md)). Consumers get `bind` on the export through the ClusterRole `providers.platform-mesh.io:apiexport-bind`, the only role a Provider may bind to subjects outside its workspace ([02 APIs, "Policies in Provider Workspaces"](./xxx-notes-011-v5-impl-02-apis.md)). Idempotent; reruns on reconnect. It only creates or adopts: it never deletes and recreates the APIExport or its identity `Secret`, so a Connection that comes back (onboarded again, imported after a disaster) finds consumers' bindings intact ([07 SDK, "Requirements on Providers"](./xxx-notes-011-v5-impl-07-provider-sdk.md)).

### Operator

The Wild West operator (`operator/wild-west`) reads its clusters from the mode abstraction (see "Two modes"); in Connection mode, that's the kubeconfig `Secret`s `provider-gateway` writes ([07 SDK, "Kubeconfig per Connection"](./xxx-notes-011-v5-impl-07-provider-sdk.md)):
- watch the `Secret`s by their `provider-sdk.platform-mesh.io/connection` label, and load each with `clientcmd`, unchanged;
- on each, start kcp-multicluster-provider's APIExport provider for its `Cowboys` export's slice, exactly as against kcp directly;
- compose all providers with `connectionID`-prefixed cluster names, the `connectionID` taken from the label;
- show the shared transport per Connection as an optional, clearly marked optimisation for many consumer clusters ([07 SDK, "Requirements on Providers"](./xxx-notes-011-v5-impl-07-provider-sdk.md));
- keep all state keyed by Connection and cluster;
- show the rules for consumer data ([07 SDK, "Requirements on Providers"](./xxx-notes-011-v5-impl-07-provider-sdk.md)), with a stand-in external resource per `Cowboy` (e.g. an object in the Provider's own cluster) if the example has none: it's found again from the `Cowboy` (an ID in its status) before anything new is created, and deleted only when the `Cowboy` is, through its finalizer, never when a Connection goes away.

### Portal

The quickstart's own small portal, served by its binary on its own path behind the same ingress (e.g. `https://wildwest.example.com/portal`). It's the Provider's website for Owners, outside PM; the Angular microfrontend in `portal/` is something else, the consumer UI that runs inside PM's portal, and stays as it is.
- **Placeholder login**: a name, no password, clearly marked as such. The name is the Owner account tokens are bound to.
- **Onboard**: creates an `OnboardingToken` with the Go module's `NewOnboardingToken` and shows, once, the `kubectl create secret …` command carrying the token (never in a manifest or a URL), next to the `Provider` (with `spec.remote`) and `ProviderOnboardingRequest` manifests to apply, the latter with the token's `tokenID` filled in. The token is the Owner's credential until it's spent, so the page that shows it is served over HTTPS and never cached.
- **Connections**: the Owner's `Connection`s, listed through the Kubernetes API with a field selector on `spec.registration.owner` (ID, `clusterName`, `Ready`, `lastSeen`), each with a Delete button that deletes the object after a confirmation.

Notifications to Owners aren't part of it: the quickstart watches `Connection` Events and logs them, which is where a Provider would notify its customers. Iterating on the portal is a [follow-up](./xxx-notes-011-v5-impl-todo.md).

## Documentation for Provider authors

The quickstart README becomes the Provider-facing guide. For the rules a Provider follows (consumer data, bootstrap, the Edge, replicas, keeping the SDK's resources durable) it points to [07 SDK, "Requirements on Providers"](./xxx-notes-011-v5-impl-07-provider-sdk.md), with the quickstart's code as the example. On top, it covers:
- how to adapt an existing single-kubeconfig operator to Connections: install `provider-gateway`, then load one kubeconfig per Connection instead of one in total;
- what the Provider can and can't do in its Provider Workspace: admin, minus the Connector's Denylist; role bindings only to ServiceAccounts in the workspace, kcp's `apis.kcp.io:binding:` subjects, or `providers.platform-mesh.io:apiexport-bind`; no SA-token `Secret`s; `ProviderEndpoint`s read-only; only the workspace and the APIExport virtual workspace, by logical cluster name;
- that its own admission policies on its exported resources also apply in consumers' workspaces;
- the trust model in one paragraph: no kcp credentials, Sessions end-to-end encrypted to the Provider's runtime, Registration protected by the Server Key named in the token, and the portal trusted to hand the token only to its Owner.

## Work breakdown

1. Install `provider-gateway` from PM's chart; ingress manifests.
2. Bootstrap per Connection.
3. The mode abstraction; the operator on it, with the kubeconfig `Secret`s in Connection mode and the existing single kubeconfig in kubeconfig mode.
4. The portal: placeholder login, *Onboard* and *Connections*.
5. README and examples.
6. Deployment variant used by the kind e2e, behind the network boundary ([10 Testing, "E2E (kind, multi-shard)"](./xxx-notes-011-v5-impl-10-testing.md)).

## Two modes

The quickstart keeps **both modes in one codebase**, selected by configuration:
- **Kubeconfig mode**, as today, for Local Providers deployed as ManagedProviders (RFC 006): one kubeconfig, one PM Instance.
- **Connection mode**, with `provider-gateway`, for Remote Providers.

Both feed the same operator code through one abstraction: a source of (cluster name prefix, kubeconfig) pairs. In kubeconfig mode there's exactly one, from the configured kubeconfig; in Connection mode, one per Connection, from the `Secret`s `provider-gateway` writes. Bootstrap and reconciliation don't know which mode they run in; both are plain kubeconfigs.

## Decided

- **Both modes kept**, behind one abstraction (above): kubeconfig mode for the ManagedProvider flow PM keeps supporting, Connection mode for Remote Providers.
- **The portal is the quickstart's own**, using the SDK's resources through the Kubernetes API; Owner notifications are log-only for now.
- **Ingress documentation**: a tested Gateway API example on traefik, plus the generic checklist; no per-vendor guides.
