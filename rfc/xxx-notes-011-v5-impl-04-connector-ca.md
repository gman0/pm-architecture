# v5 implementation plan: connector CA

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). The PKI that authenticates [connectors](./xxx-notes-011-v5-impl-05-connector.md) to Providers: a connector CA managed by cert-manager, connector certificates issued from it, the trusted set of CA certificates Providers pin, and the `ConnectorCABundle` resource that reports it. The design rationale is in v5, "Session credentials: the connector CA" and "Distributing the connector CA to Providers".

## What it provides

- **Connector certificates**: one per shard's connector, short-lived, used to sign session assertions and as the inner TLS client certificate ([01 Protocol, "Sessions"](./xxx-notes-011-v5-impl-01-protocol.md)).
- **The trusted set**: the connector CA certificates Providers pin, which the operator sends at registration and pushes whenever it changes ([01 Protocol, "Connector CA bundle"](./xxx-notes-011-v5-impl-01-protocol.md), [03 Operator, "Connector CA trusted set"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)).
- **Rollover progress**, reported in one resource, `ConnectorCABundle`.

## Principles

- **cert-manager owns the CA, and the rollover is automatic.** cert-manager renews the CA with a new key; the operator pushes the new CA certificate to every Provider; connectors switch to certificates from the new CA only once Providers trust it. Nobody takes part.
- **Public and private halves are separate.** The operator and connectors only ever see public certificates. Keys stay with cert-manager or the external PKI.
- **Connector certificates are nobody's pin**: short-lived, new key on every renewal.
- **Each installation has its own CA.** Providers pin the CA cert-manager manages for this installation, never a shared root, so a session is bound to its installation ([01 Protocol, "What binds a session to its PM installation"](./xxx-notes-011-v5-impl-01-protocol.md)).

## The CA

One cert-manager `Certificate` with `isCA: true`, rendered by the `providerconnections` chart ([09 Deployment, "Connector CA"](./xxx-notes-011-v5-impl-09-deployment.md)):
- **Lifetime 1 year, renewed 90 days before expiry**, `rotationPolicy: Always`: every renewal has a new key, so a stolen CA key is useful for at most about a year.
- **Issued by** a self-signed issuer by default, or by an `issuerRef` (e.g. Vault or a cloud CA). With an `issuerRef` the CA is an intermediate under the external root; Providers pin the intermediate, not the root, so installations sharing an external PKI still have their own CAs. Either way, the CA's own key sits in its Secret in the cluster; an external PKI only keeps the root out of it.
- **In cert-manager's cluster resource namespace**, which only cert-manager reads. A `ClusterIssuer` of type CA on its Secret issues the connector certificates. Neither the operator nor connectors can read the key.
- **The same setup in development and production.**

## The trusted set

What Providers pin. Kept by the operator ([03 Operator, "Connector CA trusted set"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)):
- **New CA certificates come from the CA's `CertificateRequest`s**, whose status carries the issued certificate: public data, no key access. The operator validates each one (a CA, allowed to sign, not expired) before adding it; an invalid one is reported (`Valid=False`) and never pushed.
- **The set is the current CA plus the previous one**, until 48 hours after the current one was first seen. By then every connector has switched (see "Connector certificates"), and no connector certificate from the previous CA is still valid.
- **A history of every CA PM has trusted** is kept alongside, by fingerprint. The operator's tamper check compares Providers' pinned bundles against it, so a Provider returning after a long outage, still pinning an old CA, isn't mistaken for tampering.
- **Both live in a ConfigMap in the operator's namespace**, so they survive restarts; the `ConnectorCABundle` status mirrors them for people and connectors.

## ConnectorCABundle

A cluster-scoped, status-only singleton in PM's runtime cluster, `connectorcabundles.providerconnections.platform-mesh.io` (short name `ccab`), named `default`. Example, mid-rollover:

