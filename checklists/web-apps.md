# Web Application Hardening Checklist

For browser-facing apps: server-rendered, SPA, or hybrid. Work through it in design review, again before release, and after major framework upgrades.

Each item has the check, why it matters, a secure default, and where useful a way to verify it on your own app.

## 1. TLS and transport

- [ ] **HTTPS on every route**: All pages, assets, and API calls load over HTTPS. HTTP returns a 301 to HTTPS and nothing else.
  - **Why it matters:** One HTTP request can leak a cookie or let a network attacker inject script.
  - **Secure default:** Redirect at the load balancer or CDN. No app logic served on port 80.
  - **How to verify:** `curl -sI http://your-host/` returns 301 with an `https://` `Location`. Browser devtools show no mixed content warnings.

- [ ] **TLS 1.2 minimum, TLS 1.3 preferred**: SSLv3, TLS 1.0, and TLS 1.1 are disabled.
  - **Why it matters:** Old protocol versions have known weaknesses and fail compliance checks (PCI DSS, most internal baselines).
  - **Secure default:** Mozilla "intermediate" TLS profile, or "modern" if you have no legacy clients.
  - **How to verify:** Run `testssl.sh` or SSL Labs against your own host. Grade A or better.

- [ ] **Strong cipher suites only**: No RC4, 3DES, export, NULL, or CBC-only suites for TLS 1.2. Forward secrecy (ECDHE) on all suites.
  - **Why it matters:** Weak ciphers allow decryption of recorded traffic.
  - **Secure default:** Mozilla intermediate cipher list.

- [ ] **HSTS deployed**: `Strict-Transport-Security` is sent on all HTTPS responses.
  - **Why it matters:** Stops SSL stripping after the first visit.
  - **Secure default:** Start with `max-age=300`, raise to `max-age=31536000; includeSubDomains` once every subdomain serves HTTPS. Consider HSTS preload only when you are sure.
  - **How to verify:** Response headers on the main domain and a sample of subdomains.

- [ ] **Certificates managed and monitored**: Automated renewal (ACME or cloud cert manager). Expiry alerts at 30 and 7 days. CAA DNS records restrict which CAs can issue.
  - **Why it matters:** Expired certs cause outages and train users to click through warnings.
  - **Secure default:** ACME auto-renewal plus Certificate Transparency monitoring for your domains.

## 2. Cookies

- [ ] **Session cookies are `Secure` and `HttpOnly`**: Both flags on every cookie that carries a session or auth token.
  - **Why it matters:** `HttpOnly` keeps script (including injected script) from reading the token. `Secure` keeps it off plain HTTP.
  - **Secure default:** Set in framework session config, not per response.
  - **How to verify:** Devtools Application tab, or check `Set-Cookie` headers after login.

- [ ] **Explicit `SameSite`**: `Lax` for most session cookies, `Strict` for admin apps. `None` only with a documented cross-site need, and always with `Secure`.
  - **Why it matters:** `SameSite` is a strong first layer against cross-site request forgery.
  - **Secure default:** `SameSite=Lax`.

- [ ] **Cookie prefixes for session cookies**: Use the `__Host-` prefix where possible.
  - **Why it matters:** Browsers enforce `Secure`, `Path=/`, and no `Domain` for `__Host-` cookies, so a subdomain cannot overwrite them.
  - **Secure default:** `__Host-sid=...; Secure; HttpOnly; SameSite=Lax; Path=/`.

- [ ] **Narrow `Domain` and `Path`**: Avoid `Domain=.example.com` unless subdomains really must share the session.
  - **Why it matters:** A single weak subdomain (marketing site, old staging) can read or set parent domain cookies.
  - **Secure default:** Host-only cookies.

- [ ] **No sensitive data in cookies**: Cookies hold an opaque session ID or a signed, encrypted token. No raw user IDs, roles, or PII.
  - **Why it matters:** Clients can read and change cookie values. Roles in cookies get edited.
  - **Secure default:** Opaque random session ID (at least 128 bits of entropy) mapped to server-side state.

## 3. Security headers

