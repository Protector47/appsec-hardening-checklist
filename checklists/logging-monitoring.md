# Logging and Monitoring Checklist

What to log, how to protect it, and what to alert on. Use it during design reviews and before go-live.

## What to log

- [ ] **Authentication events**: Log successful and failed logins, MFA challenges, password resets, and lockouts.
  - **Why it matters:** These are the first signals of credential stuffing and account takeover.
  - **Secure default:** Include user ID, source IP, user agent, result, and timestamp in UTC.

- [ ] **Authorization failures**: Log denied access attempts with the resource and action requested.
  - **Why it matters:** Repeated 403s against different object IDs usually mean someone is probing.
  - **Secure default:** Structured log entry per denial, sampled only if volume is extreme.

- [ ] **Admin and sensitive actions**: Role changes, permission grants, config changes, data exports, and key operations.
  - **Why it matters:** You need to answer "who changed this and when" during an incident.
  - **Secure default:** Append-only audit log separate from app debug logs.

- [ ] **Input validation and security control events**: WAF blocks, CSP reports, rate limit hits, and rejected uploads.
  - **Why it matters:** Shows which controls are firing and whether they are tuned right.
  - **Secure default:** Route to the same pipeline as app logs with a clear event type.

## What not to log

- [ ] **No secrets or credentials in logs**: Passwords, tokens, API keys, session IDs, and private keys are never logged.
  - **Why it matters:** Logs get wide read access and long retention. Secrets in logs are leaked secrets.
  - **Secure default:** Redaction filters in the logging library. Test them in CI.

- [ ] **PII minimized**: Log user IDs instead of names, emails, or card numbers. Mask what you must keep.
  - **Why it matters:** Logs fall under privacy rules (GDPR and similar) like any other data store.
  - **Secure default:** Documented field allowlist for log events.

## Log format and integrity

- [ ] **Structured logs**: JSON or another structured format with consistent field names.
  - **Why it matters:** Free-text logs are slow to search and hard to alert on.
  - **Secure default:** Shared schema across services (for example, ECS or OpenTelemetry semantic conventions).

- [ ] **Correlation IDs**: Every request carries a trace or request ID across services.
  - **Why it matters:** Lets you follow one request through the whole system during an incident.
  - **Secure default:** W3C Trace Context headers, propagated by default.

- [ ] **Log injection prevented**: Encode or strip newlines and control characters in user-supplied values.
  - **Why it matters:** Attackers can forge log lines or break parsers.
  - **Secure default:** Structured logging libraries that escape values.

- [ ] **Logs shipped off-host and protected**: Logs go to a central store that app servers cannot modify or delete.
  - **Why it matters:** An attacker on the box will try to clean up after themselves.
  - **Secure default:** Write-only shipping to a SIEM or log account with restricted delete rights.

- [ ] **Retention defined**: Retention matches investigation needs and legal requirements.
  - **Why it matters:** Many breaches are found months later.
  - **Secure default:** At least 90 days hot and 1 year archived for security-relevant logs.

## Alerting and response

- [ ] **Alerts for high-signal events**: Brute force patterns, admin role grants, MFA disabled, new API keys, and spikes in 401/403/5xx.
  - **Why it matters:** Logs nobody reads do not stop incidents.
  - **Secure default:** Each alert has an owner, a severity, and a runbook link.

- [ ] **Alert noise reviewed**: Tune or remove alerts that fire often without action.
  - **Why it matters:** Noisy alerts get ignored, including the real ones.
  - **Secure default:** Monthly review of alert volume and false positive rate.

- [ ] **Clock sync**: All hosts use NTP and log in UTC.
  - **Why it matters:** Skewed timestamps make cross-system timelines unreliable.
  - **Secure default:** Managed NTP on all hosts and containers.

- [ ] **Incident runbooks exist and are tested**: Common scenarios (leaked key, compromised account, data exposure) have written steps.
  - **Why it matters:** People make fewer mistakes under pressure with a checklist.
  - **Secure default:** Tabletop exercise at least once a year.
