# RFC 011 v5 notes: trust asymmetry

Who carries which risk when something goes wrong, for the RFC's principles. Each side is weaker in a different way: the **Provider-side on trust** (it can't verify its counterpart and holds few levers), the **PM-side on blast radius** (a compromised Provider reaches consumer workspaces), but the PM-side holds the levers to contain that.

## The Provider-side is weaker on trust

- **It can't authenticate the PM Instance.** A non-goal: PM Instances can be private, and there's no registry to check them against. Anyone the Provider's portal gives an Onboarding Token to can register any PM Instance, a fake one included; from then on the Provider-side trusts that Connection's Connector CA.
- **It acts on what PM tells it.** Consumer objects arrive from kcp over Sessions, and the Provider's operator acts on them: provisions resources, spends money, writes credentials back. A malicious or compromised PM Instance can feed fake consumers and collect what the operator writes (e.g. DB passwords for a DBaaS).
- **The PM-side holds nearly every lever.** It opens every connection, decides when Secret Changes and Connector CA rollovers happen, pushes Connector CA Sets, and can `DELETE` the Connection. The Provider-side's levers: refuse or stop issuing tokens, retire Server Keys, delete a `Connection`, or stop `provider-gateway`.

**What bounds it:** the PM-side can't initiate anything in the Provider's cluster. Every request on a Session comes from the Provider's operator; the PM-side only answers. The worst case is a misled operator, not a compromised cluster: the same risk as any SaaS vendor serving requests from its customers. The Provider's real controls are its portal (who gets tokens) and its own quotas per Owner account.

## The PM-side and its consumers carry the bigger blast radius

- **A compromised Provider runtime acts as the Provider in kcp:** it can read and write the bound APIs and whatever its permission claims grant in consumer workspaces, `Secret`s included. Consumers are exposed to a security posture they didn't choose; their Owner chose it for them.
- **What bounds it:**
  - the Provider's ServiceAccount RBAC;
  - the admission policies in the Provider Workspace;
  - the Connector Denylist (no minting SA tokens);
  - no kcp credential ever crosses, so a compromise leaves nothing reusable behind;
  - revocation is immediate: once the Connectors stop proxying, the Provider-side has nothing left to use.

## Concentration on both sides

| Compromised | Holds | Affects |
|---|---|---|
| One PM Instance (its runtime cluster) | the Connector CA and every Connection Secret | every Provider connected to it: a fake kcp toward all of them, and `DELETE` on all their Connections |
| One Provider (`provider-gateway`) | Server Keys, shared by all its Connections | consumers on every PM Instance it serves |

## Recovery is asymmetric

- **The Provider-side loses state:** the PM-side notices (`SecretRejected`, or an unknown Connection) and repairs the Connection Secret, or replaces the Connection by Onboarding again, which needs a new token from the Owner.
- **The PM-side loses the `ProviderEndpointRegistration`'s `Secret`:** the Connection Secret is gone and the Provider-side holds only its hash, so it can't help. Left to RFC 012 (recovery).

The PM-side is the more resilient side and also the bigger single point of failure.

## Suggested RFC wording

A short "Trust asymmetry" paragraph next to the principles, since reviewers will ask:

> The two sides don't carry the same risk. The Provider-side can't verify the PM Instance it serves and acts on what that instance sends, but nothing the PM-side does can reach into the Provider's cluster: it trusts PM about as much as any SaaS vendor trusts its customers. The PM-side, which holds its consumers' data, carries the bigger blast radius if a Provider is compromised, so it keeps every lever: kcp credentials never leave it, the Provider's access is bounded by policy, and revocation is immediate.

See also [the Design foreword notes](./xxx-notes-011-v5-design-foreword.md), section 4 (Trust).
