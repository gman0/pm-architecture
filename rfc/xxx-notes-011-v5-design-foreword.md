# RFC 011: a structure for the Design foreword (draft)

Working draft, not RFC text: a proposed order for the part of "Design" that comes before the flow, with the diagrams for it. Each subsection answers one question and introduces only the terms that question needs, so that by the time the reader reaches the Edge and the two TLS layers, they already know who talks to whom and what's being protected. The API shapes are in the [flow draft](./xxx-notes-011-v5-api-shapes.md).

| # | Subsection | Question it answers | Diagram |
|---|---|---|---|
| 1 | Shape | What runs where, and who connects to whom? | D1: components |
| 2 | Three layers | What kinds of interaction are there? | table |
| 3 | Where the APIs live | Which object sits where, and who writes it? | D2: API topology |
| 4 | Trust | Who verifies whom, and with what? What can't the Provider do? | D3: credentials and pins |
| 5 | The Edge | Why two TLS layers, and what each hop sees | D4: layers |
| 6 | Shards | Where Sessions open, and why | D5: Sessions per shard |
| 7 | Principles | The rules everything above follows | — |

---

## 1. Shape

What to say:
- **The PM-side opens every connection.** It may sit behind NAT, so the Provider-side can't reach it; the Provider-side only needs an HTTPS endpoint that passes WebSockets.
- **Three components.** On the PM-side, `platform-mesh-operator` holds all Connection state and makes every call that isn't traffic, and one Connector per kcp shard carries the traffic. On the Provider-side, `provider-gateway` accepts both and is the proxy the Provider's operator uses.
- **The Provider-side never holds a kcp credential.** The Connector adds a short-lived token to each request on its way to kcp.
- **Availability.** `platform-mesh-operator` isn't on the traffic path, and Connectors are stateless and need only kcp: an outage of the former delays onboarding and changes, but doesn't stop traffic.

D1:

```
 PM Instance (may be behind NAT)                      Provider's cluster
┌──────────────────────────────┐                     ┌───────────────────────────────┐
│ platform-mesh-operator       │── Registration ────▶│ provider-gateway              │
│   one active replica,        │                     │   Connection API server,      │
│   all Connection state       │── Control Calls ───▶│   forward proxy               │
│                              │                     │                               │
│ Connector @ shard A          │── Sessions ────────▶│                               │
│ Connector @ shard B          │── Sessions ────────▶│         ▲                     │
│   stateless, one per shard,  │                     │         │ CONNECT             │
│   adds the SA token          │                     │ Provider's operator           │
│        │                     │                     │   no protocol code; one       │
│        ▼                     │                     │   kubeconfig per Connection   │
│ kcp                          │                     │                               │
└──────────────────────────────┘                     └───────────────────────────────┘

 every arrow between the two sides is opened by the PM-side; the Edge is left out (D4)
```

## 2. Three layers

| Layer | Between | Carries | Authenticated by | How often |
|---|---|---|---|---|
| **Kubernetes APIs** | the Owner, PM's components and the Provider, through kcp and the two clusters | what each side wants and knows: `Provider`, request, `ProviderEndpoint`, `Connection`, … | RBAC, plus PM's protections in Provider Workspaces | continuously |
| **Control** (Registration, Control Calls) | `platform-mesh-operator` ↔ `provider-gateway` | the Connection's lifecycle: register, status, Connector CA pushes, delete | the Onboarding Token, then the Connection Secret, inside Inner TLS | rarely |
| **Session** | Connectors ↔ `provider-gateway` | the Provider's requests to kcp | Connector Certificate ⇄ Server Key, Inner TLS | continuously |

What to say with it:
- **Control and traffic are separate**: different components, different credentials, and a failure in one doesn't stop the other.
- **The Kubernetes APIs are where the sides meet.** The two wire layers only carry out what those objects say.

## 3. Where the APIs live

D2 (PM = `platform-mesh-operator`; cert-manager and the portal are named where they write):

```
┌─ Owner's workspace ─────────────────┐  ┌─ :root:providers:system ────────────┐  ┌─ Provider Workspace ────────────────┐
│ Provider                      Owner │  │ Connection Export                PM │  │ ProviderEndpoint            PM only │
│ ProviderOnboardingRequest     Owner │  │   serves ProviderEndpoint           │  │ APIExports, schemas        Provider │
│ token Secret                  Owner │  │   claims endpoint slices            │  │ endpoint slices                 kcp │
└─────────────────────────────────────┘  └─────────────────────────────────────┘  └─────────────────────────────────────┘
┌─ PM's runtime cluster ──────────────┐  ┌─ Provider's cluster ────────────────┐
│ ProviderEndpointRegistration     PM │  │ Connection                  gateway │
│   + its Secret                   PM │  │ OnboardingToken              portal │
│ ConnectorCASet                   PM │  │ ServerKey                   gateway │
│ Connector CA           cert-manager │  │ kubeconfig Secrets          gateway │
│ Connector Certificates cert-manager │  └─────────────────────────────────────┘
└─────────────────────────────────────┘                                         
```

