# v5 implementation plan: testing

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). How each component and the whole flow are tested. Several guarantees in v5 are security properties; they get tests of their own, not only happy paths.

## Principles

- **The protocol has one set of test vectors**, used by both the SDK and the PM-side ([01 Protocol, "Deliverables"](./xxx-notes-011-v5-impl-01-protocol.md)).
- **Security guarantees are tested as negative cases**: what must be refused is enumerated and checked, for every verb and both kcp path shapes.
- **E2E runs with a real network boundary**: PM can't reach the Provider except through its base URL, and the Provider can't reach PM at all.
- **Every row of v5's failure table has a test**, at the level that fits: a unit test for a component's own reaction, an E2E scenario when components or the network are involved, a security test for an attack.

## Unit tests

**Every rule in a component's document is unit-tested with that component's work** ([02](./xxx-notes-011-v5-impl-02-apis.md), [03](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md), [04](./xxx-notes-011-v5-impl-04-connector-ca.md), [05](./xxx-notes-011-v5-impl-05-connector.md), [07](./xxx-notes-011-v5-impl-07-provider-sdk.md)), including what must be refused. Shared across components:
- **the protocol's test vectors** ([01 Protocol, "Deliverables"](./xxx-notes-011-v5-impl-01-protocol.md)), run by the SDK, the connector and platform-mesh-operator;
- **the SDK's fake PM side and test Provider-side server** ([07 SDK, "Testing support"](./xxx-notes-011-v5-impl-07-provider-sdk.md)), so platform-mesh-operator and the connector test against the real protocol, and Providers test against a real PM side;
- **envtest** for every CRD and its controllers: the SDK's resources, `ProviderEndpointRegistration`, `ConnectorCABundle`;
- **configurable timings** (see "Decided"), so time-based rules are tested without waiting.

## E2E (kind, multi-shard)

**Environment**, extending platform-mesh-operator's kind e2e setup (`operators/platform-mesh-operator/test/e2e/kind`; see also [09 Deployment, "Local development"](./xxx-notes-011-v5-impl-09-deployment.md)): kcp with at least two shards and a front-proxy, cert-manager, platform-mesh-operator, a connector per shard, and the [quickstart](./xxx-notes-011-v5-impl-08-provider-quickstart.md) Provider in a separate network reachable only through an ingress at its base URL. A test ingress that terminates TLS stands in for the Provider's edge.

### Onboarding

| Scenario | Checks |
|---|---|
| Onboarding | The request reaches `Registered`; the `Provider` goes `Provisioning → Ready`, with the `ProviderEndpoint` written only after the first control call; front-proxy session up; the Provider bootstraps its export through the connection; the token Secret's data is gone |
| Crash before persisting | platform-mesh-operator is killed after `POST`, before persisting its result: the retry returns the same `connectionID`, the `Provider` reaches `Ready`, and the Provider-side has no second connection |
| Control path blocked | The test edge blocks `/control`: the request is `Registered`, the `Provider` shows `Registered=False` with the edge-diagnosis label, no `ProviderEndpoint` is written and no session opens; unblocking lets it continue |
| kcp unreachable | kcp unreachable from the connectors: the `Provider` stays `Provisioning` with `Connected=False` and `KcpUnreachable`, without a timeout; it becomes `Ready` once fixed |

### Shards and replicas

| Scenario | Checks |
|---|---|
| Consumers on a second shard | A binding on shard B makes shard B's connector open a session; the Provider reconciles consumers on both shards; removing the last binding on shard B closes that session |
| Connector replica killed | Every replica of the shard holds a session per connection: killing one cuts only the streams on its own sessions, which client-go reopens on the others' sessions at once |
| Shard connector or virtual workspace down | Only that shard's endpoint traffic stops; with its virtual workspace down, connectors keep the last known `ProviderEndpoint`s and slices |
| Host rename | A shard's virtual-workspace host changes: the connector lists old and new hosts in its hellos, endpoint slices switch, then the old host is dropped; no session gap in either order of changes, no certificate reissuance |
| Provider standby replicas | Two quickstart replicas behind the test ingress: upgrades reaching the standby get `503` with `Retry-After: 5`, connectors land on the leader within a few attempts, never retrying sooner; killing the leader moves all sessions to the new leader |

