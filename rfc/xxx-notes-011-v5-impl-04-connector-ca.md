# v5 implementation plan: connector CA

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). The PKI that authenticates [connectors](./xxx-notes-011-v5-impl-05-connector.md) to Providers: CA generations that Providers pin, connector certificates issued from them, and the `ConnectorCABundle` resource that declares the generations and reports rollover progress. The design rationale is in v5, "Session credentials: the connector CA", "Pinning immutable CA generations" and "Distributing the connector CA to Providers".

## What it provides

- **Connector certificates**: one per shard's connector, short-lived, used to sign session assertions and as the inner TLS client certificate ([01 Protocol, "Sessions"](./xxx-notes-011-v5-impl-01-protocol.md)).
- **CA certificates per generation**, which the operator sends to Providers as the connector CA bundle at registration and during rollovers ([01 Protocol, "Connector CA bundle"](./xxx-notes-011-v5-impl-01-protocol.md), [03 Operator](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)).
- **Generation states and rollover progress**, in one resource, `ConnectorCABundle`.

## Principles

- **Providers pin CA certificates, and generations are immutable.** A generation is never renewed with a new key; changing anything means a new generation, added with overlap.
- **The pinned certificate's expiry is a hard deadline.** A generation that wasn't rolled over stops working everywhere; signals for the people running PM fire long before (see "Signals").
- **Public and private halves are separate.** The operator and connectors only ever see public certificates. Keys stay with their key source, and only the chart and cert-manager connect the two.
- **Connector certificates are nobody's pin**: short-lived, new key on every renewal.
- **The key source is replaceable.** Nothing in the protocol, the operator or the connector depends on where a generation's key lives.

## Key sources

Each generation names where its key lives, mirroring how PM's `infra` chart configures kcp's own CA (`kcp.certificates`):

| Key source | Meaning |
|---|---|
| `caSecretRef` | An existing CA Secret (certificate and key), e.g. created with a small CLI command or synced from a secret store. The chart renders a cert-manager CA issuer on it |
| `issuerRef` | An existing cert-manager issuer that issues from the generation's key, e.g. backed by Vault or a cloud CA. The key never enters the cluster |

- **cert-manager never manages a generation's CA certificate**, so it can't renew it, and the renewal trap (v5, "Pinning immutable CA generations") can't happen.
- **The generation's public certificate is supplied with it**, in values, alongside the key source. Creating a generation is a deliberate step, about once a year; the CLI command (for `caSecretRef`) prints the certificate, an external PKI documents how to fetch it.
- **Each PM installation has its own generations.** A generation is never shared between installations, not even through a common external PKI issuer: the per-connection pinned generation is what binds a session to its installation ([01 Protocol, "What binds a session to its PM installation"](./xxx-notes-011-v5-impl-01-protocol.md)). The runbook states this rule; nothing enforces it.
- **Key Secrets live in a namespace only cert-manager reads** (for `caSecretRef`); neither the operator nor connectors can read them.
- **Development only** (kind, e2e): the chart can generate a self-signed generation with cert-manager. Its public certificate isn't known at render time, so in this mode only, behind an explicit flag, the operator reads `ca.crt` from the generation's Secret.

## ConnectorCABundle

A cluster-scoped singleton in PM's runtime cluster, `connectorcabundles.providerconnections.platform-mesh.io` (short name `ccab`), named `default`. It holds the **public halves of every generation and their lifecycle state**; the bundle Providers pin is derived from it. Example, mid-rollover:

