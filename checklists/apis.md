# API Hardening Checklist

For REST, GraphQL, gRPC, and similar service APIs consumed by first-party or partner clients.

## Authentication and tokens

- [ ] **Strong auth for every non-public endpoint**: Prefer OAuth 2.1 / OIDC patterns; avoid long-lived static API keys in browsers.
  - **Why it matters:** Stolen or leaked credentials are the most common API breach vector.
  - **Secure default:** Short-lived access tokens; refresh tokens bound to client/context where possible.

- [ ] **Audience and issuer validation**: Validate `iss`, `aud`, signature, expiry, and algorithm on every JWT/token check.
  - **Why it matters:** Accepting arbitrary tokens or `alg=none` breaks trust boundaries.
  - **Secure default:** Allowlist algorithms (e.g., RS256/ES256); reject missing/invalid claims.

- [ ] **Least-privilege scopes**: Tokens carry minimal scopes/roles for the task; separate admin and user audiences.
  - **Why it matters:** Over-scoped tokens amplify a single compromise.
  - **Secure default:** Fine-grained scopes; deny by default.

## Authorization

- [ ] **Object-level authorization**: Every request that names a resource ID is checked against the caller’s permissions (BOLA/IDOR prevention by design).
  - **Why it matters:** Changing an ID in a request is a top real-world API failure.
  - **Secure default:** Central authZ helper/policy engine; never trust client-supplied ownership flags.

- [ ] **Function-level authorization**: Admin and privileged operations require explicit role/permission checks, not “hidden” routes.
  - **Why it matters:** Undocumented admin endpoints are frequently exposed without authZ.
  - **Secure default:** Deny-by-default middleware; inventory all routes.

- [ ] **Mass assignment / over-posting controls**: Allowlist writable fields; never bind raw request bodies to sensitive model fields (`role`, `isAdmin`, price, etc.).
  - **Why it matters:** Extra JSON fields can elevate privileges or alter business state.
  - **Secure default:** Explicit DTOs / schema validation with allowlists.

## Input validation and schemas

- [ ] **Schema validation at the edge**: Validate content-type, required fields, types, ranges, and max sizes before business logic.
  - **Why it matters:** Unexpected payloads cause injection, logic bugs, and resource exhaustion.
  - **Secure default:** OpenAPI/JSON Schema enforced in CI and at runtime.

- [ ] **Consistent rejection behavior**: Invalid requests return generic 4xx without leaking schema internals beyond what clients need.
  - **Why it matters:** Verbose validation errors can map the attack surface.
  - **Secure default:** Stable error codes; details for developers only in non-prod.

## Rate limits and abuse controls

- [ ] **Per-client and per-user rate limits**: Protect auth, password reset, search, and expensive queries separately.
  - **Why it matters:** APIs are automated-attack friendly without throttles.
  - **Secure default:** Tiered limits; burst + sustained caps; clear `Retry-After`.

- [ ] **Pagination and result caps**: Enforce max page size; avoid unbounded list endpoints.
  - **Why it matters:** Large responses enable data scraping and DoS.
  - **Secure default:** Default page size small; hard maximum enforced server-side.

## Data exposure

- [ ] **Response field minimization**: Return only fields the client needs; strip internal IDs, secrets, and debug flags.
  - **Why it matters:** Excess data in JSON is a privacy and recon risk.
  - **Secure default:** Explicit response serializers / GraphQL field auth.

- [ ] **PII and secrets never in URLs**: Prefer headers/body for tokens and sensitive identifiers; avoid secrets in query strings and logs.
  - **Why it matters:** URLs leak via logs, proxies, and Referer.
  - **Secure default:** Authorization header / secure cookies; redact logs.

## GraphQL-specific (when applicable)

- [ ] **Disable unused introspection in production**: Or restrict it to trusted operators.
  - **Why it matters:** Full schemas accelerate discovery of sensitive fields.
  - **Secure default:** Introspection off in prod unless required and access-controlled.

- [ ] **Query depth/complexity limits**: Cap nesting, aliases, and batch size to protect resolvers.
  - **Why it matters:** Nested queries can exhaust CPU/DB without classic rate limits firing.
  - **Secure default:** Depth + cost analysis middleware; timeouts on resolvers.

## Operations

- [ ] **Versioning and deprecation policy**: Document breaking changes; retire insecure old versions on a schedule.
  - **Why it matters:** Forgotten v1 endpoints often skip modern controls.
  - **Secure default:** Sunset headers / dates; monitor usage of deprecated versions.

- [ ] **TLS and mTLS where appropriate**: TLS for all external APIs; consider mTLS for service-to-service in zero-trust networks.
  - **Why it matters:** Cleartext or weak TLS exposes tokens and PII.
  - **Secure default:** TLS 1.2+; strong cipher suites; certificate rotation process.
