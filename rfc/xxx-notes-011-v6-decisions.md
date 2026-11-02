# RFC 011 v6: decisions

Private working notes. State as of 2026-10-31. **This file is superseded by
[`xxx-notes-remote-access-overview.md`](./xxx-notes-remote-access-overview.md)**
which reframes the whole area from first principles. What is here is kept for
historical reference; the new overview document is the working baseline.

---

v6 halves v5: onboarding and kubeconfig delivery are decoupled from tunnelling.
The aim is UX: onboarding a Provider should be smooth. Real kcp credentials (an
SA token) reach the Provider-side.

## Cast and terms

From RFC 011, unchanged:

| Term | Who or what |
|---|---|
| **Owner** | A person (or their GitOps) controlling a workspace in PM, e.g. `:root:orgs:acme:alice`. The Provider's customer: wants the Provider's service available in PM. Creates the `Provider` object; admin of the Provider Workspace. |
| **Provider** | The service provider (e.g. Wild West Inc.). A different persona from the Owner. Runs its own cluster. |
| **Provider Workspace** | The kcp workspace PM creates for a `Provider`, where the Provider publishes its APIExports. |
| **The Provider's operator** | The Provider's own controllers, reconciling its service through the APIExport VWs. |
| **The Provider's portal** | The Provider's website for its customers. |

New in v6:

| Term | Side | What it is |
|---|---|---|
| **`platform-mesh-join`** (working name) | PM | New component. Reconciles `JoinToken`s and serves the anonymous redeem endpoint. |
| **`platform-mesh-access`** | Provider | Deployment, no CRD. Redeems join tokens, writes kubeconfig Secrets, renews SA tokens, detects offboarding. |
| **`platform-mesh-tunnel`** | Provider | Follow-up RFC. L4 CONNECT, a "direction inverter" for when only the Provider-side is reachable. |
| **`JoinToken`** (JT) | PM | Created by the Owner in the Provider Workspace; references an SA. |
| **Join token** | both | `pmj1.<base64url(redeem base URL)>.<tokenID>.<secret>`. Single use. `K_mac = HKDF(secret, "info")`, `K_auth = HKDF(secret, "auth")`. |

`provider-gateway` has no role in v6: its jobs split into `platform-mesh-access`
(kubeconfig) and `platform-mesh-tunnel` (follow-up).

## Principles

- **Exactly one side must be reachable**, natively or through its own
  single-sided tunnel. v6 covers "PM reachable"; `platform-mesh-tunnel` covers
  "Provider-side reachable". Two-sided overlays need the other side to match:
  out of scope.
- **Every Provider-side component serves many PM instances**
  (`platform-mesh-access`, `platform-mesh-tunnel`).
- **No identity provider is assumed** on either side: kcp SAs only. OIDC is an
  optional follow-up.
- **The Provider is a regular user** who happens to reconcile exports: a
  standard admin SA in its Provider Workspace. No lockdown VAPs, no Denylist,
  no scoped SA.
- **The Provider pulls.** No PM → Provider calls.

## Flow

```
            PM side                                      Provider side (Provider's cluster)
 ┌───────────────────────────────────┐
 │ Provider Workspace                │
 │   JoinToken ──(1)──► ┐     │
 │   ServiceAccount             │    │
 │   Secret <jt>-token ◄──┘    │      Owner ──(2) hands the token over──► Provider
 └───────────────────────────────────┘                                             │
                                                                                   ▼
 ┌───────────────────────────────────┐                                  Secret pm-eu-orders
 │ ingress /platform-mesh/join       │                                    joinToken
 │   └► platform-mesh-join           │◄──(3) GET info ────────────────  platform-mesh-access
 │                                   │─────► URLs, CA bundle, HMAC ──►        │
 │                                   │◄──(4) POST redeem, K_auth ──────       │
 │                                   │─────► SA token ───────────────►        │ (5) writes into
 └───────────────────────────────────┘                                          ▼     the same Secret
 ┌───────────────────────────────────┐                                  Secret pm-eu-orders
 │ front-proxy                       │◄──(6) TokenRequest, SA token ───   kubeconfig, token
 │                                   │─────► new SA token ───────────►        │ (7) mount
 │ shards' VWs                       │◄──(8) list/watch, SA token ─────  Provider's operator
 └───────────────────────────────────┘
```

1. The Owner creates the `Provider` (`Ready` = the Provider Workspace is
   ready), then a JoinToken in the Provider Workspace referencing an SA of their
   choice. `platform-mesh-join` issues the join token into a delivery Secret.
