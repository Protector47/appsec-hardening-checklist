# Web Application Hardening Checklist

Use this when reviewing browser-facing apps (SSR, SPA, hybrid). Mark each item during design reviews, security gates, and release readiness.

## Transport and cookies

- [ ] **HTTPS everywhere**: Redirect HTTP to HTTPS; disable mixed content; HSTS with a sensible `max-age` (and `includeSubDomains` when ready).
  - **Why it matters:** Plain HTTP leaks credentials, cookies, and session tokens.
  - **Secure default:** HSTS `max-age=31536000` after staged rollout; TLS 1.2+ only.

- [ ] **Secure cookie flags**: Session cookies use `Secure`, `HttpOnly`, and an explicit `SameSite` (`Lax` or `Strict` unless a cross-site flow requires `None` + `Secure`).
  - **Why it matters:** Missing flags enable theft via XSS or cross-site request abuse.
  - **Secure default:** `Secure; HttpOnly; SameSite=Lax` for session cookies.

- [ ] **Cookie scope minimized**: Prefer host-only cookies; avoid overly broad `Domain` and long-lived session cookies without rotation.
  - **Why it matters:** Broad cookies increase blast radius across subdomains.
  - **Secure default:** Host-only; short idle timeout; regenerate session ID on login.

## Browser security headers

- [ ] **Content-Security-Policy (CSP)**: Deploy a restrictive CSP; avoid `unsafe-inline` / `unsafe-eval` where possible; use nonces or hashes for required inline scripts.
  - **Why it matters:** CSP reduces impact of XSS and unwanted third-party script execution.
  - **Secure default:** `default-src 'self'`; report-only first, then enforce.

- [ ] **X-Content-Type-Options**: Send `nosniff` on all responses that serve user-influenced content.
  - **Why it matters:** Prevents MIME sniffing that can turn uploads or errors into executable content.
  - **Secure default:** `X-Content-Type-Options: nosniff`

- [ ] **Referrer-Policy**: Limit sensitive path/query leakage in the `Referer` header.
  - **Why it matters:** URLs may contain tokens or PII that leak to third parties.
  - **Secure default:** `strict-origin-when-cross-origin` or stricter.

- [ ] **Permissions-Policy**: Disable unused powerful features (camera, mic, geolocation, payment, etc.).
  - **Why it matters:** Compromised or third-party scripts can abuse browser capabilities.
  - **Secure default:** Deny features the product does not need.

- [ ] **Frame controls**: Use `frame-ancestors` in CSP (and legacy `X-Frame-Options` where needed) to prevent unwanted embedding.
  - **Why it matters:** Clickjacking and UI redressing attacks rely on framing.
  - **Secure default:** `frame-ancestors 'self'` or `'none'`.

## Authentication and session UX

- [ ] **Login rate limiting and lockout policy**: Protect credential endpoints against credential stuffing without creating easy account lockout DoS.
  - **Why it matters:** Unbounded login attempts enable account takeover at scale.
  - **Secure default:** Progressive delays / CAPTCHA / IP+account throttles; alert on anomalies.

- [ ] **Multi-factor authentication**: Offer MFA for privileged and high-risk accounts; prefer phishing-resistant factors where feasible.
  - **Why it matters:** Password-only accounts fail under phishing and reuse.
  - **Secure default:** MFA required for admins; encourage WebAuthn/passkeys.

- [ ] **Logout and session revocation**: Server-side session invalidation on logout; revoke refresh tokens; clear client storage intentionally.
  - **Why it matters:** Client-only logout leaves stolen sessions usable.
  - **Secure default:** Invalidate server session + rotate/revoke refresh tokens.

## Input, output, and file handling

- [ ] **Output encoding by context**: Encode for HTML, attribute, JS, URL, and CSS contexts; prefer frameworks’ auto-escaping.
  - **Why it matters:** Context-wrong encoding is a common XSS root cause.
  - **Secure default:** Auto-escape templates; avoid `dangerouslySetInnerHTML` / raw HTML unless sanitized and reviewed.

- [ ] **File upload hardening**: Validate type/size server-side; store outside web root or in object storage; serve with safe `Content-Type` and `Content-Disposition`; scan when appropriate.
  - **Why it matters:** Uploads become malware hosting, XSS, or RCE vectors when mishandled.
  - **Secure default:** Allowlist extensions/MIME; random object keys; no execute permissions.

- [ ] **Error messages sanitized**: Generic client errors; detailed diagnostics only in server logs.
  - **Why it matters:** Stack traces and SQL fragments aid reconnaissance.
  - **Secure default:** Correlation ID in UI; full detail in logs only.

## Dependencies and third parties

- [ ] **Third-party scripts inventory**: Document every CDN/script/tag manager; prefer SRI (`integrity`) and strict CSP hosts.
  - **Why it matters:** Supply-chain compromise of a tag can own every session.
  - **Secure default:** Minimize third parties; SRI + CSP allowlist.

- [ ] **Dependency updates**: Track CVEs for frontend and backend deps; patch critical issues on a defined SLA.
  - **Why it matters:** Known vulnerable libraries are frequent breach entry points.
  - **Secure default:** Automated Dependabot/Renovate + CI fail on critical CVEs.

## Access control in the UI layer

- [ ] **UI is not authorization**: Hide/disable controls for UX only; enforce every action on the server.
  - **Why it matters:** Attackers call APIs directly and bypass the UI.
  - **Secure default:** Server-side authorization on every mutating and sensitive read path.