```yaml
apiVersion: providerconnections.platform-mesh.io/v1alpha1
kind: ConnectorCABundle
metadata:
  name: default
status:
  trusted:                               # what Providers should pin
    - fingerprint: sha256:9a07…
      notAfter: "2027-10-13T00:00:00Z"
      firstSeen: "2026-10-13T08:00:00Z"
      converged: false                   # not yet pinned by every connection the operator reaches
    - fingerprint: sha256:4f1c…
      notAfter: "2027-01-12T00:00:00Z"
      firstSeen: "2025-10-13T08:00:00Z"
      converged: true
      dropAfter: "2026-10-15T08:00:00Z"  # 48 hours after its successor was first seen
  shards:                                # one entry per shard worker of the operator
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
    - type: Converged                    # the current CA is pinned by every connection the operator reaches
      status: "False"
      reason: Pushing
```

Printer columns: Current, Converged, Age.

### Who writes what

| Part | Written by |
|---|---|
| `status.shards[<shard>]` | That shard's operator worker, by server-side apply with a per-shard field manager; the list is keyed by shard name, so workers never conflict ([03 Operator, "Partitioning by shard"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)) |
| The rest of `status` | One aggregator in the operator, under its own Lease. It mirrors the trusted set and computes convergence and conditions from the shard entries |

**Convergence** counts only the connections whose control calls succeed. A connection the operator can't reach, whether its Provider is down or its control path is broken, is listed as not reached with its last error and catches up on its first successful control call; nothing waits for it.

## Connector certificates

- **One cert-manager `Certificate` per shard**, issued by the CA's `ClusterIssuer`.
- **Lifetime 24 hours, renewed halfway** (`renewBefore: 12h`), `rotationPolicy: Always`. A stolen certificate is useful for at most a day, which is what lets the connector CA do without revocation; cert-manager can be down for up to 12 hours before any certificate expires, and even then open sessions keep running ([05 Connector](./xxx-notes-011-v5-impl-05-connector.md)). It's far shorter than any certificate elsewhere in PM on purpose.
- **Key usages**: digital signature and client authentication.
- **Minimal content**: key usages, and the shard name in the subject for audit. **No hostnames and no installation identity**: served hosts travel in each session's assertion ([01 Protocol, "Hosts a session serves"](./xxx-notes-011-v5-impl-01-protocol.md)), and the installation binding is the pinned CA. Nothing in a connector certificate changes when PM renames hosts.
- **Delivered to the connector** as a mounted Secret, picked up on renewal without a restart ([09 Deployment, "Connector"](./xxx-notes-011-v5-impl-09-deployment.md)).
- **Switch rule** in the connector: a renewed certificate is used once its CA is in the trusted set and `converged`, or when the current certificate has less than about an hour left, whichever comes first. Until then the connector keeps its current certificate, which stays valid for at least 12 hours after a renewal ([05 Connector, "Connector certificate"](./xxx-notes-011-v5-impl-05-connector.md)).
- **All replicas of a shard's connector share it.** Per-replica certificates are a later option with an external issuer.

## Rollover

Nobody takes part:
1. **cert-manager renews the CA** 90 days before it expires, with a new key.
2. **The operator sees the new CA certificate**, validates it, adds it to the trusted set and pushes the set to every connection, with compare-and-swap. Within minutes, every connection it reaches pins both CAs.
3. **The new CA converges.** Connectors' certificates are renewed by cert-manager on their own schedule, now from the new CA; each connector switches to its renewed certificate once the new CA has converged (see "Switch rule").
4. **Connectors replace their remaining old-certificate sessions** gracefully, opening new sessions first ([05 Connector](./xxx-notes-011-v5-impl-05-connector.md)).
5. **After 48 hours the operator drops the old CA** and pushes the set again. Any session still open with an old-CA certificate is closed by the Provider-side with go-away, and reconnects at once ([01 Protocol, "Connector CA bundle"](./xxx-notes-011-v5-impl-01-protocol.md)).