2. The Owner reads the token and hands it to the Provider (e.g. pastes it into
   the Provider's portal, or sends it to the Provider's admin). This is the
   only manual, cross-organisation step.
3. The Provider puts it into a Secret in its cluster; `platform-mesh-access`
   calls `GET info` anonymously and verifies the HMAC with `K_mac`.
4. `POST redeem` with `Bearer K_auth`, over TLS verified against the bundle
   from `info`. The response is an SA token.
5. `platform-mesh-access` writes the kubeconfig and token into the same Secret.
6. Self-rotation via TokenRequest, hourly.
7. The Provider's operator mounts the Secret.
8. The operator reaches the front-proxy and every shard's VW with the SA token.

## Decided

### Credentials and revocation

- **Token:** a real token for the referenced SA, an admin in the Provider
  Workspace.
- **Rotation:** `platform-mesh-access` self-renews via TokenRequest with kcp's
  default TTL (1 h; `expirationSeconds` unset). A flag can come later.
- **Offboarding = deleting the SA.** The RFC states that tokens minted through
  consumers' `serviceaccounts/token` claims survive this. Capping them
  (`--service-account-max-token-expiration`) goes in the docs only.

### Reachability and TLS

- **Reachable URLs:** the front-proxy and every shard's VW URL must be
  reachable from the Provider-side.
- **CA bundle:** an explicit bundle in the kubeconfig from day one, built like
  `buildAdminAuthTrustBundle` or from kcp-operator's
  `<rootShard>-merged-ca-bundle`, plus extra PEMs. Empty means web PKI.
- **Rotation of the CA bundle and of URLs:** a single follow-up. PM publishes
  the current values in an object readable by the SA (e.g. filled from a
  ClusterCachedResource), and `platform-mesh-access` rewrites the kubeconfig.
  Old and new must overlap. Design it now so it doesn't block us.
- **No Inner TLS.**

### Redeem flow (B1)

- **`info`:** anonymous `GET info` returns the front-proxy URL, the workspace,
  the CA bundle, and an HMAC with `K_mac`.
- **`redeem`:** `POST redeem` with `Bearer K_auth` over TLS verified against
  the bundle.

### Items from the review of v5's other principles

1. **tokenFile.** The kubeconfig uses `tokenFile: token` (a relative path),
   with a `token` key in the same Secret. It's mounted without `subPath`;
   client-go re-reads token files about once a minute.
2. **Liveness:** a follow-up. A Lease with a special label; Leases are
   aggregated onto the `Provider`.
3. **Offboarding detection in `platform-mesh-access`:**
   - **Expired:** a 401 after the JWT's `exp`.
   - **Revoked:** persistent 401s before `exp`, for about 2 minutes or more,
     including a failed renewal.
   - **Never counted:** 403, 5xx and network errors.
   - **Then:** it deletes the Secret, reports through a Pod Event, a log line
     and not-ready, and waits for a new join token.
   - Being offline longer than the TTL therefore means re-onboarding.
4. **Token delivery and state:**
   - **Delivery Secret:** `<jt>-token`, holding the token and `K_mac`,
     owned by the JoinToken and referenced from `status.secretRef`.
   - **State on the JoinToken:** `status.tokenID`, `status.tokenHash`
     (`H(K_auth)`), `status.redeemed` and the expiry. There's no separate
     record: `platform-mesh-join`'s VW informer indexes JoinTokens by `tokenID`.
   - **Single use:** the status write uses the `resourceVersion`, before
     minting.
   - **`K_mac` never goes in status.**
   - **Status writes:** a VAP allows them only from `platform-mesh-join`.
   - **After redeem or expiry,** the delivery Secret is deleted.
   - **Join token lifetime:** fixed, no spec field. The value is open (below).
5. **Redeem endpoint placement:**
   - **Where:** inside `platform-mesh-join`, a separate component. Not in
     `platform-mesh-operator`, which holds the kcp admin kubeconfig and
     installs PM.
   - **Privileges:** its APIExport VW only, plus claims on `secrets` and
     `serviceaccounts/token`.
   - **Replicas:** all of them serve redeem.
   - **Hardening:** limits, rate limits per IP and `tokenID`, constant-time
     comparison, identical 404 for unknown, spent and expired IDs.
   - **Routing:** an ingress path route on the front-proxy hostname, in front
     of the front-proxy. It can't be a front-proxy mapping (verified, see
     below).
   - **`platform-mesh-operator`:** only deploys `platform-mesh-join`, and sets
     up the export and the WorkspaceType default binding.
