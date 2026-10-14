# v5 implementation plan: Provider quickstart

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). `pm-provider-quickstart` is the reference Provider: it shows Provider authors how to use the [SDK](./xxx-notes-011-v5-impl-07-provider-sdk.md), and it's the Provider in PM's [e2e tests](./xxx-notes-011-v5-impl-10-testing.md). Today it exposes the Wild West `Cowboys` API through an APIExport, with `ProviderMetadata` and `ContentConfiguration`, and assumes a kubeconfig for PM's kcp.

## What changes

| Today | With v5 |
|---|---|
| One kubeconfig, one PM instance | Still supported (see "Two modes"); in addition, any number of connections, one per PM instance (or per provider workspace), from the SDK |
| Operator watches one endpoint slice | Operator runs one multicluster-runtime provider per connection and endpoint slice, cluster names prefixed by `connectionID` |
| No connection server | The SDK's connection server, mounted under the base URL behind an ingress with WebSocket support |
| No portal flow for onboarding | A minimal portal flow: sign in, issue a token, list, freeze and delete connections |

## Principles

- **Show the intended way, nothing more.** The quickstart uses SDK defaults, the Kubernetes reference store, and the multicluster-runtime helper; anything Provider-specific is clearly marked as such.
- **Bootstrap of the Provider's own APIs stays the Provider's job.** The quickstart keeps creating its APIExport, schemas and endpoint slices in its provider workspace, now through a connection instead of a kubeconfig, and keeps its Service endpoint slices in that workspace (the connector's session rule depends on it, [05 Connector, "Which sessions"](./xxx-notes-011-v5-impl-05-connector.md)).

## Pieces

### Connection server

- Mounted under the base URL path the token names (e.g. `https://wildwest.example.com/pm`).
- Behind an ingress that allows WebSocket upgrades with suitable idle timeouts and maximum connection age. The manifests ship a tested **Gateway API `HTTPRoute` on traefik**, the setup PM itself uses and the e2e runs, plus a **generic checklist** for any other ingress, cloud load balancer, CDN or WAF:
  - **WebSocket upgrades allowed** on the base URL's `/v1/connections/*/session` and `/v1/connections/*/control` paths;
  - **no bot protection on those paths**: the connectors and platform-mesh-operator aren't browsers, so JavaScript challenges and CAPTCHAs (e.g. a CDN's bot mode) block them. Exempt both paths;
  - **binary WebSocket messages up to 32 KB passed through untouched**: WAFs (web application firewalls, usually a feature of the CDN, load balancer or ingress) typically inspect only the upgrade request, which is fine; WAFs that inspect or block WebSocket payloads need a rule exempting the two paths. The payload is TLS ciphertext, so there's nothing to inspect;
  - **idle timeout above the session ping interval** (some load balancers default to 60 seconds);
  - **a maximum connection age long enough** for connectors to replace sessions gracefully (hours, not minutes);
  - **no request-body buffering or size limits** that break long-lived streams;
  - **no client-IP affinity or sticky sessions** on those paths, so a connector that reaches a standby replica can reach the leader on its next attempt;
  - **a plain load balancer or ingress, not a message-based WebSocket API gateway**: gateways that route individual messages to backends, cap message sizes and limit connection lifetimes (such as cloud "WebSocket API" products) don't work.

  The quickstart's README walks through running the SDK's **edge self-test** against the deployed base URL, which checks each point ([07 SDK, "Edge self-test"](./xxx-notes-011-v5-impl-07-provider-sdk.md)).
- Storage: the SDK's Kubernetes reference store, in the quickstart's namespace.

### Replicas

The quickstart runs the connection server and its controllers **with leader election**, as one active replica; standbys answer upgrades with `503` and `Retry-After = max(5s, ceil(LeaseDuration / 3))` ([07 SDK, "Replicas"](./xxx-notes-011-v5-impl-07-provider-sdk.md)). Its README states the constraints for vendors: controllers live in the same process as the connection server and scale vertically until the owner-replica mode is available, and the edge must not pin the session and control paths to one backend (no client-IP affinity or sticky sessions). The Gateway API example routes those paths without affinity.

