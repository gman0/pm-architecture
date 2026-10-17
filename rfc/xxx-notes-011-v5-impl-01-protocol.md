# v5 implementation plan: protocol

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). The wire contract between the PM-side ([platform-mesh-operator](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md), which makes every Registration and Control Call, and the [Connectors](./xxx-notes-011-v5-impl-05-connector.md)) and the Provider-side (the [SDK](./xxx-notes-011-v5-impl-07-provider-sdk.md)). Both sides implement exactly this document; neither relies on anything the other does beyond it.

## Principles

- **Everything that grants anything travels inside Inner TLS.** Only WebSocket upgrades are plain HTTP, and they carry no credentials.
- **The Provider-side is the verifier of every PM credential**; PM verifies the Provider-side by pinned Server Keys, the first of which comes in the Onboarding Token. Neither side needs the other's CA infrastructure beyond pinned material.
- **Every request is idempotent or safely retryable**, and responses carry nothing secret.
- **The protocol is versioned** from the first release, and every message is forward-compatible (unknown fields ignored).

## Endpoints at a glance

All paths are relative to the Provider's **Base URL** from the Onboarding Token.

| Endpoint | Transport | Caller | Authenticated by | Purpose |
|---|---|---|---|---|
| `GET /v1/connections/register/{tokenID}` | WebSocket + Inner TLS | `platform-mesh-operator` | Inner TLS: the Server Key named in the token (server); `tokenSecret` inside (client) | Register a Connection |
| `GET /v1/connections/{id}/control` | WebSocket + Inner TLS | `platform-mesh-operator` | Inner TLS: pinned Server Key (server); Connection Secret inside (client) | Control Calls |
| `GET /v1/connections/{id}/session` | WebSocket + inner mutual TLS | Connector | Inner TLS: Connector Certificate (client), pinned Server Key (server) | A Session carrying the multiplexer |

Every connection to the Base URL has an outer TLS, verified strictly with web PKI, or with the `Provider`'s `spec.remote.caBundle` when set; Connectors get it from the `ProviderEndpoint`. The outer TLS isn't the trust anchor: it's terminated by the Provider's Edge, and what protects Registration, Control Calls and Sessions is the Inner TLS. But it has to succeed for anything to connect. **The Inner TLS checks no hostnames, on either side**: Server Keys, the Connector CA and Connector Certificates are independent of any URL, so neither a PM-side rename nor a change of a Provider's Base URL touches them.

The three WebSocket endpoints accept the upgrade before the caller is authenticated; authentication happens in the Inner TLS (Sessions) or on each request inside it (Registration, Control Sessions). Before the upgrade, the Provider-side refuses:
- **on `register`**: an unknown, expired or spent token, or one whose Server Key is missing, not Ready or has `retireAt` set (see "Registration");
- **on `session`**: unknown or deleted Connection IDs;
- **on all three**: callers beyond the handshake limits, with `503` and `Retry-After`.

`control` accepts the upgrade for unknown IDs too, so that "unknown" is answered inside the Inner TLS, where it can be trusted (see "Errors and retries").

**Handshake limits**: a counter per Connection and a total for known IDs, plus a separate, small total for unknown IDs on `control` (e.g. 2 concurrent, 1 per second). There's no limit per source address: behind the Provider's Edge, all callers share a few. A slot is held from the upgrade until the handshake ends, and each handshake has a short timeout (a few seconds). Connection IDs are random and unguessable, so probing can't find valid ones, but they aren't secret: Owners, Provider Workspaces and the Provider's Edge all see them.

## Onboarding Token

- **Format**: `pm1.<base64url(base URL)>.<keyfp>.<tokenID>.<tokenSecret>`. The Base URL inside ensures the token is only ever sent to its issuer. `keyfp` is the `key-sha256` of a Server Key (the SHA-256 of its `SubjectPublicKeyInfo`, DER), which `platform-mesh-operator` pins for the Registration (see "Registration").
- **Bound to one Server Key**, chosen when the token is issued. Tokens are issued and used only over Server Keys without `retireAt`, so a Server Key rotation refuses tokens issued before it: the Owner asks the Provider's portal for a new one.
- **Single-use and short-lived**; the Provider-side stores only a hash of `tokenSecret`.
- **Bound to the issuing Owner account** on the Provider-side, which owns the Connection it registers (portal listing, billing).
- **Spent on first successful Registration**, but a retry with the same idempotency key and Connection Secret hash within a window (7 days by default) returns the same result (see "Registration").
- **Its delivery path must be trusted.** The token reaches `platform-mesh-operator` through the Provider's portal and the Owner. Whoever reads it there before PM uses it can register in the Owner's name; PM's own Registration then fails as spent, and the Owner deletes that Connection in the portal and asks for a new token.

