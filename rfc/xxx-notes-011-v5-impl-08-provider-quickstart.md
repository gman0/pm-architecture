# v5 implementation plan: Provider quickstart

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). `pm-provider-quickstart` is the reference Provider: it shows Provider authors how to use the [SDK](./xxx-notes-011-v5-impl-07-provider-sdk.md), and it's the Provider in PM's [e2e tests](./xxx-notes-011-v5-impl-10-testing.md). Today it exposes the Wild West `Cowboys` API through an APIExport, with `ProviderMetadata` and `ContentConfiguration`, and assumes a kubeconfig for PM's kcp.

## What changes

| Today | With v5 |
|---|---|
| One kubeconfig, one PM Instance | Still supported (see "Two modes"); in addition, any number of Connections, one per PM Instance (or per Provider Workspace), from the SDK |
| Operator watches one endpoint slice | Operator runs one multicluster-runtime provider per Connection and endpoint slice, cluster names prefixed by `connectionID` |
| No Connection API server | The SDK's Connection API server, mounted under the Base URL behind an ingress with WebSocket support |
| No portal flow for Onboarding | Its own small portal: sign in, get the Onboarding command and manifests, list and delete Connections |

## Principles

- **Show the intended way, nothing more.** The quickstart uses SDK defaults, the SDK's resources through the Kubernetes API, and the multicluster-runtime helper; anything Provider-specific is clearly marked as such.

## Pieces

### Connection API server

