# v5 implementation plan: APIs

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). The kcp-side types, exports and policies. Types for `providers.platform-mesh.io` already exist in `platform-mesh/apis/providers/v1alpha1`; this document lists what changes there and what's new.

## Principles

- **Two exports, two audiences.** The providers export faces owners (requests, `Provider` objects, token Secrets). The connection export faces provider workspaces (`ProviderEndpoint`s, endpoint slices). platform-mesh-operator consumes both ([03 Operator](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)); connectors only the second.
- **Owner-writable data is input, never a link** (overview, principle 4).
- **PM-written data in provider workspaces is protected by kcp**, not by convention.

## providers.platform-mesh.io (owner-facing)

### ProviderOnboardingRequest

Defined in `provideronboardingrequest_types.go` (bringing the type in line with this plan is on the [follow-ups list](./xxx-notes-011-v5-impl-todo.md)):
- **Served by the providers export**, as `provideronboardingrequests` and its `APIResourceSchema` ([09 Deployment, "kcp bootstrap"](./xxx-notes-011-v5-impl-09-deployment.md)).
- **Spec**: a reference to an existing remote `Provider` and to the token Secret. The owner creates the `Provider` (with `spec.remote`) and the request, in either order; the request waits while the `Provider` doesn't exist. The owner doesn't choose where the connection secret goes: it never enters the owner's workspace (see "The connection Secret").

  ```yaml
  kind: ProviderOnboardingRequest
  spec:
    providerRef: { name: my-provider }
    tokenSecretRef: { name: my-provider-token, key: token }
  ```
