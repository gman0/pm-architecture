---
title: Remote providers flow (core)

---

# RFC 011: Remote providers flow

| Status  | Proposed   |
|---------|------------|
| Author  | @gman0     |
| Created |            |
| Updated |            |

## Summary

Remote Provider flow support for lifecycling third-party Providers. Defines APIs, protocols, libraries and flows to enable automatic onboarding, securing, and offboarding of said Providers.

- Any PM Instance can be reconciled by the Provider. There can be many PM Instances that the Provider is able to reconcile.
- The Provider is automatically onboarded on a PM Instance without manual intervention; this includes the full lifecycle: Onboarding, Credential rotation, Offboarding.

| Concept | Meaning | Implemented by |
|---|---|---|
| **PM Instance** | one Platform Mesh deployment | — |
| **PM-side** | the PM Instance's part of a Connection | `platform-mesh-operator`, Connectors |
| **Provider** | the party offering services: its controllers and portal in its own cluster | represented on the PM-side by a `Provider` |
| **Provider-side** | the Provider's part of a Connection | `provider-gateway` |
| **Local Provider** / **Remote Provider** | a `Provider` without / with `spec.remote`: a kubeconfig (RFC 006) vs a Connection (this RFC) | `Provider` |

## Motivation

| Concept | Meaning | Implemented by |
|---|---|---|
| **Owner** | whoever controls the workspace holding a `Provider`; the Provider's customer | — |

Today, the Provider onboarding is fully manual: the Owner (of the user workspace that contains the `Provider` object) *waits* until the Provider workspace is ready, *retrieves* the kubeconfig (referenced by `Provider.status.kubeconfigSecretRef`), *copies it* over to a "safe location" such that the service provider can retrieve it. Then, the service provider *deploys* their service operator with this kubeconfig (who populates the `root:providers:<Provider ws name>` workspace) and starts reconciling consumers.

Because the manual flow couples the Owner and Provider-side, there is a fundamental assumption: the Owner and the Provider are the same persona (you trust the kubeconfig, so you deliver it yourself). For you, as a service provider, this has far-reaching consequences:

- You yourself reach out to the PM Instance, register yourself, onboard yourself. It is your iniciative to expose your services. If a company running an internal PM Instance wants your services, you still have to onboard yourself.
- You communicate with the PM Instance's network team: I need my `root:providers:<Provider ws name>` to be reachable from my operators that are running in my company's cluster, can you create a tunnel for me?
- You keep your kubeconfigs safe: rotating SA tokens, revocation on leaks, ...,

In its current form, Platform Mesh puts the burden of being able to offer a service fully onto the Provider.

**Goals:**

- Allow distinct personas for "I want to publish a service on this PM Instance" (i.e. Owner) and "I want to offer my service to PM Instances" (i.e. Provider).
- The Provider doesn't need to care about one specific PM Instance; onboarding and up-keep is fully automatic through API.
- Assume as little as possible about the Provider's infrastructure. Any HTTPS endpoint that supports WebSockets must do. No TLS passthrough, no client-certificate-verifying ingress.
- No changes in Provider's operator required. All they need is being able to run multicluster-runtime reconcilers against many endpoints (kubeconfigs in secrets).
- Current manual flows continue to be supported.

**Non-goals:**

- Converting a Local Provider into a Remote Provider. ManagedProvider-flow is a different trust model than a Remote Provider.
- Ensuring service's own data plane. The Remote Provider is still responsible for making non-KRM APIs of their service available to its consumers.

## Design

This RFC defines a number of deliverables to enable Remote Provider flows.

| Concept | Meaning | Implemented by |
|---|---|---|
| **Connector** | one shard's deployment of `provider-connector`: keeps that shard's Sessions, authorizes and proxies Provider requests to kcp. | a Deployment per shard |
| **Session** | a WebSocket from a Connector Replica to the Base URL, with Inner TLS, a Hello and yamux; one per Connection, shard and Connector Replica, only where the Connection has traffic | `…/session` |
| `Connection` | kind, in `provider-sdk.platform-mesh.io` group: the Provider-side half of a Connection | the Provider's cluster |
| **Onboarding** | creating a Connection, initiated from PM-side by creating `ProviderOnboardingRequest`, exchanging initial TLS material | `ProviderOnboardingRequest` |
| `/v1/connections…` | the Connection API's endpoints: `…/register/{tokenID}`, `GET /v1/connections/{id}`, `…/control`, `…/session`, `PUT …/connector-ca-set`, `DELETE` | under the Base URL |
| **Base URL** | the Provider's URL under which the Connection API is served | `provider-gateway`, owned by Provider |
| **Registration** | the request on `…/register/{tokenID}`, inside Inner TLS, that creates a Connection | — |
| **Provider Workspace** | `:root:providers:<name>-<owner cluster>`, created by `platform-mesh-operator` for each `Provider` | kcp `Workspace` |
| **Outer TLS** | Provider-side TLS - Web PKI, or private CA bundle | — |
| **Inner TLS** | TLS inside a Registration, a Session or a Control Session, authenticating the Provider-side by a pinned Server Key, and on Sessions the Connector too | — |

