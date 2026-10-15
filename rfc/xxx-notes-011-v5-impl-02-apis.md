# v5 implementation plan: APIs

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). The kcp-side types, exports and policies, and the one type in PM's runtime cluster that holds each Connection's state. Types for `providers.platform-mesh.io` already exist in `platform-mesh/apis/providers/v1alpha1`; this document lists what changes there and what's new.

## Principles

- **Two exports, two audiences.** The Providers Export faces Owners (requests, `Provider` objects, token `Secret`s). The Connection Export faces Provider Workspaces (`ProviderEndpoint`s, endpoint slices). `platform-mesh-operator` consumes both ([03 Operator](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)); Connectors only the second.
- **Owner-writable data is input, never a link** (overview, principle 4).
- **PM-written data in Provider Workspaces is protected by kcp**, not by convention.

## `providers.platform-mesh.io`, facing Owners

### ProviderOnboardingRequest

Defined in `provideronboardingrequest_types.go` (bringing the type in line with this plan is on the [follow-ups list](./xxx-notes-011-v5-impl-todo.md)):
- **Served by the Providers Export**, as `provideronboardingrequests` and its `APIResourceSchema` ([09 Deployment, "kcp bootstrap"](./xxx-notes-011-v5-impl-09-deployment.md)).
- **Spec**: a reference to an existing remote `Provider` and to the token `Secret`. The Owner creates the `Provider` (with `spec.remote`) and the request, in either order; the request waits while the `Provider` doesn't exist. The Owner doesn't choose where the Connection Secret goes: it never enters the Owner's workspace (see "ProviderEndpointRegistration").

  ```yaml
  kind: ProviderOnboardingRequest
  spec:
    providerRef: { name: my-provider }
    tokenSecretRef: { name: my-provider-token, key: token }
  ```
