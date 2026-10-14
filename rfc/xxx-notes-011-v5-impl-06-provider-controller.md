# v5 implementation plan: Provider controller

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). The RFC 006 Provider controller lives in `platform-mesh/operators/platform-mesh-operator` (`internal/controller/providers/provider_controller.go`, subroutines in `pkg/subroutines/providers`). It runs its subroutines through the `go.platform-mesh.io/subroutines` lifecycle with a condition manager. Today it handles local Providers: it creates the provider workspace `:root:providers:<name>-<owner cluster>`, an admin SA, a legacy SA-token Secret and a scoped kubeconfig. Under v5 it also owns **remote Providers** (`spec.remote` set) end to end: their bootstrap here, their connection through the subroutines described in [03 Connection handling](./xxx-notes-011-v5-impl-03-providerconnections-operator.md).

## Principles

- **One controller owns a `Provider`**, local or remote: every subroutine, every status field, one finalizer.
- **No kcp credential for remote Providers** (overview, principle 1): no token Secret, no kubeconfig.
- **Local Providers and ManagedProviders keep working as today**, apart from the `phase` values (see "Conditions and phase").

## Subroutines

Each kind of `Provider` has its own sequence; a subroutine runs only once the ones before it have succeeded. Running one lifecycle per kind (two controllers on the same type, filtered by `spec.remote`) keeps each `Provider`'s conditions to its own subroutines.

| Order | Local Providers | Remote Providers |
|---|---|---|
| 1 | Workspace (`ProviderWorkspaceSubroutine` today) | `Workspace`: the provider workspace, the admin SA (cluster-admin in the workspace, the SA connectors act as) |
| 2 | Scoped kubeconfig (`ScopedKubeconfigSubroutine` today): SA-token Secret, kubeconfig copied to the owner's workspace | `ConnectionBinding`: the binding to the connection export (below) |
| 3 | | `ConnectorAccess`: the connector RBAC (below) |
| 4 | | `Registered`: waits for a registration, makes the first control call, writes the record ([03, "The `Registered` subroutine"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)) |
| 5 | | `Connected`: health on every poll, secret changes, trusted-set pushes ([03, "The `Connected` subroutine"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)) |

The local subroutines' names are renamed for consistency later ([follow-ups](./xxx-notes-011-v5-impl-todo.md)); until then, the workspace subroutine shared by both kinds keeps its current name.

### Connection export binding

- **Created by the `ConnectionBinding` subroutine, with a fixed name,** in the provider workspace, referencing the connection export in `:root:providers:system` and accepting its one permission claim (read-only `apiexportendpointslices`) ([02 APIs, "The binding in provider workspaces"](./xxx-notes-011-v5-impl-02-apis.md)).
- **Recreated if deleted.** The Provider can't delete it through a session ([05 Connector, "Authorization and policy"](./xxx-notes-011-v5-impl-05-connector.md)), but a PM operator error could.
- **Bind permission**: the controller's identity needs `bind` on the connection export.

### Connector RBAC

