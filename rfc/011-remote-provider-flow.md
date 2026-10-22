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
- Once the Owner requests it, the Provider is onboarded on a PM Instance with no further manual steps on either side; this includes the full lifecycle: Onboarding, Credential rotation, Offboarding.

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

Today, the Provider onboarding is fully manual: the Owner (of the user workspace that contains the `Provider` object) *waits* until the Provider workspace is ready, *retrieves* the kubeconfig (referenced by `Provider.status.providerKubeconfigSecretRef`), *copies it* over to a "safe location" such that the service provider can retrieve it. Then, the service provider *deploys* their service operator with this kubeconfig (who populates the `root:providers:<Provider ws name>` workspace) and starts reconciling consumers.

Because the manual flow couples the Owner and Provider-side, there is a fundamental assumption: the Owner and the Provider are the same persona (you trust the kubeconfig, so you deliver it yourself). For you, as a service provider, this has far-reaching consequences:

- You yourself reach out to the PM Instance, register yourself, onboard yourself. It is your initiative to expose your services. If a company running an internal PM Instance wants your services, you still have to onboard yourself.
- You communicate with the PM Instance's network team: I need my `root:providers:<Provider ws name>` to be reachable from my operators that are running in my company's cluster, can you create a tunnel for me?
- You keep your kubeconfigs safe: rotating SA tokens, revocation on leaks, ...,

In its current form, Platform Mesh puts the burden of being able to offer a service fully onto the Provider.

**Goals:**

- Allow distinct personas for "I want to publish a service on this PM Instance" (i.e. Owner) and "I want to offer my service to PM Instances" (i.e. Provider).
- The Provider doesn't need to care about one specific PM Instance; onboarding and up-keep is fully automatic through API.
- Assume as little as possible about the Provider's infrastructure. Any HTTPS endpoint that supports WebSockets must do. No TLS passthrough, no client-certificate-verifying ingress.
- No protocol code in the Provider's operator: it runs standard multicluster-runtime reconcilers against one kubeconfig per Connection. A multicluster provider that discovers those kubeconfigs is a follow-up.
- Current manual flows continue to be supported.

**Non-goals:**

- Converting a Local Provider into a Remote Provider. ManagedProvider-flow is a different trust model than a Remote Provider.
- Ensuring service's own data plane. The Remote Provider is still responsible for making non-KRM APIs of their service available to its consumers.
- Verifying the PM Instance onto which this Provider is to be onboarded is genuine. PM Instances can be private; there is no way for a Provider to verify PM's authenticity (we would need a public registry).

## Design

This RFC defines a number of deliverables to enable Remote Provider flows.

| Concept | Meaning | Implemented by |
|---|---|---|
| **Connector** | one shard's deployment of `provider-connector`: keeps that shard's Sessions, authorizes and proxies Provider requests to kcp. | a Deployment per shard |
| **Session** | a long-lived WebSocket from a Connector to the Base URL, with Inner TLS, carrying the Provider's requests to kcp; opened only where the Connection has traffic | `…/session` |
| `Connection` | kind, in `provider-sdk.platform-mesh.io` group: the Provider-side half of a Connection | the Provider's cluster |
| **Onboarding** | Owner creates `ProviderOnboardingRequest`; triggers the exchange of keys and CAs that forms two-way trust | `ProviderOnboardingRequest` |
| **Base URL** | the Provider's URL under which the Connection API is served | `provider-gateway`, owned by Provider |
| **Registration** | the request on `…/register/{tokenID}`, inside Inner TLS, that creates a Connection | — |
| **Provider Workspace** | `:root:providers:<name>-<owner cluster>`, created by `platform-mesh-operator` for each `Provider` | kcp `Workspace` |

We deliver `provider-gateway` that the Provider-side deploys:
- It's an HTTP server that accepts every connection the PM-side opens: Registration, Control Sessions and Sessions.
- It's also a forward proxy: K8s clients reach kcp through it, over their Connection's Sessions.
- It maintains a set of kubeconfig Secrets in Provider's cluster, one per Connection. K8s clients connect to those to:
  - bootstrap the Provider Workspace (create your exports, schemas, content configuration, ...),
  - and reconcile Provider's APIExportEndpointSlices (multicluster-runtime, kcp's apiexport mc-provider, ...; all standard tooling).
- It never initiates Sessions itself, and it never receives any kcp credentials!
  - All communication is initiated by PM-side. Sessions exist only for Connections created by a successful Onboarding; anyone else never gets past the Inner TLS.

We deliver `/v1/connections` REST API served by a server running in-process with `provider-gateway`:
- It has three parts, each listening for WebSockets from the PM-side:
  - **Registration:** PM-side directs creation of a Connection on the Provider-side, authorised by the Owner's Onboarding Token,
  - **Session:** carry the Provider's traffic to kcp over a WebSocket between a Connector (the caller, PM-side) and the `provider-gateway` (the receiver, Provider-side),
  - **Control:** the PM-side checks the Connection's health, learns the Provider's Server Keys, pushes Connector CA rotations, and deletes the Connection as clean-up at offboarding.
- Provider-side exposes this API at an URL that PM-side can reach.