- Mounted under the Base URL path the token names (e.g. `https://wildwest.example.com/pm`).
- Behind an ingress that allows WebSocket upgrades with suitable idle timeouts and maximum connection age. The manifests ship a tested **Gateway API `HTTPRoute` on traefik**, the setup PM itself uses and the e2e runs, plus a **generic checklist** for any other ingress, cloud load balancer, CDN or WAF:
  - **WebSocket upgrades allowed** on the Base URL's `/v1/connections/*/session` and `/v1/connections/*/control` paths;
  - **no bot protection on those paths**: the Connectors and `platform-mesh-operator` aren't browsers, so JavaScript challenges and CAPTCHAs (e.g. a CDN's bot mode) block them. Exempt both paths;
  - **binary WebSocket messages up to the protocol's maximum (32 KB) passed through untouched**: WAFs (web application firewalls, usually a feature of the CDN, load balancer or ingress) typically inspect only the upgrade request, which is fine; WAFs that inspect or block WebSocket payloads need a rule exempting the two paths. The payload is TLS ciphertext, so there's nothing to inspect;
  - **idle timeout above the outer ping interval** ([01 Protocol, "WebSocket framing"](./xxx-notes-011-v5-impl-01-protocol.md); some load balancers default to 60 seconds);
  - **a maximum connection age long enough** for Connectors to replace Sessions gracefully (hours, not minutes);
  - **no request-body buffering or size limits** that break long-lived Streams;
  - **no client-IP affinity or sticky sessions** on those paths, so a Connector that reaches a standby replica can reach the leader on its next attempt;
  - **enough concurrent connections**, also per source IP: every replica of PM's Connector for a shard keeps its own Session, so expect up to R Sessions per Connection and shard (R typically 2–3), all from a PM Instance's few egress addresses. Streams, and so watches and requests, aren't multiplied ([01 Protocol, "Sessions and Streams"](./xxx-notes-011-v5-impl-01-protocol.md));
  - **a plain load balancer or ingress, not a message-based WebSocket API gateway**: gateways that route individual messages to backends, cap message sizes and limit connection lifetimes (such as cloud "WebSocket API" products) don't work;
  - **a server certificate from a public CA**, or, with a private CA, Owners set it as their `Provider`'s `caBundle`. Before switching to a new private CA, tell Owners, so they add it next to the old one; otherwise their new Sessions fail with `EdgeTLS`.

  This checklist is the one list of Edge requirements; the SDK's requirements and the v5 notes refer to it. The quickstart's README walks through checking each point on the deployed Base URL; `platform-mesh-operator` reports violations it sees on the `Provider` with a labelled message ([03 Operator, "Edge diagnosis"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
- Replicas: **leader election**, one active replica; standbys answer upgrades with `503` and `Retry-After: 5`; controllers in the same process as the Connection API server, scaling vertically for now ([07 SDK, "Replicas"](./xxx-notes-011-v5-impl-07-provider-sdk.md)). The Gateway API example routes the `…/session` and `…/control` paths without affinity.
- State: the SDK's resources (`Connection`, `OnboardingToken`, `ServerKey`), in the quickstart's namespace.
- Metrics: the SDK's collectors on the operator's metrics endpoint, scraped by a `ServiceMonitor` in the manifests.

### Bootstrap per Connection

Bootstrap of the Provider's own APIs stays the Provider's job. When a Connection appears, the quickstart's bootstrap (today `pkg/bootstrap`) creates its APIExport, schemas, `ProviderMetadata`, `ContentConfiguration` and endpoint slice in that Connection's Provider Workspace, through the Connection's client config. Its Service endpoint slices stay in that workspace, since Connectors open Sessions on a shard only if such a slice lists it ([05 Connector, "Which Sessions"](./xxx-notes-011-v5-impl-05-connector.md)). Idempotent; reruns on reconnect. It only creates or adopts: it never deletes and recreates the APIExport or its identity `Secret`, so a Connection that comes back (onboarded again, imported after a disaster) finds consumers' bindings intact ([07 SDK, "Requirements on Providers"](./xxx-notes-011-v5-impl-07-provider-sdk.md)).

### Operator

The Wild West operator (`operator/wild-west`) reads its clusters from the mode abstraction (see "Two modes"); in Connection mode, that's the SDK's Connection registry:
- on each Connection, start an endpoint-slice provider for its `Cowboys` export's slice;
- compose all providers with `connectionID`-prefixed cluster names;
- keep all state keyed by Connection and cluster ([07 SDK, "Connections for multicluster-runtime"](./xxx-notes-011-v5-impl-07-provider-sdk.md));
- show the rules for consumer data ([07 SDK, "Requirements on Providers"](./xxx-notes-011-v5-impl-07-provider-sdk.md)), with a stand-in external resource per `Cowboy` (e.g. an object in the Provider's own cluster) if the example has none: it's found again from the `Cowboy` (an ID in its status) before anything new is created, and deleted only when the `Cowboy` is, through its finalizer, never when a Connection goes away.

### Portal

The quickstart's own small portal, served by its binary on its own path behind the same ingress (e.g. `https://wildwest.example.com/portal`). It's the Provider's website for Owners, outside PM; the Angular microfrontend in `portal/` is something else, the consumer UI that runs inside PM's portal, and stays as it is.
- **Placeholder login**: a name, no password, clearly marked as such. The name is the Owner account tokens are bound to.
- **Onboard**: creates an `OnboardingToken` with `NewOnboardingToken` and shows, once, the `kubectl create secret …` command carrying the token (never in a manifest or a URL), next to the `Provider` (with `spec.remote`) and `ProviderOnboardingRequest` manifests to apply.
- **Connections**: the Owner's `Connection`s, listed through the Kubernetes API with a field selector on `spec.registration.owner` (ID, `clusterName`, `Ready`, `lastSeen`), each with a Delete button that deletes the object after a confirmation.

Notifications to Owners aren't part of it: the quickstart watches `Connection` Events and logs them, which is where a Provider would notify its customers. Iterating on the portal is a [follow-up](./xxx-notes-011-v5-impl-todo.md).

## Documentation for Provider authors

The quickstart README becomes the Provider-facing guide. For the rules a Provider follows (consumer data, bootstrap, the Edge, replicas, keeping the SDK's resources durable) it points to [07 SDK, "Requirements on Providers"](./xxx-notes-011-v5-impl-07-provider-sdk.md), with the quickstart's code as the example. On top, it covers:
- how to adapt an existing single-kubeconfig operator to Connections;
- what the Provider can and can't do in its Provider Workspace (admin, minus the Connector's Denylist, and `ProviderEndpoint`s are read-only);
- the trust model in one paragraph: no kcp credentials, Sessions end-to-end encrypted to the Provider's runtime.

## Work breakdown

1. Mount the SDK's Connection API server; ingress manifests.
2. Bootstrap per Connection.
3. The mode abstraction; the operator on it, with the multicluster-runtime helper in Connection mode and the existing single kubeconfig in kubeconfig mode.
4. The portal: placeholder login, *Onboard* and *Connections*.
5. README and examples.
6. Deployment variant used by the kind e2e, behind the network boundary ([10 Testing, "E2E (kind, multi-shard)"](./xxx-notes-011-v5-impl-10-testing.md)).

## Two modes

The quickstart keeps **both modes in one codebase**, selected by configuration:
- **Kubeconfig mode**, as today, for Local Providers deployed as ManagedProviders (RFC 006): one kubeconfig, one PM Instance.
- **Connection mode**, with the SDK, for Remote Providers.

Both feed the same operator code through one abstraction: a source of (cluster name prefix, client config) pairs. In kubeconfig mode there's exactly one, from the kubeconfig; in Connection mode, one per Connection from the SDK's registry. Bootstrap and reconciliation don't know which mode they run in.

## Decided

- **Both modes kept**, behind one abstraction (above): kubeconfig mode for the ManagedProvider flow PM keeps supporting, Connection mode for Remote Providers.
- **The portal is the quickstart's own**, using the SDK's resources through the Kubernetes API; Owner notifications are log-only for now.
- **Ingress documentation**: a tested Gateway API example on traefik, plus the generic checklist; no per-vendor guides.
