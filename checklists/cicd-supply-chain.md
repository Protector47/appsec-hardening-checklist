# CI/CD and Supply Chain Hardening Checklist

Source control, build pipelines, dependencies, artifacts, and deployment. Aligned with SLSA, the OpenSSF Scorecard checks, and the OWASP Top 10 CI/CD Security Risks.

## 1. Source control

- [ ] **Branch protection or rulesets on the default branch**: Require PRs, at least one approving review from someone other than the author, passing status checks, and up-to-date branches. Block force pushes and deletion.
  - **How to verify:** OpenSSF Scorecard "Branch-Protection" check, or repo settings review.

- [ ] **Admins not exempt**: Enforce rules for administrators too.

- [ ] **CODEOWNERS on sensitive paths**: Auth code, crypto, CI workflows, IaC, and dependency manifests need review from the owning team.

- [ ] **Dismiss stale approvals**: New commits after approval require a fresh review.

- [ ] **Signed commits or verified pushes for release branches**: Require signed commits where your tooling supports it.

- [ ] **Org-wide MFA**: Required for all members and outside collaborators.

- [ ] **SSO enforced on the org**: Members access the org through your IdP.

- [ ] **Least privilege repo roles**: Most engineers have write, few have admin. Review org owners quarterly.

- [ ] **Base permissions restricted**: Org base permission set to "read" or "none".

- [ ] **Deploy keys and PATs reviewed**: Prefer GitHub Apps or fine-grained tokens with expiry. Remove unused deploy keys.

- [ ] **Forking of private repos disabled** unless needed.

## 2. Secrets in code

- [ ] **Secret scanning on all repos**: Including history and private repos.

- [ ] **Push protection on**: Blocks commits that contain known secret formats.

- [ ] **Pre-commit hooks**: gitleaks or trufflehog in a pre-commit hook for local feedback.

- [ ] **Leak response runbook**: A leaked secret is revoked and rotated first, then removed from history. Removing it from git alone is not enough.

## 3. Pipeline configuration

- [ ] **Default token read-only**: `permissions: contents: read` at the top of every workflow. Jobs raise permissions only as needed.
  - **How to verify:** Scorecard "Token-Permissions" check.

- [ ] **Actions pinned to full commit SHA**: Including reusable workflows. Comment the version next to the SHA for readability.
  - **Why it matters:** Tags can be moved to malicious commits by a compromised maintainer.
  - **How to verify:** Scorecard "Pinned-Dependencies" check.

- [ ] **Allowed actions restricted**: Org policy limits actions to verified creators and an allowlist.

- [ ] **No `pull_request_target` with PR checkout**: If you need `pull_request_target`, never check out or run code from the PR head in that job.
  - **Why it matters:** That combination runs untrusted code with write tokens and secrets.

- [ ] **No untrusted input in `run:` expressions**: Do not interpolate `${{ github.event.issue.title }}`, PR titles, branch names, or commit messages directly into shell steps. Pass them through environment variables.
  - **Why it matters:** Prevents script injection in workflows.
  - **How to verify:** actionlint or zizmor in CI.

- [ ] **Fork PRs need approval to run workflows**: Require approval for first-time and outside contributors.

- [ ] **Workflow files protected by CODEOWNERS**: `.github/workflows/` changes need security or platform review.

- [ ] **Caches not shared across trust levels**: Do not restore caches built by untrusted PRs into release jobs.

## 4. Secrets in CI

- [ ] **OIDC federation to cloud**: No static cloud keys stored as CI secrets. Trust policy scoped to repo, branch, and environment (`sub` claim).
  - **Why it matters:** A trust policy that only checks the org lets any repo in the org assume the role.

- [ ] **Environment-scoped secrets**: Production secrets only available to jobs in the protected `production` environment.

- [ ] **Secrets not echoed**: Masking on. Do not print env, and do not run with shell tracing (`set -x`) around secrets.

- [ ] **Short-lived credentials**: Prefer tokens minted per job over stored long-lived tokens.

## 5. Runners

- [ ] **Hosted or ephemeral runners**: Self-hosted runners are ephemeral (one job per runner) and autoscaled.