## Connection Secret

- **Generated by the PM-side**: prefix plus 256 random bits, `pmc1_<base64url>`. The prefix lets secret scanners recognise it.
- **Only its hash leaves the PM-side** (`sha256:<hex>`): in the Registration request, and in the Connection's `ProviderEndpoint` when it changes (see "Changing the Connection Secret"). High entropy makes a plain SHA-256 sufficient.
- **Used only inside Control Sessions**, never on plain HTTPS.
- **Stored by the PM-side outside kcp**, in `platform-mesh-operator`'s namespace in PM's runtime cluster ([02 APIs, "Its `Secret`"](./xxx-notes-011-v5-impl-02-apis.md)). The Provider-side stores only the current hash.

## Changing the Connection Secret

The Connection Secret changes only through the Connection's **`ProviderEndpoint`**, never through a call:
1. `platform-mesh-operator` generates a new Connection Secret, stores it, and writes its hash into the `ProviderEndpoint`'s `pendingSecretHash`.
2. The Provider-side reads the `ProviderEndpoint` over one of the Connection's Sessions and makes that hash the current one. The old Connection Secret stops working.
3. `platform-mesh-operator` confirms with a Control Call using the new Connection Secret, then clears `pendingSecretHash`.

Why this is safe:
- **Only PM can write the `ProviderEndpoint`**: the Connection Export's maximal permission policy refuses writes even to the Provider Workspace's admins ([02 APIs, "Connection Export"](./xxx-notes-011-v5-impl-02-apis.md)).
- **The Provider-side reads it only over a valid Session**: one whose Connector Certificate verified against the Connection's Pinned Bundle, and whose Hello named the Connection's `connectionID` and `clusterName` (see "Sessions").
- **The Provider-side applies it only to its own Connection**: a `ProviderEndpoint` whose `connectionID` isn't the Connection's is ignored.
- **So no call, Onboarding Token or portal account can change the Connection Secret.**

**One exception: a leaked Connection Secret.** It only exists on the PM-side, so whoever holds one got it from a PM-side leak. With it they can push a Pinned Bundle with a CA of their own (see "Pinned Bundle"), and through a Session under that CA change the Connection Secret, which locks PM out of that one Connection. It gives no kcp access. If `platform-mesh-operator`'s tamper check gets there first, it restores the bundle and changes the Connection Secret; otherwise the `Provider` shows `SecretRejected`, and recovery is out-of-band: the Provider Administrators re-pin the bundle and `platform-mesh-operator` repairs the Connection Secret, or the Owner abandons the Connection and onboards again. It's handled like any other PM-side secret leak: find the leak and rotate PM's credentials ([03 Operator, "Secret Changes"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).

It needs a working Session. Without one, the change waits; the Provider-side keeps the current hash meanwhile.

## Registration

A WebSocket upgrade on `/v1/connections/register/{tokenID}`, then a TLS 1.3 handshake inside it, like a Control Session's (see "Control Sessions"):
- **Server**: the Provider-side presents a self-signed certificate for the token's Server Key; `platform-mesh-operator` accepts only the key whose `key-sha256` is the token's `keyfp`.
- **Client**: no client certificate. The request inside carries `tokenSecret` as a bearer credential.
- **ALPN**: `pm-register/1`.
- **Inside**: one HTTP/1.1 request, then the Registration Session closes.

Before the upgrade, the Provider-side checks the token by `tokenID`: it must be known, unexpired and unspent, and its Server Key present, Ready and without `retireAt`. Otherwise the upgrade is refused with a stable status, which `platform-mesh-operator` reports as `TokenRefused`. Since the key is checked before the upgrade, a refused token is never confused with a handshake failure.

Request: idempotency key (the `ProviderOnboardingRequest` UID), `clusterName` (the logical cluster name of the Provider Workspace, as kcp assigns it), the Connection Secret's hash, and the Connector CA Set, which becomes the Connection's Pinned Bundle (see "Pinned Bundle"). Its version is the path's (`/v1`).

