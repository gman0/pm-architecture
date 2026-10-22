---
title: Remote Providers flow

---

# RFC 011: Remote providers flow

| Status  | Proposed   |
|---------|------------|
| Author  | @gman0     |
| Created |            |
| Updated |            |

## Summary

Remote Provider flow support for lifecycling third-party Providers. Defines APIs, protocols, components and flows to enable automatic onboarding, securing, and offboarding of Providers.

## Motivation

Today, the Provider onboarding is fully manual: the Owner (of the user workspace that contains the `Provider` object) *waits* until the Provider workspace is ready, *retrieves* the kubeconfig (referenced by `Provider.status.providerKubeconfigSecretRef`), *copies it* over to a "safe location" so that the service provider can retrieve it. Then, the service provider *deploys* their service operator with this kubeconfig (which populates the `:root:providers:<Provider ws name>` workspace) and starts reconciling consumers.

Because the manual flow couples the Owner and Provider, there is a fundamental assumption: the Owner and the Provider are the same persona (you trust the kubeconfig, so you deliver it yourself). For you, as a service provider, this has far-reaching consequences:

- You yourself reach out to the PM Instance, register yourself, onboard yourself. It is your initiative to expose your services. If a company running an internal PM Instance wants your services, you still have to onboard yourself.
- You communicate with the PM Instance's network team: I need my `:root:providers:<Provider ws name>` to be reachable from my operators that are running in my company's cluster, can you create a tunnel for me?
- You keep your kubeconfigs safe: rotating SA tokens, revocation on leaks, etc.

In its current form, Platform Mesh puts the burden of being able to offer a service fully onto the Provider.

**Goals:**

- Allow distinct personas for "I want to publish a service on this PM Instance" (i.e. Owner) and "I want to offer my service to PM Instances" (i.e. Provider).
- The Provider doesn't need to care about one specific PM Instance; onboarding and up-keep are fully automatic through APIs.
- Assume as little as possible about the Provider's infrastructure. Any HTTPS endpoint that supports WebSockets must do. No TLS passthrough, no client-certificate-verifying ingress.
- No protocol code in the Provider's operator: it runs standard multicluster-runtime reconcilers against one kubeconfig.
  - There can be many kubeconfigs, one per PM connection. A multicluster provider that discovers those kubeconfigs is a follow-up.
- Current manual flows continue to be supported.

**Non-goals:**

