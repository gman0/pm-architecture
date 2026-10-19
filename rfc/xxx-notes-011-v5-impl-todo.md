# v5 implementation plan: follow-ups

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). Work that the plan doesn't depend on and that can be done separately, before, alongside or after it. Each item says what and why in brief; the details are worked out when it's picked up.

## Cleanups in existing code

- **Rename the Local Provider steps in `platform-mesh-operator`'s code** (`ProviderWorkspaceSubroutine`, `ScopedKubeconfigSubroutine`) to `Workspace` and `Kubeconfig`. Their names become the condition types Owners see, consistent with the remote steps (`Workspace`, `ConnectionBinding`, `ConnectorAccess`, `Registered`, `Connected`).
- **ManagedProviders and `Degraded`**: decide how a ManagedProvider reacts when its `Provider` goes from `Ready` to `Degraded`. Its wait step compares `phase` with `Ready`.
- **Fix the `Provider`'s `KubeconfigSecret` printer column**: it points at `.status.kubeconfigSecretRef.name`, while the field is `providerKubeconfigSecretRef`.
- **Update the types in `platform-mesh/apis`** to the plan: the `ProviderOnboardingRequest` type still has the `Rebinding` phase and `connectionSecretRef`; `Provider`'s condition constants still list `SessionsConnected`, `KcpReachable`, `SecretCurrent` and `Rebinding`; the doc comment on `status.connection` still mentions rebinding. The request also gains `spec.tokenID` and an immutable spec. The new `ProviderEndpointRegistration`, `ConnectorCASet` (with `spec.distrust`) and `ProviderEndpoint` kinds go into `providers.platform-mesh.io`, next to `ManagedProvider` in `platform-mesh-operator`'s `api/providers/v1alpha1`.

## Larger follow-ups

