# v5 implementation plan: APIs

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). The kcp-side types, exports and policies. Types for `providers.platform-mesh.io` already exist in `platform-mesh/apis/providers/v1alpha1`; this document lists what changes there and what's new.

## Principles

- **Two exports, two audiences.** The providers export faces owners (requests, `Provider` objects, token Secrets). The connection export faces provider workspaces (records, endpoint slices). The [operator](./xxx-notes-011-v5-impl-03-providerconnections-operator.md) consumes both; connectors only the second.
- **Owner-writable data is input, never a link** (overview, principle 4).
- **PM-written data in provider workspaces is protected by kcp**, not by convention.

## providers.platform-mesh.io (existing, owner-facing)

### ProviderOnboardingRequest

Exists today (`provideronboardingrequest_types.go`), with a CRD in platform-mesh-operator, but **not yet served by the providers APIExport**. Changes:
- **Add `provideronboardingrequests` to the providers export** and its `APIResourceSchema` ([09 Deployment, "kcp bootstrap"](./xxx-notes-011-v5-impl-09-deployment.md)).
- **Spec**: the token Secret reference and the `Provider` to create or reuse (name and `spec.remote`). The owner doesn't choose where the connection secret goes: it never enters the owner's workspace (see "The connection Secret").
- **Status**: phase (`WaitingForProvider → Registering → VerifyingControl → VerifyingKcp → Ready`, with `Failed` only before registration succeeds), `providerRef`, `connectionID`, conditions with reasons for every problem the owner can act on.
- **For an already registered `Provider`**, a request is accepted only when the Provider-side reports its connection unknown; it then registers a new connection, with a new `connectionID`, on the same workspace. It's meant for switching Providers or a connection the Provider won't restore ([03 Operator, "Onboarding"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)).
- **Lifecycle**: a request is done once `Ready` or `Failed`. Re-running needs a new request. **Once registration has succeeded, a request never fails**: if readiness doesn't arrive in time, it stays in `VerifyingKcp` with a warning condition and becomes `Ready` when the problem goes away ([03 Operator, "Onboarding"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)).
- **Who may create them**: the owner, i.e. whoever manages `Provider`s in that workspace. The operator acts with its own rights, so `create` on `provideronboardingrequests` is equivalent to managing `Provider`s there; PM ships one default ClusterRole that grants both together, and the operator doesn't check the creator's identity.
- **One active request per `Provider`**: the oldest unfinished request wins; newer ones fail at once, naming the active one.
- **Spent tokens are removed**: once registration has succeeded and its result is persisted, the operator deletes the token Secret's content (see [03 Operator, "Onboarding"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)). This removes a spent token from the owner's workspace, where backups, dumps and GitOps repositories could otherwise keep it. If GitOps re-applies the Secret, the restored token is spent and refused by the Provider-side; the request is already done and doesn't act on it again.

### Provider

- **`spec.remote`** marks a remote Provider; `caBundle` optionally pins a private CA for the base URL. For remote Providers, `spec.providerKubeconfigSecret` is ignored ([06 Provider controller](./xxx-notes-011-v5-impl-06-provider-controller.md)).
- **`status.connection`**: `url` and `connectionID`, written by the operator.
- **Connection conditions**, written by the operator: `Registered` (with reasons `ConnectionUnknown` and `ConnectionFrozen` when false), `SessionsConnected`, `KcpReachable`, `SecretCurrent`, and the warnings `ProtocolOutdated` and `ConnectorCABundleTampered`. Aggregate only; no shard names.
- **Bootstrap conditions** stay with the Provider controller, plus one it adds for the operator to wait on: `ConnectionPrerequisitesReady` (workspace, SA, binding, connector RBAC in place). The operator treats it as a hint and verifies through PM-owned objects ([03 Operator, "Finding the provider workspace"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)).

### Status ownership

The Provider controller and the operator both write `Provider.status`. They use **server-side apply with separate field managers**, each owning a disjoint set of fields and condition types. Neither ever replaces the whole status.

### ProviderConnection (remove)

`providerconnection_types.go` in `providers.platform-mesh.io` is replaced by `ProviderEndpoint` in its own group (below). It isn't served yet, so removal has no migration.

### The connection Secret

**In the operator's namespace in PM's runtime cluster, not in kcp.** One Secret per connection, named after the provider workspace's logical cluster (e.g. `connection-<clusterName>`), so it exists before registration returns a `connectionID`:

| Key | Content |
|---|---|
| `secret` | the current connection secret (`pmc1_…`) |
| `pendingSecret` | only while the secret changes ([03 Operator, "Secret changes"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)) |
| `connectionID`, `serverKeys` | the registration's result, persisted together in one write before anything else happens; the server keys are trust anchors, so they never live in owner-writable status |

- **Read and written only by the operator.** Neither owners nor Providers can reach it. In an owner's workspace, every Provider with an accepted Secrets claim there could read it, since PM's bindings accept that claim with `matchAll`; and an owner deleting it would have needed a recovery flow.
- **Deleted by the operator** at offboarding ([03 Operator, "Offboarding"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)); there's no owner reference across kcp and the runtime cluster.
- **Not a source of truth for links**: the operator takes `connectionID` and `url` from the `ProviderEndpoint`.
- **Server keys don't live here**: they're public and live in the `ProviderEndpoint`, where connectors can read them.
- **Lost Secrets are repaired** by changing the secret through the record; nobody has to act.

## connections.platform-mesh.io (new, provider-workspace-facing)

### ProviderEndpoint

One per provider workspace, cluster-scoped, with a fixed name (e.g. `default`), written only by the operator:

| Field | Purpose |
|---|---|
| `connectionID` | Which Provider-side connection this workspace belongs to |
| `url` | The Provider's base URL that connectors dial (validated, `https` only) |
| `serviceAccount` | Namespace and name of the SA connectors mint tokens for |
| `serverKeys` | The Provider-side's pinned server public keys, for the inner TLS |
| `pendingSecretHash` | The hash of the connection secret the operator is switching to; empty otherwise. The Provider-side adopts it ([01 Protocol, "Changing the secret"](./xxx-notes-011-v5-impl-01-protocol.md)) |

- **No `clusterName`**: the object's logical cluster is the provider workspace.
- **No status**: connectors write nothing; health lives in metrics and in the operator's view.
- **Validation** in the schema: `https` URLs only, bounded list sizes, key and hash format. The operator's egress checks apply on top ([03 Operator, "Egress"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)).
- **Readable by the Provider** in its own workspace. Nothing in it is secret: `pendingSecretHash` is the hash of a 256-bit secret.
- **The Provider-side's only authenticated source for its secret.** It reads the record over its sessions, so the record's protection (below) is what keeps anyone but PM from changing a connection's secret.

### Connection export

`APIExport connections.platform-mesh.io` in **`:root:providers:system`**:
- **Resources**: `providerendpoints`.
- **Permission claim**: `apis.kcp.io/apiexportendpointslices`, verbs `get`, `list`, `watch`. The only claim. Connectors use it to see which shards have consumers ([05 Connector, "Which sessions"](./xxx-notes-011-v5-impl-05-connector.md)).
- **`maximalPermissionPolicy.local`**, with RBAC in `:root:providers:system`:
  - `get`, `list`, `watch` on `providerendpoints` for the group `apis.kcp.io:binding:system:authenticated`;
  - writes (including `deletecollection` and `/status` if ever added) only for `apis.kcp.io:binding:<operator identity>`.
- **Endpoint slice**: one `APIExportEndpointSlice` for the export, consumed by the operator and every connector.
- **Content access**: `apiexports/content` on this export for the operator (read and write) and the connectors (read only). Nobody else.

### The binding in provider workspaces

- **Created by the Provider controller** when it creates the workspace, with a **fixed name** (e.g. `connections.platform-mesh.io`), accepting the one permission claim ([06 Provider controller, "Connection export binding"](./xxx-notes-011-v5-impl-06-provider-controller.md)).
- **Protected by the connector's policy**: the Provider can't update or delete it through a session ([05 Connector, "Authorization and policy"](./xxx-notes-011-v5-impl-05-connector.md)). If it disappears anyway, the Provider controller recreates it and the operator rewrites the record.

## Decided

- **Request authorization**: owners create requests; a default ClusterRole grants requests and `Provider`s together; no identity checks in the operator.
- **Concurrent requests**: the oldest wins.
- **Connection Secrets** live in the operator's namespace, not in owners' workspaces.
- **Version** of the new group: `v1alpha1`. The protocol coupling is carried by the protocol version ([01 Protocol, "Versioning"](./xxx-notes-011-v5-impl-01-protocol.md)), not the API version.
- **`ProviderEndpoint` scope**: cluster-scoped with a fixed name; the integration tests confirm the operator can write it through the APIExport virtual workspace ([10 Testing, "Integration tests against kcp"](./xxx-notes-011-v5-impl-10-testing.md)).