- [ ] **Content-Security-Policy enforced**: A CSP that blocks inline script and limits script sources.
  - **Why it matters:** CSP is the main browser-side control that limits damage from XSS.
  - **Secure default:** Nonce-based strict CSP: `script-src 'nonce-{random}' 'strict-dynamic'; object-src 'none'; base-uri 'none'`. Roll out with `Content-Security-Policy-Report-Only` first.
  - **How to verify:** Google CSP Evaluator on your policy. Check that no `unsafe-inline` or `unsafe-eval` remains in `script-src`.

- [ ] **CSP violation reporting**: `report-to` or `report-uri` points to an endpoint you monitor.
  - **Why it matters:** Reports show broken pages after a deploy and unexpected script injection.
  - **Secure default:** Send reports to your logging pipeline with rate limiting.

- [ ] **`frame-ancestors` set**: CSP `frame-ancestors 'none'` or `'self'`. Add `X-Frame-Options: DENY` for older browsers.
  - **Why it matters:** Prevents clickjacking on sensitive actions.

- [ ] **`X-Content-Type-Options: nosniff`**: On every response.
  - **Why it matters:** Stops browsers from treating uploaded files or JSON as HTML or script.

- [ ] **`Referrer-Policy`**: `strict-origin-when-cross-origin` or `no-referrer` on pages with tokens in URLs.
  - **Why it matters:** Full URLs leak to third parties through the `Referer` header.

- [ ] **`Permissions-Policy`**: Turn off features you do not use, for example `camera=(), microphone=(), geolocation=(), payment=()`.
  - **Why it matters:** Limits what a compromised third-party script can access.

- [ ] **Cross-origin isolation headers where needed**: `Cross-Origin-Opener-Policy: same-origin` for apps with sensitive windows. `Cross-Origin-Resource-Policy: same-origin` on private assets.
  - **Why it matters:** Reduces cross-window attacks and cross-origin data leaks through side channels.

- [ ] **No version disclosure headers**: Remove `Server` version strings, `X-Powered-By`, `X-AspNet-Version`.
  - **Why it matters:** Makes it harder to match your stack to known CVEs. Low value alone, but free.
  - **How to verify:** `curl -sI` on a page, an API route, a static file, and an error page.

## 4. Session management

- [ ] **Session ID regenerated on login and privilege change**: New ID after authentication, MFA, and role change. Old ID is invalidated.
  - **Why it matters:** Prevents session fixation.
  - **How to verify:** Note the session cookie before and after login. It must change.

- [ ] **Server-side logout**: Logout destroys the server-side session and revokes refresh tokens.
  - **Why it matters:** Clearing the cookie alone leaves a copied session usable.
  - **How to verify:** Save the session cookie, log out, replay a request with the saved cookie. It must fail.

- [ ] **Idle and absolute timeouts**: Idle timeout (15 to 30 minutes for admin or financial apps) and an absolute lifetime (8 to 24 hours).
  - **Why it matters:** Limits the window for stolen sessions and shared computers.

- [ ] **Re-authentication for sensitive actions**: Ask for password or MFA again before changing email, password, MFA settings, payout details, or deleting the account.
  - **Why it matters:** A hijacked session should not be enough to take over the account permanently.

- [ ] **Session list and revoke**: Users can see active sessions and sign out of other devices. Password change signs out other sessions.
  - **Why it matters:** Gives users and support a way to recover a compromised account.

- [ ] **No tokens in URLs**: Session IDs, reset tokens, and API keys never appear in query strings, except single-use tokens that are invalidated on first use.
  - **Why it matters:** URLs end up in logs, browser history, analytics, and `Referer` headers.

## 5. CSRF protection

- [ ] **CSRF tokens or equivalent on state-changing requests**: Framework anti-CSRF tokens on forms and cookie-authenticated API calls, in addition to `SameSite`.
  - **Why it matters:** `SameSite=Lax` does not cover every case (top-level GET with side effects, same-site subdomains).
  - **Secure default:** Framework built-in CSRF middleware. Synchronizer token or signed double-submit cookie.

- [ ] **No state changes on GET**: GET and HEAD are safe and idempotent.
  - **Why it matters:** GET requests are sent by links, image tags, and prefetchers.
  - **How to verify:** Grep your route table for GET handlers that write to the database.

