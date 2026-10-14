# v5 implementation plan: connector

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). The **data plane**: a new service at `platform-mesh/services/providerconnections-vw`, built on the virtual workspace framework, one per kcp shard, deployed as a kcp-operator `VirtualWorkspace` with `shardRef` ([09 Deployment, "Connector (providerconnections-vw)"](./xxx-notes-011-v5-impl-09-deployment.md)). It opens sessions to Providers and proxies their requests to kcp with each connection's SA token.

## Responsibilities

**Owns:**
- deciding which sessions its shard needs, and opening, keeping and replacing them;
- the PM end of the session protocol ([01 Protocol, "Sessions"](./xxx-notes-011-v5-impl-01-protocol.md));
- authenticating, authorizing, auditing and proxying every request that arrives over a session;
- minting and caching SA tokens.

**Doesn't own:**
- records or anything else in kcp (it writes only TokenRequests);
- connection secrets or control calls ([03 Operator](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md));
- issuing its certificate ([04 Connector CA](./xxx-notes-011-v5-impl-04-connector-ca.md)).

## Principles

- **Stateless** (overview, principle 5): everything comes from its deployment (shard name, its virtual-workspace host, the front-proxy host, its connector certificate) and the connection export.
- **The Provider's only path into kcp**: whatever it allows, the Provider can do; the policy is the security guarantee and is tested as such ([10 Testing](./xxx-notes-011-v5-impl-10-testing.md)).
- **kcp's serving stack, not a hand-written proxy**: authentication, authorization, audit, timeouts, limits and metrics come from the generic handler chain.
- **Forward bytes, don't decode objects.** The connector never parses response bodies.

## Structure

```
 providerconnections-vw @ shard (one process per replica)
 ┌───────────────────────────────────────────────────────────────────────────────┐
 │ session controller (post-start hook)                                          │
 │   discovery: connection export (ProviderEndpoints, claimed endpoint slices)   │
 │   decides sessions for this shard; every replica serves every connection     │
 │   dials WebSocket → inner mutual TLS → hello → yamux (Provider opens streams) │
 │                     │                                                         │
 │                     ▼                                                         │
 │ in-process listener per session, tagged with connectionID                     │
 │   full handler chain: authn → authz → audit → timeouts → limits               │
 │     authn: session authenticator (connection principal from the tag)          │
 │     authz: own connection's paths only + denylist                             │
 │   providerconnections handler: check Host, strip credentials, add SA token,   │
 │     map host, reverse-proxy → this shard's virtual workspace or front-proxy   │
 └───────────────────────────────────────────────────────────────────────────────┘
```

## Discovery

- **Consumes the connection export's endpoint slice** through the front-proxy, and list/watches through every shard's virtual-workspace URL in it, with read-only `apiexports/content`.
- **Sees two things**:
  - `ProviderEndpoint`s, each arriving with its provider workspace's logical cluster; the endpoint that delivered it tells which shard hosts the workspace;
  - the provider workspaces' Service `apiexportendpointslices`, through the export's permission claim.
- **An outage of one shard's virtual workspace** freezes the records and slices from that shard; informers keep the last state.

## Which sessions

A shard's connector keeps a session for a connection when either:
- **the connection's `ProviderEndpoint` came from its own shard's endpoint**: the provider workspace is hosted here, so this connector carries the front-proxy session, from the start; or
- **one of the connection's Service endpoint slices lists its own shard's virtual-workspace URL**, counting only slices in the provider workspace that point at an export in that same workspace.

The rule follows kcp's own: a shard is listed in a slice exactly when it has a consumer binding.

## Replicas