We deliver `provider-gateway` that the Provider-side deploys:
- It's a reverse-proxy that maintains Sessions against the PM-side.
- It maintains a set of kubeconfig Secrets in Provider's cluster, one per Connection. K8s clients connect to those to:
  - bootstrap the Provider Workspace (create your exports, schemas, content configuration, ...),
  - and reconcile Provider's APIExportEndpointSlices (multicluster-runtime, kcp's apiexport mc-provider, ...; all standard tooling).
- It never initiates Sessions itself, and it never receives any kcp credentials!
  - All communication is initiated by PM-side. Connections may be established only once Onboarded successfully - failing that, requests never get past TLS.

We deliver `/v1/connections` REST API that runs in a server, inside `provider-gateway`:
- It defines HOW to deliver data related to:
  - **Onboarding:** Owner wishes to onboard Provider,
  - **Registration:** onboarded Provider can create Connections against kcp,
  - **Session:** pin an incoming WebSocket between a Connector (the caller, PM-side) and the `provider-gateway` (the receiver, Provider-side),
  - **Control:** exchange of TLS material between the two sides,
  - **Offboarding:** Owner wishes to delete this Connection.
- Provider-side exposes it at an URL that PM-side can reach.

We deliver Provider-SDK K8s APIs that are part of the `provider-gateway`:
- CRDs delivered in the same Helm chart as the gateway, and reconciled in that.
- `provider-sdk.platform-mesh.io` group.
- Namely:
  - `connections`: describes Provider-side of the contract,
  - `onboardingtokens`: contains a single-use short-lived secret that is *somehow* surfaced to the Owner (e.g. Provider's onboarding portal), and the Owner uses it to file an onboarding request,
  - `serverkeys`: Provider-side keypair used to verify PM-side is the one this Provider is onboarded.

We deliver `provider-connector` that the PM-side deploys, one per shard:
- It opens WebSocket connections against Provider's `GET /v1/connections/{id}/session`.
- It authorizes the Provider against kcp: it mints a short-lived SA token (against the SA in Provider Workspace) that's injected each Provider-side request.
- It proxies Provider-side requests to:
  - Shard's APIExport VW, or
  - Provider Workspace.
  - Any request that doesn't match by path is refused.
- It knows that it needs to open this connection because:
  - The controller finds a `ProviderEndpoint` object inside Provider Workspace that's inside *THIS* shard. This is how a Provider gains access to their workspace. This `ProviderEndpoint` is created and owned by PM's provider controller; binding is protected by Maximal Permission Policy - Provider SA gets read-only RBAC against this resource.
  - The controller finds an `APIExportEndpointSlice` inside Provider Workspace, whose `.status.urls[].url` matches the APIExport VW URL for *THIS* shard. The export `ProviderEndpoint` gets a read-only claim against `APIExportEndpointSlices`, so that's how `provider-connector` gets access to those.

We deliver K8s/kcp APIs and exports that PM-side installs:
- `providers.platform-mesh.io` group.
- Namely:
  - `providerendpoints`: describes PM-side of the contract, bound in Provider Workspaces,
  - `connectorcasets`: set of Connector CAs 

- Deliver `provider-gateway` that the Provider deploys. It handles Provider lifecycling (onboard, offboard) and Connections towards onboarded PM Instances. Each Connection is backed by a kubeconfig secret that any multicluster-aware controller can reconcile.

- Provider never reaches out to PM-side. Instead, PM-side dials out to the `provider-gateway` API, acting on Owner's request. Onboarding at Provider-side is protected by a one-time token that the Provider mints; that's how a Provider can trust a PM Instance.

- Deliver `connector` that the PM-side deployes per each shard.

- PM-side runs `connector` per each shard. `provider-gateway`

### Overview

```
Provider's operator ── client-go, a kubeconfig per connection (proxy-url → provider-gateway)
                                   │ CONNECT host → provider-gateway opens a stream on a session for that host
             ┌──── one WebSocket per shard (outbound from PM) ──────────┘
             │       inner TLS 1.3 (connector certificate ↔ pinned server key)
             │         yamux inside; plain HTTP on each stream
             ▼
   connector@shard ── in-process http.Server + reverse proxy:
                      route by path, strip auth headers, add SA token ── TLS ──▶ kcp
```