### PM-side restarts

| Scenario | Checks |
|---|---|
| PM-side restarts | A connector replica, platform-mesh-operator, and platform-mesh-operator's leader in turn: each recovers without registration calls; a standby takes the Lease within seconds and reconciles every connection once, within the per-Provider rate limits; traffic continues while platform-mesh-operator is down, and onboarding waits |
| Connection breaks after `Ready` | The test edge cuts all sessions of a `Ready` remote Provider: `Connected=False` with `NoSessions`, `phase` `Degraded`; back to `Ready` once sessions return |

### Edge

| Scenario | Checks |
|---|---|
| Edge | The quickstart's traefik `HTTPRoute` passes sessions and control sessions, and 32 KB binary messages both ways. The test ingress in turn returns an HTML challenge page on upgrade, drops binary messages, enforces a short idle timeout, and cuts connections at a short age: each yields its label on the `Provider`. With a private CA set as the `Provider`'s `caBundle`, registration, control sessions and sessions work; a rotation with the new CA added first keeps sessions working; without the update, new sessions fail with `EdgeTLS`, and updating `caBundle` recovers them |

### Connection secret

| Scenario | Checks |
|---|---|
| Secret change | A new value of the `providers.platform-mesh.io/rotate-connection-secret` annotation: the hash goes through the `ProviderEndpoint`, the Provider-side adopts it, the old secret is refused afterwards; sessions unaffected |
| Secret change without sessions | All sessions blocked at the test edge: the change waits, the old secret keeps working, and the change completes once sessions return |
| Secret repair | In turn: the `ProviderEndpointRegistration`'s Secret deleted; the `ProviderEndpointRegistration` deleted (its Secret goes with it, and it's rebuilt from the `ProviderEndpoint`); the quickstart's `Connection` restored from an older snapshot, so it rejects PM's secret. Each time the secret is repaired with the same `connectionID`, without owner action, and repeated failures stay within the rate limit |

### Connector CA rollover

| Scenario | Checks |
|---|---|
| Automatic CA rollover | With shortened lifetimes, cert-manager renews the CA (or `cmctl renew` forces it), and connector certificates renew along the way: platform-mesh-operator pushes the trusted set, the new CA converges, connectors switch, and the old CA is dropped connection by connection within the spread; sessions still on the old CA's certificate hand over with go-away and reopen at once; no connection is left without a session |
| Control path broken during a rollover | The test edge blocks a Provider's `/control` path while its sessions stay up: convergence doesn't wait for it, it's listed as not reached, its sessions are lost after the connectors switch, and once `/control` is unblocked the catch-up push restores them within seconds |
| Connector restarted mid-rollover | A connector restarted before the new CA converges uses its newest certificate at once; sessions to connections not yet pinning the new CA fail until the push lands, then recover |
| Provider down during a rollover | The rollover completes for reachable connections; when the Provider returns, its first successful control call triggers the bundle push and sessions open within seconds. Also after two rollovers and with every pinned CA expired: the push is accepted and no tampering is reported |
| cert-manager down past the renewal budget | Connector certificates expire; the expiry metric reaches zero and readiness reports it; open sessions keep serving traffic, no replacements are attempted; a session cut by the test edge's maximum connection age isn't reopened until renewal; after cert-manager recovers, renewal and replacements resume |

### Recovery and offboarding