We deliver Provider-SDK K8s APIs that are part of the `provider-gateway`:
- CRDs delivered in the same Helm chart as the gateway, and reconciled in that.
- `provider-sdk.platform-mesh.io` group.
- Namely:
  - `Connection`: describes Provider-side of the contract,
  - `OnboardingToken`: the Provider-side's record of an Onboarding Token it issued to an Owner, and whether it's been spent,
  - `ServerKey`: Provider-side keypair used to let PM-side verify "this is the Provider I have onboarded".

We deliver `provider-connector` that the PM-side deploys, one per shard:
- It opens WebSocket connections against Provider's `GET /v1/connections/{id}/session`.
- It authorizes the Provider against kcp: it mints a short-lived SA token (against the SA in Provider Workspace) that's injected into each Provider-side request.
- It proxies Provider-side requests to:
  - Shard's APIExport VW, or
  - Provider Workspace.
  - Any request that doesn't match either of those two by path is refused.

We deliver K8s/kcp APIs and exports that PM-side installs:
- `providers.platform-mesh.io` group.
- Namely:
  - `Provider`: gains `spec.remote`,
  - `ProviderOnboardingRequest`: Owner's desire to onboard a Provider,
  - `ProviderEndpoint`: describes PM-side of the contract, bound in Provider Workspaces, written only by PM,
  - `ConnectorCASet`: singleton object on the PM runtime cluster that tracks the issuances of the Connector CA. Needed to roll a renewed CA over without disrupting healthy Connections.
  - `ProviderEndpointRegistration`: authoritative record of Provider's Connection registered with this PM Instance.

The PM-side (admins) delivers:
- Egress to Providers

The Provider-side delivers:
- multicluster-aware operator(s) for their service,
- PM-facing microfrontend (technically optional),
- An HTTPS endpoint at their Base URL that the PM side can reach *somehow* (we support public Web PKI and private CA bundles (delivered out-of-band)), AND it allows WebSocket upgrades.
- A way for the Owner to *somehow* get their Onboarding Token (e.g. Provider's portal, where the Owner has created an account beforehand).

### Overview

| Concept | Meaning | Implemented by |
|---|---|---|
| **Connection Secret** | the bearer credential for Control Calls; the Provider-side stores only its hash, and only PM changes it | key `secret` in `<clusterName>-secret` (PM-side); its hash stored in `Connection.status.secretHash` (Provider-Side) |
| **Onboarding Token** | single-use, short-lived token from the Provider's portal | `OnboardingToken` (Provider-side resource); a `Secret` in the Owner's workspace |
| **Edge** | whatever sits in front of the Base URL: ingress, load balancer, CDN, WAF | the Provider's infrastructure |
| **Outer TLS** | the TLS handshake from PM-side up to Provider's Edge (can be public Web PKI, or private bundled CA) | — |
| **Inner TLS** | the TLS handshake inside a Registration, a Session or a Control Session, authenticating the Provider-side by a pinned Server Key, and on Sessions the Connector too | — |

In general, any communication that ever takes place in the Remote Provider flow starts on the PM-side. Before PM-side can start opening sessions on Provider-side, trust needs to be established. This is done through a number of REST API calls that exchange TLS material and key fingerprints. Only once two-way trust is established, Connectors may start opening Sessions against Provider's `provider-gateway`, and Provider's service operators start reconciling.

This RFC tries to follow zero-trust principles, and that's why two layers of TLS are used. Specifically, Outer TLS that's terminated at Provider's Edge, and Inner TLS that's terminated at `provider-gateway`. The Edge has no role in the Connection, so there is no need for it to see its data in plaintext (which could contain sensitive credentials like DB passwords for DBaaS, etc.).

Principles:

- **Everything that grants anything travels inside Inner TLS.** Only WebSocket upgrades are plain HTTP, and they carry no credentials.
- **kcp credentials stay on the PM-side.** You can't leak what you don't have.
- **Nothing outside PM and the Provider's runtime** can take over a Connection: not a stolen Onboarding Token or a compromised portal account.
- **Revocation is immediate.** When the Connectors stop proxying, the Provider-side has nothing left to use.

```
  PM                                           │ internet │   Provider
                                               │          │                   ┌────────────── trust boundary ──────────────┐
  kcp ◀══TLS══ connector ══════════════════════╪══════════╪═▶ ingress/CDN ──▶ provider-gateway ◀══ CONNECT ══ operator     │
              adds SA token    outer TLS,      │          │   outer TLS       │                    tunnel, TLS to the      │
                               WebSocket       │          │   ends here       │                    gateway's certificate   │
              connector ════ inner TLS 1.3 ════╪══════════╪═════════════════════▶ provider-gateway                         │
                                               │ client: connector certificate (from a pinned connector CA)                │
                                               │ server: pinned server key                                                 │
                                               │   ╚═ yamux streams, plain HTTP on each, inside the inner TLS              │
                                               │          │                   └────────────────────────────────────────────┘

  connector: routes each request by path to the front-proxy or its shard's VW, checks it, adds the SA token

  ingress/CDN sees:               opaque bytes
  acts as the Provider in kcp:    the Provider's runtime only (provider-gateway, and the operator through it)
  can fake kcp to the Provider:   nobody without a connector certificate (PM's side)
  sees the connection secret:     nobody (control calls run inside the inner TLS too)
```

### Flow & APIs

There are many APIs, many moving pieces :/ It's best to see the flow in a "real" example. In the sections that follow, each one describes one step of the flow with its respective APIs.

#### 0. Before anything happens


