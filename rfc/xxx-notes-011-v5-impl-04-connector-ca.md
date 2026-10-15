# v5 implementation plan: connector CA

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). The PKI that authenticates [connectors](./xxx-notes-011-v5-impl-05-connector.md) to Providers: a connector CA managed by cert-manager, connector certificates issued from it, the trusted set of CA certificates Providers pin, and the `ConnectorCABundle` resource that reports it. The design rationale is in v5, "Session credentials: the connector CA" and "Distributing the connector CA to Providers".

## What it provides

- **Connector certificates**: one per shard's connector, short-lived, used as the client certificate in each session's inner TLS, which is how sessions are authenticated ([01 Protocol, "Sessions"](./xxx-notes-011-v5-impl-01-protocol.md)).
- **The trusted set**: the connector CA certificates Providers pin, which platform-mesh-operator sends at registration and pushes whenever it changes ([01 Protocol, "Connector CA bundle"](./xxx-notes-011-v5-impl-01-protocol.md), [03 Operator, "Connector CA trusted set"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
- **Rollover progress**, reported in one resource, `ConnectorCABundle`.

## Principles

- **cert-manager owns the CA, and the rollover is automatic.** cert-manager renews the CA with a new key; platform-mesh-operator pushes the new CA certificate to every Provider; connectors switch to certificates from the new CA only once Providers trust it. Nobody takes part.
- **Public and private halves are separate.** platform-mesh-operator and connectors only ever see public certificates. Keys stay with cert-manager or the external PKI.
- **Connector certificates are nobody's pin**: short-lived, new key on every renewal.
- **Each installation has its own CA.** Providers pin the CA cert-manager manages for this installation, never a shared root, so a session is bound to its installation ([01 Protocol, "What binds a session to its PM installation"](./xxx-notes-011-v5-impl-01-protocol.md)).

## The CA

One cert-manager `Certificate` with `isCA: true`, rendered by the `providerconnections` chart ([09 Deployment, "Connector CA"](./xxx-notes-011-v5-impl-09-deployment.md)):
- **Lifetime 1 year, renewed 90 days before expiry**, `rotationPolicy: Always`: every renewal has a new key, so a stolen CA key is useful for at most about a year.
- **Issued by** a self-signed issuer by default, or by an `issuerRef` (e.g. Vault or a cloud CA). With an `issuerRef` the CA is an intermediate under the external root; Providers pin the intermediate, not the root, so installations sharing an external PKI still have their own CAs. Either way, the CA's own key sits in its Secret in the cluster; an external PKI only keeps the root out of it.
- **In cert-manager's cluster resource namespace** (the namespace cert-manager looks in for Secrets of cluster-scoped resources; by default the one it runs in), which only cert-manager reads. A `ClusterIssuer` of type CA on its Secret issues the connector certificates, in the connectors' namespace. Neither platform-mesh-operator nor connectors can read the key.
- **`revisionHistoryLimit: 3`**: cert-manager keeps the last three `CertificateRequest`s of the CA and deletes older ones (see "The trusted set").
- **Any new issuance is a rollover**: a renewal, `cmctl renew`, or a change to the CA `Certificate`'s spec such as its issuer. Each goes through the automatic rollover below.
- **The same setup in development and production.**

## The trusted set

What Providers pin. platform-mesh-operator keeps no state for it: everything is derived from the CA's `CertificateRequest`s on every reconcile ([03 Operator, "Connector CA trusted set"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).

**The `CertificateRequest`s.** cert-manager creates one per issuance of the CA `Certificate`, next to it in the cluster resource namespace, and keeps the last three. Each carries the issued CA certificate in its status (public data; platform-mesh-operator never reads the CA's Secret), its creation time, and its revision number. platform-mesh-operator watches them, read-only, and validates each certificate (a CA, allowed to sign, not expired); an invalid one is reported (`Valid=False`) and never pushed.

**Derived from them**, with CR(n) the newest:

| | Rule |
|---|---|
| **Current CA** | CR(n)'s certificate |
| **Trusted set** | the current CA, plus CR(n−1)'s CA until its drop time; CAs listed in `distrust` are excluded at once |
| **Drop time of CR(n−1)'s CA, per connection** | CR(n)'s creation time + 48 hours + a spread of up to 6 hours derived from the `connectionID`. By then every connector has switched, and spreading the drop spreads the handovers it causes (see "Rollover") |
| **PM's unexpired CAs**, for the tamper check | the retained requests' certificates that haven't expired |

**At most two of PM's CAs are unexpired at once**: each lives a year and is renewed 90 days before it expires, so the predecessor stays valid for 90 days after a renewal. Keeping three requests leaves a spare for a forced extra renewal.

**The tamper check only cares about unexpired CAs.** A pinned CA that's unexpired and isn't one of PM's unexpired CAs can only have come from someone holding the connection secret. An expired pinned CA verifies nothing, so it's never tampering; the push just replaces it. So a short history is enough: a Provider returning after a long absence pins only expired CAs of PM's.

**Nothing is lost on a restart or failover**: the same requests give the same result. If someone deletes the CA's `CertificateRequest`s, a Provider still pinning the predecessor within its last 90 days would be treated as tampering: its bundle is restored (which it needs anyway), its secret changed unnecessarily, and an event recorded. The runbook says not to delete them.

## ConnectorCABundle

A cluster-scoped, status-only singleton in PM's runtime cluster, `connectorcabundles.providerconnections.platform-mesh.io` (short name `ccab`), named `default`. Its status is purely derived (see "The trusted set"): rebuilt on every reconcile, so losing it costs nothing. Example, mid-rollover:

```yaml
apiVersion: providerconnections.platform-mesh.io/v1alpha1
kind: ConnectorCABundle
metadata:
  name: default
status:
  trusted:                               # what Providers should pin
    - fingerprint: sha256:9a07…
      notAfter: "2027-10-13T00:00:00Z"
      issued: "2026-10-13T08:00:00Z"     # its CertificateRequest's creation time
      converged: false                   # not yet pinned by every connection platform-mesh-operator reaches
    - fingerprint: sha256:4f1c…
      notAfter: "2027-01-12T00:00:00Z"
      issued: "2025-10-13T08:00:00Z"
      converged: true
      dropWindow:                        # dropped per connection within this window
        from: "2026-10-15T08:00:00Z"     # 48 hours after its successor was issued
        to: "2026-10-15T14:00:00Z"
  shards:                                # counts per shard, for the people running PM
    - name: root
      connections: 140
      pinnedCurrent: 139
      notReached: 1                      # control calls failing; listed with their last error
      lastUpdateTime: "2026-10-13T08:04:10Z"
    - name: nereus
      connections: 151
      pinnedCurrent: 151
      notReached: 0
      lastUpdateTime: "2026-10-13T08:03:55Z"
  conditions:
    - type: Valid                        # every trusted certificate parses, is a CA, can sign, isn't expired
      status: "True"
    - type: Converged                    # the current CA is pinned by every connection platform-mesh-operator reaches
      status: "False"
      reason: Pushing
```

Printer columns: Current, Converged, Age.

### Who writes what

The whole status has one writer: platform-mesh-operator's active replica. It mirrors the trusted set, counts connections per shard, and computes each CA's `converged` flag and the conditions ([03 Operator, "Connector CA trusted set"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).

**Convergence** counts only the connections whose control calls succeed. A connection platform-mesh-operator can't reach, whether its Provider is down or its control path is broken, is listed as not reached with its last error and catches up on its first successful control call; nothing waits for it.

## Connector certificates

- **One cert-manager `Certificate` per shard**, issued by the CA's `ClusterIssuer`.
- **Lifetime 24 hours, renewed halfway** (`renewBefore: 12h`), `rotationPolicy: Always`. A stolen certificate is useful for at most a day, which is what lets the connector CA do without revocation; cert-manager can be down for up to 12 hours before any certificate expires, and even then open sessions keep running ([05 Connector](./xxx-notes-011-v5-impl-05-connector.md)). It's far shorter than any certificate elsewhere in PM on purpose.
- **Key usages**: digital signature and client authentication.
- **Minimal content**: key usages, and the shard name in the subject for audit. **No hostnames and no installation identity**: served hosts travel in each session's hello ([01 Protocol, "Hosts a session serves"](./xxx-notes-011-v5-impl-01-protocol.md)), and the installation binding is the pinned CA. Nothing in a connector certificate changes when PM renames hosts.
- **Delivered to the connector** as a mounted Secret, picked up on renewal without a restart ([09 Deployment, "Connector (providerconnections-vw)"](./xxx-notes-011-v5-impl-09-deployment.md)).
- **Switch rule** in the connector: a renewed certificate is used once its CA is in the trusted set and `converged`, or when the current certificate has less than about an hour left, whichever comes first. Until then the connector keeps its current certificate, which stays valid for at least 12 hours after a renewal ([05 Connector, "Connector certificate"](./xxx-notes-011-v5-impl-05-connector.md)).
- **All replicas of a shard's connector share it.** Per-replica certificates are a later option with an external issuer.

## Rollover

Nobody takes part:
1. **cert-manager renews the CA** 90 days before it expires, with a new key.
2. **platform-mesh-operator sees the new CA certificate**, validates it, adds it to the trusted set and pushes the set to every connection, with compare-and-swap. Within minutes, every connection it reaches pins both CAs.
3. **The new CA converges.** Connectors' certificates are renewed by cert-manager on their own schedule, now from the new CA; each connector switches to its renewed certificate once the new CA has converged (see "Switch rule").
4. **After 48 hours, platform-mesh-operator drops the old CA**, connection by connection over the next 6 hours (see "The trusted set"). At each Provider, sessions still open with an old-CA certificate are closed with go-away and a grace period, and the connector reopens them at once with its current certificate ([01 Protocol, "Connector CA bundle"](./xxx-notes-011-v5-impl-01-protocol.md)). Long-running watches on those sessions relist. Only sessions not replaced in the meantime are affected (before the edge's maximum connection age, after drops, on connector rollouts), and the spread keeps the relists from hitting kcp at once.

What this means for connections:
- **Healthy connections** keep working throughout: they trust the new CA before any connector uses it. Their only disturbance is the handover at the drop, for sessions still on the old CA's certificate.
- **A connection platform-mesh-operator can't reach** keeps its old pins. If its Provider is down, nothing changes until it returns; its first successful control call brings it the current set, whatever it missed, since control calls don't depend on the connector CA. If its control path is broken while sessions work, it loses each session at the next replacement after the connectors switch, until its control path works again.
- **A connection that also can't authenticate** (a lost or rejected secret) stays without sessions: repairing its secret needs a session. The Provider's administrators re-pin the bundle out-of-band ([07 SDK, "Administration"](./xxx-notes-011-v5-impl-07-provider-sdk.md)).
- **If platform-mesh-operator is down for more than about 12 hours when the CA renews**, the new CA never converges, and connectors switch only as their old certificates expire. Open sessions keep running, but new sessions fail at Providers that don't pin the new CA, until platform-mesh-operator is back and pushes it. That's the one way a control-plane outage affects traffic.
- **A connector restarted mid-rollover** only has its newest certificate on disk and uses it at once. If the new CA hasn't converged yet, its sessions to connections not yet pinning it fail for the few minutes until the push lands.

## Compromise response

- **A shard's connector certificate**: stop issuing it (delete or suspend its `Certificate`), and renew the CA now (below). Renewing the CA matters because sessions opened with the stolen certificate outlive its expiry; dropping its CA closes them.
- **The CA's key**: renew the CA now (`cmctl renew`), and list the compromised CA's fingerprint in platform-mesh-operator's `distrust` configuration (chart values). platform-mesh-operator drops it from the trusted set at once, without the 48 hours or the spread, pushes the set, and never trusts it again. The Provider-side then closes every session opened under it ([01 Protocol, "Connector CA bundle"](./xxx-notes-011-v5-impl-01-protocol.md)).

## Signals

For the **people running the PM installation** (owners and Providers never see these). PM has no alerting pipeline today (the `observability` chart's Alertmanager is disabled and no chart ships alert rules), so each signal is a **metric**, plus a **condition or event** where one fits, plus a **documented alert rule** with a severity. Installations wire the rules into whatever alerting they run ([09 Deployment, "Observability"](./xxx-notes-011-v5-impl-09-deployment.md)).

| Signal | Metric (exposed by) | Condition or event | Documented rule |
|---|---|---|---|
| CA renewal failing | `providerconnections_connector_ca_expiry_seconds` (operator) | cert-manager's `Ready=False` on the CA `Certificate` | < 60 days left (renewal 30 days overdue): warning; < 30 days: critical |
| Invalid CA certificate | `providerconnections_connector_ca_valid` (operator) | `Valid=False` on the `ConnectorCABundle` | == 0: critical |
| CA not converging | `providerconnections_connector_ca_connections{state="pinned_current\|behind\|not_reached"}` (operator) | `Converged=False` | Behind (reached but not current) > 0 for more than an hour: warning |
| Connector holding its previous certificate | `providerconnections_connector_certificate_pending_switch_seconds{shard}` (connector) | — | > 6 hours: warning (the new CA hasn't converged) |
| Connector certificate not renewed | `providerconnections_connector_certificate_expiry_seconds{shard}` (connector) | Event on the connector's `Certificate` (cert-manager) | < 6 hours left (half the renewal budget used): warning |
| Connector certificate expired | same | Connector readiness reports it (rollout tooling can hold) | ≤ 0: critical. From then on every session that ends is an outage for its connection on that shard ([05 Connector, "Opening a session"](./xxx-notes-011-v5-impl-05-connector.md)) |
| A pinned bundle held an unexpired CA that isn't one of PM's | `providerconnections_connector_ca_bundle_tampered_total` (operator) | An event on the `Provider` ([03 Operator, "Connector CA trusted set"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)) | Increase: critical (a connection secret was used by someone other than PM) |

## Work breakdown

1. The status-only `ConnectorCABundle` CRD and its printer columns.
2. Chart templates: the CA `Certificate` and its issuer (self-signed or `issuerRef`), the CA `ClusterIssuer`, per-shard connector `Certificate`s, the `distrust` configuration ([09 Deployment](./xxx-notes-011-v5-impl-09-deployment.md)).
3. platform-mesh-operator: watching the CA's `CertificateRequest`s, deriving the trusted set, drop times and unexpired CAs from them, the `ConnectorCABundle` status ([03 Operator](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
4. Connector: the switch rule ([05 Connector](./xxx-notes-011-v5-impl-05-connector.md)).
5. Runbooks: what the `ConnectorCABundle` status means, connections not reached, compromise, an expired connector certificate (what ends open sessions, what can't start), the CA's `CertificateRequest`s (don't delete them; with approver-policy, the policy must allow them), with the documented alert rules from "Signals".

## Decided

- **The CA is managed by cert-manager and rolled over automatically**: 1 year, renewed 90 days before expiry, new key each time; self-signed by default, `issuerRef` optional.
- **Everything is derived from the CA's `CertificateRequest`s**, the last three of which cert-manager keeps; platform-mesh-operator stores nothing and never reads key Secrets.
- **The previous CA is dropped 48 hours after its successor is issued, spread over 6 hours per connection**, or at once if distrusted. Connectors don't replace sessions because of a rollover; the drop's handover does.
- **The tamper check only considers unexpired CAs.**
- **Only connections platform-mesh-operator reaches count for convergence**; nothing waits for the others.
- **An operator outage of more than about 12 hours at a CA renewal is accepted**: new sessions fail until platform-mesh-operator pushes the new CA; open sessions keep running. Letting platform-mesh-operator trigger CA renewals itself, only while it's healthy, would avoid it, but needs rights on cert-manager's `Certificate`s in cert-manager's namespace, which is cert-manager's scope, not platform-mesh-operator's.
- **Connector certificates carry no hostnames or installation identity**; served hosts are in the session's hello.
- **No intermediates**: connector certificates are issued directly by the CA Providers pin. The CA rolls over yearly by itself, so an intermediate would add nothing; with an external PKI, PM's CA is itself the intermediate, pinned as the trust anchor. **No name constraints** on the CA: there are no meaningful names in connector certificates to constrain.
- **No installation identifier**: the per-installation CA is the binding. A PM-wide installation identity would be a broader design.
- **Connector certificates**: 24 hours, renewed halfway, no revocation. kcp doesn't revoke its certificates either, but relies on removing RBAC bindings, which Providers can't do for PM; a short lifetime is the equivalent.
- **Naming**: connector CA, connector certificates, trusted set, `ConnectorCABundle`.
