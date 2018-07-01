---
title: Authentication Policy
description: Describes Istio's authentication policy
weight: 10
keywords: [security,authentication]
aliases:
    - /docs/concepts/network-and-auth/auth.html
---

Istio authentication policy enables operators to specify authentication requirements for a service (or services). Istio authentication policy is composed of two parts:

* Peer: verifies the party, the direct client, that makes the connection. The common authentication mechanism for this is [mutual TLS](/docs/concepts/security/mutual-tls/).

* Origin: verifies the party, the original client, that makes the request (e.g end-users, devices etc). JWT is the only supported mechanism for origin authentication at the moment.

Istio configures the server side to perform authentication, however, it does not enforce the policy on the client side. For mutual TLS authentication, users can use [destination rules](/docs/concepts/traffic-management/rules-configuration/#destination-rules) to configure client side to follow the expected protocol. For other cases, the application is responsible to acquire and attach the credential (e.g JWT) to the request.

Identities from both authentication parts, if applicable, are output to the next layer (e.g authorization, Mixer). To simplify the authorization rules, the policy can also specify which identity (peer or origin) should be used as 'the principal'. By default, it is set to the peer's identity.

## Architecture

Authentication policies are saved in Istio config store (in 0.7, the storage implementation uses Kubernetes CRD), and distributed by control plane. Depending on the size of the mesh, config propagation may take a few seconds to a few minutes. During the transition, you can expect traffic lost or inconsistent authentication results.

{{< image width="80%" ratio="75%"
    link="./authn.svg"
    caption="Authentication Policy Architecture"
    >}}

Policy is scoped to namespaces, with (optional) target selector rules to narrow down the set of services (within the same namespace as the policy) on which the policy should be applied. This aligns with the ACL model based on Kubernetes RBAC. More specifically, only the admin of the namespace can set policies for services in that namespace.

Authentication is implemented by the Istio sidecars. For example, with an Envoy sidecar, it is a combination of SSL setting and HTTP filters. If authentication fails, requests will be rejected (either with SSL handshake error code, or http 401, depending on the type of authentication mechanism). If authentication succeeds, the following authenticated attributes will be generated:

* **source.principal**: peer principal. If peer authentication is not used, the attribute is not set.
* **request.auth.principal**: depends on the policy principal binding, this could be peer principal (if USE_PEER) or origin principal (if USE_ORIGIN).
* **request.auth.audiences**: reflect the audience (`aud`) claim within the origin JWT (JWT that is used for origin authentication)
* **request.auth.presenter**: similarly, reflect the authorized presenter (`azp`) claim of the origin JWT.
* **request.auth.claims**: all raw string claims from origin-JWT.

Origin principal (principal from origin authentication) is not explicitly output. In general, it can always be reconstructed by joining (`iss`) and subject (`sub`) claims with a "/" separator (for example, if `iss` and `sub` claims are "*googleapis.com*" and "*123456*" respectively, then origin principal is "*googleapis.com/123456*"). On the other hand, if principal binding is USE_ORIGIN, **request.auth.principal** carries the same value as origin principal.

## Anatomy of a policy

### Target selectors

Defines rules to find service(s) on which policy should be applied. If no rule is provided, the policy is matched to all services in the same namespace of the policy, so-called namespace-level policy (as opposed to service-level policies which have non-empty selector rules). Istio uses the service-level policy if available, otherwise it falls back to namespace-level policy. If neither is defined, it uses the default policy based on service mesh config and/or service annotation, which can only set mutual TLS setting (these are mechanisms before Istio 0.7 to configure mutual TLS for Istio service mesh). See [testing Istio mutual TLS](/docs/tasks/security/mutual-tls/).

> Starting with 0.8, authentication policy is the recommended way to enable/disable mutual TLS per service. The option to use service annotation will be removed in a future release.

Operators are responsible for avoiding conflicts, e.g create more than one service-level policy that matches to the same service(s) (or more than one namespace-level policy on the same namespace).

Example: rule to select product-page service (on any port), and reviews:9000.

{{< text yaml >}}
targets:
- name: product-page
- name: reviews
  ports:
  - number: 9000
{{< /text >}}

### Peer authentication

Defines authentication methods (and associated parameters) that are supported for peer authentication. It can list more than one method; only one of them needs to be satisfied for the authentication to pass. However, starting with the 0.7 release, only mutual TLS is supported. Omit this if peer authentication is not needed.

Example of peer authentication using mutual TLS:

{{< text yaml >}}
peers:
- mtls:
{{< /text >}}

> Starting with Istio 0.7, the `mtls` settings doesn't require any parameters (hence `-mtls: {}`, `- mtls:` or `- mtls: null` declaration is sufficient). In future, it may carry arguments to provide different mutual TLS implementations.

### Origin authentication

Defines authentication methods (and associated parameters) that are supported for origin authentication. Only JWT is supported for this, however, the policy can list multiple JWTs by different issuers. Similar to peer authentication, only one of the listed methods needs to be satisfied for the authentication to pass.

{{< text yaml >}}
origins:
- jwt:
    issuer: "https://accounts.google.com"
    jwksUri: "https://www.googleapis.com/oauth2/v3/certs"
{{< /text >}}

### Principal binding

Defines what is the principal from the authentication. By default, this will be the peer's principal (and if peer authentication is not applied, it will be left unset). Policy writers can choose to overwrite it with USE_ORIGIN. In future, we will also support *conditional-binding* (e.g USE_PEER when peer is X, otherwise USE_ORIGIN)