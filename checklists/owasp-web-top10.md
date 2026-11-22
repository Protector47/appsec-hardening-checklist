# OWASP Top 10 (2021) Defensive Checklist

Defensive checks mapped to each [OWASP Top 10 2021](https://owasp.org/Top10/) category. Use it to show coverage in reviews and audits. For deeper checks, follow the links to the topic checklists in this repo.

## A01:2021 Broken Access Control

- [ ] **Deny by default**: Every route requires authentication and an authorization decision unless it is on an explicit public list.
  - **Why it matters:** Access control is the most common serious finding in web apps.
  - **How to verify:** Export the route table and diff it against the public list in CI.

- [ ] **Object ownership checked server-side**: Queries are scoped to the caller's user or tenant. Never trust an ID or `owner` field sent by the client.
  - **How to verify:** Integration tests with two users. Each tries to read, update, and delete the other's objects. All attempts fail.

- [ ] **Role checks in a central layer**: Middleware, decorators, or a policy engine. Not scattered `if user.isAdmin` checks.

- [ ] **No access control decisions from client-supplied fields**: Hidden form fields, cookies, or headers like `X-User-Role` are not trusted.

- [ ] **CORS limited**: Allowlisted origins only. See [web-apps.md](web-apps.md#6-cors).

- [ ] **Directory listing off**: Web server does not list directories. `.git`, `.env`, backups, and source maps are not served from production.
  - **How to verify:** Request `/.git/HEAD` and `/.env` against your own production host. Both should return 404.

- [ ] **Rate limits on access-controlled APIs**: Limits slow down automated enumeration.

- [ ] **JWT and session invalidation on logout**: Stateless tokens are short-lived. Server-side sessions are destroyed.

- [ ] **Access control failures logged and alerted**.

- [ ] **Path traversal prevented**: Paths from input are resolved and checked against a base directory.

- [ ] **SSRF controls**: Covered under A10 below. Review them together with access control, since SSRF often reaches internal admin endpoints.

## A02:2021 Cryptographic Failures

- [ ] **Data classified**: You know which fields are sensitive (credentials, PII, financial, health) and where they are stored and sent.

- [ ] **No sensitive data stored without need**: Delete it or tokenize it. Data you do not keep cannot leak.

- [ ] **Encryption in transit everywhere**: TLS 1.2+, HSTS, no plaintext internal hops for sensitive data.

- [ ] **Encryption at rest**: Databases, backups, and object storage encrypted. Field-level encryption for the most sensitive fields.

- [ ] **Modern algorithms only**: AES-GCM or ChaCha20-Poly1305 for symmetric encryption. RSA 2048+ or ECDSA P-256 / Ed25519 for signatures. SHA-256 or better for hashing. No MD5, SHA-1, DES, 3DES, RC4, or ECB mode.
  - **How to verify:** SAST rules for weak crypto APIs.

- [ ] **Passwords hashed with a password hashing function**: Argon2id, scrypt, or bcrypt. See [identity.md](identity.md#2-passwords).

- [ ] **Randomness from a CSPRNG**: `crypto.randomBytes`, `secrets`, `SecureRandom`. Not `Math.random` or `random`.

- [ ] **IVs and nonces never reused**: Use library defaults that generate them for you.

- [ ] **Keys managed by a KMS or vault**: Not in source, config files, or images. Rotation documented.

- [ ] **Caching disabled for sensitive responses**: `Cache-Control: no-store`.

- [ ] **Certificate validation not disabled**: No `verify=False`, `rejectUnauthorized: false`, or trust-all managers in code.
  - **How to verify:** Grep and SAST for these patterns.

## A03:2021 Injection

- [ ] **Parameterized queries for SQL**: Including stored procedures called with dynamic SQL.

- [ ] **Allowlists for dynamic identifiers**: Sort columns, table names, and directions come from a fixed list.

- [ ] **NoSQL queries built safely**: Reject objects where scalars are expected so operators like `$ne` or `$where` cannot be injected. Disable server-side JavaScript in MongoDB.

- [ ] **No OS command construction**: Use libraries. If unavoidable, `execFile` / `subprocess.run([...], shell=False)` with argument arrays.

- [ ] **LDAP queries escaped**: Use the library's filter encoding.

- [ ] **Output encoding for XSS**: Context-aware escaping, sanitizer for rich text, strict CSP. XSS is grouped under injection in 2021.

- [ ] **Template injection prevented**: User input is data, never template source.

- [ ] **Expression language and eval avoided**: No `eval`, `Function()`, SpEL, OGNL, or similar on user input.

- [ ] **Header injection prevented**: Strip CR and LF from values written to HTTP headers and email headers.

- [ ] **Server-side input validation**: Allowlist validation of type, length, format, and range.

- [ ] **SAST and code review**: Injection rules enabled in SAST. Reviewers check new query and command code.

- [ ] **Least privilege DB accounts**: App users cannot drop tables or read other schemas. Limits blast radius.

## A04:2021 Insecure Design

- [ ] **Threat modeling for new features**: A short threat model (STRIDE or a simple "what can go wrong" session) for anything touching auth, payments, or sensitive data.

- [ ] **Security requirements written**: Each feature lists its abuse cases and the controls that address them.

- [ ] **Secure design patterns reused**: Shared libraries for auth, authorization, input validation, and file handling.

- [ ] **Business logic limits**: Rate limits, quantity limits, and per-user quotas designed in. See [business-logic.md](business-logic.md).

- [ ] **Tenant separation designed in**: Decided at the data layer from day one.

- [ ] **Abuse case tests**: Integration tests for misuse paths, not only happy paths.

- [ ] **Credential recovery designed securely**: No security questions. See [identity.md](identity.md#5-account-recovery).

- [ ] **Segregated tiers**: Frontend, API, and data tiers separated by network and identity boundaries.

## A05:2021 Security Misconfiguration

- [ ] **Hardened, repeatable config**: Environments built from IaC with the same security settings. Differences are intentional and documented.

- [ ] **Debug off in production**: Framework debug modes, verbose errors, and admin consoles disabled.
  - **How to verify:** CI check on production config. Trigger an error in prod and inspect the response.

- [ ] **Default accounts removed**: No default credentials on databases, admin panels, or appliances.

- [ ] **Unused features removed**: Sample apps, unused routes, unused ports, unused HTTP methods (`TRACE`).

- [ ] **Security headers set**: See [web-apps.md](web-apps.md#3-security-headers).

- [ ] **Cloud storage permissions checked**: See [cloud.md](cloud.md#3-object-storage).

- [ ] **XML parsers hardened**: XXE is in this category for 2021. Disable DTDs and external entities.

- [ ] **Automated config scanning**: IaC scanning in CI and posture monitoring in the cloud.

- [ ] **Error handling uniform**: Same generic format for all errors.

## A06:2021 Vulnerable and Outdated Components

- [ ] **Inventory of components**: SBOM for each service, including transitive dependencies and base images.

- [ ] **Continuous scanning**: SCA in CI and on a schedule for deployed versions.

- [ ] **Patch SLAs**: Defined and tracked by severity.

- [ ] **Unmaintained components replaced**: Track end-of-life dates for frameworks, runtimes, and OS versions.

- [ ] **Components from official sources**: Signed packages from official registries over secure links.

- [ ] **Unused dependencies removed**: Fewer packages means fewer CVEs to track.

- [ ] **Runtime versions supported**: Node.js, Python, Java, .NET, and PHP versions still receive security updates.

## A07:2021 Identification and Authentication Failures

- [ ] **MFA available and required for privileged users**.

- [ ] **Credential stuffing defenses**: Rate limits, breached password checks, bot detection.

- [ ] **No default or weak passwords**: Length rules and breach list checks.

- [ ] **Uniform responses**: No user enumeration on login, signup, or reset.

- [ ] **Secure session management**: Rotation on login, timeouts, server-side invalidation.

- [ ] **Session IDs not in URLs**.

- [ ] **Secure password storage**: Argon2id, scrypt, or bcrypt.

- [ ] **Failed logins logged and alerted**.

## A08:2021 Software and Data Integrity Failures

- [ ] **Signed artifacts and updates**: Verify signatures on packages, container images, and app updates.

- [ ] **Dependencies from trusted registries**: Private proxy, lockfiles with integrity hashes.

- [ ] **CI/CD pipeline protected**: See [cicd-supply-chain.md](cicd-supply-chain.md).

- [ ] **No unsafe deserialization**: Untrusted data parsed as JSON with schemas. No native object deserialization.

- [ ] **Integrity on client-side state**: Data sent to the client and returned (cookies, hidden fields, JWTs) is signed and verified, or kept server-side.

- [ ] **Subresource Integrity on CDN assets**.

- [ ] **Auto-update channels secured**: Plugin and update mechanisms verify signatures before installing.

## A09:2021 Security Logging and Monitoring Failures

- [ ] **Security events logged**: Logins, failures, access control denials, high-value transactions.

- [ ] **Logs in a format your SIEM can parse**.

- [ ] **Log injection prevented**: Values encoded.

- [ ] **Audit trail for high-value actions**: Append-only, tamper-evident.

- [ ] **Alerting with owners and runbooks**.

- [ ] **Incident response plan exists and is tested**.

Full detail in [logging-monitoring.md](logging-monitoring.md).

## A10:2021 Server-Side Request Forgery (SSRF)

- [ ] **Avoid fetching user-supplied URLs**: Where possible, accept file uploads or IDs from an allowlist instead of URLs.

- [ ] **Allowlist destinations**: If users choose a URL, restrict scheme to `https`, and restrict host and port to an allowlist where you can.

- [ ] **Block internal ranges after DNS resolution**: Loopback, RFC 1918, link-local (`169.254.0.0/16`), IPv6 equivalents, and cloud metadata addresses. Check the resolved IP, then connect to that same IP.
  - **Why it matters:** Checking the hostname only can be defeated by DNS changes between check and use.

- [ ] **Redirects handled**: Disable redirects or re-validate every hop.

- [ ] **Network-level egress control**: Fetchers run in a segment with no route to internal services or metadata. This is the control that still works when app checks miss something.

- [ ] **IMDSv2 and metadata header requirements**: See [cloud.md](cloud.md#6-compute).

- [ ] **Raw responses not returned to users**: Return only the parsed fields you need.

- [ ] **Outbound requests logged**: Destination, resolved IP, and initiating user.
