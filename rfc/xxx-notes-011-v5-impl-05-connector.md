# v5 implementation plan: Connector

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). The Connector: a new service at `platform-mesh/services/providerconnections-connector`, a standalone proxy server modelled on kcp's front-proxy, one per kcp shard, deployed as a plain Deployment ([09 Deployment, "Connector"](./xxx-notes-011-v5-impl-09-deployment.md)). It opens Sessions to Providers and proxies their requests to kcp with each Connection's SA token.

## Responsibilities

**Owns:**
- deciding which Sessions its shard needs, and opening, keeping and replacing them;
- the PM end of the Session protocol ([01 Protocol, "Sessions"](./xxx-notes-011-v5-impl-01-protocol.md));
- authorizing, logging and proxying every request that arrives over a Session;
- minting and caching SA tokens.

**Doesn't own:**
- `ProviderEndpoint`s or anything else in kcp (it writes only TokenRequests);
- Connection Secrets or Control Calls ([03 Operator](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md));
- issuing its certificate ([04 Connector CA](./xxx-notes-011-v5-impl-04-connector-ca.md)).

## Principles

- **Stateless** (overview, principle 5): everything comes from its deployment (shard name, its virtual-workspace host, the front-proxy host, its Connector Certificate) and the Connection Export.
- **The Provider's only path into kcp**: whatever it allows, the Provider can do; the policy is the security guarantee and is tested as such ([10 Testing](./xxx-notes-011-v5-impl-10-testing.md)).
- **A standalone proxy, like kcp's front-proxy** (`kcp/pkg/proxy`): its own `http.Server`, a few filters taken from upstream `k8s.io/apiserver` (`WithPanicRecovery`, `WithHTTPLogging`), its own path parsing (see "Path parsing"), and `httputil.ReverseProxy`. No generic apiserver.
- **kcp owns request timeouts.** The Connector applies none of its own to proxied requests and has no shared in-flight cap; its limits are per Connection (see "Limits").
- **Forward bytes, don't decode objects.** The Connector never parses response bodies.

## Structure

```
 connector @ shard (Deployment, one process per replica, stateless)
 ┌──────────────────────────────────────────────────────────────────────────────────┐
 │ Session manager (controller)                                                     │
 │   watches the connection export: ProviderEndpoints + claimed endpoint slices     │
 │   session rule → which connections need a session from this shard                │
 │   dials WebSocket → inner mutual TLS → hello → yamux (Provider opens streams)    │
 │   replaces sessions, honours go-away, backoff, certificate switch rule           │
 │                                     │ yamux streams                              │
 │                                     ▼                                            │
 │ Stream server: one http.Server, a listener per session yielding its streams      │
 │   ConnContext: connectionID, clusterName; header-read timeout, max header size   │
 │                                     ▼                                            │
 │ Request pipeline                                                                 │
 │   recover → limits → Host check → parse path → authorize → strip headers         │
 │   → SA token → log + metrics → reverse proxy                                     │
 │                                     ▼                                            │
 │                  this shard's virtual workspace / front-proxy                    │
 │                                                                                  │
 │ Ops listener (cluster-internal): /metrics, /healthz, /readyz; nothing else       │
 └──────────────────────────────────────────────────────────────────────────────────┘
```

Nothing reaches the Connector from the network except the ops listener: it dials out to Providers and serves Streams on in-process listeners.

## Discovery

- **Consumes the Connection Export's endpoint slice** through the front-proxy, and list/watches through every shard's virtual-workspace URL in it, with read-only `apiexports/content`, using multicluster-runtime's endpoint-slice provider.
- **Sees two things**:
  - `ProviderEndpoint`s, each arriving with its Provider Workspace's logical cluster; the endpoint that delivered it tells which shard hosts the workspace;
  - the Provider Workspaces' Service `apiexportendpointslices`, through the export's permission claim.
- **An outage of one shard's virtual workspace** freezes the `ProviderEndpoint`s and slices from that shard; informers keep the last state.

## Which Sessions

