# API Hardening Checklist

For REST, GraphQL, gRPC, and webhook APIs used by your own frontends, mobile apps, partners, or other services. See [owasp-api-top10.md](owasp-api-top10.md) for the same ground organized by OWASP category.

## 1. Inventory and documentation

- [ ] **Every API is in an inventory**: Each API has an owner, environment list, data classification, and auth method.
  - **Why it matters:** Unknown APIs do not get patched, monitored, or retired.
  - **Secure default:** OpenAPI or GraphQL schema in the repo, published to an internal catalog.

- [ ] **Spec matches reality**: The OpenAPI spec is generated from code or checked against it in CI.
  - **Why it matters:** Undocumented routes skip security review.
  - **How to verify:** Compare the gateway or framework route table against the spec.

- [ ] **Old versions retired**: Deprecated versions have a sunset date. Traffic is monitored and the version is removed after the date.
  - **Why it matters:** `/v1/` endpoints often lack controls added later in `/v2/`.

- [ ] **Non-production APIs not public**: Staging, QA, and debug APIs sit behind VPN, SSO, or IP allowlists.

## 2. Authentication

- [ ] **Every non-public endpoint requires auth**: Global auth middleware, with an explicit list of public routes.
  - **How to verify:** Call each route without credentials in a test environment. Only allowlisted routes return 2xx.

- [ ] **Standard protocols**: OAuth 2.0 / OIDC for user-delegated access. Authorization code flow with PKCE for SPAs and mobile. No implicit flow. No resource owner password grant.
  - **Why it matters:** Home-grown token schemes tend to have subtle flaws.

- [ ] **JWT validation complete**: Verify signature, `alg` against an allowlist, `iss`, `aud`, `exp`, `nbf`. Reject `alg: none`. Do not pick keys from untrusted `jku` or `x5u` headers.
  - **Secure default:** Use a maintained library with explicit algorithm and audience configuration.
  - **How to verify:** Unit tests that feed expired, wrong-audience, wrong-issuer, and unsigned tokens and expect 401.

- [ ] **Short-lived access tokens**: 5 to 60 minutes. Refresh tokens rotate on use.

- [ ] **Token revocation works**: Logout, password change, and account disable revoke refresh tokens. For opaque tokens, introspection reflects revocation.

- [ ] **API keys for machine clients only**: Keys are per client, scoped, rotatable, and stored hashed on the server. Never ship API keys with elevated access in browser or mobile code.
  - **Why it matters:** Anything in a client binary or JS bundle is public.

- [ ] **Service-to-service auth**: mTLS or signed tokens (workload identity, SPIFFE) between internal services. Network location alone is not trust.

## 3. Authorization

- [ ] **Object-level checks on every ID**: Any endpoint that takes an object ID (path, query, body, header) checks that the caller may access that object.
  - **Why it matters:** Broken object level authorization (BOLA) is the top API risk.
  - **Secure default:** Scope queries by tenant and owner: `WHERE id = ? AND tenant_id = ?`. Do not fetch first and check later, which is easy to forget.
  - **How to verify:** Integration tests with two users in different tenants. User A requests User B's object IDs and gets 403 or 404.

- [ ] **Function-level checks**: Admin and privileged operations check role or permission explicitly in the handler or a policy layer.
  - **How to verify:** Call admin routes with a normal user token in tests. Expect 403.