6. **SSRF checks:**
   - **None:** no denylist and no host pin. Whoever knows `K_mac` has the
     whole token anyway.
   - **Trust:** `platform-mesh-access` trusts `info` once the HMAC checks.
   - **Egress settings** (proxy, `additionalTrustedCAs`) stay.
   - **The redeem base URL travels inside the join token,** because there can
     be many PM instances.
7. **Join identity on the Provider-side:** one Secret per join, created by the
   Provider by hand (or by its portal's automation), never through GitOps or
   Helm.
   - **Labels and keys:** label `access.platform-mesh.io/join: "true"`, key
     `joinToken`.
   - **After redeem:** `platform-mesh-access` writes `kubeconfig` and `token`
     into the same Secret, removes `joinToken`, and adds annotations: redeem
     URL, workspace, SA, `token-expires-at`.
   - **Re-onboarding:** put `joinToken` back in.
   - **Local removal:** delete the Secret.

### Escalation check on the JoinToken

- **Rule:** the requester must be able to `create serviceaccounts/token` on the
  referenced SA. This is enforced by an export-side VAP with
  `authorizer…check('create')`, evaluated in the Provider Workspace.
- **Immutability:** `spec.serviceAccountRef` is immutable (same VAP).
- **kcp version:** this needs the kcp fix below. Without it, the fallback is a
  MutatingAdmissionPolicy that stamps `request.userInfo` plus a
  SubjectAccessReview by `platform-mesh-join`. Only needed if unfixed kcp
  versions must be supported.

## Verified by e2e (kcp, uncommitted, all passed)

| Test | Finding |
|---|---|
| `test/e2e/virtual/apiexport/serviceaccount_revocation_test.go` `TestServiceAccountDeletionRevokesTokensAtVW` | Deleting the SA gets its token refused after about 10 s at the base endpoint and about 65 s at a VW on another shard (shardlookup cache, 1 min TTL). An open watch keeps delivering events. A recreated SA doesn't revive old tokens. The default TTL is 1 h. |
| same file, `TestTokenMintedThroughClaimOutlivesProviderRevocation` | A consumer SA token minted through a claim gets the full 48 h requested, and survives deleting both the provider SA and the APIBinding. Only deleting the consumer SA revokes it. |
| `test/e2e/conformance/validatingadmissionpolicy_authorizer_test.go` | The export-side VAP applies to bound resources; consumer-local policies don't. With the fix, the CEL authorizer evaluates in the consumer workspace, and rights held only in the export workspace don't count. The consumer admin can't see or delete the policy. |
| `test/e2e/proxy/anonymous_mapping_test.go` | Without additional auth on the front-proxy, mapped paths pass everything through and forward `Authorization` as is. With `--service-account-key-file` (PM's default through kcp-operator), every request without valid kcp credentials gets a 401 before reaching the backend. |

**kcp fix (uncommitted; unit and e2e tests passed):**

- **The bug:** the CEL authorizer called kcp's authorizer with
  `context.TODO()` and no cluster, so it denied everyone.
- **The fix:** `pkg/authorization/cluster_scoped_authorizer.go`
  (`NewClusterScopedAuthorizer`) wraps the authorizer in the VAP and MAP
  plugins, pinned to the target workspace.
- **Still to check:** webhook `matchConditions` are probably still affected.
- **Side note:** `messageExpression` is evaluated without an authorizer
  (upstream).

## Open (from v6; see overview doc for current status)

1. **Join token lifetime:** 1 h, or longer (e.g. 24 h) because of the
   cross-organisation hand-over from the Owner to the Provider?
2. **Remaining review items:**
   - 8: reachability for all VW types, e.g. the replication VW.
   - 9: adding a shard needs exposure, DNS and bundle coverage.
   - 10: fairness moves to kcp APF (unverified).
   - 11: audit moves to kcp's audit log.
   - 12: protocol versioning: `pmj1.`, `/v1/`, current plus previous.
   - 13: no separate Remote Provider; `spec.remote` isn't needed.
3. **Second redeem attempt:** should a second attempt also revoke the first
   redeem (theft detection), or only fail?
4. **A generic join primitive**, beyond Providers.
5. **Provider Workspace per Owner vs M:N** (RFC 006).
6. **Security claims:** the wording replacing v5's "no kcp credential ever
   crosses", and whether to add a "what v6 gives up" table.
7. **Document:** rewrite RFC 011 in place, or a new RFC superseding it? Is v5
   shelved, or kept as a follow-up for a no-credentials mode?
