# v5 implementation plan: follow-ups

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). Work that the plan doesn't depend on and that can be done separately, before, alongside or after it. Each item says what and why in brief; the details are worked out when it's picked up.

## Cleanups in existing code

- **Rename the local Provider steps in platform-mesh-operator's code** (`ProviderWorkspaceSubroutine`, `ScopedKubeconfigSubroutine`) to `Workspace` and `Kubeconfig`. Their names become the condition types owners see, consistent with the remote steps (`Workspace`, `ConnectionBinding`, `ConnectorAccess`, `Registered`, `Connected`).
- **ManagedProviders and `Degraded`**: decide how a ManagedProvider reacts when its `Provider` goes from `Ready` to `Degraded`. Its wait step compares `phase` with `Ready`.
- **Fix the `Provider`'s `KubeconfigSecret` printer column**: it points at `.status.kubeconfigSecretRef.name`, while the field is `providerKubeconfigSecretRef`.
- **Update the types in `platform-mesh/apis`** to the plan: the `ProviderOnboardingRequest` type still has the `Rebinding` phase, `connectionSecretRef` and no `VerifyingControl`; `Provider`'s condition constants still list `SessionsConnected`, `KcpReachable`, `SecretCurrent` and `Rebinding`; the doc comment on `status.connection` still mentions rebinding. The new `ProviderEndpointRegistration` kind goes into platform-mesh-operator's `api/providers/v1alpha1`, next to `ManagedProvider`.

## Larger follow-ups

- **Split the `Provider` part of platform-mesh-operator into its own operator**, taking the remote Providers' connection code with it. Until then, code that talks to Providers on the internet runs in PM's most privileged component; the split ends that.
- **Deletion protection for `Provider`s with consumer bindings.** Deleting a `Provider` deletes its workspace and the APIExport's identity, which kcp can't re-pair with existing bindings.
- **Recovery runbooks**: for Provider administrators, importing and re-pinning `Connection`s with `kubectl` (07 SDK, "Administration", has the basics); for PM administrators, editing a `ProviderEndpointRegistration`'s base URL and server keys. When and how, step by step.
- **Periodic connection secret rotation, if compliance requires it.** Not needed for security: everything harmful someone could do with a stolen secret is either detected and undone (a pushed bundle) or impossible (changing the secret), so rotation would only shorten how long they could read status. It's a small addition to the existing secret change: a last-change timestamp in the `ProviderEndpointRegistration`, a check on each health poll against a configured interval (zero means off), and due times spread by `connectionID`. The code carries a comment at the list of secret-change triggers saying exactly this: what would change, why it might be wanted (compliance), and why it isn't needed now.
- **Horizontal scale for Providers**: an opt-in SDK mode with owner replicas. Each connection gets an owner replica (rendezvous hashing over live replicas, or a Lease per connection); any replica accepts an upgrade and forwards the raw bytes, in-cluster, to the owner, which terminates the inner TLS, so forwarding replicas see only ciphertext. Control sessions are forwarded the same way. Each replica runs controllers only for the connections it owns; when connections move on scaling, the old owner sends go-away and the connector reconnects. Vendors who opt in accept controllers sharded by connection (reconcile logic must not assume one replica sees all connections), pod-to-pod connectivity in their namespace, RBAC for Leases or EndpointSlices, and relists when connections move. No PM-side work: the protocol already makes no assumption about which replica a session reaches.
- **An edge self-test for Providers**: an optional echo endpoint in the SDK's connection server, enabled by a test secret, and a command Providers run from outside their network against their base URL. It checks that upgrades succeed for non-browser clients, that 32 KB binary messages pass unchanged, and how the edge treats idle connections and connection age. Until then, the quickstart's checklist and platform-mesh-operator's edge diagnosis cover it.
- **Periodic server key rotation, if compliance requires it**: the SDK creating a new `ServerKey` on a schedule and setting `retireAt` on the old one, the same steps an administrator takes today.
- **Suspending a connection without deleting it**, if Providers need it: a field on the `Connection` that takes it out of the registry (its controllers disengage, sessions and upkeep continue), reported in the status `GET` so platform-mesh-operator can show it to the owner.
- **Per-replica connector certificates** with an external issuer, so no private key is mounted into connectors.
- **Iterate on the quickstart's portal** once the RFC is implemented: real login, owner notifications from `Connection` Events, more connection detail.
- **In-band rotation of a private outer CA**, if Providers with many owners on private CAs need it: the Provider-side announces a new CA in the status `GET` (protected by the inner TLS), and platform-mesh-operator adds it to the `ProviderEndpoint`'s `caBundle`, so owners don't have to update theirs. Until then, rotation is out-of-band.

## Outside this plan

- **Profile validation** in platform-mesh-operator: refuse a `providerconnections` shard list that doesn't match `infra`'s.
- **Alerting**: PM has no alerting pipeline; the plan ships metrics, `ServiceMonitor`s and documented alert rules.
- **Moving to platform-mesh-deployer**, which needs per-shard hosts, per-instance client certificates and rights outside the module workspace first.
- **Narrowing the providers export's Secrets claim** with a label selector in PM's bindings, a PM-wide hardening.
- **Audit policy and retention** for connector traffic.
- **Proposing `ServeProxy` upstream** into the virtual workspace framework, instead of the connector's copy.
- **Possible kcp issue**: impersonation on the APIExport virtual workspace (v5 notes, observation 5). Confirm with an e2e test before reporting it.