- [ ] **No self-hosted runners on public repos**.

- [ ] **Runner groups per trust level**: Production deploy runners separate from general build runners.

- [ ] **Runner network restricted**: Egress allowlist where practical. No access to production networks from build runners.

- [ ] **Harden-runner or egress monitoring**: Monitor outbound connections from build jobs.

## 6. Dependencies

- [ ] **Lockfiles committed**: `package-lock.json`, `pnpm-lock.yaml`, `poetry.lock`, `go.sum`, `Cargo.lock`, `Gemfile.lock`.

- [ ] **Deterministic installs in CI**: `npm ci`, `pnpm install --frozen-lockfile`, `pip install --require-hashes`, `go mod verify`.

- [ ] **Automated updates**: Dependabot or Renovate for dependencies, actions, and base images.

- [ ] **Vulnerability scanning**: OSV-Scanner, `npm audit`, `pip-audit`, `govulncheck`, Trivy, or your SCA tool on every PR and nightly on the default branch.

- [ ] **Patch SLAs**: Critical within 7 days, high within 30 days, tracked.

- [ ] **Install scripts controlled**: Consider `npm ci --ignore-scripts` with explicit allowlisting for packages that need scripts.
  - **Why it matters:** Malicious packages often run their payload at install time.

- [ ] **New dependency review**: Adding a dependency needs a quick check: maintenance activity, download count, maintainers, license, and Scorecard score.

- [ ] **Typosquat awareness**: Review package names carefully in PRs that add dependencies.

- [ ] **Dependency confusion prevented**: Scoped or namespaced internal packages. Registry config sends internal scopes only to your private registry.

- [ ] **Private registry proxy**: Builds pull from an internal proxy (Artifactory, Nexus, CodeArtifact) that can block known bad packages.

- [ ] **License policy**: Block licenses your legal team does not allow.

## 7. Static analysis and testing in CI

- [ ] **SAST on every PR**: CodeQL, Semgrep, or equivalent. New high-severity findings block merge.

- [ ] **IaC scanning**: Checkov, Trivy config, or KICS.

- [ ] **Container image scanning**: Trivy or Grype on built images.

- [ ] **Workflow linting**: actionlint and zizmor on `.github/workflows`.

- [ ] **Security tests in the suite**: Authorization tests (user A cannot access user B's data) run with unit and integration tests.

## 8. Build integrity

- [ ] **Builds from version control only**: No manual uploads of artifacts.

- [ ] **Ephemeral, isolated build environments**.

- [ ] **Build provenance**: SLSA provenance generated by the build platform (for example GitHub artifact attestations or slsa-github-generator).

- [ ] **Artifact signing**: Sigstore cosign for container images and binaries.

- [ ] **SBOM per release**: CycloneDX or SPDX, generated in the pipeline (Syft, CycloneDX tools) and attached to the release.

- [ ] **Minimal base images**: Distroless, Chainguard, or slim images. Pin base images by digest.

- [ ] **Non-root containers**: `USER` set to a non-root UID in Dockerfiles.

## 9. Artifact storage

- [ ] **Registries private by default**.

- [ ] **Immutable tags for releases**: Tag immutability on the registry.

- [ ] **Write access limited to CI**: Humans cannot push to release repositories.

- [ ] **Retention and cleanup**: Old, unsigned, and untested images are cleaned up.

## 10. Deployment

- [ ] **Protected production environment**: Required reviewers, wait timers if useful, and branch restrictions.

- [ ] **Deploy only signed artifacts**: Admission control or deploy step verifies signature and provenance.

- [ ] **GitOps or audited deploys**: Every deploy maps to a commit and an approver.

- [ ] **Rollback tested**: You can roll back within minutes.

- [ ] **Separate deploy identity**: The deploy role cannot change IAM or read unrelated secrets.

## 11. Releasing open source

- [ ] **Security policy**: `SECURITY.md` with a reporting address and expected response time.

- [ ] **Private vulnerability reporting enabled** on the repo.

- [ ] **Release process documented**: Who can release, how, and with what signing.

- [ ] **Scorecard action**: Run OpenSSF Scorecard and fix low-scoring checks.
