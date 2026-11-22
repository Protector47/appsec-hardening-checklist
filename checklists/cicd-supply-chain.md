# CI/CD and Supply Chain Hardening Checklist

For build pipelines, source control, package dependencies, and release artifacts.

## Source control

- [ ] **Protected default branch**: Require pull requests, at least one review, and passing checks before merge. No force pushes.
  - **Why it matters:** A single compromised account should not be able to push straight to prod.
  - **Secure default:** Branch protection or rulesets on `main`, with CODEOWNERS for sensitive paths.

- [ ] **MFA and SSO for the org**: Everyone with write access uses MFA. Org access goes through SSO where available.
  - **Why it matters:** Developer accounts are a direct path to production code.
  - **Secure default:** Org-wide MFA requirement. Remove inactive members.

- [ ] **Secret scanning and push protection on**: Commits with credentials are blocked or flagged.
  - **Why it matters:** Leaked keys in git history get found and used fast.
  - **Secure default:** Native secret scanning with push protection, plus a pre-commit hook such as gitleaks.

## Pipeline security

- [ ] **No long-lived cloud keys in CI**: Pipelines get cloud access through OIDC federation with short-lived tokens.
  - **Why it matters:** CI secrets leak through logs, forks, and compromised actions.
  - **Secure default:** GitHub Actions OIDC to AWS/Azure/GCP, scoped to repo and branch.

- [ ] **Minimal token permissions**: The default workflow token is read-only. Jobs request only what they need.
  - **Why it matters:** A compromised step inherits whatever the token can do.
  - **Secure default:** `permissions: contents: read` at workflow level, raised per job.

- [ ] **Third-party actions pinned**: Pin actions and reusable workflows to a full commit SHA, not a moving tag.
  - **Why it matters:** A retagged or hijacked action runs in your pipeline with your secrets.
  - **Secure default:** SHA pinning, with Dependabot or Renovate to update pins.

- [ ] **Untrusted PRs isolated**: Workflows on forked PRs do not get secrets. Avoid `pull_request_target` with checkout of PR code.
  - **Why it matters:** Outside contributors can modify workflow steps in their PR.
  - **Secure default:** `pull_request` trigger for untrusted code. Manual approval for first-time contributors.

- [ ] **Self-hosted runners treated as sensitive**: Ephemeral runners, no reuse across repos, no runners on public repos.
  - **Why it matters:** Persistent runners can be backdoored by one malicious job.
  - **Secure default:** Ephemeral, autoscaled runners in an isolated network.

## Dependencies

- [ ] **Lockfiles committed and enforced**: Builds install from lockfiles (`npm ci`, `pip install --require-hashes`, `go.sum`).
  - **Why it matters:** Floating versions pull in whatever was published last night.
  - **Secure default:** Lockfile required in CI. Fail if it is out of date.

- [ ] **Automated dependency updates and CVE checks**: Dependabot, Renovate, or similar opens update PRs. CI flags known vulnerable packages.
  - **Why it matters:** Most dependency CVEs already have a fix. The gap is applying it.
  - **Secure default:** Patch critical CVEs within 7 days, high within 30.

- [ ] **Private package namespaces claimed**: Internal package names are reserved or scoped on public registries. Registry config points to the right source.
  - **Why it matters:** Prevents dependency confusion where a public package shadows an internal one.
  - **Secure default:** Scoped packages (`@org/name`) and a single proxied registry.

## Build and release

- [ ] **Reproducible, isolated builds**: Builds run in clean, ephemeral environments from source in version control.
  - **Why it matters:** Builds on shared or long-lived machines can be tampered with.
  - **Secure default:** Hosted or ephemeral runners. No manual steps between commit and artifact.

- [ ] **Artifacts signed with provenance**: Container images and release binaries are signed and carry build provenance.
  - **Why it matters:** Lets deploy systems verify an artifact came from your pipeline.
  - **Secure default:** Sigstore cosign signing, SLSA provenance, verification at deploy time.

- [ ] **SBOM generated per release**: Each release ships a software bill of materials.
  - **Why it matters:** When the next big library CVE drops, you can answer "are we affected" in minutes.
  - **Secure default:** CycloneDX or SPDX SBOM attached to each release.

- [ ] **Deploy approvals for production**: Prod deploys need an environment approval or a change ticket.
  - **Why it matters:** Stops a compromised branch or token from deploying on its own.
  - **Secure default:** Protected environments with required reviewers.