Every replica of a shard's connector applies the session rule on its own and keeps a session for every connection the shard needs. Replicas don't know about each other.
- **Why it works**: sessions aren't exclusive. A connection has one session per replica per shard, and the Provider-side spreads new streams across all sessions serving a host ([01 Protocol, "Sessions and streams"](./xxx-notes-011-v5-impl-01-protocol.md)).
- **What replicas are for**: availability, and spreading traffic. A crashed replica cuts only the streams on its own sessions; client-go reopens them on the other replicas' sessions at once. Rollouts go one replica at a time without a gap: a leaving replica sends go-away and drains. Requests and watches spread over the replicas, so each carries about an R-th of the shard's traffic.
- **What it costs**: every replica holds a session for every connection, so a Provider sees up to R sessions per (connection, shard), where R is the replica count (typically 2–3); [07 SDK, "Requirements on Providers"](./xxx-notes-011-v5-impl-07-provider-sdk.md) says so. Each replica also mints its own SA tokens.
- **Scale-up** opens the new replica's sessions spread with jitter, so Providers aren't hit at once.
- **If session counts at Providers ever matter more** than spreading traffic, replicas can divide connections by rendezvous hashing of `connectionID` over the live replicas instead. That's a contained change inside the connector: no protocol, SDK or API change, and a rollout between the two is harmless because sessions aren't exclusive.

## Opening a session

1. **Dial the base URL** from the `ProviderEndpoint`, under the egress rules (see "Egress to Providers"), and upgrade to a WebSocket. The upgrade carries no credentials.
2. **Run the inner TLS client handshake** with the connector certificate as client certificate, accepting only server keys from the `ProviderEndpoint`. That handshake is the session's authentication ([01 Protocol, "Inner layer"](./xxx-notes-011-v5-impl-01-protocol.md)); a failure ends with a TLS alert (e.g. `unknown_ca`), which the connector reports in its metrics.
3. **Send the hello**: `clusterName` (the logical cluster of the record it acts on), `hosts` (the shard's virtual-workspace host from the connector's configuration; during a host rename, both the old and the new one) and an opaque `sub` for audit ([01 Protocol, "Hello"](./xxx-notes-011-v5-impl-01-protocol.md)).
4. **Start yamux** and accept streams opened by the Provider-side.
5. **Replace sessions** before the edge's maximum connection age (configurable), gracefully; reconnect with backoff on failure. An upgrade refused with `503` and `Retry-After` is retried using the value as the lower bound for the normal reconnect backoff; a session the Provider-side ends gracefully (go-away) is reopened at once. The connector assumes nothing about which Provider replica a session reaches ([01 Protocol, "Provider replicas"](./xxx-notes-011-v5-impl-01-protocol.md)). No replacement while the connector's own certificate is expired (see below).

### Connector certificate

