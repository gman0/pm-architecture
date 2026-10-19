# Notes v5: ready-made transports for RFC 011

Working notes for [011](./011-provider-onboarding-flow-improvements.md), next to the [v5 notes](./xxx-notes-011-provider-onboarding-flow-improvements-v5.md) and the [implementation plan](./xxx-notes-011-v5-impl-00-overview.md). Not the RFC itself. The question: is there an easier, ready-made, standard solution for v5's tunnelling and crypto that works through standard HTTPS ingress and egress?

**Short answer**: no product covers what v5 needs. v5's transport is already built from standard parts (WebSocket, RFC 6455; TLS 1.3; yamux); most of v5's complexity is in the identity lifecycle around it, which no tunnel product provides. One alternative would change the transport layer itself: SSH over WebSocket. With SSH certificates it removes the most custom protocol but costs the automatic, cert-manager-driven Connector CA; keeping the Connector CA is possible, but then it saves about as much custom protocol as it adds.

## What the transport has to do

From v5's constraints ("Problem", "Trust boundaries", "Sessions"):
- **Pass through any Edge**: a plain HTTPS ingress with WebSocket support, terminated by a load balancer, CDN or WAF that isn't trusted. No TLS passthrough, no UDP, no client certificates at the ingress, no HTTP/2 guaranteed end to end.
- **Be dialled by the PM-side**, which may be behind NAT and only dials out, while **the Provider-side opens the streams**, since it's the client talking to kcp.
- **Encrypt and authenticate end to end, through the Edge**: the PM-side pins the Provider-side's key, and the Provider-side pins PM's CA, so the Edge sees only opaque bytes.
- **Many multiplexed streams, with flow control per stream**, opened on demand, so client-go's standard transport (pooling, health checks, separate connections for upgrades) works unchanged.
- **Rotation without anyone taking part**: Connector credentials renew on their own; a new CA reaches every Provider before it's used.

And around the transport, specific to Platform Mesh:
- **Onboarding** through a Provider's portal and a single-use token; Connections bound to an Owner account;
- **one Provider-side serving many PM Instances**, with state per Connection;
- **an authorizing proxy on the PM-side** that adds SA tokens and applies the Denylist, so the Provider-side never holds a kcp credential;
- **Owners learning nothing about PM's shards.**

## Candidates