Response: `connectionID`, the Provider-side's **Server Keys** (public keys, a list to allow rotation), and `unusedExpiresAt` (see "Orphans expire").

Rules:
- **Only new Connections.** Registration never changes an existing Connection.
- **Idempotency**: a retry matches on `tokenID`, idempotency key, `secretHash` and `clusterName`, and gets the same response (`connectionID`, Server Keys and `unusedExpiresAt`); anything else is refused. A changed Connector CA Set on a retry is ignored: the first Control Call pushes the current set anyway ([03 Operator, "First Control Call and ProviderEndpoint"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)). After the token's expiry, a retry is still honoured for a **generous window, 7 days by default**, so an operator that crashed between the call and persisting its result can always recover it.
- **Orphans expire**: a Connection that has seen neither an authenticated Control Call nor a verified Session by the end of that window is removed by the Provider-side. It's meant for a Registration whose result PM never stored. The same happens when PM stored the result but couldn't reach the Provider for the whole window (its control path blocked, or `platform-mesh-operator` down); nothing is lost, since the Connection never had a Session, and the Owner onboards again once the path works ([03 Operator, "Health"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)). The deadline, `unusedExpiresAt`, is fixed by the Provider-side when it creates the Connection and returned with it, so `platform-mesh-operator` can warn the Owner. A Connection that was used once is never expired for being idle.
- **Addressing is immutable**: `clusterName` is fixed for the Connection's life.
- **Trust in the response** comes from the Inner TLS pinned to the token's `keyfp`; it's what makes the returned Server Keys safe to pin. The Provider's Edge sees neither `tokenSecret`, the Connection Secret's hash nor the Connector CA Set, and can't change the response.

## Control Sessions

A WebSocket upgrade on `/v1/connections/{id}/control`, then a TLS 1.3 handshake inside the WebSocket:
- **Server**: the Provider-side presents a self-signed certificate for one of its Server Keys; `platform-mesh-operator` accepts only keys pinned for this Connection. The upgrade carries the `key-sha256` of the key `platform-mesh-operator` expects, in the `PM-Server-Key` header, and the Provider-side presents that key if it has it; this is what lets it answer an unknown Connection inside the Inner TLS.
- **Client**: no client certificate. `platform-mesh-operator` sends the Connection Secret as a bearer credential on each request inside the Inner TLS. A client certificate can't be used: the only candidate is one from the Connector CA, and Control Calls must not depend on the Connector CA, since they're what brings a Provider with an outdated bundle up to date.
- **Inside**: HTTP/1.1 with keep-alive, so one Control Session can carry several calls in a row. ALPN names the control protocol and its version (e.g. `pm-control/1`).
- **Bound to the outer path's `{id}`.** An inner request for another Connection is refused with `ConnectionMismatch` and closes the Control Session. It isn't an authentication failure: Server Keys are per Provider, not per Connection, so this is what an Edge rewriting `/connections/A/control` to `/connections/B/control` causes, and `platform-mesh-operator` reports it as `EdgePathRewritten`.
- **First failed authentication closes the Control Session.** Every attempt with a wrong Connection Secret therefore costs a new WebSocket and handshake, which the limits on handshakes govern; `platform-mesh-operator` never fails authentication on purpose.

Control Calls:

| Call | Effect |
|---|---|
| `GET /v1/connections/{id}` | Status (see "Status"), Server Keys, and `pinnedBundle`: the Pinned Bundle's `etag` and fingerprints |
| `PUT /v1/connections/{id}/connector-ca-set` | Replace the Pinned Bundle (Rollover), with `If-Match`; see "Pinned Bundle" |
| `DELETE /v1/connections/{id}` | Delete the Connection; all its Sessions close |

There's no call that changes the Connection Secret (see "Changing the Connection Secret").

### Pinned Bundle

The Connector CA certificates a Connection pins travel as a **PEM bundle**, like every other CA bundle in Kubernetes: one `CERTIFICATE` block per CA of PM's **Connector CA Set**, the current Connector CA, and during a Rollover the previous ones ([04 Connector CA, "The Connector CA Set"](./xxx-notes-011-v5-impl-04-connector-ca.md)). The rationale, including how this compares with other trust-anchor APIs, is in v5, "Distributing the Connector CA to Providers".

