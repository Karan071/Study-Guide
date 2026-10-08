# Authentication, Authorization & Modern Security Protocols: A Backend Engineer's Reference

Audience: SDEs designing, building, and operating production backends (monolith or microservices).
Scope: 20 core concepts, each broken into **Core Mechanism**, **Engineering Blueprint**, **Production Pitfalls**, **SDE Best Practices**, and **Resources**.
Last reviewed: 2026-10-08. Standards move (OAuth 2.1 and the browser-based-apps BCP are still IETF drafts at the time of writing); verify against the linked specs before locking a design.

---

## Table of Contents

0. [System Map: How the 20 Concepts Fit Together](#0-system-map-how-the-20-concepts-fit-together)
1. [Authentication](#1-authentication)
2. [Authorization](#2-authorization)
3. [Identity Provider (IdP)](#3-identity-provider-idp)
4. [OAuth 2.0](#4-oauth-20)
5. [OpenID Connect (OIDC)](#5-openid-connect-oidc)
6. [Authorization Code Flow](#6-authorization-code-flow)
7. [PKCE (Proof Key for Code Exchange)](#7-pkce-proof-key-for-code-exchange)
8. [State Parameter](#8-state-parameter)
9. [Nonce Parameter](#9-nonce-parameter)
10. [Access Token](#10-access-token)
11. [Refresh Token](#11-refresh-token)
12. [ID Token](#12-id-token)
13. [JSON Web Token (JWT)](#13-json-web-token-jwt)
14. [JWKS (JSON Web Key Set)](#14-jwks-json-web-key-set)
15. [RS256](#15-rs256)
16. [Cookies / HttpOnly / SameSite](#16-cookies--httponly--samesite)
17. [Sessions](#17-sessions)
18. [RBAC (Role-Based Access Control)](#18-rbac-role-based-access-control)
19. [Casbin](#19-casbin)
20. [Tenant Isolation](#20-tenant-isolation)
21. [Production Checklist & Cheat Sheet](#21-production-checklist--cheat-sheet)
22. [Master Resource List](#22-master-resource-list)

---

## 0. System Map: How the 20 Concepts Fit Together

Three questions, three layers. Most auth bugs come from mixing them up.

| Question | Layer | Concepts |
| --- | --- | --- |
| Who are you? | Authentication | 1, 3, 5, 6, 7, 8, 9, 12 |
| What may this client do on your behalf? | Delegated authorization (OAuth) | 4, 6, 7, 10, 11 |
| What may this user do to this object? | Application authorization | 2, 18, 19, 20 |
| How do we carry the answer between hops? | Transport & state | 13, 14, 15, 16, 17 |

### Actors and trust boundaries

```text
                 untrusted                         |            trusted (your infra)
                                                   |
  +-----------+      +----------------+            |   +-----------------+     +----------------+
  |  Browser  |<---->|  Web client /  |<-----------+-->|  API gateway    |---->|  Service A     |
  |  (user)   |      |  BFF / SPA     |            |   |  (JWT verify,   |     |  (authz: RBAC/ |
  +-----+-----+      +--------+-------+            |   |   coarse authz) |     |   Casbin, RLS) |
        |                     |                    |   +--------+--------+     +-------+--------+
        |  redirect           | back-channel       |            |                      |
        v                     v  (token exchange)  |            | JWKS fetch           | SQL + tenant ctx
  +---------------------------------------+        |            v                      v
  |  Identity Provider (IdP / OP / AS)    |<-------+--- /.well-known/jwks.json     +-----------+
  |  login, MFA, consent, token issuance  |        |                               | Postgres  |
  +---------------------------------------+        |                               | (RLS)     |
```

### End-to-end login, with each concept placed

```text
Browser            Client backend (BFF)            IdP (OIDC OP)               API / Resource Server
   |  GET /login          |                               |                              |
   |--------------------->| generate state, nonce,        |                              |
   |                      | code_verifier -> code_challenge (PKCE)                       |
   |                      | store in short-lived signed cookie / server session [16,17]  |
   |<-- 302 /authorize?response_type=code&scope=openid...&state&nonce&code_challenge     |
   |-------------------------------------------------------->| authenticate user [1]      |
   |                      |                               | MFA, consent                 |
   |<-- 302 redirect_uri?code=...&state=...&iss=...        |                              |
   |--------------------->| verify state [8]              |                              |
   |                      |-- POST /token (code, code_verifier, client auth) ---------->|
   |                      |<-- { access_token [10], id_token [12], refresh_token [11] } -|
   |                      | verify ID token sig via JWKS [14] (RS256 [15]),              |
   |                      | check iss/aud/exp/nonce [9], map (iss,sub) -> local user    |
   |<-- Set-Cookie: __Host-sid=...; HttpOnly; Secure; SameSite=Lax  [16][17]             |
   |                      |                               |                              |
   |  GET /api/invoices   |                               |                              |
   |--------------------->|-- Authorization: Bearer <access_token> ------------------->|
   |                      |                               | verify JWT [13][14][15]      |
   |                      |                               | authz: role -> permission [2][18][19]
   |                      |                               | tenant ctx -> RLS [20]       |
   |<---------------------|<------------------------------ 200 JSON ---------------------|
```

### Decision defaults (use these unless you have a measured reason not to)

| Decision | Default |
| --- | --- |
| Where do users log in? | Managed IdP over OIDC. Do not store passwords yourself. |
| Flow | Authorization Code + PKCE (S256), always, including confidential clients. |
| Browser app | Backend-for-Frontend holds tokens; browser holds only an `HttpOnly` session cookie. |
| Access token | Short-lived (5 to 15 min) signed JWT (RFC 9068 profile), audience-restricted. |
| Refresh token | Opaque, rotated on every use, reuse detection, held server-side. |
| Signing | RS256 (broadest compatibility) or ES256/EdDSA (smaller, faster); JWKS with `kid` rotation. |
| AuthZ | Deny by default; check permissions not roles; enforce object ownership and tenant in the data layer. |
| Multi-tenancy | Tenant from verified token claim, enforced by Postgres RLS plus application checks. |

---

## 1. Authentication

Authentication (authn) answers one question: *who is making this request?* It binds a request to a verified principal by checking proof of an identity claim. It produces an identity assertion (a session, a token, a certificate), never a permission decision.

### Core mechanism

- A principal presents a claim ("I am user 42") plus evidence from one or more factors: knowledge (password), possession (TOTP device, passkey, hardware key), inherence (biometrics, which stay local to a device).
- The verifier compares evidence to stored reference data, then issues a durable artifact so the proof is not repeated on every request: a server-side session ID, a signed token, or a client certificate.
- Architecturally it lives at the edge of trust: the login endpoint (or the IdP) performs the proof; every other service only validates the artifact. Keep the proof step in exactly one place.
- NIST SP 800-63B defines Authenticator Assurance Levels: AAL1 (single factor), AAL2 (two factors, with a phishing-resistant option offered), AAL3 (hardware-bound, verifier-impersonation resistant).

### Engineering blueprint

Password verification, minimum viable shape:

```text
stored = argon2id$v=19$m=19456,t=2,p=1$<salt b64>$<hash b64>

verify(input, stored):
  params, salt, hash = parse(stored)
  candidate = argon2id(input, salt, params)
  return constant_time_equal(candidate, hash)     // never ==
```

| Primitive | Parameters (OWASP Password Storage Cheat Sheet) |
| --- | --- |
| Argon2id | m=19 MiB, t=2, p=1 minimum; raise memory until a hash takes ~250 ms on your hardware |
| scrypt | N=2^17, r=8, p=1 |
| bcrypt | cost >= 10 (legacy only; 72-byte input limit, so pre-hash or reject longer input) |
| PBKDF2-HMAC-SHA256 | 600,000 iterations (FIPS-constrained environments only) |

MFA and passwordless:

- **TOTP (RFC 6238):** `HOTP(K, floor(unixtime / 30))`, 6 digits, HMAC-SHA1 by default. Phishable by real-time relay.
- **WebAuthn / passkeys (W3C):** the authenticator signs a server challenge with a per-origin private key. The origin is part of the signed data, so a look-alike domain cannot harvest a reusable assertion. This is the only mainstream phishing-resistant factor.
- **Server-side WebAuthn verification:** random `challenge` (>= 16 bytes) stored with the session; verify `clientDataJSON.type`, `.challenge`, `.origin`; verify `rpIdHash`; verify the signature over `authenticatorData || SHA256(clientDataJSON)`; check `signCount` did not regress (when non-zero).

### Production pitfalls

- Credential stuffing and brute force: no rate limit per account *and* per IP/device; no progressive delay.
- Account enumeration via different messages, sizes, or timing for "unknown user" vs "wrong password". Return one generic message and equalize work (hash a dummy value for unknown users).
- Fast hashes (SHA-256, MD5) for passwords; unsalted or globally salted hashes.
- Weak recovery paths: password reset or SMS fallback undoes MFA. Reset tokens must be single-use, >= 128 bits of entropy, stored hashed, expiring in <= 15 minutes.
- MFA fatigue (push bombing) and SIM-swap against SMS OTP.
- Treating a login as proof forever: no re-authentication before sensitive actions (email change, payout, API key creation).
- Session fixation: not rotating the session ID at login (see [Sessions](#17-sessions)).

### SDE best practices

- Do not build your own credential store unless you must. Delegate to an IdP ([section 3](#3-identity-provider-idp)) and consume OIDC; you inherit MFA, breach detection, and lockout.
- If you must store passwords: Argon2id, per-user random salt (>= 16 bytes), optional server-side pepper held in a KMS/HSM, rehash on login when parameters are outdated, screen against breached-password lists (Have I Been Pwned k-anonymity API), enforce length (>= 8 with MFA, >= 15 without; allow 64+) and drop composition rules (NIST 800-63B).
- Record `auth_time` and authentication methods (`amr`) in the session or token so downstream services can demand step-up (`acr_values`, `max_age`).
- Authenticate service-to-service calls separately from users: mTLS or OAuth client credentials with short-lived tokens, never one static API key shared fleet-wide.
- Log every auth event (success, failure, MFA enrol, reset, token issue) with user ID, IP, user agent, request ID. Never log the credential, token, or OTP.

### Resources

- [NIST SP 800-63B, Authentication & Lifecycle Management](https://pages.nist.gov/800-63-4/sp800-63b.html)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
- [OWASP Multifactor Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Multifactor_Authentication_Cheat_Sheet.html)
- [RFC 6238 (TOTP)](https://datatracker.ietf.org/doc/html/rfc6238) / [RFC 4226 (HOTP)](https://datatracker.ietf.org/doc/html/rfc4226)
- [W3C WebAuthn Level 3](https://www.w3.org/TR/webauthn-3/) and [passkeys.dev](https://passkeys.dev/)
- [Have I Been Pwned: Pwned Passwords API](https://haveibeenpwned.com/API/v3#PwnedPasswords)

---

## 2. Authorization

Authorization (authz) answers: *is this authenticated principal allowed to perform this action on this resource, in this context?* It is evaluated on every request, close to the data, and must not be inferred from the fact that authentication succeeded.

### Core mechanism

- A decision is a function: `allow? = f(subject, action, resource, context)`. Subject attributes (roles, groups, tenant), resource attributes (owner, tenant, state), and context (time, IP, MFA level) feed it.
- XACML vocabulary maps cleanly to modern code:
  - **PEP** (Policy Enforcement Point): middleware or a repository wrapper that asks and enforces.
  - **PDP** (Policy Decision Point): the engine that evaluates (Casbin, OPA, Cedar, OpenFGA, or hand-written code).
  - **PAP** (Policy Administration Point): where policies are authored and versioned.
  - **PIP** (Policy Information Point): where extra attributes come from (DB, directory, token claims).
- Enforce in layers, each stricter than the last:

```text
 L1 Gateway     : token valid? audience right? scope covers this route?        (coarse, cheap)
 L2 Service     : does the user's permission set allow this operation?          (RBAC / policy engine)
 L3 Object      : does THIS record belong to this user / tenant / team?        (ownership, ReBAC, RLS)
 L4 Field       : may they see/modify this attribute (PII, price, admin flags)? (serializer allow-lists)
```

### Engineering blueprint

Model comparison:

| Model | Decides on | Strength | Weakness |
| --- | --- | --- | --- |
| ACL | per-object list of principals | simple, explicit | does not scale, hard to audit |
| RBAC | role -> permissions | easy to reason about, auditable | role explosion, no object context |
| ABAC / PBAC | attributes + policy rules | fine-grained, contextual | policy complexity, harder to test |
| ReBAC | relationships in a graph (Zanzibar) | sharing, hierarchies (folder -> doc) | needs a graph store, consistency model |

Scopes are **not** user permissions. A scope is what the *user delegated to a client*. The effective permission is the intersection:

```text
effective(request) = permissions(user)  ∩  scopes(access_token)  ∩  object_policy(resource)
```

A minimal enforcement middleware (TypeScript/Express):

```ts
const can = (perm: string) => async (req, res, next) => {
  const { sub, tenant, scopes } = req.auth;                  // from verified token
  if (!scopes.has(requiredScopeFor(perm))) return res.sendStatus(403);
  if (!(await pdp.enforce(sub, tenant, perm))) return res.sendStatus(403);
  next();                                                    // object-level check still happens in the handler/repo
};
router.get('/invoices/:id', can('invoice:read'), async (req, res) => {
  const inv = await repo.findById(req.auth.tenant, req.params.id);   // tenant in the WHERE clause
  if (!inv) return res.sendStatus(404);                               // 404 not 403: do not leak existence
  res.json(serialize(inv, req.auth));                                 // field-level filtering
});
```

### Production pitfalls

- **BOLA / IDOR** (OWASP API #1): `GET /invoices/123` checks the role but not that invoice 123 belongs to the caller. The most common critical API bug.
- **BFLA** (broken function-level auth): admin routes reachable by normal users because only the UI hides the button.
- Authorizing in the client or at the gateway only; internal services trust any caller on the network.
- Mass assignment: `PATCH /users/me` with `{"role":"admin"}` accepted because the model binder copies every field.
- Checking roles (`if user.isAdmin`) scattered through code; impossible to audit or change.
- Stale decisions: permissions baked into a long-lived JWT keep working after revocation.
- Fail-open on error: a PDP timeout returning `allow`. Default must be deny.
- Confused deputy: service A calls service B with its own powerful identity on behalf of a user without propagating the user's authority.

### SDE best practices

- **Deny by default.** Every route declares its required permission; a route with none fails CI (lint or test that enumerates routes).
- Centralize decisions behind one interface (`enforce(sub, tenant, action, resource)`), so the engine can change without touching handlers.
- Put the object/tenant predicate in the query (`WHERE tenant_id = $1 AND id = $2`), not in a post-fetch `if`. Back it with RLS ([section 20](#20-tenant-isolation)).
- Return 404 for objects the caller may not know exist; 403 when existence is already public.
- Propagate the end-user identity across internal calls (token exchange, RFC 8693, or a signed internal context), and also authenticate the calling service.
- Keep token claims coarse (tenant, roles/groups); resolve fine-grained permissions server-side so revocation is immediate (cache with a short TTL and invalidation events).
- Write authorization tests as a matrix: for each role x each endpoint x (own object, other user's object, other tenant's object) assert allow/deny.
- Audit-log denials and privileged allows with the policy version that decided.

### Resources

- [OWASP Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)
- [OWASP API Security Top 10 (2023): API1 BOLA](https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/) and [API5 BFLA](https://owasp.org/API-Security/editions/2023/en/0xa5-broken-function-level-authorization/)
- [Google Zanzibar paper](https://research.google/pubs/zanzibar-googles-consistent-global-authorization-system/)
- [Open Policy Agent](https://www.openpolicyagent.org/docs/latest/), [Cedar](https://www.cedarpolicy.com/), [OpenFGA](https://openfga.dev/docs), [Casbin](https://casbin.org/docs/overview)
- [RFC 8693: OAuth 2.0 Token Exchange](https://datatracker.ietf.org/doc/html/rfc8693)
- [NIST SP 800-162: ABAC Guide](https://csrc.nist.gov/pubs/sp/800/162/upd2/final)

---

## 3. Identity Provider (IdP)

An Identity Provider is the system of record for identities and the component that performs authentication, then asserts the result to other systems. In OIDC terms it is the **OpenID Provider (OP)**; in OAuth terms it hosts the **Authorization Server (AS)**; in SAML terms it is the **IdP**.

### Core mechanism

- Owns: user store, credentials, MFA enrolment, login UI, consent screens, session (the SSO session), signing keys, client registry.
- Exposes: `/authorize`, `/token`, `/userinfo`, `/introspect`, `/revoke`, `jwks_uri`, `end_session_endpoint`.
- Your application becomes a **Relying Party (RP)** or **client**: it redirects to the IdP, receives a signed assertion, and never sees the password.
- Federation: an IdP can itself delegate to upstream IdPs (Google, corporate Entra ID/Okta via SAML or OIDC), producing a broker topology.

### Engineering blueprint

Everything an RP needs is in the discovery document (OIDC Discovery, RFC 8414 for OAuth-only):

```http
GET https://idp.example.com/.well-known/openid-configuration
```

```json
{
  "issuer": "https://idp.example.com",
  "authorization_endpoint": "https://idp.example.com/oauth2/authorize",
  "token_endpoint": "https://idp.example.com/oauth2/token",
  "userinfo_endpoint": "https://idp.example.com/oauth2/userinfo",
  "jwks_uri": "https://idp.example.com/.well-known/jwks.json",
  "end_session_endpoint": "https://idp.example.com/oauth2/logout",
  "introspection_endpoint": "https://idp.example.com/oauth2/introspect",
  "revocation_endpoint": "https://idp.example.com/oauth2/revoke",
  "response_types_supported": ["code"],
  "grant_types_supported": ["authorization_code", "refresh_token", "client_credentials"],
  "subject_types_supported": ["public"],
  "id_token_signing_alg_values_supported": ["RS256", "ES256"],
  "code_challenge_methods_supported": ["S256"],
  "token_endpoint_auth_methods_supported": ["client_secret_basic", "private_key_jwt"],
  "scopes_supported": ["openid", "profile", "email", "offline_access"]
}
```

| Protocol | Format | Typical use | Notes |
| --- | --- | --- | --- |
| OIDC | JSON / JWT | consumer apps, SPAs, mobile, APIs | default for new builds |
| SAML 2.0 | XML (signed assertions) | enterprise SSO, legacy SaaS | XML signature wrapping bugs; use a vetted library |
| SCIM 2.0 (RFC 7643/7644) | JSON REST | user/group provisioning and deprovisioning | complements SSO; makes offboarding real |

Common products: Keycloak, Auth0, Okta, Microsoft Entra ID, AWS Cognito, Google Identity, Ory Hydra/Kratos, Zitadel, Authentik, Dex.

### Production pitfalls

- **Identifying users by email** across IdPs. Emails change and are reassignable; an attacker-controlled IdP asserting `victim@corp.com` yields account takeover (the "nOAuth" class). Key accounts on the pair `(iss, sub)`.
- Selecting the JWKS by trusting the unverified `iss` in the token. Only accept issuers from an allow-list you configured.
- Multi-tenant issuers (Entra `/common`, per-tenant realms): the `iss` contains a tenant ID that must be validated against the tenant you expect, not just "is a valid Entra token".
- Not handling IdP downtime: login is a hard dependency. Cache JWKS, keep sessions alive independently, design a degraded mode for already-authenticated users.
- Hard-coding endpoints instead of reading discovery; breaks on key or URL changes.
- Wildcard or loose redirect URIs registered at the IdP.
- Deprovisioning gap: user disabled at the IdP, but your app session and refresh tokens live on for weeks.

### SDE best practices

- Build vs buy: buy unless identity is your product. Self-hosting Keycloak/Ory shifts patching, HA, and key management onto you.
- Use the discovery document at startup; refresh periodically; validate that `issuer` in the document equals the URL you fetched it from.
- Keep your own `users` table keyed by `(issuer, subject)` and do just-in-time provisioning; store only what you need from claims.
- Set a **maximum app-session lifetime** shorter than your tolerance for deprovisioning lag, and support OIDC back-channel logout or SCIM deactivation hooks.
- Register one client per application and environment; exact-match redirect URIs; separate public (SPA/mobile) from confidential (server) clients.
- Treat the IdP's signing keys and admin console as tier-0 assets: MFA on admins, audit logs shipped out, break-glass account stored offline.

### Resources

- [OpenID Connect Discovery 1.0](https://openid.net/specs/openid-connect-discovery-1_0.html)
- [RFC 8414: OAuth 2.0 Authorization Server Metadata](https://datatracker.ietf.org/doc/html/rfc8414)
- [OpenID Connect Core: Claims & Identifiers (sub, iss)](https://openid.net/specs/openid-connect-core-1_0.html#ClaimStability)
- [OASIS SAML 2.0 overview](https://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-tech-overview-2.0.html)
- [RFC 7644: SCIM Protocol](https://datatracker.ietf.org/doc/html/rfc7644)
- [Keycloak docs](https://www.keycloak.org/documentation), [Ory docs](https://www.ory.sh/docs/), [Auth0 docs](https://auth0.com/docs)
- [OIDC Back-Channel Logout](https://openid.net/specs/openid-connect-backchannel-1_0.html)

---

## 4. OAuth 2.0

OAuth 2.0 (RFC 6749) is a **delegated authorization** framework: a resource owner lets a client obtain limited access to a resource server without sharing credentials. It is not an authentication protocol (that is [OIDC](#5-openid-connect-oidc)).

### Core mechanism

Four roles:

| Role | Example |
| --- | --- |
| Resource Owner | the end user |
| Client | your web/mobile/server app (public or confidential) |
| Authorization Server (AS) | the IdP's OAuth endpoints |
| Resource Server (RS) | your API that accepts access tokens |

Grant types (what to use today):

| Grant | Use | Status |
| --- | --- | --- |
| Authorization Code + PKCE | any user-facing app | **default** |
| Client Credentials | service-to-service, no user | current |
| Refresh Token | renew access without re-login | current |
| Device Authorization (RFC 8628) | TVs, CLIs, IoT | current |
| JWT Bearer (RFC 7523), Token Exchange (RFC 8693) | federation, delegation/impersonation | current |
| Implicit | tokens in URL fragment | **removed in OAuth 2.1; do not use** |
| Resource Owner Password Credentials | app collects the password | **removed in OAuth 2.1; do not use** |

### Engineering blueprint

Token endpoint, client credentials with `private_key_jwt` (preferred over shared secrets):

```http
POST /oauth2/token HTTP/1.1
Host: idp.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&scope=reports.read
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=eyJhbGciOiJSUzI1NiIsImtpZCI6ImNsaWVudC1rZXktMSJ9...
```

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6ImF0K2p3dCIsImtpZCI6IjIwMjYtMTAifQ...",
  "token_type": "Bearer",
  "expires_in": 600,
  "scope": "reports.read"
}
```

Client authentication methods at `/token`: `client_secret_basic`, `client_secret_post`, `private_key_jwt`, `tls_client_auth` (mTLS), `none` (public clients, PKCE only).

Bearer usage (RFC 6750): `Authorization: Bearer <token>`. Error responses: `WWW-Authenticate: Bearer error="invalid_token", error_description="expired"`.

OAuth 2.1 (draft) folds in the security BCP (RFC 9700): PKCE required for all code flows, exact redirect URI matching, no implicit, no ROPC, no bearer tokens in query strings, refresh tokens for public clients must be sender-constrained or rotated.

### Production pitfalls

- **Using OAuth for login** ("pseudo-authentication"): an access token proves the user once consented to a client, not who is present. Use OIDC and the ID token.
- Redirect URI validation by prefix/wildcard/regex: open redirects and subdomain takeovers leak authorization codes. Require exact string match.
- Authorization code injection and CSRF on the callback: mitigated by PKCE and `state` ([7](#7-pkce-proof-key-for-code-exchange), [8](#8-state-parameter)).
- **Mix-up attacks** when a client talks to several ASes: validate the `iss` response parameter (RFC 9207) and use per-AS redirect URIs.
- Over-broad scopes ("*", "admin") requested by default; consent fatigue.
- Tokens accepted by the wrong API (no audience check): a token minted for API A replayed against API B.
- Tokens in URLs (query string, fragment): land in logs, `Referer`, browser history.
- Shared client secrets in mobile/SPA bundles: not secret. Public clients have no secret; rely on PKCE.
- Client secrets without rotation plan or leaked via CI logs.

### SDE best practices

- Authorization Code + PKCE for every interactive client; Client Credentials with short-lived tokens for machines.
- Use Resource Indicators (RFC 8707) or the `audience` parameter so tokens are minted for exactly one API; RS verifies `aud`.
- Define scopes as capabilities of your API (`invoices:read`, `invoices:write`), not roles. Document each with the data it exposes.
- Make clients first-class: ownership, redirect URIs, secrets/keys with expiry, per-client token lifetimes, audit trail.
- Pushed Authorization Requests (PAR, RFC 9126) and signed request objects (JAR, RFC 9101) for high-assurance setups (open banking: FAPI profile).
- Never roll your own AS. Certified implementations exist (OpenID Foundation conformance suite).

### Resources

- [RFC 6749: The OAuth 2.0 Authorization Framework](https://datatracker.ietf.org/doc/html/rfc6749)
- [RFC 6750: Bearer Token Usage](https://datatracker.ietf.org/doc/html/rfc6750)
- [RFC 9700: Best Current Practice for OAuth 2.0 Security](https://datatracker.ietf.org/doc/html/rfc9700)
- [OAuth 2.1 (IETF draft)](https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/)
- [RFC 6819: OAuth 2.0 Threat Model](https://datatracker.ietf.org/doc/html/rfc6819)
- [RFC 8252: OAuth for Native Apps](https://datatracker.ietf.org/doc/html/rfc8252)
- [RFC 8628: Device Authorization Grant](https://datatracker.ietf.org/doc/html/rfc8628)
- [RFC 8707: Resource Indicators](https://datatracker.ietf.org/doc/html/rfc8707), [RFC 9207: Issuer Identification](https://datatracker.ietf.org/doc/html/rfc9207), [RFC 9126: PAR](https://datatracker.ietf.org/doc/html/rfc9126), [RFC 9101: JAR](https://datatracker.ietf.org/doc/html/rfc9101)
- [OAuth 2.0 Simplified (Aaron Parecki)](https://www.oauth.com/)
- [OWASP OAuth 2.0 Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/OAuth2_Cheat_Sheet.html)
- [PortSwigger Web Security Academy: OAuth](https://portswigger.net/web-security/oauth)

---

## 5. OpenID Connect (OIDC)

OpenID Connect 1.0 is an identity layer on top of OAuth 2.0. It adds a standard way to learn *who authenticated and how*: the **ID token**, the `openid` scope, a **UserInfo** endpoint, discovery, and standard claims.

### Core mechanism

- An OIDC request is an OAuth request with `scope` containing `openid`. The AS (now an OP) returns an extra artifact, the `id_token`, a signed JWT about the authentication event.
- Separation of concerns: the ID token is *for the client* (audience = `client_id`); the access token is *for the resource server*.
- Standard scopes map to claim sets: `profile`, `email`, `address`, `phone`; `offline_access` requests a refresh token.
- Flows: Authorization Code (use this), Implicit and Hybrid (legacy).
- Session management specs: RP-Initiated Logout, Front-Channel Logout, Back-Channel Logout.

### Engineering blueprint

Authentication request:

```http
GET /authorize?response_type=code
  &client_id=web-app
  &redirect_uri=https%3A%2F%2Fapp.example.com%2Fcallback
  &scope=openid%20profile%20email
  &state=Zp3...        &nonce=n-0S6_WzA2Mj
  &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM&code_challenge_method=S256
  &prompt=login        &max_age=300        &acr_values=urn:example:mfa
```

Useful request parameters: `prompt` (`none | login | consent | select_account`), `max_age` (force re-auth if older), `acr_values` (request assurance level), `login_hint`, `id_token_hint`, `claims` (request specific claims).

UserInfo (call with the access token; the `sub` must match the ID token's `sub`):

```http
GET /userinfo
Authorization: Bearer <access_token>
```
```json
{ "sub": "248289761001", "name": "Jane Doe", "email": "jane@example.com", "email_verified": true }
```

Logout:

- **RP-initiated:** redirect to `end_session_endpoint?id_token_hint=...&post_logout_redirect_uri=...&state=...`.
- **Back-channel:** the OP POSTs a signed `logout_token` (JWT with `events` and `sid`/`sub`) to your `backchannel_logout_uri`; you delete matching sessions server-side.

### Production pitfalls

- Treating `email` as identity key; not checking `email_verified`.
- Skipping ID token validation steps ([section 12](#12-id-token) checklist) because "the library did it".
- Using the access token as proof of authentication, or the ID token as an API credential.
- Putting authorization data (roles, permissions) only in the ID token: it is meant for the client, may be stale, and some IdPs make claims availability differ between ID token and UserInfo.
- No logout propagation: the user "logs out" of your app but the IdP session lives on (one-click re-login) or the reverse (IdP logout leaves app sessions active).
- `prompt=none` silent auth in iframes fails under third-party cookie restrictions; use refresh tokens or top-level redirects.
- Accepting `alg: none` ID tokens or tokens signed with an algorithm the client did not register.

### SDE best practices

- Use a maintained OIDC client library (e.g. `openid-client` / `oauth4webapi` for Node, Authlib for Python, Spring Security / Nimbus for Java, `coreos/go-oidc` for Go). Configure from discovery.
- Key local users by `(iss, sub)`. Treat claims as a snapshot at login; refresh via UserInfo or token refresh when needed.
- Request the minimum scopes; request `offline_access` only if you really need background access.
- Enforce `max_age` / `acr` for sensitive operations and verify `auth_time` / `acr` in the returned ID token.
- Implement back-channel logout and keep a `sid -> app_session` index so you can terminate sessions on demand.

### Resources

- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
- [OIDC Discovery](https://openid.net/specs/openid-connect-discovery-1_0.html)
- [RP-Initiated Logout](https://openid.net/specs/openid-connect-rpinitiated-1_0.html), [Back-Channel Logout](https://openid.net/specs/openid-connect-backchannel-1_0.html)
- [OpenID Connect FAQ & certification](https://openid.net/certification/)
- [oauth4webapi (panva)](https://github.com/panva/oauth4webapi), [openid-client](https://github.com/panva/node-openid-client), [coreos/go-oidc](https://github.com/coreos/go-oidc), [Authlib](https://docs.authlib.org/)
- [Okta: OIDC overview](https://developer.okta.com/docs/concepts/oauth-openid/)

---

## 6. Authorization Code Flow

The front-channel/back-channel split flow: the browser only ever carries a short-lived, single-use **authorization code**; the actual tokens are exchanged server-to-server on the back channel. It is the recommended grant for web apps, SPAs (via BFF), and mobile apps (with PKCE).

### Core mechanism

```text
 User-Agent               Client (confidential: BFF/server)            Authorization Server
     |                                |                                         |
 (1) | GET /login                     |                                         |
     |------------------------------->| generate state, nonce, code_verifier    |
     |                                | save in session; compute code_challenge |
 (2) |<-- 302 Location: /authorize?... (front channel)                          |
     |----------------------------------------------------------------------->  |
 (3) |            login + MFA + consent (AS session cookie)                     |
 (4) |<-- 302 Location: redirect_uri?code=SplxlOBe...&state=...&iss=...          |
     |------------------------------->| verify state                            |
 (5) |                                |-- POST /token (back channel) ---------->|
     |                                |   grant_type=authorization_code         |
     |                                |   code, redirect_uri, code_verifier     |
     |                                |   + client authentication               |
 (6) |                                |<-- 200 {access_token,id_token,refresh}  |
     |                                | validate ID token; create app session   |
 (7) |<-- Set-Cookie: __Host-sid (HttpOnly) |                                    |
```

Properties the design relies on:

- The code is bound to `client_id`, `redirect_uri`, and (with PKCE) the verifier. It is useless alone.
- Single use, short TTL (RFC 6749 recommends <= 10 minutes; use 60 seconds or less). Reuse must be treated as an attack and the tokens issued from that code revoked.
- Tokens never pass through the browser URL, so they stay out of history, logs, and `Referer`.

### Engineering blueprint

Step 5, the token request (confidential client, secret in Basic auth):

```http
POST /oauth2/token HTTP/1.1
Host: idp.example.com
Authorization: Basic d2ViLWFwcDpzM2NyM3Q=
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=SplxlOBeZQQYbYS6WxSbIA
&redirect_uri=https%3A%2F%2Fapp.example.com%2Fcallback
&code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
```

Callback handler skeleton (Node, using `openid-client`-style API):

```ts
app.get('/callback', async (req, res) => {
  const saved = req.session.oidc;                       // { state, nonce, verifier } from step 1
  delete req.session.oidc;                              // single use
  if (!saved || req.query.state !== saved.state) return res.status(400).end();
  if (req.query.error) return res.status(400).send(String(req.query.error));
  if (req.query.iss && req.query.iss !== ISSUER) return res.status(400).end();   // RFC 9207

  const tokens = await client.callback(REDIRECT_URI, req.query,
    { state: saved.state, nonce: saved.nonce, code_verifier: saved.verifier });
  const claims = tokens.claims();                       // signature, iss, aud, exp, nonce already verified
  const user = await users.upsertByIssuerSub(claims.iss, claims.sub, claims);
  await regenerateSession(req);                         // new session ID -> prevents fixation
  req.session.userId = user.id;
  req.session.tokens = encrypt(tokens);                 // server-side only
  res.redirect('/');
});
```

**BFF pattern** for SPAs (current IETF browser-based-apps guidance): the SPA never receives tokens. The BFF performs the code flow, stores tokens server-side, and proxies API calls, attaching `Authorization` itself. The browser holds an `HttpOnly; Secure; SameSite` session cookie. This removes the XSS-steals-token class of attack.

### Production pitfalls

- Missing or non-validated `state` (login CSRF) and missing PKCE (code injection/interception).
- Registering broad `redirect_uri` patterns; code delivered to an attacker-controlled path or subdomain.
- Exchanging the code from the browser with an embedded client secret.
- Not sending the **same** `redirect_uri` at the token step (AS rejects with `invalid_grant`; confusing to debug).
- Not regenerating the session after login (fixation).
- Race: user double-clicks "login", two codes, the second callback arrives with a stale cookie. Keep per-attempt state keyed by `state` value rather than one global slot.
- Open redirect after login: `?next=` parameter used without allow-listing; post-login redirect to attacker site.
- Mobile apps using custom URI schemes that other apps can claim; prefer claimed HTTPS links (App Links/Universal Links) plus PKCE (RFC 8252).
- Loopback redirect handling for CLIs: use `http://127.0.0.1:<random-port>/callback` (RFC 8252 section 7.3) or device flow.

### SDE best practices

- Always PKCE (S256), even for confidential clients; it is defense in depth.
- Generate `state`, `nonce`, and `code_verifier` with a CSPRNG (>= 32 bytes), store them server-side (or in an encrypted/signed short-lived cookie scoped to the callback path), and delete on first use.
- Authenticate confidential clients with `private_key_jwt` or mTLS where possible instead of static secrets.
- Time-box the whole login transaction (e.g. 10 minutes) and clean up abandoned ones.
- After the code exchange, drop the code and verifier immediately; never log request URLs containing `code=`.
- Test the failure paths: user denies consent (`error=access_denied`), `login_required`, expired code, replayed code, state mismatch, clock skew.

### Resources

- [RFC 6749 section 4.1: Authorization Code Grant](https://datatracker.ietf.org/doc/html/rfc6749#section-4.1)
- [OIDC Core section 3.1: Authorization Code Flow](https://openid.net/specs/openid-connect-core-1_0.html#CodeFlowAuth)
- [OAuth 2.0 for Browser-Based Apps (BCP draft, covers BFF)](https://datatracker.ietf.org/doc/draft-ietf-oauth-browser-based-apps/)
- [RFC 8252: OAuth 2.0 for Native Apps](https://datatracker.ietf.org/doc/html/rfc8252)
- [RFC 9700 section 4: Attacks and Mitigations](https://datatracker.ietf.org/doc/html/rfc9700#section-4)
- [Auth0: Authorization Code Flow](https://auth0.com/docs/get-started/authentication-and-authorization-flow/authorization-code-flow)
- [Duende BFF documentation (pattern explained)](https://docs.duendesoftware.com/bff/)

---

## 7. PKCE (Proof Key for Code Exchange)

PKCE (RFC 7636, pronounced "pixy") binds the authorization request to the token request with a one-time secret that never leaves the client until the back channel. It defeats authorization code interception and injection.

### Core mechanism

1. Client creates a random `code_verifier`.
2. Client derives `code_challenge = BASE64URL(SHA256(ASCII(code_verifier)))` and sends it (plus `code_challenge_method=S256`) in the `/authorize` request.
3. AS stores the challenge with the issued code.
4. At `/token`, client sends the raw `code_verifier`. AS recomputes the hash and compares. Mismatch => `invalid_grant`.

An attacker who steals the code (malicious app registered for the same custom scheme, referrer leak, proxy log) lacks the verifier, so the code is worthless.

### Engineering blueprint

Constraints (RFC 7636): verifier is 43 to 128 characters from `[A-Z] [a-z] [0-9] - . _ ~`; use 32 random bytes encoded as base64url (43 chars).

```ts
import { randomBytes, createHash } from 'node:crypto';

const b64url = (b: Buffer) => b.toString('base64url');
const code_verifier  = b64url(randomBytes(32));                       // 43 chars
const code_challenge = b64url(createHash('sha256').update(code_verifier).digest());
// request:  code_challenge=<code_challenge>&code_challenge_method=S256
// token  :  code_verifier=<code_verifier>
```

```python
import os, base64, hashlib
verifier  = base64.urlsafe_b64encode(os.urandom(32)).rstrip(b"=").decode()
challenge = base64.urlsafe_b64encode(hashlib.sha256(verifier.encode()).digest()).rstrip(b"=").decode()
```

Test vector from RFC 7636 appendix B:

```text
verifier  = dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
challenge = E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
```

Server side (if you build an AS, which you should not): persist `(code, client_id, redirect_uri, code_challenge, method)`; at exchange, require a verifier whenever a challenge was stored; compare in constant time; reject `plain` unless forced by legacy clients; **reject a token request that includes a verifier when no challenge was bound** (downgrade).

### Production pitfalls

- Using `plain` method (challenge == verifier): protects nothing if the authorization request is observable.
- Low-entropy verifier (timestamp, user ID, `Math.random()`).
- Reusing one verifier across logins or storing it in `localStorage` where XSS or another tab reads it. Keep it per-transaction, in memory/session.
- AS not enforcing PKCE: client sends it, AS ignores it, attacker simply omits it (**downgrade**). Enforce server-side: "PKCE required for this client".
- Believing PKCE replaces `state`/`nonce`. It stops code injection; it does not by itself bind the ID token to the session (see [nonce](#9-nonce-parameter)). RFC 9700 says PKCE can serve CSRF protection if the AS supports it, but keep `state` for app-level data and defense in depth.
- Base64 (standard) instead of base64url, or leaving `=` padding: hash mismatch.

### SDE best practices

- Make PKCE mandatory for **all** clients (public and confidential) at the AS configuration level, S256 only.
- Generate with the platform CSPRNG; never hand-roll the base64url or hashing: use the library's helper.
- One verifier per authorization attempt, discarded on use or timeout.
- Discovery check: refuse to proceed if `code_challenge_methods_supported` lacks `S256`.

### Resources

- [RFC 7636: PKCE](https://datatracker.ietf.org/doc/html/rfc7636)
- [RFC 9700 section 2.1.1: PKCE recommendation](https://datatracker.ietf.org/doc/html/rfc9700#section-2.1.1)
- [OAuth.com: PKCE](https://www.oauth.com/oauth2-servers/pkce/)
- [Auth0: Authorization Code Flow with PKCE](https://auth0.com/docs/get-started/authentication-and-authorization-flow/authorization-code-flow-with-pkce)

---

## 8. State Parameter

`state` is an opaque, unguessable value the client puts in the authorization request; the AS echoes it unchanged in the response. The client verifies the echo matches the one it stored for **this** browser. It prevents CSRF on the redirect URI ("login CSRF"), where an attacker forces the victim's browser to complete a flow with the attacker's authorization code and thereby log the victim into the attacker's account.

### Core mechanism

```text
Attack without state:
  1. Attacker starts a flow with the IdP, gets  /callback?code=ATTACKER_CODE  but does not follow it.
  2. Attacker lures the victim to  https://app.example.com/callback?code=ATTACKER_CODE
  3. App exchanges the code, the victim's session is now bound to the attacker's account.
     Anything the victim uploads (files, cards) lands in the attacker's account.

With state:
  App stores state=S in the VICTIM'S browser session before redirect.
  Callback must carry state == S from the same browser. Attacker's link carries a different/no state -> rejected.
```

### Engineering blueprint

```ts
// at /login
const state = randomBytes(32).toString('base64url');
req.session.oidc = { state, nonce, verifier, returnTo: safeReturnTo(req.query.next) };
// authorize URL includes state

// at /callback
if (!timingSafeEqualStr(req.query.state, req.session.oidc?.state)) return reject(400);
```

Stateless variant: put app data in the `state` as a signed + encrypted blob (`HMAC`/`AEAD`) containing `{nonce, returnTo, exp}` and **also** bind it to the browser with a cookie holding a hash of it. A signature alone is not binding: an attacker can obtain a validly signed state from their own session.

### Production pitfalls

- **Absent state**, or present but never compared.
- Static or predictable value (constant string, incrementing counter, user ID).
- Not bound to the user's browser (global in-memory map keyed by state only): attacker's own valid state works in the victim's browser.
- Reusable: state kept after use; allows replay.
- Putting sensitive data in state: it is visible in the URL and logs. If you carry `returnTo`, validate it against an allow-list of relative paths to avoid open redirects.
- Cookie holding state set with `SameSite=Strict`: the cross-site redirect back from the IdP does not carry it; use `Lax` for the login-transaction cookie.
- Multiple concurrent logins in different tabs overwriting one session slot.

### SDE best practices

- >= 128 bits from a CSPRNG; store per-attempt (map of state -> transaction) with TTL; delete on first use.
- Compare in constant time; fail closed with a clear user-facing "login expired, try again".
- Use `state` for CSRF and app routing only; use PKCE for code binding and `nonce` for ID-token binding. Do not conflate the three.
- Validate the `iss` response parameter as well (RFC 9207) when multiple ASes are possible.

### Resources

- [RFC 6749 section 10.12: CSRF](https://datatracker.ietf.org/doc/html/rfc6749#section-10.12)
- [RFC 9700 section 4.7: CSRF](https://datatracker.ietf.org/doc/html/rfc9700#section-4.7)
- [OWASP CSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
- [PortSwigger: OAuth authentication vulnerabilities (forced profile linking)](https://portswigger.net/web-security/oauth)
- [Auth0: Prevent attacks and redirect users with OAuth 2.0 state parameters](https://auth0.com/docs/secure/attack-protection/state-parameters)

---

## 9. Nonce Parameter

`nonce` is an OIDC parameter that binds an **ID token** to the client session that requested it. The client sends a random value in the authorization request; the OP copies it verbatim into the ID token's `nonce` claim; the client checks equality. It prevents ID-token replay and injection: an attacker cannot feed the client an ID token minted for a different authentication request.

### Core mechanism

```text
Client session S1:  nonce = N1  -> /authorize?...&nonce=N1
OP issues id_token: { ..., "nonce": "N1", "aud": "web-app" }
Client verifies: id_token.nonce == S1.nonce  (and signature, iss, aud, exp)

Replay: attacker captured an old id_token (nonce N0) and injects it into S2 (expects N2) -> mismatch -> rejected.
```

Where required: **mandatory for Implicit and Hybrid** flows (ID token exposed via front channel). For the Code flow it is optional per spec but recommended; with a code flow the ID token arrives over the back channel, and PKCE protects the code, so nonce adds defense in depth against token injection.

### State vs nonce vs PKCE

| Parameter | Binds | Travels in | Checked by | Stops |
| --- | --- | --- | --- | --- |
| `state` | response -> browser session | redirect URL (echoed) | client | login CSRF |
| `code_verifier` | token request -> authorization request | back channel | **AS** | code interception/injection |
| `nonce` | ID token -> authorization request | inside signed ID token | client | ID-token replay/injection |

### Engineering blueprint

```ts
const nonce = randomBytes(32).toString('base64url');
req.session.oidc.nonce = nonce;           // or store sha256(nonce) in a cookie and send nonce in the URL
// ... later, after validating signature/iss/aud/exp:
if (claims.nonce !== req.session.oidc.nonce) throw new Error('nonce mismatch');
delete req.session.oidc.nonce;            // one-time
```

Some OPs (Auth0, Okta) allow passing the raw nonce and checking is on the client; others (Google One Tap, Apple) may accept a **hashed** nonce when the raw value originates on a different device. Follow the provider's docs.

### Production pitfalls

- Sending a nonce but never verifying it (most common).
- Verifying only when present in the token, not when expected: if the session expected a nonce and the token has none, **fail**.
- Reusing one nonce across sessions; weak randomness.
- Storing the nonce in `localStorage` for an SPA doing implicit flow: XSS-readable and defeats the point.
- Confusing it with `jti` or with a replay-prevention cache for API calls. `nonce` is login-time; API replay is handled by short `exp` + TLS + DPoP.

### SDE best practices

- Always send and verify a nonce for OIDC logins, including code flow.
- Generate per transaction, store server-side with the `state` record, delete after verification.
- Let a library handle it (pass `nonce` into the callback check), but write a test that proves a mismatched nonce is rejected.
- For hybrid flow also validate `c_hash` / `at_hash` (hash of code/access token inside the ID token).

### Resources

- [OIDC Core section 3.1.2.1: Authentication Request (nonce)](https://openid.net/specs/openid-connect-core-1_0.html#AuthRequest)
- [OIDC Core section 15.5.2: Nonce Implementation Notes](https://openid.net/specs/openid-connect-core-1_0.html#NonceNotes)
- [OIDC Core section 3.1.3.7: ID Token Validation](https://openid.net/specs/openid-connect-core-1_0.html#IDTokenValidation)
- [Auth0: Mitigate replay attacks when using the Implicit Flow](https://auth0.com/docs/get-started/authentication-and-authorization-flow/implicit-flow-with-form-post/mitigate-replay-attacks-when-using-the-implicit-flow)

---

## 10. Access Token

An access token is the credential a client presents to a resource server to call an API. It encodes (or references) a grant: *this client, acting for this user (or itself), may do these things (scopes) to this audience until this time*. RFC 6749 treats it as opaque to the client; only the RS and AS need to understand it.

### Core mechanism

Two representations:

| | Opaque (reference) token | Structured token (JWT, RFC 9068) |
| --- | --- | --- |
| Content | random string, lookup key | signed claims |
| RS validation | call AS `/introspect` (RFC 7662) or shared store | verify signature via JWKS locally |
| Revocation | immediate (delete the record) | not until `exp` (unless extra checks) |
| Latency | network hop (cacheable) | none |
| Leak impact | no data disclosed | claims readable by anyone holding it |
| Fit | high-security, few RS, instant revoke | microservices, high-throughput APIs |

Bearer semantics: **whoever holds it can use it**. Sender-constrained tokens fix that by binding the token to a key the client proves possession of (DPoP, mTLS).

### Engineering blueprint

JWT access token per RFC 9068 (`typ: at+jwt`):

```json
// header
{ "alg": "RS256", "typ": "at+jwt", "kid": "2026-10-a" }
// payload
{
  "iss": "https://idp.example.com",
  "sub": "5f1c6a2e-...",                    // user ID (or client ID for client_credentials)
  "aud": "https://api.example.com",         // the RESOURCE SERVER, not the client
  "client_id": "web-app",
  "scope": "invoices:read invoices:write",
  "iat": 1760000000, "exp": 1760000600,     // 10 minutes
  "jti": "7d1b...",
  "tid": "tenant_42",                       // custom: tenant context
  "cnf": { "jkt": "0ZcOCORZNYy-DWpqq30jZyJGHTN0d2HglBV3uiguA4I" }   // only if DPoP-bound
}
```

Resource server validation order (all must pass):

1. `Authorization: Bearer <token>` present, exactly one scheme/header.
2. Parse; `typ` is `at+jwt`; `alg` in your allow-list (e.g. `RS256`), never from an attacker-influenced config.
3. Resolve key by `kid` from the **configured** issuer's JWKS; verify signature.
4. `iss` equals configured issuer; `aud` contains this API's identifier.
5. `exp` in the future, `nbf`/`iat` sane (+/- 30 to 60 s skew).
6. Required `scope`/permission for this route is present.
7. Extra checks if applicable: `jti` denylist, `cnf` proof (DPoP header), token-binding to tenant.

Opaque token introspection (RFC 7662):

```http
POST /introspect
Authorization: Basic <rs-credentials>
Content-Type: application/x-www-form-urlencoded

token=2YotnFZFEjr1zCsicMWpAA&token_type_hint=access_token
```
```json
{ "active": true, "scope": "invoices:read", "client_id": "web-app", "sub": "5f1c...", "aud": "https://api.example.com", "exp": 1760000600 }
```

DPoP (RFC 9449) request:

```http
GET /invoices HTTP/1.1
Authorization: DPoP eyJhbGciOiJSUzI1NiIs...
DPoP: eyJ0eXAiOiJkcG9wK2p3dCIsImFsZyI6IkVTMjU2IiwiandrIjp7Imt0eSI6IkVDIi...   // {typ:"dpop+jwt", jwk:{...}}.{htm:"GET", htu:"https://api.example.com/invoices", iat, jti, ath:<hash of access token>}
```

### Production pitfalls

- **Missing `aud` check**: any token from the same IdP works on any API (cross-service replay, confused deputy).
- Long-lived access tokens (hours/days) as a substitute for proper refresh handling.
- Accepting tokens via query string (`?access_token=`): ends up in logs, proxies, `Referer`.
- Storing in `localStorage`/`sessionStorage`: any XSS exfiltrates it. Mobile: plain SharedPreferences/UserDefaults.
- Logging full tokens (Authorization header in access logs, APM traces, error reports).
- Trusting scopes without matching them to user permissions and object ownership.
- Passing the end user's token to downstream services that should get a down-scoped one (token passthrough => lateral movement).
- Introspection call on every request without caching, creating a hot dependency on the AS; or caching too long and ignoring revocation.
- Using the ID token as an access token (wrong audience, wrong semantics).

### SDE best practices

- Lifetimes: 5 to 15 minutes for user tokens; shorter for high-risk scopes. Clients renew with a refresh token.
- Audience-restrict every token to one RS (or a small set); mint per-API tokens via resource indicators or token exchange for internal hops.
- Prefer JWT for inter-service scale, but pair with: short TTL, `jti` denylist for emergency revocation, and a **token version / session epoch** check for user-level kill switches (compare `iat` against `user.tokens_valid_after`).
- Sender-constrain (DPoP or mTLS) for high-value APIs and public clients holding tokens in the browser/mobile.
- Never put secrets or heavy PII in JWT access tokens; they are base64, not encrypted. Keep size small (< 1 KB ideally) so headers/cookies don't overflow limits.
- Redact `Authorization` and `Cookie` in all logging middleware by default.
- In microservices: verify at the edge **and** in each service (zero trust). Cache JWKS locally; if you introspect, cache positive results for seconds-to-minutes, keyed by token hash.

### Resources

- [RFC 6750: Bearer Token Usage](https://datatracker.ietf.org/doc/html/rfc6750)
- [RFC 9068: JWT Profile for OAuth 2.0 Access Tokens](https://datatracker.ietf.org/doc/html/rfc9068)
- [RFC 7662: Token Introspection](https://datatracker.ietf.org/doc/html/rfc7662)
- [RFC 9449: DPoP](https://datatracker.ietf.org/doc/html/rfc9449) and [RFC 8705: mTLS client auth & certificate-bound tokens](https://datatracker.ietf.org/doc/html/rfc8705)
- [RFC 7009: Token Revocation](https://datatracker.ietf.org/doc/html/rfc7009)
- [RFC 9700 section 2.2: Access token privilege restriction](https://datatracker.ietf.org/doc/html/rfc9700#section-2.2)
- [OAuth.com: Access Tokens](https://www.oauth.com/oauth2-servers/access-tokens/)

---

## 11. Refresh Token

A refresh token is a long-lived credential the client exchanges at the token endpoint for new access tokens without user interaction. It exists so access tokens can be short-lived while sessions stay long. It is only ever sent to the AS, never to resource servers.

### Core mechanism

```http
POST /oauth2/token
grant_type=refresh_token&refresh_token=8xLOxBtZp8&scope=invoices:read&client_id=web-app
```
```json
{ "access_token": "eyJ...", "expires_in": 600, "refresh_token": "9yMPyCuAq9", "token_type": "Bearer" }
```

Requested `scope` may only narrow, never widen, the original grant. In OIDC, a refresh token is issued only if `offline_access` was requested and consented (for most OPs).

### Engineering blueprint: rotation with reuse detection

Each refresh token is single-use and belongs to a **family** (one per login/grant). Using a token returns a new one and invalidates the old. Presenting an already-used token means theft or a bug: revoke the whole family.

```text
login      -> RT1 (family F)
refresh    -> present RT1 -> issue RT2, mark RT1 used
refresh    -> present RT2 -> issue RT3, mark RT2 used
attacker   -> present RT1 (stolen, already used) -> REUSE DETECTED -> revoke family F (RT3 dies too) -> force re-login
```

Server data model:

```sql
CREATE TABLE refresh_tokens (
  token_hash   bytea PRIMARY KEY,        -- SHA-256(token); token itself is 256-bit random so a fast hash is fine
  family_id    uuid NOT NULL,
  user_id      uuid NOT NULL,
  client_id    text NOT NULL,
  tenant_id    uuid NOT NULL,
  scopes       text[] NOT NULL,
  issued_at    timestamptz NOT NULL,
  idle_expires timestamptz NOT NULL,     -- sliding (e.g. 30 days)
  abs_expires  timestamptz NOT NULL,     -- hard cap (e.g. 90 days)
  used_at      timestamptz,              -- set on rotation
  replaced_by  bytea,
  revoked_at   timestamptz
);
```

Refresh handler pseudocode (transactional, `SELECT ... FOR UPDATE` on the row):

```text
row = find(hash(presented)) FOR UPDATE
if !row or row.revoked or now > row.idle_expires or now > row.abs_expires: 400 invalid_grant
if row.used_at != null:
    if now - row.used_at < GRACE (e.g. 10s) and same client/ip fingerprint: return the same replacement (network retry)
    else: revoke_family(row.family_id); 400 invalid_grant     // reuse detected
issue new RT (same family), new access token; mark row.used_at = now, replaced_by = new
```

### Production pitfalls

- **Refresh token in browser storage** (or in a JS-readable cookie): XSS gives the attacker a month of access. For SPAs use BFF; otherwise rotation + sender-constraint + short absolute lifetime.
- No rotation: a stolen token is valid until expiry and invisible.
- Rotation without a grace window: mobile network retries (response lost after server rotated) log legit users out. Conversely, an overlong grace window weakens detection.
- Concurrent refresh in multi-tab apps / parallel API calls: two requests present RT1 at once; one wins, the other trips reuse detection. Serialize refresh in the client (single-flight/mutex, `BroadcastChannel`, Web Locks).
- Storing refresh tokens in plaintext in DB (DB leak == mass account takeover). Store hashes.
- Not revoking on password change, MFA reset, "log out everywhere", or user deactivation.
- Refresh tokens that never expire and never require re-authentication; no step-up for risky changes.
- Letting a confidential client's refresh token be used without client authentication.
- Leaking refresh tokens to APIs (sent as `Authorization`) or logs.

### SDE best practices

- Opaque, >= 256-bit random, stored hashed server-side. Bind to client (and optionally device/DPoP key via `cnf.jkt`).
- Dual lifetimes: idle (e.g. 14 to 30 days) and absolute (e.g. 60 to 90 days; shorter for admin roles).
- Rotate on every use; family revocation on reuse; short grace (<= 10 s) for retry idempotency.
- On the **server** (BFF or backend) keep the refresh token encrypted at rest with a KMS key; the browser only holds the session cookie.
- Refresh proactively (e.g. at 80% of access token life) with single-flight; handle `invalid_grant` by redirecting to login, not by looping.
- Expose "active sessions/devices" so users can revoke; wire to `revoke_family`.
- Revocation endpoint (RFC 7009) on logout: revoke the refresh token (and the AS should cascade to access tokens when it can).

### Resources

- [RFC 6749 section 6: Refreshing an Access Token](https://datatracker.ietf.org/doc/html/rfc6749#section-6)
- [RFC 9700 section 4.14: Refresh Token Protection](https://datatracker.ietf.org/doc/html/rfc9700#section-4.14)
- [RFC 7009: Token Revocation](https://datatracker.ietf.org/doc/html/rfc7009)
- [Auth0: Refresh Token Rotation](https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation)
- [OAuth 2.0 Browser-Based Apps draft (refresh tokens in browsers)](https://datatracker.ietf.org/doc/draft-ietf-oauth-browser-based-apps/)
- [OIDC Core section 11: Offline Access](https://openid.net/specs/openid-connect-core-1_0.html#OfflineAccess)

---

## 12. ID Token

The ID token is a JWT issued by the OP that asserts the **result of an authentication event** to the client. Audience is the client. It answers "who logged in, when, and how", not "what may they call".

### Core mechanism

Mandatory claims: `iss`, `sub`, `aud`, `exp`, `iat`. Contextual: `auth_time`, `nonce`, `acr`, `amr`, `azp`, `at_hash`, `c_hash`, `sid`. Profile claims (`name`, `email`, `email_verified`, `picture`, `locale`) depend on scopes and OP behavior.

```json
{
  "iss": "https://idp.example.com",
  "sub": "248289761001",
  "aud": "web-app",
  "exp": 1760000600,
  "iat": 1760000000,
  "auth_time": 1759999990,
  "nonce": "n-0S6_WzA2Mj",
  "acr": "urn:example:mfa",
  "amr": ["pwd", "otp"],
  "at_hash": "77QmUPtjPfzWtF2AnpK9RQ",
  "sid": "08a5019c-17e1-4977-8f42-65a12843ea02",
  "email": "jane@example.com",
  "email_verified": true
}
```

`at_hash` = base64url of the **left half** of SHA-256(access_token) (for RS256/ES256), letting the client verify the access token wasn't swapped.

### Engineering blueprint: validation checklist (OIDC Core 3.1.3.7)

| # | Check | Failure it prevents |
| --- | --- | --- |
| 1 | Signature valid with a key from the OP's JWKS; `alg` equals the one registered for the client (default `RS256`), not `none` | forgery, alg confusion |
| 2 | `iss` exactly equals the OP's issuer from discovery | token from another IdP |
| 3 | `aud` contains your `client_id`; if multiple audiences, `azp` equals your `client_id` | token meant for another client |
| 4 | `exp` > now (with small skew) | stale token |
| 5 | `iat` not unreasonably old/future | pre-minted tokens |
| 6 | `nonce` equals the value sent | replay/injection |
| 7 | `auth_time` satisfies `max_age` if sent | stale authentication |
| 8 | `acr` satisfies requested level | step-up bypass |
| 9 | `at_hash` / `c_hash` match when present/required | token substitution |
| 10 | Token received directly from the token endpoint over TLS (code flow) lets you skip signature check per spec, but verifying anyway is cheap | n/a |

### Production pitfalls

- Sending the ID token to APIs as a bearer. It carries the wrong `aud`, and APIs that accept it can be fed ID tokens obtained by other apps (token substitution).
- Using `email` as the user key, or trusting `email` without `email_verified`.
- Keeping the ID token as a long-lived session proof: it expires in minutes by design; create your own session after validating.
- Storing it client-side beyond need; it contains PII.
- Not validating `aud`/`iss`, accepting any token the IdP signed (multi-tenant IdPs sign for thousands of apps).
- Assuming claims are fresh: group/role membership in an ID token is as of login.

### SDE best practices

- Validate once at login in the backend; then discard or keep only `sub`, `iss`, `sid`, `auth_time`, and `id_token` solely if you need `id_token_hint` for logout.
- Create a local session (cookie) and map `(iss, sub) -> user`; run just-in-time provisioning.
- Use `sid` to support back-channel logout; use `acr`/`amr`/`auth_time` for step-up decisions.
- Use `at_hash` check when the library offers it.
- Treat ID-token claims as untrusted input for display until validated; escape on render.

### Resources

- [OIDC Core section 2: ID Token](https://openid.net/specs/openid-connect-core-1_0.html#IDToken)
- [OIDC Core section 3.1.3.7: ID Token Validation](https://openid.net/specs/openid-connect-core-1_0.html#IDTokenValidation)
- [OIDC Core section 5.1: Standard Claims](https://openid.net/specs/openid-connect-core-1_0.html#StandardClaims)
- [Auth0: ID Tokens](https://auth0.com/docs/secure/tokens/id-tokens)
- [Why you shouldn't use ID tokens as access tokens (Auth0 blog)](https://auth0.com/blog/id-token-access-token-what-is-the-difference/)

---

## 13. JSON Web Token (JWT)

A JWT (RFC 7519) is a compact, URL-safe container for claims, protected by a **JWS** signature (RFC 7515) or **JWE** encryption (RFC 7516). It is a *format*, not a protocol: OAuth access tokens, OIDC ID tokens, and session tokens can all use it.

### Core mechanism

```text
BASE64URL(header) . BASE64URL(payload) . BASE64URL(signature)

header    = {"alg":"RS256","typ":"JWT","kid":"2026-10-a"}
payload   = {"iss":"https://idp.example.com","sub":"u_123","aud":"api","exp":1760000600,...}
signature = RSASSA-PKCS1-v1_5-SHA256( privateKey, ASCII(BASE64URL(header) + "." + BASE64URL(payload)) )
```

- **JWS compact**: 3 parts. Integrity + authenticity; payload is *readable by anyone*.
- **JWE compact**: 5 parts (`header.encryptedKey.iv.ciphertext.tag`). Confidentiality. Rarely needed; prefer not putting secrets in tokens.
- Registered claims: `iss` issuer, `sub` subject, `aud` audience, `exp` expiry, `nbf` not-before, `iat` issued-at, `jti` unique ID.
- Algorithms (RFC 7518): `HS256` (symmetric HMAC), `RS256/PS256` (RSA), `ES256` (ECDSA P-256), `EdDSA` (Ed25519, RFC 8037).

### Engineering blueprint

Verify with a vetted library, pinning everything:

```ts
import { createRemoteJWKSet, jwtVerify } from 'jose';

const JWKS = createRemoteJWKSet(new URL('https://idp.example.com/.well-known/jwks.json'));

export async function verifyAccessToken(token: string) {
  const { payload, protectedHeader } = await jwtVerify(token, JWKS, {
    issuer: 'https://idp.example.com',
    audience: 'https://api.example.com',
    algorithms: ['RS256'],         // allow-list; never read from the token
    typ: 'at+jwt',                 // prevents cross-JWT confusion (ID token used as access token)
    clockTolerance: 30,            // seconds
  });
  return payload;
}
```

```python
import jwt  # PyJWT
claims = jwt.decode(
    token, signing_key.key, algorithms=["RS256"],
    audience="https://api.example.com", issuer="https://idp.example.com",
    options={"require": ["exp", "iat", "iss", "aud", "sub"]},
)
```

Manual signature verification, to understand what libraries do:

```text
1. split on "." -> h, p, s  (exactly 3 parts)
2. header = JSON(b64url_decode(h)); reject if alg not in allow-list; reject unknown "crit"
3. key = JWKS[header.kid] (kid selects, never trusts jku/x5u/jwk header)
4. valid = rsa_verify(key, sha256, signing_input = h + "." + p, signature = b64url_decode(s))
5. only then parse payload and check claims
```

### Production pitfalls

- **`alg: none`**: unsigned token accepted by naive verifiers. Always specify the allowed algorithms.
- **Algorithm confusion (RS256 -> HS256)**: attacker signs with HMAC using the RSA *public key* as the secret; a verifier that picks the algorithm from the header and uses the same "key" variable accepts it. Fix: algorithm allow-list per key type.
- **Header-supplied keys**: `jwk`, `jku`, `x5u`, `x5c` honored from the token => attacker supplies their own key. Ignore them.
- **`kid` injection**: `kid` used in a SQL query or file path (`../../dev/null` => empty key). Treat as an opaque lookup into a pre-loaded key map.
- Not checking `exp`, `aud`, `iss`, or `nbf`. Treating "signature valid" as "authorized".
- **Cross-JWT confusion**: an ID token (valid, signed by the same IdP) accepted as an access token, or a token for service A accepted by service B. Pin `typ`, `aud`, and required claims.
- **Revocation is not built in**: stateless tokens are valid until `exp`. Solutions: short TTL + refresh, `jti` denylist, per-user `tokens_valid_after` check, opaque tokens for high-risk APIs.
- **Sensitive data in payload** (PII, internal IDs, permissions that change). Payload is only encoded.
- Weak HS256 secrets (brute-forceable offline with hashcat); secrets shared across services widen blast radius.
- Large tokens (many roles/permissions) exceeding header limits (8 KB commonly) or cookie limits (4 KB).
- Using JWTs as **session cookies** for browser apps without a revocation story, then being unable to log users out.
- Non-constant-time HMAC comparison; library bugs from outdated versions. Keep JOSE libraries patched.

### SDE best practices

- Use asymmetric signing (RS256/ES256/EdDSA) when more than one party verifies; HS256 only when issuer == verifier and the secret is >= 256-bit random in a secret manager.
- Verification checklist: pinned `alg`, key from trusted JWKS by `kid`, `iss`, `aud`, `exp`, `nbf`, `typ`, plus required claims present.
- Keep claims minimal and stable: `sub`, `tenant`, coarse roles/scopes. Look up fine-grained permissions at request time.
- TTL: access 5 to 15 min; never issue non-expiring JWTs.
- Add `jti` and a denylist (Redis with TTL = remaining lifetime) only where you need immediate revocation.
- Rotate signing keys (see [JWKS](#14-jwks-json-web-key-set)); never reuse the same key for different token types.
- Don't decode-and-trust in the frontend; client-side `jwt-decode` is for display only.
- Fuzz and unit-test your verification wrapper with known-bad tokens: `alg:none`, wrong `aud`, expired, tampered payload, HS256-with-public-key, missing `kid`.

### Resources

- [RFC 7519: JSON Web Token](https://datatracker.ietf.org/doc/html/rfc7519)
- [RFC 7515: JWS](https://datatracker.ietf.org/doc/html/rfc7515), [RFC 7516: JWE](https://datatracker.ietf.org/doc/html/rfc7516), [RFC 7518: JWA](https://datatracker.ietf.org/doc/html/rfc7518), [RFC 7517: JWK](https://datatracker.ietf.org/doc/html/rfc7517)
- [RFC 8725: JWT Best Current Practices](https://datatracker.ietf.org/doc/html/rfc8725)
- [PortSwigger: JWT attacks](https://portswigger.net/web-security/jwt)
- [OWASP JSON Web Token Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_Cheat_Sheet.html)
- [jwt.io debugger and library directory](https://jwt.io/)
- [panva/jose](https://github.com/panva/jose), [PyJWT](https://pyjwt.readthedocs.io/), [Nimbus JOSE+JWT](https://connect2id.com/products/nimbus-jose-jwt), [golang-jwt](https://github.com/golang-jwt/jwt)
- [Critical vulnerabilities in JSON Web Token libraries (Auth0, 2015)](https://auth0.com/blog/critical-vulnerabilities-in-json-web-token-libraries/)

---

## 14. JWKS (JSON Web Key Set)

A JWKS (RFC 7517) is a JSON document listing public keys that verifiers use to check JWT signatures. The IdP publishes it at the `jwks_uri` from its discovery document, so signing keys can rotate without redeploying every consumer.

### Core mechanism

```json
GET https://idp.example.com/.well-known/jwks.json

{
  "keys": [
    {
      "kty": "RSA",
      "use": "sig",
      "alg": "RS256",
      "kid": "2026-10-a",
      "n": "0vx7agoebGcQSuuPiLJXZptN9nndrQmbXEps2aiAFbWhM78LhWx4cbbfAAtVT86zwu1RK7aPFFxuhDR1L6tSoc_BJECPebWKRXjBZCiFV4n3oknjhMstn64tZ_2W-5JsGY4Hc5n9yBXArwl93lqt7_RN5w6Cf0h4QyQ5v-65YGjQR0_FDW2QvzqY368QQMicAtaSqzs8KJZgnYb9c7d0zgdAZHzu6qMQvRL5hajrn1n91CbOpbISD08qNLyrdkt-bFTWhAI4vMQFh6WeZu0fM4lFd2NcRwr3XPksINHaQ-G_xBniIqbw0Ls1jF44-csFCur-kEgU8awapJzKnqDKgw",
      "e": "AQAB"
    },
    { "kty": "EC", "use": "sig", "alg": "ES256", "kid": "2026-10-ec", "crv": "P-256", "x": "f83OJ3D2xF1Bg8vub9tLe1gHMzV76e8Tus9uPHvRVEU", "y": "x_FEzRu9m36HLN_tue659LNpXW6pCyStikYjKIWI5a0" }
  ]
}
```

Fields: `kty` (key type), `use` (`sig`/`enc`), `alg`, `kid` (key ID that JWT headers reference), RSA `n` (modulus) and `e` (exponent) as base64url big-endian integers; EC `crv`, `x`, `y`. Optional `x5c`/`x5t#S256` for certificate chains.

Selection logic at the verifier: read `kid` from the JWT header -> find the key with the same `kid` (and compatible `kty`/`alg`) -> verify.

### Engineering blueprint

**Key rotation (zero downtime):**

```text
T0    JWKS = [K1]                      signing with K1
T1    JWKS = [K1, K2]                  publish K2 FIRST; consumers' caches pick it up
T2    sign with K2                     (wait >= max JWKS cache TTL after T1)
T3    JWKS = [K1, K2]                  K1 still published: tokens it signed live until exp
T4    JWKS = [K2]                      remove K1 after (max token lifetime + cache TTL)
```

**Verifier caching rules:**

- Fetch from the `jwks_uri` obtained from discovery over HTTPS; never from a URL in the token.
- Cache with HTTP `Cache-Control` honoring (typically 5 to 60 minutes), keep last-good copy on fetch failure.
- On unknown `kid`: refetch once, but **rate-limit** (e.g. at most one forced refresh per 30 s), otherwise an attacker sending random `kid`s turns your service into a DoS tool against the IdP (and vice versa).
- Support multiple keys with the same `alg` and select strictly by `kid`; if `kid` is absent and the set has one matching key, use it, else reject.

```ts
// jose does cache + cooldown for you
const JWKS = createRemoteJWKSet(new URL(jwksUri), {
  cooldownDuration: 30_000,   // min gap between refetches triggered by unknown kid
  cacheMaxAge: 10 * 60_000,
  timeoutDuration: 5_000,
});
```

Generating and serving your own keys (when you are the issuer):

```bash
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:3072 -out sign-2026-10-a.pem
openssl rsa -in sign-2026-10-a.pem -pubout -out sign-2026-10-a.pub.pem
# derive kid as the RFC 7638 JWK thumbprint (SHA-256 over canonical JSON) so it is stable and collision-free
```

Keep private keys in KMS/HSM (AWS KMS, GCP KMS, Vault Transit); sign via API; only the public JWK leaves.

### Production pitfalls

- **Trusting `jku`/`x5u`/`jwk` headers** in the token: attacker points to their own key set (SSRF + forgery).
- Fetching the JWKS on every request (latency, IdP rate limits, outage amplification).
- Never refreshing the cache: after a rotation all tokens fail until restart. Or refreshing without cooldown => thundering herd.
- Removing the old key immediately at rotation: all in-flight tokens fail.
- No `kid` in tokens or duplicated `kid`s across different key types.
- Accepting a key with `use: "enc"` for signature verification, or an `alg` that doesn't match.
- Fetching the JWKS over HTTP, or following redirects to internal hosts (SSRF). Restrict outbound destinations.
- Multi-issuer APIs: one shared "key by kid" map across issuers lets issuer A's key validate issuer B's claimed tokens. Namespace keys by issuer.
- Treating JWKS endpoint failure as "skip verification".

### SDE best practices

- Use the library's remote-JWKS helper (cache, cooldown, timeouts, `kid` matching) rather than a custom fetcher.
- Pin: `iss` allow-list -> discovery -> `jwks_uri`. Cache discovery too.
- Rotate signing keys on a schedule (e.g. every 30 to 90 days) and on suspicion of compromise; practice the emergency path (remove key immediately, accept mass-failure and force re-login).
- Monitor: JWKS fetch failures, unknown-`kid` rate, signature failures. Alert on spikes.
- Offline/air-gapped or high-availability services: bake last-known JWKS into a config store with periodic refresh.
- Expose your JWKS through a CDN with correct `Cache-Control` so it's fast and highly available.

### Resources

- [RFC 7517: JSON Web Key (JWK)](https://datatracker.ietf.org/doc/html/rfc7517)
- [RFC 7638: JWK Thumbprint](https://datatracker.ietf.org/doc/html/rfc7638)
- [RFC 8725 section 3.10: Do not trust received `jku`/`x5u`](https://datatracker.ietf.org/doc/html/rfc8725#section-3.10)
- [Auth0: JSON Web Key Sets](https://auth0.com/docs/secure/tokens/json-web-tokens/json-web-key-sets)
- [Auth0: Rotate signing keys](https://auth0.com/docs/get-started/tenant-settings/signing-keys/rotate-signing-keys)
- [mkjwk.org: JWK generator](https://mkjwk.org/) (for learning; do not generate production keys in a web page)
- [jose: createRemoteJWKSet](https://github.com/panva/jose/blob/main/docs/jwks/remote/functions/createRemoteJWKSet.md)

---

## 15. RS256

`RS256` is the JOSE algorithm identifier for **RSASSA-PKCS1-v1_5 using SHA-256** (RFC 7518 section 3.3). The issuer signs with an RSA private key; any holder of the public key can verify but not forge. It is the interoperability default for OIDC (the spec says `RS256` MUST be supported by OPs).

### Core mechanism

Signing `M = BASE64URL(header) || "." || BASE64URL(payload)`:

1. `H = SHA-256(M)` (32 bytes).
2. Build the encoded message (EMSA-PKCS1-v1_5), length = modulus length `k`:

```text
EM = 0x00 || 0x01 || PS (0xFF repeated, >= 8 bytes) || 0x00 || DigestInfo
DigestInfo (SHA-256) = 30 31 30 0d 06 09 60 86 48 01 65 03 04 02 01 05 00 04 20 || H
```

3. Signature `s = EM^d mod n` (private exponent `d`). Output length equals the modulus size: 256 bytes for a 2048-bit key; base64url-encoded it becomes the third JWT segment.
4. Verify: `EM' = s^e mod n` (usually `e = 65537`), re-encode `H` and compare to `EM'` in full. Padding-check shortcuts caused historical forgeries (Bleichenbacher '06, small `e`); use a maintained library.

Properties: **deterministic** (same input => same signature, unlike PSS or ECDSA), verification is fast (public exponent small), signing is slow (big modular exponentiation), keys and signatures are big.

### Engineering blueprint

| Alg | Type | Sig size | Sign speed | Verify speed | Notes |
| --- | --- | --- | --- | --- | --- |
| HS256 | HMAC-SHA256, shared secret | 32 B | very fast | very fast | verifier can also forge; only if issuer == verifier |
| **RS256** | RSA-PKCS1v1.5 + SHA-256 | 256 B (2048-bit) | slow | fast | universal support |
| PS256 | RSA-PSS + SHA-256 | 256 B | slow | fast | randomized padding, provable security; supported less widely |
| ES256 | ECDSA P-256 + SHA-256 | 64 B | fast | medium | signature is raw `r||s`, **not** DER; needs good RNG per signature |
| EdDSA | Ed25519 | 64 B | very fast | fast | deterministic, smaller; verify library/IdP support |

Parameters:

- Minimum modulus **2048 bits** (RFC 7518 requires it for RS256). 3072 for long-lived or high-assurance keys.
- Public exponent 65537.
- Key per environment; key per purpose (don't sign ID tokens and webhooks with the same key).
- Private key custody: KMS/HSM; sign via API; rotate (see JWKS).

OpenSSL sanity check of a JWT signature (illustrative):

```bash
printf '%s' "$HEADER.$PAYLOAD" | openssl dgst -sha256 -verify pub.pem -signature <(echo -n "$SIG_B64URL" | tr '_-' '/+' | base64 -d)
```

### Production pitfalls

- **Algorithm confusion:** verifier lets the token header choose `HS256` and feeds the RSA public key (PEM text) as the HMAC secret. Defense: fix `algorithms: ['RS256']` per issuer/key.
- Accepting RSA keys < 2048 bits from a JWKS or config.
- Mixing up RS256 (signature) with RSA-OAEP / RSA1_5 (JWE key encryption). `RSA1_5` is vulnerable to padding oracle attacks (Bleichenbacher); use `RSA-OAEP`.
- Signing performance at scale (each sign ~1 ms+ for 2048-bit); fine for IdPs, but if you mint a JWT per request internally, consider ES256/EdDSA or caching.
- Converting ECDSA DER<->raw `r||s` incorrectly when moving from a KMS (which returns DER) to JOSE (raw). A classic cause of "invalid signature" bugs.
- Leaking the private key via repos/CI logs/container images; sharing one key across staging and prod.
- Library defaults that are lenient about padding or accept `e=3` keys.

### SDE best practices

- Default to RS256 for third-party interoperability; choose ES256 or EdDSA for internal systems where you control all verifiers and want small, fast tokens.
- Hard-code the expected algorithm(s) in the verifier; reject everything else, including `none` and `HS*` on endpoints expecting asymmetric keys.
- Use KMS-backed signing and set `kid` to the JWK thumbprint (RFC 7638).
- Keep a rotation runbook and test it quarterly; keep the previous key published for the whole max-token-TTL window.
- Prefer verified, FIPS-validated or widely audited crypto (OpenSSL, BoringSSL, Go `crypto/rsa`, Java JCA); never implement RSA padding yourself.

### Resources

- [RFC 7518 section 3.3: RSASSA-PKCS1-v1_5 (RS256)](https://datatracker.ietf.org/doc/html/rfc7518#section-3.3)
- [RFC 8017: PKCS #1 v2.2 (RSASSA-PKCS1-v1_5, PSS, OAEP)](https://datatracker.ietf.org/doc/html/rfc8017)
- [RFC 8037: EdDSA in JOSE](https://datatracker.ietf.org/doc/html/rfc8037)
- [RFC 8725 section 3.1-3.2: Algorithm verification](https://datatracker.ietf.org/doc/html/rfc8725#section-3.1)
- [NIST SP 800-57 Part 1: Key Management](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final) and [SP 800-131A: Transitioning crypto algorithms](https://csrc.nist.gov/pubs/sp/800/131/a/r2/final)
- [PortSwigger: Algorithm confusion attacks](https://portswigger.net/web-security/jwt/algorithm-confusion)

---

## 16. Cookies / HttpOnly / SameSite

Cookies are browser-managed key/value state attached automatically to matching requests. They are the standard carrier for session identifiers in browser apps because the browser can enforce attributes that JavaScript-readable storage cannot.

### Core mechanism

```http
Set-Cookie: __Host-sid=8f3c...e1; Path=/; Secure; HttpOnly; SameSite=Lax; Max-Age=3600
Cookie: __Host-sid=8f3c...e1
```

| Attribute | Effect | Security role |
| --- | --- | --- |
| `HttpOnly` | JS (`document.cookie`) cannot read it | stops token theft by XSS exfiltration (not XSS actions) |
| `Secure` | sent only over HTTPS | prevents network sniffing/downgrade |
| `SameSite=Strict` | never sent on cross-site requests | strongest CSRF defense; breaks inbound deep-link logins |
| `SameSite=Lax` | sent on top-level GET navigations only | CSRF defense for state-changing methods; browsers default to this when unset |
| `SameSite=None; Secure` | always sent (cross-site) | needed for embeds/third-party; opens CSRF back up |
| `Domain=` | when set, includes all subdomains | widens exposure; **omit** for host-only cookie |
| `Path=` | URL path scope | not a security boundary (same origin can read via iframe) |
| `Max-Age` / `Expires` | persistence | session cookie if omitted |
| `Partitioned` (CHIPS) | cookie keyed by top-level site | embedded third-party cookies without cross-site tracking |
| `__Host-` prefix | must be `Secure`, `Path=/`, no `Domain` | host-locked: subdomains/other paths cannot overwrite (defeats cookie tossing) |
| `__Secure-` prefix | must be `Secure` | weaker prefix |

"Site" = scheme + registrable domain (eTLD+1); "origin" = scheme + host + port. `app.example.com` and `evil.example.com` are **same-site**, so SameSite does not stop CSRF between sibling subdomains.

### Engineering blueprint

Session cookie recipe for a first-party web app:

```http
Set-Cookie: __Host-sid=<128-bit random>; Path=/; Secure; HttpOnly; SameSite=Lax; Max-Age=28800
```

CSRF defense in depth (cookie auth is ambient authority):

1. `SameSite=Lax` (or Strict) on the session cookie.
2. Verify `Origin` (fallback `Referer`) against an allow-list on all state-changing requests; use `Sec-Fetch-Site: same-origin | none` when present.
3. Anti-CSRF token for non-idempotent requests: synchronizer token tied to the session, or HMAC double-submit (`HMAC(secret, sessionId || random)` in a cookie and header; **signed**, never a naive unsigned double-submit).
4. Never mutate on GET. Require `Content-Type: application/json` or a custom header (forces CORS preflight) for APIs.

CORS with cookies:

```http
Access-Control-Allow-Origin: https://app.example.com     // exact origin, never "*"
Access-Control-Allow-Credentials: true
Vary: Origin
```

Client must opt in: `fetch(url, { credentials: 'include' })`.

Express example:

```ts
app.use(session({
  name: '__Host-sid', secret: process.env.SESSION_SECRET, resave: false, saveUninitialized: false,
  cookie: { httpOnly: true, secure: true, sameSite: 'lax', maxAge: 8 * 3600_000, path: '/' },
  store: new RedisStore({ client }),
}));
```

### Production pitfalls

- **`HttpOnly` != XSS-proof.** XSS can still issue authenticated requests from the victim's browser. CSP and output encoding remain mandatory.
- `SameSite=None` added "to fix a cross-site problem" without CSRF tokens.
- `Domain=.example.com` makes cookies readable/overwritable by every subdomain (including forgotten or takeover-able ones). Cookie tossing and session fixation via sibling subdomains.
- Missing `Secure` in dev copied to prod; mixed content.
- Expecting `SameSite=Lax` to protect `GET` endpoints that mutate state, or the 2-minute Lax+POST allowance Chrome applies to cookies with no explicit `SameSite`.
- Safari/Firefox third-party-cookie blocking and ITP breaking cross-site embedded login or SSO iframes; do not build on third-party cookies.
- Large cookies (>4 KB) silently dropped; JWT-in-cookie that grows with roles.
- CORS misconfig: reflecting arbitrary `Origin` with `Allow-Credentials: true` => any site reads authenticated responses.
- Putting the access token in a non-HttpOnly cookie (JS must read it) => same exposure as localStorage, plus auto-sent (CSRF).
- Logout that clears the cookie client-side only; server session still valid.

### SDE best practices

- Browser app: keep tokens server-side (BFF), give the browser an opaque `__Host-` session cookie with `HttpOnly; Secure; SameSite=Lax`.
- Short-lived transaction cookies (login state/nonce) use `SameSite=Lax`, path-scoped, `Max-Age` of minutes.
- Sign or encrypt any cookie whose content you read back (AEAD, e.g. AES-GCM / XChaCha20-Poly1305); rotate keys with `kid`.
- Split concerns: auth cookie (opaque, HttpOnly) vs UI-preference cookies (JS-readable, harmless).
- Send `Cache-Control: no-store` on authenticated responses; add `Vary: Cookie` where applicable.
- Add CSP (`default-src 'self'`, nonces), `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `Strict-Transport-Security`.
- Native/mobile/API clients: don't use cookies; use bearer or DPoP tokens in the OS secure store (Keychain, Keystore).

### Resources

- [MDN: Set-Cookie header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie)
- [MDN: Using HTTP cookies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies)
- [RFC 6265bis (Cookies, SameSite, prefixes) IETF draft](https://datatracker.ietf.org/doc/draft-ietf-httpbis-rfc6265bis/) / [RFC 6265](https://datatracker.ietf.org/doc/html/rfc6265)
- [OWASP CSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
- [OWASP Session Management Cheat Sheet: Cookies](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html#cookies)
- [web.dev: SameSite cookies explained](https://web.dev/articles/samesite-cookies-explained) and [Understanding "same-site" and "same-origin"](https://web.dev/articles/same-site-same-origin)
- [MDN: CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)
- [MDN: CHIPS / Partitioned cookies](https://developer.mozilla.org/en-US/docs/Web/Privacy/Privacy_sandbox/Partitioned_cookies)
- [OWASP XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)

---

## 17. Sessions

A session is server-recognized state tying a series of requests to one authenticated principal. Two models: **server-side sessions** (opaque ID -> state in a store) and **stateless sessions** (state in a signed/encrypted token the client carries). In OIDC deployments there are really *two* sessions: the IdP's SSO session and each application's own.

### Core mechanism

```text
Server-side session:
  cookie: __Host-sid = <opaque 128-bit ID>
  store  : sess:sha256(ID) -> { userId, tenantId, authTime, amr, createdAt, lastSeen, ip, ua, csrf, tokens(enc) }
  request: cookie -> lookup -> context  (O(1) Redis GET)

Stateless token session:
  cookie/header: JWT { sub, sid, exp, ... }
  request: verify signature -> context  (no store lookup, but no instant revoke)
```

| | Server-side | Stateless (JWT) |
| --- | --- | --- |
| Revoke / logout / kick | immediate | only at expiry (or denylist, which re-adds state) |
| Scale | needs shared store (Redis/DB) | no lookup |
| Size on wire | tiny | grows with claims |
| Sensitive state | stays on server | exposed (or must be encrypted) |
| Best for | browser apps, admin consoles | service-to-service, short-lived API access |

### Engineering blueprint

Lifecycle rules:

- **Create** after successful authentication; ID from CSPRNG, >= 128 bits (OWASP minimum 64 bits of entropy). Store only a hash of the ID server-side so a store leak cannot be replayed.
- **Rotate the ID** on login, privilege elevation (MFA step-up), and password change. Prevents session fixation.
- **Timeouts:** idle (e.g. 15 to 30 min for sensitive apps, up to days for consumer) and absolute (e.g. 8 to 24 h for workforce apps, 30 to 90 days for "remember me"). Enforce both on the server.
- **Invalidate** on logout (delete server record), password/MFA change, role change, account disable. "Log out all devices" = delete by `userId`.
- **Bind softly** to context (user agent hash, coarse IP/ASN) for anomaly signals; don't hard-bind to IP (mobile networks change).

Redis layout:

```text
SET   sess:{sha256(id)}   <json>   EX 28800
SADD  user_sessions:{userId}  {sha256(id)}      # enables "logout everywhere" and an active-sessions UI
```

SSO logout topology:

```text
User clicks logout in App A
  -> App A deletes its session
  -> redirect to IdP end_session_endpoint (id_token_hint, post_logout_redirect_uri)
  -> IdP ends SSO session, sends back-channel logout_token (sid) to Apps B, C
  -> Apps B, C look up sid -> their session -> delete
```

### Production pitfalls

- **Session fixation**: accepting an attacker-supplied ID and keeping it after login.
- Session IDs in URLs (`;jsessionid=`, `?sid=`): leak via `Referer`, logs, sharing.
- Predictable IDs (sequential, timestamp, `Math.random`, UUIDv1/v7 used as secret).
- Sliding expiration with no absolute cap: a stolen session stays alive forever with periodic use.
- In-memory sessions behind a load balancer with no sticky routing, or lost on deploy.
- Redis without persistence or TTL discipline; unbounded growth from anonymous sessions (create sessions only when needed).
- Logout that doesn't propagate to the IdP, other apps, or refresh tokens.
- Storing authorization decisions in the session forever (stale roles). Reload roles/permissions periodically or on change events.
- Cross-subdomain session sharing with `Domain=` cookies, widening blast radius.
- Session data containing raw OAuth tokens in plaintext in Redis; encrypt with an app key (envelope encryption).

### SDE best practices

- Default to **server-side sessions behind an opaque `__Host-` cookie** for browser clients ([section 16](#16-cookies--httponly--samesite)); use short-lived JWT access tokens for APIs.
- Store sessions in Redis/Valkey with TTL = absolute timeout; add a secondary index per user.
- Keep session payload small: IDs and timestamps, not full profiles; fetch profile from DB with caching.
- Re-authenticate (or step-up MFA) for sensitive operations, tracking `authTime` in the session.
- Emit security events on creation, rotation, invalidation, and anomalous reuse (new geo/UA). Offer users a device/session list.
- For stateless needs, combine: short access JWT (10 min) + server-side refresh/session record = revocation within one TTL.
- Test: fixation (set cookie pre-login, ensure changes post-login), concurrent logout, expiry boundaries, replay of old IDs after rotation.

### Resources

- [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
- [OWASP Session Fixation](https://owasp.org/www-community/attacks/Session_fixation)
- [OIDC Session Management 1.0](https://openid.net/specs/openid-connect-session-1_0.html) and [Back-Channel Logout 1.0](https://openid.net/specs/openid-connect-backchannel-1_0.html)
- [OIDC Front-Channel Logout 1.0](https://openid.net/specs/openid-connect-frontchannel-1_0.html)
- [NIST SP 800-63B section 5: Session Management](https://pages.nist.gov/800-63-4/sp800-63b.html)
- [Redis: Key expiration](https://redis.io/docs/latest/commands/expire/)
- [Stop using JWTs as session tokens (joepie91, widely cited critique)](http://cryto.net/~joepie91/blog/2016/06/13/stop-using-jwt-for-sessions/)

---

## 18. RBAC (Role-Based Access Control)

RBAC grants permissions to **roles**, and roles to **users**, so access is managed by job function instead of per-user lists. Formalized by NIST/ANSI INCITS 359 (Sandhu et al.). It is the default authorization model for business applications.

### Core mechanism

```text
 User ──< UserRole >── Role ──< RolePermission >── Permission (action on resource type)
                          │
                          └── (optional) parent Role  -> inheritance (hierarchical RBAC)
```

NIST levels:

1. **Core RBAC:** users, roles, permissions, sessions (active role subset).
2. **Hierarchical RBAC:** `admin` inherits `editor` inherits `viewer`.
3. **Constrained RBAC:** Separation of Duty. *Static* SoD (cannot hold both `requester` and `approver`), *dynamic* SoD (cannot activate both in one session).

Permission = `(resource_type, action)` e.g. `invoice:approve`. Check **permissions**, never role names, in code.

### Engineering blueprint

Relational schema with tenant scoping:

```sql
CREATE TABLE roles (
  id uuid PRIMARY KEY, tenant_id uuid,              -- NULL tenant = built-in/system role
  name text NOT NULL, UNIQUE (tenant_id, name)
);
CREATE TABLE permissions (id text PRIMARY KEY);     -- 'invoice:read', 'invoice:approve', ...
CREATE TABLE role_permissions (
  role_id uuid REFERENCES roles(id), permission_id text REFERENCES permissions(id),
  PRIMARY KEY (role_id, permission_id)
);
CREATE TABLE user_roles (
  user_id uuid, tenant_id uuid, role_id uuid REFERENCES roles(id),
  granted_by uuid, granted_at timestamptz DEFAULT now(), expires_at timestamptz,
  PRIMARY KEY (user_id, tenant_id, role_id)
);
```

Resolution and enforcement:

```ts
async function permissionsFor(userId: string, tenantId: string): Promise<Set<string>> {
  // cache key includes a per-user "authz version" bumped on any role change
  return cache.getOrLoad(`perm:${tenantId}:${userId}:${await authzVersion(userId)}`, 60, () =>
    db.query(`SELECT DISTINCT rp.permission_id FROM user_roles ur
              JOIN role_permissions rp USING (role_id)
              WHERE ur.user_id=$1 AND ur.tenant_id=$2 AND (ur.expires_at IS NULL OR ur.expires_at>now())`,
             [userId, tenantId]).then(r => new Set(r.rows.map(x => x.permission_id))));
}

export const requires = (perm: string) => async (req, res, next) =>
  (await permissionsFor(req.auth.sub, req.auth.tenant)).has(perm) ? next() : res.sendStatus(403);
```

Where roles live:

| Option | Pros | Cons |
| --- | --- | --- |
| Roles in JWT claim | no lookup, fast | stale until expiry, token bloat, revocation lag |
| Roles from IdP groups | single source for workforce | app-specific roles leak into IdP; mapping drift |
| Roles in app DB (recommended) | instant change, tenant-specific, auditable | one lookup (cacheable) |
| Policy engine (Casbin/OPA) | centralized, testable | another component to operate |

### Production pitfalls

- **Role explosion**: `editor-eu-finance-readonly-contractor`. Roles can't express "owner of this record" or "during business hours"; add attributes/relationships instead of more roles.
- Code checking `role == 'admin'` everywhere; changing the model means touching hundreds of call sites.
- RBAC alone leaves **BOLA**: `invoice:read` on all invoices of the tenant is not "this invoice belongs to this user's team".
- Privilege creep: users accumulate roles; no review or expiry.
- Self-service escalation: endpoint that assigns roles doesn't check that the grantor has the permission being granted (or any `role:assign`).
- Last-admin lockout and orphaned resources when a user is deleted.
- Cached permissions not invalidated on revoke.
- Global roles used in multi-tenant systems: admin of tenant A treated as admin of tenant B.
- Default role on signup too powerful; "super admin" backdoor flags left in prod.

### SDE best practices

- Model permissions as the stable public vocabulary of your API (`resource:action`), map roles -> permissions in data, ship built-in roles (`owner`, `admin`, `member`, `viewer`) and allow custom roles per tenant if the product needs it.
- Deny by default; require an explicit permission on every route/handler; fail CI if a route has none.
- Add object-level checks (ownership, team, state) beside RBAC; consider ABAC/ReBAC when roles can't express the rule.
- Enforce SoD for money/approvals; log who granted what, when, and why. Support expiring grants (JIT access) and periodic access reviews.
- Tenant-scoped role assignment; no cross-tenant inheritance.
- Version the cache key with an "authz epoch" per user/tenant, bumped on role changes; short TTL as a backstop.
- Test with a role x endpoint matrix generated from your route table.

### Resources

- [NIST: Role Based Access Control project](https://csrc.nist.gov/projects/role-based-access-control)
- [OWASP Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)
- [NIST SP 800-162: ABAC (for comparison)](https://csrc.nist.gov/pubs/sp/800/162/upd2/final)
- [Casbin RBAC docs](https://casbin.org/docs/rbac)
- [Kubernetes RBAC reference (a well-designed real-world RBAC)](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

---

## 19. Casbin

Casbin is an open-source authorization library (Go, Node.js, Java, Python, Rust, .NET, PHP and more) that evaluates access requests against a **model** (a configuration of the access-control logic) and a **policy** (the data). Changing from ACL to RBAC to ABAC means editing the model file, not the code. It runs **in-process** (library, not a server), with optional distributed policy sync.

### Core mechanism: the PERM metamodel

`Policy, Effect, Request, Matchers`:

| Section | Purpose |
| --- | --- |
| `[request_definition]` | shape of the question: `r = sub, dom, obj, act` |
| `[policy_definition]` | shape of a policy rule: `p = sub, dom, obj, act` |
| `[role_definition]` | grouping relations: `g = _, _, _` (user, role, domain) |
| `[policy_effect]` | how rule results combine: `some(where (p.eft == allow))` |
| `[matchers]` | boolean expression comparing request to each policy rule |

Evaluation: for the request, iterate policy lines, evaluate the matcher against each, collect `p.eft` (default `allow`), and combine per the effect expression. Role resolution (`g`) is a graph traversal (transitive, cycle-safe, optionally depth-limited).

### Engineering blueprint

**model.conf**, multi-tenant RBAC with domains (the usual SaaS shape):

```ini
[request_definition]
r = sub, dom, obj, act

[policy_definition]
p = sub, dom, obj, act, eft

[role_definition]
g = _, _, _

[policy_effect]
e = some(where (p.eft == allow)) && !some(where (p.eft == deny))

[matchers]
m = g(r.sub, p.sub, r.dom) && r.dom == p.dom && keyMatch2(r.obj, p.obj) && (r.act == p.act || p.act == "*")
```

**policy.csv**:

```csv
p, admin,  tenant_42, /invoices,     *,     allow
p, member, tenant_42, /invoices,     read,  allow
p, member, tenant_42, /invoices/:id, read,  allow
p, member, tenant_42, /invoices/:id, delete, deny

g, alice, admin,  tenant_42
g, bob,   member, tenant_42
g, bob,   member, tenant_77
```

**Go usage:**

```go
e, _ := casbin.NewSyncedEnforcer("model.conf", gormadapter.NewAdapterByDB(db)) // policy in Postgres
e.EnableAutoSave(true)
e.StartAutoLoadPolicy(30 * time.Second)           // or use a watcher for push-based sync

ok, _ := e.Enforce(claims.Sub, claims.Tenant, r.URL.Path, strings.ToLower(r.Method))
if !ok { http.Error(w, "forbidden", http.StatusForbidden); return }

e.AddRoleForUserInDomain("carol", "member", "tenant_42")      // admin API
e.AddPolicy("member", "tenant_42", "/reports", "read", "allow")
```

**Node.js:**

```ts
import { newEnforcer } from 'casbin';
import TypeORMAdapter from 'typeorm-adapter';
const e = await newEnforcer('model.conf', await TypeORMAdapter.newAdapter(typeormOpts));
const allowed = await e.enforce(sub, tenant, obj, act);
```

Useful matcher functions: `keyMatch`, `keyMatch2` (`/foo/:id`), `keyMatch3` (`/foo/{id}`), `regexMatch`, `ipMatch`, `globMatch`. ABAC variant: `m = r.sub.Owner == r.obj.Owner && r.act == "write"` passing structs/objects as `sub`/`obj`.

Operational pieces:

| Component | Role |
| --- | --- |
| **Adapter** | persists policies (file, Gorm/SQL, MongoDB, Redis, S3, ...). Load/Save/Add/Remove |
| **Watcher** | notifies other instances to reload after a change (Redis pub/sub, etcd, Kafka, NATS) |
| **Dispatcher** | cluster-wide consistent policy updates (newer feature) |
| **Role manager** | resolves `g` links; custom ones for LDAP/DB-backed hierarchies |
| **Online editor** | casbin.org/editor for modelling and testing a model + policy + request |

### Production pitfalls

- **Stale in-memory policy** across replicas: instance A updates, B keeps old policy until reload. Add a watcher (or short auto-reload) and understand the consistency window.
- Concurrency: the plain `Enforcer` is not safe for concurrent modification in Go; use `SyncedEnforcer`.
- Matcher built from user input (`e.Enforce` with matcher string composed at runtime): expression injection. Keep matchers static in `model.conf`.
- Unsanitized values inserted into the CSV/DB policy store (commas, wildcards): `Enforce("*"...)`-style surprises. Validate subject/object identifiers.
- Wrong effect for deny rules (`some(where allow)` alone ignores `deny`). Choose `allow-and-deny` or `priority` explicitly and test.
- Missing domain in the matcher: roles from tenant A grant access in tenant B (cross-tenant bug). Always include `r.dom == p.dom` and use `g(..., dom)`.
- Full policy set loaded per instance: memory/latency blow-up with millions of rules (per-user policies). Prefer role-level policies plus `g` links; use filtered loading (`LoadFilteredPolicy`) per tenant.
- `Enforce` is only RBAC logic; it does **not** verify the identity. `sub` must come from a validated token/session, never from request parameters.
- Treating Casbin as the sole control: still need DB-level tenant filtering/RLS and object-level checks.

### SDE best practices

- Pick the smallest model that expresses your rules: RBAC with domains for SaaS; add ABAC attributes only where roles fall short.
- Policies as data in Postgres via a SQL adapter, with migrations and an audit table; changes through an admin API that itself requires permission.
- Add a watcher (Redis pub/sub) in multi-instance deployments; filtered loading per tenant for large installations; in-process cache of `Enforce` results with short TTL if hot.
- Wrap in a thin interface (`Authorizer.Can(ctx, subject, tenant, action, resource)`) so you can swap in OPA/Cedar/OpenFGA if relationships (ReBAC) become central.
- Unit-test policies: a table of `(sub, dom, obj, act) -> expected` run in CI against the real `model.conf` and a fixture policy; use `EnforceEx` to explain decisions in logs.
- Map HTTP to `(obj, act)` deliberately: `obj` = route template (`/invoices/:id`), `act` = verb or domain action (`approve`), not raw URLs with IDs.

### Resources

- [Casbin documentation: overview](https://casbin.org/docs/overview)
- [Syntax for Models (PERM, matchers, effects)](https://casbin.org/docs/syntax-for-models)
- [RBAC with domains/tenants](https://casbin.org/docs/rbac-with-domains)
- [Adapters](https://casbin.org/docs/adapters) / [Watchers](https://casbin.org/docs/watchers) / [Function reference](https://casbin.org/docs/function)
- [Casbin online editor](https://casbin.org/editor)
- Source: [casbin/casbin (Go)](https://github.com/casbin/casbin), [node-casbin](https://github.com/casbin/node-casbin), [jCasbin](https://github.com/casbin/jcasbin), [PyCasbin](https://github.com/casbin/pycasbin)
- Alternatives: [OPA/Rego](https://www.openpolicyagent.org/), [Cedar](https://www.cedarpolicy.com/), [OpenFGA](https://openfga.dev/), [SpiceDB](https://authzed.com/docs)

---

## 20. Tenant Isolation

Tenant isolation guarantees that in a multi-tenant system one customer's data, compute, and behavior cannot be read, modified, or degraded by another. It is the central security property of SaaS; a single missing `WHERE tenant_id = ?` is a reportable breach.

### Core mechanism

Isolation models (data tier):

| Model | How | Isolation | Cost / ops | Typical use |
| --- | --- | --- | --- | --- |
| **Silo** | database (or cluster/account) per tenant | strongest, per-tenant keys, backup, region | highest; migrations x N | regulated, large enterprise |
| **Bridge** | schema per tenant in a shared DB | strong logical | medium; many schemas stress catalog | mid-market |
| **Pool** | shared tables with `tenant_id` | logical only (needs enforcement) | lowest; best density | long-tail SaaS |

Isolation must hold in **every** layer: auth (tenant identity), API routing, DB rows, cache keys, queues/topics, object storage prefixes, search indexes, background jobs, logs/metrics, encryption keys, rate limits (noisy neighbor), and secrets.

### Engineering blueprint

**1. Establish tenant context from a trusted source.** Derive it from the verified token/session (`org_id`/`tenant` claim, or session record), never from a client-controllable header, body, or path alone. If the URL has `/t/{tenant}/...`, assert it equals the token tenant. Users in many tenants: mint a token per active tenant (switching tenants = new token/session), so each token has exactly one tenant.

```ts
const tenant = req.auth.tenant;                            // from verified JWT
if (req.params.tenant && req.params.tenant !== tenant) return res.sendStatus(404);
req.ctx = { tenant, user: req.auth.sub };                  // propagate via AsyncLocalStorage / context.Context
```

**2. Enforce in the database with Postgres Row-Level Security** (defense behind your app code):

```sql
ALTER TABLE invoices ENABLE ROW LEVEL SECURITY;
ALTER TABLE invoices FORCE  ROW LEVEL SECURITY;      -- also applies to the table owner

CREATE POLICY tenant_isolation ON invoices
  USING      (tenant_id = current_setting('app.tenant_id', true)::uuid)   -- reads / update / delete visibility
  WITH CHECK (tenant_id = current_setting('app.tenant_id', true)::uuid);  -- inserts / updates must stay in tenant

-- app connects as a role that is NOT superuser, NOT BYPASSRLS, NOT table owner
CREATE ROLE app_user LOGIN NOBYPASSRLS;
GRANT SELECT, INSERT, UPDATE, DELETE ON invoices TO app_user;
```

Set the context **per transaction**:

```ts
await db.tx(async (t) => {
  await t.query(`SELECT set_config('app.tenant_id', $1, true)`, [ctx.tenant]);   // true = LOCAL to this transaction
  return handler(t);                                                              // all queries run under RLS
});
```

`SET LOCAL`/`set_config(..., true)` is safe with PgBouncer **transaction pooling**; session-level `SET` is not (value leaks to the next client on that connection). Missing setting returns NULL => policy matches no rows (fail closed, thanks to `current_setting(..., true)`).

**3. Schema hygiene for pool model:**

```sql
-- tenant_id first in indexes, in uniqueness, and in foreign keys so cross-tenant references are impossible
CREATE TABLE invoices (tenant_id uuid NOT NULL, id uuid NOT NULL, customer_id uuid NOT NULL, ...,
  PRIMARY KEY (tenant_id, id),
  FOREIGN KEY (tenant_id, customer_id) REFERENCES customers (tenant_id, id));
```

**4. Other layers:**

```text
cache key     : t:{tenant}:invoice:{id}
queue/topic   : message carries tenant_id; consumer sets tenant context before touching data
object store  : s3://bucket/{tenant}/...   + IAM policy / presigned URLs scoped to that prefix
search index  : filter on tenant_id (mandatory query filter) or index-per-tenant
encryption    : per-tenant data key (envelope encryption, KMS), enables crypto-shredding on offboarding
rate limits   : per-tenant quotas and concurrency pools (noisy neighbor)
logs          : include tenant_id; restrict support access by tenant, with audit
```

### Production pitfalls

- **Missing tenant filter** in one query (reports, exports, admin, raw SQL, ORM `.findById`) => cross-tenant read. ORM default scopes get bypassed by joins, raw queries, and aggregations.
- Trusting `X-Tenant-ID` / `tenant_id` from the request body. Tenant IDs are often guessable.
- Connection pool context bleed: setting tenant with session-level `SET` and returning the connection to the pool.
- RLS bypass: app connects as owner/superuser/`BYPASSRLS`; `FORCE ROW LEVEL SECURITY` missing; policies applied to `SELECT` only; views running with owner rights (use `security_invoker` views on PG 15+); `SECURITY DEFINER` functions.
- Background jobs/webhooks/cron running with no tenant context (or "all tenants") and reusing it for later work.
- Shared caches without tenant in the key; shared temp files; global singletons holding per-request data (async context lost across `await` in poorly written code).
- Global uniqueness (`UNIQUE(email)`) leaking existence across tenants or blocking legitimate duplicates.
- Support/admin impersonation without audit, or "super admin" tokens that skip tenant checks globally.
- Error messages and timing revealing other tenants' records (403 vs 404).
- Noisy neighbor: one tenant's heavy query/export starving others; no per-tenant limits.
- Search/analytics pipelines that copy data into un-isolated stores.
- Tenant deletion that leaves data in backups, caches, search, or object storage.

### SDE best practices

- **Defense in depth**: token tenant -> middleware context -> repository functions that *require* a tenant argument (no tenant-less query helper exists) -> RLS -> per-tenant encryption keys for sensitive data.
- Make the unsafe path hard: a typed `TenantScoped<Repo>`; lint rules that flag raw SQL lacking a tenant predicate; forbid direct DB access outside repositories.
- Use RLS on every tenant-scoped table; add a CI check that lists tables with a `tenant_id` column but no policy.
- Automated cross-tenant tests: create tenants A and B with data; for every endpoint, call as A with B's IDs and assert 404; run the suite on each PR. Add fuzzing with random IDs, and a canary tenant monitored for foreign access.
- Choose model by customer need, and plan the migration: pool by default, move whales or regulated customers to silo (routing layer maps tenant -> cluster/region).
- Quotas, rate limits, and DB statement timeouts per tenant; separate worker pools for heavy tenants.
- Offboarding: documented purge across DB, caches, search, object storage, backups (crypto-shred keys).
- Log and alert on tenant-context anomalies (token tenant != resource tenant; RLS policy denial counts).
- Combine with RBAC: roles are scoped `(user, tenant, role)`; permissions never cross tenants ([sections 18](#18-rbac-role-based-access-control), [19](#19-casbin)).

### Resources

- [PostgreSQL: Row Security Policies](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)
- [AWS SaaS Lens / SaaS Architecture Fundamentals: Tenant isolation](https://docs.aws.amazon.com/whitepapers/latest/saas-architecture-fundamentals/tenant-isolation.html)
- [AWS whitepaper: SaaS Tenant Isolation Strategies](https://docs.aws.amazon.com/whitepapers/latest/saas-tenant-isolation-strategies/saas-tenant-isolation-strategies.html)
- [Microsoft Azure Architecture Center: Multitenant architecture guidance](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/overview)
- [OWASP Multi-Tenant Application Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Multi_Tenant_Security_Cheat_Sheet.html)
- [Crunchy Data: Row Level Security for Tenants in Postgres](https://www.crunchydata.com/blog/row-level-security-for-tenants-in-postgres)
- [Supabase: Row Level Security guide](https://supabase.com/docs/guides/database/postgres/row-level-security)
- [OWASP API Security Top 10 2023: API1 BOLA](https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/)
- [Citus: multi-tenant data modeling](https://docs.citusdata.com/en/stable/sharding/data_modeling.html#multi-tenant-model)

---

## 21. Production Checklist & Cheat Sheet

### Recommended lifetimes and storage

| Artifact | Lifetime | Where it lives | Notes |
| --- | --- | --- | --- |
| Authorization code | <= 60 s (max 10 min) | AS only | single use; bound to PKCE challenge |
| `state` / `nonce` / `code_verifier` | login transaction (<= 10 min) | server session or short-lived signed cookie | delete on first use |
| ID token | minutes | validated then discarded | keep `id_token_hint` only if needed for logout |
| Access token | 5 to 15 min | server memory (BFF) / OS secure store (native) | never `localStorage` |
| Refresh token | idle 14 to 30 d, absolute 60 to 90 d | server-side, hashed in AS DB / encrypted at BFF | rotate every use, reuse detection |
| App session | idle 15 to 60 min, absolute 8 to 24 h (workforce) | Redis; cookie holds opaque ID | rotate ID on login and privilege change |
| Signing key | rotate every 30 to 90 d | KMS/HSM; public part in JWKS | overlap = max token TTL + cache TTL |
| Password-reset / email-verify token | <= 15 min | DB (hashed) | single use |

### Pre-launch checklist

**Authentication / OIDC**
- [ ] Authorization Code + PKCE (S256) everywhere; implicit and password grants disabled.
- [ ] `state` and `nonce` generated per attempt, bound to the browser session, verified, deleted on use.
- [ ] Exact-match redirect URIs; no wildcards; no open redirects in `next`/`returnTo`.
- [ ] ID token validated: signature, `alg` allow-list, `iss`, `aud`/`azp`, `exp`, `nonce`, `auth_time`/`acr` where relevant.
- [ ] Users keyed by `(iss, sub)`; `email_verified` honoured; no auto-linking by email.
- [ ] MFA available; phishing-resistant option (passkeys) for admins; step-up for sensitive actions.
- [ ] Rate limiting + lockout/backoff; generic error messages; breached-password screening if you store passwords.

**Tokens & keys**
- [ ] Access tokens: short TTL, `aud` pinned and checked, `typ` checked, scopes enforced.
- [ ] JWT verification pins algorithms; ignores `jku`/`jwk`/`x5u`; `kid` is a pure map lookup.
- [ ] JWKS cached with cooldown; discovery from trusted issuer allow-list; rotation runbook tested.
- [ ] Refresh tokens rotated, hashed at rest, family revocation on reuse, absolute lifetime set.
- [ ] No tokens in URLs, logs, or browser storage; `Authorization`/`Cookie` redacted in logs/APM.
- [ ] Sender-constraining (DPoP/mTLS) considered for high-value APIs.

**Cookies & sessions**
- [ ] Session cookie: `__Host-` prefix, `Secure; HttpOnly; SameSite=Lax` (or Strict), no `Domain`.
- [ ] CSRF: Origin/Fetch-Metadata checks + tokens on state-changing requests; no state change on GET.
- [ ] CORS: exact origin allow-list; credentials only where needed; no reflected `Origin`.
- [ ] Session ID rotation on login/privilege change; idle and absolute timeouts server-enforced.
- [ ] Logout kills server session, revokes refresh token, calls IdP logout; back-channel logout wired.
- [ ] CSP, HSTS, `nosniff`, `Referrer-Policy` set.

**Authorization & tenancy**
- [ ] Deny by default; every route declares a permission; CI fails on undeclared routes.
- [ ] Permissions (not role names) checked; effective access = permissions ∩ scopes ∩ object policy.
- [ ] Object-level checks in queries (`WHERE tenant_id AND id`); 404 on foreign objects.
- [ ] Tenant derived from verified token only; RLS enabled and `FORCE`d; app role lacks `BYPASSRLS`.
- [ ] Tenant in cache keys, queue messages, object paths, search filters, logs.
- [ ] Casbin/OPA policy sync tested across replicas; policy unit tests in CI.
- [ ] Role changes invalidate caches; grants audited; SoD enforced for sensitive flows.
- [ ] Cross-tenant and cross-user test suite runs on every PR.

### Common attack -> control map

| Attack | Primary control |
| --- | --- |
| Authorization code interception / injection | PKCE (S256), exact redirect URIs |
| Login CSRF | `state` bound to browser |
| ID token replay / injection | `nonce`, short `exp`, `aud` check |
| Token theft via XSS | BFF + HttpOnly cookie, CSP, short TTL, DPoP |
| Token replay from logs/network | TLS, short TTL, sender-constrained tokens, log redaction |
| JWT forgery (`none`, alg confusion, header keys) | pinned algorithms, trusted JWKS, ignore `jku`/`jwk` |
| Cross-API token reuse | `aud` + `typ` checks, resource indicators |
| Refresh token theft | rotation + reuse detection, server-side storage |
| Session fixation | rotate ID at login; `__Host-` cookie |
| CSRF on cookie sessions | SameSite + Origin check + CSRF token |
| BOLA / IDOR | object-level checks in the query, RLS |
| Privilege escalation via mass assignment | explicit allow-list DTOs; role assignment needs permission |
| Cross-tenant data access | tenant from token, RLS, tenant-scoped keys, tests |
| Account takeover via IdP email trust | key by `(iss, sub)`; require `email_verified`; manual linking |
| Mix-up attack | `iss` response parameter validation, per-AS redirect URIs |

### Quick glossary

| Term | One line |
| --- | --- |
| AS / OP | Authorization Server / OpenID Provider; issues tokens |
| RS | Resource Server; your API |
| RP / Client | Relying party; app that logs the user in |
| BFF | Backend-for-Frontend; server-side component that keeps tokens away from the browser |
| `sub` | stable user ID **within an issuer** |
| `aud` | who the token is for |
| `azp` | authorized party (client the ID token was issued to) |
| `jti` | unique token ID |
| `cnf` | confirmation claim; binds token to a key (DPoP `jkt`, mTLS `x5t#S256`) |
| `amr` / `acr` | how / how strongly the user authenticated |
| `kid` | key ID referencing a JWKS entry |
| PEP/PDP/PAP/PIP | enforcement / decision / administration / information points |
| BOLA / BFLA | broken object-level / function-level authorization |
| RLS | Postgres Row-Level Security |

---

## 22. Master Resource List

### Standards (IETF)

| Topic | Link |
| --- | --- |
| OAuth 2.0 core | [RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749) |
| Bearer tokens | [RFC 6750](https://datatracker.ietf.org/doc/html/rfc6750) |
| OAuth threat model | [RFC 6819](https://datatracker.ietf.org/doc/html/rfc6819) |
| Security BCP (current) | [RFC 9700](https://datatracker.ietf.org/doc/html/rfc9700) |
| OAuth 2.1 (draft) | [draft-ietf-oauth-v2-1](https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/) |
| PKCE | [RFC 7636](https://datatracker.ietf.org/doc/html/rfc7636) |
| Native apps | [RFC 8252](https://datatracker.ietf.org/doc/html/rfc8252) |
| Browser-based apps (draft) | [draft-ietf-oauth-browser-based-apps](https://datatracker.ietf.org/doc/draft-ietf-oauth-browser-based-apps/) |
| AS metadata | [RFC 8414](https://datatracker.ietf.org/doc/html/rfc8414) |
| Issuer identification | [RFC 9207](https://datatracker.ietf.org/doc/html/rfc9207) |
| Resource indicators | [RFC 8707](https://datatracker.ietf.org/doc/html/rfc8707) |
| Token introspection / revocation | [RFC 7662](https://datatracker.ietf.org/doc/html/rfc7662), [RFC 7009](https://datatracker.ietf.org/doc/html/rfc7009) |
| Token exchange | [RFC 8693](https://datatracker.ietf.org/doc/html/rfc8693) |
| JWT access token profile | [RFC 9068](https://datatracker.ietf.org/doc/html/rfc9068) |
| DPoP / mTLS | [RFC 9449](https://datatracker.ietf.org/doc/html/rfc9449), [RFC 8705](https://datatracker.ietf.org/doc/html/rfc8705) |
| PAR / JAR | [RFC 9126](https://datatracker.ietf.org/doc/html/rfc9126), [RFC 9101](https://datatracker.ietf.org/doc/html/rfc9101) |
| Device grant | [RFC 8628](https://datatracker.ietf.org/doc/html/rfc8628) |
| JWT / JWS / JWE / JWK / JWA | [7519](https://datatracker.ietf.org/doc/html/rfc7519), [7515](https://datatracker.ietf.org/doc/html/rfc7515), [7516](https://datatracker.ietf.org/doc/html/rfc7516), [7517](https://datatracker.ietf.org/doc/html/rfc7517), [7518](https://datatracker.ietf.org/doc/html/rfc7518) |
| JWT BCP | [RFC 8725](https://datatracker.ietf.org/doc/html/rfc8725) |
| JWK thumbprint | [RFC 7638](https://datatracker.ietf.org/doc/html/rfc7638) |
| EdDSA in JOSE | [RFC 8037](https://datatracker.ietf.org/doc/html/rfc8037) |
| PKCS #1 (RSA) | [RFC 8017](https://datatracker.ietf.org/doc/html/rfc8017) |
| Cookies | [RFC 6265](https://datatracker.ietf.org/doc/html/rfc6265), [6265bis draft](https://datatracker.ietf.org/doc/draft-ietf-httpbis-rfc6265bis/) |
| TOTP / HOTP | [RFC 6238](https://datatracker.ietf.org/doc/html/rfc6238), [RFC 4226](https://datatracker.ietf.org/doc/html/rfc4226) |
| SCIM | [RFC 7643](https://datatracker.ietf.org/doc/html/rfc7643), [RFC 7644](https://datatracker.ietf.org/doc/html/rfc7644) |

### OpenID Foundation & W3C

- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
- [OIDC Discovery 1.0](https://openid.net/specs/openid-connect-discovery-1_0.html)
- [OIDC RP-Initiated Logout](https://openid.net/specs/openid-connect-rpinitiated-1_0.html), [Back-Channel Logout](https://openid.net/specs/openid-connect-backchannel-1_0.html), [Front-Channel Logout](https://openid.net/specs/openid-connect-frontchannel-1_0.html), [Session Management](https://openid.net/specs/openid-connect-session-1_0.html)
- [FAPI 2.0 Security Profile](https://openid.net/specs/fapi-security-profile-2_0.html) (high-assurance OAuth)
- [W3C WebAuthn Level 3](https://www.w3.org/TR/webauthn-3/)

### OWASP & NIST

- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/): [Authentication](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html), [Authorization](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html), [Session Management](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html), [Password Storage](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html), [CSRF](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html), [OAuth 2.0](https://cheatsheetseries.owasp.org/cheatsheets/OAuth2_Cheat_Sheet.html), [JWT](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_Cheat_Sheet.html), [Multi-Tenant](https://cheatsheetseries.owasp.org/cheatsheets/Multi_Tenant_Security_Cheat_Sheet.html)
- [OWASP API Security Top 10 (2023)](https://owasp.org/API-Security/editions/2023/en/0x11-t10/)
- [OWASP ASVS (verification standard)](https://owasp.org/www-project-application-security-verification-standard/)
- [NIST SP 800-63-4 Digital Identity Guidelines](https://pages.nist.gov/800-63-4/)
- [NIST RBAC](https://csrc.nist.gov/projects/role-based-access-control), [SP 800-162 ABAC](https://csrc.nist.gov/pubs/sp/800/162/upd2/final), [SP 800-57 Key Management](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)

### Learning sites & tooling

- [OAuth.com (Aaron Parecki)](https://www.oauth.com/), [OAuth 2.0 Playground](https://www.oauth.com/playground/)
- [Google OAuth 2.0 Playground](https://developers.google.com/oauthplayground/)
- [jwt.io](https://jwt.io/), [PortSwigger Web Security Academy: JWT](https://portswigger.net/web-security/jwt), [OAuth](https://portswigger.net/web-security/oauth)
- [Casbin docs](https://casbin.org/docs/overview), [Casbin editor](https://casbin.org/editor)
- [OpenID Foundation certification](https://openid.net/certification/)
- Libraries: [panva/jose](https://github.com/panva/jose), [panva/oauth4webapi](https://github.com/panva/oauth4webapi), [coreos/go-oidc](https://github.com/coreos/go-oidc), [Authlib](https://docs.authlib.org/), [Nimbus JOSE+JWT](https://connect2id.com/products/nimbus-jose-jwt)
- Self-hosted IdPs: [Keycloak](https://www.keycloak.org/), [Ory Hydra/Kratos](https://www.ory.sh/), [Zitadel](https://zitadel.com/), [Authentik](https://goauthentik.io/), [Dex](https://dexidp.io/)
- Authorization engines: [OPA](https://www.openpolicyagent.org/), [Cedar](https://www.cedarpolicy.com/), [OpenFGA](https://openfga.dev/), [SpiceDB](https://authzed.com/spicedb), [Google Zanzibar paper](https://research.google/pubs/zanzibar-googles-consistent-global-authorization-system/)

### Multi-tenancy

- [PostgreSQL Row Security Policies](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)
- [AWS SaaS tenant isolation](https://docs.aws.amazon.com/whitepapers/latest/saas-architecture-fundamentals/tenant-isolation.html)
- [Azure multitenant architecture guidance](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/overview)

### Books

- *OAuth 2 in Action*, Justin Richer & Antonio Sanso (Manning)
- *Solving Identity Management in Modern Applications*, Yvonne Wilson & Abhishek Hingnikar (Apress)
- *Web Application Security*, Andrew Hoffman (O'Reilly)
- *Security Engineering*, Ross Anderson (free online: [cl.cam.ac.uk/~rja14/book.html](https://www.cl.cam.ac.uk/~rja14/book.html))
- *Serious Cryptography*, Jean-Philippe Aumasson (No Starch Press)
- *Designing Data-Intensive Applications*, Martin Kleppmann (O'Reilly), for the consistency trade-offs behind sessions, revocation, and policy sync

---

*Note: links were checked for reachability on the date above. Standards marked "draft" change; confirm status before citing them in a design review.*
