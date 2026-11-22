# Identity and Access Hardening Checklist

Covers user authentication, SSO, MFA, sessions, and admin access for applications and internal tools.

## Authentication

- [ ] **Use a central IdP**: Apps authenticate through OIDC or SAML against one identity provider instead of local password stores.
  - **Why it matters:** One place to enforce MFA, disable users, and review access.
  - **Secure default:** OIDC with PKCE for public clients. SAML only where OIDC is not supported.

- [ ] **Password storage done right**: If you must store passwords, use Argon2id, scrypt, or bcrypt with a per-user salt.
  - **Why it matters:** Fast hashes (MD5, SHA-1, unsalted SHA-256) are cracked quickly after a database leak.
  - **Secure default:** Argon2id with parameters from the OWASP Password Storage Cheat Sheet.

- [ ] **Password policy follows NIST 800-63B**: Minimum 8 characters (15 for single-factor), check against breached password lists, no forced periodic rotation.
  - **Why it matters:** Complexity rules and rotation push users toward weak, predictable passwords.
  - **Secure default:** Length over complexity. Block known breached passwords.

- [ ] **MFA enforced for admins and sensitive actions**: Admins always use MFA. Users are prompted again for high-risk actions like changing email or payout details.
  - **Why it matters:** Most account takeovers start with a stolen or reused password.
  - **Secure default:** WebAuthn or passkeys first, TOTP as fallback, SMS only as last resort.

## Account recovery

- [ ] **Recovery is as strong as login**: Reset links are single-use, short-lived, and sent only to verified channels.
  - **Why it matters:** A weak reset flow bypasses every other control.
  - **Secure default:** Tokens expire in 15 to 60 minutes. Invalidate on use and on password change.

- [ ] **No account enumeration**: Login, signup, and reset return the same response whether or not the account exists.
  - **Why it matters:** Enumeration feeds credential stuffing and phishing.
  - **Secure default:** Generic message such as "If an account exists, we sent an email."

## Sessions and tokens

- [ ] **Session IDs rotate on privilege change**: New session ID after login, MFA, and role change.
  - **Why it matters:** Prevents session fixation.
  - **Secure default:** Framework session regeneration on every auth state change.

- [ ] **Idle and absolute timeouts**: Sessions expire after inactivity and after a maximum lifetime.
  - **Why it matters:** Long sessions on shared or stolen devices stay usable.
  - **Secure default:** 15 to 30 minutes idle for admin apps. 8 to 24 hours absolute, depending on risk.

- [ ] **Refresh tokens rotated and revocable**: Each use issues a new refresh token. Reuse of an old one revokes the chain.
  - **Why it matters:** Stolen refresh tokens otherwise give long-term access.
  - **Secure default:** Refresh token rotation with reuse detection.

## Authorization and admin access

- [ ] **Role model documented**: Roles and permissions are written down and reviewed. Avoid one "admin" role that does everything.
  - **Why it matters:** Undocumented roles drift into over-privilege.
  - **Secure default:** RBAC or ABAC with a policy file in version control.

- [ ] **Joiner, mover, leaver process**: Access is granted through tickets or groups and removed when people change teams or leave.
  - **Why it matters:** Orphaned accounts are an easy way in.
  - **Secure default:** SCIM provisioning from the IdP. Quarterly access reviews for privileged roles.

- [ ] **Service accounts owned and scoped**: Every service account has an owner, a purpose, and the minimum permissions.
  - **Why it matters:** Unowned service accounts keep old, broad access for years.
  - **Secure default:** Tag with owner and expiry. Alert on unused accounts after 90 days.
