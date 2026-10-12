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
| SDK ([07](./xxx-notes-011-v5-impl-07-provider-sdk.md)) | Assertion verification against the vectors; inner TLS pinning both ways; client certificate must equal the assertion leaf; token issuing and spending; idempotent registration; secret staging and promotion; rebind waiting and cancellation; store compare-and-swap |
| Operator ([03](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)) | Shard manager: Leases follow shards appearing and disappearing in the endpoint slice, rebalance when replicas come and go, never two owners per shard; one worker's failure doesn't stop others; the oldest-request rule; no terminal failure after registration; every onboarding phase and the `Provider` decision table; write-before-send and resume after a crash at each step; token data deleted only after the result is persisted, never before; no Secret informers (the operator never lists or watches Secrets); edge-failure classification for each reason (non-`101`, HTML challenge page, close after a binary message, regular idle or age cuts); a re-applied spent token is ignored; rotation and rebind state machines; condition aggregation; egress rules |
| Connector ([05](./xxx-notes-011-v5-impl-05-connector.md)) | Session rule (records, slices, same-workspace exports only); rendezvous hashing; stream preamble checks; authorizer scope and denylist for both path shapes; Secret-type inspection for SA-token Secrets; header stripping |
| Provider controller ([06](./xxx-notes-011-v5-impl-06-provider-controller.md)) | Remote branch: no token Secret or kubeconfig, binding and connector RBAC, condition, finalizer ordering |
| Connector CA ([04](./xxx-notes-011-v5-impl-04-connector-ca.md)) | `ConnectorCABundle` CEL rules: exactly one `Active`, at most one `Next`, no removed generations, forward-only states, immutable certificates, rejected on hand edits too; derived sequence only increases; aggregator conditions from per-shard entries and `Certificate` objects; connector certificates' key usages, with no hostnames; the connector's chain check keeps the old certificate on a mismatch |

## Integration tests against kcp

Against a kcp test server, without Providers:
- **Maximal permission policy**, modelled on kcp's `TestMaximalPermissionPolicyAuthorizer`: an admin of a provider workspace is refused every write on `providerendpoints` and their status, and can read them; the operator's identity can write through the connection export's virtual workspace with only its prefixed-user grant, and can't without it. This closes v5 observation 19's untested point.
- **Record scope**: the operator can write the cluster-scoped, fixed-name `ProviderEndpoint` through the connection export's virtual workspace.
- **Permission claim**: connectors can read `apiexportendpointslices` through the connection export.
- **Connector token minting** with the RBAC from [06](./xxx-notes-011-v5-impl-06-provider-controller.md): only for the named SA.
- **SA-token Secrets**: confirm kcp issues tokens for them (so the connector's inspection is needed) and that the inspection blocks every write form.

## E2E (kind, multi-shard)

**Environment**, extending platform-mesh-operator's kind e2e setup (`operators/platform-mesh-operator/test/e2e/kind`; see also [09 Deployment, "Local development"](./xxx-notes-011-v5-impl-09-deployment.md)): kcp with at least two shards and a front-proxy, cert-manager, platform-mesh-operator, providerconnections-operator, a connector per shard, and the [quickstart](./xxx-notes-011-v5-impl-08-provider-quickstart.md) Provider in a separate network reachable only through an ingress at its base URL. A test ingress that terminates TLS stands in for the Provider's edge.

**Scenarios:**

| Scenario | Checks |
|---|---|
| Onboarding | Request reaches `Ready`; record written; front-proxy session up; the Provider bootstraps its export through the connection; the token Secret's data is gone |
| Consumers on a second shard | Binding on shard B makes shard B's connector open a session; the Provider reconciles consumers on both shards |
| Restarts | Connector replica, operator, Provider-side: traffic recovers without registration calls |
| Shard connector down | Only that shard's endpoint traffic stops |
| Operator down | Traffic continues; onboarding waits |
| Operator replica lost | Its shards' Leases move; control work resumes on the remaining replicas |
| Readiness never arrives | Request stays in `VerifyingKcp` with a warning condition; becomes `Ready` once fixed |
| Two requests for one `Provider` | The newer one fails, naming the active one |
| Misconfigured edge | The test ingress in turn: returns an HTML challenge page on upgrade, drops binary messages, enforces a short idle timeout, cuts connections at a short age. Each yields its distinct reason on the request and `Provider`; the SDK's self-test flags the same problem from outside |
| Well-configured edge | The self-test passes against the quickstart's traefik `HTTPRoute`; 32 KB binary messages pass both ways |
| Secret rotation | Rotates automatically; sessions unaffected |
| Lost connection Secret | Rebind completes after a shortened waiting period; `connectionID` and state kept |
| Rebind with the old secret alive | Cancelled by the next poll |
| Host rename | A shard's virtual-workspace host changes: the connector lists old and new hosts in its assertions, endpoint slices switch, then the old host is dropped; no session gap in either order of changes, no certificate reissuance |
| Provider with standby replicas | Two quickstart replicas behind the test ingress: upgrades reaching the standby get `503` with `Retry-After: 5` (default Lease) and connectors land on the leader within a few attempts, never retrying sooner than that; killing the leader moves all sessions to the new leader |
| Server key rotation | Record updated before switch; sessions keep working |
| Connector CA rollover | The four value changes, each waiting for its condition (`ReadyToActivate`, `ReadyToRetire`, `BundleConverged`); connector certificates reissued; old generation unpinned |
| Provider down during a rollover | The rollover completes for reachable connections; when the Provider returns, its first successful control call triggers the bundle push and sessions open within seconds |
| cert-manager down past the renewal budget | Connector certificates expire; the expiry metric reaches zero and readiness reports it; open sessions keep serving traffic, no replacements are attempted; a session cut by the test edge's maximum connection age isn't reopened until renewal; after cert-manager recovers, renewal and replacements resume |
| Wrong certificate in values | A generation whose inline certificate doesn't match its key: connectors keep their previous certificate and report the chain-check failure; no Provider pins a bundle that breaks sessions |
| Offboarding | Record deleted, sessions closed, `DELETE` received, workspace removed |
| Provider-side deletes the connection | Sessions refused; `Registered=False` |

## Security tests

- **Provider attempts through a session**: edit or delete its record (refused by kcp), delete the binding, write endpoint-slice status, create a TokenRequest, create an SA-token Secret, create a CSR, `exec`, impersonation headers, requests to another connection's workspace, streams to hosts outside the allow-list. All refused and audited.
- **Owner attempts**: edit the record (no rights), point `Provider` status or spec at another workspace (ignored by the operator).
- **Malicious edge**: the test ingress tries to read session contents, open its own streams, replay an assertion, and swap the server key. It sees only opaque bytes and every attempt fails.
- **Forged sessions**: a certificate from an unpinned CA, an expired connector certificate, a mismatched inner client certificate, a wrong `cluster` claim.
- **Egress**: base URLs pointing at private, link-local or metadata addresses are refused by the operator and connectors.

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
- **Shortened timings**: every component exposes its waiting periods, lifetimes and intervals as configuration (rebind waiting period, readiness timeout, poll interval, rotation period, connector certificate lifetime, Lease durations), so CI can run full cycles in minutes.