- Converting a Local Provider (i.e. the manual flow, [RFC 006](./006_provider-bootstrap-operator.md)) into a Remote Provider (this RFC). ManagedProvider-flow (also RFC 006) has a different trust model from a Remote Provider.
- Tunneling the service's own data plane. The Remote Provider is still responsible for making non-KRM APIs of its own service available to its consumers.
- Verifying the PM Instance onto which this Provider is to be onboarded is genuine. PM Instances can be private; there is no way for a Provider to verify PM's authenticity (we would need a public registry).
- Ensuring every possible network configuration under the sun works out-of-the-box with this RFC. If PM-side cannot reach Provider-side, network teams still need to *somehow* allow HTTPS traffic with WebSocket upgrades to pass from the PM-side into the Provider-side (they would need that anyway, so that the consumers can reach the service's own data plane). This RFC is only ensuring that as long as the PM-side can reach the Provider-side, the onboarding *can* succeed and the Provider *can* serve consumers on the PM Instance.

## Architecture

**The general idea is:** because the Provider doesn't care about one specific PM Instance (there can be many), it is the PM Instance that calls out to the Provider to "onboard". And because it is **always** the PM-side that opens the connections first, the Provider-side has to expose an HTTPS endpoint that the PM-side can reach. This is a critical pre-condition to the whole design, and every design choice here relies on that.

What runs on that endpoint is ours to deliver (`provider-gateway`, this RFC discusses that), and the Provider just deploys and applies standard security restrictions as with any ingress-facing Deployment. The traffic may go over the Internet and private/public egress/ingress (CDNs, WAFs, LBs, ...) and into remote clusters (not ours), and we cannot assume control of any of those:
- CDN can log the body of requests / responses - we can't possibly know who has access to those,
- Provider's cluster falls behind K8s version upgrades, and is now vulnerable to attacks - putting the consumers of their services at risk,
- PM's runtime cluster falls behind K8s version upgrades, and is now vulnerable to attacks - putting the connected Providers at risk,
- any other horror scenario one could dream of.

That's why this RFC employs a set of rules:
- All traffic is under TLS end-to-end. Because we cannot assume the Provider would be happy to give PM a TLS passthrough, we employ a two-layer TLS setup: **Outer-TLS** between the PM-side and the Provider's ingress, where it's terminated. **Inner-TLS** then travels inside the upgraded WebSocket connection, and is terminated only by our HTTP server, the `provider-gateway` - on the Provider-side.
  - Why does either side trust the Inner-TLS? Initially (before onboarding) neither of them does, but they build this trust progressively: starting from the token handover during onboarding, and then both sides pin each other's public key material, and verify the handshake against that.
- Every key, certificate and CA this RFC introduces can be rotated without re-onboarding (most are already designed to rotate automatically, or can be made automatic as a follow-up).
- No private key or other secret crosses sides; only public keys, certificates and hashes do - with two exceptions:
  - Onboarding Token: single-use, short-lived, starts Onboarding.
  - Connection Secret: PM's bearer credential *towards* the Provider. It authenticates Control Calls for this specific Connection (see next sections).
  - Both of those are used *only* in the Inner-TLS.
- Severing the Provider's connection on the PM-side immediately terminates the Provider's access to kcp. The Provider simply never receives any kcp credentials that it could abuse once its connection is closed.

The trust boundary, compared with today:

| | Today (RFC 006) | With this RFC |
|---|---|---|
| Parties that trust each other | PM Instance ⇄ Provider's cluster | The same |
| Who opens connections | The Provider-side, to kcp | The PM-side, to `provider-gateway` |
| Inbound endpoint on the Provider-side | None: the operator dials out to kcp | `provider-gateway`'s endpoint, which accepts nothing without the pinned Inner-TLS |
| How the sides authenticate | The Provider checks kcp's certificate against the kubeconfig's CA; kcp checks a static SA token. Requires TLS passthrough to kcp to keep the SA token private. | Inner-TLS pinned both ways; the Connection Secret on Control Calls |
| kcp credentials outside the PM-side | A kubeconfig with a static SA token, held by the Owner, the copy channel and the Provider | None: Connectors add a short-lived SA token to each request on the PM-side |
| Untrusted parties | The network and PM's ingress (TLS passes through to kcp) | The network, the Provider's Edge, the Owner and the Provider's portal; they hold at most a single-use Onboarding Token |
| Rotation | Manual; an open question in RFC 006 | Built in, without re-onboarding |
| Revocation | Rotate or delete the SA token; until then it works from anywhere | Immediate: once Connectors stop proxying, the Provider-side has nothing left |
| Private networks | PM's network team opens a tunnel to kcp for the Provider | The Provider exposes one HTTPS endpoint the PM-side can reach |

```
  ┌─ PM Instance ──────────────────┐                              ┌─ Provider's cluster ─────────────┐
  │ kcp                            │                              │                                  │
  │   Provider Workspace           │                              │                                  │
  │   consumer workspaces          │                              │                                  │
  │                                │                              │                                  │
  │ platform-mesh-operator         │────────┐                     │                                  │
  │   PERs, Connection Secrets     │        │                     │                                  │
  │   Connector CA                 │        ├─▶ Edge ════════════▶│ provider-gateway                 │
  │                                │        │                     │   Server Keys                    │
  │ Connectors, one per shard      │────────┘                     │   Connections (secret hashes)    │
  │   mint SA tokens for kcp       │                              │                                  │
  │                                │                              │ Provider's operator              │
  │                                │                              │   one kubeconfig per Connection  │
  │                                │                              │   CONNECTs to provider-gateway   │
  └────────────────────────────────┘                              └──────────────────────────────────┘
                                   └─────── untrusted path ───────┘
                                         the network, the Edge

    Owner ◀─────────── Onboarding Token, out-of-band ──────────── Provider's portal
      └─▶ a Secret in the Owner's workspace                         └─▶ has provider-gateway issue it

    ──▶ / ══▶   opened only by the PM-side: Registration and Control Calls from platform-mesh-operator,
                Sessions from Connectors. Outer-TLS ends at the Edge, Inner-TLS (pinned) at provider-gateway.

    compromised PM Instance       creates consumers at will, floods the Provider (as today).
    compromised Provider cluster  reads consumers' data within its claims (as today);
                                  nothing reusable once PM severs the Connection.
```

With this RFC, each side gains one new exposure:
- On the Provider-side:
  - **What:** an inbound endpoint for `provider-gateway`.
  - **What can we do about it:** `provider-gateway` authenticates everything itself, and limits are configurable. Beyond that, generic service protection applies.
- On the PM-side:
  - **What:** outbound connections to Owner-supplied Providers.
  - **What can we do about it:** loopback, link-local and cloud-metadata addresses are always refused, and PM Administrators can configure further egress limits.

There are quite a few moving pieces, so it's easier to show the whole architecture and flow as a concrete example.

**The actors:**

| Actor | Side | What it is |
|---|---|---|
| **Owner** | — | A person (or their GitOps) controlling a workspace in PM, here `:root:orgs:acme:alice`. The Provider's customer: they want the Provider's service available in PM. |
| **The Provider's portal** | Provider-side | The Provider's website for its customers. Issues Onboarding Tokens. |
| **`provider-gateway`** | Provider-side | Installed by the Provider from PM's Helm chart, in the Provider's cluster. Serves the Connection API, holds Sessions, and is the proxy through which the Provider's operator reaches kcp. "The SDK" for short. |
| **The Provider's operator** | Provider-side | The Provider's own controllers that reconcile its service (here: the Wild West `Cowboys` API). Unchanged by v5. |
| **The Provider's Edge** | Provider-side | Whatever sits in front of the Provider's public URL: ingress, load balancer, CDN, WAF. Carries traffic, but isn't trusted with it. |
| **`platform-mesh-operator`** | PM-side | PM's operator. Owns every `Provider` object, makes every call to the Provider that isn't traffic, and holds all Connection state. |
| **Connectors** | PM-side | One small proxy Deployment per kcp shard (`providerconnections-connector`). They open Sessions to the Provider and carry all traffic between it and kcp. Stateless. |
| **PM Administrators / Provider Administrators** | either | People running each side. Only needed for rare recovery. |