What this means for connections:
- **Healthy connections** see no interruption: they trust the new CA before any connector uses it, and open sessions are never checked again.
- **A connection the operator can't reach** keeps its old pins. If its Provider is down, nothing changes until it returns; its first successful control call brings it the current set, whatever it missed, since control calls don't depend on the connector CA. If its control path is broken while sessions work, it loses each session at the next replacement after the connectors switch, until its control path works again.
- **A connection that also can't authenticate** (a lost or rejected secret) stays without sessions: repairing its secret needs a session. The Provider's administrators re-pin the bundle out-of-band ([07 SDK, "Administration"](./xxx-notes-011-v5-impl-07-provider-sdk.md)).
- **If the operator is down for more than about 12 hours when the CA renews**, the new CA never converges, and connectors switch only as their old certificates expire. Open sessions keep running, but new sessions fail at Providers that don't pin the new CA, until the operator is back and pushes it. That's the one way a control-plane outage affects traffic.
- **A connector restarted mid-rollover** only has its newest certificate on disk and uses it at once. If the new CA hasn't converged yet, its sessions to connections not yet pinning it fail for the few minutes until the push lands.

## Compromise response

- **A shard's connector certificate**: stop issuing it (delete or suspend its `Certificate`), and renew the CA now (below). Renewing the CA matters because sessions opened with the stolen certificate outlive its expiry; dropping its CA closes them.
- **The CA's key**: renew the CA now (`cmctl renew`), and list the compromised CA's fingerprint in the operator's `distrust` configuration (chart values). The operator drops it from the trusted set at once instead of after 48 hours, pushes the set, and never trusts it again. The Provider-side then closes every session opened under it ([01 Protocol, "Connector CA bundle"](./xxx-notes-011-v5-impl-01-protocol.md)).

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
| A pinned bundle held a CA PM never trusted | `providerconnections_connector_ca_bundle_tampered_total` (operator) | `ConnectorCABundleTampered` on the `Provider`, and an event ([03 Operator, "Connector CA trusted set"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)) | Increase: critical (a connection secret was used by someone other than PM) |

## Work breakdown

1. The status-only `ConnectorCABundle` CRD and its printer columns.
2. Chart templates: the CA `Certificate` and its issuer (self-signed or `issuerRef`), the CA `ClusterIssuer`, per-shard connector `Certificate`s, the `distrust` configuration ([09 Deployment](./xxx-notes-011-v5-impl-09-deployment.md)).
3. Operator: reading CA certificates from `CertificateRequest`s, the trusted set and its history, per-shard status entries, the aggregator ([03 Operator](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)).
4. Connector: the switch rule and replacing old-certificate sessions ([05 Connector](./xxx-notes-011-v5-impl-05-connector.md)).
5. Runbooks: what the `ConnectorCABundle` status means, connections not reached, compromise, with the documented alert rules from "Signals".

## Decided

- **The CA is managed by cert-manager and rolled over automatically**: 1 year, renewed 90 days before expiry, new key each time; self-signed by default, `issuerRef` optional.
- **CA certificates are read from `CertificateRequest` status**; the operator never reads key Secrets.
- **The previous CA is dropped 48 hours after its successor is first seen**, or at once if distrusted.
- **Only connections the operator reaches count for convergence**; nothing waits for the others.
- **Connector certificates carry no hostnames or installation identity**; served hosts are in the session assertion. **No name constraints** on the CA: there are no meaningful names in connector certificates to constrain.
- **No installation identifier**: the per-installation CA is the binding. A PM-wide installation identity would be a broader design.
- **Connector certificates**: 24 hours, renewed halfway, no revocation. kcp doesn't revoke its certificates either, but relies on removing RBAC bindings, which Providers can't do for PM; a short lifetime is the equivalent.
- **Naming**: connector CA, connector certificates, trusted set, `ConnectorCABundle`.
