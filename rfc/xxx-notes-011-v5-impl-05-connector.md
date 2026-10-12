# v5 implementation plan: connector

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). The **data plane**: a new service at `platform-mesh/services/providerconnections-vw`, built on the virtual workspace framework, one per kcp shard, deployed as a kcp-operator `VirtualWorkspace` with `shardRef` ([09 Deployment, "Connector"](./xxx-notes-011-v5-impl-09-deployment.md)). It opens sessions to Providers and proxies their requests to kcp with each connection's SA token.

## Responsibilities

**Owns:**
- deciding which sessions its shard needs, and opening, keeping and replacing them;
- the PM end of the session protocol ([01 Protocol, "Sessions"](./xxx-notes-011-v5-impl-01-protocol.md));
- authenticating, authorizing, auditing and proxying every request that arrives over a session;
- minting and caching SA tokens.

**Doesn't own:**
- records or anything else in kcp (it writes only TokenRequests);
- connection secrets or control calls ([03 Operator](./xxx-notes-011-v5-impl-03-providerconnections-operator.md));
- issuing its certificate ([04 Session CA](./xxx-notes-011-v5-impl-04-session-ca.md)).

## Principles

- **Stateless** (overview, principle 5): everything comes from its deployment (shard name, its virtual-workspace host, the front-proxy host, its shard certificate) and the connection export.
- **The Provider's only path into kcp**: whatever it allows, the Provider can do; the policy is the security guarantee and is tested as such ([10 Testing](./xxx-notes-011-v5-impl-10-testing.md)).
- **kcp's serving stack, not a hand-written proxy**: authentication, authorization, audit, timeouts, limits and metrics come from the generic handler chain.
- **Forward bytes, don't decode objects.** The connector never parses response bodies.

## Structure

