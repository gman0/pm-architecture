# Notes v5: a Connection gateway on the Provider-side

Working notes for [011](./011-provider-onboarding-flow-improvements.md), next to the [v5 notes](./xxx-notes-011-provider-onboarding-flow-improvements-v5.md) and the [implementation plan](./xxx-notes-011-v5-impl-00-overview.md). Not the RFC itself. The question: can the SDK's server run as a separate Deployment, delivered by PM as a chart, so that the Provider's operator needs only the smallest changes and stays as close as possible to standard client-go and multicluster-runtime?

**Decision**: variant A without the provider package. The SDK is delivered as `provider-gateway`, and the operator uses standard client-go, multicluster-runtime and kcp-multicluster-provider with a kubeconfig per Connection (`proxy-url` with a credential per Connection, `tls-server-name` with `provider-gateway`'s one certificate). Its known cost, a connection pool per client, is accepted, within the Connector's per-Connection limits; a shared transport per Connection is documented as an optional optimisation. The plan describes it in [07 SDK](./xxx-notes-011-v5-impl-07-provider-sdk.md). The rest of these notes is the analysis behind it.

**Short answer**: yes. A **Connection gateway** takes over everything the SDK's Connection API server does, and the operator reaches kcp through it. Two variants were considered: the gateway as a forward proxy (`proxy-url` in kubeconfigs), or as a reverse proxy addressed by path. The reverse proxy keeps client-go's transport sharing and needs only a URL-mapping function in a public option of kcp-multicluster-provider, which a small multicluster-runtime provider package can hide entirely.

## Why

Today the SDK is a library compiled into the Provider's operator ([07 SDK](./xxx-notes-011-v5-impl-07-provider-sdk.md)):
- **The operator runs in the SDK's process**: "the dial hook only reaches Sessions held by its own process", so controllers run in the same process as the Connection API server, with one active replica (07, "Replicas").
- **Protocol upgrades need the Provider to rebuild** its operator against a new SDK release; until then the `Provider` shows `ProtocolOutdated`.
- **The operator must be written in Go** and use the SDK's transport and registry.

A gateway separates the two: PM ships the security-critical code as a component, versioned with the protocol, and the operator becomes an ordinary multicluster controller.

## The gateway

A Deployment in the Provider's cluster, from a chart PM delivers, running a PM-built image:
- **Owns everything the SDK's server owns today**: the Connection API under the Base URL (Registration, Control Sessions, Sessions with Inner TLS, the Hello and yamux), Server Keys, the `Connection`, `OnboardingToken` and `ServerKey` resources and their controllers, health, metrics and Events. The protocol ([01 Protocol](./xxx-notes-011-v5-impl-01-protocol.md)) doesn't change; only where it runs does.
- **Holds Sessions in memory**, in its active replica. PM's Connectors dial the Base URL as today and end at the gateway.
- **Serves the operator in the cluster**: an in-cluster endpoint through which the operator's requests reach kcp over the Connection's Sessions (the two variants below differ here).
- **Writes one kubeconfig `Secret` per Connection** into its own namespace, which the operator reads through a Role the Provider adds, labelled with the `connectionID`. It represents the Connection, not its Sessions: created with the `Connection` at Registration, deleted with it, unchanged while Sessions come and go. Whether Sessions are up is the `Connection`'s `Ready`.
- **Stays inside the Provider's trust boundary**, like the SDK today; it sees the operator's traffic in the clear.

```
  Provider's cluster (inside the trust boundary)
 ┌──────────────────────────────────────────────────────────────────────────────┐
 │  Connection gateway (PM-delivered chart + image)                             │
 │    Connection API: Registration, Control Sessions, Sessions (Inner TLS,      │◀══ Base URL (through the Edge)
 │    Hello, yamux), Server Keys, the SDK's CRDs and controllers                │
 │    in-cluster endpoint for the operator ──────┐                              │
 │    one kubeconfig Secret per Connection       │                              │
 │                                               │ HTTPS, the gateway's CA      │
 │  Provider's operator                          │                              │
 │    reads the kubeconfig Secrets ──────────────┘                              │
 │    client-go, multicluster-runtime, kcp-multicluster-provider                │
 └──────────────────────────────────────────────────────────────────────────────┘
```

### Restarts and state

- **A restart or failover of the active replica loses its Sessions**, as an SDK restart does today: PM's Connectors reconnect, get `503` with `Retry-After` from standbys, and land on the new leader within roughly 10–20 seconds. Standbys fail readiness on the in-cluster endpoint, so the operator's Service reaches only the leader.
- **The operator's open requests and watches break** and client-go reconnects; informers resume their watches, or relist if kcp no longer has the resource version. The operator itself keeps running with warm caches, which is better than today, where it shares the SDK's process.
- **The kubeconfig `Secret`s stay valid**: they name the gateway's Service, not a pod, and hold only persisted data. Every piece of state that must survive a restart lives in resources, never only in memory: the self-generated CA's key (without cert-manager), the Connections' credentials (their hashes, e.g. in the `Connection`'s status), and what the SDK keeps today. That's 07's existing principle ("The Kubernetes API is the only source of truth").
- **The `Secret`s change only** when a `Connection` is created or deleted, when a Connection's credential is rotated, and when the CA's trust bundle changes, which is rare and deliberate.

