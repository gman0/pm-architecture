# v5 implementation plan: Connector CA

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). The PKI that authenticates [connectors](./xxx-notes-011-v5-impl-05-connector.md) to Providers: a Connector CA managed by cert-manager, Connector Certificates issued from it, the Connector CA Set that Providers pin, and the `ConnectorCASet` resource that reports it. The design rationale is in v5, "Session credentials: the Connector CA" and "Distributing the Connector CA to Providers".

## What it provides

- **Connector Certificates**: one per shard's Connector, short-lived, used as the client certificate in each Session's Inner TLS, which is how Sessions are authenticated ([01 Protocol, "Sessions"](./xxx-notes-011-v5-impl-01-protocol.md)).
- **The Connector CA Set**: the Connector CA certificates Providers pin, which `platform-mesh-operator` sends at Registration and pushes whenever it changes ([01 Protocol, "Pinned Bundle"](./xxx-notes-011-v5-impl-01-protocol.md), [03 Operator, "Connector CA Set"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
- **Rollover progress**, reported in one resource, `ConnectorCASet`.

## Principles

- **cert-manager owns the CA, and the Rollover is automatic.** cert-manager renews the CA with a new key; `platform-mesh-operator` pushes the new CA certificate to every Provider; Connectors switch to certificates from the new CA only once Providers trust it. Nobody takes part.
- **Public and private halves are separate.** `platform-mesh-operator` and Connectors only ever see public certificates. Keys stay with cert-manager or the external PKI.
- **Connector Certificates are nobody's pin**: short-lived, new key on every renewal.
- **Each PM Instance has its own CA.** Providers pin the CA cert-manager manages for this PM Instance, never a shared root, so a Session is bound to its PM Instance ([01 Protocol, "What binds a Session to its PM Instance"](./xxx-notes-011-v5-impl-01-protocol.md)).

## The CA

One cert-manager `Certificate` with `isCA: true`, rendered by the `providerconnections` chart ([09 Deployment, "Connector CA"](./xxx-notes-011-v5-impl-09-deployment.md)):
- **Lifetime 1 year, renewed 90 days before expiry**, `rotationPolicy: Always`: every renewal has a new key, so a stolen CA key is useful for at most about a year.
- **Issued by** a self-signed issuer by default, or by an `issuerRef` (e.g. Vault or a cloud CA). With an `issuerRef` the CA is an intermediate under the external root; Providers pin the intermediate, not the root, so PM Instances sharing an external PKI still have their own CAs. Either way, the CA's own key sits in its `Secret` in the cluster; an external PKI only keeps the root out of it.
- **In cert-manager's cluster resource namespace** (the namespace cert-manager looks in for `Secret`s of cluster-scoped resources; by default the one it runs in), which only cert-manager reads. A `ClusterIssuer` of type CA on its `Secret` issues the Connector Certificates, in the Connectors' namespace. Neither `platform-mesh-operator` nor Connectors can read the key.
- **`revisionHistoryLimit: 3`**: cert-manager keeps the last three `CertificateRequest`s of the CA and deletes older ones (see "The Connector CA Set").
- **Any new issuance is a Rollover**: a renewal, `cmctl renew`, or a change to the CA `Certificate`'s spec such as its issuer. Each goes through the automatic Rollover below.
- **The same setup in development and production.**

## The Connector CA Set

What Providers pin. `platform-mesh-operator` keeps no state for it: everything is derived from the CA's `CertificateRequest`s on every reconcile ([03 Operator, "Connector CA Set"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).

**The `CertificateRequest`s.** cert-manager creates one per issuance of the CA `Certificate`, next to it in the cluster resource namespace, and keeps the last three. Each carries the issued CA certificate in its status (public data; `platform-mesh-operator` never reads the CA's `Secret`), its creation time, and its revision number. `platform-mesh-operator` watches them, read-only, and validates each certificate (a CA, allowed to sign, not expired); an invalid one is reported (`Valid=False`) and never pushed.

**Derived from them**, with CR(n) the newest:

| | Rule |
|---|---|
| **Current CA** | CR(n)'s certificate |
| **Connector CA Set** | the current CA, plus CR(n−1)'s CA until its drop time; CAs listed in `distrust` are excluded at once |
| **Drop time of CR(n−1)'s CA, per Connection** | CR(n)'s creation time + 48 hours + a spread of up to 6 hours derived from the `connectionID`. By then every Connector has switched, and spreading the drop spreads the handovers it causes (see "Rollover") |
| **PM's unexpired CAs**, for the tamper check | the retained requests' certificates that haven't expired |

**Only valid CAs enter the Connector CA Set.** Each certificate must parse, be a CA, be able to sign and be unexpired; one that isn't is never pushed, and is reported as `Valid=False` on the `ConnectorCASet` (see "Signals").

**At most two of PM's CAs are unexpired at once**: each lives a year and is renewed 90 days before it expires, so the predecessor stays valid for 90 days after a renewal. Keeping three requests leaves a spare for a forced extra renewal.

**The tamper check only cares about unexpired CAs.** A pinned CA that's unexpired and isn't one of PM's unexpired CAs can only have come from someone holding the Connection Secret. An expired pinned CA verifies nothing, so it's never tampering; the push just replaces it. So a short history is enough: a Provider returning after a long absence pins only expired CAs of PM's.

**Nothing is lost on a restart or failover**: the same requests give the same result. If someone deletes the CA's `CertificateRequest`s, a Provider still pinning the predecessor within its last 90 days would be treated as tampering: its bundle is restored (which it needs anyway), its Connection Secret changed unnecessarily, and an event recorded. The runbook says not to delete them.

## ConnectorCASet

A cluster-scoped, status-only singleton in PM's runtime cluster, `connectorcasets.providerconnections.platform-mesh.io` (short name `ccs`), named `default`. It represents the Connector CA Set: `status.certificates` lists its members, identified by fingerprint, and how far each has reached the Connections. Its status is purely derived (see "The Connector CA Set"): rebuilt on every reconcile, so losing it costs nothing. Example, mid-Rollover:

```yaml
apiVersion: providerconnections.platform-mesh.io/v1alpha1
kind: ConnectorCASet
metadata:
  name: default
status:
  certificates:                          # the members: Connector CA certificates PM currently trusts
    - fingerprint: sha256:9a07…
      notAfter: "2027-10-13T00:00:00Z"
      issued: "2026-10-13T08:00:00Z"     # its CertificateRequest's creation time
      converged: false                   # not yet pinned by every Connection platform-mesh-operator reaches
    - fingerprint: sha256:4f1c…
      notAfter: "2027-01-12T00:00:00Z"
      issued: "2025-10-13T08:00:00Z"
      converged: true
      dropWindow:                        # dropped per Connection within this window
        from: "2026-10-15T08:00:00Z"     # 48 hours after its successor was issued
        to: "2026-10-15T14:00:00Z"
  shards:                                # counts per shard, for PM Administrators
    - name: root
      connections: 140
      pinnedCurrent: 139
      notReached: 1                      # Control Calls failing; listed with their last error
      lastUpdateTime: "2026-10-13T08:04:10Z"
    - name: nereus
      connections: 151
      pinnedCurrent: 151
      notReached: 0
      lastUpdateTime: "2026-10-13T08:03:55Z"
  conditions:
    - type: Valid                        # every trusted certificate parses, is a CA, can sign, isn't expired
      status: "True"
    - type: Converged                    # the current CA is pinned by every Connection platform-mesh-operator reaches
      status: "False"
      reason: Pushing
```

Printer columns: Current, Converged, Age.

### Who writes what

The whole status has one writer: `platform-mesh-operator`'s active replica. It mirrors the Connector CA Set, counts Connections per shard, and computes each CA's `converged` flag and the conditions ([03 Operator, "Connector CA Set"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).

**Convergence** counts only the Connections whose Control Calls succeed. A Connection `platform-mesh-operator` can't reach, whether its Provider is down or its control path is broken, is listed as not reached with its last error and catches up on its first successful Control Call; nothing waits for it.

## Connector Certificates

- **One cert-manager `Certificate` per shard**, issued by the CA's `ClusterIssuer`.
- **Lifetime 24 hours, renewed halfway** (`renewBefore: 12h`), `rotationPolicy: Always`. A stolen certificate is useful for at most a day, which is what lets the Connector CA do without revocation; cert-manager can be down for up to 12 hours before any certificate expires, and even then open Sessions keep running ([05 Connector](./xxx-notes-011-v5-impl-05-connector.md)). It's far shorter than any certificate elsewhere in PM on purpose.
- **Key usages**: digital signature and client authentication.
- **Minimal content**: key usages, and the shard name in the subject for audit. **No hostnames and no PM Instance identity**: served hosts travel in each Session's Hello ([01 Protocol, "Hosts a Session serves"](./xxx-notes-011-v5-impl-01-protocol.md)), and the binding to the PM Instance is the pinned CA. Nothing in a Connector Certificate changes when PM renames hosts.
- **Delivered to the Connector** as a mounted `Secret`, picked up on renewal without a restart ([09 Deployment, "Connector"](./xxx-notes-011-v5-impl-09-deployment.md)).
- **Switch rule** in the Connector: a renewed certificate is used once its CA is in the Connector CA Set and `converged`, or when the current certificate has less than about an hour left, whichever comes first. Until then the Connector keeps its current certificate, which stays valid for at least 12 hours after a renewal ([05 Connector, "Connector Certificate"](./xxx-notes-011-v5-impl-05-connector.md)).
- **All replicas of a shard's Connector share it.** Per-replica certificates are a later option with an external issuer.

## Rollover

Nobody takes part:
1. **cert-manager renews the CA** 90 days before it expires, with a new key.
2. **`platform-mesh-operator` sees the new CA certificate**, validates it, adds it to the Connector CA Set and pushes the set to every Connection, with compare-and-swap. Within minutes, every Connection it reaches pins both CAs.
3. **The new CA converges.** Connectors' certificates are renewed by cert-manager on their own schedule, now from the new CA; each Connector switches to its renewed certificate once the new CA has converged (see "Switch rule").
4. **After 48 hours, `platform-mesh-operator` drops the old CA**, Connection by Connection over the next 6 hours (see "The Connector CA Set"). At each Provider, Sessions still open with an old-CA certificate are closed with go-away and a grace period, and the Connector reopens them at once with its current certificate ([01 Protocol, "Pinned Bundle"](./xxx-notes-011-v5-impl-01-protocol.md)). Long-running watches on those Sessions relist. Only Sessions not replaced in the meantime are affected (before the Edge's maximum connection age, after drops, on Connector rollouts), and the spread keeps the relists from hitting kcp at once.

What this means for Connections:
- **Healthy Connections** keep working throughout: they trust the new CA before any Connector uses it. Their only disturbance is the handover at the drop, for Sessions still on the old CA's certificate.
- **A Connection `platform-mesh-operator` can't reach** keeps its old pins. If its Provider is down, nothing changes until it returns; its first successful Control Call brings it the current set, whatever it missed, since Control Calls don't depend on the Connector CA. If its control path is broken while Sessions work, it loses each Session at the next replacement after the Connectors switch, until its control path works again.
- **A Connection that also can't authenticate** (a lost or rejected Connection Secret) stays without Sessions: repairing its Connection Secret needs a Session. The Provider Administrators re-pin the bundle out-of-band ([07 SDK, "Administration"](./xxx-notes-011-v5-impl-07-provider-sdk.md)).
- **If `platform-mesh-operator` is down for more than about 12 hours when the CA renews**, the new CA never converges, and Connectors switch only as their old certificates expire. Open Sessions keep running, but new Sessions fail at Providers that don't pin the new CA, until `platform-mesh-operator` is back and pushes it. That's the one way a `platform-mesh-operator` outage affects traffic.
- **A Connector restarted mid-Rollover** only has its newest certificate on disk and uses it at once. If the new CA hasn't converged yet, its Sessions to Connections not yet pinning it fail for the few minutes until the push lands.

## Compromise response

- **A shard's Connector Certificate**: stop issuing it (delete or suspend its `Certificate`), and renew the CA now (below). Renewing the CA matters because Sessions opened with the stolen certificate outlive its expiry; dropping its CA closes them.
- **The CA's key**: renew the CA now (`cmctl renew`), and list the compromised CA's fingerprint in `platform-mesh-operator`'s `distrust` configuration (chart values). `platform-mesh-operator` drops it from the Connector CA Set at once, without the 48 hours or the spread, pushes the set, and never trusts it again. The Provider-side then closes every Session opened under it ([01 Protocol, "Pinned Bundle"](./xxx-notes-011-v5-impl-01-protocol.md)).

## Signals

For **PM Administrators** (Owners and Providers never see these). PM has no alerting pipeline today (the `observability` chart's Alertmanager is disabled and no chart ships alert rules), so each signal is a **metric**, plus a **condition or event** where one fits, plus a **documented alert rule** with a severity. Installations wire the rules into whatever alerting they run ([09 Deployment, "Observability"](./xxx-notes-011-v5-impl-09-deployment.md)).

| Signal | Metric (exposed by) | Condition or event | Documented rule |
|---|---|---|---|
| CA renewal failing | `providerconnections_connector_ca_expiry_seconds` (operator) | cert-manager's `Ready=False` on the CA `Certificate` | < 60 days left (renewal 30 days overdue): warning; < 30 days: critical |
| Invalid CA certificate | `providerconnections_connector_ca_valid` (operator) | `Valid=False` on the `ConnectorCASet` | == 0: critical |
| CA not converging | `providerconnections_connector_ca_connections{state="pinned_current\|behind\|not_reached"}` (operator) | `Converged=False` | Behind (reached but not current) > 0 for more than an hour: warning |
| Connector holding its previous certificate | `providerconnections_connector_certificate_pending_switch_seconds{shard}` (Connector) | — | > 6 hours: warning (the new CA hasn't converged) |
| Connector Certificate not renewed | `providerconnections_connector_certificate_expiry_seconds{shard}` (Connector) | Event on the Connector's `Certificate` (cert-manager) | < 6 hours left (half the renewal budget used): warning |
| Connector Certificate expired | same | Connector readiness reports it (rollout tooling can hold) | ≤ 0: critical. From then on every Session that ends is an outage for its Connection on that shard ([05 Connector, "Opening a Session"](./xxx-notes-011-v5-impl-05-connector.md)) |
| A Pinned Bundle held an unexpired CA that isn't one of PM's | `providerconnections_connector_ca_bundle_tampered_total` (operator) | An event on the `Provider` ([03 Operator, "Connector CA Set"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)) | Increase: critical (a Connection Secret was used by someone other than PM) |

## Work breakdown

1. The status-only `ConnectorCASet` CRD and its printer columns.
2. Chart templates: the CA `Certificate` and its issuer (self-signed or `issuerRef`), the CA `ClusterIssuer`, per-shard Connector `Certificate`s, the `distrust` configuration ([09 Deployment](./xxx-notes-011-v5-impl-09-deployment.md)).
3. `platform-mesh-operator`: watching the CA's `CertificateRequest`s, deriving the Connector CA Set, drop times and unexpired CAs from them, the `ConnectorCASet` status ([03 Operator](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
4. Connector: the switch rule ([05 Connector](./xxx-notes-011-v5-impl-05-connector.md)).
5. Unit tests for the rules above, and chart-output tests for the templates.
6. Runbooks: what the `ConnectorCASet` status means, Connections not reached, compromise, an expired Connector Certificate (what ends open Sessions, what can't start), the CA's `CertificateRequest`s (don't delete them; with approver-policy, the policy must allow them), with the documented alert rules from "Signals".

## Decided

- **The CA is managed by cert-manager and rolled over automatically**: 1 year, renewed 90 days before expiry, new key each time; self-signed by default, `issuerRef` optional.
- **Everything is derived from the CA's `CertificateRequest`s**, the last three of which cert-manager keeps; `platform-mesh-operator` stores nothing and never reads key `Secret`s.
- **The previous CA is dropped 48 hours after its successor is issued, spread over 6 hours per Connection**, or at once if distrusted. Connectors don't replace Sessions because of a Rollover; the drop's handover does.
- **The tamper check only considers unexpired CAs.**
- **Only Connections `platform-mesh-operator` reaches count for convergence**; nothing waits for the others.
- **An operator outage of more than about 12 hours at a CA renewal is accepted**: new Sessions fail until `platform-mesh-operator` pushes the new CA; open Sessions keep running. Letting `platform-mesh-operator` trigger CA renewals itself, only while it's healthy, would avoid it, but needs rights on cert-manager's `Certificate`s in cert-manager's namespace, which is cert-manager's scope, not `platform-mesh-operator`'s.
- **Connector Certificates carry no hostnames or PM Instance identity**; served hosts are in the Session's Hello.
- **No intermediates**: Connector Certificates are issued directly by the CA that Providers pin. The CA rolls over yearly by itself, so an intermediate would add nothing; with an external PKI, PM's CA is itself the intermediate, pinned as the trust anchor. **No name constraints** on the CA: there are no meaningful names in Connector Certificates to constrain.
- **No PM Instance identifier**: the CA of each PM Instance is the binding. A PM-wide identity for PM Instances would be a broader design.
- **Connector Certificates**: 24 hours, renewed halfway, no revocation. kcp doesn't revoke its certificates either, but relies on removing RBAC bindings, which Providers can't do for PM; a short lifetime is the equivalent.
- **Naming**: Connector CA, Connector Certificates, Connector CA Set, `ConnectorCASet`.
