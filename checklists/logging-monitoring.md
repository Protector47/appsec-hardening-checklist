# Logging and Monitoring Checklist

What to log, what to keep out of logs, how to protect logs, and what to alert on. Aligned with the OWASP Logging Cheat Sheet and OWASP Top 10 A09.

## 1. Security events to log

- [ ] **Authentication**: Login success and failure, logout, MFA challenge and result, MFA enrollment and removal, passkey registration.
  - **Fields:** user ID, auth method, result, failure reason (internal), source IP, user agent, device ID, timestamp.

- [ ] **Account changes**: Password change and reset request, email change, phone change, recovery code use, account deletion.

- [ ] **Authorization failures**: Every 403 with caller, route, action, and target object ID.
  - **Why it matters:** Repeated denials across many object IDs are the clearest sign of someone probing access control.

- [ ] **Privilege changes**: Role grants and removals, group membership changes, API key creation and revocation, impersonation start and end.

- [ ] **Sensitive data access**: Exports, bulk downloads, report generation, viewing of highly sensitive records (health, financial, identity documents).

- [ ] **Admin and config changes**: Feature flag changes, security settings, SSO config, webhook endpoints, integrations.

- [ ] **Input validation and control events**: Schema validation rejects, WAF blocks, rate limit hits, CSRF failures, file upload rejections, CSP violation reports.

- [ ] **Business events with fraud risk**: Payouts, refunds, gift card redemptions, coupon use, changes to bank details.

- [ ] **Application errors**: Unhandled exceptions with stack traces (server-side only) and correlation ID.

- [ ] **System events**: Deploys, config changes, scaling events, service start and stop.

## 2. Data that must not be logged

- [ ] **No credentials**: Passwords (including failed attempts), OTPs, recovery codes, private keys.

- [ ] **No tokens**: Session IDs, access and refresh tokens, API keys, signed URLs, reset tokens. Log a hash or a short prefix if you need correlation.

- [ ] **No full payment data**: Card numbers, CVV, full bank account numbers.

- [ ] **PII minimized**: Prefer internal user IDs. Mask emails and phone numbers if needed.

- [ ] **Request and response bodies off by default**: If enabled for debugging, use field-level redaction and auto-expire the setting.

- [ ] **Redaction tested**: Unit tests feed known secret formats through the logger and assert they are masked.
  - **How to verify:** Search a sample of production logs for `Authorization:`, `password`, `Bearer` tokens, and known key prefixes.

- [ ] **Header logging filtered**: `Authorization`, `Cookie`, `Set-Cookie`, and `X-Api-Key` stripped in access logs at the proxy and app.

## 3. Format and context

- [ ] **Structured JSON logs**.

- [ ] **Common schema across services**: Same field names for user ID, tenant ID, request ID, source IP. Consider OpenTelemetry semantic conventions or Elastic Common Schema.

- [ ] **Event type field**: Each security event has a stable name (`auth.login.failure`, `authz.denied`, `user.role.granted`) for queries and alerts.

- [ ] **Correlation and trace IDs**: W3C `traceparent` propagated across services, queues, and background jobs.

- [ ] **Real client IP**: Taken from a trusted proxy header and only from proxies you control. Do not trust `X-Forwarded-For` from the internet.

- [ ] **UTC timestamps in ISO 8601** with millisecond precision. NTP on all hosts.

- [ ] **Tenant context**: Multi-tenant apps log tenant ID on every event.

## 4. Integrity and protection

- [ ] **Log injection prevented**: Structured logging escapes values. If you write text logs, strip CR, LF, and control characters from user input.

- [ ] **Shipped off-host quickly**: Agents forward logs within seconds to a central store.

- [ ] **Write-only from producers**: Apps and hosts can append but not modify or delete.

- [ ] **Immutable storage for audit logs**: Object lock or WORM storage in a separate account.

- [ ] **Access to logs restricted and logged**: Logs contain sensitive context. Reading them is audited.

- [ ] **Log pipeline failures alert**: Missing logs from a service for N minutes raises an alert.
  - **Why it matters:** Attackers disable logging. Broken agents look the same as a quiet system.

## 5. Retention

- [ ] **Retention policy written**: Per log type, based on investigation needs and regulation (PCI DSS requires 1 year, 3 months immediately available).

- [ ] **Hot and archive tiers**: For example 90 days searchable, 1 year or more archived.

- [ ] **Deletion follows the policy**: Logs past retention are deleted, especially those with personal data.

## 6. Detection and alerting

- [ ] **Brute force and stuffing**: Many failed logins for one account, or one source failing across many accounts.

- [ ] **Account takeover signals**: Login from a new country followed by email or MFA change, password reset followed by payout detail change.

- [ ] **Access control probing**: One user hitting many 403s or 404s on object routes in a short window.

- [ ] **Privilege changes**: Any new admin, any role grant outside the approval workflow.

- [ ] **Mass data access**: Exports or reads far above the user's normal baseline.

- [ ] **Security control changes**: MFA disabled, SSO config changed, audit logging disabled, WAF rule disabled.

- [ ] **Error rate spikes**: Sudden 5xx increases on specific routes, which may indicate injection attempts or DoS.

- [ ] **Canary tokens**: Decoy credentials or records that alert when used.

- [ ] **Each alert has an owner, a severity, and a runbook**.

- [ ] **Alert quality reviewed monthly**: Tune or remove alerts that fire without action.

## 7. Response readiness

- [ ] **Runbooks**: Leaked credential, compromised user account, compromised admin, data exposure, malicious dependency, ransomware.

- [ ] **On-call path**: Security alerts route to a person 24/7 for critical severity.

- [ ] **Forensic readiness**: You can answer who, what, when, and from where for any sensitive action in the last 90 days.

- [ ] **Tabletop exercises**: At least once a year, using real log queries.

- [ ] **Post-incident review**: Every incident produces new detections or log fields where gaps were found.
