# Sample Review: Orders Portal (Web App)

A filled-in example of [checklists/web-apps.md](../checklists/web-apps.md) for a fictional internal app. Section names and item names match the checklist so findings can be traced back. Copy this layout for your own reviews.

| Field | Value |
| --- | --- |
| App | Orders Portal (`orders.example.internal`) |
| Stack | React SPA, Node.js (Express) API, PostgreSQL, behind an AWS ALB and CloudFront |
| Auth | Corporate OIDC IdP, server-side sessions in Redis |
| Reviewer | AppSec team |
| Date | 2026-11-22 |
| Release | v2.4.0 |
| Checklist version | `web-apps.md` at commit `eb1596f` |

## Legend

- `[x]` passes
- `[ ]` with **Blocker** must be fixed before release
- `[ ]` with a ticket is a tracked follow-up
- **N/A** means the item does not apply, with the reason

## Summary

Ready to ship after two fixes:

1. The `sid` session cookie is missing `HttpOnly` (section 2).
2. The CSP allows `script-src 'unsafe-inline'` (section 3).

Five lower-risk items have follow-up tickets. Everything else passes or does not apply.

## Results

### 1. TLS and transport

- [x] **HTTPS on every route**: ALB listener on port 80 returns 301 to HTTPS. `curl -sI http://orders.example.internal/` confirmed. No mixed content in devtools.
- [x] **TLS 1.2 minimum, TLS 1.3 preferred**: ALB policy `ELBSecurityPolicy-TLS13-1-2-2021-06`. testssl.sh shows TLS 1.0 and 1.1 off.
- [x] **Strong cipher suites only**: Same policy. All suites use ECDHE. No CBC-only suites for TLS 1.2.
- [x] **HSTS deployed**: `max-age=31536000; includeSubDomains` on the app host. Not preloaded, which is fine for an internal host.
- [x] **Certificates managed and monitored**: ACM auto-renewal. Expiry alarm in CloudWatch. CAA record limits issuance to Amazon.

### 2. Cookies

- [ ] **Session cookies are `Secure` and `HttpOnly`**: `sid` has `Secure` but not `HttpOnly`. The frontend reads it to decide whether to show the login button. **Blocker.** Fix: set `httpOnly: true` in the Express session config and use `/api/me` for login state. Ticket SEC-142.
- [x] **Explicit `SameSite`**: `SameSite=Lax`.
- [ ] **Cookie prefixes for session cookies**: Cookie is named `sid`, no `__Host-` prefix. Rename together with the SEC-142 fix. Ticket SEC-151.
- [x] **Narrow `Domain` and `Path`**: Host-only cookie, `Path=/`.
- [x] **No sensitive data in cookies**: `sid` is an opaque 32-byte random value. Role and user data live in Redis.

### 3. Security headers

- [ ] **Content-Security-Policy enforced**: Policy is enforced but includes `script-src 'self' 'unsafe-inline'`. Two inline scripts in `index.html` (feature flag bootstrap and analytics init) are the reason. **Blocker.** Fix: move both into bundled files, or add per-response nonces, then drop `unsafe-inline`. CSP Evaluator run attached to the ticket. Ticket SEC-143.
- [x] **CSP violation reporting**: `report-to` sends to the logging pipeline. About 20 reports per day, all from browser extensions.
- [x] **`frame-ancestors` set**: `frame-ancestors 'none'` plus `X-Frame-Options: DENY`.
- [x] **`X-Content-Type-Options: nosniff`**: Set by helmet on all responses, and by CloudFront on static assets.
- [x] **`Referrer-Policy`**: `strict-origin-when-cross-origin`.
- [ ] **`Permissions-Policy`**: Not set. The app uses no camera, microphone, or location, so risk is low. Ticket SEC-150.
- [x] **Cross-origin isolation headers where needed**: `Cross-Origin-Opener-Policy: same-origin` set. `Cross-Origin-Resource-Policy: same-origin` on API responses.
- [x] **No version disclosure headers**: `X-Powered-By` removed by helmet. CloudFront `Server` header shows only `CloudFront`. Checked a page, an API route, a static file, and a 404.

### 4. Session management

- [x] **Session ID regenerated on login and privilege change**: `req.session.regenerate()` after OIDC callback. Cookie value changes after login (confirmed in devtools).
- [x] **Server-side logout**: Logout deletes the Redis session. Replaying a saved cookie after logout returns 401.
- [x] **Idle and absolute timeouts**: 30 minutes idle, 12 hours absolute.
- [x] **Re-authentication for sensitive actions**: N/A for most of the app. The one sensitive action (bulk order cancel) forces an OIDC `prompt=login` round trip with `max_age=300`.
- [ ] **Session list and revoke**: No UI for active sessions. Sessions are revoked when the user is disabled in the IdP. Acceptable for an internal tool. Ticket SEC-152, low priority.
- [x] **No tokens in URLs**: OIDC `code` appears in the callback URL only and is single-use. No other tokens in query strings.

### 5. CSRF protection

- [x] **CSRF tokens or equivalent on state-changing requests**: `csrf-csrf` double-submit with signed cookie on all `POST`, `PUT`, `PATCH`, and `DELETE` routes.
- [x] **No state changes on GET**: Route table reviewed. All 41 GET handlers are read-only.
- [x] **Origin check on sensitive endpoints**: Middleware rejects state-changing requests where `Sec-Fetch-Site` is `cross-site`.

### 6. CORS