### Certificates

The standard setup for in-cluster services: a long-lived CA that clients trust and that practically never changes, issuing short-lived serving certificates that renew on their own:
- **With cert-manager**, the chart's default: a `SelfSigned` Issuer, a long-lived CA `Certificate`, a CA Issuer and a serving `Certificate` for the gateway's Service names. The gateway mounts the serving certificate and reloads it on renewal; its `ca.crt` goes into every kubeconfig `Secret`.
- **Without cert-manager**, a chart value: the gateway generates a CA once, without expiry, into a `Secret` in its namespace, and issues serving certificates from it in memory.
- **Changing the CA** is rare and deliberate; the overlap is the ecosystem's job (e.g. trust-manager), not something the gateway reimplements.
- **One CA per gateway, not per Connection.** Per-Connection CAs would add little: every key would sit in the same process anyway, the kcp hostnames are the same for every Connection of a PM Instance, and each Connection's credential already binds requests to it.

## Variant A: forward proxy (`proxy-url`)

The operator uses kcp's own URLs, and client-go tunnels each connection through the gateway with `CONNECT`.

**The kubeconfig `Secret`:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: connection-c-7f3k9q2mx8w4
  namespace: wild-west-system                  # the gateway's namespace
  labels: { provider-sdk.platform-mesh.io/connection: c-7f3k9q2mx8w4 }
stringData:
  kubeconfig: |
    clusters:
    - name: workspace
      cluster:
        server: https://front-proxy.pm.invalid/clusters/3y9j0def   # placeholder; never resolved
        proxy-url: https://c-7f3k9q2mx8w4:<credential>@gateway.wild-west-system.svc:3128
        tls-server-name: gateway.wild-west-system.svc            # verify every handshake against the gateway's name
        certificate-authority-data: <the gateway's serving certificate (self-signed), or its CA>
    # no user: towards kcp, the Connector adds the SA token
```

**One request:**
```
 operator (client-go)        gateway                              Connector (PM)          kcp
   │ CONNECT nereus.kcp…:443   │                                      │                     │
   │ Proxy-Authorization: c-7f…│ credential → Connection c-7f…        │                     │
   │──────────────────────────▶│ pick a Session serving nereus.kcp…   │                     │
   │                           │ open a yamux Stream on it ══════════▶│                     │
   │◀── 200 Connection Est. ───│                                      │                     │
   │ TLS, server name          │ terminates it with its own serving   │                     │
   │ "gateway.<ns>.svc" ══════▶│ certificate                          │                     │
   │ GET …/cowboys?watch=true  │ relays the plain HTTP onto the Stream ══════▶ route, Denylist,
   │                           │                                      │   SA token ───────▶ │
   │◀═══ watch events ═════════│◀══════════════════ Stream ═══════════│◀────────────────────│