```yaml
apiVersion: providerconnections.platform-mesh.io/v1alpha1
kind: ConnectorCABundle
metadata:
  name: default
  generation: 7
  # annotations:
  #   providerconnections.platform-mesh.io/ignore-degraded-connections-for: g3   # activate g3 despite degraded connections
spec:
  generations:                          # rendered by the chart from values; public data only
    - name: g1
      state: Retiring                   # Next → Active → Retiring → Retired
      certificate: |
        -----BEGIN CERTIFICATE-----
        ...
    - name: g2
      state: Active
      certificate: |
        -----BEGIN CERTIFICATE-----
        ...
status:
  observedGeneration: 7
  published:                            # what Providers should pin: Next, Active, Retiring
    generations: [g1, g2]
  generations:
    - name: g1
      fingerprint: sha256:4f1c…
      notAfter: "2030-03-01T00:00:00Z"
      validConnectorCertificates: 1     # connector certificates from g1 not yet expired
    - name: g2
      fingerprint: sha256:9a07…
      notAfter: "2031-03-01T00:00:00Z"
      validConnectorCertificates: 2
  shards:                               # one entry per shard worker of the operator
    - name: root
      connections: 140
      pinnedCurrent: 140
      unreachable: 0
      degraded: 0
      lastUpdateTime: "2026-10-12T14:02:10Z"
    - name: nereus
      connections: 151
      pinnedCurrent: 148
      unreachable: 2
      degraded: 1                       # sessions up, control calls failing; listed with their last error
      lastUpdateTime: "2026-10-12T14:01:55Z"
  connections:
    total: 291
    pinnedCurrent: 288
    unreachable: 2
    degraded: 1
  conditions:
    - type: Valid                       # every certificate parses, is a CA, can sign, isn't expired
      status: "True"
    - type: BundleConverged             # every reachable connection pins the published generations, by fingerprint
      status: "False"
      reason: ConnectionsUnreachable
    - type: ReadyToActivate             # a Next generation is pinned by every reachable connection, and none is degraded
      status: "False"
      reason: NoNextGeneration
    - type: ReadyToRetire               # no valid connector certificate from a Retiring generation remains
      status: "False"
      reason: ConnectorCertificatesValid
```

Printer columns: Active, Next, Converged, Age.

### Validation

CEL in the CRD's schema, so it applies to every writer, including hand edits:
- the name is `default`;
- exactly one `Active` generation, at most one `Next`;
- unique generation names;
- generations are never removed: every name in the old object is still present;
- states only move forward (ordinals compared with the old object);
- a generation's certificate is immutable;
- certificates have PEM shape. Real checks (a CA, allowed to sign, not expired) are done by the operator and reported in `Valid`.

Forward-only states keep every rollover in order. Providers' copies are compared by fingerprints and replaced by compare-and-swap ([01 Protocol, "Connector CA bundle"](./xxx-notes-011-v5-impl-01-protocol.md)).

### Why the certificate is inline

