# v5 implementation plan: session CA

Part of the [v5 implementation plan](./xxx-notes-011-v5-impl-00-overview.md). The PKI that authenticates [connectors](./xxx-notes-011-v5-impl-05-connector.md) to Providers. The design rationale is in v5, "Session credentials: the session CA" and "Pinning immutable CA generations"; this document plans the implementation with a **cert-manager CA** first and a Vault- or KMS-backed issuer later.

## What it provides

- **Shard certificates**: one per shard's connector, short-lived, used to sign session assertions and as the inner TLS client certificate ([01 Protocol, "Sessions"](./xxx-notes-011-v5-impl-01-protocol.md)).
- **CA certificates**, per generation, that the operator sends to Providers at registration and rollover ([03 Operator](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)).
- **Generation states**, so the operator knows what to pin and unpin.

## Principles

- **Providers pin CA certificates, and generations are immutable.** A generation is never renewed with a new key; changing anything means a new generation added with overlap.
- **The pinned certificate's expiry is a hard deadline.** A generation that wasn't rolled over stops working everywhere; alerts must fire long before.
- **Shard certificates are nobody's pin**: short-lived, new key on every renewal.
- **The issuer is replaceable.** Nothing in the protocol, the operator or the connector depends on cert-manager specifically; a KMS-backed issuer can replace it.

## Generations

- **One cert-manager `Certificate` per generation**, self-signed, CA, long-lived (e.g. 5 years), with `rotationPolicy: Never`. Its renewal point (by default two-thirds of the lifetime) must lie after the generation's planned retirement, so it never renews in practice; if it does, the key stays the same and the old pinned certificate keeps verifying until it expires.
- **One CA `Issuer` per generation**, backed by that generation's Secret.
- **Naming** encodes the generation (e.g. `session-ca-g1`, `-g2`), never reused.
- **Key storage**: the generation's Secret in a namespace only cert-manager and the issuer can read. Not readable by the operator or connectors.
- **Constraints**: basic constraints CA with path length 1 (room for a future intermediate), key usage certificate signing, and optionally name constraints on the URI SAN namespace used by shard certificates.

## Publishing generations

A **PM-owned declaration of generations and their states** in the runtime cluster (e.g. a ConfigMap, or a small CRD if it grows), listing for each generation its public certificate and state:

| State | Meaning |
|---|---|
| `next` | Being pinned on every connection; not yet issuing |
| `active` | Issues shard certificates; pinned everywhere |
| `retiring` | Still pinned; no longer issues; waiting for its shard certificates to expire |
| `retired` | Unpinned everywhere |

- **The operator reads it**: registration sends `active` plus `next`; rollover fans out changes ([03 Operator, "Session CA rollover"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)).
- **Exactly one generation is `active`.**
- **Changed by humans or automation through GitOps**, not by the operator; the operator only reflects it to Providers.

## Shard certificates

- **One `Certificate` per shard**, issued by the `active` generation's issuer, for that shard's connector.
- **Lifetime** short (e.g. 24 hours), renewed well before expiry, `rotationPolicy: Always`.
- **Key usages**: digital signature and client authentication.
- **Identity**: a URI SAN naming the shard and the virtual-workspace host its sessions serve. The Provider-side reads the served host from it ([01 Protocol, "Hosts a session serves"](./xxx-notes-011-v5-impl-01-protocol.md)).
- **Delivered to the connector** as a mounted Secret, picked up on renewal without a restart ([09 Deployment, "Connector"](./xxx-notes-011-v5-impl-09-deployment.md)).
- **All replicas of a shard's connector share it.** Per-replica certificates are a later option with a KMS-backed issuer (v5, "Connector replicas").

## Rollover

1. **Create the next generation** (`Certificate` and `Issuer`) and mark it `next`.
2. **The operator pins it** on every connection, alongside `active`, and reports convergence.
3. **When every reachable connection pins it**, mark it `active` and the old one `retiring`; switch shard `Certificate`s to the new issuer; cert-manager reissues them.
4. **After the last old shard certificate expires**, the operator unpins the old generation everywhere; mark it `retired`.
5. **Delete the retired generation's key.**

Planned yearly (for example), never urgent unless the CA key is in doubt. Connections that can't be reached during a rollover are reported; if they stay unreachable past the old generation's expiry, they need a rebind or a new connection.

## Compromise response

- **A shard**: stop issuing its certificate (delete or suspend its `Certificate`); its current certificate expires within hours.
- **A generation's key**: roll over immediately and unpin the compromised generation as soon as the next is pinned.

## Alerts

- A generation's remaining lifetime below a threshold (e.g. 6 months) while it's `active` or `retiring`.
- A shard certificate not renewed in time.
- Rollover convergence stalled.

## Later: KMS- or Vault-backed issuer

- **The generation's key never leaves the KMS**; cert-manager (or a dedicated issuer) asks it to sign shard certificates.
- **Optionally per-replica certificates**, each replica generating a key in memory at start.
- **No protocol change**: Providers still pin generation certificates.

## Work breakdown

1. Chart templates for generations, issuers and per-shard certificates ([09 Deployment](./xxx-notes-011-v5-impl-09-deployment.md)).
2. The generation declaration and its schema.
3. Runbooks: first setup, rollover, compromise.
4. Alerts.

## Open questions

To be revisited separately, together:

- **Declaration format**: a ConfigMap is enough for one list; a CRD gives validation and status. With the operator in PM's cluster, the declaration lives there too; it only moves into kcp if operator replicas ever span clusters ([03 Operator, "Partitioning by shard"](./xxx-notes-011-v5-impl-03-providerconnections-operator.md)).
- **Lifetimes**: generation lifetime and rollover interval, shard certificate lifetime and renewal margin.
- **URI SAN scheme** for shard certificates, and whether to add name constraints to generations.
- **Where the generation Secrets live**, and who can read them in PM's runtime cluster.
