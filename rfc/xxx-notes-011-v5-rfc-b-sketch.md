# Notes v5: sketch of RFC 012 (B), Operating Connections

Working notes, not the RFC. An outline of RFC B's sections and what each must say, to see the full scope and to draw from when writing it. It follows the RFC structure in [the RFC README](./README.md). The RFC itself is self-contained apart from naming RFC 011 as what it builds on; the "From" lines are only for its author. RFC A is sketched in [its own notes](./xxx-notes-011-v5-rfc-a-sketch.md); the seam is in the [split notes](./xxx-notes-011-v5-rfc-split.md).

**Size**: small to medium. One mechanism (the Connector CA's lifecycle) worked out in full, two smaller additions, and the procedures. No protocol, Connector or `provider-gateway` change.

---

## Header

- **Title**: RFC 012: Operating Connections.
- **Status**: Draft. **Authors**.
- **Builds on**: RFC 011 (Remote Providers through Connections). Requires it to be implemented; no compatibility with anything older.

## 1. Summary

- RFC 011 makes Connections work and keeps them secure; this RFC makes them smooth to run.
- Connector CA renewals drop the previous CA on a schedule instead of at its expiry; distrusting a compromised CA waits until PM's own Connectors are safe, unless asked otherwise; Owners see which `Provider`s still trust a distrusted CA; more Edge problems get their own label; every manual procedure is written down.
- It changes `platform-mesh-operator`, the `ConnectorCASet` and the chart, and nothing on the wire.

## 2. Context and Problem Statement

What RFC 011 leaves rough, each with its consequence:
- **The previous CA stays trusted until it expires**, up to a year after a renewal: a CA renewed because of suspicion, without a distrust, keeps working at every Provider.
- **Distrust applies at once**: Connectors still holding certificates from the distrusted CA can't open Sessions until those are renewed, so a compromise response risks an outage of PM's own traffic.
- **An Owner can't tell** that their Connection missed a distrust while it was unreachable.
- **Edge diagnosis covers only the basic cases**: bot challenges, payload inspection, idle and age cuts and path rewrites show up as generic failures.
- **The CA request history** is cert-manager's default; a few forced renewals within a year push out a CA that Connections still pin, which then looks like tampering.
- **No written procedures** for compromise, recovery or routine operations.

Non-goals: protocol changes; automating Connector Certificate renewal on distrust; per-replica Connector Certificates; Edge diagnosis for Sessions; periodic Secret or key rotation.

## 3. Approach

### 3.1 Rollover with a drop schedule

- **The drop time of a previous CA, per Connection**: its successor's issue time + 48 hours + a spread of up to 6 hours derived from the `connectionID`.
- **Per-CA drop times**: each previous CA has its own, so two renewals within 48 hours don't drop a CA still in use.
- **The leaf gate**: a previous CA isn't dropped while any Connector's current certificate still chains to it.
- **The invariant**: 48 hours exceeds a Connector Certificate's renewal time plus convergence; the leaf gate covers the case where it doesn't.
- **Handover**: the drop's push makes the Provider-side close Sessions still on the old CA's certificates with go-away; Connectors reopen them at once; the spread spreads the relists.
- **`Dropped`** as a state in `ConnectorCASet.status.certificates`.
- **Rollover timeline figure** (issue, push, convergence, Connectors switch, 48 hours, drop spread over 6 hours).

From: 04 "The Connector CA Set" (drop time, leaf gate, invariant), "Rollover"; v5 "Automatic Rollover".

### 3.2 Gated distrust

- **`immediate` on `spec.distrust` entries**, default `false`.
- **The gate**: the new CA has converged and no Connector's current certificate chains to the distrusted one; then the CA is dropped from every Connection, with a short spread, and `Distrusted`.
- **`DistrustPending`** with `waitingFor` (the shards whose Connectors haven't switched).
- **`immediate: true`** skips the gate: dropped at once, as in RFC 011, with the stated effect on Connectors.
- **Guards**: never empty (stays `DistrustPending`, `Valid=False`); unmatched entries; rollback before and after it's applied; `DistrustedKeyReused`.
- **Upgrade from RFC 011**: entries already applied stay `Distrusted`; only an entry added around the upgrade waits for the gate.
- **Distrust timeline figure**, gated and immediate.

From: 04 "ConnectorCASet", "Compromise response"; split notes "`ConnectorCASet`" row.

### 3.3 `TrustOutdated`

- A warning condition on a `Provider` whose Connection wasn't reached since a distrust, cleared by the push that catches it up (which also changes its Connection Secret, as in RFC 011's `DistrustedCAPinned`).

From: 03 "Connector CA Set", "Health".

### 3.4 The full Edge diagnosis

- **New labels**: `EdgeBotChallenge` (an HTML page on upgrade), `EdgePayloadBlocked` (a close right after a binary message), `EdgeTimeout` (regular idle or age cuts), `EdgePathRewritten` (`ConnectionMismatch` on a Control Session; no repair, no Secret Change).
- **How each is recognised** on Registration and Control Sessions, and the message pointing at the Edge checklist.
- **The checklist** grows the matching items.

From: 03 "Edge diagnosis"; 01 "Diagnosing the Edge"; 08 "Connection API server".

### 3.5 CA request history

- `revisionHistoryLimit: 10` on the CA `Certificate`, and why it matters (the tamper check only knows CAs whose requests it still sees).
- The alert when the oldest retained request's CA is unexpired while the limit is reached.

From: 04 "The Connector CA Set", "Signals"; 09 "Connector CA".

### 3.6 Signals

- The remaining rows of the signals table: distrust waiting, CA request history full, and the B-specific uses of the convergence and pending-switch metrics. Each a metric, a condition or Event where it fits, and a documented alert rule with severity.

From: 04 "Signals".

### 3.7 Operational procedures

Step by step, each with when to use it, the commands, and how to tell it worked:
- **Shards**: adding or removing one (`infra`'s shards and `kcpTopology` in one profile change).
- **Connector CA**: reading the `ConnectorCASet`; Connections not reached; the CA's `CertificateRequest`s (don't delete, raise the limit, approver-policy); an expired Connector Certificate.
- **Compromise response**: renewing the CA and every Connector Certificate by label, getting a `key-sha256` fingerprint, `spec.distrust`, when to use `immediate`, rollback; a shard's Connector Certificate key.
- **A leaked Connection Secret**, as a PM-side incident.
- **Provider Administrators**: importing and re-pinning `Connection`s, rotating Server Keys and proxy credentials, moving the Base URL.
- **PM Administrators**: Base URL and Server Key edits in `ProviderEndpointRegistration`s, re-entering a `connectionID`.
- **Owners**: abandoning a Connection, switching Providers, rotating a private CA in `caBundle`.
- **Alerts**: what each documented rule means and what to do.

These can live as an appendix or as separate documents the RFC lists.

From: 04 work breakdown item 6, 07 "Administration", 09 "Charts"; the runbooks item in the follow-ups.

### 3.8 Restart and failure semantics (added rows)

- Two renewals within 48 hours; the drop handover; gated distrust with a Connector stuck on the old CA; a Connection unreachable during a distrust (`TrustOutdated`); each new Edge label's scenario.

From: v5 "Restart and failure semantics".

### 3.9 Testing

- Rollover with the 48-hour drop and spread (shortened timings); two renewals within the drop delay; distrust with the gate and with `immediate`; guards; a distrusted CA at an unreachable Connection; the full Edge rows (challenge page, dropped binary messages, idle and age cuts, rewritten paths); a full CA request history.

From: 10 "Rollover", "Compromise", "Edge" (B's rows).

## 4. Pros & Cons

- **Pros**: a renewed CA stops being trusted within days, not a year; compromise response without breaking PM's own traffic; Owners see a missed distrust; Edge problems named precisely; procedures written down; nothing changes on the wire or on the Provider-side.
- **Cons**: more state to derive (per-CA drop times, the leaf gate, distrust states); more conditions and alerts; a 48-hour window during which two CAs are trusted after every renewal.

## 5. Alternatives

- Dropping the previous CA right after convergence (no fixed delay): Connectors restarted around the drop would fail at Providers already updated.
- `platform-mesh-operator` triggering CA renewals or Connector Certificate renewals itself (needs rights in cert-manager's scope).
- Immutable, long-lived CA generations with a manual Rollover.
- A grace period after the distrust gate (rejected: the gate already waits for the Connectors).

From: v5 "Rejected alternatives" (the Rollover and renewal entries); 04 "Decided".

## 6. Open questions

- None known.

## Figures to draw

1. Rollover timeline with the drop schedule.
2. Distrust timeline, gated and immediate.