| Option | Through the Edge | Reverse streams | End-to-end authentication | What it would replace in v5 | Why not |
|---|---|---|---|---|---|
| **SSH over WebSocket** (`golang.org/x/crypto/ssh`; the shape of chisel and of Teleport's reverse tunnels) | yes: one byte stream in binary messages | yes: the client requests remote forwarding, and the server opens channels back to it | server host key, pinned (as Server Keys are); client by public key or OpenSSH certificate | Inner TLS and yamux, as one audited protocol (and the Hello, with SSH certificates) | SSH certificates mean giving up cert-manager; keeping the X.509 Connector CA needs a binding step and keeps the Hello; see "SSH over WebSocket" |
| **WireGuard through a WebSocket relay** (wstunnel, Tailscale's DERP pattern) | only with UDP encapsulated in TCP | yes, at the network layer | static peer keys (Noise IK) | Inner TLS and yamux | TCP inside WireGuard inside TCP; a userspace network stack in the SDK; per-peer static keys, i.e. the per-Connection key distribution v5 rejected; no CA |
| **Mesh VPNs** (Tailscale or Headscale, NetBird, ZeroTier) | through their relays | yes | the product's own identity | everything at the network level | a coordination server both sides must reach and trust; its identity model doesn't map to Onboarding through a Provider's portal; a third-party or extra control plane per PM Instance |
| **Managed tunnels** (Cloudflare Tunnel, ngrok) | yes | built for the opposite direction | the vendor's | the transport | a vendor dependency, and the vendor's edge becomes trusted, which v5 rules out |
| **HTTP CONNECT, MASQUE (RFC 9298), WebTransport** | needs extended CONNECT or HTTP/3 through the Edge | not standardised in the reverse direction | TLS | the transport | CDNs and ingresses don't reliably pass them |
| **gRPC tunnels** (Konnectivity) | needs HTTP/2 end to end, with trailers and long-lived streams | yes | TLS to the proxy | the transport | often broken by CDNs; the Edge sees plaintext unless another layer is added; as a dial proxy it hands kcp credentials to the Provider-side (v5, "Rejected alternatives") |
| **frp, rathole** | yes | yes (frp multiplexes with yamux too) | a shared token, TLS to the server | the transport | built for a trusted server; no per-Connection identity and no pinning model |

## Why the hard parts stay

Each candidate either moves trust onto the Edge or a third party, or leaves these to be built anyway:
- Onboarding with a single-use token, and Registration protected by the Server Key the token names;
- pinning per Connection, and distributing the Connector CA to every Provider with compare-and-swap;
- the Connection Secret and its changes through the PM-written `ProviderEndpoint`;
- the authorizing proxy with SA tokens, the Denylist and the admission policies in Provider Workspaces;
- Sessions per shard, following kcp's endpoint slices.

They're Platform Mesh's identity and authorization model, not tunnelling.

## SSH over WebSocket

The one option that would make the transport layer itself simpler.

**How it maps:**
- **The Connector is the SSH client**, dialling the Base URL's WebSocket, as today. The SDK is the SSH server.
- **The server's host key is the Server Key**, pinned by the Connector from the `ProviderEndpoint`; `platform-mesh-operator` pins it the same way for control. KMS keys work through `ssh.NewSignerFromSigner`.
- **Streams are SSH channels**: the Connector requests remote forwarding at the start, and the SDK opens a `forwarded-tcpip` channel per client-go connection. The Connector accepts no other channel type. Each channel has its own flow-control window, so a slow reader doesn't stall others.
- **Keepalives and rekeying** are built in.
- **Control Calls** could run as HTTP over a channel, or as SSH requests, with the Connection Secret as today.
- **How the Connector authenticates** decides the rest; there are two variants (below).

**Advantages, whichever variant:**
- **One standard, widely audited protocol** (RFC 4251–4254) for key exchange, encryption, channels, flow control, keepalives and rekeying, instead of Inner TLS with pinning hooks plus yamux. The yamux integration and its rules go, with their test vectors.
- **Reverse streams are part of the protocol**: remote forwarding, and refusing other channel types, replace v5's convention that only the Provider-side opens Streams.
- **Host-key rotation has a standard mechanism**: OpenSSH's `hostkeys-00@openssh.com` (`UpdateHostKeys`) announces new host keys over an authenticated session, the same idea as v5's Server Key announcement in `GET`, already specified and implemented.
- **Familiar operations and tooling**: host-key pinning and SSH CAs are understood by most operators; Sessions can be inspected with standard clients (`ssh` with a WebSocket `ProxyCommand`) and audited with existing tools (`ssh-audit`).
- **Proven in production** for this shape: Teleport's agents and chisel carry reverse SSH tunnels over TLS or WebSocket.
- **Compact Ed25519 keys and certificates**, slightly smaller handshakes against Edge message limits.

**What doesn't get simpler, whichever variant**: the Edge requirements and WebSocket framing; Registration, the Onboarding Token and the Connection Secret; distributing the Connector CA to every Provider; the authorizing proxy, SA tokens, the Denylist, the admission policies and the Session rule per shard.

**More Edges in the way, whichever variant**: SSH's version banner and key-exchange negotiation are plaintext inside the WebSocket, so a WAF that inspects payloads can recognise SSH and block it; the payload-inspection exemption becomes required more often (see "Passing through the Edge").

**A small regression, whichever variant**: yamux has a go-away, which v5 uses for graceful Session replacement and Rollover handovers. SSH has none from server to client: the server can refuse new channels or disconnect, so graceful drain needs a convention, e.g. a global request.

### Variant A: SSH certificates

The Connector authenticates with an OpenSSH user certificate signed by a PM SSH CA, which the Provider-side trusts per Connection (like `TrustedUserCAKeys`).
- **Identity is bound by signature**: `connectionID`, `clusterName` and served hosts can be principals or critical options in the certificate, checked as part of authentication, so the Hello goes away. With one certificate per shard this equals today's Hello check; it's stronger only with certificates per Connection.
- **No cert-manager**: cert-manager issues only X.509. The automatic Rollover (a cert-manager `Certificate` renewed yearly, its `CertificateRequest`s as the source of public certificates, short-lived certificates renewed on their own) would have to be rebuilt, either as an SSH CA operated by PM, whose key a PM component would hold, against v5's rule that only cert-manager holds CA keys, or as an external SSH CA such as Vault's SSH secrets engine, a new required dependency.
- **Thinner tooling** than X.509's: no `CertificateRequest`-style audit trail, and revocation by key revocation lists rather than short lifetimes managed by a standard controller.
- **Or no CA at all**, pinned client public keys per Connector: that brings back the per-shard, per-Connection key registration v5 replaced with the Connector CA (v5, "Session credentials: the Connector CA", options table).

### Variant B: the X.509 Connector CA, bound to the SSH key

The Connector CA stays exactly as v5 has it; only the handshake changes.
- **The Connector's SSH key is its Connector Certificate's key.** cert-manager can issue it as ECDSA P-256 or Ed25519, both SSH key types (`ecdsa-sha2-nistp256`, `ssh-ed25519`).
- **SSH public-key authentication proves possession** of that key, by signing the session identifier, as the TLS handshake does today.
- **The Connector also hands over its X.509 certificate**, and the Provider-side checks what the Inner TLS checks today: issued directly by a CA in the Pinned Bundle, valid, with the `clientAuth` EKU, and its public key is the one that just authenticated. Go's SSH library doesn't implement X.509 in SSH (RFC 6187), so the certificate travels in a step of its own: a second authentication step (`PartialSuccessError`, then keyboard-interactive answered with the certificate), or the first global request, with no channel accepted until it's verified.
- **Everything else in the Connector CA design stays**: cert-manager, the automatic Rollover, the Pinned Bundle with compare-and-swap, distrust, and re-checking open Sessions when the bundle changes, with the certificate kept per Session as today.
- **The Hello stays**, as a global request: Connector Certificates carry no Connection identity, so `connectionID`, `clusterName` and served hosts are still the Connector's own statement, and the check stays essential.
- **So the saving shrinks**: SSH replaces Inner TLS and yamux, but the Hello stays and a certificate-binding step is added. The custom protocol surface ends up about the size of today's, arranged differently.

### Verdict

- **Variant A** removes the most custom protocol, but gives up cert-manager and with it the automatic Connector CA, which is a larger part of v5's goal that nobody takes part in rotations than the transport code it saves. Worth it only if PM adopts an SSH CA for other reasons.
- **Variant B** keeps the Connector CA, but its custom surface is about what v5 has today, plus the go-away convention, for a different protocol underneath. It trades a familiar TLS-and-yamux stack for a familiar SSH stack without making the design smaller.

## Passing through the Edge

The Edge belongs to the Provider, so getting through it is the Provider's configuration, not something to evade: v5 has the Edge checklist ([08 Quickstart, "Connection API server"](./xxx-notes-011-v5-impl-08-provider-quickstart.md)) and the diagnosis labels ([03 Operator, "Edge diagnosis"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)) for that.

| What the Edge does | Remedy | In v5 |
|---|---|---|
| Refuses WebSocket upgrades, or doesn't route the paths | allow upgrades on `/v1/connections/…` | checklist; `EdgeUpgradeRefused` |
| Bot protection (JavaScript challenge, CAPTCHA) | exempt the connection paths | checklist; `EdgeBotChallenge` |
| Inspects WebSocket payloads (WAF) | exempt the paths from payload inspection; the payload is ciphertext anyway | checklist; `EdgePayloadBlocked` |
| Limits message size | messages of at most 32 KB | framing rule |
| Idle timeout | outer pings every 30 seconds | framing rule; `EdgeTimeout` |
| Maximum connection age | Connectors replace Sessions before it | Connector; `EdgeTimeout` |
| Limits connections per client address | allow R × shards Sessions per Connection from PM's egress addresses | checklist |
| Message-based WebSocket gateways (cloud "WebSocket API" products) | a plain load balancer or ingress | checklist |
| Re-signs TLS with its own CA (an inspecting proxy) | the Owner adds that CA to `caBundle`; the outer TLS isn't the trust anchor | `EdgeTLS` |

**With SSH**, the version banner (`SSH-2.0-…`) and the key-exchange negotiation are plaintext inside the WebSocket. A WAF that inspects WebSocket payloads may recognise SSH and block it; many block SSH on principle. The options:
1. **Exempt the paths from payload inspection**: the same checklist item as today, and the honest solution; but with SSH, more Edges block until it's configured.
2. **Wrap SSH in TLS inside the WebSocket**: it then looks like any TLS stream, at the cost of TLS, then SSH, then channels, more layers than v5's Inner TLS and yamux, which gives up SSH's main advantage.
3. **Obfuscate the stream** (obfs4-style padding or XOR): not recommended. It's non-standard, looks like evading the Provider's own controls, and reviewers on both sides would object.

Inner TLS has the practical advantage: inside the WebSocket it looks like ordinary TLS records, the most common thing an Edge sees, and nothing in it names a protocol a WAF would block on principle.

**If WebSockets aren't allowed at all:**
- **HTTP streaming or long-polling pairs** (a long `GET` for down, chunked `POST`s for up, as Socket.IO and SignalR fall back to): passes Edges without WebSocket support, but buffering proxies break streaming, latency rises, and it's a second transport to build, test and diagnose;
- **HTTP/2 extended CONNECT (RFC 8441), HTTP/3 or WebTransport**: fewer Edges pass these than WebSockets.

Neither is proposed: WebSocket support is a stated prerequisite (v5, "Provider-side prerequisites"), and nearly every ingress, load balancer and CDN supports it. A fallback transport is worth considering only if real Providers can't meet it.

## Recommendation

Keep v5's transport. It's built from standard parts that work through any WebSocket-capable Edge, and the parts that make v5 large are ones no ready-made transport provides. Revisit SSH over WebSocket (variant A) only if PM adopts an SSH CA for other reasons.

## Prior art

- **Teleport** reverse tunnels: agents dial the proxy and carry SSH over TLS or WebSocket, with SSH certificates from Teleport's own CA; Kubernetes access with identity injected by the proxy.
- **chisel** (`jpillora/chisel`): TCP tunnels over HTTP, secured with SSH over WebSocket, with reverse forwarding and server-fingerprint pinning.
- **wstunnel**, **Tailscale DERP**: WireGuard or TCP carried over WebSocket or HTTPS relays.
- **frp**, **rathole**: reverse proxies for exposing services behind NAT, with yamux multiplexing (frp).
- **Konnectivity**: a gRPC dial proxy for the Kubernetes control plane.
- **Cloudflare Tunnel**, **ngrok**: managed outbound tunnels.
- **RFC 9298** (MASQUE, proxying UDP in HTTP) and **RFC 8441** (WebSockets over HTTP/2).
