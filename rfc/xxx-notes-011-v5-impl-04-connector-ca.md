# v5 implementation plan: Connector CA

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). The PKI that authenticates [Connectors](./xxx-notes-011-v5-impl-05-connector.md) to Providers: a Connector CA managed by cert-manager, Connector Certificates issued from it, the Connector CA Set that Providers pin, and the `ConnectorCASet` resource that reports it. The design rationale is in v5, "Session credentials: the Connector CA" and "Distributing the Connector CA to Providers".

## What it provides

- **Connector Certificates**: one per shard's Connector, short-lived, used as the client certificate in each Session's Inner TLS, which is how Sessions are authenticated ([01 Protocol, "Sessions"](./xxx-notes-011-v5-impl-01-protocol.md)).
- **The Connector CA Set**: the Connector CA certificates Providers pin, which `platform-mesh-operator` sends at Registration and pushes whenever it changes ([01 Protocol, "Pinned Bundle"](./xxx-notes-011-v5-impl-01-protocol.md), [03 Operator, "Connector CA Set"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
- **Rollover progress and distrust**, in one resource, `ConnectorCASet`.

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
- **`revisionHistoryLimit: 10`**: cert-manager keeps the last ten `CertificateRequest`s of the CA and deletes older ones (see "The Connector CA Set"). Ten covers every CA that's still unexpired within a year, even with several forced renewals on top of the yearly one.
- **Any new issuance is a Rollover**: a renewal, `cmctl renew`, or a change to the CA `Certificate`'s spec such as its issuer. Each goes through the automatic Rollover below.
- **The same setup in development and production.**

## The Connector CA Set

What Providers pin. `platform-mesh-operator` keeps no state for it: everything is derived on every reconcile from the CA's `CertificateRequest`s, the `ConnectorCASet`'s `spec.distrust`, and the Connectors' current certificates ([03 Operator, "Connector CA Set"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).

**The `CertificateRequest`s.** cert-manager creates one per issuance of the CA `Certificate`, next to it in the cluster resource namespace, and keeps the last ten. Each carries the issued CA certificate in its status (public data; `platform-mesh-operator` never reads the CA's `Secret`), its conditions, and its revision number (the `cert-manager.io/certificate-revision` annotation). `platform-mesh-operator` watches them, read-only, and validates each certificate (a CA, allowed to sign, not expired). A request that's pending (e.g. waiting for approval), failed or denied holds no certificate and is ignored. An issued certificate that fails validation is reported (`Valid=False`), skipped, and never pushed.

**The Connectors' certificates.** `platform-mesh-operator` also reads, read-only, the Connector Certificates' `CertificateRequest`s in the Connectors' namespace, to see which CA each shard's current Connector Certificate was issued by. That's the leaf gate below.

**Derived from them**:

| | Rule |
|---|---|
| **Order** | by revision, not creation time |
| **Current CA** | the newest request that is `Ready=True`, holds a certificate and passes validation. If the newest issued one is invalid, the previous valid CA stays current |
| **Issued** | when a request became `Ready=True`, i.e. when Connectors could first get certificates from its CA; not its creation time, since approval can take hours |
| **Drop time of a previous CA, per Connection** | its successor's issue time + 48 hours + a spread of up to 6 hours derived from the `connectionID`. By then every Connector has switched, and spreading the drop spreads the handovers it causes (see "Rollover"). Each previous CA has its own, so two issuances within 48 hours don't drop a CA still in use |
| **Leaf gate** | a previous CA isn't dropped, whatever its drop time, while any Connector's current certificate still chains to it |
| **Connector CA Set** | the current CA, plus every retained previous CA whose drop time hasn't passed or that the leaf gate holds; distrusted CAs follow their own gate (see "Compromise response") |
| **PM's unexpired CAs**, for the tamper check | the retained requests' certificates that haven't expired, except distrusted ones |

**Only valid CAs enter the Connector CA Set**, and it's never empty: if deriving it would leave no CA (for example a distrust of the only current CA), nothing is pushed, the `ConnectorCASet` shows `Valid=False`, and the alert fires (see "Signals").

**The invariant behind the timings**: the 48 hours before a drop must exceed a Connector Certificate's renewal time (12 hours) plus convergence (minutes, or up to hours with `platform-mesh-operator` down). The leaf gate holds the drop when that doesn't hold.

**The tamper check only cares about unexpired CAs.** A pinned CA that's unexpired and isn't one of PM's unexpired CAs can only have come from someone holding the Connection Secret. An expired pinned CA verifies nothing, so it's never tampering; the push just replaces it. So the last ten requests are enough: a Provider returning after a long absence pins only expired CAs of PM's. A pinned distrusted CA isn't PM's anymore, but isn't treated as tampering either ([03 Operator, "Connector CA Set"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).

**Nothing is lost on a restart or failover**: the same requests give the same result. If someone deletes the CA's `CertificateRequest`s, a Provider still pinning an unexpired CA whose request is gone would be treated as tampering: its bundle is restored (which it needs anyway), its Connection Secret changed unnecessarily, and an event recorded. The runbook says not to delete them, and an alert fires when the oldest retained request's CA is still unexpired while the limit is reached, so the limit is raised before that happens (see "Signals").

## ConnectorCASet

A cluster-scoped singleton in PM's runtime cluster, `connectorcasets.providers.platform-mesh.io` (short name `ccs`), named `default`, shipped with `platform-mesh-operator`, which creates it if it's missing. Its `spec` is written by PM Administrators and holds only `distrust`; its `status` is purely derived (see "The Connector CA Set"), rebuilt on every reconcile, so losing it costs nothing. Example, mid-Rollover, with a previous CA being distrusted:

```yaml
apiVersion: providers.platform-mesh.io/v1alpha1
kind: ConnectorCASet
metadata:
  name: default
spec:
  distrust:                                  # written by PM Administrators
    - fingerprint: key-sha256:4f1c…          # a CA's key, not a certificate
      immediate: false                       # true skips the gate (see "Compromise response")
status:
  certificates:                              # every unexpired CA PM knows
    - certFingerprint: cert-sha256:9a07…
      keyFingerprint: key-sha256:77d2…
      issued: "2026-10-13T08:00:00Z"         # when its CertificateRequest became Ready
      notAfter: "2027-10-13T08:00:00Z"
      state: Current
    - certFingerprint: cert-sha256:e510…
      keyFingerprint: key-sha256:4f1c…
      issued: "2026-01-11T08:00:00Z"
      notAfter: "2027-01-11T08:00:00Z"
      state: DistrustPending                 # Current | Previous | DistrustPending | Distrusted | Dropped
      distrust:
        since: "2026-10-13T08:05:00Z"
        waitingFor: [nereus]                 # shards whose Connector Certificate still chains to it
  distrust:                                  # one result per spec.distrust entry
    - fingerprint: key-sha256:4f1c…
      result: Pending                        # Pending | Applied | Unmatched
  conditions:
    - type: Valid                            # the set isn't empty; every trusted certificate parses, is a CA, can sign, isn't expired
      status: "True"
    - type: Converged                        # the current CA is pinned by every Connection platform-mesh-operator reaches
      status: "False"
      reason: Pushing
```

- **`state`** per certificate: `Current` (the current CA), `Previous` (still in the set, waiting for its drop time or the leaf gate), `DistrustPending` (distrusted, still in the set until its gate is met), `Distrusted` (out of the set, never trusted again), `Dropped` (out of the set after its drop time; still unexpired). Each entry lists both fingerprints (see "Compromise response").
- **No counts, no per-Connection lists** in the status: how many Connections pin which CA, and which aren't reached, are metrics (see "Signals"). The `Converged` message is written only when the condition changes, so the object doesn't churn on every poll.
- **Printer columns**: Current (the current CA's `keyFingerprint`), Converged, Age.

### Who writes what

- **`spec`**: PM Administrators, with `kubectl` or GitOps. `spec.distrust` accepts only `key-sha256:<64 hex>` (a schema pattern); there's no normalising of other notations, so a wrong format fails at apply time instead of matching nothing.
- **`status`**: `platform-mesh-operator`'s active replica only. It mirrors the Connector CA Set, computes each certificate's `state` and the distrust results, and the conditions ([03 Operator, "Connector CA Set"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).

**Convergence** counts only the Connections platform-mesh-operator reaches, against the current CA: each registered Connection is **pinned** (a Control Call that succeeded after the CA was issued confirmed it), **not reached** (an attempt after the CA was issued failed, whatever the cause) or **unknown** (not tried since the CA was issued, or not since `platform-mesh-operator` started). `Converged` is true when no Connection is unknown and none is reached but not pinning. A Connection that isn't reached catches up on its first successful Control Call; nothing waits for it. Unknown Connections do hold it back, so a restart or failover never declares convergence before every Connection has been polled.

## Connector Certificates

- **One cert-manager `Certificate` per shard**, issued by the CA's `ClusterIssuer`.
- **Lifetime 24 hours, renewed halfway** (`renewBefore: 12h`), `rotationPolicy: Always`. A stolen certificate is useful for at most a day, which is what lets the Connector CA do without revocation; cert-manager can be down for up to 12 hours before any certificate expires, and even then open Sessions keep running ([05 Connector](./xxx-notes-011-v5-impl-05-connector.md)). It's far shorter than any certificate elsewhere in PM on purpose.
- **Key usages**: digital signature and client authentication.
- **Minimal content**: key usages, and the shard name in the subject for audit. **No hostnames and no PM Instance identity**: served hosts travel in each Session's Hello ([01 Protocol, "Hosts a Session serves"](./xxx-notes-011-v5-impl-01-protocol.md)), and the binding to the PM Instance is the pinned CA. Nothing in a Connector Certificate changes when PM renames hosts.
- **Delivered to the Connector** as a mounted `Secret`, picked up on renewal without a restart ([09 Deployment, "Connector"](./xxx-notes-011-v5-impl-09-deployment.md)).
- **Labelled** with a common label, so the runbook can renew all of them at once (`cmctl renew` by label selector).
- **Switch rule** in the Connector: a renewed certificate is used once its CA has `state: Current` and the `Converged` condition is true, or when the current certificate has less than about an hour left, whichever comes first. A certificate whose CA has any other state than `Current` or `Previous` is never switched to. Until then the Connector keeps its current certificate, which stays valid for at least 12 hours after a renewal ([05 Connector, "Connector Certificate"](./xxx-notes-011-v5-impl-05-connector.md)).
- **All replicas of a shard's Connector share it.** Per-replica certificates are a later option with an external issuer.

## Rollover

Nobody takes part:
1. **cert-manager renews the CA** 90 days before it expires, with a new key.
2. **`platform-mesh-operator` sees the new CA certificate**, validates it, adds it to the Connector CA Set and pushes the set to every Connection, with compare-and-swap. Within minutes, every Connection it reaches pins both CAs.
3. **The new CA converges.** Connectors' certificates are renewed by cert-manager on their own schedule, now from the new CA; each Connector switches to its renewed certificate once the new CA has converged (see "Switch rule").
4. **After 48 hours, `platform-mesh-operator` drops the old CA**, Connection by Connection over the next 6 hours (see "The Connector CA Set"), and only once no Connector's current certificate chains to it. At each Provider, Sessions still open with an old-CA certificate are closed with go-away and a grace period, and the Connector reopens them at once with its current certificate ([01 Protocol, "Pinned Bundle"](./xxx-notes-011-v5-impl-01-protocol.md)). Long-running watches on those Sessions relist. Only Sessions not replaced in the meantime are affected (before the Edge's maximum connection age, after drops, on Connector rollouts), and the spread keeps the relists from hitting kcp at once.

What this means for Connections:
- **Healthy Connections** keep working throughout: they trust the new CA before any Connector uses it. Their only disturbance is the handover at the drop, for Sessions still on the old CA's certificate.
- **A Connection `platform-mesh-operator` can't reach** keeps its old pins. If its Provider is down, nothing changes until it returns; its first successful Control Call brings it the current set, whatever it missed, since Control Calls don't depend on the Connector CA. If its control path is broken while Sessions work, it loses each Session at the next replacement after the Connectors switch, until its control path works again.
- **A Connection that also can't authenticate** (a lost or rejected Connection Secret) stays without Sessions: repairing its Connection Secret needs a Session. The Provider Administrators re-pin the bundle out-of-band ([07 SDK, "Administration"](./xxx-notes-011-v5-impl-07-provider-sdk.md)).
- **If `platform-mesh-operator` is down for more than about 12 hours when the CA renews**, the new CA never converges, and Connectors switch only as their old certificates expire. Open Sessions keep running, but new Sessions fail at Providers that don't pin the new CA, until `platform-mesh-operator` is back and pushes it. That's the one way a `platform-mesh-operator` outage affects traffic.
- **A Connector restarted mid-Rollover** only has its newest certificate on disk and uses it at once. If the new CA hasn't converged yet, its Sessions to Connections not yet pinning it fail for the few minutes until the push lands.

## Compromise response

**Distrust is by key.** A CA is identified in `spec.distrust` by `key-sha256:<hex>`, the SHA-256 of its `SubjectPublicKeyInfo` (DER), so it covers every certificate ever issued for that key, including future ones; `cert-sha256:<hex>` (the SHA-256 of the certificate's DER) identifies a certificate everywhere else (`etag`, `GET`, the tamper check). If a certificate with a distrusted key shows up again, e.g. a renewal that reused the key, the `ConnectorCASet` reports `DistrustedKeyReused` with `Valid=False`, and that certificate isn't trusted. The runbook gives the commands that print the right value:

```sh
# from a CA certificate
openssl x509 -in ca.crt -noout -pubkey | openssl pkey -pubin -outform DER | sha256sum
# from its private key
openssl pkey -in ca.key -pubout -outform DER | sha256sum
```

prefixed with `key-sha256:`. The `ConnectorCASet`'s printer column shows the current CA's key fingerprint, and `status.certificates` lists both fingerprints for every CA.

**The CA's key** (or an issuance path that's compromised), in order:
1. **Renew the CA now** (`cmctl renew`). The new CA is pushed like any Rollover.
2. **Renew every Connector Certificate now**, by label (`cmctl renew -l <label>`), so Connectors get certificates from the new CA within minutes instead of up to 12 hours. Automating this in `platform-mesh-operator` is left for later; keeping it a separate, manual step also guards against a wrong CA listed in `distrust`.
3. **List the compromised CA's key in `spec.distrust`.** The CA becomes `DistrustPending` and stays in the set until its gate is met: the new CA has converged, every Connector's current certificate chains to a CA other than the distrusted one (`waitingFor` names the shards that don't yet). As soon as it is, `platform-mesh-operator` drops it from every Connection, spread over a short time (e.g. 15 minutes), instead of the 48 hours and 6 hours of a Rollover, and never trusts that key again: `Distrusted`.
4. **The Provider-side closes every Session opened under it** when its push arrives ([01 Protocol, "Pinned Bundle"](./xxx-notes-011-v5-impl-01-protocol.md)), and Connectors reopen them with their new certificates. Switching certificates resets a Connector's reconnect backoff, so this happens at once ([05 Connector, "Connector Certificate"](./xxx-notes-011-v5-impl-05-connector.md)).

With the gate, PM's own Connectors keep working throughout. **`immediate: true`** skips the gate, for when ending an attacker's Sessions matters more than PM's: the CA is dropped at once. Connectors still holding a certificate from it then can't open Sessions at Providers that received the push, until their certificates are renewed: minutes with step 2, up to 12 hours without.

**Guards**:
- **Never empty**: a distrust that would leave the set without a CA stays `DistrustPending`, with `Valid=False` and the alert; renew the CA first.
- **No match**: an entry that matches no CA PM knows gets the result `Unmatched`, the warning `DistrustUnmatched` on the `ConnectorCASet`, and an alert. A typo never silently does nothing.
- **Rollback**: removing an entry before it's applied cancels it. Afterwards, removing it makes the CA trusted again only if it's otherwise in the set.
- **Connections that weren't reached** keep the distrusted CA until their next successful Control Call; their `Provider`s show the warning `TrustOutdated` meanwhile, and the push that catches them up also changes their Connection Secret (`DistrustedCAPinned`, [03 Operator, "Connector CA Set"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).

**A shard's Connector Certificate key**: renew that shard's certificate (`cmctl renew`), and renew the CA now and distrust the old one as above, since Sessions opened with the stolen certificate outlive its expiry and only dropping its CA closes them. Stop issuing (delete or suspend its `Certificate`) only if the issuance path itself is compromised.

## Signals

For **PM Administrators** (Owners and Providers never see these). PM has no alerting pipeline today (the `observability` chart's Alertmanager is disabled and no chart ships alert rules), so each signal is a **metric**, plus a **condition or event** where one fits, plus a **documented alert rule** with a severity. Installations wire the rules into whatever alerting they run ([09 Deployment, "Observability"](./xxx-notes-011-v5-impl-09-deployment.md)).

| Signal | Metric (exposed by) | Condition or event | Documented rule |
|---|---|---|---|
| CA renewal failing | `providerconnections_connector_ca_expiry_seconds` (operator) | cert-manager's `Ready=False` on the CA `Certificate` | < 60 days left (renewal 30 days overdue): warning; < 30 days: critical |
| Invalid or empty CA set | `providerconnections_connector_ca_valid` (operator) | `Valid=False` on the `ConnectorCASet` | == 0: critical |
| CA not converging | `providerconnections_connector_ca_connections{state="pinned\|behind\|not_reached\|unknown", shard}`, with a `reason` label for not reached (operator) | `Converged=False` | Behind (reached but not pinning) or unknown > 0 for more than an hour: warning |
| Distrust waiting | `providerconnections_connector_ca_distrust_pending_seconds` (operator) | `state: DistrustPending`, with `waitingFor` | > 1 hour: warning (renew the Connector Certificates) |
| Distrust matching nothing | `providerconnections_connector_ca_distrust_unmatched` (operator) | `DistrustUnmatched` on the `ConnectorCASet` | > 0: critical |
| CA request history full | `providerconnections_connector_ca_history_exhausted` (operator): the oldest retained request's CA is unexpired and the limit is reached | — | == 1: warning (raise `revisionHistoryLimit`) |
| Connector holding its previous certificate | `providerconnections_connector_certificate_pending_switch_seconds{shard}` (Connector) | — | > 6 hours: warning (the new CA hasn't converged) |
| Connector Certificate not renewed | `providerconnections_connector_certificate_expiry_seconds{shard}` (Connector) | Event on the Connector's `Certificate` (cert-manager) | < 6 hours left (half the renewal budget used): warning |
| Connector Certificate expired | same | Connector readiness reports it (rollout tooling can hold) | ≤ 0: critical. From then on every Session that ends is an outage for its Connection on that shard ([05 Connector, "Opening a Session"](./xxx-notes-011-v5-impl-05-connector.md)) |
| A Pinned Bundle held an unexpired CA that isn't one of PM's | `providerconnections_connector_ca_bundle_tampered_total` (operator) | An event on the `Provider` ([03 Operator, "Connector CA Set"](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)) | Increase: critical (a Connection Secret was used by someone other than PM) |
| A Pinned Bundle still held a distrusted CA | `providerconnections_connector_ca_distrusted_pinned_total` (operator) | `DistrustedCAPinned` event on the `Provider` | — (expected after a distrust; it's handled) |

## Work breakdown

1. `platform-mesh-operator`: the `ConnectorCASet` CRD in `providers.platform-mesh.io`, with `spec.distrust` and its schema pattern, its status and printer columns, and creating the `default` object if it's missing.
2. Chart templates: the CA `Certificate` (with `revisionHistoryLimit: 10`) and its issuer (self-signed or `issuerRef`), the CA `ClusterIssuer`, per-shard Connector `Certificate`s with their common label ([09 Deployment](./xxx-notes-011-v5-impl-09-deployment.md)).
3. `platform-mesh-operator`: watching the CA's and the Connectors' `CertificateRequest`s; deriving the current CA, the Connector CA Set, per-CA drop times, the leaf gate, distrust states and unexpired CAs; the `ConnectorCASet` status ([03 Operator](./xxx-notes-011-v5-impl-03-platform-mesh-operator.md)).
4. Connector: the switch rule ([05 Connector](./xxx-notes-011-v5-impl-05-connector.md)).
5. Unit tests for the rules above, and chart-output tests for the templates.
6. Runbooks: what the `ConnectorCASet` status means, Connections not reached, compromise (renewing the CA and every Connector Certificate, getting a key fingerprint, `distrust`, `immediate`, rollback), an expired Connector Certificate (what ends open Sessions, what can't start), the CA's `CertificateRequest`s (don't delete them; raise the limit when its alert fires; with approver-policy, the policy must allow them), with the documented alert rules from "Signals".

## Decided

- **The CA is managed by cert-manager and rolled over automatically**: 1 year, renewed 90 days before expiry, new key each time; self-signed by default, `issuerRef` optional.
- **Everything is derived from the CA's `CertificateRequest`s**, the last ten of which cert-manager keeps, plus `spec.distrust` and the Connectors' current certificates; `platform-mesh-operator` stores nothing and never reads key `Secret`s.
- **The current CA** is the newest request that's issued and valid, by revision.
- **Each previous CA is dropped 48 hours after its successor is issued, spread over 6 hours per Connection**, and not while a Connector's certificate still chains to it. Connectors don't replace Sessions because of a Rollover; the drop's handover does.
- **Distrust is by key**, written by PM Administrators in `ConnectorCASet.spec.distrust`, and gated on the new CA and every Connector having switched, unless `immediate`. Renewing the Connector Certificates is a manual runbook step for now.
- **The tamper check only considers unexpired CAs**, and not distrusted ones.
- **Only Connections `platform-mesh-operator` reaches count for convergence**; nothing waits for the others, but unknown ones hold it back.
- **Counts are metrics**, not status.
- **An operator outage of more than about 12 hours at a CA renewal is accepted**: new Sessions fail until `platform-mesh-operator` pushes the new CA; open Sessions keep running. Letting `platform-mesh-operator` trigger CA renewals itself, only while it's healthy, would avoid it, but needs rights on cert-manager's `Certificate`s in cert-manager's namespace, which is cert-manager's scope, not `platform-mesh-operator`'s.
- **Connector Certificates carry no hostnames or PM Instance identity**; served hosts are in the Session's Hello.
- **No intermediates**: Connector Certificates are issued directly by the CA that Providers pin. The CA rolls over yearly by itself, so an intermediate would add nothing; with an external PKI, PM's CA is itself the intermediate, pinned as the trust anchor. **No name constraints** on the CA: there are no meaningful names in Connector Certificates to constrain.
- **No PM Instance identifier**: the CA of each PM Instance is the binding. A PM-wide identity for PM Instances would be a broader design.
- **Connector Certificates**: 24 hours, renewed halfway, no revocation. kcp doesn't revoke its certificates either, but relies on removing RBAC bindings, which Providers can't do for PM; a short lifetime is the equivalent.
- **Naming**: Connector CA, Connector Certificates, Connector CA Set, `ConnectorCASet` (in `providers.platform-mesh.io`, shipped by `platform-mesh-operator`).
