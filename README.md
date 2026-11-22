<p align="center">
  <img src="assets/logo.svg" alt="AppSec Hardening Checklist" width="640">
</p>

<p align="center">
  <a href="https://github.com/Protector47/appsec-hardening-checklist/actions/workflows/ci.yml"><img src="https://github.com/Protector47/appsec-hardening-checklist/actions/workflows/ci.yml/badge.svg" alt="CI"></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-MIT-blue.svg" alt="MIT License"></a>
</p>

# AppSec Hardening Checklist

Practical hardening checklists for web apps, APIs, cloud, identity, CI/CD, logging, business logic, and Android and iOS apps, with mappings to the OWASP Top 10, OWASP API Top 10, and MASVS. Each item says what to check, why it matters, and what a secure default looks like.

## The problem

Most security reviews fail in the same places: a cookie without `HttpOnly`, an API that trusts the object ID in the URL, a CI job with an admin cloud key, logs that contain session tokens. Teams usually know these rules. They just do not have a short list to check against when it counts.

Standards like OWASP ASVS and the CIS Benchmarks are thorough but long. This repo is the shorter version you can actually get through in a design review or before a release. It links to the standards where you need more depth.

## Contents

### Core checklists

| Checklist | Covers |
| --- | --- |
| [Web apps](checklists/web-apps.md) | TLS, cookies, security headers, sessions, CSRF, CORS, injection, XSS, uploads, SSRF, caching |
| [APIs](checklists/apis.md) | Inventory, token validation, object and property level authorization, schemas, rate limits, GraphQL, webhooks |
| [Cloud](checklists/cloud.md) | Org guardrails, IAM, storage exposure, encryption, network, compute, Kubernetes, IaC, DNS |
| [Identity](checklists/identity.md) | OIDC and SAML, password storage, MFA, login protection, recovery, sessions, access governance |
| [CI/CD and supply chain](checklists/cicd-supply-chain.md) | Branch protection, workflow hardening, runners, dependencies, SLSA provenance, signing, SBOMs |
| [Logging and monitoring](checklists/logging-monitoring.md) | Events to log, data to keep out, log integrity, retention, detections, response readiness |
| [Business logic](checklists/business-logic.md) | Price and quantity tampering defenses, coupons, state machines, entitlements, race conditions, idempotency |

### OWASP mappings

| Checklist | Covers |
| --- | --- |
| [OWASP Top 10 (2021)](checklists/owasp-web-top10.md) | Defensive checks for A01 to A10 |
| [OWASP API Security Top 10 (2023)](checklists/owasp-api-top10.md) | Defensive checks for API1 to API10 |

### Mobile (OWASP MASVS v2 aligned)

| Checklist | Covers |
| --- | --- |
| [Mobile common](checklists/mobile-common.md) | Storage, crypto, auth, network, platform, code, resilience, and privacy controls for both platforms |
| [Android](checklists/mobile-android.md) | Keystore, backup rules, Network Security Config, exported components, WebView, App Links, Play Integrity |
| [iOS](checklists/mobile-ios.md) | Keychain, Secure Enclave, data protection classes, ATS, pasteboard, Universal Links, App Attest |

### Examples and guides

- [Sample web app review](examples/sample-web-review.md): a filled-in review with findings and follow-up tickets
- [Using in your SDLC](docs/using-in-sdlc.md): where these checklists fit in design reviews, PRs, and release gates

## Checklist format

Every item follows the same layout:

```markdown
- [ ] **Item name**: What to check.
  - **Why it matters:** The failure this prevents.
  - **Secure default:** A concrete setting or pattern to start from.
  - **How to verify:** A defensive check you can run on your own system.
```

Not every item needs all four lines. Short items in the OWASP and mobile lists skip what is obvious.

## How to use

1. Pick the checklists that match the system you are reviewing. A typical public web product needs web apps, APIs, identity, business logic, and logging. Add mobile and cloud as they apply.
2. Copy them into your repo or ticket, for example `docs/security/review-2026-11.md`.
3. Tick items that pass. For items that fail, open a ticket with an owner and due date.
4. Mark items that do not apply as N/A with a one-line reason.
5. Keep the completed review in version control so the next review has something to compare against.

See [examples/sample-web-review.md](examples/sample-web-review.md) for what a finished review looks like.

## Scope

This repo is defensive only. It covers configuration, design, and process controls. It does not include exploit code, attack payloads, or scanning tools.

## References

### OWASP web and API

- [OWASP Top 10 2021](https://owasp.org/Top10/): the ten most critical web application risk categories
- [OWASP API Security Top 10 2023](https://owasp.org/API-Security/editions/2023/en/0x11-t10/): the API-specific list
- [OWASP ASVS](https://owasp.org/www-project-application-security-verification-standard/): detailed verification requirements, useful when you need formal levels (L1 to L3)
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/): implementation guidance per topic

### OWASP mobile

- [OWASP MASVS](https://mas.owasp.org/MASVS/): the mobile security requirements standard. v2 groups controls into STORAGE, CRYPTO, AUTH, NETWORK, PLATFORM, CODE, RESILIENCE, and PRIVACY. The mobile checklists here follow those groups.
- [OWASP MASTG](https://mas.owasp.org/MASTG/): the testing guide that explains how to verify each MASVS control
- [OWASP MAS Checklist](https://mas.owasp.org/checklists/): MASVS controls mapped to MASTG tests

### Other standards

- [CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks)
- [NIST SP 800-63B](https://pages.nist.gov/800-63-4/sp800-63b.html)
- [SLSA](https://slsa.dev/)
- [OpenSSF Scorecard](https://securityscorecards.dev/)
- [OWASP Top 10 CI/CD Security Risks](https://owasp.org/www-project-top-10-ci-cd-security-risks/)

## Contributing

Fixes and additions are welcome. Read [CONTRIBUTING.md](CONTRIBUTING.md) first.

## License

[MIT](LICENSE)

---

Maintained by [Protector47](https://github.com/Protector47) · <https://www.protector47.com/>