- [ ] **Property-level checks on read**: Sensitive fields (internal notes, other users' emails, cost price, flags) are returned only to callers allowed to see them.

- [ ] **Property-level checks on write**: Writable fields are allowlisted per role. Fields like `role`, `tenant_id`, `balance`, `verified`, `price` are never bound from the request body for normal users.
  - **Why it matters:** Prevents mass assignment.
  - **Secure default:** Separate request DTOs per operation. Reject unknown fields.

- [ ] **Tenant isolation enforced in one place**: Multi-tenant apps apply the tenant filter in a data access layer or row-level security, not ad hoc in each handler.
  - **Secure default:** PostgreSQL row-level security or a repository layer that requires a tenant context.

- [ ] **Unpredictable IDs are not a control**: UUIDs reduce guessing, but authorization checks are still required.

- [ ] **Policy decisions tested**: Authorization rules have unit tests that cover allow and deny for each role.

## 4. Input validation

- [ ] **Schema validation on every request**: Types, required fields, string length, numeric ranges, enum values, array sizes.
  - **Secure default:** Validate against the OpenAPI schema at the gateway or framework layer. Reject unknown fields.

- [ ] **Content-Type enforced**: Accept only the expected type (`application/json`). Return 415 otherwise.
  - **Why it matters:** Parsers for other types (XML, form data) may be enabled by default and less hardened.

- [ ] **Request size limits**: Max body size at the proxy and app. Max JSON depth and array length.

- [ ] **Injection-safe data access**: Parameterized queries, safe NoSQL query construction (no raw operator objects from user input), no dynamic code evaluation.

- [ ] **Consistent number handling**: Reject `NaN`, infinities, negative values where not allowed, and values past safe integer range.

## 5. Rate limiting and resource controls

- [ ] **Rate limits per user, per client, and per IP**: Separate, stricter limits for login, OTP, password reset, signup, and search.
  - **Secure default:** Token bucket at the gateway. Return 429 with `Retry-After`.
  - **How to verify:** Load test in staging and confirm 429s at the configured threshold.

- [ ] **Pagination enforced**: Default and maximum `limit` values. Cursor-based pagination for large sets.

- [ ] **Timeouts on everything**: Request timeouts, DB query timeouts, and timeouts on outbound calls.

- [ ] **Cost controls on expensive operations**: Report generation, exports, and bulk operations are queued with per-user quotas.

- [ ] **Spend limits on paid integrations**: SMS, email, AI, and other per-call billed services have per-user and global caps.
  - **Why it matters:** An open SMS endpoint can run up large bills (SMS pumping).

## 6. Response handling

- [ ] **Return only needed fields**: Explicit response serializers. Never return the whole ORM object.

- [ ] **Generic errors**: No stack traces, SQL, or internal hostnames. Use stable error codes.

- [ ] **Consistent 403 vs 404 policy**: Decide whether unauthorized object access returns 403 or 404 and apply it everywhere. 404 avoids confirming that an object exists.

- [ ] **Security headers on API responses**: `Cache-Control: no-store` for user data, `X-Content-Type-Options: nosniff`, correct `Content-Type`.

## 7. GraphQL

- [ ] **Introspection disabled or restricted in production**.

- [ ] **Field suggestions disabled in production**: Turn off "Did you mean" hints in error messages.
  - **Why it matters:** Suggestions leak schema details when introspection is off.

- [ ] **Depth, complexity, and alias limits**: Max query depth, a cost budget per query, and a cap on aliases and batched operations.
  - **Secure default:** graphql-armor or the equivalent in your server.

- [ ] **Authorization in resolvers or a shared layer**: Each type and field enforces access, not just the top-level query.

- [ ] **Persisted queries for first-party clients**: Allow only known query hashes from your own apps where practical.

## 8. Webhooks

- [ ] **Outbound webhooks signed**: HMAC signature with a timestamp header. Receivers verify signature and reject old timestamps (for example older than 5 minutes).

- [ ] **Inbound webhooks verified**: Verify provider signatures (Stripe, GitHub, etc.) with constant-time comparison before processing.

- [ ] **Webhook targets restricted**: When customers register webhook URLs, block private IP ranges and metadata endpoints to prevent SSRF.

- [ ] **Idempotent processing**: Use the event ID to ignore duplicates.

## 9. Transport

- [ ] **TLS 1.2+ on all endpoints**, including internal ones where practical.

- [ ] **No sensitive data in query strings**: Tokens, passwords, and PII go in headers or the body.

- [ ] **mTLS for high-trust partner integrations**.

## 10. Gateway and logging

- [ ] **API gateway or shared middleware**: Auth, rate limits, size limits, and logging are applied consistently in one layer.

- [ ] **Per-request audit fields**: Caller identity, client ID, route, object IDs, status code, latency, and correlation ID.

- [ ] **Alerts on authorization failures**: Spikes in 401 and 403 from one client or user, and sequential object ID access patterns.
