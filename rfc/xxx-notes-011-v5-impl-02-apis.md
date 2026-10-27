# v5 implementation plan: APIs

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). The kcp-side types, exports and policies, and the type in PM's runtime cluster that holds each Connection's state. Every PM-side type is in one API group, `providers.platform-mesh.io`, served in kcp through two exports and in PM's runtime cluster as CRDs; each kind is served either in kcp or in PM's runtime cluster, never both, and the tables say which (`ConnectorCASet` there is described in [04 Connector CA](./xxx-notes-011-v5-impl-04-connector-ca.md)). Types for `providers.platform-mesh.io` already exist in `platform-mesh/apis/providers/v1alpha1`; this document lists what changes there and what's new.

## Principles

- **Two exports, two audiences.** The Providers Export faces Owners (requests, `Provider` objects, token `Secret`s). The Connection Export faces Provider Workspaces (`ProviderEndpoint`s, endpoint slices). `platform-mesh-operator` consumes both ([03 Operator](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)); Connectors only the second.
- **Owner-writable data is input, never a link** (overview, principle 4).
- **PM-written data in Provider Workspaces is protected by kcp**, not by convention.

## The Providers Export, facing Owners

### ProviderOnboardingRequest

Defined in `provideronboardingrequest_types.go` (bringing the type in line with this plan is on the [follow-ups list](./xxx-notes-011-v5-impl-todo.md)):
- **Served by the Providers Export**, as `provideronboardingrequests` and its `APIResourceSchema` ([09 Deployment, "kcp bootstrap"](./xxx-notes-011-v5-impl-09-deployment.md)).
- **Spec**: a reference to an existing remote `Provider`, the token `Secret`, and the token's `tokenID`. The Owner creates the `Provider` (with `spec.remote`) and the request, in either order; while the `Provider` doesn't exist, the request has the condition `ProviderNotFound`. The Owner doesn't choose where the Connection Secret goes: it never enters the Owner's workspace (see "ProviderEndpointRegistration").

  ```yaml
  kind: ProviderOnboardingRequest
  spec:
    providerRef: { name: my-provider }
    tokenSecretRef: { name: my-provider-token, key: token }
    tokenID: 7Kq2…                             # the token's ID, as in the token Secret
  ```

  **The whole spec is immutable** (a CEL rule). `platform-mesh-operator` refuses a token whose ID isn't `spec.tokenID`, so the token `Secret` can't be swapped under a request that's already running, and retries always present the same token.
