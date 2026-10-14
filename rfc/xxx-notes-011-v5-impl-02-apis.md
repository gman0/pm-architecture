# v5 implementation plan: APIs

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). The kcp-side types, exports and policies. Types for `providers.platform-mesh.io` already exist in `platform-mesh/apis/providers/v1alpha1`; this document lists what changes there and what's new.

## Principles

- **Two exports, two audiences.** The providers export faces owners (requests, `Provider` objects, token Secrets). The connection export faces provider workspaces (records, endpoint slices). platform-mesh-operator consumes both ([03 Connection handling](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)); connectors only the second.
- **Owner-writable data is input, never a link** (overview, principle 4).
- **PM-written data in provider workspaces is protected by kcp**, not by convention.

## providers.platform-mesh.io (existing, owner-facing)

### ProviderOnboardingRequest

Exists today (`provideronboardingrequest_types.go`), with a CRD in platform-mesh-operator, but **not yet served by the providers APIExport**, and still shaped for an earlier design (updating it is on the [follow-ups list](./xxx-notes-011-v5-impl-todo.md)). Changes:
- **Add `provideronboardingrequests` to the providers export** and its `APIResourceSchema` ([09 Deployment, "kcp bootstrap"](./xxx-notes-011-v5-impl-09-deployment.md)).
- **Spec**: the token Secret reference and the `Provider` to create or reuse (name and `spec.remote`). The owner doesn't choose where the connection secret goes: it never enters the owner's workspace (see "The connection Secret").
- **Status**: phase (`Pending → Registering → Registered`, or `Failed` only before registration succeeds), `providerRef`, `connectionID`, and a message pointing to the `Provider` once `Registered`.
- **A request is one action**: spending a single-use token to register the `Provider`'s connection. It's done once `Registered` or `Failed`; everything after registration (the first control call, the record, sessions, kcp) is reported on the `Provider` ([06 Provider controller, "Conditions and phase"](./xxx-notes-011-v5-impl-06-provider-controller.md)). Onboarding again needs a new request.
- **For an already registered `Provider`**, a request is accepted only when the Provider-side reports its connection unknown; it then registers a new connection, with a new `connectionID`, on the same workspace. It's meant for switching Providers or a connection the Provider won't restore ([03 Connection handling, "Onboarding requests"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)).
- **Who may create them**: the owner, i.e. whoever manages `Provider`s in that workspace. platform-mesh-operator acts with its own rights, so `create` on `provideronboardingrequests` is equivalent to managing `Provider`s there; PM ships one default ClusterRole that grants both together, and the creator's identity isn't checked.
- **One active request per `Provider`**: the oldest unfinished request wins; newer ones fail at once, naming the active one.
- **Spent tokens are removed**: once registration has succeeded and its result is persisted, the token Secret's content is deleted. This removes a spent token from the owner's workspace, where backups, dumps and GitOps repositories could otherwise keep it. If GitOps re-applies the Secret, the restored token is spent and refused by the Provider-side; the request is already done and doesn't act on it again.

### Provider

- **`spec.remote`** marks a remote Provider; `caBundle` optionally pins a private CA for the base URL. A CEL rule rejects `providerKubeconfigSecret` and `hostOverride` on remote Providers: they only apply to local ones ([06 Provider controller, "Spec validation"](./xxx-notes-011-v5-impl-06-provider-controller.md)).
- **One writer**: platform-mesh-operator's Provider controller writes the whole status, for both kinds ([06 Provider controller](./xxx-notes-011-v5-impl-06-provider-controller.md)).
- **Conditions**: one per subroutine, plus the aggregate `Ready`. For remote Providers: `Workspace`, `ConnectionBinding`, `ConnectorAccess`, `Registered`, `Connected`, `Ready`, and the warning `ProtocolOutdated`. Specific causes are in the messages, which start with a stable label. Aggregate only; no shard names.
- **`phase`**, for both kinds: `Pending → Provisioning → Ready ⇄ Degraded`, and `Deleting`, computed from the conditions; `Degraded` only for a `Provider` that was `Ready` before (`status.lastReadyTime`).
- **`status.connection`**: `url` and `connectionID` in use, for the owner's information; never used as a link.
- **Annotation `providers.platform-mesh.io/rotate-connection-secret`**: any new value makes platform-mesh-operator change the connection's secret once ([03 Connection handling, "Secret changes"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)).

### ProviderConnection (remove)

`providerconnection_types.go` in `providers.platform-mesh.io` is replaced by `ProviderEndpoint` in its own group (below). It isn't served yet, so removal has no migration.

### The connection Secret

**In platform-mesh-operator's namespace in PM's runtime cluster, not in kcp.** One Secret per connection, named after the provider workspace's logical cluster (e.g. `connection-<clusterName>`), so it exists before registration returns a `connectionID`:

