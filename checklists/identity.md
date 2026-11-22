# Identity and Access Hardening Checklist

User authentication, SSO, MFA, sessions, account recovery, and access governance. Aligned with NIST SP 800-63B and the OWASP Authentication, Session Management, and Forgot Password cheat sheets.

## 1. Identity architecture

- [ ] **One identity provider for workforce apps**: Internal tools use SSO through your IdP (Entra ID, Okta, Google Workspace). No local accounts.
  - **Why it matters:** One place to enforce MFA, disable leavers, and audit access.

- [ ] **Customer identity handled by a vetted service or library**: Auth0, Cognito, Entra External ID, Keycloak, or a mature framework module. Avoid writing auth from scratch.

- [ ] **OIDC done correctly**: Authorization code flow with PKCE. Validate `state` and `nonce`. Exact-match redirect URIs, no wildcards.
  - **Why it matters:** Loose redirect URI matching leaks authorization codes.
  - **How to verify:** Review registered redirect URIs in the IdP for every client.

- [ ] **SAML hardened**: Validate signatures on the assertion (not only the response), check audience, recipient, and `NotOnOrAfter`. Use a maintained library. Reject unsigned assertions.
  - **Why it matters:** SAML signature handling bugs are a common source of auth bypass.

- [ ] **Account linking is explicit**: Do not auto-merge accounts just because an external IdP returns the same email. Require the user to prove ownership of the existing account.
  - **Why it matters:** Some IdPs let users set unverified emails.

## 2. Passwords

- [ ] **Password hashing**: Argon2id (preferred), scrypt, or bcrypt (cost 10 or higher), with a unique salt per password. PBKDF2 with a high iteration count only where FIPS requires it.
  - **How to verify:** Inspect the stored hash format in a test database (`$argon2id$`, `$2b$`).

- [ ] **Length over complexity**: Minimum 8 characters with MFA, 15 without. Allow at least 64 characters. Allow all printable characters and spaces.

- [ ] **Breached password check**: Reject passwords found in breach corpora (Have I Been Pwned k-anonymity API or a local list).

- [ ] **No forced periodic rotation**: Require change only on evidence of compromise.

- [ ] **No password hints or knowledge-based questions**.

- [ ] **Paste and password managers allowed**: Do not block paste in password fields. Use correct `autocomplete` attributes (`current-password`, `new-password`).

- [ ] **Password change requires the current password** (or a recent re-auth).

## 3. Multi-factor authentication

- [ ] **MFA required for admins and staff**: No exceptions for production access.

- [ ] **MFA available to all customers**: Required for high-risk roles (finance, org admins).

- [ ] **Phishing-resistant factors preferred**: WebAuthn security keys and passkeys first. TOTP next. SMS and email OTP only as fallback.
  - **Why it matters:** OTP codes can be relayed by phishing proxies. WebAuthn is bound to the origin.

- [ ] **OTP handling**: Codes single-use, expire in a few minutes, and attempts are rate limited per account.
  - **How to verify:** Submit a used code in a test environment. It must fail.

- [ ] **Push MFA has number matching**: Prevents approval fatigue attacks.

- [ ] **Recovery codes**: One-time codes, stored hashed, shown once. Using one notifies the user.

- [ ] **MFA changes protected**: Adding or removing a factor requires re-auth and sends a notification to the user.

## 4. Login protection

- [ ] **Credential stuffing defenses**: Rate limits per account and per IP, bot detection on login, and checks for known breached credentials.

- [ ] **Lockout that does not enable denial of service**: Prefer progressive delays and CAPTCHA over hard lockout. If you lock, lock for a short period and notify the user.

- [ ] **No user enumeration**: Same message, status code, and similar response time for unknown user and wrong password. Same on signup and reset.
  - **How to verify:** Compare responses for a real and a non-existent account in a test environment.

- [ ] **New device and location alerts**: Email the user on sign-in from a new device.

## 5. Account recovery

- [ ] **Reset tokens are random and single-use**: At least 128 bits from a CSPRNG. Stored hashed. Expire in 15 to 60 minutes.

- [ ] **Reset link host is fixed**: Build reset URLs from server config, never from the request `Host` header.
  - **Why it matters:** Host header poisoning can send the reset link to an attacker's domain.

- [ ] **Reset invalidates sessions**: After a reset, all sessions and refresh tokens are revoked.

- [ ] **Reset does not log the user in automatically when MFA is on**: The user still completes MFA.

- [ ] **Email change verified on both addresses**: Confirm the new address and notify the old one with a way to revert.

- [ ] **Support-assisted recovery has a script**: Helpdesk verifies identity with defined steps. No resets based on caller knowledge of public data.

## 6. Sessions and tokens

- [ ] **Session ID entropy**: At least 128 bits from a CSPRNG.

- [ ] **Session rotation**: New ID on login, MFA, and privilege change.

- [ ] **Timeouts**: Idle and absolute timeouts appropriate to risk.

- [ ] **Server-side revocation**: Logout, password reset, and admin disable revoke sessions immediately.

- [ ] **Refresh token rotation with reuse detection**: Reuse of a rotated token revokes the whole token family.

- [ ] **Token binding where available**: DPoP or mTLS-bound tokens for high-value APIs.

- [ ] **Browser token storage**: Prefer HttpOnly cookies or a backend-for-frontend over storing tokens in `localStorage`.
  - **Why it matters:** Any XSS can read `localStorage`.

## 7. Authorization model

- [ ] **Roles and permissions documented**: Written role matrix, kept in version control.

- [ ] **Central policy enforcement**: One authorization layer or policy engine (OPA, Cedar, Casbin, framework policies).

- [ ] **Least privilege defaults**: New users get the minimum role. Elevation needs approval.

- [ ] **Separation of duties**: The person who requests a payout cannot approve it. Admins cannot approve their own access.

- [ ] **Just-in-time admin access**: Time-bound elevation with approval and logging (PIM, Teleport, or similar) instead of standing admin rights.

- [ ] **Impersonation features audited**: "Login as user" for support is logged, time-limited, visible to the user, and blocked for sensitive actions.

## 8. Lifecycle and governance

- [ ] **Joiner, mover, leaver automation**: SCIM provisioning from HR or the IdP. Access removed within 24 hours of departure.

- [ ] **Quarterly access reviews**: Owners review who has privileged access to each system.

- [ ] **Service accounts inventoried**: Owner, purpose, scope, credential type, and rotation date for each.

- [ ] **Shared accounts eliminated**: Where unavoidable, stored in a vault with check-out logging.

- [ ] **Dormant accounts disabled**: Accounts unused for 90 days are disabled after notice.

## 9. Monitoring

- [ ] **Auth events logged**: Logins, failures, MFA events, resets, role changes, token issuance.

- [ ] **Alerts**: Impossible travel, many failures across accounts from one source, MFA disabled on an admin, new admin role grants.