- **Split the `Provider` part of `platform-mesh-operator` into its own operator**, taking the Remote Providers' Connection code with it. Until then, code that talks to Providers on the internet runs in PM's most privileged component; the split ends that.
- **Deletion protection for `Provider`s with consumer bindings.** Deleting a `Provider` deletes its workspace and the APIExport's identity, which kcp can't re-pair with existing bindings.
- **Recovery runbooks**: for Provider Administrators, importing and re-pinning `Connection`s with `kubectl` (07 SDK, "Administration", has the basics); for PM Administrators, editing a `ProviderEndpointRegistration`'s Base URL and Server Keys. When and how, step by step.
- **Periodic Connection Secret rotation, if compliance requires it.** Not needed for security: a stolen Connection Secret is used at once or not at all (a pushed bundle is detected and undone if PM sees it first; a lock-out of the Connection is recovered out-of-band), so rotation would only shorten how long they could read status. It's a small addition to the existing Secret Change: a last-change timestamp in the `ProviderEndpointRegistration`, a check on each health poll against a configured interval (zero means off), and due times spread by `connectionID`. The code carries a comment at the list of Secret Change triggers saying exactly this: what would change, why it might be wanted (compliance), and why it isn't needed now.
- **Horizontal scale for `provider-gateway`**: an opt-in mode with owner replicas. Each Connection gets an owner replica (rendezvous hashing over live replicas, or a Lease per Connection); any replica accepts an upgrade and forwards the raw bytes, in-cluster, to the owner, which terminates the Inner TLS, so forwarding replicas see only ciphertext. Control Sessions and the operator's tunnels are forwarded the same way, to the owner of the tunnel's Connection. When Connections move on scaling, the old owner sends go-away and the Connector reconnects. It needs pod-to-pod connectivity in `provider-gateway`'s namespace and RBAC for Leases or EndpointSlices; the Provider's operator is unaffected. No PM-side work: the protocol already makes no assumption about which replica a Session reaches.
- **An Edge self-test for Providers**: an optional echo endpoint in `provider-gateway`'s Connection API server, enabled by a test secret, and a command Providers run from outside their network against their Base URL. It checks that upgrades succeed for non-browser clients, that 32 KB binary messages pass unchanged, and how the Edge treats idle connections and connection age. Until then, the quickstart's checklist and `platform-mesh-operator`'s Edge diagnosis cover it.
- **Server Keys in a KMS or HSM**, for Providers that need non-exportable keys: an optional key source on `ServerKey` (immutable, so a different source is a new `ServerKey`), with built-in signers in `provider-gateway` for the common KMSes (e.g. sigstore's packages: Google Cloud KMS, AWS KMS, Azure Key Vault, Vault Transit; access from `provider-gateway`'s pod identity, configured in its chart), and an external-signer socket, the way Kubernetes' KMS v2 plugins work, for HSMs (PKCS#11) and in-house KMSes, so no code has to go into PM's image. Moving an existing key there is a normal rotation. No protocol or PM-side change: the PM-side pins public keys only. It stays easy as long as the Inner TLS signs through `crypto.Signer`, key sources stay immutable, and keys are ECDSA P-256.
- **Periodic Server Key rotation, if compliance requires it**: the SDK creating a new `ServerKey` on a schedule and setting `retireAt` on the old one, the same steps an administrator takes today.
- **Suspending a Connection without deleting it**, if Providers need it: a field on the `Connection` that takes it out of the registry (its controllers disengage, Sessions and upkeep continue), reported in the status `GET` so `platform-mesh-operator` can show it to the Owner.
- **Per-replica Connector Certificates** with an external issuer, so no private key is mounted into Connectors.
- **Iterate on the quickstart's portal** once the RFC is implemented: real login, Owner notifications from `Connection` Events, more Connection detail.
- **Moving to platform-mesh-deployer** (`operators/platform-mesh-deployer`, unreleased as of 2026-10-12, today used only for kcp in the Tilt dev environment). It changes packaging, not design:
  - *Connectors* become a `per-shard` component in each shard's cluster, with a minted kubeconfig instead of a kcp-operator `Kubeconfig`;
  - *the work for Remote Providers* moves with the `Provider` part of `platform-mesh-operator` once that's its own operator, as a `root-shard` component;
  - *kcp bootstrap* moves to the module workspace `root:modules:providerconnections`; the Connection Export's home changes, so `ProviderEndpoint`s and bindings are migrated or recreated;
  - *`kcpTopology`* is replaced field for field by the deployer's per-shard context, and profile validation becomes unnecessary;
  - *the Connector CA*'s keys and issuers stay on the management cluster; Connector Certificates are minted there and synced to each shard's cluster;
  - *identities* become minted users (`module:providerconnections:<kubeconfig>`); rights outside the module workspace are granted to those names.

  The deployer needs first: hosts passed to per-shard instances (shard, virtual-workspace and front-proxy URLs, in-cluster addresses); client certificates per instance from a configurable issuer, with chosen key usages; rights outside the module workspace for minted identities, or a documented place where other components grant them.