cert-manager renews the connector certificate every 12 hours, for 24 hours each ([04 Connector CA, "Connector certificates"](./xxx-notes-011-v5-impl-04-connector-ca.md)). The connector keeps its current certificate and key in memory and decides when to use a renewed one:
- **Switch rule**: use the renewed certificate once its CA is in the trusted set and marked `converged` in the `ConnectorCABundle`'s status (public data; the connector may read it), or when the current certificate has less than about an hour left, whichever comes first. A renewal under the same CA switches at once; one from a new CA waits until every connection platform-mesh-operator reaches trusts that CA, normally minutes. Open sessions keep their handshake either way.
- **Switching replaces no session.** Open sessions keep the certificate they were opened with. Sessions still on the old CA's certificate when platform-mesh-operator drops the old CA are closed by the Provider-side with go-away and a grace period, and the connector reopens them at once with its current certificate ([01 Protocol, "Connector CA bundle"](./xxx-notes-011-v5-impl-01-protocol.md)).
- **A forced switch** (the current certificate about to expire, its successor's CA not converged, e.g. because platform-mesh-operator is down): only new sessions use the new certificate, failing at Providers that don't pin its CA yet.
- **After a restart**, only the newest certificate is on disk, so the connector uses it at once. If its CA hasn't converged yet, sessions to connections that don't pin it yet fail until platform-mesh-operator's push lands, normally within minutes.
- **Reported**: how long a renewed certificate has been waiting to be used, and the current certificate's expiry (metrics, see [04 Connector CA, "Signals"](./xxx-notes-011-v5-impl-04-connector-ca.md)).

**Provider downtime doesn't affect certificate renewal.** Providers pin the connector CA, not connector certificates, so a renewal needs nothing from them. While a Provider is down, the connector retries with backoff; when it's back, the connector dials with its current certificate. If a rollover happened meanwhile, platform-mesh-operator's first successful control call brings the Provider the current trusted set, and the next dial succeeds. Open sessions aren't affected when the certificate that opened them expires: the handshake was checked when the session opened, and TLS doesn't re-check expiry on an open connection.

**cert-manager is a dependency.** Renewal normally overlaps by 12 hours: a renewed certificate is in place while the current one has half its life left. Only if renewal keeps failing past that (cert-manager down, the issuer broken, requests not approved) does a certificate expire. Then the connector opens no new sessions and replaces none; open sessions keep carrying traffic until something else ends them. Readiness reports the expired state, so rollouts hold; replicas shouldn't be restarted until renewal resumes. The signals warn well before expiry ([04 Connector CA, "Signals"](./xxx-notes-011-v5-impl-04-connector-ca.md)); what an outage looks like in detail is in the runbook.

## Serving streams

- **Hand the stream to an in-process HTTP server** whose connection context carries the session's `connectionID`. Streams carry HTTP/1.1 from their first byte; there's nothing to read before. The server's handler is its **full handler chain**, not the virtual workspace handler alone.
- **Path rewriting**: the server prefixes the request path with `/services/providerconnections/<connectionID>`, keeping `Host`, so the virtual workspace framework routes it.
- **Header limits** per request apply as for any HTTP server, so reading a request's headers before its `Host` is checked stays bounded.

## Authentication

A session authenticator in the server's authenticator union:
- maps the `connectionID` from the connection context to the **connection principal** (e.g. `pm:connection:<id>`);
- only in-process listeners set that context; requests from the network never carry it.

## Authorization and policy

The virtual workspace's authorizer, for the connection principal:
- **Scope**: only its own connection's paths, i.e. its provider workspace through the front-proxy (`/clusters/<its clusterName>/…`), and the APIExport virtual-workspace endpoints of exports in that workspace. Both kcp path shapes are parsed explicitly.
- **Denylist** (v5, "Connector policy"):
  - `update`, `patch`, `delete` on the connection export's APIBinding;
  - writes to `apiexportendpointslices/status`;
  - `create` on `serviceaccounts/token`;
  - `create` on `certificatesigningrequests` and their approval;
  - upgrade requests (`exec`, `attach`, `port-forward`);
  - **Secrets of type `kubernetes.io/service-account-token`**: kcp issues tokens for them (platform-mesh-operator uses them today for local Providers), so creating or updating such a Secret is a token-minting path. The authorizer can't see a body's type, so the proxy handler inspects Secret writes (create, update, patch, server-side apply) before forwarding and refuses that type;
  - `create` on `workspaces` below the provider workspace: not needed to reconcile an APIExport, and avoids unbounded workspaces under `:root:providers`.
- **Records are not in the denylist**: kcp's maximal permission policy already makes them read-only ([02 APIs, "Connection export"](./xxx-notes-011-v5-impl-02-apis.md)).
- **Denials are audited** with the connection principal.

## Proxying

For each authorized request:
1. **Check `Host`** against the allow-list: this shard's virtual-workspace host and the front-proxy host, from the connector's deployment configuration. It's the same list for every connection; anything else is refused. This is the only target check: the stream carries no separate target.
2. **Strip credentials and identity headers**: `Authorization`, `Impersonate-*`, `Proxy-*`, cookies, hop-by-hop headers.
3. **Add the SA token** for the connection (see "Identity towards kcp").
4. **Map the external host to an in-cluster address** for the dial, keeping `Host` and the TLS server name, so traffic doesn't hairpin through PM's load balancer ([09 Deployment, "Network"](./xxx-notes-011-v5-impl-09-deployment.md)).
5. **Forward over TLS** verified with PM's own trust for kcp; resolve once, dial the checked address.
6. **Stream the response** without buffering; watches and LISTs pass through with immediate flushing.

Settings that matter:
- **Long-running requests and timeouts**: classify watches and proxied LISTs as long-running, or raise their timeout, so the generic chain's default doesn't cut them.
- **Buffers**: a small copy buffer per stream, and a cap on the upstream HTTP/2 receive buffer per stream, so a slow Provider's backlog stays upstream ([10 Testing, "Load"](./xxx-notes-011-v5-impl-10-testing.md)).
- **Hop budget**: confirm the extra hop's effect on `X-Kcp-Virtual-Resource-Hops`.

## Identity towards kcp

- **SA tokens**: each replica mints tokens for the SA named in the `ProviderEndpoint` of every connection it has sessions for, with TokenRequest through the front-proxy, caches them in memory, refreshes at half their lifetime, and never writes or sends them anywhere else.
- **The connector's own identity** comes from the `VirtualWorkspace`'s `kubeconfigSecretRef`, which must always be set: without it, kcp-operator mounts its privileged logical-cluster-admin credential into the pod. All shards' connectors share **one group**. The identity needs:
  - read-only `apiexports/content` on the connection export;
  - in every provider workspace, `access` and `create` on `serviceaccounts/token` for that one SA, granted by platform-mesh-operator ([03 Operator, "Connector RBAC"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
- **In the runtime cluster**: read access to the `ConnectorCABundle` (public data, for the switch rule) and its own connector certificate Secret. Nothing else.
- **No `Shard` objects, no `/services/admin`.** The connector doesn't use the cache server, but kcp-operator mounts cache-server credentials into every `VirtualWorkspace` when a cache server is configured, as in PM. **Accepted and documented** as part of a compromised connector's blast radius ([09 Deployment, "Dependencies on kcp-operator"](./xxx-notes-011-v5-impl-09-deployment.md)).

## Egress to Providers

- Dial only base URLs from `ProviderEndpoint`s; apply the same rules as platform-mesh-operator ([03 Operator, "Egress"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)): `https`, public addresses, resolve once, no redirects.
- Network policy allows the connector outbound internet access and in-cluster access to its virtual workspace and the front-proxy, nothing else ([09 Deployment](./xxx-notes-011-v5-impl-09-deployment.md)).

## Limits

Per connection: concurrent streams, request rate, header sizes, in-flight requests. One Provider must not exhaust a connector for others. The limits apply per replica, so a connection's effective limit is R times the configured one; configured values account for that.

## Observability

- **Metrics**: sessions per state, streams per session, requests by connection and outcome (via `apiserver_request_*`), denials by rule, token mint outcomes, buffer pressure, and the connector certificate's expiry and how long a renewed one has waited ([04 Connector CA, "Signals"](./xxx-notes-011-v5-impl-04-connector-ca.md)). Scraped through a `ServiceMonitor` like PM's other components ([09 Deployment, "Observability"](./xxx-notes-011-v5-impl-09-deployment.md)).
- **Readiness** reflects an expired connector certificate, so rollout tooling can hold.
- **Audience**: the people running the PM installation. Owners see their connections' state on `Provider` conditions ([03 Operator, "Health"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)); Providers see theirs through the SDK.
- **Audit**: every request with the connection principal and the SA it acted as; at metadata level, never bodies.
- **Logs**: no tokens.

## Work breakdown

1. Service scaffold on the virtual workspace framework (reusing patterns from `services/virtual-workspaces` and kcp's initializing-workspaces virtual workspace, including a copy of its proxy helper).
2. Discovery and the session rule.
3. Session client: WebSocket, inner mutual TLS with pinning, hello, yamux, replacement; the certificate switch rule.
4. Stream serving: in-process listener, context tagging, path rewriting.
5. Authenticator, authorizer, denylist, Secret-type inspection.
6. Proxy handler: header stripping, token injection, host mapping, streaming, settings.
7. Token cache.
8. Limits, metrics, audit configuration.

## Decided

- **Replicas**: every replica serves every connection its shard needs; rendezvous hashing over replicas remains an option if session counts at Providers ever matter more.
- **Connector certificate mount**: through the `VirtualWorkspace`'s `extraVolumes` and `extraVolumeMounts` ([09 Deployment, "Connector (providerconnections-vw)"](./xxx-notes-011-v5-impl-09-deployment.md)).
- **Workspace creation** below the provider workspace: denied.
- **Connector identity**: one group for all shards; `kubeconfigSecretRef` always set.
- **Cache-server credentials**: accepted, documented.
- **Where Service endpoint slices live**: a stated requirement for Providers: in the provider workspace, for exports in that workspace ([07 SDK](./xxx-notes-011-v5-impl-07-provider-sdk.md), [08 Quickstart](./xxx-notes-011-v5-impl-08-provider-quickstart.md)).