- [ ] **Origin check on sensitive endpoints**: Validate `Origin` (or `Sec-Fetch-Site`) against an allowlist for state-changing requests.
  - **Why it matters:** Cheap second layer against cross-site requests.

## 6. CORS

- [ ] **CORS allowlist, not reflection**: `Access-Control-Allow-Origin` is set from a fixed allowlist. Never echo the request `Origin` blindly.
  - **Why it matters:** Reflected origin with `Allow-Credentials: true` lets any site read authenticated responses.
  - **Secure default:** Exact match on scheme, host, and port. No regex with unescaped dots or suffix matching.
  - **How to verify:** Send a request from a non-allowlisted origin in a test environment. The response must not include `Access-Control-Allow-Origin` for it.

- [ ] **No `null` origin in allowlist**: Do not allow `Origin: null`.
  - **Why it matters:** Sandboxed iframes and some redirects send `null`, and any site can produce it.

- [ ] **Credentials only when needed**: `Access-Control-Allow-Credentials: true` only on routes that need cookies cross-origin.

## 7. Input handling and injection prevention

- [ ] **Parameterized queries everywhere**: SQL uses prepared statements or an ORM's parameter binding. No string concatenation into queries, including `ORDER BY` and table names.
  - **Why it matters:** SQL injection is still common and often critical.
  - **Secure default:** ORM or query builder. Dynamic identifiers come from a hard-coded allowlist.
  - **How to verify:** SAST rule set for your language (Semgrep, CodeQL). Grep for raw query helpers.

- [ ] **No shell command construction from input**: Use library APIs instead of shelling out. If you must run a process, pass arguments as an array, never through a shell.
  - **Why it matters:** Prevents OS command injection.

- [ ] **Server-side validation on every input**: Type, length, range, format, and allowed values checked on the server. Client-side checks are UX only.
  - **Secure default:** Schema validation library (Zod, Pydantic, Joi, Bean Validation) at the controller boundary.

- [ ] **Safe deserialization**: No native object deserialization of untrusted data (Java `ObjectInputStream`, Python `pickle`, PHP `unserialize`, .NET `BinaryFormatter`).
  - **Why it matters:** Unsafe deserialization often leads to remote code execution.
  - **Secure default:** JSON with a strict schema.

- [ ] **XML parsers hardened**: External entities and DTDs disabled.
  - **Why it matters:** Prevents XXE (file read, SSRF).
  - **Secure default:** Follow the OWASP XXE Prevention Cheat Sheet for your parser.

- [ ] **Template engines not fed user input as templates**: Users supply data to templates, never template source.
  - **Why it matters:** Server-side template injection can lead to code execution.

- [ ] **Regex safe from catastrophic backtracking**: Review regexes applied to user input. Use linear-time engines (RE2) where possible, and cap input length.
  - **Why it matters:** ReDoS can take down a service with one request.

## 8. Output encoding and XSS prevention

- [ ] **Context-aware auto-escaping on**: Template engine escapes by default for HTML. Do not turn it off globally.
  - **Why it matters:** Most XSS comes from one place where escaping was skipped.

- [ ] **Raw HTML sinks reviewed**: Every use of `innerHTML`, `dangerouslySetInnerHTML`, `v-html`, `[innerHTML]`, `|safe`, `html_safe`, or `Html.Raw` is listed and justified.
  - **Secure default:** Sanitize with DOMPurify (or a server-side equivalent) using a tight allowlist.
  - **How to verify:** Grep for the sink names. Add a lint rule that flags new uses.

- [ ] **No user input in `href`, `src`, or event handlers without validation**: URLs from users are checked for `http:` or `https:` schemes only.
  - **Why it matters:** `javascript:` URLs run script even when HTML is escaped.

- [ ] **Trusted Types for DOM sinks**: Enable `require-trusted-types-for 'script'` in CSP on apps that can support it.
  - **Why it matters:** Makes DOM XSS sinks fail closed in supported browsers.

- [ ] **JSON responses served as JSON**: `Content-Type: application/json` with `nosniff`. Do not embed raw JSON inside `<script>` without escaping `<`, `>`, and `&`.

## 9. File uploads and downloads

