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
- **envtest** for every CRD and its controllers: the SDK's resources, `ProviderEndpointRegistration`, `ConnectorCASet`;
- **configurable timings** (see "Decided"), so time-based rules are tested without waiting.

How kcp-multicluster-provider engages clusters, including when a logical cluster moves between shards, is tested there, not in PM; PM's E2E scenarios below cover what PM relies on.

## E2E (kind, multi-shard)

**Environment**, extending `platform-mesh-operator`'s kind e2e setup (`operators/platform-mesh-operator/test/e2e/kind`; see also [09 Deployment, "Local development"](./xxx-notes-011-v5-impl-09-deployment.md)): kcp with at least two shards and a front-proxy, cert-manager, `platform-mesh-operator`, a Connector per shard, and the [quickstart](./xxx-notes-011-v5-impl-08-provider-quickstart.md) Provider in a separate network reachable only through an ingress at its Base URL. A test ingress that terminates TLS stands in for the Provider's Edge. The Provider's address is private (the docker network), which needs no configuration: only loopback, link-local, unspecified and multicast addresses are refused.

### Onboarding

| Scenario | Checks |
|---|---|
| Onboarding | The request reaches `Registered` through a Registration inside Inner TLS pinned to the token's key; the policies in the Provider Workspace are confirmed by the dry-run check before any access is granted; the `Provider` goes `Provisioning → Ready`, with the `ProviderEndpoint` written only after the first Control Call, which pushes the current Connector CA Set; the first Session comes only from the Connector of the shard hosting the Provider Workspace; the Provider bootstraps its export through the Connection; the token `Secret`'s data is gone |
| Remote Providers not installed | A request created before the `providerconnections` HelmRelease is Ready stays `Pending` with `RemoteProvidersNotReady`, and proceeds once it is |
| Request before its `Provider` | The request shows `ProviderNotFound` until the Owner creates the `Provider`, then proceeds |
| Token refused | An expired token, a spent one, and one whose Server Key got `retireAt` after it was issued: each fails the request with `TokenRefused`, before any upgrade |
| Crash before persisting | `platform-mesh-operator` is killed after Registration, before persisting its result: the retry returns the same `connectionID`, Server Keys and `unusedExpiresAt`, also when a Rollover changed the Connector CA Set in between; the `Provider` reaches `Ready`, and the Provider-side has no second Connection |
| First Control Call rejected | The quickstart's `Connection` loses its `secretHash` between Registration and the first Control Call: the `ProviderEndpoint` is written with `pendingSecretHash` anyway, the Connection Secret is repaired without waiting for the rate limit, and `Registered` becomes true |
| Control path blocked | The test Edge blocks `/control`: the request is `Registered`, the `Provider` shows `Registered=False` with the edge-diagnosis label and the `unusedExpiresAt` deadline, no `ProviderEndpoint` is written and no Session opens; unblocking before the deadline lets it continue. Blocked past the (shortened) deadline: the quickstart removes the `Connection` with `Removed`, the next Control Call gets "unknown", `Registered=False` with `ConnectionUnknown`, and a new request onboards the `Provider` again |
| kcp unreachable | kcp unreachable from the Connectors: the `Provider` stays `Provisioning` with `Connected=False` and `KcpUnreachable`, without a timeout; it becomes `Ready` once fixed |

### Shards and replicas

| Scenario | Checks |
|---|---|
| Connector grants | A Connector with only the documented grants ([02 APIs, "Connection Export"](./xxx-notes-011-v5-impl-02-apis.md)) starts, discovers Connections, and reaches a non-root shard's virtual workspace directly with its `Kubeconfig` |
| Consumers on a second shard | A binding on shard B makes shard B's Connector open a Session; the Provider reconciles consumers on both shards; a watch through the Connector stays open for over an hour and ends only when kcp ends it; removing the last binding on shard B closes that Session |
| Workspace moved, no consumers | A PM Administrator moves a Provider Workspace without consumers from shard A to shard B: B's Connector opens the Session and A's closes it; `Connected` recovers; the next `ProviderEndpoint` write succeeds |
| Workspace moved, consumers on the old shard | The same, with consumers on shard A: A's Connector keeps its Sessions, now for its consumers only, and B's Connector opens the Connection's first Session |
| Connector Replica killed | Every replica of the shard holds a Session per Connection: killing one cuts only the Streams on its own Sessions, which client-go reopens on the others' Sessions at once |
| A shard's Connector or virtual workspace down | Only that shard's endpoint traffic stops; with its virtual workspace down, Connectors keep the last known `ProviderEndpoint`s and slices |
| Host rename | A shard's virtual-workspace host changes: the Connector lists old and new hosts in its Hellos, endpoint slices switch, then the old host is dropped; no Session gap in either order of changes, no certificate reissuance. The front-proxy's host changes in the Connectors' configuration only: Providers notice nothing |
| Provider standby replicas | Two quickstart replicas behind the test ingress: upgrades reaching the standby get `503` with `Retry-After: 5`, Connectors land on the leader within a few attempts, never retrying sooner; killing the leader moves all Sessions to the new leader |

