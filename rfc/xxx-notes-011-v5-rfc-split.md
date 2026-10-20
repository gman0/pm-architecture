# Notes v5: splitting the design into two RFCs

Working notes for [011](./011-provider-onboarding-flow-improvements.md), next to the [v5 notes](./xxx-notes-011-provider-onboarding-flow-improvements-v5.md) and the [implementation plan](./xxx-notes-011-v5-impl-00-overview.md). Not the RFC itself. An outline for landing the design as two RFCs, one after the other, because the full design is too big for one review. No backwards compatibility between the two is needed.

## The seam

- **RFC A, "Remote Providers through Connections" (core)**: a complete, secure Connection, with the whole protocol, and every lifecycle operation an Owner or Provider needs, with simpler rules where the full ones are only about smoothness or visibility.
- **RFC B, "Operating Connections" (hardening)**: the Connector CA's full Rollover timing and gated distrust, full Edge diagnosis and the recovery runbooks, **without changing the protocol**.

Why this seam, and not Outer TLS first and Inner TLS later:
- **A is secure on its own.** The Edge is untrusted from the start: Inner TLS, Server Key pinning, Registration inside the Inner TLS. There's no interim trust assumption to walk back. Shipping Outer TLS alone would be v5's rejected "plain Sessions, trusting the Edge", a regression from RFC 006, and would need a Session token that the second RFC removes again (see the [Outer-TLS-only notes](./xxx-notes-011-v5-outer-tls-only.md)).
- **A runs on its own indefinitely.** Renewals of the Connector CA are automatic in A, with an overlapping set; a compromise has an immediate distrust; Server Keys rotate in band; Owners can abandon a Connection and Providers can move their Base URL.
- **B is additive.** The protocol in A already carries everything B uses, and B extends A's resources (`ConnectorCASet`) instead of adding new ones for the same purpose.

## RFC A: Remote Providers through Connections

**In scope:**
- **Problem, terminology, trust boundaries, threat model**, with the Edge outside the trust boundary, the reasons for not trusting it and for not requiring TLS passthrough, and the non-goal on verifying who registers.
- **The full protocol**: Registration inside Inner TLS with `keyfp`; Control Sessions bound to their path, "unknown" answered inside the Inner TLS, `PM-Server-Key`; Sessions with the Hello (`connectionID`, `clusterName`, hosts) and yamux; framing; the Pinned Bundle with compare-and-swap and the re-check of open Sessions; Server Keys as a list with in-band announcement; status; versioning; errors; `unusedExpiresAt`.
- **APIs**: `ProviderOnboardingRequest` (with `tokenID`, immutable spec), `Provider` (with `rotate-connection-secret` and `abandon-connection`), `ProviderEndpointRegistration` (with a mutable `url`) and its `Secret`, `ProviderEndpoint`, the Connection Export with its grants, the policies in Provider Workspaces, and `ConnectorCASet` in its A form.
- **`platform-mesh-operator`**: the reconcile steps with onboarding requests under the `Provider`'s key, the policies and their dry-run check, Onboarding and Registration, the first Control Call and the `ProviderEndpoint`, Base URL changes, abandoning a Connection, health with the basic labels and the `unusedExpiresAt` warning, Secret Changes (on demand with rate limit and coalescing, repair, after tampering, a distrusted CA still pinned), the leaked-Connection-Secret limitation, the Connector CA Set in its A form, egress, deletion.
- **The Connector**, in full, including the certificate switch rule: discovery and the hosting shard, the Session rule, replicas, opening Sessions and replacing them on a `url` or `caBundle` change, routing by path, canonical paths, the Denylist, identity, limits.
- **`provider-gateway`**: the Connection API, Sessions, the operator proxy and kubeconfig `Secret`s (with `rotate-proxy-credential`), the `Connection`, `OnboardingToken` and `ServerKey` resources, orphan expiry, in-band Server Key rotation.
- **Quickstart, deployment, tests** for that scope.

### What A does more simply

