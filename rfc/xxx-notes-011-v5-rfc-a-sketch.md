# Notes v5: sketch of RFC 011 (A), Remote Providers through Connections

Working notes, not the RFC. An outline of RFC A's sections and what each must say, to see the full scope and to draw from when writing it. It follows the RFC structure in [the RFC README](./README.md). The RFC itself is self-contained: it cites none of these notes; the "From" lines are only for its author. The split between A and B is in the [split notes](./xxx-notes-011-v5-rfc-split.md); RFC B is sketched in [its own notes](./xxx-notes-011-v5-rfc-b-sketch.md).

**Size**: large. Most of the design is here. Expect roughly the v5 notes' length minus observations and most rejected alternatives, with the wire contract and the APIs as appendices.

---

## Header

- **Title**: RFC 011: Remote Providers through Connections.
- **Status**: Draft. **Authors**.
- **Supersedes**: the earlier 011 draft (`old-011-…`); extends RFC 006 (Local Providers stay as they are).
- **Followed by**: RFC 012, Operating Connections (named as a later RFC, no detail).

## 1. Summary

Three to five sentences:
- A Provider serves any number of PM Instances without holding any kcp credential, and without PM's kcp being reachable from the Provider.
- PM dials out to the Provider's Base URL; Sessions inside an end-to-end TLS carry the Provider's requests to an authorizing proxy (the Connector) that adds the kcp identity.
- Onboarding is one token from the Provider's portal and two objects in the Owner's workspace.
- The Provider's operator stays unchanged: PM ships `provider-gateway`, which hands it one ordinary kubeconfig per Connection.

## 2. Context and Problem Statement

- **Today (RFC 006)**: one static kubeconfig per Provider, an SA token without rotation, a 1:1 mapping between Provider and PM Instance, PM's kcp reachable from the Provider.
- **What's needed**: one Provider on many PM Instances; PM behind NAT, dialling out only; onboarding started by a token the Provider hands out; recovery from restarts and outages without manual steps.
- **Why not deliver kcp credentials** (the four costs: visible at TLS-terminating hops, kcp CA trust on the Provider, refresh and non-revocable bound tokens, a long-lived PM identity to push them).
- **Constraints**: any HTTPS endpoint with WebSockets must do (no TLS passthrough, no client-certificate ingress); nothing about kcp's PKI involves Providers; Owners learn nothing about PM's topology; Connectors stay stateless.
- **Non-goals**: verifying which PM Instance registers (the token is a bearer credential); Local-to-remote switching of an existing `Provider`; `direct` mode; other kcp virtual workspaces (CachedResources); an L7 guarantee for the service's own data plane.

From: v5 "Problem", "Trust boundaries"; old 011.

## 3. Terminology

- The deliverables table (components, chart, module, kinds, annotations, endpoints) and the concepts table, trimmed to what the RFC uses.
- The naming convention in one paragraph (code formatting for deliverables, Title Case for concepts).

From: v5 "Terminology".

## 4. Approach

### 4.1 Overview

- **Core-idea figure**: operator → `provider-gateway` (kubeconfig per Connection, `CONNECT`) → Sessions (WebSocket, Inner TLS, yamux, outbound from PM) → Connector per shard → kcp.
- **Component map figure**: Owner's workspace, PM's runtime cluster (`platform-mesh-operator`, Connectors, cert-manager), kcp (Provider Workspace, Connection Export), the Provider's cluster (`provider-gateway`, operator, portal).
- **Who writes what** table.
- **End-to-end walkthrough**, the flow in about ten steps: token, `Provider` and request, workspace and policies, Registration, first Control Call, `ProviderEndpoint`, Sessions where needed, readiness, run, Secret Changes and Rollovers, offboarding.

From: 00 "Component map", "End-to-end walkthrough", "Who writes what"; v5 "Core idea", "Flow overview".

### 4.2 Trust model

