# Provider onboarding flow improvements

Continuation of ./006_provider-bootstrap-operator.md

Definitions:

- Provider: the Provider object on the kcp side, and the associated controller+portal deployments running in the service provider's runtime cluster.

Current state:

- A Provider and Platform-mesh instance have 1:1 mapping:
  - No way for a single Provider to support multiple PM instances.
  - Reason: the controller watches a single APIExportEndpointSlice.

- Provider onboarding flow currently involves a concious decision from both sides (PM & Provider) that the Provider is made available on the PM instance:
  - The service owner creates the Provider object, retrieves the kubeconfig from `Provider.status.providerKubeconfigSecretRef`, copies it over to their runtime cluster, creates a new deployment of their service provider so that it points to the kubeconfig+APIExportEndpointSlice and runs.

- The kubeconfig is backed by a static ServiceAccount token on the kcp side in `root:providers:<Provider.name>-<Cluster>`:
  - No support for token rotation, no support for other means of Authz.

- A Provider requires connectivity against the PM instance in order to reach the APIExportEndpointSlice:
  - The PM instance is assumed to be reachable on the same network as the Provider. In practice, this means public Providers (offering their services to _all_) would assume the PM instance is also open to the internet, otherwise it would be unreachable without the involvement of both parties.

## Proposal

The goal is to make Providers to be "onboarding-aware".

- The Provider side needs to hand out a Provider-owned one-time token that the PM instance uses to onboard this Provider into the instance.
  - Ensures that the requesting side is known.
  - Provider is able to use any Authz they want to hand out the token.
  - Follow-up: PM instance should sign the token, Provider should be able to verify that it is the correct instance of PM who ends up onboarding this Provider.

- Service providers need to be able to accept and serve requests to "become a PM provider" from external PM instances.

- Accepting the "become a PM provider" request means the Provider can reconcile consumers on the PM instance. No other manual steps should be needed.


#### Providers machinery + SDK

#### APIs

# Notes

PM side:
0. Owner goes and retrieves the token
    - Logs into Provider's infra
    - Copies the token
    - Token: PlatformMeshTokenV1:<Identifier>:<Alg>:<PublicKey>
    - Single use, expirable (set by Provider)
1. Create Provider with token already in spec
2. root:providers:# is created, SA created, kubeconfig copied back to root:orgs:<Org>:<User>
3. Make /onboard request:
    - schema:
        - envelope: {"kind": "PlatformMeshOnboardRequest", "version": "v1", identifier: "<Identifier from token>", payload: "<base64-encoded encrypted payload>"}
        - payload: {"kubeconfig": "<base64-encoded kubeconfig>": "apiexportendpointslice": "<APIExportEndpointSliceName>"}
4. Continuous reconcilliation on APIExportEndpointSlice by Provider
5. Use TokenRequest API to manage the token rotation
    - Provider uses Token API to manage this

Provider side:
1. /onboard handler receives the json data in POST
2. Decrypts with private key assigned to public key associated with <Envelope.Identifier>
3. Retrieve the kubeconfig
4. Establish the connection:
    - <Establish the HTTP tunnel>
    - Create REST config, create k8s client
    - Create Lease
    - On Success, ProviderConnection.status.phase=Connected

TODO:
- Idempotency
- Provider down, reconnect
- PM down, reconnect
- Provider down, can't reconnect because SA token has rotated
- Direct flow for locally available Providers
- How to evict Provider?