| Key | Content |
|---|---|
| `secret` | the current connection secret (`pmc1_…`) |
| `pendingSecret` | only while the secret changes ([03 Connection handling, "Secret changes"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)) |
| `connectionID`, `serverKeys` | the registration's result, persisted together in one write before anything else happens |
| `rotationRequest` | the last value of the rotation annotation acted on |

- **Read and written only by platform-mesh-operator.** Neither owners nor Providers can reach it. In an owner's workspace, every Provider with an accepted Secrets claim there could read it, since PM's bindings accept that claim with `matchAll`; and an owner could delete it.
- **Deleted at offboarding** ([03 Connection handling, "Offboarding"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)); there's no owner reference across kcp and the runtime cluster.
- **Not a source of truth for links**: `connectionID` and `url` come from the `ProviderEndpoint` once it exists.
- **The server keys here** are the trust anchors registration returned, kept until the record is written; connectors read them from the `ProviderEndpoint`. They never live in owner-writable status.
- **Lost Secrets are repaired** by changing the secret through the record; nobody has to act.

## connections.platform-mesh.io (new, provider-workspace-facing)

### ProviderEndpoint

One per provider workspace, cluster-scoped, with a fixed name (e.g. `default`), written only by platform-mesh-operator:

| Field | Purpose |
|---|---|
| `connectionID` | Which Provider-side connection this workspace belongs to |
| `url` | The Provider's base URL that connectors dial (validated, `https` only) |
| `serviceAccount` | Namespace and name of the SA connectors mint tokens for |
| `serverKeys` | The Provider-side's pinned server public keys, for the inner TLS |
| `pendingSecretHash` | The hash of the connection secret platform-mesh-operator is switching to; empty otherwise. The Provider-side adopts it ([01 Protocol, "Changing the secret"](./xxx-notes-011-v5-impl-01-protocol.md)) |

- **No `clusterName`**: the object's logical cluster is the provider workspace.
- **No status**: connectors write nothing; health lives in metrics and in platform-mesh-operator's view.
- **Validation** in the schema: `https` URLs only, bounded list sizes, key and hash format. platform-mesh-operator's egress checks apply on top ([03 Connection handling, "Egress"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)).
- **Readable by the Provider** in its own workspace. Nothing in it is secret: `pendingSecretHash` is the hash of a 256-bit secret.
- **The Provider-side's only authenticated source for its secret.** It reads the record over its sessions, so the record's protection (below) is what keeps anyone but PM from changing a connection's secret.

### Connection export

`APIExport connections.platform-mesh.io` in **`:root:providers:system`**:
- **Resources**: `providerendpoints`.
- **Permission claim**: `apis.kcp.io/apiexportendpointslices`, verbs `get`, `list`, `watch`. The only claim. Connectors use it to see which shards have consumers ([05 Connector, "Which sessions"](./xxx-notes-011-v5-impl-05-connector.md)).
- **`maximalPermissionPolicy.local`**, with RBAC in `:root:providers:system`:
  - `get`, `list`, `watch` on `providerendpoints` for the group `apis.kcp.io:binding:system:authenticated`;
  - writes (including `deletecollection` and `/status` if ever added) only for `apis.kcp.io:binding:<operator identity>`.
- **Endpoint slice**: one `APIExportEndpointSlice` for the export, consumed by platform-mesh-operator and every connector.
- **Content access**: `apiexports/content` on this export for platform-mesh-operator (read and write) and the connectors (read only). Nobody else.

### The binding in provider workspaces

- **Created by the Provider controller** when it creates the workspace, with a **fixed name** (e.g. `connections.platform-mesh.io`), accepting the one permission claim ([06 Provider controller, "Connection export binding"](./xxx-notes-011-v5-impl-06-provider-controller.md)).
- **Protected by the connector's policy**: the Provider can't update or delete it through a session ([05 Connector, "Authorization and policy"](./xxx-notes-011-v5-impl-05-connector.md)). If it disappears anyway, the Provider controller recreates it and platform-mesh-operator rewrites the record.

## Decided

- **Request authorization**: owners create requests; a default ClusterRole grants requests and `Provider`s together; no identity checks in platform-mesh-operator.
- **Concurrent requests**: the oldest wins.
- **Connection Secrets** live in platform-mesh-operator's namespace, not in owners' workspaces.
- **Version** of the new group: `v1alpha1`. The protocol coupling is carried by the protocol version ([01 Protocol, "Versioning"](./xxx-notes-011-v5-impl-01-protocol.md)), not the API version.
- **`ProviderEndpoint` scope**: cluster-scoped with a fixed name; the integration tests confirm platform-mesh-operator can write it through the APIExport virtual workspace ([10 Testing, "Integration tests against kcp"](./xxx-notes-011-v5-impl-10-testing.md)).
