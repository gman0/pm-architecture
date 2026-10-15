# v5 implementation plan: testing

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). How each component and the whole flow are tested. Several guarantees in v5 are security properties; they get tests of their own, not only happy paths.

## Principles

- **The protocol has one set of test vectors**, used by both the SDK and the PM-side ([01 Protocol, "Deliverables"](./xxx-notes-011-v5-impl-01-protocol.md)).
- **Security guarantees are tested as negative cases**: what must be refused is enumerated and checked, for every verb and both kcp path shapes.
- **E2E runs with a real network boundary**: PM can't reach the Provider except through its Base URL, and the Provider can't reach PM at all.
- **Every row of v5's failure table has a test**, at the level that fits: a unit test for a component's own reaction, an E2E scenario when components or the network are involved, a security test for an attack.

## Unit tests

**Every rule in a component's document is unit-tested with that component's work** ([02](./xxx-notes-011-v5-impl-02-apis.md), [03](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md), [04](./xxx-notes-011-v5-impl-04-connector-ca.md), [05](./xxx-notes-011-v5-impl-05-connector.md), [07](./xxx-notes-011-v5-impl-07-provider-sdk.md)), including what must be refused. Shared across components:
- **the protocol's test vectors** ([01 Protocol, "Deliverables"](./xxx-notes-011-v5-impl-01-protocol.md)), run by the SDK, the Connector and `platform-mesh-operator`;
- **the SDK's fake PM-side and test Provider-side server** ([07 SDK, "Testing support"](./xxx-notes-011-v5-impl-07-provider-sdk.md)), so `platform-mesh-operator` and the Connector test against the real protocol, and Providers test against a real PM-side;
- **envtest** for every CRD and its controllers: the SDK's resources, `ProviderEndpointRegistration`, `ConnectorCABundle`;
- **configurable timings** (see "Decided"), so time-based rules are tested without waiting.

## E2E (kind, multi-shard)

**Environment**, extending `platform-mesh-operator`'s kind e2e setup (`operators/platform-mesh-operator/test/e2e/kind`; see also [09 Deployment, "Local development"](./xxx-notes-011-v5-impl-09-deployment.md)): kcp with at least two shards and a front-proxy, cert-manager, `platform-mesh-operator`, a Connector per shard, and the [quickstart](./xxx-notes-011-v5-impl-08-provider-quickstart.md) Provider in a separate network reachable only through an ingress at its Base URL. A test ingress that terminates TLS stands in for the Provider's Edge.

### Onboarding

| Scenario | Checks |
|---|---|
| Onboarding | The request reaches `Registered`; the `Provider` goes `Provisioning → Ready`, with the `ProviderEndpoint` written only after the first Control Call; Front-Proxy Session up; the Provider bootstraps its export through the Connection; the token `Secret`'s data is gone |
| Crash before persisting | `platform-mesh-operator` is killed after `POST`, before persisting its result: the retry returns the same `connectionID`, the `Provider` reaches `Ready`, and the Provider-side has no second Connection |
| Control path blocked | The test Edge blocks `/control`: the request is `Registered`, the `Provider` shows `Registered=False` with the edge-diagnosis label, no `ProviderEndpoint` is written and no Session opens; unblocking lets it continue |
| kcp unreachable | kcp unreachable from the Connectors: the `Provider` stays `Provisioning` with `Connected=False` and `KcpUnreachable`, without a timeout; it becomes `Ready` once fixed |

### Shards and replicas

| Scenario | Checks |
|---|---|
| Consumers on a second shard | A binding on shard B makes shard B's Connector open a Session; the Provider reconciles consumers on both shards; a watch through the Connector stays open for over an hour and ends only when kcp ends it; removing the last binding on shard B closes that Session |
| Connector Replica killed | Every replica of the shard holds a Session per Connection: killing one cuts only the Streams on its own Sessions, which client-go reopens on the others' Sessions at once |
| A shard's Connector or virtual workspace down | Only that shard's endpoint traffic stops; with its virtual workspace down, Connectors keep the last known `ProviderEndpoint`s and slices |
| Host rename | A shard's virtual-workspace host changes: the Connector lists old and new hosts in its Hellos, endpoint slices switch, then the old host is dropped; no Session gap in either order of changes, no certificate reissuance |
| Provider standby replicas | Two quickstart replicas behind the test ingress: upgrades reaching the standby get `503` with `Retry-After: 5`, Connectors land on the leader within a few attempts, never retrying sooner; killing the leader moves all Sessions to the new leader |

