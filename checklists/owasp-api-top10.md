# OWASP API Security Top 10 (2023) Defensive Checklist

Defensive checks mapped to the [OWASP API Security Top 10 2023](https://owasp.org/API-Security/editions/2023/en/0x11-t10/). Pair with [apis.md](apis.md) for implementation detail.

## API1:2023 Broken Object Level Authorization

- [ ] **Authorization check on every object ID**: Path, query, body, and header IDs all checked against the caller.
  - **Secure default:** Data access layer requires a user or tenant context and adds it to every query.

- [ ] **Check on every operation, not just read**: Update, delete, export, share, and sub-resource routes (`/orders/{id}/invoice`) included.

- [ ] **Nested IDs checked**: For `/projects/{p}/files/{f}`, verify `f` belongs to `p` and the caller can access `p`.
  - **Why it matters:** A common bug is checking the parent and trusting the child ID.

- [ ] **Bulk endpoints check every ID in the list**.

- [ ] **Automated tests**: Two-user and two-tenant authorization tests for each resource type, run in CI.

- [ ] **Monitoring**: Alert on one caller accessing an unusual number of distinct object IDs or getting many 403s.

## API2:2023 Broken Authentication

- [ ] **All auth endpoints rate limited**: Login, token, OTP verify, password reset, and refresh endpoints.

- [ ] **Standard flows and libraries**: OAuth 2.0 / OIDC with PKCE. Maintained JWT libraries with explicit algorithms.

- [ ] **Token validation complete**: Signature, algorithm allowlist, issuer, audience, expiry.

- [ ] **No credentials in URLs**.

- [ ] **Sensitive changes need re-auth**: Email, password, MFA, and payout changes.

- [ ] **Microservices authenticate each other**: mTLS or signed service tokens.

- [ ] **API keys are not user authentication**: Keys identify client apps. Users still authenticate.

## API3:2023 Broken Object Property Level Authorization

- [ ] **Response allowlists**: Serializers return only the fields each role may see.
  - **How to verify:** Snapshot tests of API responses per role. Review diffs when models change.

- [ ] **Request allowlists**: Each operation accepts only its defined writable fields. Unknown fields are rejected.

- [ ] **Sensitive properties protected**: `role`, `isAdmin`, `tenantId`, `ownerId`, `balance`, `price`, `status`, `emailVerified` writable only by trusted code paths.

- [ ] **GraphQL field-level authorization**: Sensitive fields have resolvers with authorization checks.

- [ ] **No generic "update object" endpoints** that bind the full body to a model.

## API4:2023 Unrestricted Resource Consumption

- [ ] **Rate limits and quotas**: Per user, per client, per IP, and per tenant.

- [ ] **Request size limits**: Body size, file size, number of files, JSON depth, array length.

- [ ] **Pagination max enforced**.

- [ ] **Timeouts and memory limits**: On handlers, queries, and container resources.

- [ ] **Expensive operations queued**: Async jobs with per-user concurrency limits.

- [ ] **GraphQL cost limits**: Depth, complexity, aliases, and batch size.

- [ ] **Third-party spend caps**: SMS, email, AI, and payment API calls capped per user and globally, with alerts.

## API5:2023 Broken Function Level Authorization

- [ ] **Role checks on every privileged function**: Admin routes, management APIs, and internal endpoints.

- [ ] **Admin APIs separated**: Different path prefix or host, with separate auth policies and network restrictions.

- [ ] **HTTP method checks**: A user allowed to `GET` a resource is not automatically allowed to `PUT` or `DELETE` it.

- [ ] **Deny by default for new routes**.

- [ ] **Tests per role**: Each privileged route has a test that a lower role gets 403.

## API6:2023 Unrestricted Access to Sensitive Business Flows

- [ ] **Sensitive flows identified**: Purchase, signup, reservation, referral rewards, coupon redemption, posting content, sending messages.

- [ ] **Per-flow abuse limits**: Per account, per device, per payment instrument, per address.

- [ ] **Bot mitigation on high-value flows**: Device fingerprinting, behavioral signals, or challenges where abuse is seen.

- [ ] **Human-paced limits**: Block activity faster than a human could do it.

- [ ] **Business monitoring**: Dashboards and alerts on flow volume per account and per IP range.

See [business-logic.md](business-logic.md) for detailed controls.

## API7:2023 Server Side Request Forgery

- [ ] **User-supplied URLs avoided or allowlisted**.

- [ ] **Resolved IP checked against private and metadata ranges**.

- [ ] **Redirects disabled or re-validated**.

- [ ] **Fetchers isolated at the network level**.

- [ ] **Webhook targets validated on registration and on each delivery**.

- [ ] **Only expected response content returned** to the caller.

## API8:2023 Security Misconfiguration

- [ ] **TLS everywhere**: Including internal service-to-service where practical.

- [ ] **Unneeded HTTP methods disabled**.

- [ ] **CORS restricted**: Allowlist, no reflected origins.

- [ ] **Security headers on API responses**: `Cache-Control: no-store` for user data, `nosniff`.

- [ ] **Errors generic**: No stack traces, versions, or internal paths.

- [ ] **Consistent config across environments**: From IaC, scanned in CI.

- [ ] **Parsers strict**: Unused content types (XML, YAML) disabled. XML external entities off.

## API9:2023 Improper Inventory Management

- [ ] **Inventory of all APIs and versions**: Owner, environment, data sensitivity, exposure (public, partner, internal).

- [ ] **Specs generated and published**: OpenAPI or GraphQL schemas in a catalog.

- [ ] **Deprecated versions retired on schedule**.

- [ ] **Non-prod environments protected**: No production data in non-prod. Access restricted.

- [ ] **Third-party data flows documented**: What data goes to which partner API, and why.

- [ ] **Discovery from traffic**: Gateway or traffic analysis finds endpoints not in the inventory.

## API10:2023 Unsafe Consumption of APIs

- [ ] **Validate third-party API responses**: Schema-validate data from partners before use. Treat it as untrusted input.

- [ ] **TLS and certificate validation for outbound calls**.

- [ ] **Timeouts and circuit breakers**: Partner outages should not take you down.

- [ ] **Redirects from third-party APIs not followed blindly**.

- [ ] **Limits on response size** from third-party APIs.

- [ ] **Partner security reviewed**: Vendor assessment for APIs that handle your sensitive data.

- [ ] **Credentials for third-party APIs scoped and rotated**.