- **Sent at Registration** and **replaced by `PUT …/connector-ca-set`** whenever the Connector CA Set changes, which is a few times a year. Both carry the Connector CA Set as a JSON string field, `connectorCASet`; the Provider-side stores it as the Connection's Pinned Bundle.
- **Parsed strictly**: only `CERTIFICATE` blocks, without PEM headers; only whitespace before, between and after blocks; no duplicate certificates; every certificate a CA (`basicConstraints CA:true`, `keyCertSign`); at least one certificate, and at least one unexpired when it's received; bounded number of certificates and total size. Anything else is refused as a whole, with a stable reason. The same rules apply to Registration, pushes and administrators' re-pins.
- **Fingerprints**: `cert-sha256:<hex>` is the SHA-256 of a certificate's DER, used for the `etag`, in `GET` and in the tamper check; `key-sha256:<hex>` is the SHA-256 of its `SubjectPublicKeyInfo` (DER), used where a key matters more than a certificate, e.g. distrusting a CA ([04 Connector CA, "Compromise response"](./xxx-notes-011-v5-impl-04-connector-ca.md)).
- **Compare-and-swap**: `GET` returns, in `pinnedBundle`, the Pinned Bundle's `etag` (a hash over the sorted `cert-sha256` fingerprints of its certificates, computed by the Provider-side) and the CAs' fingerprints. A `PUT` must carry `If-Match: <etag>`; a mismatch is refused with `412` and the current `etag`, and a bundle with the same set of certificates as the pinned one is a no-op, whatever its order or whitespace. Reordered or duplicated calls, for example from two operator workers during a handover, therefore can't overwrite a newer bundle unnoticed.
- **PM can always restore its bundle**, even after someone holding the Connection Secret replaced it (see [03 Operator, "Connector CA Set"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
- **Accepted even if every pinned CA has expired.** A push is authenticated by the Connection Secret and protected by compare-and-swap; the pinned CAs play no part. A Provider that was down through any number of Rollovers is caught up by its first successful Control Call.
- **Only on the control channel**, never through a Session: a Session is authenticated by the CA it would be changing.
- **Open Sessions are checked against a new bundle.** Sessions are only verified when they open, so on its own, removing a CA wouldn't affect Sessions already opened under it. When the Provider-side accepts a new bundle (a push, or a re-pin by its administrators), it checks each open Session of that Connection: if the Session's client certificate, kept from its handshake, no longer chains to a CA in the new bundle, the Provider-side sends go-away, lets in-flight requests finish within a short grace period (e.g. 30 seconds), then closes the Session. Only the issuer counts, not the certificate's expiry, so Sessions still outlive an expired Connector Certificate. A bundle that only adds CAs closes nothing. PM's Connectors reconnect at once with a certificate from a trusted CA; anyone else can't.

  Why: someone who gets a Connector CA's key or a Connector Certificate's key on its own (from a compromised external PKI, or a single `Secret` leaked in a backup or log) can open Sessions and act as PM's Connector, feeding the Provider made-up objects and receiving what it writes back, without any kcp access. PM's response is to renew the CA and distrust the leaked one; this rule is what ends the Sessions already opened with it. It doesn't help against an attacker who still has access to PM's runtime cluster, who would also have kcp access and could do it again. In a normal Rollover, this is also how Sessions still on the old CA's certificate move to the new one: `platform-mesh-operator` spreads the old CA's drop over several hours, so the handovers, and the relists of long-running watches on those Sessions, are spread too ([03 Operator, "Connector CA Set"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).

## WebSocket framing

Applies to Registration, Control Sessions and Sessions alike, so that they pass common edges (ingresses, load balancers, CDNs, WAFs) unchanged:
- **Binary messages only.** Everything inside is TLS ciphertext; text messages are a protocol error and close the WebSocket.
- **At most 32 KB per message**, in both directions. TLS records are at most 16 KB anyway; the cap keeps every message under common Edge limits. A larger message is a protocol error.
- **Pings at the outer level** at a fixed interval (e.g. 30 seconds), so idle timeouts above that never fire.
- **No reliance on message boundaries**: the Inner TLS is a byte stream over the messages; either side may split or coalesce.
- **Only the upgrade request is plain HTTP.** It's an ordinary `GET` with the `Upgrade: websocket` handshake, from a non-browser client: no cookies, no JavaScript. Edges must not answer it with a bot challenge.

## Diagnosing the Edge

Most Onboarding problems come from the Provider's Edge, not the protocol. The PM-side classifies failures on Registration, on the first Control Session after Registration, and on every reconnect, and reports them with distinct reasons ([03 Operator, "Health"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)):
- **The outer TLS failed**: the server certificate isn't trusted by web PKI or the `Provider`'s `caBundle`, typically a private CA the Provider rotated before the Owner added it to `caBundle`;
- **upgrade refused** (a non-`101` response), including **an HTML page in response**, the typical sign of a bot challenge or WAF block page. A non-`101` response before the upgrade is always reported as the Edge's, never as an answer from the Provider-side;
- **The WebSocket closed right after a binary message**, the typical sign of a WAF inspecting WebSocket payloads;
- **The WebSocket closed at a regular age or after a regular idle time**, the sign of an Edge timeout below what the protocol needs;
- **`ConnectionMismatch`** inside a Control Session: the Edge rewrote the path to another Connection (see "Control Sessions").

An Inner TLS handshake that fails against the pinned Server Keys means the Provider-side didn't answer: it's never reported as an authentication failure or an unknown Connection.

Providers check their Edge against the quickstart's checklist ([08 Quickstart, "Connection API server"](./xxx-notes-011-v5-impl-08-provider-quickstart.md)); a self-test tool for Providers is a [follow-up](./xxx-notes-011-v5-impl-todo.md).

## Sessions

### Sessions and Streams

A **Session** is a long-lived tunnel between PM and a Provider; **Streams** are the lightweight connections running inside it.

| | Session | Stream |
|---|---|---|
| **What it is** | One WebSocket, with Inner TLS and the yamux multiplexer inside | One logical connection inside a Session, created by yamux |
| **Who opens it** | PM: a Connector Replica dials the Provider | The Provider-side: the SDK opens one whenever client-go wants a new connection to kcp |
| **For what** | One Connection (one Provider Workspace), on one shard | One HTTP connection from client-go: requests one after another, or one long watch |
| **Authentication** | Once, when it opens: the Inner TLS handshake, then the Hello | None of its own: it inherits the Session's. The Connector authorizes each request on it |
| **Cost to open** | A WebSocket upgrade through the Provider's Edge, plus a TLS handshake | A yamux frame |
| **Lifetime** | Hours; replaced before the Edge's maximum connection age | Seconds for requests (or reused with keep-alive), hours for a watch |
| **How many** | One per (Connection, shard with traffic, Connector Replica): typically single digits per Connection | One per concurrent request or watch: typically tens per Provider |
| **Visible at the Provider's Edge** | Yes, as one WebSocket connection | No, only as opaque bytes inside the Session |
| **If it ends** | All its Streams end; client-go reopens them on another Session | Only that request or watch; client-go retries or re-watches |

It's like an HTTP/2 connection and its streams, with one difference in direction: PM sets up the Session, because the Provider can't reach PM, but the Provider opens the Streams, because it's the client talking to kcp.

**How they fit together in practice.** The Provider's controller wants to watch `cowboys` on shard B. client-go asks the SDK for a connection to shard B's virtual-workspace host. The SDK picks one of the Sessions serving that host and opens a Stream on it. client-go sends `GET …/cowboys?watch=true` over that Stream; the Connector authorizes it, adds the SA token and forwards it to kcp; the watch events stream back on the same Stream for hours. Other watches and requests on shard B get their own Streams, on the same Session or another one serving that host.

### Upgrade

A plain WebSocket upgrade on `/v1/connections/{id}/session`, without credentials. Unknown Connection IDs are refused, and the handshake limits apply (see "Endpoints at a glance").

### Inner layer

After the upgrade, a TLS 1.3 handshake inside the WebSocket, with mutual authentication:
- **Client**: the Connector presents its Connector Certificate. The Provider-side verifies it against **only** the CA certificates in the Connection's Pinned Bundle: the certificate must be issued directly by one of them (no intermediates), be valid, with the pinned CA itself unexpired too, and allow signing and client authentication (key usage `digitalSignature`, EKU `clientAuth`). Go's verifier requires EKU `serverAuth` unless told otherwise, so the SDK passes `clientAuth` explicitly.
- **Server**: the Provider-side presents a self-signed certificate for one of its Server Keys; the Connector accepts only keys pinned for this Connection (read from the `ProviderEndpoint`, see [02 APIs](./xxx-notes-011-v5-impl-02-apis.md)).
- **ALPN** names the Session protocol and version (e.g. `pm-session/1`), so a future change is negotiated, not guessed.

That handshake is the whole authentication; no separate token is needed:
- **Possession of the key**: the Connector signs the handshake transcript with its certificate's key.
- **No replay**: the transcript includes fresh random values from both sides, so a recorded handshake is useless for a new one.
- **Bound to this Provider**: the transcript includes the Provider's server certificate, and the Connector only accepts the Server Key pinned for this Connection. A Provider that forwarded the Connector's handshake to another Provider would present the wrong key.

### Hello

The first message inside the Inner TLS, before the multiplexer starts, sent by the Connector:
- **`connectionID`**: the Connection this Session is for, from the `ProviderEndpoint` the Connector acts on.
- **`clusterName`**: the Provider Workspace this Session is for. The Connector takes it from the logical cluster of that `ProviderEndpoint`.
- **`hosts`**: the hosts this Session serves (see "Hosts a Session serves").
- **`sub`**: an opaque Session-group ID, for audit.

It's protected by the Inner TLS, so it needs no signature. The Provider-side checks that `connectionID` equals the outer path's `{id}` and that `clusterName` equals the one stored at Registration, and closes the Session otherwise. **This check is essential, not defense in depth**: the Inner TLS proves only that the Connector belongs to PM, since one Connector Certificate serves every Connection of a PM Instance. Without the check, an Edge that rewrote `/connections/A/session` to `/connections/B/session` would get a Session meant for one Connection attached to another, and a PM-side bug that mixed up `ProviderEndpoint`s could make a Connector serve one Connection's Session with another Provider Workspace's SA. A length-prefixed message, with a small maximum size.

### Multiplexer

- **yamux** runs inside the Inner TLS.
- **Only the Provider-side opens Streams.** The Connector refuses Streams it didn't expect and never opens any.
- **Each Stream carries plain HTTP/1.1 from its first byte.** The Connector routes each request by its path and checks it before forwarding it ([05 Connector, "Request pipeline"](./xxx-notes-011-v5-impl-05-connector.md)). The Provider-side uses the target address only to choose which Session to open the Stream on.
- **Flow control per Stream** is required: a slow reader on one Stream must not stall others or force unbounded buffering.
- **Go-away from either side**: the Connector signals graceful replacement with yamux's go-away, and the Provider-side may send it too (e.g. when a Connection moves between its replicas). On go-away from the Connector, the Provider-side stops opening Streams on that Session and lets in-flight ones finish.
- **Liveness**: WebSocket pings at the outer level (so ingresses see traffic), yamux keepalives inside.

### Hosts a Session serves

- **The Provider Workspace, on every Session.** Its requests (`/clusters/<clusterName>/…`) go to the front-proxy, which any Connector reaches, so any Session of the Connection serves them. The Provider-side has no name for the front-proxy: its client config uses a placeholder host that only exists inside the SDK, and the Connector ignores the `Host` of these requests ([05 Connector, "Request pipeline"](./xxx-notes-011-v5-impl-05-connector.md)).
- **The hosts in the Hello's `hosts`**: the shard's virtual-workspace host, taken from the Connector's configuration and sent at the start of every new Session. These are real kcp hostnames, from endpoint slices; they select the shard.

The hosts are **routing information, not authority**. The Provider-side never resolves them (its dial hook opens Streams on Sessions), so they're labels it can't verify independently, and it doesn't need to: a Session that passed the Inner TLS and the Hello's checks already belongs to PM's Connector for this Connection, and which host it serves only decides where Streams go. Keeping hosts out of the Connector Certificate means a hostname change on PM's side never waits for certificates to be reissued: during a rename of a virtual-workspace host, Connectors list both the old and the new host until endpoint slices have switched, so the order of DNS, configuration and endpoint-slice changes doesn't matter. A rename of the front-proxy is a PM-side configuration change only.

### What binds a Session to its PM Instance

The Connector CA pinned **for this Connection**: each PM Instance has its own CA, managed by its own cert-manager; with an external PKI, each PM Instance has its own intermediate, and that's what Providers pin ([04 Connector CA, "The CA"](./xxx-notes-011-v5-impl-04-connector-ca.md)). Connector Certificates carry no PM Instance identity and no hostnames. If two PM Instances shared a CA, a compromised Connector of one could open Sessions for the other's Connections.

## Provider replicas

The protocol makes **no assumption about which Provider replica a Session or Control Session reaches.** Two Sessions of one Connection, or two consecutive Sessions on one shard, may end on different replicas; the Provider-side may close or go-away any Session at any time and expects the Connector to reconnect. This keeps every replica strategy the SDK offers ([07 SDK, "Replicas"](./xxx-notes-011-v5-impl-07-provider-sdk.md)) invisible to PM.

## Recovery

What each side can lose, and how it comes back:
- **The PM-side loses the Connection Secret, or the Provider-side rejects it** (e.g. its store was restored from an older backup): `platform-mesh-operator` changes the Connection Secret through the `ProviderEndpoint` (see "Changing the Connection Secret"). The `connectionID` and all Provider-side state stay.
- **The Provider-side no longer knows the Connection** (deleted in the portal, its store lost, or expired as an orphan): it answers with the stable "unknown Connection" status, inside the Inner TLS (see "Errors and retries"). Two ways back: Provider Administrators import the old entry, which keeps the `connectionID` and is preferred when the Provider still has the data; or the Owner onboards the same `Provider` again, which registers a **new** Connection, meant for switching Providers or a Connection the Provider won't restore ([03 Operator, "Onboarding requests"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
- **No Session verifies and Control Calls fail authentication** (a Pinned Bundle that missed a Rollover, and a Connection Secret the Provider-side lost or rejects): nothing in the protocol can fix this. Pushing a bundle needs the Connection Secret, and repairing the Connection Secret needs a Session. A missed Rollover alone isn't this case: `platform-mesh-operator`'s first successful Control Call pushes the current bundle. The Provider Administrators re-pin the bundle out-of-band ([07 SDK, "Administration"](./xxx-notes-011-v5-impl-07-provider-sdk.md)).
- **The Provider's Base URL changes**: a PM Administrator edits the Connection's Base URL on the PM-side; the `connectionID`, Server Keys and all Provider-side state stay. Connectors replace their Sessions when the `ProviderEndpoint`'s URL changes ([03 Operator, "Writing ProviderEndpoints"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).

Disasters on either side (a Provider's cluster rebuilt, for example) are handled the same way: out-of-band steps between the people running both sides, on top of the properties above. The protocol keeps them possible: `connectionID`s are stable, the Provider-side can import a `Connection`, and the PM-side can take a Connection's Base URL and Server Keys from an administrator. The procedures themselves aren't part of this plan.

## Server Keys

- **A list**, so the Provider-side can rotate with overlap: when its administrators start a rotation, a new key is announced in `GET` before it's used, and the old one retired after the overlap.
- **Switch per Connection when the `ProviderEndpoint` is ready**: the Provider-side presents a new key on a Connection only once that Connection's `ProviderEndpoint` lists it (it reads the `ProviderEndpoint` over Sessions anyway), and retires the old key once every Connection's `ProviderEndpoint` lists the new one, or after a deadline. A Connection whose `platform-mesh-operator` is down for longer doesn't lose its Sessions.
- **No refresh over plain HTTPS.** A Server Key endpoint verified only by web PKI would let the Provider's Edge swap keys and undo the Inner TLS. After a lost or missed key, a PM Administrator supplies the new keys ([03 Operator, "Writing ProviderEndpoints"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).

## Status

`GET /v1/connections/{id}` returns what the Provider-side's clients actually use:
- **`workspace`**, always present: the Provider Workspace through the front-proxy. Its live Sessions are all the Connection's Sessions, since any Session serves it;
- **`virtualWorkspaces`**, keyed by host: each endpoint-slice host its clients use, with the live Sessions covering it.

For each: the number of live Sessions, whether its last requests reached kcp, and the last error, without sensitive detail.

`platform-mesh-operator` turns this into aggregate conditions ([03 Operator, "Health"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)). No shard names or PM topology appear.

## Errors and retries

- **Upgrades refused with `503` and `Retry-After`** (e.g. by a standby Provider replica, or beyond the handshake limits): the Connector treats the value as the **lower bound for its normal reconnect backoff** (jitter, growth on repeated failures), with no special handling. A standby built on the SDK sends `Retry-After: 5` ([07 SDK, "Replicas"](./xxx-notes-011-v5-impl-07-provider-sdk.md)). A `503` from the Provider's Edge without `Retry-After` falls back to the normal backoff.
- **A Session closed gracefully by the Provider-side** (go-away, then close): Connectors reconnect at once, with jitter.

- **Authentication failures** reveal no detail: a failed Session handshake ends with a standard TLS alert (e.g. `unknown_ca` when the Connector's CA isn't pinned, which the Connector reports in its metrics), and a failed Control Call gets a uniform response, then the Control Session closes. The Provider-side records the reason for its own audit, in logs and metrics; a failed authentication changes nothing on the Provider-side.
- **Rate limits** return `429` with `Retry-After`; the PM-side honours it with jitter.
- **Transient failures** (`5xx`, connection errors) are retried with backoff; Registration retries reuse the same idempotency key and hash.
- **A refused token** (unknown, expired, spent, or its Server Key unusable) is refused before the Registration upgrade with a stable status, reported as `TokenRefused`.
- **An unknown or deleted Connection** refuses everything with a stable, distinguishable status, so the PM-side can report it instead of retrying forever. Only the answer given **inside the Inner TLS on `control`**, verified against the pinned Server Key, counts as "unknown"; a refusal of a Session upgrade, or anything the Edge could have sent, doesn't. Before answering "unknown", the Provider-side reads the `Connection` uncached; if its cache isn't synced or its API can't be reached, it answers `503` instead, and so do standbys. "Unknown" is also what lets an Owner onboard the same `Provider` again (see "Recovery").

## Versioning

- **Path version** (`/v1`) for the endpoints, **ALPN** inside the Inner TLS for the Registration, Session and control protocols.
- **Compatibility window: current and previous version.** The SDK and every PM-side component support both.
- **Negotiated independently per channel, never stored**: every Registration, Control Session and Session negotiates its own version through ALPN (e.g. `pm-register/1`, `pm-control/1`, `pm-session/1`). A channel opened after both sides upgrade uses the newer version, so Connectors, `platform-mesh-operator` and SDKs can be upgraded independently, without re-registering Connections.

## Deliverables

- **This contract as a versioned specification** in the SDK module, with the message schemas.
- **Test vectors**: Onboarding Tokens (malformed, a `keyfp` that doesn't match the presented key), Registration (a refused token, a retry with a changed hash or `clusterName`), Inner TLS (a client certificate from an unpinned CA, expired, without the `clientAuth` EKU, or with an intermediate in its chain; a pinned CA that has expired; an unpinned Server Key), Hellos (wrong `connectionID` or `clusterName`, missing or malformed `hosts`, oversized), Control Sessions (wrong Connection Secret, closed on the first failure; an inner request for another Connection), Pinned Bundles (malformed PEM, a non-`CERTIFICATE` block, a block with PEM headers, text between blocks, a duplicated certificate, a non-CA certificate, no certificate, only expired certificates, too many certificates, the same set in another order), Stream requests (non-canonical paths, workspace paths instead of logical cluster names). Shared by the SDK and the Connector's tests ([10 Testing](./xxx-notes-011-v5-impl-10-testing.md)).

## Decided

- Compatibility window: current and previous version, negotiated per channel (see "Versioning").
- Registration runs inside Inner TLS, pinned to the Server Key named in the Onboarding Token; outer TLS stays strict but isn't the trust anchor.
- Sessions are authenticated by the inner mutual TLS alone, with a Hello message for `connectionID`, `clusterName`, `hosts` and `sub`; no Session token. The Hello's checks are essential.
- Control Sessions are bound to the Connection in their path; "unknown" counts only inside the Inner TLS on `control`.
- Streams carry HTTP from the first byte; no preamble.
- No intermediate certificates: Connector Certificates are issued directly by a pinned CA.
- Control Calls use HTTP/1.1; the first failed authentication closes the Control Session.
- `clusterName` is the Provider Workspace's logical cluster name, as kcp assigns it; the Provider-side has no front-proxy URL.
- Rate limits on Registration per Owner: SDK defaults, configurable by each Provider ([07 SDK](./xxx-notes-011-v5-impl-07-provider-sdk.md)).
- The Connection Secret changes only through the `ProviderEndpoint`; the Pinned Bundle is replaced by compare-and-swap.
