# v5 implementation plan: follow-ups

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). Work that the plan doesn't depend on and that can be done separately, before, alongside or after it. Each item says what and why in brief; the details are worked out when it's picked up.

## Cleanups in existing code

- **Rename the Local Provider steps in `platform-mesh-operator`'s code** (`ProviderWorkspaceSubroutine`, `ScopedKubeconfigSubroutine`) to `Workspace` and `Kubeconfig`. Their names become the condition types Owners see, consistent with the remote steps (`Workspace`, `ConnectionBinding`, `ConnectorAccess`, `Registered`, `Connected`).
- **ManagedProviders and `Degraded`**: decide how a ManagedProvider reacts when its `Provider` goes from `Ready` to `Degraded`. Its wait step compares `phase` with `Ready`.
- **Fix the `Provider`'s `KubeconfigSecret` printer column**: it points at `.status.kubeconfigSecretRef.name`, while the field is `providerKubeconfigSecretRef`.
- **Update the types in `platform-mesh/apis`** to the plan: the `ProviderOnboardingRequest` type still has the `Rebinding` phase, `connectionSecretRef` and no `VerifyingControl`; `Provider`'s condition constants still list `SessionsConnected`, `KcpReachable`, `SecretCurrent` and `Rebinding`; the doc comment on `status.connection` still mentions rebinding. The new `ProviderEndpointRegistration` kind goes into `platform-mesh-operator`'s `api/providers/v1alpha1`, next to `ManagedProvider`.

## Larger follow-ups

- **Split the `Provider` part of `platform-mesh-operator` into its own operator**, taking the Remote Providers' Connection code with it. Until then, code that talks to Providers on the internet runs in PM's most privileged component; the split ends that.
- **Deletion protection for `Provider`s with consumer bindings.** Deleting a `Provider` deletes its workspace and the APIExport's identity, which kcp can't re-pair with existing bindings.
- **Recovery runbooks**: for Provider Administrators, importing and re-pinning `Connection`s with `kubectl` (07 SDK, "Administration", has the basics); for PM Administrators, editing a `ProviderEndpointRegistration`'s Base URL and Server Keys. When and how, step by step.
- **Periodic Connection Secret rotation, if compliance requires it.** Not needed for security: everything harmful someone could do with a stolen Connection Secret is either detected and undone (a pushed bundle) or impossible (changing the Connection Secret), so rotation would only shorten how long they could read status. It's a small addition to the existing Secret Change: a last-change timestamp in the `ProviderEndpointRegistration`, a check on each health poll against a configured interval (zero means off), and due times spread by `connectionID`. The code carries a comment at the list of Secret Change triggers saying exactly this: what would change, why it might be wanted (compliance), and why it isn't needed now.
- **Horizontal scale for Providers**: an opt-in SDK mode with owner replicas. Each Connection gets an owner replica (rendezvous hashing over live replicas, or a Lease per Connection); any replica accepts an upgrade and forwards the raw bytes, in-cluster, to the owner, which terminates the Inner TLS, so forwarding replicas see only ciphertext. Control Sessions are forwarded the same way. Each replica runs controllers only for the Connections it owns; when Connections move on scaling, the old owner sends go-away and the Connector reconnects. Vendors who opt in accept controllers sharded by Connection (reconcile logic must not assume one replica sees all Connections), pod-to-pod connectivity in their namespace, RBAC for Leases or EndpointSlices, and relists when Connections move. No PM-side work: the protocol already makes no assumption about which replica a Session reaches.
- **An Edge self-test for Providers**: an optional echo endpoint in the SDK's Connection API server, enabled by a test secret, and a command Providers run from outside their network against their Base URL. It checks that upgrades succeed for non-browser clients, that 32 KB binary messages pass unchanged, and how the Edge treats idle connections and connection age. Until then, the quickstart's checklist and `platform-mesh-operator`'s Edge diagnosis cover it.
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

## Outside this plan

- **Profile validation** in `platform-mesh-operator`: refuse a `providerconnections` shard list that doesn't match `infra`'s.
- **Alerting**: PM has no alerting pipeline; the plan ships metrics, `ServiceMonitor`s and documented alert rules.
- **Narrowing the Providers Export's `Secret`s claim** with a label selector in PM's bindings, a PM-wide hardening.
- **Retention of the Connector's request log**, PM's audit trail for Provider traffic.
- **Proposing `ServeProxy` upstream** into the virtual workspace framework, instead of the Connector's copy.
- **Possible kcp issue**: impersonation on the APIExport virtual workspace (v5 notes, observation 5). Confirm with an e2e test before reporting it.