A shard's Connector keeps a Session for a Connection when either:
- **the Connection's `ProviderEndpoint` came from its own shard's endpoint**: the Provider Workspace is hosted here, so this Connector carries the Front-Proxy Session, from the start; or
- **one of the Connection's Service endpoint slices lists its own shard's virtual-workspace URL**, counting only slices in the Provider Workspace that point at an export in that same workspace.

The rule follows kcp's own: a shard is listed in a slice exactly when it has a consumer binding.

## Replicas

Every replica of a shard's Connector applies the Session rule on its own and keeps a Session for every Connection the shard needs. Replicas don't know about each other.
- **Why it works**: Sessions aren't exclusive. A Connection has one Session per replica per shard, and the Provider-side spreads new Streams across all Sessions serving a host ([01 Protocol, "Sessions and Streams"](./xxx-notes-011-v5-impl-01-protocol.md)).
- **What replicas are for**: availability, and spreading traffic. A crashed replica cuts only the Streams on its own Sessions; client-go reopens them on the other replicas' Sessions at once. Rollouts go one replica at a time without a gap: a leaving replica sends go-away and drains. Requests and watches spread over the replicas, so each carries about an R-th of the shard's traffic.
- **What it costs**: every replica holds a Session for every Connection, so a Provider sees up to R Sessions per (Connection, shard), where R is the replica count (typically 2–3); the quickstart's Edge checklist says so ([08 Quickstart, "Connection API server"](./xxx-notes-011-v5-impl-08-provider-quickstart.md)). Each replica also mints its own SA tokens.
- **Scale-up** opens the new replica's Sessions spread with jitter, so Providers aren't hit at once.
- **If Session counts at Providers ever matter more** than spreading traffic, replicas can divide Connections by rendezvous hashing of `connectionID` over the live replicas instead. That's a contained change inside the Connector: no protocol, SDK or API change, and a rollout between the two is harmless because Sessions aren't exclusive.

## Opening a Session

1. **Dial the Base URL** from the `ProviderEndpoint`, under the egress rules (see "Egress to Providers"), verify the outer TLS with the `ProviderEndpoint`'s `caBundle` when set (web PKI otherwise), and upgrade to a WebSocket. The upgrade carries no credentials.
2. **Run the Inner TLS client handshake** with the Connector Certificate as client certificate, accepting only Server Keys from the `ProviderEndpoint`. That handshake is the Session's authentication ([01 Protocol, "Inner layer"](./xxx-notes-011-v5-impl-01-protocol.md)); a failure ends with a TLS alert (e.g. `unknown_ca`), which the Connector reports in its metrics.
3. **Send the Hello**: `clusterName` (the logical cluster of the `ProviderEndpoint` it acts on), `hosts` (the shard's virtual-workspace host from the Connector's configuration; during a host rename, both the old and the new one) and an opaque `sub` for audit ([01 Protocol, "Hello"](./xxx-notes-011-v5-impl-01-protocol.md)).
4. **Start yamux** and accept Streams opened by the Provider-side.
5. **Replace Sessions** before the Edge's maximum connection age (configurable), gracefully; reconnect with backoff on failure. An upgrade refused with `503` and `Retry-After` is retried using the value as the lower bound for the normal reconnect backoff; a Session the Provider-side ends gracefully (go-away) is reopened at once. The Connector assumes nothing about which Provider replica a Session reaches ([01 Protocol, "Provider replicas"](./xxx-notes-011-v5-impl-01-protocol.md)). No replacement while the Connector's own certificate is expired (see below).

### Connector Certificate

