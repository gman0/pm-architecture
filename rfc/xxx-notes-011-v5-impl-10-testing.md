# v5 implementation plan: testing

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). How each component and the whole flow are tested. Several guarantees in v5 are security properties; they get tests of their own, not only happy paths.

## Principles

- **The protocol has one set of test vectors**, used by both the SDK and the PM-side ([01 Protocol, "Deliverables"](./xxx-notes-011-v5-impl-01-protocol.md)).
- **Security guarantees are tested as negative cases**: what must be refused is enumerated and checked, for every verb and both kcp path shapes.
- **E2E runs with a real network boundary**: PM can't reach the Provider except through its base URL, and the Provider can't reach PM at all.
- **Every row of v5's failure table has a test.**

## Unit tests

| Component | Focus |
|---|---|
| SDK ([07](./xxx-notes-011-v5-impl-07-provider-sdk.md)) | Inner mutual TLS against the vectors (client certificates only issued directly by a pinned CA, with `clientAuth`; pinned server keys); the hello's `clusterName` check; handshake limits and timeouts, unknown connection IDs refused before the upgrade; control sessions closed on the first failed authentication; random connection IDs; token issuing and spending; idempotent registration; adopting `pendingSecretHash` only from the record, read over a valid session, and never through a call; bearer check against the current hash only; connector CA bundle compare-and-swap (`412` on a missing or stale `If-Match`, identical body a no-op), accepted even when every pinned CA has expired; on a bundle change, sessions whose certificate no longer chains get go-away and are closed after the grace period, while expired-but-chaining ones and bundles that only add CAs close nothing; registration retries honoured for the whole window, and orphaned connections expired after it, never used ones only; server key switch per connection only once its record lists the key; import and re-pin commands; store compare-and-swap |
| platform-mesh-operator, onboarding and connections ([03](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)) | The oldest-request rule; the request's phases (`Pending → Registering → Registered`, `Failed` only before registration succeeds) and the `Provider` decision table, including a new registration only on a definite "unknown connection"; write-before-send and resume after a crash at each step, including between `POST` and persisting its result, and between the first control call and the record; `connectionID` and server keys persisted in one write, never only in memory; the record written only after a first successful control call; token data deleted only after the result is persisted, never before; connection Secrets only in its own namespace, no Secret informers in kcp; edge-failure classification for each reason (non-`101`, HTML challenge page, close after a binary message, regular idle or age cuts); a re-applied spent token is ignored; the secret-change state machine and its triggers, with the repair rate limit; the trusted set derived from the CA's `CertificateRequest`s (a new CA added after validation; the previous one dropped per connection within the spread after 48 hours, or at once when distrusted), and a restart deriving the same result; push decisions by fingerprint, `412` handling, the catch-up push on the first successful control call, and the tamper response (an unexpired CA that isn't one of PM's unexpired CAs → restore, change secret, report; an expired pinned CA, whoever issued it, isn't tampering; with the `CertificateRequest`s deleted, a Provider pinning the predecessor takes the harmless tamper path); convergence counting only reached connections; never giving up on an unreachable connection; condition aggregation; egress rules |
| Connector ([05](./xxx-notes-011-v5-impl-05-connector.md)) | Certificate switch rule (same CA: at once; new CA: once converged, or with less than an hour left), replacing no session; session rule (records, slices, same-workspace exports only), applied by each replica on its own; per-connection limits per replica; sending the hello; `Host` checks against the allow-list on every request; authorizer scope and denylist for both path shapes; Secret-type inspection for SA-token Secrets; header stripping |
| platform-mesh-operator, `Provider`s ([03](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)) | The remote steps in order (`Workspace`, `ConnectionBinding`, `ConnectorAccess`, `Registered`, `Connected`), each only after the previous one is done; no token Secret or kubeconfig; conditions and labelled messages for each failure; `phase` from the conditions for both kinds, `Degraded` only after `Ready` and kept across restarts; `ProtocolOutdated` not affecting `Ready`; the rotation annotation acted on once per value; the remote finalizer's order |
| Connector CA ([04](./xxx-notes-011-v5-impl-04-connector-ca.md)) | Chart output: the CA `Certificate` (self-signed and `issuerRef`), the CA `ClusterIssuer`, per-shard connector `Certificate`s with their key usages and no hostnames; the `ConnectorCABundle` status and conditions from one writer; an invalid CA certificate is reported and never pushed |

## Integration tests against kcp

Against a kcp test server, without Providers:
- **Maximal permission policy**, modelled on kcp's `TestMaximalPermissionPolicyAuthorizer`: an admin of a provider workspace is refused every write on `providerendpoints` and their status, and can read them; platform-mesh-operator's identity can write through the connection export's virtual workspace with only its prefixed-user grant, and can't without it. This closes v5 observation 8's untested point.
- **Record scope**: platform-mesh-operator can write the cluster-scoped, fixed-name `ProviderEndpoint` through the connection export's virtual workspace.
- **Permission claim**: connectors can read `apiexportendpointslices` through the connection export.
- **Connector token minting** with the RBAC from [03](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md): only for the named SA.
- **SA-token Secrets**: confirm kcp issues tokens for them (so the connector's inspection is needed) and that the inspection blocks every write form.
- **`Provider` validation**: the CEL rule rejects `providerKubeconfigSecret` or `hostOverride` together with `spec.remote`, on create and on update.

## E2E (kind, multi-shard)

**Environment**, extending platform-mesh-operator's kind e2e setup (`operators/platform-mesh-operator/test/e2e/kind`; see also [09 Deployment, "Local development"](./xxx-notes-011-v5-impl-09-deployment.md)): kcp with at least two shards and a front-proxy, cert-manager, platform-mesh-operator, providerconnections-operator, a connector per shard, and the [quickstart](./xxx-notes-011-v5-impl-08-provider-quickstart.md) Provider in a separate network reachable only through an ingress at its base URL. A test ingress that terminates TLS stands in for the Provider's edge.

**Scenarios:**

| Scenario | Checks |
|---|---|
| Onboarding | The request reaches `Registered`; the `Provider` goes `Provisioning → Ready`, with the record written only after the first control call; front-proxy session up; the Provider bootstraps its export through the connection; the token Secret's data is gone |
| Consumers on a second shard | Binding on shard B makes shard B's connector open a session; the Provider reconciles consumers on both shards |
| Restarts | Connector replica, platform-mesh-operator, Provider-side: traffic recovers without registration calls |
| Connector replica killed | Every replica of the shard holds a session per connection: killing one cuts only the streams on its own sessions, which client-go reopens on the others' sessions at once; no connection loses its sessions on that shard |
| Shard connector down | Only that shard's endpoint traffic stops |
| platform-mesh-operator down | Traffic continues; onboarding waits |
| platform-mesh-operator leader lost | A standby takes the Lease within seconds, rebuilds its caches and reconciles every connection once, within the per-Provider rate limits |
| Readiness never arrives | kcp unreachable from the connectors: the request is `Registered`, the `Provider` stays `Provisioning` with `Connected=False` and the `KcpUnreachable` label, without a timeout; it becomes `Ready` once fixed |
| Control path blocked at onboarding | The test edge blocks `/control`: the request is `Registered`, the `Provider` shows `Registered=False` with the edge-diagnosis label, no record is written and no session opens; unblocking lets it continue |
| Connection breaks after `Ready` | The test edge cuts all sessions of a `Ready` remote Provider: `Connected=False` with `NoSessions`, `phase` `Degraded`; back to `Ready` once sessions return |
| Operator crash during onboarding | platform-mesh-operator is killed after `POST`, after persisting its result, and after the first control call, in turn: each time the request still reaches `Registered`, the `Provider` reaches `Ready` with the same `connectionID`, without a second connection on the Provider-side |
| Two requests for one `Provider` | The newer one fails, naming the active one |
| Misconfigured edge | The test ingress in turn: returns an HTML challenge page on upgrade, drops binary messages, enforces a short idle timeout, cuts connections at a short age. Each yields its distinct label in the `Provider`'s condition message |
| Well-configured edge | The quickstart's traefik `HTTPRoute` passes sessions and control sessions; 32 KB binary messages pass both ways |
| Secret change on demand | The `providers.platform-mesh.io/rotate-connection-secret` annotation on the `Provider`; re-applying the same value does nothing; a new value: the hash goes through the record, the Provider-side adopts it, the old secret is refused afterwards; sessions unaffected |
| Lost connection Secret | Deleted from platform-mesh-operator's namespace: platform-mesh-operator repairs it through the record; `connectionID` and state kept; no owner action |
| Provider-side rejects PM's secret | The quickstart's store is restored from an older snapshot: platform-mesh-operator repairs the secret; repeated failures stay within the rate limit |
| Secret change without sessions | All sessions blocked at the test edge: the change waits, the old secret keeps working, and the change completes once sessions return |
| Connection unknown at the Provider-side | Deleted in the quickstart's portal: `Registered=False` with `ConnectionUnknown`; a new request for the same `Provider` registers a new connection, the record switches, consumers' bindings keep working, and the quickstart adopts existing `Cowboy` resources through the IDs in their status. A new request while the connection is still known fails, and so does one while the Provider-side is unreachable |
| Switching Providers | A second quickstart instance at another base URL: after the first deletes the connection, a request with the second's token registers there; bindings keep working and the second adopts the existing APIExport |
| Host rename | A shard's virtual-workspace host changes: the connector lists old and new hosts in its hellos, endpoint slices switch, then the old host is dropped; no session gap in either order of changes, no certificate reissuance |
| Provider with standby replicas | Two quickstart replicas behind the test ingress: upgrades reaching the standby get `503` with `Retry-After: 5` (default Lease) and connectors land on the leader within a few attempts, never retrying sooner than that; killing the leader moves all sessions to the new leader |
| Server key rotation | Record updated before switch; sessions keep working. With platform-mesh-operator stopped during the rotation, the Provider-side keeps the old key on that connection until the record lists the new one |
| Provider-side store lost | The quickstart's store is wiped and the connection imported with the administration command, with new server keys given to platform-mesh-operator as administrator input: sessions resume under the same `connectionID`, the secret is repaired through the record, and platform-mesh-operator adopts the existing `Cowboy` resources without creating new ones |
| Automatic CA rollover | With shortened lifetimes, cert-manager renews the CA (or `cmctl renew` forces it): platform-mesh-operator pushes the trusted set, the new CA converges, connectors switch, and the old CA is dropped connection by connection within the spread window; sessions still on the old CA's certificate hand over with go-away and reopen at once; no connection is left without a session |
| Control path broken during a rollover | The test edge blocks a Provider's `/control` path while its sessions stay up: convergence doesn't wait for it, it's listed as not reached, its sessions are lost after the connectors switch, and once `/control` is unblocked the catch-up push restores them within seconds |
| Connector restarted mid-rollover | A connector restarted before the new CA converges uses its newest certificate at once; sessions to connections not yet pinning the new CA fail until the push lands, then recover |
| Provider down during a rollover | The rollover completes for reachable connections; when the Provider returns, its first successful control call triggers the bundle push and sessions open within seconds. Also after two rollovers and with every pinned CA expired: the push is accepted and no tampering is reported |
| Out-of-date bundle and a lost secret | A connection whose bundle missed the rollover and whose Provider-side rejects PM's secret stays without sessions; re-pinning with the administration command brings sessions back, then the secret is repaired |
| cert-manager down past the renewal budget | Connector certificates expire; the expiry metric reaches zero and readiness reports it; open sessions keep serving traffic, no replacements are attempted; a session cut by the test edge's maximum connection age isn't reopened until renewal; after cert-manager recovers, renewal and replacements resume |
| Offboarding | Record deleted, sessions closed, `DELETE` received, connection Secret deleted, workspace removed |
| Provider-side deletes the connection | Sessions refused; `Registered=False` |

## Security tests

- **Provider attempts through a session**: edit or delete its record (refused by kcp), delete the binding, write endpoint-slice status, create a TokenRequest, create an SA-token Secret, create a CSR, `exec`, impersonation headers, requests to another connection's workspace, streams to hosts outside the allow-list. All refused and audited.
- **Owner attempts**: edit the record (no rights), point `Provider` status or spec at another workspace (ignored by platform-mesh-operator), read a connection secret (none in the owner's workspace).
- **Stolen connection secret**: used against the Provider-side to push a bundle with a foreign CA. platform-mesh-operator detects it, restores the bundle, changes the secret through the record and records an event on the `Provider`; the stolen secret is refused afterwards. No call lets the thief change the secret.
- **Stolen onboarding token or portal account**: registers only a new, separate connection that receives no sessions from PM; it can't change an existing connection's secret or bundle.
- **Malicious edge**: the test ingress tries to read session contents, open its own streams, replay a recorded handshake, and swap the server key. It sees only opaque bytes and every attempt fails.
- **Forged sessions**: a client certificate from an unpinned CA, an expired connector certificate, a chain with an intermediate, a hello with another connection's `clusterName`; a flood of upgrades with a known connection ID stays within the handshake limits.
- **Control-path guessing**: requests with wrong secrets end the control session at the first one; each new attempt costs a handshake and counts against the limits.
- **Leaked CA key**: a test client signs a connector certificate with the CA's key and opens a session; after `cmctl renew` and listing the CA in `distrust`, the Provider-side closes that session within the grace period and refuses new ones, while PM's connectors keep working.
- **Tampered bundle with sessions**: someone holding the connection secret pins their own CA and opens a session with it; platform-mesh-operator's tamper response restores the bundle, and the Provider-side closes the thief's session.
- **Egress**: base URLs pointing at private, link-local or metadata addresses are refused by platform-mesh-operator and connectors.

## Load

The connector is a second, lighter hop in front of the APIExport virtual workspace (v5, "Load"). Measure, for one shard's connector at a realistic watch count:
- memory and goroutines per idle watch, compared with the APIExport virtual workspace's;
- peak memory per stream with a slow reader, with and without the HTTP/2 and multiplexer caps;
- CPU per GB proxied;
- cost per session.

## Work breakdown

1. Protocol test vectors.
2. Unit tests per component (with each component's work).
3. kcp integration tests (policy, claim, token minting, SA-token Secrets).
4. Multi-shard kind environment with the network boundary and a TLS-terminating test ingress.
5. E2E scenarios and security tests.
6. Load test harness.

## Follow-ups

- **Profile validation** (platform-mesh-operator, a follow-up to this delivery): a profile whose `providerconnections` shard list doesn't match `infra`'s is refused and nothing changes in the cluster. Tested with that work, not here ([09 Deployment, "Charts"](./xxx-notes-011-v5-impl-09-deployment.md)).

## Decided

- **The kind environment extends platform-mesh-operator's** `test/e2e/kind`.
- **Shortened timings**: every component exposes its waiting periods, lifetimes and intervals as configuration (poll interval, repair rate limit, server key overlap deadline, connector CA and connector certificate lifetimes, the old CA's drop delay, the registration retry window, the session grace period, Lease durations), so CI can run full cycles in minutes.