- [ ] **Allowlist file types**: Check extension and content (magic bytes) against an allowlist. Do not trust the client `Content-Type`.
  - **Why it matters:** Unrestricted upload can lead to stored XSS, malware hosting, or code execution.

- [ ] **Size and count limits**: Enforce maximum file size, request size, and files per request at the proxy and in the app.

- [ ] **Store outside the web root**: Use object storage or a non-executable directory. Generate random file names. Keep the original name only as metadata.
  - **Why it matters:** Uploaded files should never be executed or served from your app origin as HTML.

- [ ] **Serve user files from a separate domain**: For example `usercontent.example-cdn.com`, with `Content-Disposition: attachment` for non-image types.
  - **Why it matters:** Any script that slips through runs on a domain without your session cookies.

- [ ] **Image processing isolated**: Image and document converters run in a sandbox with resource limits and patched libraries.
  - **Why it matters:** Image libraries (ImageMagick, libwebp, PDF renderers) have a long CVE history.

- [ ] **Archive extraction safe**: Check paths during unzip to prevent writing outside the target directory. Cap total extracted size and file count.
  - **Why it matters:** Prevents path traversal on extract ("zip slip") and zip bombs.

- [ ] **Download endpoints check authorization per file**: Every download checks the caller can access that specific file. Signed URLs are short-lived.

## 10. Server-side request handling

- [ ] **Outbound requests from user-supplied URLs are restricted**: Webhooks, URL previews, and import-by-URL features use an allowlist or block private, loopback, link-local, and metadata IP ranges after DNS resolution.
  - **Why it matters:** Prevents SSRF into internal services and cloud metadata.
  - **Secure default:** Dedicated egress proxy for user-driven fetches. Disable redirects or re-check each hop.

- [ ] **Path handling safe**: File paths built from input are normalized and checked to stay inside a base directory.
  - **Why it matters:** Prevents path traversal.

- [ ] **Open redirects removed**: Redirect targets are relative paths or come from an allowlist.
  - **Why it matters:** Open redirects help phishing and can leak OAuth codes.

## 11. Error handling

- [ ] **Generic error pages in production**: No stack traces, SQL errors, framework debug pages, or internal paths shown to users.
  - **Secure default:** Debug mode off in production config, enforced in CI.
  - **How to verify:** Trigger a 404, a 405, and a validation error. Check response bodies.

- [ ] **Correlation ID in errors**: Error responses include a request ID that maps to full server logs.

- [ ] **Fail closed**: If an authorization or validation check throws, the request is denied.

## 12. Third-party code and dependencies

- [ ] **Script inventory**: Every third-party script (analytics, chat, tag manager, A/B testing) is listed with an owner and purpose.
  - **Why it matters:** Each one can read every page it runs on, including forms with passwords and card data.

- [ ] **Subresource Integrity on static third-party files**: `integrity` and `crossorigin` attributes on CDN scripts and styles.

- [ ] **Tag managers restricted**: Limit who can publish tags. Require review for new tags. Keep them off login and payment pages.

- [ ] **Dependency scanning in CI**: `npm audit`, `pip-audit`, OSV-Scanner, or similar runs on every build. Critical findings fail the build.

## 13. Access control in the web layer

- [ ] **Server-side authorization on every request**: Hiding a button is not access control. Every route checks the caller's permissions.

- [ ] **Deny by default**: New routes require authentication unless explicitly marked public.
  - **Secure default:** Global auth middleware with an explicit public allowlist.
  - **How to verify:** List all routes and compare against the public allowlist.

- [ ] **Admin interfaces separated**: Admin panels on a separate host or path with network restrictions, SSO, and MFA.

## 14. Caching

- [ ] **No caching of authenticated responses at shared caches**: `Cache-Control: no-store` (or `private`) on pages with user data.
  - **Why it matters:** CDNs and proxies can serve one user's data to another.
  - **How to verify:** Check `Cache-Control` and CDN cache status headers on authenticated pages.

- [ ] **Cache keys include what matters**: If the CDN caches by path only, make sure responses do not vary by unkeyed headers like `Host` or `X-Forwarded-Host`.
  - **Why it matters:** Prevents cache poisoning and cache deception.