### Bootstrap per connection

When a connection appears, the quickstart's bootstrap (today `pkg/bootstrap`) creates its APIExport, schemas, `ProviderMetadata`, `ContentConfiguration` and endpoint slice in that connection's provider workspace, through the connection's client config. Idempotent; reruns on reconnect. It only creates or adopts: it never deletes and recreates the APIExport or its identity Secret, so a connection that comes back (onboarded again, imported after a disaster) finds consumers' bindings intact ([07 SDK, "Requirements on Providers"](./xxx-notes-011-v5-impl-07-provider-sdk.md)).

### Operator

The Wild West operator (`operator/wild-west`) reads its clusters from the mode abstraction (see "Two modes"); in connection mode, that's the SDK's connection registry:
- on each connection, start an endpoint-slice provider for its `Cowboys` export's slice;
- compose all providers with `connectionID`-prefixed cluster names;
- keep all state keyed by connection and cluster ([07 SDK, "Connections for multicluster-runtime"](./xxx-notes-011-v5-impl-07-provider-sdk.md));
- show the rules for consumer data ([07 SDK, "Requirements on Providers"](./xxx-notes-011-v5-impl-07-provider-sdk.md)), with a stand-in external resource per `Cowboy` (e.g. an object in the Provider's own cluster) if the example has none: it's found again from the `Cowboy` (an ID in its status) before anything new is created, and deleted only when the `Cowboy` is, through its finalizer, never when a connection goes away.

### Portal

The existing Angular portal gains:
- **Issue token**: shows the token once, with a `kubectl create secret` command, never a manifest, and never puts the token in a URL;
- **Connections**: list with status (from the SDK store), freeze, unfreeze, delete;
- **Notifications**: a simple list of events from the SDK's hooks.

Authentication of portal users is out of scope; the quickstart uses a placeholder login and says so.

## Documentation for Provider authors

The quickstart README becomes the Provider-facing guide:
- what a Provider must run (connection server, ingress settings, storage);
- how to adapt an existing single-kubeconfig operator to connections;
- what the Provider can and can't do in its provider workspace (admin, minus the connector's denylist, and records are read-only);
- the rules for consumer data and bootstrap, and why: keyed with the `connectionID`, adopt before creating, nothing deleted when a connection goes away, no recreated APIExport identity; and keeping the SDK store as durable as consumer data;
- the trust model in one paragraph: no kcp credentials, sessions end-to-end encrypted to the Provider's runtime.

## Work breakdown

1. Mount the SDK's connection server; ingress manifests.
2. Bootstrap per connection.
3. The mode abstraction; the operator on it, with the multicluster-runtime helper in connection mode and the existing single kubeconfig in kubeconfig mode.
4. Portal pages.
5. README and examples.
6. Deployment variant used by the kind e2e, behind the network boundary ([10 Testing, "E2E (kind, multi-shard)"](./xxx-notes-011-v5-impl-10-testing.md)).

## Two modes

The quickstart keeps **both modes in one codebase**, selected by configuration:
- **Kubeconfig mode**, as today, for local Providers deployed as ManagedProviders (RFC 006): one kubeconfig, one PM instance.
- **Connection mode**, with the SDK, for remote Providers.

Both feed the same operator code through one abstraction: a source of (cluster name prefix, client config) pairs. In kubeconfig mode there's exactly one, from the kubeconfig; in connection mode, one per connection from the SDK's registry. Bootstrap and reconciliation don't know which mode they run in.

## Decided

- **Both modes kept**, behind one abstraction (above).
- **Ingress documentation**: a tested Gateway API example on traefik, plus the generic checklist; no per-vendor guides.
