# Using These Checklists in Your SDLC

The checklists only help if they show up at the points where decisions get made. This is how we suggest wiring them in.

## Design review

Pick the checklists that match the system. A new public API usually needs `apis.md`, `identity.md`, and `logging-monitoring.md`. Go through the items with the team before code is written. Anything marked "not applicable" should have a one-line reason.

## Pull requests

Do not paste a full checklist into every PR. Instead:

1. Add a short security section to your PR template with three or four questions tied to the checklists (for example, "Does this change add a new endpoint? If so, is object-level authorization checked?").
2. Use CODEOWNERS so changes to auth, crypto, and CI config get a security reviewer.

## Pre-release gate

Before a major release or launch, run the relevant checklist end to end. Use `examples/sample-web-review.md` as the format. Record blockers as tickets with an owner and due date. Ship only when blockers are closed or have a signed risk acceptance.

## Periodic review

Systems drift. Re-run the cloud and CI/CD checklists every quarter, and after any big infrastructure change. Compare against the last review and look at what changed.

## Tracking

Keep completed reviews in version control next to the service, for example `docs/security/review-2026-11.md`. That gives you a history, and auditors can see it without extra work.

## Mapping to standards

These checklists are practical notes, not a replacement for a standard. If you need formal coverage, map items to:

- [OWASP ASVS](https://owasp.org/www-project-application-security-verification-standard/) for application controls
- [CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks) for cloud and OS settings
- [NIST SP 800-63B](https://pages.nist.gov/800-63-4/sp800-63b.html) for authentication
- [SLSA](https://slsa.dev/) for build and supply chain integrity