A reference would have to point at a CA Secret (which holds the key), a cert-manager object (which doesn't publish the CA certificate) or an external PKI (which the operator would need credentials for). Inline certificates keep the operator away from keys and PKIs, let the schema pin each generation to exactly one certificate (a referenced Secret could be renewed with a new key under the same name), work the same for every key source, and leave an audit trail in Git.

### Who writes what

| Part | Written by |
|---|---|
| `spec` | The `providerconnections` chart, from values ([09 Deployment](./xxx-notes-011-v5-impl-09-deployment.md)). Nothing else |
| `status.shards[<shard>]` | That shard's operator worker, by server-side apply with a per-shard field manager; the list is keyed by shard name, so workers never conflict ([03 Operator, "Partitioning by shard"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)) |
| The rest of `status` | One aggregator in the operator, under its own Lease. It reads the shard entries and the connector `Certificate` objects |

Shard workers build the bundle from the **spec**, not from the aggregator's status, so pushing a rollover never waits on reporting.

### How the operator consumes it

1. **Watch** the singleton; every shard worker sees the same spec.
2. **Validate** the certificates; a failing `Valid` stops rollout work, nothing is pushed.
3. **Build** the bundle: a SPIFFE-format JWKS with the certificates of `Next`, `Active` and `Retiring` generations.
4. **Use it**: registration sends the current bundle; each shard worker pushes it, with compare-and-swap, to its connections whose pinned fingerprints differ; health polls compare pinned fingerprints with the published generations, and flag any pinned certificate that isn't one of PM's generations as tampering ([03 Operator, "Connector CA rollover"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)).
5. **Report** per-shard progress into `status.shards`; the aggregator computes totals and conditions.

`ReadyToRetire` comes from the connector `Certificate` objects (read-only, not their Secrets): their issuer and validity show whether any certificate from a `Retiring` generation is still valid.

## Connector certificates

- **One cert-manager `Certificate` per shard**, issued by the `Active` generation's issuer (the chart renders issuers for `Next` and `Active` generations only).
- **Lifetime 24 hours, renewed halfway** (`renewBefore: 12h`), `rotationPolicy: Always`. A stolen certificate is useful for at most a day, which is what lets the connector CA do without revocation; cert-manager can be down for up to 12 hours before any certificate expires, and even then open sessions keep running ([05 Connector](./xxx-notes-011-v5-impl-05-connector.md)). It's far shorter than any certificate elsewhere in PM on purpose.
- **Key usages**: digital signature and client authentication.
- **Minimal content**: key usages, and the shard name in the subject for audit. **No hostnames and no installation identity**: served hosts travel in each session's assertion ([01 Protocol, "Hosts a session serves"](./xxx-notes-011-v5-impl-01-protocol.md)), and the installation binding is the per-connection pinned generation ([01 Protocol, "What binds a session to its PM installation"](./xxx-notes-011-v5-impl-01-protocol.md)). Nothing in a connector certificate changes when PM renames hosts.
- **Delivered to the connector** as a mounted Secret, picked up on renewal without a restart ([09 Deployment, "Connector"](./xxx-notes-011-v5-impl-09-deployment.md)).
- **Checked by the connector**: on every renewal, it verifies that its new certificate chains to the `Active` generation's certificate in the `ConnectorCABundle` (public, readable). On a mismatch, for example a wrong certificate copied into values, it keeps using its previous certificate until that expires, and reports it (metric and event, see "Signals") ([05 Connector](./xxx-notes-011-v5-impl-05-connector.md)).
- **All replicas of a shard's connector share it.** Per-replica certificates are a later option with a KMS-backed issuer.

## Rollover

Four changes to values, each gated by a condition:

1. **Add the new generation as `Next`** (key source and public certificate). The operator pins it everywhere. Wait for **`ReadyToActivate`**. It stays false while any connection is **degraded** (sessions up, control calls failing): activating would cut that connection's traffic. Fix the control path with the Provider, or proceed knowingly by annotating the resource with `providerconnections.platform-mesh.io/ignore-degraded-connections-for: "<next generation>"`; those connections lose their sessions until their control path works again and they catch up. The operator honours the annotation only while the named generation is `Next`, so it can't carry over to a later rollover; for any other value it records an event and ignores it. It can be set through values (the chart renders it, and the decision is recorded in Git) or with `kubectl annotate` in an incident; if an installation enables Flux's drift correction for the release, a hand-set annotation may be reverted at the next reconcile.
2. **Set it `Active` and the old one `Retiring`.** The chart switches connector `Certificate`s to the new issuer; cert-manager reissues them; connectors check the chain. Wait for **`ReadyToRetire`**.
3. **Set the old one `Retired`.** The operator unpins it everywhere. Wait for **`BundleConverged`**.
4. **Delete the old generation's key**: its Secret, or retire it in the external PKI.

**The procedure is manual by design**: people create generations and change their states; the operator, cert-manager and connectors do everything else. A generation lives **10 years**, aligned with kcp-operator's CA default. Unlike kcp's CAs, which cert-manager renews 30 days before expiry because nobody outside PM pins them, a connector CA rollover must **start at least a year before the active generation expires**, to leave time for the procedure and for Providers that are unreachable for a while. It's otherwise planned, never urgent unless a key is in doubt. Connections that stay **unreachable** (down: no control calls, no sessions) don't block the conditions but are listed per shard, and catch up as soon as their control calls succeed again; **degraded** connections block activation as described in step 1. Even after the old generation has expired, their first successful control call still brings them the current bundle, since control sessions don't depend on the connector CA. Only a connection that also can't authenticate (a lost or rejected secret) stays stuck: repairing its secret needs a session, and no session verifies against its out-of-date bundle. The Provider's administrators then re-pin the bundle out-of-band ([07 SDK, "Administration"](./xxx-notes-011-v5-impl-07-provider-sdk.md)).

## Documentation and follow-ups

- **Documentation is a deliverable of this work**: a runbook for the first generation at installation, for a regular rollover (each of the four value changes, the condition to wait for, how to read the `ConnectorCABundle` status, what to do about unreachable connections), and for a compromise. A rollover is rare, so the runbook has to stand on its own.
- **Automation is a follow-up**, not part of this delivery. The likely shape: people only add generations, each with a planned activation date; the operator advances the effective state itself once the date has passed and the gating condition holds, recording effective states in status. That changes the `ConnectorCABundle`'s ownership model (the spec becomes a plan) and is designed separately.

## Compromise response

- **A shard**: stop issuing its certificate (delete or suspend its `Certificate`); the current one expires within hours.
- **A generation's key**: roll over immediately, skipping the wait for `ReadyToRetire` only as far as the risk justifies, and unpin the compromised generation as soon as the next one is pinned.

## Signals

For the **people running the PM installation** (owners and Providers never see these). PM has no alerting pipeline today (the `observability` chart's Alertmanager is disabled and no chart ships alert rules), so each signal is a **metric**, plus a **condition or event** where one fits, plus a **documented alert rule** with a severity. Installations wire the rules into whatever alerting they run ([09 Deployment, "Observability"](./xxx-notes-011-v5-impl-09-deployment.md)).

| Signal | Metric (exposed by) | Condition or event | Documented rule |
|---|---|---|---|
| Active generation nearing expiry | `providerconnections_connector_ca_generation_expiry_seconds{generation,state}` (operator, aggregator) | — | Active with < 18 months left: warning (start the rollover within 6 months); < 12 months: critical |
| Connector certificate not renewed | `providerconnections_connector_certificate_expiry_seconds{shard}` (connector) | Event on the connector's `Certificate` (cert-manager) | < 6 hours left (half the renewal budget used): warning |
| Connector certificate expired | same | Connector readiness reports it (rollout tooling can hold) | ≤ 0: critical. From then on every session that ends is an outage for its connection on that shard ([05 Connector, "Opening a session"](./xxx-notes-011-v5-impl-05-connector.md)) |
| Chain check failed | `providerconnections_connector_certificate_chain_valid{shard}` (connector) | Event on the `ConnectorCABundle` | == 0: critical (the configured certificate doesn't match the key) |
| Invalid generation in the configuration | `providerconnections_connector_ca_valid` (operator) | `Valid=False` on the `ConnectorCABundle` | == 0: critical |
| Rollover stalled | `providerconnections_connector_ca_connections{state="pinned_current\|behind\|unreachable"}` (operator) | `BundleConverged=False` | Behind (reachable but not current) > 0 for more than an hour: warning |
| A pinned bundle held a CA PM never issued | `providerconnections_connector_ca_bundle_tampered_total` (operator) | `ConnectorCABundleTampered` on the `Provider`, and an event ([03 Operator, "Connector CA rollover"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)) | Increase: critical (a connection secret was used by someone other than PM) |

## Later: KMS- or Vault-backed keys

Already covered by `issuerRef`: the key never leaves the external PKI, and nothing else changes. Per-replica connector certificates, each replica generating a key in memory at start, become possible with such an issuer.

## Work breakdown

1. The `ConnectorCABundle` CRD with its CEL rules and printer columns.
2. Chart templates: the CR from values, CA issuers for `caSecretRef` generations, per-shard connector `Certificate`s, the development-only self-signed generation ([09 Deployment](./xxx-notes-011-v5-impl-09-deployment.md)).
3. Operator: consumption, per-shard status entries, the aggregator ([03 Operator](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)).
4. Connector: chain check on renewal ([05 Connector](./xxx-notes-011-v5-impl-05-connector.md)).
5. A CLI command that creates a `caSecretRef` generation and prints its public certificate.
6. Runbooks: first setup, rollover, compromise, with the documented alert rules from "Signals".

## Decided

- **Generation lifetime**: 10 years, aligned with kcp-operator's CA default; rollovers are manual and start at least a year before the active generation expires.
- **Rollover documentation** is part of this delivery; **automation** is a follow-up.
- **Connector certificates carry no hostnames or installation identity**; served hosts are in the session assertion. **No name constraints** on generations: there are no meaningful names in connector certificates to constrain, and constraints would be fixed for 10 years.
- **No installation identifier**: the per-installation generation is the binding. A PM-wide installation identity would be a broader design.
- **Connector certificates**: 24 hours, renewed halfway, no revocation. kcp doesn't revoke its certificates either, but relies on removing RBAC bindings, which Providers can't do for PM; a short lifetime is the equivalent. Revocation entries in the bundle remain possible later if longer lifetimes are ever wanted.
- **Key sources** mirror `kcp.certificates`: `caSecretRef` or `issuerRef`, with the public certificate supplied in values; a self-signed generation for development only.
- **Declaration**: the `ConnectorCABundle` CR, with inline public certificates, CEL validation, and status for rollover progress.
- **Naming**: connector CA, connector certificates, `ConnectorCABundle`.