What to say, the *why* of each placement:
- **The Owner's workspace holds no Connection credential**: only the token, whose data is deleted once spent. Other Providers with claims there, and the Owner, can't reach the Connection Secret.
- **The `ProviderEndpoint` lives in the Provider Workspace**, served by PM's Connection Export, next to the shard that hosts it, so Connectors find everything they need in kcp. Only PM can write it.
- **The Connection Secret stays in PM's runtime cluster**; the Provider-side keeps only its hash.
- **All Provider-side state is keyed by the Connection**, which is what lets one Provider serve many PM Instances and Owners without mixing them up.
- **Owners see aggregate conditions** on their `Provider`, never PM's shards.

## 4. Trust

D3:

```
  PM-side                                                             Provider-side

  Connector CA (cert-manager, renewed yearly)
     │ issues; 24 h each
     ▼
  Connector Certificate     ═══ presented on every Session ════════▶  verified against the Pinned Bundle
  Connector CA certificate  ─── sent at Registration, then pushed ─▶  Pinned Bundle, per Connection

  pinned Server Keys        ◀═══ presented in every Inner TLS ══════  Server Key (provider-gateway)
    the first one from the Onboarding Token

  Connection Secret         ═══ bearer on every Control Call ══════▶  checked against its hash,
    (platform-mesh-operator)                                          sent at Registration

  ═══▶ proven on every connection      ───▶ delivered, then pinned
```

| Credential | Held by | Proves | Lifetime |
|---|---|---|---|
| Onboarding Token | the Owner, until PM spends it | the Owner's right to register one Connection | minutes, single use |
| Server Key | `provider-gateway` | "this is the Provider-side PM registered with" | until the Provider rotates it |
| Connector Certificate | each shard's Connector | "this is one of this PM Instance's Connectors" | 24 hours |
| Connector CA | cert-manager | issues Connector Certificates; pinned per Connection | 1 year, renewed automatically |
| Connection Secret | `platform-mesh-operator` | "this Control Call comes from PM" | until PM changes it |
| SA token | Connectors, in memory | the Provider's identity towards kcp | minutes, never leaves the PM-side |

What to say:
- **The Connector CA is renewed automatically**, and Connectors only use a new one once every Connection trusts it, so a renewal never disrupts a healthy Connection. CA updates travel only as Control Calls, never over a Session.
- **The Connection Secret changes only through the `ProviderEndpoint`**, which only PM can write; no call, token or portal account can change it.
- **What an admin Provider can't do.** It's admin in its workspace, yet it can't write its `ProviderEndpoint` (the Connection Export's maximal permission policy), can't hand anyone a second way into kcp (admission policies on role bindings and token `Secret`s), and can't do the few operations the Connector refuses (removing PM's binding or policies, minting tokens). That's what keeps "kcp credentials stay on the PM-side" true.

## 5. The Edge

D4, replacing the current one-picture version: the nesting first, then the in-cluster hop, then who sees what.

```
Connector ══ outer TLS (web PKI or caBundle) ══▶ Edge ──▶ provider-gateway
          ╚═ WebSocket  GET …/v1/connections/{id}/session ═══════════════╝
               ╚═ inner TLS 1.3: Connector Certificate ⇄ Server Key ═════╝
                    ╚═ Hello, then Streams: plain HTTP to kcp ═══════════╝

Provider's operator ══ CONNECT, TLS to the gateway's own certificate ══▶ provider-gateway ── Stream ──▶ …

  the Edge sees:                 the path, then opaque bytes
  acts as the Provider in kcp:   only the Provider's runtime (provider-gateway, and the operator through it)
  can fake kcp to the Provider:  nobody without a Connector Certificate
  sees the Connection Secret:    nobody (Control Calls run inside the inner TLS too)
```

What to say:
- **The Edge carries the Connection without being part of it.** It may be a third party's CDN, a shared ingress, or just keep logs; PM and consumers can't judge it, so nothing depends on it.
- **What would otherwise cross it**: the Provider's authority over every consumer's resources in kcp, kcp's view the Provider acts on, the credentials it writes back for its consumers, and the Connection Secret.
- **No TLS passthrough is required**: most CDNs and managed load balancers can't do it, and it would still rest on web PKI.

## 6. Shards

D5:

```
  Provider Workspace on shard A; consumers of its export on shards A and C

  Connector @ A ═══ Session ══▶ provider-gateway    the workspace lives here: always, from Onboarding on
  Connector @ B                                     no Session: no consumers on B
  Connector @ C ═══ Session ══▶ provider-gateway    consumers on C, as kcp lists them in the endpoint slice
```

What to say:
- **Sessions follow traffic**: the Hosting Shard's Connector from Onboarding on, others only once the Provider has consumers there, as kcp itself lists them.
- **Nothing is registered per shard**, and adding a shard involves no Provider.

## 7. Principles

The four already in the RFC, plus the ones the sections above establish:
- **The PM-side opens every connection.**
- **Everything that grants anything travels inside Inner TLS.** Only WebSocket upgrades are plain HTTP, and they carry no credentials.
- **kcp credentials stay on the PM-side.**
- **Nothing outside PM and the Provider's runtime can take over a Connection**: not a stolen Onboarding Token, a compromised portal account, or the Edge.
- **Revocation is immediate**: when the Connectors stop proxying, the Provider-side has nothing left to use.
- **Credentials renew without anyone's involvement.**
- **Owners learn nothing about PM's topology.**

---

## Left for elsewhere

- Versioning, observability, limits: the protocol and component details, not the foreword.
- Recovery and incident response (lost state, distrusting a CA, abandoning a Connection): RFC 012.