cert-manager renews the Connector Certificate every 12 hours, for 24 hours each ([04 Connector CA, "Connector Certificates"](./xxx-notes-011-v5-impl-04-connector-ca.md)). The Connector keeps its current certificate and key in memory and decides when to use a renewed one:
- **Switch rule**: use the renewed certificate once its CA is in the Connector CA Set and its entry in the `ConnectorCASet`'s `status.certificates` is `converged` (public data; the Connector may read it), or when the current certificate has less than about an hour left, whichever comes first. A renewal under the same CA switches at once; one from a new CA waits until every Connection `platform-mesh-operator` reaches trusts that CA, normally minutes. Open Sessions keep their handshake either way.
- **Switching replaces no Session.** Open Sessions keep the certificate they were opened with. Sessions still on the old CA's certificate when `platform-mesh-operator` drops the old CA are closed by the Provider-side with go-away and a grace period, and the Connector reopens them at once with its current certificate ([01 Protocol, "Pinned Bundle"](./xxx-notes-011-v5-impl-01-protocol.md)).
- **A forced switch** (the current certificate about to expire, its successor's CA not converged, e.g. because `platform-mesh-operator` is down): only new Sessions use the new certificate, failing at Providers that don't pin its CA yet.
- **After a restart**, only the newest certificate is on disk, so the Connector uses it at once. If its CA hasn't converged yet, Sessions to Connections that don't pin it yet fail until `platform-mesh-operator`'s push lands, normally within minutes.
- **Reported**: how long a renewed certificate has been waiting to be used, and the current certificate's expiry (metrics, see [04 Connector CA, "Signals"](./xxx-notes-011-v5-impl-04-connector-ca.md)).

**Provider downtime doesn't affect certificate renewal.** Providers pin the Connector CA, not Connector Certificates, so a renewal needs nothing from them. While a Provider is down, the Connector retries with backoff; when it's back, the Connector dials with its current certificate. If a Rollover happened meanwhile, `platform-mesh-operator`'s first successful Control Call brings the Provider the current Connector CA Set, and the next dial succeeds. Open Sessions aren't affected when the certificate that opened them expires: the handshake was checked when the Session opened, and TLS doesn't re-check expiry on an open connection.

**cert-manager is a dependency.** Renewal normally overlaps by 12 hours: a renewed certificate is in place while the current one has half its life left. Only if renewal keeps failing past that (cert-manager down, the issuer broken, requests not approved) does a certificate expire. Then the Connector opens no new Sessions and replaces none; open Sessions keep carrying traffic until something else ends them. Readiness reports the expired state, so rollouts hold; replicas shouldn't be restarted until renewal resumes. The signals warn well before expiry ([04 Connector CA, "Signals"](./xxx-notes-011-v5-impl-04-connector-ca.md)); what an outage looks like in detail is in the runbook.

## Serving Streams

- **One `http.Server`** serves all Streams. Each Session is a listener that yields its yamux Streams; `ConnContext` puts the Session's `connectionID` and `clusterName` into every request's context. Streams carry HTTP/1.1 from their first byte; there's nothing to read before.
- **That context is the caller's identity.** Only these in-process listeners set it, so nothing arriving from the network can carry it, and there's nothing else to authenticate.
- **A header-read timeout and a maximum header size** on the server, so reading a request's headers before its `Host` is checked stays bounded and a Provider can't hold Streams by sending headers slowly.

## Request pipeline

Plain `http.Handler` wrappers, in order, the way kcp's front-proxy builds its handler:
1. **Recover** from panics (`WithPanicRecovery`).
2. **Limits** (see "Limits").
3. **Check `Host`** against the allow-list: this shard's virtual-workspace host and the front-proxy host, from the Connector's deployment configuration. It's the same list for every Connection; anything else is refused. This is the only target check: the Stream carries no separate target.
4. **Parse the path** (see "Path parsing").
5. **Authorize** (see "Authorization and policy").
6. **Strip credentials and identity headers**: `Authorization`, `Impersonate-*`, `Proxy-*`, cookies, hop-by-hop headers, and kcp-internal headers such as `X-Kcp-Virtual-Resource-Hops`.
7. **Add the SA token** for the Connection (see "Identity towards kcp").
8. **Log and count** the request (see "Observability").
9. **Reverse-proxy** with `httputil.ReverseProxy`:
   - **map the external host to an in-cluster address** for the dial, keeping `Host` and the TLS server name, so traffic doesn't hairpin through PM's load balancer ([09 Deployment, "Network"](./xxx-notes-011-v5-impl-09-deployment.md));
   - **forward over TLS** verified with PM's own trust for kcp; resolve once, dial the checked address;
   - **stream the response** with immediate flushing, so watches and LISTs pass through without buffering;
   - **buffers**: a small copy buffer per Stream, and a cap on the upstream HTTP/2 receive buffer per stream, so a slow Provider's backlog stays upstream.

## Path parsing

Two kcp path shapes, parsed explicitly:
- **`/clusters/<cluster>/…`**, the Provider Workspace through the front-proxy;
- **`/services/apiexport/<cluster>/<export>/clusters/<cluster or *>/…`**, the APIExport virtual workspace.

The `/clusters/<cluster>` part is stripped by a copy of kcp's `ClusterPathFromAndStrip` (`kcp/pkg/server/filters/filters.go`, about 20 lines, with a comment naming its origin), and cluster names are checked with `kcp-dev/logicalcluster`. The rest of the path goes to upstream `k8s.io/apiserver`'s `RequestInfoFactory` (`pkg/endpoints/request`, imported, not kcp's fork), which gives verb (`get`, `list`, `watch`, `deletecollection`, …), group, resource, subresource, namespace and name. Anything that doesn't parse is refused.

## Authorization and policy

For the request's Connection:
- **Scope**: only its own Connection's paths, i.e. its Provider Workspace through the front-proxy (`/clusters/<its clusterName>/…`), and the APIExport virtual-workspace endpoints of exports in that workspace.
- **Denylist** (v5, "Connector policy"):
  - `update`, `patch`, `delete` on the Connection Export's APIBinding;
  - writes to `apiexportendpointslices/status`;
  - `create` on `serviceaccounts/token`;
  - `create` on `certificatesigningrequests` and their approval;
  - upgrade requests (`exec`, `attach`, `port-forward`);
  - **`Secret`s of type `kubernetes.io/service-account-token`**: kcp issues tokens for them (`platform-mesh-operator` uses them today for Local Providers), so creating or updating such a `Secret` is a token-minting path. The type is in the body, so `Secret` writes (create, update, patch, server-side apply) are inspected before forwarding and that type is refused;
  - `create` on `workspaces` below the Provider Workspace: not needed to reconcile an APIExport, and avoids unbounded workspaces under `:root:providers`.
- **`ProviderEndpoint`s are not in the Denylist**: kcp's maximal permission policy already makes them read-only ([02 APIs, "Connection Export"](./xxx-notes-011-v5-impl-02-apis.md)).
- **Plain Go functions** on the parsed request and its Connection, independent of how they're wired into the server, so they're tested on their own.
- **Denials are logged** with the Connection and the rule.

## Identity towards kcp

- **SA tokens**: each replica mints tokens for the fixed admin SA `default/provider` in the Provider Workspace of every Connection it has Sessions for, with TokenRequest through the front-proxy, caches them in memory, refreshes at half their lifetime, and never writes or sends them anywhere else.
- **The Connector's own identity** comes from a kcp-operator `Kubeconfig` per shard ([09 Deployment, "Connector"](./xxx-notes-011-v5-impl-09-deployment.md)): a client certificate for **one group** shared by all shards' Connectors. The identity needs:
  - read-only `apiexports/content` on the Connection Export;
  - in every Provider Workspace, `access` and `create` on `serviceaccounts/token` for that one SA (`resourceNames: [provider]`), granted by `platform-mesh-operator` ([03 Operator, "Connector RBAC"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
- **In the runtime cluster**: read access to the `ConnectorCASet` in `providers.platform-mesh.io` (public data, for the switch rule) and its own Connector Certificate `Secret`. Nothing else.
- **No `Shard` objects, no `/services/admin`, no cache server.**

## Egress to Providers

- Dial only Base URLs from `ProviderEndpoint`s; apply the same rules as `platform-mesh-operator` ([03 Operator, "Egress"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)): `https`, public addresses, resolve once, no redirects.
- Network policy allows the Connector outbound internet access and access to its virtual workspace and the front-proxy, nothing else ([09 Deployment](./xxx-notes-011-v5-impl-09-deployment.md)).

## Limits

- **Per Connection**: concurrent Streams, request rate, header sizes, in-flight requests. One Provider must not exhaust a Connector for others. The limits apply per replica, so a Connection's effective limit is R times the configured one; configured values account for that.
- **Per replica**: a ceiling on total Streams, to protect the replica's memory.
- **No request timeouts**: kcp applies its own to every request, with the SA's identity; a second, shorter one in the Connector would only cut watches and large LISTs.

## Observability

- **Metrics**, the Connector's own Prometheus collectors: Sessions per state, Streams, requests by outcome, latency, denials by rule, token mint outcomes, buffer pressure, and the Connector Certificate's expiry and how long a renewed one has waited ([04 Connector CA, "Signals"](./xxx-notes-011-v5-impl-04-connector-ca.md)). **No `connectionID` labels**: with thousands of Connections they'd explode cardinality; per-Connection detail is in the logs. Documented alert rules use only these metrics. Scraped through a `ServiceMonitor` like PM's other components ([09 Deployment, "Observability"](./xxx-notes-011-v5-impl-09-deployment.md)).
- **Request log**: one structured line per request with the Connection, the SA it acted as, verb, path, outcome and latency; metadata only, never bodies. This is the Connector's audit trail.
- **Readiness** reflects an expired Connector Certificate, so rollout tooling can hold.
- **Audience**: PM Administrators. Owners see their Connections' state on `Provider` conditions ([03 Operator, "Health"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)); Providers see theirs through the SDK.
- **Logs**: no tokens.

## Work breakdown

1. Service scaffold: `http.Server`, ops listener, configuration, following kcp's front-proxy (`kcp/pkg/proxy`).
2. Discovery and the Session rule.
3. Session manager: WebSocket, inner mutual TLS with pinning, Hello, yamux, replacement; the certificate switch rule.
4. Stream server: a listener per Session, `ConnContext`, header limits.
5. Path parsing; authorization, Denylist, `Secret`-type inspection.
6. Proxy: header stripping, token injection, host mapping, streaming, buffers.
7. Token cache.
8. Limits, metrics, request log.
9. Unit tests for the rules above, including the protocol's test vectors and the refusals.

## Decided

- **A standalone proxy server**, modelled on kcp's front-proxy; no generic apiserver, no request timeouts of its own, limits per Connection.
- **Path parsing**: a copy of kcp's `ClusterPathFromAndStrip`, then upstream `RequestInfoFactory`.
- **Replicas**: every replica serves every Connection its shard needs; rendezvous hashing over replicas remains an option if Session counts at Providers ever matter more.
- **Deployment**: a plain Deployment per shard, its identity from a kcp-operator `Kubeconfig`, the Connector Certificate as an ordinary volume ([09 Deployment, "Connector"](./xxx-notes-011-v5-impl-09-deployment.md)).
- **Workspace creation** below the Provider Workspace: denied.
- **Connector identity**: one group for all shards.
- **Where Service endpoint slices live**: a stated requirement for Providers: in the Provider Workspace, for exports in that workspace ([07 SDK](./xxx-notes-011-v5-impl-07-provider-sdk.md), [08 Quickstart](./xxx-notes-011-v5-impl-08-provider-quickstart.md)).

## Declined

- **A virtual workspace on kcp's virtual workspace framework**, deployed as a kcp-operator `VirtualWorkspace`. For a proxy, the generic handler chain's timeouts and shared in-flight limits would have to be switched off (its default resolver doesn't recognise `/clusters/…` or `/services/apiexport/…` as resource requests, so watches would be cut after 60 seconds and hold in-flight slots); it would need a Session authenticator and path rewriting just to fit the framework, and tie the Connector to kcp's Kubernetes version. The `VirtualWorkspace` resource would render a Service, serving certificate and `external` address nobody uses, and mount cache-server credentials. kcp's own front-proxy is the better model.