- **Load tests of the Connector**, the second, lighter hop in front of the APIExport virtual workspace (v5, "Load"). For one shard's Connector at a realistic watch count: memory and goroutines per idle watch, compared with the APIExport virtual workspace's; peak memory per Stream with a slow reader, with and without the HTTP/2 and multiplexer caps; CPU per GB proxied; cost per Session.
- **In-band rotation of a private outer CA**, if Providers with many Owners on private CAs need it: the Provider-side announces a new CA in the status `GET` (protected by the Inner TLS), and `platform-mesh-operator` adds it to the `ProviderEndpoint`'s `caBundle`, so Owners don't have to update theirs. Until then, rotation is out-of-band.
- **In-band Base URL changes**: the Provider-side announces a new Base URL in the status `GET` (protected by the Inner TLS), and `platform-mesh-operator` verifies it (egress rules, outer TLS with the `Provider`'s `caBundle`, Inner TLS with the pinned Server Key) before switching the `ProviderEndpointRegistration`'s `url`, so a missing `caBundle` update only delays the switch. Today a PM Administrator edits the URL per Connection. Nothing blocks it: the SDK isn't bound to one host or path prefix, Connectors replace Sessions when `url` changes, and `url` is already mutable.
- **Automate renewing the Connector Certificates** in the compromise response: `platform-mesh-operator` triggering it when a CA is listed in `distrust`, instead of the runbook's `cmctl renew` by label. It needs rights on cert-manager's `Certificate`s in the Connectors' namespace. Keeping it manual for now also guards against a wrong CA listed in `distrust`.
- **MAC'd `connectionID`s**, if unknown-ID handshakes on `/control` ever become a load problem: IDs carrying a MAC under a Provider-side key, so the SDK refuses forged IDs before the upgrade without looking them up. Today the separate, small limit for unknown IDs covers it.
- **Egress through an HTTP proxy**, for PM Instances whose networks only allow egress through one:
  - *configuration*: an explicit proxy (`providerEgress.proxy.url`, optional `credentialsSecretRef`), from one profile value for `platform-mesh-operator` and the Connectors, used only by the Provider-dial transports (the operator's protocol client, the Connector's Session dialer), never taken from the environment;
  - *address checks*: the configured proxy address is exempt from the fixed refusals (so a loopback auth helper works); target checks become string-only (IP literals, `localhost`), since the proxy resolves names; loopback and metadata on the proxy's side are the proxy's to enforce;
  - *diagnosis*: proxy failures (`proxyconnect` errors, a refused `CONNECT`) get their own label, `EgressProxy`, never an Edge label;
  - *TLS-inspecting proxies*: first choice, exempting Provider domains from inspection (a runbook item); fallback, `additionalTrustedCAs` added to web PKI and every `caBundle`, which doesn't weaken Connections since outer TLS isn't the trust anchor; Edge-label messages say "the Provider's Edge or the egress proxy" when a proxy is configured;
  - *authentication*: Basic, from a `Secret` that reloads on change, preferably over an `https://` proxy URL; NTLM and Kerberos through a loopback auth-helper sidecar, not implemented.
- **CachedResources for Remote Providers**: evaluate whether Providers need the `replication` virtual workspace, whether it's served per shard, and how its endpoint slices look; then add it as a path shape to the Connector.

## Edge diagnosis for Sessions

### What

Extend Edge diagnosis ([03 Operator, "Edge diagnosis"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)) from Control Sessions to Sessions, so that an Owner whose `Provider` shows `Connected=False` learns whether the Provider's Edge is the cause, and which Edge rule.

### Why

Edge labels come only from `platform-mesh-operator`'s Control Sessions. Connectors classify their Session failures the same way, but only into their metrics, without Connections, for PM Administrators; Connectors write nothing. Sessions meet the Edge differently from Control Sessions:

| Edge problem | Control Session | Session |
|---|---|---|
| A route or rule for `/control` but not `/session` | works | refused, or an HTML page |
| Payload inspection triggered by size or content | tiny messages, passes | real traffic up to 32 KB, cut |
| Idle or maximum-age limits | short-lived, rarely reaches them | lives for hours, cut at regular times |
| Per-client connection limits | one connection | R replicas × shards, all from PM's egress address |
| `/session` pinned to a standby replica | — | `503` with `Retry-After`, forever |

In each case the Owner sees a bare `NoSessions`. The only people who can tell why are PM Administrators reading Connector metrics and logs, and they can't act on it: fixing the Edge is the Provider's job, and reaching the Provider is the Owner's.

### How

Who sees what decides where each piece goes:
- **Connectors** see every failure, including what the Edge answers on `/session`, but write nothing.
- **The SDK** sees whatever reaches it: upgrades, handshakes, and how Sessions end. When the Edge blocks the upgrade, the SDK sees nothing, and that absence is the signal.
- **`platform-mesh-operator`** reads the SDK's status `GET` and knows when it wrote the `ProviderEndpoint`, so it knows when Connectors should have dialled.

The pieces:
1. **The SDK reports Session-path observations** in the status `GET` ([01 Protocol, "Status"](./xxx-notes-011-v5-impl-01-protocol.md)), per Connection, over a recent window (e.g. 15 minutes), aggregated and without shard names: upgrades received; handshakes completed or failed; Sessions ended by kind (go-away from either side, clean close, cut without a close frame); and for cuts, their typical age and whether a large message came just before. The same go into the SDK's metrics, without Connections.
2. **`platform-mesh-operator` labels them** in `Connected`'s message, once the state has lasted a few minutes:
   - `EdgeSessionPathBlocked`: the `ProviderEndpoint` has existed long enough, but no upgrade reached the Provider-side; the Edge doesn't route `/session`, refuses it, or sends it only to a standby replica;
   - `EdgeTimeout`: Sessions cut without a close frame at a regular age or idle time;
   - `EdgePayloadBlocked`: Sessions cut right after large messages;
   - otherwise, `NoSessions` as today.

   Each message points at the Edge checklist, like the existing labels.
3. **A payload probe on the control path**: on the first Control Call and after a reconnect, `platform-mesh-operator` makes one padded call inside the Control Session, e.g. `POST /v1/connections/{id}/echo` with about 64 KB, which the SDK echoes, with a size cap. Since the checklist routes all of `/v1/connections/*` the same way, payload blocking then shows up as `EdgePayloadBlocked` before any Session exists.
4. **PM-side causes first**: if no Connector is Ready, the reason is `ConnectorsUnavailable`, not an Edge label, so a PM outage isn't blamed on the Provider. `platform-mesh-operator` reads Connector readiness through its runtime-cluster access, without the shard list.
5. **The Edge checklist** ([08 Quickstart, "Connection API server"](./xxx-notes-011-v5-impl-08-provider-quickstart.md)) adds per-client connection limits (R × shards from one address) and no pinning of `/session` to one backend.

**Limits**:
- The exact response the Edge gives on `/session` (status, HTML page, TLS error) is seen only by Connectors and stays in their logs and metrics, for PM Administrators. `EdgeSessionPathBlocked` is enough to send the Provider to the checklist.
- The SDK keeps the counts in memory, so a Provider failover resets the window; labelling only after the state has lasted a few minutes (like `Ready`'s) avoids false labels.

**Declined**: Connectors writing per-Connection status somewhere `platform-mesh-operator` could read it. It would end "Connectors write nothing", for a gain the SDK's observations mostly cover.

**Tests** ([10 Testing](./xxx-notes-011-v5-impl-10-testing.md)): the test Edge breaks only `/session`, cuts long-lived Sessions, drops large binary messages only on `/session`, and pins `/session` to a standby; each case yields its label. With the Connectors stopped, the label is `ConnectorsUnavailable`.

## Outside this plan

- **Profile validation** in `platform-mesh-operator`: refuse a `providerconnections` shard list that doesn't match `infra`'s.
- **Alerting**: PM has no alerting pipeline; the plan ships metrics, `ServiceMonitor`s and documented alert rules.
- **Telling consumers when the implementation behind an export changes**, e.g. after an Owner switches Providers: a PM-wide question (marketplace, notifications), not specific to Remote Providers.
- **Narrowing the Providers Export's `Secret`s claim** with a label selector in PM's bindings, a PM-wide hardening.
- **Retention of the Connector's request log**, PM's audit trail for Provider traffic.
- **Possible kcp issue**: impersonation on the APIExport virtual workspace (v5 notes, observation 5). Confirm with an e2e test before reporting it.
- **kcp-multicluster-provider: re-engagement when a logical cluster migrates between shards.** The provider watches each shard's virtual workspace separately, and nothing orders events across shards. If the cluster's objects appear on the new shard before they disappear from the old one, `update()` skips the add because the cluster is still engaged, and the old shard's delete then disengages it by name; the cluster stays disengaged until its `APIBinding` changes again or the informer resyncs (`pkg/provider/provider.go`, at `fa033a7`). Fix upstream: `update()` replaces an engaged cluster that came from a different endpoint (by the engaged cluster's config `Host`), and the delete handler only disengages a cluster engaged from its own endpoint; tests for both event orders live there. Connectors and `platform-mesh-operator` both rely on it: Connectors decide which shard hosts a Provider Workspace from the engaged cluster's config `Host` ([05 Connector, "Which Sessions"](./xxx-notes-011-v5-impl-05-connector.md)).
