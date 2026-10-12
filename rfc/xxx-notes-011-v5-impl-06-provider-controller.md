# v5 implementation plan: Provider controller changes

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). The RFC 006 Provider controller lives in `platform-mesh/operators/platform-mesh-operator` (`internal/controller/providers/provider_controller.go`, subroutines in `pkg/subroutines/providers`). Today it creates the provider workspace `:root:providers:<name>-<owner cluster>`, an admin SA, a legacy SA-token Secret and a scoped kubeconfig. For **remote Providers** (`spec.remote` set), v5 changes what it creates.

## Principles

- **The Provider controller still owns the provider workspace and everything kcp-side in it**, except the record.
- **No kcp credential for remote Providers** (overview, principle 1): no token Secret, no kubeconfig.
- **Local Providers and ManagedProviders are unchanged.**

## What changes for remote Providers

| Today | Remote Providers under v5 |
|---|---|
| Workspace `:root:providers:<name>-<owner cluster>` | Unchanged |
| Admin SA with a cluster-admin binding | Unchanged: the SA connectors act as |
| Legacy SA-token Secret | **Not created** |
| Scoped kubeconfig Secret, copied to the owner's workspace | **Not created** |
| — | **Binding to the connection export** (below) |
| — | **Connector RBAC** (below) |
| — | **`ConnectionPrerequisitesReady` condition** |

## Connection export binding

- **Created with the workspace**, with a fixed name, referencing the connection export in `:root:providers:system` and accepting its one permission claim (read-only `apiexportendpointslices`) ([02 APIs, "The binding in provider workspaces"](./xxx-notes-011-v5-impl-02-apis.md)).
- **Recreated if deleted.** The Provider can't delete it through a session ([05 Connector, "Authorization and policy"](./xxx-notes-011-v5-impl-05-connector.md)), but a PM operator error could.
- **Bind permission**: the Provider controller's identity needs `bind` on the connection export.

## Connector RBAC

In each remote Provider's workspace, a role and binding for the **connectors' identity** (one group shared by all shards' connectors):
- `access` on `/` (workspace content);
- `create` on `serviceaccounts/token` with `resourceNames` limited to the Provider's SA, in the SA's namespace.

Nothing else. Connectors read records through the connection export, not through this RBAC ([05 Connector, "Identity towards kcp"](./xxx-notes-011-v5-impl-05-connector.md)).

## Readiness for the operator

- **`ConnectionPrerequisitesReady`** on the `Provider` once the workspace, SA, binding and connector RBAC exist, written with its own field manager ([02 APIs, "Status ownership"](./xxx-notes-011-v5-impl-02-apis.md)).
- **The operator treats it as a hint** and verifies through PM-owned objects ([03 Operator, "Finding the provider workspace"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)).
- **Shared naming functions**: move the workspace and SA name derivations into a package both operators import, so they can't drift.

## Deletion order

- **The operator's finalizer runs first**: it deletes the record (sessions close), calls `DELETE` on the Provider-side, then removes its finalizer ([03 Operator, "Offboarding"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)).
- **The Provider controller waits** until the operator's finalizer is gone before deleting the connector RBAC, SA, binding and workspace. Otherwise connectors could lose their token rights while sessions are still closing, and the record's deletion could fail with the binding gone.

## No migration

`spec.remote` isn't used by any code today, so there are no remote Providers to migrate. Existing Providers keep the RFC 006 kubeconfig flow; v5 applies only to Providers onboarded through a `ProviderOnboardingRequest`.

## Work breakdown

1. Branch the subroutines on `spec.remote`: skip the token Secret and kubeconfig, add binding and connector RBAC subroutines.
2. Shared naming package.
3. `ConnectionPrerequisitesReady` condition with its own field manager.
4. Finalizer coordination with the operator.
5. Tests: unit for each subroutine, kind e2e for a remote Provider's bootstrap ([10 Testing](./xxx-notes-011-v5-impl-10-testing.md)).

## Decided

- **No migration**: `spec.remote` is unused today.
- **Connector identity**: one group for all shards' connectors.
- **The admin SA keeps cluster-admin** in the provider workspace, as in RFC 006; the connector's denylist and the record's maximal permission policy narrow it where PM needs to.