- **Status**: phase (`Pending → Registering → Registered`, or `Failed` only before Registration succeeds), `connectionID`, conditions, and a message pointing to the `Provider` once `Registered`. A token the Provider-side refuses (unknown, expired, spent, or its Server Key unusable) fails the request with `TokenRefused`.
- **A request is one action**: spending a single-use token to register the `Provider`'s Connection. It's done once `Registered` or `Failed`; everything after Registration (the first Control Call, the `ProviderEndpoint`, Sessions, kcp) is reported on the `Provider` ([03 Operator, "Conditions and phase"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)). Onboarding again needs a new request.
- **For an already registered `Provider`**, a request is accepted only when the Provider-side reports its Connection unknown, inside the Inner TLS ([01 Protocol, "Errors and retries"](./xxx-notes-011-v5-impl-01-protocol.md)), or after the Owner abandoned the Connection (see "Provider"); it then registers a new Connection, with a new `connectionID`, on the same workspace. It's meant for switching Providers or a Connection the Provider won't restore ([03 Operator, "Onboarding requests"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
- **Who may create them**: the Owner, i.e. whoever manages `Provider`s in that workspace. `platform-mesh-operator` acts with its own rights, so `create` on `provideronboardingrequests` is equivalent to managing `Provider`s there; PM ships one default ClusterRole that grants both together, and the creator's identity isn't checked.
- **One active request per `Provider`**: the oldest unfinished request wins; newer ones fail at once, naming the active one.
- **Spent tokens are left in place**: `platform-mesh-operator` never writes to the token `Secret`. A spent token is refused by the Provider-side, so it's useless where backups, dumps or GitOps repositories keep it, and the Owner may delete the `Secret` once the request is `Registered`. Removing it would fight GitOps and secret stores, which re-create it ([03 Operator, "Declined"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).

### Provider

- **`spec.remote`** marks a Remote Provider; `caBundle` optionally pins a private CA for the Base URL. It verifies the outer TLS of every connection to the Base URL: Registration, Control Sessions and Sessions; without it, web PKI. It's a **PEM bundle that may hold several CA certificates**, parsed strictly (only `CERTIFICATE` blocks, each a CA) and bounded in count and size. `platform-mesh-operator` copies it into the `ProviderEndpoint`. It's input, never a link: a wrong value only stops the Owner's own Connection. **Rotating it is out-of-band**: the Provider tells its Owners about a new CA, Owners add it next to the old one before the Provider switches its server certificate, and remove the old one afterwards. New Sessions use the updated bundle; open Sessions continue. A missed update shows `EdgeTLS` on the `Provider` ([03 Operator, "Edge diagnosis"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)), and updating the field recovers it. A CEL rule rejects `providerKubeconfigSecret` and `hostOverride` on Remote Providers: they only apply to local ones ([03 Operator, "Spec validation"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
- **Created by the Owner**, never by `platform-mesh-operator`. A `Provider` left behind by a failed Onboarding waits for the next request.
- **Admin SA**: for Remote Providers, `platform-mesh-operator` creates it as `provider` in namespace `default` in every Provider Workspace; Connectors know the name as a constant. Local Providers keep `<name>-provider` (RFC 006).
- **One writer**: `platform-mesh-operator` writes the whole status, for both kinds ([03 Operator](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
- **Conditions**: one per reconcile step, plus the aggregate `Ready`. For Remote Providers: `Workspace`, `ConnectionBinding`, `ConnectorAccess`, `Registered`, `Connected`, `Ready`, and the warnings `ProtocolOutdated` and `TrustOutdated`, which don't affect `Ready`. Specific causes are in the messages, which start with a stable label. Aggregate only; no shard names.
- **`phase`**, for both kinds: `Pending → Provisioning → Ready ⇄ Degraded`, and `Deleting`, computed from the conditions; `Degraded` only for a `Provider` that was `Ready` before (`status.lastReadyTime`).
- **`status.connection`**: `url` and `connectionID` in use, for the Owner's information; never used as a link. When a new Connection replaces an earlier one, an Event names the old and the new Base URL host and `connectionID` ([03 Operator, "Onboarding requests"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
- **Annotation `providers.platform-mesh.io/rotate-connection-secret`**: any new value makes `platform-mesh-operator` change the Connection Secret once; values arriving while a change runs are coalesced, and on-demand changes are rate-limited ([03 Operator, "Secret Changes"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
- **Annotation `providers.platform-mesh.io/abandon-connection=<connectionID>`**: gives up the Connection, for a Provider that won't come back. It acts only if the value is the current `connectionID`. `platform-mesh-operator` deletes the `ProviderEndpoint`, sends a best-effort `DELETE` to the Provider-side, and deletes the `ProviderEndpointRegistration` and its `Secret`; the Provider Workspace, its SA, the binding and the RBAC stay. The `Provider` then shows `Registered=False` with `ConnectionAbandoned`, plus an Event, and a normal request onboards it anew, with a new `connectionID` ([03 Operator, "Abandoning a Connection"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).

### ProviderConnection (remove)

`providerconnection_types.go` in `providers.platform-mesh.io` is replaced by `ProviderEndpoint` in the same group, served by the Connection Export (below). It isn't served yet, so removal has no migration.

## In PM's runtime cluster

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
  url: https://provider.example.com/pm       # the base URL from the token; PM Administrators may change it
  serverKeys: [ "<PEM SPKI>", ... ]          # from registration, then updated from GET
  unusedExpiresAt: "2026-10-23T10:00:00Z"    # from registration: removed by the Provider-side if never reached by then
  rotationRequest: ""                        # the last value of the rotation annotation acted on
```

- **Authoritative**, together with its `Secret` (below): it holds everything PM knows about the Connection that can't be recomputed. The `ProviderEndpoint` is written from them (see "ProviderEndpoint"), and every change (a new or retired Server Key, a new Base URL, administrator input for recovery) goes into the `ProviderEndpointRegistration` first.
- **Only `spec`, no `status`**: its content can't be recomputed, so it isn't status. Health is reported on the `Provider`.
- **`connectionID`, `url`, `serverKeys` and `unusedExpiresAt` are persisted in one write**, as soon as Registration returns ([03 Operator, "Registration"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)). The token's data is deleted afterwards, so this is where the Base URL survives.
- **Validation** in the schema: `https` URLs only, key format, bounded list sizes.
- **Printer columns**: `connectionID` and `url`, so `kubectl get` lists every remote Connection without access to `Secret`s.
- **Written by `platform-mesh-operator`, from one reconcile per `Provider`** ([03 Operator, "Reconciling Providers"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)). PM Administrators may edit `url` (a Provider moved to a new Base URL) and `serverKeys` (for example after a Provider rebuilt elsewhere lost its keys), and re-enter `connectionID` when both this object and the `ProviderEndpoint` were lost ([03 Operator, "Writing ProviderEndpoints"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)); every write is an update with `resourceVersion`, so concurrent edits conflict instead of overwriting each other. Neither Owners nor Providers can reach it.
- **`url` may change after Registration**; nothing caches it elsewhere, and the `ProviderEndpoint` follows.
- **The Server Keys here** are the trust anchors Registration returned; Connectors read their copy from the `ProviderEndpoint`. They never live in Owner-writable status.
- **Deleted at offboarding** ([03 Operator, "Deletion"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)) and when the Owner abandons the Connection; there's no owner reference across kcp and the runtime cluster.
- **Lost `ProviderEndpointRegistration`s are rebuilt** without anyone acting: `connectionID`, `url` and `serverKeys` come from the `ProviderEndpoint`, the only time the `ProviderEndpoint` is read as input, and the Connection Secret is replaced through a Secret Change, since the `Secret` goes with it. Both copies are written only by `platform-mesh-operator`.

### Its `Secret`

`<clusterName>-secret`, in the same namespace, with an owner reference to the `ProviderEndpointRegistration`, so deleting one deletes the other. It holds only the credentials:

| Key | Content |
|---|---|
| `secret` | the current Connection Secret (`pmc1_…`) |
| `pendingSecret` | only while the Connection Secret changes ([03 Operator, "Secret Changes"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)) |

- **Read and written only by `platform-mesh-operator`.** In an Owner's workspace, every Provider with an accepted `Secret`s claim there could read it, since PM's bindings accept that claim with `matchAll`; and an Owner could delete it.
- **Lost `Secret`s are repaired** through a Secret Change; nobody has to act.

## The Connection Export, facing Provider Workspaces

### ProviderEndpoint

`providers.platform-mesh.io/v1alpha1`, served only by the Connection Export. One per Provider Workspace, cluster-scoped, with a fixed name (e.g. `default`), written only by `platform-mesh-operator`, from the `ProviderEndpointRegistration`, its `Secret` and the `Provider`'s `caBundle`:

| Field | Purpose |
|---|---|
| `connectionID` | Which Connection this workspace belongs to. The Provider-side applies a `ProviderEndpoint` only if this is its Connection's ID |
| `url` | The Provider's Base URL that Connectors dial (validated, `https` only). A change makes Connectors replace their Sessions |
| `serverKeys` | The Provider-side's pinned server public keys, for the Inner TLS |
| `caBundle` | The `Provider`'s `spec.remote.caBundle`, if set: the private CAs (one or more) that Connectors verify the Base URL's outer TLS with, instead of web PKI. A change makes Connectors replace their Sessions |
| `pendingSecretHash` | The hash of the `Secret`'s `pendingSecret`, the Connection Secret `platform-mesh-operator` is switching to; empty otherwise. The Provider-side adopts it ([01 Protocol, "Changing the Connection Secret"](./xxx-notes-011-v5-impl-01-protocol.md)) |

- **No `clusterName`**: the object's logical cluster is the Provider Workspace.
- **No SA reference**: Connectors act as the fixed admin SA `default/provider` (see "Provider").
- **No front-proxy URL**: the Provider-side doesn't name the front-proxy, and Connectors take it from their own configuration ([05 Connector, "Request pipeline"](./xxx-notes-011-v5-impl-05-connector.md)).
- **No status**: Connectors write nothing; health lives in metrics and in `platform-mesh-operator`'s view.
- **Validation** in the schema: `https` URLs only, bounded list sizes, key and hash format. `platform-mesh-operator`'s egress checks apply on top ([03 Operator, "Egress"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
- **Readable by the Provider** in its own workspace. Nothing in it is secret: `pendingSecretHash` is the hash of a 256-bit secret.
- **The Provider-side's only authenticated source for its Connection Secret.** It reads the `ProviderEndpoint` over its Sessions, so the `ProviderEndpoint`'s protection (below) is what keeps anyone but PM from changing a Connection Secret.

### Connection Export

`APIExport connections.providers.platform-mesh.io` in **`:root:providers:system`**:
- **Resources**: `providerendpoints` in `providers.platform-mesh.io`. The Providers Export serves other resources of the same group; kcp checks binding conflicts per resource name, not per group, and no workspace binds both exports.
- **Permission claim**: `apis.kcp.io/apiexportendpointslices`, verbs `get`, `list`, `watch`. The only claim. Connectors use it to see which shards have consumers ([05 Connector, "Which Sessions"](./xxx-notes-011-v5-impl-05-connector.md)).
- **`maximalPermissionPolicy.local`**, with RBAC in `:root:providers:system`:
  - `get`, `list`, `watch` on `providerendpoints` for the group `apis.kcp.io:binding:system:authenticated`;
  - writes (including `deletecollection` and `/status` if ever added) only for `apis.kcp.io:binding:<operator identity>`.
- **Endpoint slice**: one `APIExportEndpointSlice` for the export, consumed by `platform-mesh-operator` and every Connector.
- **Access in `:root:providers:system`**, for the Connectors' group (read only) and `platform-mesh-operator` (read and write on `content`). Nobody else:
  - `access` on `/`: kcp requires it for any workspace content, and the APIExport virtual workspace checks `apiexports/content` with a SubjectAccessReview in this workspace, which applies the same check;
  - `get`, `list`, `watch` on `apiexportendpointslices`, `resourceNames: [connections.providers.platform-mesh.io]`: multicluster-provider lists and watches the slice by name;
  - `get`, `list`, `watch` on `apiexports/content` for this export; `platform-mesh-operator` also the write verbs.

  `platform-mesh-operator`'s grants are listed even though its current identity (`kcp-admin`) has them anyway, so a dedicated identity later doesn't silently lose access.

### The binding in Provider Workspaces

- **Created by `platform-mesh-operator`** with the workspace, with a **fixed name** (e.g. `connections.providers.platform-mesh.io`), accepting the one permission claim ([03 Operator, "Connection Export binding"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
- **Protected by the Connector's policy**: the Provider can't update or delete it through a Session ([05 Connector, "Authorization and policy"](./xxx-notes-011-v5-impl-05-connector.md)). If it disappears anyway, `platform-mesh-operator` recreates it and rewrites the `ProviderEndpoint`.

## Policies in Provider Workspaces

The Provider is admin in its workspace, so it could grant rights to a kcp identity it controls and reach kcp directly, outside the Connector. Two `ValidatingAdmissionPolicy`s keep every grant inside the workspace or inside kcp's binding mechanism; `platform-mesh-operator` creates them, with their bindings and one ClusterRole, in every Remote Provider's workspace before any access is granted ([03 Operator, "Connector RBAC"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)):

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: providers.platform-mesh.io:rbac-subjects
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: [rbac.authorization.k8s.io]
        apiVersions: [v1]
        operations: [CREATE, UPDATE]
        resources: [rolebindings, clusterrolebindings]
  matchConditions:
    - name: not-platform-mesh-operator
      expression: request.userInfo.username != '<platform-mesh-operator kcp username>'   # from configuration
  variables:
    - name: toApiExportBind
      expression: >-
        object.roleRef.kind == 'ClusterRole' &&
        object.roleRef.name == 'providers.platform-mesh.io:apiexport-bind'
  validations:
    - expression: >-
        variables.toApiExportBind ||
        !has(object.subjects) ||
        object.subjects.all(s,
          s.kind == 'ServiceAccount' ||
          ((s.kind == 'User' || s.kind == 'Group') && s.name.startsWith('apis.kcp.io:binding:')))
      message: >-
        In a Provider Workspace, role bindings may only name ServiceAccounts, kcp binding
        subjects (apis.kcp.io:binding:…), or grant providers.platform-mesh.io:apiexport-bind.
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: providers.platform-mesh.io:no-sa-token-secrets
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: [v1]
        operations: [CREATE, UPDATE]
        resources: [secrets]
  matchConditions:
    - name: not-platform-mesh-operator
      expression: request.userInfo.username != '<platform-mesh-operator kcp username>'
  validations:
    - expression: object.type != 'kubernetes.io/service-account-token'
      message: ServiceAccount token Secrets aren't allowed in a Provider Workspace.
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: providers.platform-mesh.io:apiexport-bind
rules:
  - apiGroups: [apis.kcp.io]
    resources: [apiexports]
    verbs: [bind]
```

Each policy has a `ValidatingAdmissionPolicyBinding` with `validationActions: [Deny]`.
- **What stays allowed**: bindings to ServiceAccounts in the workspace; subjects prefixed `apis.kcp.io:binding:`, which kcp uses for an APIExport's maximal permission policy; and granting `bind` on the Provider's APIExports to anyone, through `providers.platform-mesh.io:apiexport-bind`, which is how consumers get to bind.
- **What it closes**: binding a user or group the Provider controls (or `system:authenticated`) to anything else, and creating `Secret`s that kcp would fill with a long-lived SA token. Together with the Connector's Denylist, which refuses `create` on `serviceaccounts/token`, this keeps every kcp credential for the workspace on the PM-side.
- **No `paramKind`**: the operator's username is written into each policy from configuration.
- **The ClusterRole has no aggregation rule**, so its rights can't grow.
- **Protected by the Connector's Denylist**: the Provider can't change or delete the policies, their bindings or the ClusterRole ([05 Connector, "Authorization and policy"](./xxx-notes-011-v5-impl-05-connector.md)).
- **They also apply to consumers' workspaces.** kcp applies an export's admission policies to the bound resources in every consumer workspace, so Providers' own policies on their exported resources do too ([07 SDK, "Requirements on Providers"](./xxx-notes-011-v5-impl-07-provider-sdk.md)). These two policies match only RBAC and `Secret`s, which no Provider exports.
- **To verify in kcp**: `ValidatingAdmissionPolicy`s on core and RBAC resources; `matchConditions` with `request.userInfo`; that ServiceAccount subjects resolve only within the workspace; that the `apis.kcp.io:binding:` prefix is reserved; how consumers get `bind` on APIExports today.

## Decided

- **Request authorization**: Owners create requests; a default ClusterRole grants requests and `Provider`s together; no identity checks in `platform-mesh-operator`.
- **Requests carry the `tokenID`**, and their spec is immutable.
- **Concurrent requests**: the oldest wins.
- **Requests reference an existing `Provider`**; the Owner creates it.
- **Connection state** lives in `platform-mesh-operator`'s namespace, not in Owners' workspaces: a `ProviderEndpointRegistration` per Connection, authoritative, with a `Secret` holding only the credentials; `ProviderEndpoint`s are written from them.
- **The `caBundle`** stays a `Provider` setting, copied into the `ProviderEndpoint`; it may hold several CAs; rotating it is out-of-band.
- **Remote Providers' admin SA** has a fixed name; the `ProviderEndpoint` doesn't carry it.
- **RBAC subjects and SA-token `Secret`s** in Provider Workspaces are restricted by `ValidatingAdmissionPolicy`s that `platform-mesh-operator` creates.
- **Abandoning a Connection** is an annotation on the `Provider`, matched against the `connectionID`.
- **One API group** for every PM-side type, `providers.platform-mesh.io`; `ProviderEndpoint` is served by its own export.
- **Version** of `ProviderEndpoint`: `v1alpha1`. The protocol coupling is carried by the protocol version ([01 Protocol, "Versioning"](./xxx-notes-011-v5-impl-01-protocol.md)), not the API version.
- **`ProviderEndpoint` scope**: cluster-scoped with a fixed name; every E2E Onboarding confirms `platform-mesh-operator` can write it through the APIExport virtual workspace ([10 Testing, "Onboarding"](./xxx-notes-011-v5-impl-10-testing.md)).