```
- No Session for that host: the `CONNECT` gets `503`, and client-go backs off and retries.
- One `CONNECT` is one Stream; client-go's pooling, keep-alive and separate upgrade connections work as against kube-apiserver.
- **One certificate, thanks to `tls-server-name`.** client-go starts TLS inside the tunnel towards the target host and would expect a certificate for `nereus.kcp…`; the gateway has to terminate that TLS to put the HTTP on a Stream, and doesn't have kcp's certificate. The kubeconfig's `tls-server-name` (`rest.Config.TLSClientConfig.ServerName`) solves that: when `ServerName` is set, Go's `net/http` uses it for SNI and certificate verification instead of the target's hostname, for the proxy's handshake and for the one inside the tunnel alike. So client-go verifies every handshake against the gateway's own name, and the gateway presents one certificate for that name (self-signed and pinned, as in variant B), learning the target host from the `CONNECT`. `rest.CopyConfig` copies `ServerName`, and kcp-multicluster-provider only changes `Host`, so every derived config keeps it. To confirm with a test: Go fills in the target name only when `ServerName` is empty, which is its known behaviour but isn't in the code checked here.
- **Without `tls-server-name`**, the gateway would need a CA that issues certificates for any kcp host, minted per host on demand and cached, since the hostnames come from PM and change with shards and PM Instances. Whoever holds that CA key could impersonate any of those hosts to the operator; it's the fallback if the test above fails.
- The gateway's TLS offers ALPN `http/1.1` only: client-go would otherwise negotiate `h2` through the tunnel, while a Stream carries one HTTP/1.1 connection to the Connector, whose server doesn't speak h2c.

**What was checked** (client-go as vendored in kcp, `kcp/vendor/k8s.io/client-go`; kcp-multicluster-provider `fa033a7`):
- `proxy-url` in kubeconfigs is supported with `http`, `https` or `socks5` (`tools/clientcmd/helpers.go`, `parseProxyURL`) and becomes `rest.Config.Proxy = http.ProxyURL(u)` (`tools/clientcmd/client_config.go`); an explicit proxy ignores `HTTPS_PROXY` and `NO_PROXY`.
- client-go builds an `http.Transport` with that proxy and the kubeconfig's TLS config (`transport/cache.go`).
- kcp-multicluster-provider copies the `rest.Config` per endpoint and only changes `Host` (`pkg/provider/provider.go`; `NewScopedCluster` in `pkg/cache/cluster.go`), so the virtual-workspace URLs keep the proxy and the CA.
- From Go's `net/http` (not vendored there; to confirm with a test): an `https://` target through a proxy is reached with `CONNECT`, then TLS verified with the transport's `TLSClientConfig`, without resolving the target name; an `https://` proxy URL's own TLS uses the same `TLSClientConfig`; user and password in the proxy URL become `Proxy-Authorization: Basic` on the `CONNECT`.

**The problem: no transport sharing.** Pooling within one `http.Transport` works (Go keeps idle connections, tunnels included, per proxy and target host); what breaks is client-go sharing one transport between clients, which is what normally keeps connections few.

*Why client-go can't share it*:
1. **The cache needs a comparable key.** client-go keys its transport cache on a struct built from the config's TLS settings (`tlsCacheKey`, `transport/cache.go`): configs with an equal key get the same `http.Transport`. Go map keys must support `==`, and function values don't (except against `nil`); `Proxy` is a function, `func(*http.Request) (*url.URL, error)`. client-go gives up on it: `// cannot determine equality for functions`.
2. **Leaving it out of the key would be wrong**: two configs with different proxies would share a transport, and requests would go through the wrong proxy, here the wrong Connection. Refusing to cache is the only safe choice.
3. **client-go solved this for other functions, not for `Proxy`.** It wraps a dialer and a certificate callback in pointers to small structs, which are comparable: `DialHolder` ("This struct indirection is used to make transport configs cacheable") and `GetCertHolder` (`transport/config.go`). There's no `ProxyHolder`.
4. **A kubeconfig loses the URL early**: `proxy-url` is a string, which could be compared, but `tools/clientcmd` turns it into `http.ProxyURL(u)` at once (`client_config.go`), and the transport layer only sees the function, so identical proxy URLs never match.

