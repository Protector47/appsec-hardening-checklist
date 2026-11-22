# Cloud Hardening Checklist

Applies to AWS, Azure, and GCP. Use it when reviewing a new account, subscription, or project, and again before production launch.

## Accounts and organization

- [ ] **Separate accounts per environment**: Prod, staging, and dev live in separate accounts or projects under one organization.
  - **Why it matters:** A leaked dev credential should not reach prod data.
  - **Secure default:** AWS Organizations, Azure management groups, or GCP folders with guardrail policies (SCPs, Azure Policy, Org Policy).

- [ ] **Root and break-glass accounts locked down**: Root has hardware MFA, no access keys, and is used only for documented break-glass steps.
  - **Why it matters:** Root compromise means full account loss.
  - **Secure default:** No root access keys. Alert on any root sign-in.

## Identity and access

- [ ] **No long-lived keys for workloads**: Workloads use instance roles, managed identities, or workload identity federation.
  - **Why it matters:** Static keys end up in repos, images, and laptops.
  - **Secure default:** OIDC federation for CI. Rotate or delete any remaining keys.

- [ ] **Least privilege IAM**: No `*:*` policies on humans or services. Review unused permissions quarterly.
  - **Why it matters:** Broad roles turn a small bug into full account access.
  - **Secure default:** Start from managed read-only roles and add specific actions. Use access analyzer tools to trim.

- [ ] **Human access through SSO**: Engineers sign in with the corporate IdP and MFA, not local IAM users.
  - **Why it matters:** Local users are rarely offboarded on time.
  - **Secure default:** IAM Identity Center, Entra ID, or Google Workspace SSO with short sessions.

## Storage and data

- [ ] **Block public access on object storage**: Account-level public access block is on. Public buckets are exceptions with a ticket and an owner.
  - **Why it matters:** Public buckets are one of the most common sources of data leaks.
  - **Secure default:** S3 Block Public Access at account level, Azure `allowBlobPublicAccess=false`, GCP uniform bucket-level access with public access prevention.

- [ ] **Encryption at rest with managed keys**: Databases, disks, buckets, and backups are encrypted. Sensitive data uses customer-managed keys with restricted key policies.
  - **Why it matters:** Key policy is a second access control layer on top of IAM.
  - **Secure default:** KMS / Key Vault / Cloud KMS keys per data class, with rotation on.

- [ ] **Backups tested and protected**: Backups are immutable or in a separate account, and restores are tested.
  - **Why it matters:** Ransomware and bad deploys both target backups.
  - **Secure default:** Object lock or vault lock. Restore drill at least twice a year.

## Network

- [ ] **No management ports open to the internet**: SSH, RDP, and database ports are not reachable from `0.0.0.0/0`.
  - **Why it matters:** Exposed admin ports get brute forced within hours.
  - **Secure default:** SSM Session Manager, Azure Bastion, or IAP for admin access.

- [ ] **Private endpoints for managed services**: Databases and internal APIs are reachable only from private networks.
  - **Why it matters:** Public endpoints rely on a single credential for protection.
  - **Secure default:** PrivateLink / Private Endpoint / Private Service Connect.

- [ ] **Egress controlled for sensitive workloads**: Outbound traffic goes through a NAT or proxy with allowlists where practical.
  - **Why it matters:** Unrestricted egress makes data exfiltration and C2 easy.
  - **Secure default:** Deny by default on sensitive subnets. Log egress.

## Metadata and compute

- [ ] **Instance metadata hardened**: IMDSv2 is required on AWS. Metadata access is limited on other providers.
  - **Why it matters:** SSRF bugs often target metadata endpoints for credentials.
  - **Secure default:** `HttpTokens=required`, hop limit 1 for containers that do not need it.

- [ ] **Images patched and minimal**: Base images come from a hardened source and are rebuilt on a schedule.
  - **Why it matters:** Old images carry known CVEs into every new instance.
  - **Secure default:** Golden images or distroless containers, rebuilt at least monthly.

## Guardrails and visibility

- [ ] **Audit logging on in every region**: CloudTrail, Azure Activity Log, and GCP Audit Logs are on, centralized, and protected from deletion.
  - **Why it matters:** No logs means no investigation.
  - **Secure default:** Org-wide trail to a separate log archive account.

- [ ] **Posture checks running**: A CSPM or native tool (Security Hub, Defender for Cloud, Security Command Center) runs against a benchmark like CIS.
  - **Why it matters:** Drift happens. Manual reviews miss it.
  - **Secure default:** CIS benchmark enabled, findings routed to an owner with an SLA.
