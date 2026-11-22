# Sample Review: Orders Portal (Web App)

A filled-in example of `checklists/web-apps.md` for a fictional internal app. Copy this layout for your own reviews.

| Field | Value |
| --- | --- |
| App | Orders Portal (`orders.example.internal`) |
| Stack | React SPA, Node.js API, PostgreSQL |
| Reviewer | AppSec team |
| Date | 2026-11-22 |
| Release | v2.4.0 |

## Summary

Ready to ship after two fixes. Missing `HttpOnly` on the session cookie and a CSP with `unsafe-inline` are the blockers. Everything else is either done or has a tracked follow-up.

## Results

### Transport and cookies

- [x] **HTTPS everywhere**: HSTS set with `max-age=31536000`. HTTP redirects to HTTPS.
- [ ] **Secure cookie flags**: `sid` cookie is missing `HttpOnly`. **Blocker.** Ticket SEC-142.
- [x] **Cookie scope minimized**: Host-only cookie. Session ID rotates on login.

### Browser security headers

- [ ] **Content-Security-Policy**: Policy exists but allows `script-src 'unsafe-inline'`. **Blocker.** Move the two inline scripts to files or add nonces. Ticket SEC-143.
- [x] **X-Content-Type-Options**: `nosniff` on all responses.
- [x] **Referrer-Policy**: `strict-origin-when-cross-origin`.
- [ ] **Permissions-Policy**: Not set. Low risk for this app. Follow-up SEC-150.
- [x] **Frame controls**: `frame-ancestors 'none'`.

### Authentication and session UX

- [x] **Login rate limiting**: Handled by the corporate IdP.
- [x] **MFA**: Enforced by the IdP for all users.
- [x] **Logout and session revocation**: Server session deleted on logout.

### Input, output, and file handling

- [x] **Output encoding**: React auto-escaping. One `dangerouslySetInnerHTML` found, input passes through DOMPurify. Reviewed and accepted.
- [x] **File upload hardening**: CSV uploads only, 5 MB max, stored in a private bucket with random keys.
- [x] **Error messages sanitized**: API returns an error code and correlation ID only.

### Dependencies and third parties

- [x] **Third-party scripts inventory**: One analytics script, loaded with SRI.
- [x] **Dependency updates**: Renovate enabled. No open critical or high CVEs.

### Access control in the UI layer

- [x] **UI is not authorization**: Spot-checked 12 API routes. All enforce role checks server-side.

## Follow-ups

| Ticket | Item | Severity | Owner | Due |
| --- | --- | --- | --- | --- |
| SEC-142 | Add `HttpOnly` to `sid` cookie | High | API team | Before release |
| SEC-143 | Remove `unsafe-inline` from CSP | High | Frontend team | Before release |
| SEC-150 | Add `Permissions-Policy` header | Low | Frontend team | Next sprint |