```
 providerconnections-vw @ shard (one process per replica)
 ┌───────────────────────────────────────────────────────────────────────────────┐
 │ session controller (post-start hook)                                          │
 │   discovery: connection export (ProviderEndpoints, claimed endpoint slices)   │
 │   decides sessions for this shard; owns a share of connections (replicas)     │
 │   dials WebSocket → assertion → inner TLS → yamux (Provider opens streams)    │
 │                     │ each stream: preamble target check                      │
 │                     ▼                                                         │
 │ in-process listener per session, tagged with connectionID                     │
 │   full handler chain: authn → authz → audit → timeouts → limits               │
 │     authn: session authenticator (connection principal from the tag)          │
 │     authz: own connection's paths only + denylist                             │
 │   providerconnections handler: strip credentials, add SA token, map host,     │
 │     reverse-proxy → this shard's virtual workspace or the front-proxy         │
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

Several replicas per shard share the connections:
- **Rendezvous hashing of `connectionID`** over live replicas; membership from the connector Service's EndpointSlices (or a Lease per replica).
- **Overlaps are harmless**: sessions aren't exclusive; two replicas briefly serving a connection both connect.
- **Hand over before cutting off** on graceful shutdown: the new owner connects, the old one sends go-away and drains.
- **Replica identity**: kcp-operator's `VirtualWorkspace` renders a Deployment, so pod names change on restart and a restarted replica's connections move to other replicas once (relisting on that shard). **Accepted**; graceful rollouts hand over first, so only crashes cost relists.

## Opening a session

1. **Sign a session assertion** with the mounted shard certificate's key: `aud` the connection's session URL, `cluster` the provider workspace's `clusterID`, short `exp`, fresh `jti` ([01 Protocol, "Outer layer"](./xxx-notes-011-v5-impl-01-protocol.md)).
2. **Dial the base URL** from the `ProviderEndpoint`, under the egress rules (see "Egress to Providers"), and upgrade to a WebSocket.
3. **Run the inner TLS client handshake** with the shard certificate, accepting only server keys from the `ProviderEndpoint`.
4. **Start yamux** and accept streams opened by the Provider-side.
5. **Replace sessions** before the edge's maximum connection age (configurable), gracefully; reconnect with backoff on failure.

The shard certificate is reloaded when cert-manager renews it; existing sessions keep their handshake, new ones use the new certificate.

## Serving streams

- **Preamble first**: read the stream's target, check it against the allow-list (this shard's virtual-workspace host and the front-proxy host, from deployment configuration). Refuse anything else before reading HTTP.
- **Hand the stream to an in-process HTTP server** whose connection context carries the `connectionID` and the target. Its handler is the server's **full handler chain**, not the virtual workspace handler alone.
- **Path rewriting**: the server prefixes the request path with `/services/providerconnections/<connectionID>`, keeping `Host`, so the virtual workspace framework routes it.
- **Host check**: each request's `Host` must equal the preamble's target.

## Authentication

A session authenticator in the server's authenticator union:
- maps the `connectionID` from the connection context to the **connection principal** (e.g. `pm:connection:<id>`);
- only in-process listeners set that context; requests from the network never carry it.

## Authorization and policy

The virtual workspace's authorizer, for the connection principal:
- **Scope**: only its own connection's paths, i.e. its provider workspace through the front-proxy (`/clusters/<its clusterID>/…`), and the APIExport virtual-workspace endpoints of exports in that workspace. Both kcp path shapes are parsed explicitly.
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
1. **Strip credentials and identity headers**: `Authorization`, `Impersonate-*`, `Proxy-*`, cookies, hop-by-hop headers.
2. **Add the SA token** for the connection (see "Identity towards kcp").
3. **Map the external host to an in-cluster address** for the dial, keeping `Host` and the TLS server name, so traffic doesn't hairpin through PM's load balancer ([09 Deployment, "Network"](./xxx-notes-011-v5-impl-09-deployment.md)).
4. **Forward over TLS** verified with PM's own trust for kcp; resolve once, dial the checked address.
5. **Stream the response** without buffering; watches and LISTs pass through with immediate flushing.

Settings that matter:
- **Long-running requests and timeouts**: classify watches and proxied LISTs as long-running, or raise their timeout, so the generic chain's default doesn't cut them.
- **Buffers**: a small copy buffer per stream, and a cap on the upstream HTTP/2 receive buffer per stream, so a slow Provider's backlog stays upstream ([10 Testing, "Load"](./xxx-notes-011-v5-impl-10-testing.md)).
- **Hop budget**: confirm the extra hop's effect on `X-Kcp-Virtual-Resource-Hops`.

## Identity towards kcp

- **SA tokens**: the replica that owns a connection mints tokens for the SA named in the `ProviderEndpoint` with TokenRequest through the front-proxy, caches them in memory, refreshes at half their lifetime, and never writes or sends them anywhere else.
- **The connector's own identity** comes from the `VirtualWorkspace`'s `kubeconfigSecretRef`, which must always be set: without it, kcp-operator mounts its privileged logical-cluster-admin credential into the pod. All shards' connectors share **one group**. The identity needs:
  - read-only `apiexports/content` on the connection export;
  - in every provider workspace, `access` and `create` on `serviceaccounts/token` for that one SA, granted by the Provider controller ([06 Provider controller, "Connector RBAC"](./xxx-notes-011-v5-impl-06-provider-controller.md)).
- **No `Shard` objects, no `/services/admin`.** The connector doesn't use the cache server, but kcp-operator mounts cache-server credentials into every `VirtualWorkspace` when a cache server is configured, as in PM. **Accepted and documented** as part of a compromised connector's blast radius ([09 Deployment, "Dependencies on kcp-operator"](./xxx-notes-011-v5-impl-09-deployment.md)).

## Egress to Providers

- Dial only base URLs from `ProviderEndpoint`s; apply the same rules as the operator ([03 Operator, "Egress"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)): `https`, public addresses, resolve once, no redirects.
- Network policy allows the connector outbound internet access and in-cluster access to its virtual workspace and the front-proxy, nothing else ([09 Deployment](./xxx-notes-011-v5-impl-09-deployment.md)).

## Limits

Per connection: concurrent streams, request rate, header sizes, in-flight requests. One Provider must not exhaust a connector for others.

## Observability

- **Metrics**: sessions per state, streams per session, requests by connection and outcome (via `apiserver_request_*`), denials by rule, token mint outcomes, buffer pressure.
- **Audit**: every request with the connection principal and the SA it acted as; at metadata level, never bodies.
- **Logs**: no tokens, no assertions.

## Work breakdown

1. Service scaffold on the virtual workspace framework (reusing patterns from `services/virtual-workspaces` and kcp's initializing-workspaces virtual workspace, including a copy of its proxy helper).
2. Discovery and the session rule.
3. Replica membership and rendezvous hashing.
4. Session client: assertion signing, WebSocket, inner TLS with pinning, yamux, replacement.
5. Stream serving: preamble, in-process listener, context tagging, path rewriting.
6. Authenticator, authorizer, denylist, Secret-type inspection.
7. Proxy handler: header stripping, token injection, host mapping, streaming, settings.
8. Token cache.
9. Limits, metrics, audit configuration.

## Decided

- **Replica identity**: accept connection moves on restart.
- **Shard certificate mount**: through the `VirtualWorkspace`'s `extraVolumes` and `extraVolumeMounts` ([09 Deployment, "Connector"](./xxx-notes-011-v5-impl-09-deployment.md)).
- **Workspace creation** below the provider workspace: denied.
- **Connector identity**: one group for all shards; `kubeconfigSecretRef` always set.
- **Cache-server credentials**: accepted, documented.
- **Where Service endpoint slices live**: a stated requirement for Providers: in the provider workspace, for exports in that workspace ([07 SDK](./xxx-notes-011-v5-impl-07-provider-sdk.md), [08 Quickstart](./xxx-notes-011-v5-impl-08-provider-quickstart.md)).