### PM-side restarts

| Scenario | Checks |
|---|---|
| PM-side restarts | A Connector Replica, `platform-mesh-operator`, and `platform-mesh-operator`'s leader in turn: each recovers without Registration calls; a standby takes the Lease within seconds and reconciles every Connection once, within the per-Provider rate limits; traffic continues while `platform-mesh-operator` is down, and Onboarding waits |
| Connection breaks after `Ready` | The test Edge cuts all Sessions of a `Ready` Remote Provider: `Connected=False` with `NoSessions`, `phase` `Degraded`; back to `Ready` once Sessions return |

### Edge

| Scenario | Checks |
|---|---|
| Edge | The quickstart's traefik `HTTPRoute` passes Sessions and Control Sessions, and 32 KB binary messages both ways. The test ingress in turn returns an HTML challenge page on upgrade, drops binary messages, enforces a short idle timeout, and cuts connections at a short age: each yields its label on the `Provider`. With a private CA set as the `Provider`'s `caBundle`, Registration, Control Sessions and Sessions work; a rotation with the new CA added first keeps Sessions working; without the update, new Sessions fail with `EdgeTLS`, and updating `caBundle` recovers them |

### Connection Secret

| Scenario | Checks |
|---|---|
| Secret Change | A new value of the `providers.platform-mesh.io/rotate-connection-secret` annotation: the hash goes through the `ProviderEndpoint`, the Provider-side adopts it, the old Connection Secret is refused afterwards; Sessions unaffected |
| Secret Change without Sessions | All Sessions blocked at the test Edge: the change waits, the old Connection Secret keeps working, and the change completes once Sessions return |
| Connection Secret repair | In turn: the `ProviderEndpointRegistration`'s `Secret` deleted; the `ProviderEndpointRegistration` deleted (its `Secret` goes with it, and it's rebuilt from the `ProviderEndpoint`); the quickstart's `Connection` restored from an older snapshot, so it rejects PM's Connection Secret. Each time the Connection Secret is repaired with the same `connectionID`, without Owner action, and repeated failures stay within the rate limit |

### Rollover

| Scenario | Checks |
|---|---|
| Automatic Rollover | With shortened lifetimes, cert-manager renews the CA (or `cmctl renew` forces it), and Connector Certificates renew along the way: `platform-mesh-operator` pushes the Trusted Set, the new CA converges, Connectors switch, and the old CA is dropped Connection by Connection within the spread; Sessions still on the old CA's certificate hand over with go-away and reopen at once; no Connection is left without a Session |
| Control path broken during a Rollover | The test Edge blocks a Provider's `/control` path while its Sessions stay up: convergence doesn't wait for it, it's listed as not reached, its Sessions are lost after the Connectors switch, and once `/control` is unblocked the catch-up push restores them within seconds |
| Connector restarted mid-Rollover | A Connector restarted before the new CA converges uses its newest certificate at once; Sessions to Connections not yet pinning the new CA fail until the push lands, then recover |
| Provider down during a Rollover | The Rollover completes for reachable Connections; when the Provider returns, its first successful Control Call triggers the bundle push and Sessions open within seconds. Also after two Rollovers and with every pinned CA expired: the push is accepted and no tampering is reported |
| cert-manager down past the renewal budget | Connector Certificates expire; the expiry metric reaches zero and readiness reports it; open Sessions keep serving traffic, no replacements are attempted; a Session cut by the test Edge's maximum connection age isn't reopened until renewal; after cert-manager recovers, renewal and replacements resume |

### Recovery and offboarding