Even a holder only helps callers that reuse it: `rest.Config.Dial` is wrapped in a new `DialHolder` on every `TransportConfig()` call (`rest/transport.go`), so a custom dialer set through `rest.Config` isn't shared either, which is why the SDK today hands each Connection one prepared `Transport`. An upstream fix would be small and compatible: a `ProxyHolder`, or keeping the proxy URL in `transport.Config` and keying the cache on it.

*What it costs* (kcp-multicluster-provider `fa033a7`; controller-runtime and Go's `net/http` from their known behaviour):
- **Transports per engaged logical cluster**: a controller-runtime client (`client.New` in `NewScopedCluster`, which builds its own HTTP client through `rest.HTTPClientFor` when none is passed), a second HTTP client for the REST mapper (`rest.HTTPClientFor` and `apiutil.NewDynamicRESTMapper`, same function), and an Event recorder's client when events are recorded there (`pkg/events/recorder/provider_manager.go`, `GetProvider`). Per endpoint, one wildcard cache (`NewWildcardCache`), where informers list and watch. Without a proxy, all of them share client-go's cached transport; with one, each has its own.
- **Memory at engagement: negligible.** Building a transport opens no connection, and the REST mapper is lazy.
- **Watches: unaffected**: informers run through the per-endpoint wildcard cache either way.
- **Connections: one pool per cluster instead of one shared pool.** A cluster's writes, uncached reads, discovery and Events go through its own transports; each first request opens a new connection, which in this variant is a `CONNECT`, a TLS handshake with the gateway and a new Stream across the internet to the Connector, then idles for Go's default 90 seconds (`IdleConnTimeout`, applied by `utilnet.SetTransportDefaults`), up to 25 per transport (`idleConnsPerHost`). Shared, the same traffic reuses a handful of connections per host, about the number of requests in flight; unshared, it's at least one per cluster written to in the last 90 seconds, times up to three transports.
- **Bursts hit the Connector's limits.** After an operator restart, a relist or a Rollover handover, controllers reconcile every consumer cluster, often writing status to each: thousands of tunnels within seconds. The Connector caps concurrent Streams per Connection ([05 Connector, "Limits"](./xxx-notes-011-v5-impl-05-connector.md)), sized for real concurrency; idle per-cluster connections hold Streams for 90 seconds and count against it, so a burst can exhaust it and the Connector refuses or delays new Streams for that Connection.
- **Latency**: an extra in-cluster round trip and handshake per new connection, noticeable only in bursts.

*The fix* is a shared `http.Transport` per Connection, built from the kubeconfig and set as `rest.Config.Transport` (with the CA moved into it, since client-go refuses a custom transport combined with TLS options), which `rest.CopyConfig` then shares by reference. That's custom transport code in the operator; the alternative, a much higher Stream limit, weakens the Connector's protection against one Provider exhausting it.

**Other costs**: TLS terminated twice per request; `socks5` is no alternative, as it would carry the credential in clear text inside the cluster.

## Variant B: reverse proxy addressed by path

The gateway is addressed like the Connector is, by path, and the operator's configs carry no proxy, so client-go's transport cache works as usual.

**The gateway's URLs**, one prefix per Connection:
- `https://gateway.<ns>.svc/connections/<connectionID>/workspace/…` for the Provider Workspace;
- `https://gateway.<ns>.svc/connections/<connectionID>/vw/<shard host>/…` for a shard's virtual workspace.

It strips the prefix, picks a Session serving that host (any Session for the workspace), and relays the request on a Stream with the real path and `Host`. It authenticates each request with the Connection's bearer token, checks it against the path's Connection, and strips it before forwarding.

**TLS** is only for its own Service name: one serving certificate from the gateway's CA. No per-host certificates, no CA that issues for arbitrary hosts.

**The kubeconfig `Secret`:**
```yaml
stringData:
  kubeconfig: |
    clusters:
    - name: workspace
      cluster:
        server: https://gateway.wild-west-system.svc/connections/c-7f3k9q2mx8w4/workspace/clusters/3y9j0def
        certificate-authority-data: <the gateway's CA>
    users:
    - name: connection
      user:
        token: <the Connection's gateway token>   # checked and stripped by the gateway; never reaches PM
```

**The operator's side:**
- **The workspace** works as is: kcp-multicluster-provider joins `/clusters/<name>` onto `Host` (`url.JoinPath` in `NewScopedCluster`), so the path prefix survives.
- **Virtual-workspace URLs** need one mapping: endpoint slices list kcp's real URLs, which would bypass the gateway. kcp-multicluster-provider has a public option for exactly this step, `provider.Options.ExtractURLsFromEndpointSlice` on `provider.NewProvider` (`pkg/provider/provider.go`). The operator passes a function mapping `https://nereus.kcp…/services/apiexport/…` to `https://gateway…/connections/<id>/vw/nereus.kcp…/services/apiexport/…`. The convenience wrapper `apiexport.New` hardcodes the default extractor (`apiexport/provider.go`), so the operator uses `provider.NewProvider` with `APIExportEndpointSlice` as the slice type; both are public.
- **Transport sharing works**: without `Proxy`, client-go shares one transport across every client with the same TLS settings, across all engaged clusters and even across Connections, since bearer tokens aren't part of the cache key.
- **Several Connections**: one kcp provider per Connection, composed with cluster names prefixed by `connectionID`. multicluster-runtime has providers for composing providers by prefix and for discovering clusters from kubeconfig `Secret`s; their exact APIs at v0.23 are to confirm.

**Clients for the virtual-workspace URLs, step by step** (kcp-multicluster-provider `fa033a7`):
1. **The provider reads the endpoint slice through the gateway.** `provider.NewProvider(cfg, sliceName, opts)` gets the Connection's kubeconfig config, whose `Host` is the gateway's workspace URL, `https://gateway.<ns>.svc/connections/c-7f…/workspace/clusters/3y9j0def`, and builds a cache for that one `APIExportEndpointSlice`, selected by name (`pkg/provider/provider.go`). The gateway strips `/connections/c-7f…/workspace` and relays the rest over any Session, to the front-proxy.
2. **The slice lists kcp's real URLs**, e.g. `https://root.kcp.example.com/services/apiexport/3y9j0def/cowboys` and `https://nereus.kcp.example.com/services/apiexport/3y9j0def/cowboys`.
3. **The URL function maps them to the gateway.** On every slice update the provider calls `opts.ExtractURLsFromEndpointSlice(obj)` (`endpointSliceUpdate`); the default reads `status.endpoints[].url` (`pkg/provider/condition.go`). The provider package wraps it:
   ```go
   func gatewayURLs(gatewayBase, connectionID string) func(client.Object) ([]string, error) {
       return func(obj client.Object) ([]string, error) {
           urls, err := provider.DefaultExtractURLsFromEndpointSlice(obj)
           if err != nil {
               return nil, err
           }
           out := make([]string, 0, len(urls))
           for _, raw := range urls {
               u, err := url.Parse(raw)
               if err != nil {
                   return nil, err
               }
               // https://nereus.kcp…/services/apiexport/… → https://gateway…/connections/<id>/vw/nereus.kcp…/services/apiexport/…
               out = append(out, gatewayBase+"/connections/"+connectionID+"/vw/"+u.Host+u.EscapedPath())
           }
           return out, nil
       }
   }
   ```
   giving `https://gateway.<ns>.svc/connections/c-7f…/vw/nereus.kcp.example.com/services/apiexport/3y9j0def/cowboys`, and the same for `root`.
4. **A config per endpoint, copied from the workspace config.** For each new URL the provider does `cfg := rest.CopyConfig(p.config); cfg.Host = url` (`endpointSliceUpdate`). The copy keeps the gateway's CA and the Connection's bearer token, which the gateway expects for these URLs too.
5. **Watches across consumer clusters.** `NewWildcardCache` appends `/clusters/*` to `Host` (`pkg/cache/wildcard.go`), so informers list and watch `https://gateway…/connections/c-7f…/vw/nereus.kcp…/services/apiexport/3y9j0def/cowboys/clusters/*/apis/wildwest.dev/v1alpha1/cowboys?watch=true`. The gateway strips `/connections/c-7f…/vw/nereus.kcp…`, picks a Session serving `nereus.kcp…`, and relays `/services/apiexport/3y9j0def/cowboys/clusters/*/apis/…` with `Host: nereus.kcp…`; the Connector routes it by path to its shard's virtual workspace, which accepts it because `Host` is its own.
6. **A client per consumer cluster.** When a consumer's `APIBinding` shows up through that endpoint, the provider engages it with `NewScopedCluster`, which joins `/clusters/<consumer>` onto the endpoint's `Host` (`pkg/cache/cluster.go`). Writes, discovery and Events for that consumer use that URL, through the same gateway path; reads come from the wildcard cache.

**One transport for all of them**: every config here has the same TLS settings (the gateway's CA) and no `Proxy`, so client-go's cache gives them one transport and one connection pool to the gateway; the bearer token is added by a wrapper, not part of the cache key. On its side, the gateway can reuse Streams to the Connector for requests of the same Connection and host.

**What the gateway's path format must get right**:
- **everything after the host passes through unchanged**, including its escaping (`u.EscapedPath()`), since the Connector refuses non-canonical paths;
- **the host segment is `host[:port]`** (letters, digits, dots, hyphens and a colon, all valid in a path segment); the gateway takes the segment after `/vw/` as the target host and refuses hosts no Session of that Connection serves;
- **slices changing** (a shard added or removed) is handled by the provider as today: the mapped URLs are stable per host, so a new shard gets a new endpoint and a removed one is stopped.

**Shipped as a multicluster-runtime provider.** The remaining glue (watch the kubeconfig `Secret`s, build a kcp provider per Connection with the URL mapping, compose them) fits a small package implementing `multicluster.Provider`, multicluster-runtime's own extension point:
```go
p := connections.NewProvider(cfg, connections.Options{EndpointSlice: "cowboys"})
mgr, _ := mcmanager.New(cfg, p, manager.Options{})
```
The rest is standard multicluster-runtime code. The package has no transport, dial hook or protocol code; all of that is in the gateway.

**Rejected within this variant: rewriting endpoint slices in the gateway**, so that even the URL mapping isn't needed. The gateway would change kcp's responses (`status.endpoints[].url` in `get`, `list` and `watch`), which is fragile: kcp's types are CRDs served as JSON today, CBOR is coming to Kubernetes, and every other resource carrying URLs would need the same. The public option is the cleaner seam.

## Variant A with the provider package

Variant A's transport-sharing problem can be solved where variant B's glue lives: in the SDK's multicluster-runtime provider package, without forking kcp-multicluster-provider. The package passes it a config that already carries a shared transport.

**Per Connection, the package:**
1. loads the kubeconfig `Secret` (`proxy-url`, the gateway's CA, the workspace URL);
2. builds the transport once, `rt, _ := rest.TransportFor(cfg)`, with the proxy and TLS inside it;
3. passes `&rest.Config{Host: cfg.Host, Transport: rt}` to `provider.NewProvider`;
4. composes the per-Connection providers with `connectionID` prefixes, as in variant B.

**Why it holds with kcp-multicluster-provider as it is** (`fa033a7`): it only ever changes `Host` on its copies (`endpointSliceUpdate`, `NewScopedCluster`, `NewWildcardCache`, the event recorder), never TLS or proxy fields; `rest.CopyConfig` copies `Transport` by reference; and `client.New`, `rest.HTTPClientFor` and the recorder's client all end in client-go's `transport.New`, which uses a set `Transport` as it is (it refuses only when TLS options are set too, which the package avoids). Every engaged cluster and endpoint of a Connection shares one pool, Streams stay at about the number of requests in flight, and the endpoint slices' URLs stay kcp's own, with no mapping.

**What the package still has to handle**: rotation of the gateway's CA and of the Connection's proxy credentials, both of which live in the transport. When the gateway updates the `Secret`, the package swaps the transport behind a small wrapper, as client-go's `atomicTransportHolder` does for CA files, instead of re-creating the Connection's provider, which would relist.

**What it doesn't change**: operators that don't use the package (another language, a hand-written setup) still get a pool per client unless they build the shared transport themselves. With `tls-server-name` (see "Variant A"), the gateway needs only one certificate for its own name, as in variant B.

## Why variant A needs a forward proxy

Variant A keeps kcp's real URLs in the operator: the workspace placeholder and the virtual-workspace URLs from endpoint slices. Two things then have to happen outside the URL:
1. **Routing**: the operator connects to a host like `nereus.kcp…`, which the Provider's cluster can't and mustn't reach directly; something has to redirect the connection to the gateway without changing the URL.
2. **Which Connection**: the same kcp hostnames serve every Connection of a PM Instance, so the hostname doesn't say which Connection a request belongs to.

A forward proxy is client-go's only standard, kubeconfig-level way to do both: `proxy-url` redirects the connection, and its credential names the Connection. The other ways to redirect without changing URLs are worse: DNS overrides for kcp's hostnames (managing cluster DNS for names that change with PM's topology, across PM Instances, and still a token for the Connection); a custom dialer or `Transport` in the operator (what the SDK does today, and what the gateway was meant to remove); TLS SNI routing (carries the hostname, not the Connection).

Variant B drops the premise: once the operator uses the gateway's URLs, routing and the Connection are both in the URL, and nothing needs redirecting. The price is mapping the endpoint slices' URLs.

The same watch on shard `nereus`, for Connection `c-7f…`, in both variants:

```
 VARIANT A: forward proxy (kcp's own URLs; the connection is redirected)

  operator (client-go)
   rest.Config: Host  = https://nereus.kcp.example.com/services/apiexport/…   ← kcp's URL, unchanged
                Proxy = https://c-7f…:<cred>@gateway.svc:3128                   ← redirect + which Connection
        │
        │ 1. TCP + TLS to gateway.svc:3128 (the proxy)
        │ 2. CONNECT nereus.kcp.example.com:443
        │    Proxy-Authorization: c-7f… ──────────────▶ which Connection
        │ 3. TLS inside the tunnel, verified as "gateway.svc" (tls-server-name; the gateway's one cert)
        │ 4. GET /services/apiexport/…/cowboys?watch=true
        ▼
  ┌──────────────────────────────────────┐
  │ gateway                              │  Connection c-7f… + host nereus.kcp… → a Session
  └──────────────────┬───────────────────┘
                     ▼  Stream ══▶ Connector ──▶ kcp

  why the proxy:  the URL names only kcp's host, so the proxy setting is what redirects
                  the connection to the gateway, and its credential says which Connection
  cost:           Proxy set → client-go can't cache transports → a pool per client


 VARIANT B: reverse proxy by path (gateway URLs; nothing to redirect)

  operator (client-go)
   rest.Config: Host = https://gateway.svc/connections/c-7f…/vw/nereus.kcp.example.com/services/apiexport/…
                          ▲                 ▲               ▲
                          │                 │               └ target host, in the path
                          │                 └ which Connection, in the path
                          └ the gateway itself: an ordinary server
                BearerToken = <c-7f…'s gateway token>
        │
        │ 1. TCP + TLS to gateway.svc (its own serving cert)
        │ 2. GET /connections/c-7f…/vw/nereus.kcp…/services/apiexport/…/cowboys?watch=true
        ▼
  ┌──────────────────────────────────────┐
  │ gateway                              │  strip prefix → Connection c-7f… + host nereus.kcp… → a Session
  └──────────────────┬───────────────────┘
                     ▼  Stream ══▶ Connector ──▶ kcp

  where the URL comes from:  the endpoint slice lists https://nereus.kcp…/services/apiexport/…;
                             ExtractURLsFromEndpointSlice maps it to the gateway URL above
  cost:                      that mapping function (or the provider package that holds it)
```

## Comparison

| | Variant A: forward proxy | Variant A with the provider package | Variant B: reverse proxy by path (with the provider package) |
|---|---|---|---|
| URLs the operator uses | kcp's own (placeholder for the workspace) | kcp's own | the gateway's, per Connection |
| client-go transport sharing | lost; a pool per client | one transport per Connection, built by the package | standard; client-go's cache |
| Certificates at the gateway | one serving certificate for its own name with `tls-server-name`; otherwise a CA issuing for any kcp host, per `CONNECT` | the same | one serving certificate for its own name, or a pinned self-signed one |
| Special code | a shared transport per Connection in the operator, plus composition | in the package: the shared transport, swapped on CA or credential rotation | in the package: a URL-mapping function in a public option |
| Operators without the package (other languages, hand-written) | must build the shared transport, or get a pool per client | the same | only need the URL mapping |
| Authentication to the gateway | credential in the proxy URL | the same | bearer token per Connection |
| Gateway's request handling | `CONNECT`, TLS as the requested host, then a byte copy | the same | HTTP reverse proxy: route by path, strip prefix and token, set `Host` |

## What both give, and cost

**Gives:**
- **PM ships the security-critical code as a component**, versioned with the protocol: a protocol upgrade is a chart upgrade, not a rebuild of every Provider's operator, so `ProtocolOutdated` gets easier to resolve.
- **The operator is decoupled**: its own replicas and leader election, no SDK in its binary, and the "same process" constraint goes away.
- **Any language** with a Kubernetes client works for the operator (07 says "Go only" today), the more so with variant B.

**Costs:**
- **One more in-cluster hop**, with the gateway in the path of every request and watch, and TLS terminated twice (Inner TLS at the gateway, then the operator's TLS to it).
- **The gateway's certificate or CA in the Provider's cluster**: in both variants one serving certificate for its own name suffices (in variant A with `tls-server-name`), self-signed and pinned, or from the gateway's CA.
- **Authentication between operator and gateway**, and RBAC on the kubeconfig `Secret`s.
- **The gateway sees all traffic in the clear**, like the SDK today, inside the trust boundary.
- **v5 rejected something like this**: "a gateway container exposing a local proxy and kubeconfig files … the Provider still needs SDK code to consume a changing set of Connections". Kubeconfig `Secret`s per Connection and a multicluster-runtime provider answer most of that; what's left is the per-Connection composition, which the provider package carries.

## What would change in the plan

- **07 SDK** splits into the gateway (a component with its chart and image) and, for variant B, the small provider package.
- **08 Quickstart**: Connection mode becomes kubeconfig mode with many kubeconfigs, plus the provider package.
- **09 Deployment**: a chart that Providers install.
- **v5**: the rejected "gateway container" alternative is revised.
- **01 Protocol**: unchanged.

## To confirm

- multicluster-runtime's composition and kubeconfig providers at v0.23, and how they compose with kcp-multicluster-provider's endpoint-slice provider.
- Variant B: the gateway relaying watches and long requests by path; `ExtractURLsFromEndpointSlice` with gateway URLs end to end.
- Variant A, if kept: Go's `CONNECT` with one `TLSClientConfig` for proxy and target, and `Proxy-Authorization` from the URL; that a set `ServerName` (`tls-server-name`) is used for the handshake inside the tunnel; tunnel counts with and without a shared transport.
- An e2e test with two Connections and two shards through the gateway.

## Recommendation

The analysis favoured variant B with the provider package: kubeconfigs, client-go's standard caching and multicluster-runtime's standard extension point, with no transport code in the operator. With the package, variant A is viable too and keeps kcp's real URLs in the operator; with `tls-server-name`, its gateway needs only one certificate too. What separates them is the shared transport in the package and `CONNECT` handling (variant A) against the URL mapping in the package and HTTP reverse-proxying (variant B, standard caching even without the package), and that operators without the package only need the URL mapping in variant B.

**Chosen: variant A without the package**, for the most familiar experience for operator authors: their code is exactly what they'd write against kcp directly, with only a different kubeconfig. Accepted with it:
- **a connection pool per client**: reconcile bursts across many consumer clusters open many tunnels; each a Stream within the Connector's per-Connection limits;
- **a shared transport per Connection as an optional optimisation**, documented with a few lines in the quickstart's README, for operators with many consumer clusters;
- **a secret credential per Connection** in the proxy URL, since client-go sends credentials to a proxy only from the URL;
- **errors before the tunnel** (unknown Connection, bad credential, no Session) reaching client-go as transport errors, retried with its normal backoff, not as API errors.