| Scenario | Checks |
|---|---|
| Connection unknown, onboarded again | The connection is deleted in the quickstart's portal: sessions and calls are refused, `Registered=False` with `ConnectionUnknown`; a new request for the same `Provider` registers a new connection, the `ProviderEndpoint` switches, consumers' bindings keep working, and the quickstart adopts existing `Cowboy` resources through the IDs in their status |
| Switching Providers | A second quickstart instance at another base URL: after the first deletes the connection, a request with the second's token registers there; bindings keep working and the second adopts the existing APIExport |
| Server key rotation | The new `ServerKey` is listed in the `ProviderEndpoint` before any connection presents it; sessions keep working. With platform-mesh-operator stopped during the rotation, the Provider-side keeps the old key on that connection until the `ProviderEndpoint` lists the new one |
| Provider-side state lost | The quickstart's `Connection` is removed without its finalizer running and imported again with `kubectl apply`, its new server key given to PM's administrators, who edit the `ProviderEndpointRegistration`: sessions resume under the same `connectionID`, the secret is repaired through the `ProviderEndpoint`, and the quickstart adopts the existing `Cowboy` resources without creating new ones |
| Out-of-date bundle and a lost secret | A connection whose bundle missed the rollover and whose Provider-side rejects PM's secret stays without sessions; re-pinning by editing the `Connection`'s `connectorCABundle` brings sessions back, then the secret is repaired |
| Offboarding | `ProviderEndpoint` deleted, sessions closed, `DELETE` received, `ProviderEndpointRegistration` and its Secret deleted, workspace removed |

## Security tests

- **Provider attempts through a session**: edit or delete its `ProviderEndpoint` (refused by kcp, through the connection export's maximal permission policy), delete the binding, write endpoint-slice status, create a TokenRequest, create an SA-token Secret, make the connector act as any SA other than `default/provider`, create a CSR, `exec`, impersonation headers, requests to another connection's workspace, streams to hosts outside the allow-list. All refused and audited.
- **Owner attempts**: edit the `ProviderEndpoint` (no rights), point `Provider` status or spec at another workspace (ignored by platform-mesh-operator), read a connection secret (none in the owner's workspace).
- **Stolen connection secret**: used against the Provider-side to push a bundle with a foreign CA. platform-mesh-operator detects it, restores the bundle, changes the secret through the `ProviderEndpoint` and records an event on the `Provider`; the stolen secret is refused afterwards. No call lets the thief change the secret.
- **Stolen onboarding token or portal account**: registers only a new, separate connection that receives no sessions from PM; it can't change an existing connection's secret or bundle.
- **Malicious edge**: the test ingress tries to read session contents, open its own streams, replay a recorded handshake, and swap the server key. It sees only opaque bytes and every attempt fails.
- **Forged sessions**: a client certificate from an unpinned CA, an expired connector certificate, a chain with an intermediate, a hello with another connection's `clusterName`; a flood of upgrades with a known connection ID stays within the handshake limits.
- **Control-path guessing**: requests with wrong secrets end the control session at the first one; each new attempt costs a handshake and counts against the limits.
- **Leaked CA key**: a test client signs a connector certificate with the CA's key and opens a session; after `cmctl renew` and listing the CA in `distrust`, the Provider-side closes that session within the grace period and refuses new ones, while PM's connectors keep working.
- **Tampered bundle with sessions**: someone holding the connection secret pins their own CA and opens a session with it; platform-mesh-operator's tamper response restores the bundle, and the Provider-side closes the thief's session.
- **Egress**: base URLs pointing at private, link-local or metadata addresses are refused by platform-mesh-operator and connectors.

## Load

Load tests of the connector are a [follow-up](./xxx-notes-011-v5-impl-todo.md).

## Work breakdown

1. Protocol test vectors.
2. Unit tests per component (with each component's work).
3. Multi-shard kind environment with the network boundary and a TLS-terminating test ingress.
4. E2E scenarios and security tests.

Follow-ups that need tests of their own (profile validation, load) are in the [follow-ups doc](./xxx-notes-011-v5-impl-todo.md).

## Decided

- **The kind environment extends platform-mesh-operator's** `test/e2e/kind`.
- **Shortened timings**: every component exposes its waiting periods, lifetimes and intervals as configuration (poll interval, repair rate limit, connector CA and connector certificate lifetimes, the old CA's drop delay and its spread, the registration retry window and with it orphan expiry, the session grace period, Lease durations, the delay before a `Connection`'s `Ready` reports a failure, the `lastSeen` interval), so CI can run full cycles in minutes.
- **No separate kcp integration tests**: what they'd check (the maximal permission policy, the permission claim, token minting) is covered by the E2E scenarios and the security tests; kcp's own features (CEL, the policy mechanism) are tested by kcp.