| Topic | In RFC A | Added in RFC B |
|---|---|---|
| Connector CA Set | **every unexpired CA** from the CA's `CertificateRequest`s, minus distrusted ones, pushed on difference. A renewal (cert-manager's yearly one, or `cmctl renew`) adds the new CA within minutes; the previous CA leaves the set when it **expires** | each previous CA dropped 48 hours after its successor, spread over 6 hours per Connection, per-CA drop times, the leaf gate; the `Dropped` state |
| Connector certificate switch | **in A**: the switch rule. A renewed certificate is used once its CA is `Current` and `Converged`, or the current one has less than about an hour left; switching resets the reconnect backoff | — |
| Convergence | **in A**: every Connection classified as pinned, not reached or unknown relative to the current CA, from the polls that already read the pinned fingerprints; the `Converged` condition on `ConnectorCASet`; the metric by state and shard, and its alert | — |
| `ConnectorCASet` | **in A**: `spec.distrust` entries of `{fingerprint}` (`key-sha256:` only, by schema pattern), all applied **at once**; `status.certificates` with both fingerprints and `state` (`Current`, `Previous`, `Distrusted`), per-entry results (`Applied`, `Unmatched`) with a warning condition and alert, the conditions `Valid` (never empty) and `Converged`; a printer column with the current CA's key fingerprint; created by `platform-mesh-operator` if missing | `immediate` on entries, default `false`: gated distrust (the new CA converged, no Connector's certificate chaining to the distrusted one), `DistrustPending` with `waitingFor`, `Dropped`, `DistrustedKeyReused` |
| A compromised Connector CA | listing its key in `spec.distrust` applies at once: the push drops it, and the Provider-side's re-check closes the Sessions opened under it. A states the effect: Connectors still presenting a certificate from that CA can't open new Sessions until their certificates are renewed. No runbook in A | the gate: distrust waits until the new CA has converged and every Connector has switched, unless `immediate`; the compromise runbook (renewing the CA and the Connector Certificates, getting a key fingerprint, rollback) |
| A distrusted CA still pinned | `DistrustedCAPinned`: push the corrected set, then change the Connection Secret; an Event and a metric reason, not the tamper alert | `TrustOutdated` on `Provider`s whose Connections weren't reached since the distrust |
| Edge diagnosis | the basic labels: `EdgeUpgradeRefused`, `EdgeTLS`, `ControlUnreachable`, `SecretRejected`, `ConnectionUnknown`, `NoSessions`, `KcpUnreachable`, `TokenRefused` | `EdgeBotChallenge`, `EdgePayloadBlocked`, `EdgeTimeout`, `EdgePathRewritten` |
| Recovery beyond repair | automatic repair of a lost or rejected Connection Secret; abandoning a Connection; Base URL changes by a PM Administrator; out-of-band otherwise | runbooks: importing and re-pinning `Connection`s, PM Administrators' recovery input for Server Keys, re-entering a `connectionID` |
| CA request history | cert-manager's default history (3); enough for the two unexpired CAs of normal operation | `revisionHistoryLimit: 10` and its alert, for several forced renewals within a year |

### Kept in A so that B stays additive

- **The protocol is complete**: B introduces no new endpoint, field or ALPN version.
- **The Pinned Bundle is a list** with compare-and-swap, and A's set already holds every unexpired CA.
- **`ConnectorCASet` exists in A**: B adds fields and states to it, not a new resource. A's distrust entries keep their effect in B: their gate is long satisfied, so their CAs stay `Distrusted`; only an entry added in the minutes around the upgrade would wait for the gate, which B's upgrade notes state.
- **Fingerprint formats** (`cert-sha256:`, `key-sha256:`) are defined in A.
- **Connector Certificates carry a common label** in A's chart, for renewing them all at once.

## RFC B: Operating Connections