### PM-side restarts

| Scenario | Checks |
|---|---|
| PM-side restarts | A Connector Replica, `platform-mesh-operator`, and `platform-mesh-operator`'s leader in turn: each recovers without Registration calls; a standby takes the Lease within seconds and reconciles every Connection once, within the per-Provider rate limits; traffic continues while `platform-mesh-operator` is down, and Onboarding waits |
| Connection breaks after `Ready` | The test Edge cuts all Sessions of a `Ready` Remote Provider: `Connected=False` with `NoSessions`, `phase` `Degraded`; back to `Ready` once Sessions return |

### Edge

| Scenario | Checks |
|---|---|
| Edge | The quickstart's traefik `HTTPRoute` passes Registration, Sessions and Control Sessions, and 32 KB binary messages both ways. The test ingress in turn returns an HTML challenge page on upgrade, drops binary messages, enforces a short idle timeout, and cuts connections at a short age: each yields its label on the `Provider` |
| Path rewritten | The test ingress rewrites `/connections/A/control` to `/connections/B/control` of another Connection on the same Provider: `EdgePathRewritten` on A's `Provider`, no repair and no Secret Change for A or B; rewriting `/session` the same way closes the Session at the Hello |
| Private CA | With a private CA set as the `Provider`'s `caBundle`, Registration, Control Sessions and Sessions work; a rotation with the new CA added first, as a second certificate in the bundle, keeps Sessions working; without the update, new Sessions fail with `EdgeTLS`, and updating `caBundle` recovers them |
| New Base URL | The quickstart is served under a second hostname, and a PM Administrator changes `url` in the `ProviderEndpointRegistration`: Connectors replace their Sessions, the `connectionID` stays, and nothing changes for the Provider's controllers |

### Connection Secret

| Scenario | Checks |
|---|---|
| Secret Change | A new value of the `providers.platform-mesh.io/rotate-connection-secret` annotation: the hash goes through the `ProviderEndpoint`, the Provider-side adopts it, the old Connection Secret is refused afterwards; Sessions unaffected |
| Rapid annotations | Several new values in quick succession: one change at a time, later values coalesced, on-demand changes within the rate limit, a deferral noted in `Connected`'s message |
| Secret Change without Sessions | All Sessions blocked at the test Edge: the change waits, the old Connection Secret keeps working, and the change completes once Sessions return |
| Connection Secret repair | In turn: the `ProviderEndpointRegistration`'s `Secret` deleted; the `ProviderEndpointRegistration` deleted (its `Secret` goes with it, and it's rebuilt from the `ProviderEndpoint`); the quickstart's `Connection` restored from an older snapshot, so it rejects PM's Connection Secret. Each time the Connection Secret is repaired with the same `connectionID`, without Owner action, and repeated failures stay within the rate limit |
| Re-onboarding during a repair | A new request for a `Provider` whose Connection Secret is being repaired: both run in order under the `Provider`'s key, and neither overwrites the other's `ProviderEndpointRegistration` changes |

### Rollover

| Scenario | Checks |
|---|---|
| Automatic Rollover | With shortened lifetimes, cert-manager renews the CA (or `cmctl renew` forces it), and Connector Certificates renew along the way: `platform-mesh-operator` pushes the Connector CA Set, the new CA converges, Connectors switch, and the old CA is dropped Connection by Connection within the spread; Sessions still on the old CA's certificate hand over with go-away and reopen at once; no Connection is left without a Session |
| Two renewals within the drop delay | `cmctl renew` twice in quick succession: the first previous CA stays in the set until its own drop time and until no Connector's certificate chains to it; no Session is lost before the Connectors switch |
| Pending or failed CA request | A CA `CertificateRequest` that waits for approval, then fails: the current CA doesn't change, nothing is pushed for it |
| Restart mid-Rollover | `platform-mesh-operator` restarted right after a new CA appears: `Converged` stays false until every Connection has been polled once, and Connectors don't switch before |
| Control path broken during a Rollover | The test Edge blocks a Provider's `/control` path while its Sessions stay up: convergence doesn't wait for it, it's counted as not reached, its Sessions are lost after the Connectors switch, and once `/control` is unblocked the catch-up push restores them within seconds |
| Connector restarted mid-Rollover | A Connector restarted before the new CA converges uses its newest certificate at once; Sessions to Connections not yet pinning the new CA fail until the push lands, then recover |
| Provider down during a Rollover | The Rollover completes for reachable Connections; when the Provider returns, its first successful Control Call triggers the bundle push and Sessions open within seconds. Also after two Rollovers and with every pinned CA expired: the push is accepted and no tampering is reported |
| cert-manager down past the renewal budget | Connector Certificates expire; the expiry metric reaches zero and readiness reports it; open Sessions keep serving traffic, no replacements are attempted; a Session cut by the test Edge's maximum connection age isn't reopened until renewal; after cert-manager recovers, renewal and replacements resume |