- **Status**: phase (`Pending → Registering → Registered`, or `Failed` only before Registration succeeds), `connectionID`, and a message pointing to the `Provider` once `Registered`.
- **A request is one action**: spending a single-use token to register the `Provider`'s Connection. It's done once `Registered` or `Failed`; everything after Registration (the first Control Call, the `ProviderEndpoint`, Sessions, kcp) is reported on the `Provider` ([03 Operator, "Conditions and phase"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)). Onboarding again needs a new request.
- **For an already registered `Provider`**, a request is accepted only when the Provider-side reports its Connection unknown; it then registers a new Connection, with a new `connectionID`, on the same workspace. It's meant for switching Providers or a Connection the Provider won't restore ([03 Operator, "Onboarding requests"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
- **Who may create them**: the Owner, i.e. whoever manages `Provider`s in that workspace. `platform-mesh-operator` acts with its own rights, so `create` on `provideronboardingrequests` is equivalent to managing `Provider`s there; PM ships one default ClusterRole that grants both together, and the creator's identity isn't checked.
- **One active request per `Provider`**: the oldest unfinished request wins; newer ones fail at once, naming the active one.
- **Spent tokens are removed**: once Registration has succeeded and its result is persisted, the token `Secret`'s content is deleted. This removes a spent token from the Owner's workspace, where backups, dumps and GitOps repositories could otherwise keep it. If GitOps re-applies the `Secret`, the restored token is spent and refused by the Provider-side; the request is already done and doesn't act on it again.

### Provider

- **`spec.remote`** marks a Remote Provider; `caBundle` optionally pins a private CA for the Base URL. It verifies the outer TLS of every connection to the Base URL: Registration, Control Sessions and Sessions; without it, web PKI. `platform-mesh-operator` copies it into the `ProviderEndpoint`. It's input, never a link: a wrong value only stops the Owner's own Connection. **Rotating it is out-of-band**: the Provider tells its Owners about a new CA, Owners add it next to the old one before the Provider switches its server certificate, and remove the old one afterwards. New Sessions use the updated bundle; open Sessions continue. A missed update shows `EdgeTLS` on the `Provider` ([03 Operator, "Edge diagnosis"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)), and updating the field recovers it. A CEL rule rejects `providerKubeconfigSecret` and `hostOverride` on Remote Providers: they only apply to local ones ([03 Operator, "Spec validation"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
- **Created by the Owner**, never by `platform-mesh-operator`. A `Provider` left behind by a failed Onboarding waits for the next request.
- **Admin SA**: for Remote Providers, `platform-mesh-operator` creates it as `provider` in namespace `default` in every Provider Workspace; Connectors know the name as a constant. Local Providers keep `<name>-provider` (RFC 006).
- **One writer**: `platform-mesh-operator` writes the whole status, for both kinds ([03 Operator](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
- **Conditions**: one per reconcile step, plus the aggregate `Ready`. For Remote Providers: `Workspace`, `ConnectionBinding`, `ConnectorAccess`, `Registered`, `Connected`, `Ready`, and the warning `ProtocolOutdated`. Specific causes are in the messages, which start with a stable label. Aggregate only; no shard names.
- **`phase`**, for both kinds: `Pending → Provisioning → Ready ⇄ Degraded`, and `Deleting`, computed from the conditions; `Degraded` only for a `Provider` that was `Ready` before (`status.lastReadyTime`).
- **`status.connection`**: `url` and `connectionID` in use, for the Owner's information; never used as a link.
- **Annotation `providers.platform-mesh.io/rotate-connection-secret`**: any new value makes `platform-mesh-operator` change the Connection Secret once ([03 Operator, "Secret Changes"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).

### ProviderConnection (remove)

`providerconnection_types.go` in `providers.platform-mesh.io` is replaced by `ProviderEndpoint` in its own group (below). It isn't served yet, so removal has no migration.

## `providers.platform-mesh.io` in PM's runtime cluster

### ProviderEndpointRegistration

**In `platform-mesh-operator`'s namespace in PM's runtime cluster, not in kcp**, next to `ManagedProvider`. One per remote Connection, named after the Provider Workspace's logical cluster, so it exists before Registration returns a `connectionID`:

```yaml
apiVersion: providers.platform-mesh.io/v1alpha1
kind: ProviderEndpointRegistration
metadata:
  name: <clusterName>                        # the provider workspace
  namespace: <platform-mesh-operator namespace>
spec:
  connectionID: "..."                        # from registration
  url: https://provider.example.com/pm       # the base URL from the token
  serverKeys: [ "<PEM SPKI>", ... ]          # from registration, then updated from GET
  rotationRequest: ""                        # the last value of the rotation annotation acted on
```

- **Authoritative**, together with its `Secret` (below): it holds everything PM knows about the Connection that can't be recomputed. The `ProviderEndpoint` is written from them (see "ProviderEndpoint"), and every change (a new or retired Server Key, administrator input for recovery) goes into the `ProviderEndpointRegistration` first.
- **Only `spec`, no `status`**: its content can't be recomputed, so it isn't status. Health is reported on the `Provider`.
- **`connectionID`, `url` and `serverKeys` are persisted in one write**, as soon as Registration returns ([03 Operator, "Registration"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)). The token's data is deleted afterwards, so this is where the Base URL survives.
- **Validation** in the schema: `https` URLs only, key format, bounded list sizes.
- **Printer columns**: `connectionID` and `url`, so `kubectl get` lists every remote Connection without access to `Secret`s.
- **Written by `platform-mesh-operator`.** PM Administrators may edit `url` and `serverKeys` for recovery, for example after a Provider rebuilt elsewhere lost its keys ([03 Operator, "Writing ProviderEndpoints"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)). Neither Owners nor Providers can reach it.
- **The Server Keys here** are the trust anchors Registration returned; Connectors read their copy from the `ProviderEndpoint`. They never live in Owner-writable status.
- **Deleted at offboarding** ([03 Operator, "Deletion"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)); there's no owner reference across kcp and the runtime cluster.
- **Lost `ProviderEndpointRegistration`s are rebuilt** without anyone acting: `connectionID`, `url` and `serverKeys` come from the `ProviderEndpoint`, the only time the `ProviderEndpoint` is read as input, and the Connection Secret is replaced through a Secret Change, since the `Secret` goes with it. Both copies are written only by `platform-mesh-operator`.

### Its `Secret`

`<clusterName>-secret`, in the same namespace, with an owner reference to the `ProviderEndpointRegistration`, so deleting one deletes the other. It holds only the credentials:

| Key | Content |
|---|---|
| `secret` | the current Connection Secret (`pmc1_…`) |
| `pendingSecret` | only while the Connection Secret changes ([03 Operator, "Secret Changes"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)) |

- **Read and written only by `platform-mesh-operator`.** In an Owner's workspace, every Provider with an accepted `Secret`s claim there could read it, since PM's bindings accept that claim with `matchAll`; and an Owner could delete it.
- **Lost `Secret`s are repaired** through a Secret Change; nobody has to act.

## `connections.platform-mesh.io` (new), facing Provider Workspaces

### ProviderEndpoint

One per Provider Workspace, cluster-scoped, with a fixed name (e.g. `default`), written only by `platform-mesh-operator`, from the `ProviderEndpointRegistration`, its `Secret` and the `Provider`'s `caBundle`:

| Field | Purpose |
|---|---|
| `connectionID` | Which Connection this workspace belongs to |
| `url` | The Provider's Base URL that Connectors dial (validated, `https` only) |
| `serverKeys` | The Provider-side's pinned server public keys, for the Inner TLS |
| `caBundle` | The `Provider`'s `spec.remote.caBundle`, if set: the private CA that Connectors verify the Base URL's outer TLS with, instead of web PKI |
| `pendingSecretHash` | The hash of the `Secret`'s `pendingSecret`, the Connection Secret `platform-mesh-operator` is switching to; empty otherwise. The Provider-side adopts it ([01 Protocol, "Changing the Connection Secret"](./xxx-notes-011-v5-impl-01-protocol.md)) |

- **No `clusterName`**: the object's logical cluster is the Provider Workspace.
- **No SA reference**: Connectors act as the fixed admin SA `default/provider` (see "Provider").
- **No status**: Connectors write nothing; health lives in metrics and in `platform-mesh-operator`'s view.
- **Validation** in the schema: `https` URLs only, bounded list sizes, key and hash format. `platform-mesh-operator`'s egress checks apply on top ([03 Operator, "Egress"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
- **Readable by the Provider** in its own workspace. Nothing in it is secret: `pendingSecretHash` is the hash of a 256-bit secret.
- **The Provider-side's only authenticated source for its Connection Secret.** It reads the `ProviderEndpoint` over its Sessions, so the `ProviderEndpoint`'s protection (below) is what keeps anyone but PM from changing a Connection Secret.

### Connection Export

`APIExport connections.platform-mesh.io` in **`:root:providers:system`**:
- **Resources**: `providerendpoints`.
- **Permission claim**: `apis.kcp.io/apiexportendpointslices`, verbs `get`, `list`, `watch`. The only claim. Connectors use it to see which shards have consumers ([05 Connector, "Which Sessions"](./xxx-notes-011-v5-impl-05-connector.md)).
- **`maximalPermissionPolicy.local`**, with RBAC in `:root:providers:system`:
  - `get`, `list`, `watch` on `providerendpoints` for the group `apis.kcp.io:binding:system:authenticated`;
  - writes (including `deletecollection` and `/status` if ever added) only for `apis.kcp.io:binding:<operator identity>`.
- **Endpoint slice**: one `APIExportEndpointSlice` for the export, consumed by `platform-mesh-operator` and every Connector.
- **Content access**: `apiexports/content` on this export for `platform-mesh-operator` (read and write) and the Connectors (read only). Nobody else.

### The binding in Provider Workspaces

- **Created by `platform-mesh-operator`** with the workspace, with a **fixed name** (e.g. `connections.platform-mesh.io`), accepting the one permission claim ([03 Operator, "Connection Export binding"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
- **Protected by the Connector's policy**: the Provider can't update or delete it through a Session ([05 Connector, "Authorization and policy"](./xxx-notes-011-v5-impl-05-connector.md)). If it disappears anyway, `platform-mesh-operator` recreates it and rewrites the `ProviderEndpoint`.

## Decided

- **Request authorization**: Owners create requests; a default ClusterRole grants requests and `Provider`s together; no identity checks in `platform-mesh-operator`.
- **Concurrent requests**: the oldest wins.
- **Requests reference an existing `Provider`**; the Owner creates it.
- **Connection state** lives in `platform-mesh-operator`'s namespace, not in Owners' workspaces: a `ProviderEndpointRegistration` per Connection, authoritative, with a `Secret` holding only the credentials; `ProviderEndpoint`s are written from them.
- **The `caBundle`** stays a `Provider` setting, copied into the `ProviderEndpoint`; rotating it is out-of-band.
- **Remote Providers' admin SA** has a fixed name; the `ProviderEndpoint` doesn't carry it.
- **Version** of the new group: `v1alpha1`. The protocol coupling is carried by the protocol version ([01 Protocol, "Versioning"](./xxx-notes-011-v5-impl-01-protocol.md)), not the API version.
- **`ProviderEndpoint` scope**: cluster-scoped with a fixed name; every E2E Onboarding confirms `platform-mesh-operator` can write it through the APIExport virtual workspace ([10 Testing, "Onboarding"](./xxx-notes-011-v5-impl-10-testing.md)).
