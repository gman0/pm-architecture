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
| Session CA ([04](./xxx-notes-011-v5-impl-04-session-ca.md)) | Generated certificates' constraints, key usages and SANs |

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
| Server key rotation | Record updated before switch; sessions keep working |
| Session CA rollover | Next generation pinned everywhere, shard certificates reissued, old unpinned |
| Offboarding | Record deleted, sessions closed, `DELETE` received, workspace removed |
| Provider-side deletes the connection | Sessions refused; `Registered=False` |

## Security tests

- **Provider attempts through a session**: edit or delete its record (refused by kcp), delete the binding, write endpoint-slice status, create a TokenRequest, create an SA-token Secret, create a CSR, `exec`, impersonation headers, requests to another connection's workspace, streams to hosts outside the allow-list. All refused and audited.
- **Owner attempts**: edit the record (no rights), point `Provider` status or spec at another workspace (ignored by the operator).
- **Malicious edge**: the test ingress tries to read session contents, open its own streams, replay an assertion, and swap the server key. It sees only opaque bytes and every attempt fails.
- **Forged sessions**: a certificate from an unpinned CA, an expired shard certificate, a mismatched inner client certificate, a wrong `cluster` claim.
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

## Decided

- **The kind environment extends platform-mesh-operator's** `test/e2e/kind`.
- **Shortened timings**: every component exposes its waiting periods, lifetimes and intervals as configuration (rebind waiting period, readiness timeout, poll interval, rotation period, shard certificate lifetime, Lease durations), so CI can run full cycles in minutes.