### Compromise

| Scenario | Checks |
|---|---|
| Distrust with the gate | `cmctl renew` of the CA, then of every Connector Certificate by label, then the old CA's `key-sha256` in `spec.distrust`: `DistrustPending` with `waitingFor` until every Connector has switched and the grace period has passed, then `Distrusted`; PM's Connectors keep working throughout |
| Distrust, `immediate` | The same without renewing the Connector Certificates first: the CA is dropped at once; Sessions under it close; Connectors still holding a certificate from it reconnect once theirs is renewed, with their backoff reset |
| Distrust guards | A distrust of the only CA stays `DistrustPending` with `Valid=False`; a fingerprint that matches nothing gives `Unmatched` and `DistrustUnmatched`; a `cert-sha256:` or openssl-style value is refused by the schema; removing a pending entry cancels it |
| Distrusted CA at an unreachable Connection | A Connection unreachable during the distrust shows `TrustOutdated`; when it's back, the corrected set is pushed first and the Connection Secret changed after, reported as `DistrustedCAPinned`, without the tamper alert |

### Recovery and offboarding

| Scenario | Checks |
|---|---|
| Connection unknown, onboarded again | The Connection is deleted in the quickstart's portal: Sessions and calls are refused, `Registered=False` with `ConnectionUnknown`; a new request for the same `Provider` registers a new Connection, the `ProviderEndpoint` switches, consumers' bindings keep working, and the quickstart adopts existing `Cowboy` resources through the IDs in their status |
| Switching Providers | A second quickstart instance at another Base URL: after the first deletes the Connection, a request with the second's token registers there; bindings keep working and the second adopts the existing APIExport; an Event on the `Provider` names the old and new Base URL host and `connectionID`, and `status.connection` follows |
| Abandoning a Connection | The quickstart is stopped for good: the waiting message carries the `abandon-connection` line; with a wrong `connectionID` the annotation does nothing; with the right one, the `ProviderEndpoint` and `ProviderEndpointRegistration` go, the workspace and bindings stay, `Registered=False` with `ConnectionAbandoned`, and a new request onboards a second quickstart |
| Server Key rotation | The new `ServerKey` is listed in the `ProviderEndpoint` before any Connection presents it; Sessions keep working. With `platform-mesh-operator` stopped during the rotation, the Provider-side keeps the old key on that Connection until the `ProviderEndpoint` lists the new one |
| Provider-side state lost | The quickstart's `Connection` is removed without its finalizer running and imported again with `kubectl apply`, its new Server Key given to PM Administrators, who edit the `ProviderEndpointRegistration`: Sessions resume under the same `connectionID`, the Connection Secret is repaired through the `ProviderEndpoint`, and the quickstart adopts the existing `Cowboy` resources without creating new ones |
| Out-of-date bundle and a lost Connection Secret | A Connection whose bundle missed the Rollover and whose Provider-side rejects PM's Connection Secret stays without Sessions; re-pinning by editing the `Connection`'s `spec.pinnedBundle` brings Sessions back, then the Connection Secret is repaired |
| Offboarding | `ProviderEndpoint` deleted, Sessions closed, `DELETE` received, `ProviderEndpointRegistration` and its `Secret` deleted, policies, RBAC and workspace removed |

## Security tests

