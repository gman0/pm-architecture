---
title: Remote ServiceAccount credentials transport
---

# RFC 012: Remote cluster access

| Status  | Proposed   |
|---------|------------|
| Author  | TBD        |
| Created | 2026-11-03 |
| Updated | 2026-11-03 |

## Summary

This RFC defines APIs and flow for delivering kcp ServiceAccount kubeconfigs to an external runtime cluster, over a single PM-initiated handshake.

It delivers APIs for PM-side:
- `access.platform-mesh.io` API group, with `AccessGrant` kind + `access-operator` for that;
- `ProviderAccessRequest` kind in the existing `providers.platform-mesh.io` group.

As well as well APIs and components deployed on the external cluster (part of a public user-facing SDK that we also deliver):
- `access-sdk.platform-mesh.io` API group, with `AccessIdentity` and `Access` kinds;
- `platform-mesh-access` operator that also seconds as an HTTPS server.

It covers the "happy path" network scenario where both sides, PM and the external cluster, can reach each other on HTTPS: PM dials the user-side `/access` endpoint, delivers credentials, and PM's kcp is directly reachable from the external cluster (front-proxy, VWs).

## Motivation

Imagine that you are a user on a PM instance, with a ServiceAccount in your workspace, and want to transport a kubeconfig for that ServiceAccount into an external cluster. You, as a user, could be a consumer who just wants get resources in your workspace reconciled further; you could be a service provider, who wants to publish an APIExport onto the platform; or you could be a part of the monitoring team, and want to point your monit stack to a kubeconfig.

Today, you'd have to:
1. retrieve the kubeconfig,
2. move it to *your* runtime cluster as a Secret,
3. deploy *your* service operator such that it points to that kubeconfig,
4. keep the SA token up to date.

If you are the same person as the one who *owns* that external cluster, then fair enough, this is somewhat feasible. But for third party personas, this is too big of a gap to overcome. For example, they may be a publicly available service provider who wishes to reconcile many PM instances. First, where do they stash all these kubeconfigs? Does each provider need to also deploy a vault instance just so that PM users can drop off their kubeconfig? Secondly, external access between this external cluster and the PM instances carries one important technical detail, namely their respective network visibility:

| PM reachable from User | User endpoint reachable from PM | Notes |
|---|---|---|---|
| Yes | Yes | This RFC |
| No | Yes | Follow-up to this RFC (needs tunnelling) |
| Yes | No | — | PM cannot reach user; User-initiated credential delivery is a follow-up |
| No | No | — | We assume the sides must have created a tunnel out-of-band; then it's the same as the first variant (both sides are reachable) |

If we want hands-free credential delivery, one side needs to be able to reach the other. The flow of how the credentials are delivered then depends on who can see whom first. This RFC defines the first variant: **both sides see each other.**

**Goals:**

- Programmatic kubeconfig delivery to an external cluster, PM-initiated, no manual Secret transport.
- As a consequence, support third-party Providers serving many PM instances.

**Non-goals:**

- Introduce dependencies on identity providers of either of the sides.

## Terminology

| Term | Meaning |
|---|---|
| **User** | A person who wants to transport credentials for their ServiceAccounts. The same user may have many, even in different PM instances. |
| **External Cluster** | The destination cluster where these credentials are to be transferred to. |
| **Access Identity** | The **User** is known by this **Identity** on this **External Cluster**. This RFC doesn't have an opinion about how this identity came to be. We only provide an API to represent it: the `AccessIdentity` kind. |
| **Access Grant** | "I can prove that I'm the one with this **Access Identity**, and I want to *push* credentials for this SA to that **External Cluster**. |

## Architecture

The


The general idea is that in order to transfer a kubeconfig anywhere, the Owner must have had an established "identity" with the remote cluster where they want their kubeconfig to appear. That identity should be then all they need in order to make the transfer safely over TLS.

Because the

The design is spread across PM and user's cluster, where the user's cluster generates an **access token**. PM-side uses that to

In order to do that, we need to know where to send the kubeconfig, and under what identity. This information is all part of the **access token** that

### APIs

This RFC brings in APIs related to remote access: the PM-side APIs . Additionally, it also integrates with existing `providers.platform-mesh.io` APIs. These are largely just wrappers to help surfacing access-related state to the `Provider` object.

#### `AccessIdentity` (user-side)

#### `AccessGrant` (PM-side)

#### `Access` (user-side)

#### `ProviderAccessRequest` (PM-side)

#### `Provider` (PM-side)

### Flow

## Alternatives considered

## Future work