- **Trust boundaries**: the Owner's workspace holds no Connection credential; the Connection is end to end, so the Provider's Edge carries it without being part of it (the Provider may trust its Edge for its own traffic; the Connection doesn't need to); the outer TLS is verified strictly but isn't the anchor; the token's delivery path (portal, Owner) is inside the boundary.
- **Why the Connection doesn't rely on the Edge**: PM and consumers rely on the Connection but can't judge the Provider's Edge; the Edge's role stays what it is today (web traffic, not the Provider's control plane, which RFC 006 sends outbound); what would cross it is unusual for an Edge (acting as the Provider in kcp, data-plane credentials); it costs the Provider nothing; it also covers Edges that aren't entirely the Provider's own (third-party CDNs, shared ingress, logging, compliance); the limit (an Edge that's part of the runtime). Not "the Edge is the enemy": the Provider doesn't defend against it, it only lets WebSockets through.
- **Why not require TLS passthrough instead**: CDNs and WAFs terminate by design, PM can't check it, a certificate for the Base URL would become the Provider's identity.
- **Today vs. this RFC figure** (who can act as the Provider in kcp, who can fake kcp, who sees what).
- **What the design guarantees** (the four "makes sure that" bullets, with the leaked-Connection-Secret exception).

From: v5 "Trust boundaries", "The Provider's Edge"; 00 "The Provider's Edge".

### 4.3 The Connection API and protocol

The normative contract; full detail in Appendix A.
- **Endpoints at a glance**: `…/register/{tokenID}`, `…/{id}/control`, `…/{id}/session`; all WebSocket upgrades without credentials.
- **Onboarding Token** (`pm1.<base URL>.<keyfp>.<tokenID>.<tokenSecret>`) and **Connection Secret** (`pmc1_…`, hash only on the Provider-side).
- **Registration** inside Inner TLS pinned to `keyfp`: request and response, idempotency and the 7-day retry window, `unusedExpiresAt`.
- **Control Sessions**: Inner TLS against a pinned Server Key (`PM-Server-Key`), bearer per call, bound to the path's `{id}` (`ConnectionMismatch`), first failed authentication closes, "unknown" answered only inside; status, Pinned Bundle replacement (compare-and-swap, `etag`, `If-Match`), delete. No call changes the Connection Secret.
- **Sessions**: the inner mutual TLS (Connector Certificate ↔ Server Key) as the whole authentication; the Hello (`connectionID`, `clusterName`, hosts), and why its check is essential; yamux, Streams opened only by the Provider-side, HTTP/1.1 from the first byte; hosts as routing, not authority.
- **Pinned Bundle**: PEM, strict parsing, never empty, at least one unexpired; re-checking open Sessions on every change.
- **Changing the Connection Secret** only through the `ProviderEndpoint`'s `pendingSecretHash`.
- **Server Keys**: a list, announced in `GET`, adopted per Connection once its `ProviderEndpoint` lists them; never fetched over plain HTTPS.
- **Framing** (binary, 32 KB, 30-second pings), **status** (`workspace`, `virtualWorkspaces`), **errors and retries** (`503` with `Retry-After`), **versioning** (path version, ALPN per channel, current and previous).
- **Layers figure** for each channel.

From: 01 (all); v5 "Connection API", "Session credentials", "Sessions", "The Provider's Edge: Inner TLS on Sessions".

### 4.4 APIs

Short descriptions here, full YAML in Appendix B.
- **For Owners** (Providers Export): `ProviderOnboardingRequest` (`providerRef`, `tokenSecretRef`, `tokenID`, immutable spec, phases ending at `Registered`, oldest-wins); `Provider` with `spec.remote.caBundle`, the annotations `rotate-connection-secret` and `abandon-connection`, conditions and labels, `phase`; the `providerRef` table.
- **In PM's runtime cluster**: `ProviderEndpointRegistration` (authoritative, mutable `url`) and its `Secret`; `ConnectorCASet` in A's form (see 4.6).
- **In Provider Workspaces** (Connection Export): `ProviderEndpoint`, the export with its permission claim and maximal permission policy, its grants, the binding.
- **Policies in Provider Workspaces**: the two `ValidatingAdmissionPolicy`s (role-binding subjects; no SA-token `Secret`s), their bindings, the `apiexport-bind` ClusterRole, `matchConditions` exempting `platform-mesh-operator`, the dry-run check.
- **Provider-side kinds** (`provider-sdk.platform-mesh.io`): `Connection`, `OnboardingToken`, `ServerKey`.

From: 02 (all); 07 "API resources"; v5 "Resources".

### 4.5 PM-side components

**`platform-mesh-operator`**:
- owns every `Provider` end to end; one active replica; requests reconciled under the `Provider`'s key (`ProviderNotFound`, `RemoteProvidersNotReady`);
- reconcile steps and conditions (`Workspace`, `ConnectionBinding`, `ConnectorAccess`, `Registered`, `Connected`, `Ready`), `phase`, the labels A ships (`EdgeUpgradeRefused`, `EdgeTLS`, `ControlUnreachable`, `SecretRejected`, `ConnectionUnknown`, `ConnectionAbandoned`, `ConnectionIDMissing`, `PolicyNotEnforced`, `NoSessions`, `KcpUnreachable`, `TokenRefused`), the `unusedExpiresAt` warning;
- Registration, the first Control Call and writing the `ProviderEndpoint` (including the `SecretRejected` path);
- health polling; Secret Changes (on demand, coalesced and rate-limited; repair; after tampering; a distrusted CA still pinned), the tamper check;
- Base URL changes, abandoning a Connection, deletion;
- Connection state in its own namespace, never trusting Owner-writable links; egress rules; internet-facing code accepted for now.

**Connectors**:
- one per shard, stateless, a standalone proxy modelled on kcp's front-proxy;
- discovery through the Connection Export; the Hosting Shard from the engaged cluster; the Session rule (Hosting Shard plus shards in Service endpoint slices); every replica serves every Connection;
- opening and replacing Sessions (scheduled, on `url` or `caBundle` changes, honouring `Retry-After`); the certificate switch rule;
- the request pipeline: canonical paths, routing by path to the front-proxy or this shard's virtual workspace, logical cluster names only, the Denylist, header stripping, SA tokens, real upstream host and TLS server name;
- identity (one kcp group, from a kcp-operator `Kubeconfig`), limits, egress, observability, out-of-reach virtual workspaces.
- **Connector figure** (session manager, stream server, pipeline).

From: 03 (minus B's parts), 05 (all); v5 "`platform-mesh-operator` and the Connectors", "The Connector as an authorizing proxy", "Connector policy", "Per-shard Connectors", "Connector Replicas".

### 4.6 The Connector CA (A's form)

- **The CA**: a cert-manager `Certificate`, 1 year, renewed 90 days before expiry, new key each time; self-signed or `issuerRef`; the key only cert-manager reads.
- **Connector Certificates**: 24 hours, renewed halfway, directly from the CA, no hostnames, a common label.
- **The Connector CA Set in A**: every unexpired CA from the CA's `CertificateRequest`s, minus distrusted ones, pushed on difference; a previous CA leaves the set when it expires; ordering by revision, the current CA, "issued".
- **Convergence**: every Connection classified pinned, not reached or unknown; `Converged`; Connectors switch only after it.
- **`ConnectorCASet`**: `spec.distrust` by `key-sha256`, applied at once; `status.certificates` (`Current`, `Previous`, `Distrusted`), results `Applied` and `Unmatched`, `Valid`, `Converged`.
- **Compromise**: listing a key in `distrust` drops it at once and the Provider-side closes Sessions opened under it; the stated effect on Connectors still holding certificates from it. No procedure in A.
- **`DistrustedCAPinned`**; the signals A ships (CA expiry, validity, convergence, Connector Certificate expiry and pending switch, tampering, distrust unmatched, distrusted CA pinned).
- **Why a CA** (authorizes Sessions only, concentrates nothing new, short certificates instead of revocation, one certificate safe for every Connection), and the options table.
- **How it compares** to established trust-anchor distribution (push vs. pull; why push behind NAT).

From: 04 (minus B's parts); v5 "Session credentials: the Connector CA"; split notes "What A does more simply".

### 4.7 The Provider-side: `provider-gateway`

- **What a Provider gets**: the Connection API server, the operator proxy, a kubeconfig `Secret` per Connection, its state as Kubernetes resources with Events, metrics; the token helper in the Go module.
- **Principles**: no kcp knowledge; everything security-relevant in `provider-gateway`; standard machinery for the operator; the Kubernetes API as the only source of truth; state per `connectionID`; failed authentications change nothing.
- **Kubeconfig per Connection**: placeholder workspace URL, `proxy-url` with a per-Connection credential, `tls-server-name`, the gateway's CA; in the gateway's namespace, read by the operator through a Role; `rotate-proxy-credential`.
- **Operator proxy**: `CONNECT`, `407`/`404`/`503`, Session choice by host, TLS inside the tunnel, ALPN `http/1.1`, status-line bookkeeping.
- **Connection API, Control Sessions, Session server** (the Provider-side's checks), orphan expiry.
- **Server Keys**: generated ECDSA P-256 in `Secret`s, rotation on demand with `retireAt`; KMS later without breaking changes.
- **Replicas**: one active `provider-gateway`, standbys `503`; the operator independent.
- **Requirements on Providers**: install the gateway and route the paths; the Edge checklist; endpoint slices in the Provider Workspace; reachable scope; restricted RBAC and `apiexport-bind`; keying consumer data with `connectionID`, adopt don't create, removing a Connection never deletes consumer data, bootstrap only creates or adopts; the connection-pooling caveat and the optional shared transport.
- **Serving many PM Instances**: one kubeconfig per Connection, a kcp-multicluster-provider per Connection, prefixed cluster names.
- **Releases**: one semver line, protocol versions, upgrade signals (`ProtocolOutdated`).
- **Provider-side figure** (gateway, operator, Secrets, Edge).

From: 07 (minus the administration runbooks); 08 "Connection API server" (the Edge checklist); v5 "Provider-side prerequisites", "Provider-side: serving many PM Instances".

### 4.8 Recovery and lifecycle

- **Each loss and its way back** (the Recovery table): PM-side registration or secret lost (repair); Connection unknown (import or onboard again, switching Providers); Provider gone for good (abandon); Base URL moved (PM Administrator edit); every Session lost with the secret lost too (out-of-band re-pin).
- **What keeps disaster recovery possible**: stable `connectionID`s, importable `Connection`s, PM Administrator input, data never deleted with a Connection. Procedures are left for later.
- **Offboarding and revocation**: `ProviderEndpoint` deleted first, Sessions closed at once, `DELETE`, nothing to revoke.

From: v5 "Recovery", "Offboarding and revocation"; 03 "Abandoning a Connection", "Deletion".

### 4.9 Restart and failure semantics

- The table, A's rows: Connector Replica and shard events, Provider-side restarts and replicas, Secret Changes, renewals with the overlapping set, immediate distrust, `platform-mesh-operator` down (including the 12-hour-at-renewal exception), outages of any length, Edge cuts, token refused, first contact past `unusedExpiresAt`, workspace moved between shards, front-proxy renamed.

From: v5 "Restart and failure semantics" (minus B's rows).

### 4.10 Security

- **What each credential exposes** (the table, including the Connectors' kcp identity and the Edge).
- **Threat model** (`platform-mesh-operator`, a Connector, Sessions opened with a leaked key, the Owner's workspace, a Connection Secret holder, the Edge, a compromised Provider-side, Owners, the Provider).
- **The leaked-Connection-Secret limitation**, stated plainly with its recovery.

From: v5 "What each credential exposes", "Threat model", "Changing the Connection Secret".

### 4.11 Deployment

- kcp bootstrap (Providers Export addition, Connection Export in `:root:providers:system`, grants).
- The `providerconnections` chart: `kcpTopology`, per-shard Connectors with `Kubeconfig`s and Connector Certificates, the CA and its issuer; deployed as a profile service.
- `platform-mesh-operator`'s additions (CRDs, RBAC, configuration of its kcp username).
- Network (in-cluster upstreams, direct egress, optional NetworkPolicies, the residual), observability (metrics, `ServiceMonitor`s, documented alert rules), rollout driven by `platform-mesh-operator`.
- The client-CA assumption for the Connectors' identity.

From: 09 (minus `revisionHistoryLimit: 10`).

### 4.12 Reference Provider and testing

- The quickstart: `provider-gateway` from the chart, both modes behind one abstraction, bootstrap per Connection, its small portal, the README as the Provider-facing guide.
- Testing approach: shared test vectors, negative security tests, e2e with a real network boundary, every failure row tested; the main e2e and security scenarios listed by name.

From: 08, 10 (minus B's rows).

## 5. Pros & Cons

- **Pros**: no kcp credential on the Provider-side, immediate revocation; PM behind NAT; one Provider, many PM Instances; the Connection end to end, independent of the Edge; nothing to rotate by hand; Owners see no topology; the Provider's operator unchanged; kcp load unchanged, one lighter hop added.
- **Cons**: a second TLS layer and Server Keys to keep; a new hop on every request (and its load to size); Sessions multiplied by Connector Replicas at the Provider's Edge; a gateway to install; no transport sharing in client-go through a proxy; internet-facing code in `platform-mesh-operator`; the leaked-Connection-Secret lock-out; the Connectors' shared kcp identity as the largest exposure.

## 6. Alternatives

The ones a reviewer would ask about, each in one or two sentences:
- delivering kcp credentials (a dial proxy, a kubeconfig per Connection);
- plain Sessions trusting the Edge; requiring TLS passthrough;
- a Session token on the upgrade; signed Control Calls;
- per-shard keys registered per Connection; one PM key; long-lived CA generations with manual Rollover; a pinned root with an intermediate;
- the SDK as a library inside the operator; a reverse proxy by path;
- HTTP/2 or HTTP/1.1 directly on the WebSocket; remotedialer;
- the Connector as a kcp virtual workspace; one Connector for all shards; impersonation instead of SA tokens;
- discovery from Owners' `Provider`s; `ProviderEndpoint`s in a PM-internal workspace or the cache;
- the Connection Secret in the Owner's workspace; closing the leaked-secret lock-out (why each option costs too much);
- folding the request into `Provider`; a separate operator for Connections.

From: v5 "Rejected alternatives", 05 "Declined"; the transport and gateway notes.

## 7. Open questions

- None blocking. Possibly: the kcp impersonation question on the APIExport virtual workspace, to report upstream; the hop budget (`X-Kcp-Virtual-Resource-Hops`).

## Appendices

- **A. Wire contract**: every endpoint, message and status code, the token and secret formats, the Hello, framing, ALPN names, errors; test vectors as a deliverable.
- **B. API reference**: full YAML for every kind, its validation and printer columns.
- **C. Facts the design relies on**: the observations about kcp (maximal permission policy, impersonation on virtual workspaces, the front-proxy's routing, kcp-operator's cache credentials), kcp-multicluster-provider's engagement, client-go's proxy handling; each with the code location.
- **D. Prior art**: kcp front-proxy, Teleport, Cloudflare Tunnel, remotedialer, Konnectivity, OCM cluster-proxy, kubeadm tokens, prefixed secrets, IAM Roles Anywhere and SPIFFE for trust anchors, HTTP conditional requests.

## Figures to draw

1. Core idea: operator → gateway → Sessions → Connector → kcp.
2. Component map across the four places.
3. Today vs. this RFC: who holds what, who sees what.
4. Layers per channel.
5. Per-shard Connectors and which Sessions exist.
6. The Connector's internals.
7. The Provider-side: gateway, operator, Secrets, Edge.
8. Onboarding sequence (Owner, portal, `platform-mesh-operator`, `provider-gateway`, kcp, Connector).