- [x] **CORS allowlist, not reflection**: API and SPA share an origin, so CORS is off. No `Access-Control-Allow-Origin` header returned for a test request from another origin.
- [x] **No `null` origin in allowlist**: No allowlist exists.
- [x] **Credentials only when needed**: Not set.

### 7. Input handling and injection prevention

- [x] **Parameterized queries everywhere**: Knex query builder. Two `knex.raw` calls found. Both use bindings. Sort column comes from a hard-coded map. Semgrep `javascript.knex` rules clean.
- [x] **No shell command construction from input**: No `child_process` use in the API.
- [x] **Server-side validation on every input**: Zod schemas on all routes, applied in a shared middleware. Unknown fields rejected.
- [x] **Safe deserialization**: JSON only.
- [x] **XML parsers hardened**: **N/A.** No XML parsing.
- [x] **Template engines not fed user input as templates**: **N/A.** No server-side templates. React only.
- [x] **Regex safe from catastrophic backtracking**: Four regexes on user input, all simple anchored patterns. Inputs capped at 256 characters by Zod.

### 8. Output encoding and XSS prevention

- [x] **Context-aware auto-escaping on**: React default escaping.
- [x] **Raw HTML sinks reviewed**: One `dangerouslySetInnerHTML` in the order notes view. Input passes through DOMPurify with the default profile. ESLint `react/no-danger` now warns on new uses.
- [x] **No user input in `href`, `src`, or event handlers without validation**: Customer website links pass through a helper that allows only `http:` and `https:`.
- [ ] **Trusted Types for DOM sinks**: Not enabled. Revisit after SEC-143 removes inline scripts. Ticket SEC-153.
- [x] **JSON responses served as JSON**: `application/json` with `nosniff`. No JSON embedded in HTML.

### 9. File uploads and downloads

- [x] **Allowlist file types**: CSV only. Extension and content checked with `file-type` plus a CSV parse.
- [x] **Size and count limits**: 5 MB per file, one file per request, enforced by `multer` and the ALB.
- [x] **Store outside the web root**: Private S3 bucket, random UUID keys.
- [x] **Serve user files from a separate domain**: **N/A.** Uploaded CSVs are imported and never served back.
- [x] **Image processing isolated**: **N/A.** No image processing.
- [x] **Archive extraction safe**: **N/A.** No archive uploads.
- [x] **Download endpoints check authorization per file**: Order export downloads use pre-signed S3 URLs with a 5 minute expiry, generated only after an ownership check.

### 10. Server-side request handling

- [x] **Outbound requests from user-supplied URLs are restricted**: **N/A.** The app never fetches user-supplied URLs.
- [x] **Path handling safe**: No filesystem paths built from input.
- [x] **Open redirects removed**: Post-login `returnTo` accepts relative paths only. Absolute URLs are dropped.

### 11. Error handling

- [x] **Generic error pages in production**: `NODE_ENV=production` enforced in the container. A 404, a 405, and a Zod validation error return generic JSON.
- [x] **Correlation ID in errors**: `requestId` in every error body and log line.
- [x] **Fail closed**: Authorization middleware wraps the policy call in try/catch and returns 403 on error. Unit test covers it.

### 12. Third-party code and dependencies

- [x] **Script inventory**: One third-party script (analytics). Owner: Product Analytics team.
- [x] **Subresource Integrity on static third-party files**: Analytics script loaded with `integrity` and `crossorigin="anonymous"`.
- [x] **Tag managers restricted**: **N/A.** No tag manager.
- [x] **Dependency scanning in CI**: `npm audit --audit-level=high` and OSV-Scanner on every PR. No open critical or high findings.

### 13. Access control in the web layer

- [x] **Server-side authorization on every request**: Spot-checked 12 API routes. All call `authorize(user, action, resource)`. Integration tests cover cross-team order access.
- [x] **Deny by default**: Global auth middleware. Public allowlist is `/healthz` and `/auth/callback`. Route table diffed against the allowlist in CI.
- [x] **Admin interfaces separated**: Admin routes on `orders-admin.example.internal`, restricted to the admin IdP group and the corporate VPN.

### 14. Caching

- [x] **No caching of authenticated responses at shared caches**: API responses send `Cache-Control: no-store`. CloudFront behavior for `/api/*` has caching disabled. Confirmed with `x-cache: Miss from cloudfront` on repeated requests.
- [x] **Cache keys include what matters**: Static assets only are cached, keyed by path with content-hashed filenames. No responses vary by `Host` or forwarded headers.

## Follow-ups

| Ticket | Section | Item | Severity | Owner | Due |
| --- | --- | --- | --- | --- | --- |
| SEC-142 | 2. Cookies | Add `HttpOnly` to `sid` | High (blocker) | API team | Before release |
| SEC-143 | 3. Security headers | Remove `unsafe-inline` from CSP | High (blocker) | Frontend team | Before release |
| SEC-150 | 3. Security headers | Add `Permissions-Policy` | Low | Frontend team | Next sprint |
| SEC-151 | 2. Cookies | Rename cookie to `__Host-sid` | Low | API team | With SEC-142 |
| SEC-152 | 4. Session management | Active session list and revoke | Low | API team | Backlog |
| SEC-153 | 8. Output encoding and XSS prevention | Enable Trusted Types | Medium | Frontend team | After SEC-143 |

## Sign-off

- [ ] SEC-142 closed and verified
- [ ] SEC-143 closed and verified
- [ ] Release approved by AppSec