| Scenario | Checks |
|---|---|
| Connection unknown, onboarded again | The Connection is deleted in the quickstart's portal: Sessions and calls are refused, `Registered=False` with `ConnectionUnknown`; a new request for the same `Provider` registers a new Connection, the `ProviderEndpoint` switches, consumers' bindings keep working, and the quickstart adopts existing `Cowboy` resources through the IDs in their status |
| Switching Providers | A second quickstart instance at another Base URL: after the first deletes the Connection, a request with the second's token registers there; bindings keep working and the second adopts the existing APIExport |
| Server Key rotation | The new `ServerKey` is listed in the `ProviderEndpoint` before any Connection presents it; Sessions keep working. With `platform-mesh-operator` stopped during the rotation, the Provider-side keeps the old key on that Connection until the `ProviderEndpoint` lists the new one |
| Provider-side state lost | The quickstart's `Connection` is removed without its finalizer running and imported again with `kubectl apply`, its new Server Key given to PM Administrators, who edit the `ProviderEndpointRegistration`: Sessions resume under the same `connectionID`, the Connection Secret is repaired through the `ProviderEndpoint`, and the quickstart adopts the existing `Cowboy` resources without creating new ones |
| Out-of-date bundle and a lost Connection Secret | A Connection whose bundle missed the Rollover and whose Provider-side rejects PM's Connection Secret stays without Sessions; re-pinning by editing the `Connection`'s `connectorCABundle` brings Sessions back, then the Connection Secret is repaired |
| Offboarding | `ProviderEndpoint` deleted, Sessions closed, `DELETE` received, `ProviderEndpointRegistration` and its `Secret` deleted, workspace removed |

## Security tests

- **Provider attempts through a Session**: edit or delete its `ProviderEndpoint` (refused by kcp, through the Connection Export's maximal permission policy), delete the binding, write endpoint-slice status, create a TokenRequest, create an SA-token `Secret`, make the Connector act as any SA other than `default/provider`, create a CSR, `exec`, impersonation headers, requests to another Connection's workspace, Streams to hosts outside the allow-list, paths in neither kcp shape. All refused and logged.
- **Owner attempts**: edit the `ProviderEndpoint` (no rights), point `Provider` status or spec at another workspace (ignored by `platform-mesh-operator`), read a Connection Secret (none in the Owner's workspace).
- **Stolen Connection Secret**: used against the Provider-side to push a bundle with a foreign CA. `platform-mesh-operator` detects it, restores the bundle, changes the Connection Secret through the `ProviderEndpoint` and records an event on the `Provider`; the stolen Connection Secret is refused afterwards. No call lets the thief change the Connection Secret.
- **Stolen Onboarding Token or portal account**: registers only a new, separate Connection that receives no Sessions from PM; it can't change an existing Connection Secret or Pinned Bundle.
- **Malicious Edge**: the test ingress tries to read Session contents, open its own Streams, replay a recorded handshake, and swap the Server Key. It sees only opaque bytes and every attempt fails.
- **Forged Sessions**: a client certificate from an unpinned CA, an expired Connector Certificate, a chain with an intermediate, a Hello with another Connection's `clusterName`; a flood of upgrades with a known Connection ID stays within the handshake limits.
- **Control-path guessing**: requests with wrong Connection Secrets end the Control Session at the first one; each new attempt costs a handshake and counts against the limits.
- **Leaked CA key**: a test client signs a Connector Certificate with the CA's key and opens a Session; after `cmctl renew` and listing the CA in `distrust`, the Provider-side closes that Session within the grace period and refuses new ones, while PM's Connectors keep working.
- **Tampered bundle with Sessions**: someone holding the Connection Secret pins their own CA and opens a Session with it; `platform-mesh-operator`'s tamper response restores the bundle, and the Provider-side closes the thief's Session.
- **Egress**: Base URLs pointing at private, link-local or metadata addresses are refused by `platform-mesh-operator` and Connectors.

## Load

Load tests of the Connector are a [follow-up](./xxx-notes-011-v5-impl-todo.md).

## Work breakdown

1. Protocol test vectors.
2. Unit tests per component (with each component's work).
3. Multi-shard kind environment with the network boundary and a TLS-terminating test ingress.
4. E2E scenarios and security tests.

Follow-ups that need tests of their own (profile validation, load) are in the [follow-ups doc](./xxx-notes-011-v5-impl-todo.md).

## Decided

- **The kind environment extends `platform-mesh-operator`'s** `test/e2e/kind`.
- **Shortened timings**: every component exposes its waiting periods, lifetimes and intervals as configuration (poll interval, repair rate limit, Connector CA and Connector Certificate lifetimes, the old CA's drop delay and its spread, the Registration retry window and with it orphan expiry, the Session grace period, Lease durations, the delay before a `Connection`'s `Ready` reports a failure, the `lastSeen` interval), so CI can run full cycles in minutes.
- **No separate kcp integration tests**: what they'd check (the maximal permission policy, the permission claim, token minting) is covered by the E2E scenarios and the security tests; kcp's own features (CEL, the policy mechanism) are tested by kcp.