- **Status**: phase (`Pending → Registering → Registered`, or `Failed` only before registration succeeds), `connectionID`, and a message pointing to the `Provider` once `Registered`.
- **A request is one action**: spending a single-use token to register the `Provider`'s connection. It's done once `Registered` or `Failed`; everything after registration (the first control call, the `ProviderEndpoint`, sessions, kcp) is reported on the `Provider` ([03 Operator, "Conditions and phase"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)). Onboarding again needs a new request.
- **For an already registered `Provider`**, a request is accepted only when the Provider-side reports its connection unknown; it then registers a new connection, with a new `connectionID`, on the same workspace. It's meant for switching Providers or a connection the Provider won't restore ([03 Operator, "Onboarding requests"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
- **Who may create them**: the owner, i.e. whoever manages `Provider`s in that workspace. platform-mesh-operator acts with its own rights, so `create` on `provideronboardingrequests` is equivalent to managing `Provider`s there; PM ships one default ClusterRole that grants both together, and the creator's identity isn't checked.
- **One active request per `Provider`**: the oldest unfinished request wins; newer ones fail at once, naming the active one.
- **Spent tokens are removed**: once registration has succeeded and its result is persisted, the token Secret's content is deleted. This removes a spent token from the owner's workspace, where backups, dumps and GitOps repositories could otherwise keep it. If GitOps re-applies the Secret, the restored token is spent and refused by the Provider-side; the request is already done and doesn't act on it again.

### Provider

- **`spec.remote`** marks a remote Provider; `caBundle` optionally pins a private CA for the base URL. A CEL rule rejects `providerKubeconfigSecret` and `hostOverride` on remote Providers: they only apply to local ones ([03 Operator, "Spec validation"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
- **Created by the owner**, never by platform-mesh-operator. A `Provider` left behind by a failed onboarding waits for the next request.
- **Admin SA**: for remote Providers, platform-mesh-operator creates it as `provider` in namespace `default` in every provider workspace; connectors know the name as a constant. Local Providers keep `<name>-provider` (RFC 006).
- **One writer**: platform-mesh-operator writes the whole status, for both kinds ([03 Operator](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
- **Conditions**: one per reconcile step, plus the aggregate `Ready`. For remote Providers: `Workspace`, `ConnectionBinding`, `ConnectorAccess`, `Registered`, `Connected`, `Ready`, and the warning `ProtocolOutdated`. Specific causes are in the messages, which start with a stable label. Aggregate only; no shard names.
- **`phase`**, for both kinds: `Pending → Provisioning → Ready ⇄ Degraded`, and `Deleting`, computed from the conditions; `Degraded` only for a `Provider` that was `Ready` before (`status.lastReadyTime`).
- **`status.connection`**: `url` and `connectionID` in use, for the owner's information; never used as a link.
- **Annotation `providers.platform-mesh.io/rotate-connection-secret`**: any new value makes platform-mesh-operator change the connection's secret once ([03 Operator, "Secret changes"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).

### ProviderConnection (remove)

`providerconnection_types.go` in `providers.platform-mesh.io` is replaced by `ProviderEndpoint` in its own group (below). It isn't served yet, so removal has no migration.

### The connection Secret

**In platform-mesh-operator's namespace in PM's runtime cluster, not in kcp.** One Secret per connection, named after the provider workspace's logical cluster (e.g. `connection-<clusterName>`), so it exists before registration returns a `connectionID`:

| Key | Content |
|---|---|
| `secret` | the current connection secret (`pmc1_…`) |
| `pendingSecret` | only while the secret changes ([03 Operator, "Secret changes"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)) |
| `connectionID`, `url`, `serverKeys` | the registration's result (`url` is the base URL from the token), persisted together in one write before anything else happens |
| `rotationRequest` | the last value of the rotation annotation acted on |

- **Authoritative**: it holds everything PM knows about the connection. The `ProviderEndpoint` is written from it (see "ProviderEndpoint"), and every change (a new or retired server key, a secret change, administrator input for recovery) goes into the Secret first.
- **Read and written only by platform-mesh-operator.** Neither owners nor Providers can reach it. In an owner's workspace, every Provider with an accepted Secrets claim there could read it, since PM's bindings accept that claim with `matchAll`; and an owner could delete it.
- **Deleted at offboarding** ([03 Operator, "Deletion"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)); there's no owner reference across kcp and the runtime cluster.
- **The server keys here** are the trust anchors registration returned; connectors read their copy from the `ProviderEndpoint`. They never live in owner-writable status.
- **Lost Secrets are repaired** without anyone acting: `connectionID`, `url` and `serverKeys` are rebuilt from the `ProviderEndpoint`, the only time the `ProviderEndpoint` is read as input, and the secret is replaced through a secret change. Both copies are written only by platform-mesh-operator.

## connections.platform-mesh.io (new, provider-workspace-facing)

### ProviderEndpoint

One per provider workspace, cluster-scoped, with a fixed name (e.g. `default`), written only by platform-mesh-operator, from the connection Secret:

| Field | Purpose |
|---|---|
| `connectionID` | Which Provider-side connection this workspace belongs to |
| `url` | The Provider's base URL that connectors dial (validated, `https` only) |
| `serverKeys` | The Provider-side's pinned server public keys, for the inner TLS |
| `pendingSecretHash` | The hash of the connection secret platform-mesh-operator is switching to; empty otherwise. The Provider-side adopts it ([01 Protocol, "Changing the secret"](./xxx-notes-011-v5-impl-01-protocol.md)) |

- **No `clusterName`**: the object's logical cluster is the provider workspace.
- **No SA reference**: connectors act as the fixed admin SA `default/provider` (see "Provider").
- **No status**: connectors write nothing; health lives in metrics and in platform-mesh-operator's view.
- **Validation** in the schema: `https` URLs only, bounded list sizes, key and hash format. platform-mesh-operator's egress checks apply on top ([03 Operator, "Egress"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
- **Readable by the Provider** in its own workspace. Nothing in it is secret: `pendingSecretHash` is the hash of a 256-bit secret.
- **The Provider-side's only authenticated source for its secret.** It reads the `ProviderEndpoint` over its sessions, so the `ProviderEndpoint`'s protection (below) is what keeps anyone but PM from changing a connection's secret.

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

- **Created by platform-mesh-operator** with the workspace, with a **fixed name** (e.g. `connections.platform-mesh.io`), accepting the one permission claim ([03 Operator, "Connection export binding"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
- **Protected by the connector's policy**: the Provider can't update or delete it through a session ([05 Connector, "Authorization and policy"](./xxx-notes-011-v5-impl-05-connector.md)). If it disappears anyway, platform-mesh-operator recreates it and rewrites the `ProviderEndpoint`.

## Decided

- **Request authorization**: owners create requests; a default ClusterRole grants requests and `Provider`s together; no identity checks in platform-mesh-operator.
- **Concurrent requests**: the oldest wins.
- **Requests reference an existing `Provider`**; the owner creates it.
- **Connection Secrets** live in platform-mesh-operator's namespace, not in owners' workspaces, and are authoritative; `ProviderEndpoint`s are written from them.
- **Remote Providers' admin SA** has a fixed name; the `ProviderEndpoint` doesn't carry it.
- **Version** of the new group: `v1alpha1`. The protocol coupling is carried by the protocol version ([01 Protocol, "Versioning"](./xxx-notes-011-v5-impl-01-protocol.md)), not the API version.
- **`ProviderEndpoint` scope**: cluster-scoped with a fixed name; the integration tests confirm platform-mesh-operator can write it through the APIExport virtual workspace ([10 Testing, "Integration tests against kcp"](./xxx-notes-011-v5-impl-10-testing.md)).