**In scope**, each with its tests and runbooks:
- **The full Connector CA Rollover timing**: each previous CA dropped 48 hours after its successor with a 6-hour spread, per-CA drop times, the leaf gate, `revisionHistoryLimit: 10` and its alert.
- **`ConnectorCASet` extended**: `immediate` with gated distrust as the default (built on A's convergence and the leaf gate), `DistrustPending` and `waitingFor`, `Dropped`, `DistrustedKeyReused`, the full signals table.
- **`TrustOutdated`** on `Provider`s.
- **The full Edge diagnosis.**
- **Recovery procedures** for Provider Administrators and PM Administrators (importing and re-pinning `Connection`s, recovery input, re-entering a `connectionID`).

B changes `platform-mesh-operator` and the charts, not the protocol, the Connector or `provider-gateway`.

## Section by section

Where each section of today's documents goes. **A** means it moves to RFC A as is, **A (simplified)** that A keeps a reduced form (see "What A does more simply"), **B** that it moves to RFC B, **A + B** that both share it and B extends it.

### v5 notes

| Section | Goes to |
|---|---|
| Terminology | A; B adds `ConnectorCASet`'s gated distrust and the `Dropped` and `DistrustPending` states |
| Problem (including Non-goals) | A |
| Trust boundaries | A |
| Core idea | A |
| Provider-side prerequisites | A |
| Flow overview | A |
| Onboarding Token | A |
| `platform-mesh-operator` and the Connectors | A |
| Connection API (on the Base URL) | A |
| Session credentials: the Connector CA | A (simplified: the CA, Connector Certificates, the Pinned Bundle, distribution, renewal with an overlapping set, convergence and the switch rule, immediate distrust); B: the 48-hour drop with spread in "Automatic Rollover" |
| Sessions: a Stream multiplexer over a WebSocket | A |
| The Provider's Edge: Inner TLS on Sessions | A |
| The Connector as an authorizing proxy | A |
| Connector policy | A |
| Connector implementation | A |
| Per-shard Connectors | A |
| Connector Replicas | A |
| `platform-mesh-operator` (section) | A + B (Rollover details, gated distrust) |
| Provider-side: serving many PM Instances | A |
| Changing the Connection Secret | A |
| Recovery | A (repair, onboarding again, abandoning, Base URL); B (disaster runbooks) |
| What each credential exposes, Threat model | A |
| Resources | A |
| Restart and failure semantics | A + B (B adds the Rollover-drop, gated-distrust and full Edge-diagnosis rows) |
| Offboarding and revocation | A |
| Observations | A |
| Rejected alternatives | A, except those about Rollover generations and triggering renewals (B) |
| Open items, Prior art | split by topic |

### Implementation plan

| Document | RFC A | RFC B |
|---|---|---|
| 00 Overview | everything, including "The Provider's Edge"; B's points marked as later | B's decisions and build order |
| 01 Protocol | all of it | — (no protocol change) |
| 02 APIs | all of it | — |
| 03 Operator | everything except: the Connector CA Set's drop timing and gated distrust; `TrustOutdated`; the full Edge diagnosis; the recovery-input runbooks | those parts |
| 04 Connector CA | What it provides, principles, the CA, the Connector CA Set (every unexpired CA, minus distrusted ones), `ConnectorCASet` (A's fields), convergence, Connector Certificates with the switch rule, compromise response (immediate), signals for CA expiry, validity, convergence, pushes and distrust | the drop timing, the leaf gate, per-CA drop times, gated distrust, `revisionHistoryLimit`, the remaining signals |
| 05 Connector | all of it | — |
| 07 SDK | all of it | — (only the administration runbooks for import and re-pin move to B's runbooks) |
| 08 Quickstart | all of it | — |
| 09 Deployment | kcp bootstrap, charts (Connector Certificate labels, `ConnectorCASet` CRD), network, rollout, local development | `revisionHistoryLimit: 10` |
| 10 Testing | Onboarding, shards and replicas, PM-side restarts, Edge (basic labels), Connection Secret, Rollover (renewal with overlap, convergence, restarts mid-Rollover, a control path broken during a Rollover, Provider down, cert-manager down), Compromise (immediate distrust, guards), recovery and offboarding, security tests | Rollover rows for the 48-hour drop and two renewals within 48 hours; Compromise rows for gated distrust and `TrustOutdated`; the full Edge rows |
| todo | follow-ups that build on A | follow-ups that build on B |

### Review findings

| Item | RFC |
|---|---|
| 1 Registration through the Edge | A |
| 2 Leaked Connection Secret | A (documented limitation) |
| 3 RBAC delegation and policies | A |
| 4 Authenticated "unknown" | A |
| 5 Control Sessions bound to their Connection, essential Hello | A |
| 6 First-call deadlock | A |
| 7 Compromise response | A (immediate distrust, `ConnectorCASet`, `DistrustedCAPinned`); B (the gate, `TrustOutdated`) |
| 8 Connector's kcp identity in the threat model | A |
| 9 Abandoning a Connection | A |
| 10 Renaming endpoints | A |
| 11 Egress | A |
| 12 Edge diagnosis for Sessions | follow-up after B |
| 13 Orphan expiry and its warning | A |
| 14 Connector CA Set derivation | A (ordering, current CA, "issued", distrusted CAs not counted, `DistrustedCAPinned`, convergence); B (per-CA drop times, the leaf gate, `revisionHistoryLimit`) |
| 15 The Connector's scope and canonical paths | A |
| 16 Grants, client CA, hosting shard | A |
| 17 Local to remote | A (not supported) |
| 18 Switching Providers | A |
| L1–L7 | A |

## Size

- **A** is most of today's design: everything except 04's drop timing, the leaf gate and gated distrust, the full Edge diagnosis and the recovery runbooks.
- **B** is a focused hardening RFC around the Connector CA's lifecycle, plus diagnosis and runbooks. It touches 03, 04, 09 and 10, and no Connector or Provider-side code.

## Runbooks

A has no runbooks: it states behaviour and effects (what distrust does, what a renewal does, what abandoning a Connection or changing a Base URL does). The operational procedures, for A's features and B's alike, come with B: the compromise response, recovery by Provider Administrators and PM Administrators, and the routine ones.
