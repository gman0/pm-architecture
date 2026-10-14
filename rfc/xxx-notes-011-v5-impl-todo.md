# v5 implementation plan: follow-ups

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). Work that the plan doesn't depend on and that can be done separately, before, alongside or after it. Each item says what and why in brief; the details are worked out when it's picked up.

## Cleanups in existing code

- **Rename the local Provider subroutines** in platform-mesh-operator (`ProviderWorkspaceSubroutine`, `ScopedKubeconfigSubroutine`) to names that read well as condition types, consistent with the remote Providers' subroutines (`Workspace`, `ConnectionBinding`, `ConnectorAccess`, `Registered`, `Connected`). The subroutine names are what owners see as condition types.
- **ManagedProviders and `Degraded`**: decide how a ManagedProvider reacts when its `Provider` goes from `Ready` to `Degraded`. Today its wait step only compares `phase` with `Ready`.
- **Fix the `Provider`'s `KubeconfigSecret` printer column**: it points at `.status.kubeconfigSecretRef.name`, while the field is `providerKubeconfigSecretRef`.
- **Update the types in `platform-mesh/apis`** to the plan: the `ProviderOnboardingRequest` type still has the `Rebinding` phase, `connectionSecretRef` and no `VerifyingControl`; `Provider`'s condition constants still list `SessionsConnected`, `KcpReachable`, `SecretCurrent` and `Rebinding`; the doc comment on `status.connection` still mentions rebinding.

## Larger follow-ups

- **Split the `Provider` part of platform-mesh-operator into its own operator**, taking the connection handling with it. Until then, code that talks to Providers on the internet runs in PM's most privileged component; the split ends that.
- **Deletion protection for `Provider`s with consumer bindings.** Deleting a `Provider` deletes its workspace and the APIExport's identity, which kcp can't re-pair with existing bindings.
- **Recovery tooling**: the SDK's import and re-pin commands, and the PM-side administrator input for a connection's base URL and server keys. The design supports them; the tools are missing.
- **Periodic connection secret rotation, if compliance requires it.** Not needed for security: everything harmful someone could do with a stolen secret is either detected and undone (a pushed bundle) or impossible (changing the secret), so rotation would only shorten how long they could read status. It's a small addition to the existing secret change: a last-change timestamp in the connection Secret, a check in the `Connected` subroutine against a configured interval (zero means off), and due times spread by `connectionID`. The code carries a comment at the list of secret-change triggers saying exactly this: what would change, why it might be wanted (compliance), and why it isn't needed now.
- **Owner replicas for Providers**: an opt-in SDK mode for horizontal scale, needing no PM-side work.
- **Per-replica connector certificates** with an external issuer, so no private key is mounted into connectors.

## Outside this plan

- **Profile validation** in platform-mesh-operator: refuse a `providerconnections` shard list that doesn't match `infra`'s.
- **Alerting**: PM has no alerting pipeline; the plan ships metrics, `ServiceMonitor`s and documented alert rules.
- **Moving to platform-mesh-deployer**, which needs per-shard hosts, per-instance client certificates and rights outside the module workspace first.
- **Narrowing the providers export's Secrets claim** with a label selector in PM's bindings, a PM-wide hardening.
- **Audit policy and retention** for connector traffic.
- **Proposing `ServeProxy` upstream** into the virtual workspace framework, instead of the connector's copy.
- **Possible kcp issue**: impersonation on the APIExport virtual workspace (v5 notes, observation 5). Confirm with an e2e test before reporting it.