- **Provider attempts through a Session**: edit or delete its `ProviderEndpoint` (refused by kcp, through the Connection Export's maximal permission policy), delete the binding, write endpoint-slice status, create a TokenRequest, make the Connector act as any SA other than `default/provider`, create a CSR, `exec`, impersonation headers, requests to another Connection's workspace, requests in neither kcp shape, a virtual-workspace request with another shard's host. All refused and logged.
- **Provider attempts on RBAC and tokens**, each through `create`, `update`, `patch` and server-side apply: a `RoleBinding` or `ClusterRoleBinding` to a user or group it controls, or to `system:authenticated`; a `Secret` of type `kubernetes.io/service-account-token`; changing or deleting PM's policies, their bindings or the `providers.platform-mesh.io:apiexport-bind` ClusterRole. All refused, by the policies or the Connector's Denylist. Allowed: bindings to the workspace's ServiceAccounts, to `apis.kcp.io:binding:` subjects, and to `providers.platform-mesh.io:apiexport-bind`. `platform-mesh-operator`'s own writes aren't affected (`matchConditions`). The dry-run check reports `PolicyNotEnforced` while a policy isn't in force.
- **Paths**: dot segments, empty segments, encoded `/` or `:`, a workspace path instead of a logical cluster name (refused with its own rule), `/clusters/*` through the front-proxy. All refused before anything reaches kcp, inside the APIExport virtual-workspace shape too.
- **Owner attempts**: edit the `ProviderEndpoint` (no rights), point `Provider` status or spec at another workspace (ignored by `platform-mesh-operator`), read a Connection Secret (none in the Owner's workspace), change a running request's spec or its token's ID (refused).
- **Stolen Connection Secret**: used against the Provider-side to push a bundle with a foreign CA. `platform-mesh-operator` detects it, restores the bundle, changes the Connection Secret through the `ProviderEndpoint` and records an event on the `Provider`; the stolen Connection Secret is refused afterwards. No call changes the Connection Secret. When the holder changes it through a Session under their own CA first, the `Provider` shows `SecretRejected`, and re-pinning the bundle on the Provider-side lets PM repair the Connection Secret.
- **Stolen Onboarding Token or portal account**: registers only a new, separate Connection that receives no Sessions from PM; it can't change an existing Connection Secret or Pinned Bundle. A token used before PM uses it makes PM's Registration fail as spent.
- **Malicious Edge**: the test ingress tries to read Session contents, open its own Streams, replay a recorded handshake, and swap the Server Key; during Registration, to read the token, change the Connector CA Set or the secret hash, or swap the returned Server Keys; and to answer "unknown Connection" itself. It sees only opaque bytes, every attempt fails, and its "unknown" never allows a new Registration.
- **Forged Sessions**: a client certificate from an unpinned CA, an expired Connector Certificate, a chain with an intermediate, a Hello with another Connection's `connectionID` or `clusterName`; a flood of upgrades with a known Connection ID stays within the handshake limits, and one with unknown IDs on `control` within its own limit.
- **Control-path guessing**: requests with wrong Connection Secrets end the Control Session at the first one; each new attempt costs a handshake and counts against the limits. Failed authentications change nothing in the quickstart's `Connection` (no `lastSeen`, no Event).
- **Bundles**: an empty bundle, or one with only expired CAs, is refused at Registration, on a push and on a re-pin.
- **Leaked CA key**: a test client signs a Connector Certificate with the CA's key and opens a Session; after the compromise procedure ([04 Connector CA, "Compromise response"](./xxx-notes-011-v5-impl-04-connector-ca.md)), the Provider-side closes that Session within the grace period and refuses new ones, while PM's Connectors keep working.
- **Tampered bundle with Sessions**: someone holding the Connection Secret pins their own CA and opens a Session with it; `platform-mesh-operator`'s tamper response restores the bundle, and the Provider-side closes the thief's Session.
- **Egress**: Base URLs pointing at loopback, link-local or metadata addresses, directly or through a DNS name that resolves to them, are refused by `platform-mesh-operator` and Connectors; private addresses are allowed. With `HTTPS_PROXY` set on the pods, kcp traffic still goes direct.

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
- **Shortened timings**: every component exposes its waiting periods, lifetimes and intervals as configuration (poll interval, repair and on-demand rate limits, Connector CA and Connector Certificate lifetimes, the old CA's drop delay and its spread, the distrust grace period and spread, the Registration retry window and with it orphan expiry, the Session grace period, Lease durations, the delay before a `Connection`'s `Ready` reports a failure, the `lastSeen` interval), so CI can run full cycles in minutes.
- **No separate kcp integration tests**: what they'd check (the maximal permission policy, the permission claim, token minting, the admission policies) is covered by the E2E scenarios and the security tests; kcp's own features (CEL, the policy mechanism) are tested by kcp.
- **multicluster-provider's behaviour is tested upstream**; PM has E2E scenarios for what it relies on.