The `ConnectorAccess` subroutine creates, in each remote Provider's workspace, a role and binding for the **connectors' identity** (one group shared by all shards' connectors):
- `access` on `/` (workspace content);
- `create` on `serviceaccounts/token` with `resourceNames` limited to the Provider's SA, in the SA's namespace.

Nothing else. Connectors read records through the connection export, not through this RBAC ([05 Connector, "Identity towards kcp"](./xxx-notes-011-v5-impl-05-connector.md)).

## Spec validation

A CEL rule on `ProviderSpec`: `spec.remote` excludes `providerKubeconfigSecret` and `hostOverride`, which only apply to local Providers. An owner who sets both gets an error at apply time, instead of fields that silently do nothing. The remote sequence still never creates a kubeconfig, as defense in depth. No existing `Provider` has `spec.remote`, so nothing becomes invalid. Whether a `Provider` may switch between local and remote isn't decided here.

## Conditions and phase

**Conditions** come from the subroutine framework's condition manager, for both kinds:
- **one per subroutine, named after it**: `True` when it completed; `False` with reason `Stopped` or `Error` when it didn't; later subroutines are marked `Skipped`;
- **`Ready`**: the aggregate, `True` when every subroutine completed. The printer column shows it.

For remote Providers that gives `Workspace`, `ConnectionBinding`, `ConnectorAccess`, `Registered`, `Connected` and `Ready`. Specific causes go into the **message**, which starts with a stable label and says what to fix, for example `EdgeBotChallenge: the Provider's edge answered the WebSocket upgrade on /control with an HTML page. Ask the Provider to exempt its connection paths from bot protection; see its edge checklist.` People read these; nothing acts on them by machine, since nothing on the PM-side can fix a Provider's edge. The labels used by `Registered` and `Connected` are listed in [03](./xxx-notes-011-v5-impl-03-providerconnections-operator.md).

One condition sits outside the framework: the warning **`ProtocolOutdated`**, written by the `Connected` subroutine. It's informational and doesn't affect `Ready`.

**`phase`**, for both kinds:

| Phase | Means |
|---|---|
| `Pending` | Nothing done yet |
| `Provisioning` | Not yet usable for the first time; the conditions say which step is pending. Replaces `ProvisioningWorkspace` and `ProvisioningKubeconfig` |
| `Ready` | Usable: `Ready` is true |
| `Degraded` | Was `Ready` once, and `Ready` is false now; the conditions say why. For local Providers this usually means the controller can't currently confirm or repair something (kcp unavailable, the token Secret recreated), while the existing kubeconfig keeps working |
| `Deleting` | Being deleted |

- **Computed in one place**, after each reconcile, from the conditions. Nothing else writes `phase`.
- **`Ready` and `Degraded` always mirror the `Ready` condition**, so they can't disagree.
- **`Degraded` needs a `Provider` that was `Ready` before**, remembered in `status.lastReadyTime`, so a controller restart doesn't turn it back into `Provisioning`.
- **ManagedProviders** keep comparing their `Provider`'s `phase` with `Ready`; how they should react to `Degraded` is a [follow-up](./xxx-notes-011-v5-impl-todo.md).

`status.connection { url, connectionID }` on remote Providers is informational for the owner: the base URL and the connection ID in use, updated when the `Provider` is onboarded again, and never used as a link.

## Deletion

One finalizer per `Provider`, running the kind's cleanup in order. For remote Providers:
1. the connection steps ([03, "Offboarding"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)): delete the record (sessions close, kcp access ends), call `DELETE` on the Provider-side, delete the connection Secret;
2. the connector RBAC, then the binding, while nothing uses them anymore;
3. the SA and the workspace.

Local Providers keep today's cleanup.

## Events

Phase changes and failures, and for remote Providers the tamper response ([03, "Connector CA trusted set"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)), are recorded as events on the `Provider`, without secrets.

## No migration

`spec.remote` isn't used by any code today, so there are no remote Providers to migrate. Existing Providers keep the RFC 006 kubeconfig flow; v5 applies only to Providers onboarded through a `ProviderOnboardingRequest`.

## Work breakdown

1. Two lifecycles on the `Provider` type, filtered by `spec.remote`.
2. The CEL rule in `platform-mesh/apis`.
3. Remote subroutines: `Workspace` (reusing the existing one), `ConnectionBinding`, `ConnectorAccess`; `Registered` and `Connected` come from [03](./xxx-notes-011-v5-impl-03-providerconnections-operator.md).
4. `phase` computed from the conditions for both kinds, with `status.lastReadyTime`.
5. The remote finalizer's order.
6. Tests: unit for each subroutine and for `phase`, the CEL rule against kcp, kind e2e for a remote Provider's bootstrap and deletion ([10 Testing](./xxx-notes-011-v5-impl-10-testing.md)).

## Decided

- **One controller owns each `Provider`**: platform-mesh-operator's Provider controller, local and remote.
- **Conditions are the subroutines'**, with labelled messages; `ProtocolOutdated` is the only extra condition.
- **`phase` for both kinds**: `Pending → Provisioning → Ready ⇄ Degraded`, and `Deleting`, computed from the conditions.
- **Local fields are rejected on remote Providers** by CEL.
- **No migration**: `spec.remote` is unused today.
- **Connector identity**: one group for all shards' connectors.
- **The admin SA keeps cluster-admin** in the provider workspace, as in RFC 006; the connector's denylist and the record's maximal permission policy narrow it where PM needs to.
