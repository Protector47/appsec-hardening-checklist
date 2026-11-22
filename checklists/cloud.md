# Cloud Hardening Checklist

Applies to AWS, Azure, and GCP. Provider-specific names are in parentheses. Use for new accounts, before production launch, and in quarterly reviews. Map to the CIS Benchmark for your provider for formal coverage.

## 1. Organization and account structure

- [ ] **Separate accounts or projects per environment**: Prod, staging, dev, and sandbox are isolated.
  - **Why it matters:** A leaked dev credential should not reach prod data.
  - **Secure default:** AWS Organizations OUs, Azure management groups, GCP folders.

- [ ] **Dedicated security and log archive accounts**: Logs and security tooling live in accounts that workload teams cannot administer.

- [ ] **Organization guardrails**: Preventive policies at the org level (AWS SCPs, Azure Policy deny effects, GCP Organization Policy).
  - **Secure default:** Deny disabling CloudTrail or audit logs, deny leaving the org, deny use of unapproved regions, deny public bucket ACLs.

- [ ] **Root and global admin accounts locked down**: Hardware MFA, no access keys, credentials in a sealed break-glass process. Alerts on any use.
  - **How to verify:** AWS IAM credential report shows no root access keys. Check sign-in alerts fire in a test.

- [ ] **Contact and billing info current**: Security contact set so the provider can reach you about abuse or compromise.

- [ ] **Budget and anomaly alerts**: Cost anomaly detection enabled.
  - **Why it matters:** Sudden spend is often the first sign of compromised credentials (crypto mining).

## 2. Identity and access management

- [ ] **Human access through SSO with MFA**: No local IAM users for people.
  - **Secure default:** AWS IAM Identity Center, Entra ID, Google Workspace / Cloud Identity.

- [ ] **No long-lived access keys**: Workloads use roles (instance profiles, IRSA / EKS Pod Identity, Azure managed identities, GCP workload identity). CI uses OIDC federation.
  - **How to verify:** List access keys older than 90 days. The target is zero.

- [ ] **Least privilege policies**: No `Action: "*"` with `Resource: "*"` on anything except break-glass roles. No `Owner` or `Editor` on service accounts.
  - **Secure default:** Generate policies from access activity (IAM Access Analyzer policy generation, GCP IAM Recommender, Entra access reviews).

- [ ] **Unused permissions removed**: Review last-used data quarterly and remove unused roles, permissions, and principals.

- [ ] **Privilege escalation paths reviewed**: Restrict `iam:PassRole`, `iam:CreatePolicyVersion`, `iam:AttachUserPolicy`, role assumption chains, and their Azure/GCP equivalents (`Microsoft.Authorization/roleAssignments/write`, `iam.serviceAccounts.actAs`).
  - **Why it matters:** These permissions let a low-privilege identity grant itself admin.

- [ ] **Permission boundaries for delegated admins**: Teams that can create roles are limited by a permission boundary or equivalent.

- [ ] **Cross-account trust scoped**: Role trust policies name specific principals and use `ExternalId` for third parties. No `Principal: "*"` without strong conditions.
  - **How to verify:** IAM Access Analyzer external access findings.

- [ ] **Service account keys disabled (GCP)**: Org policy `iam.disableServiceAccountKeyCreation` enforced.

## 3. Object storage

- [ ] **Public access blocked at account level**: S3 Block Public Access (account), Azure storage `allowBlobPublicAccess=false`, GCP `storage.publicAccessPrevention` enforced.
  - **Why it matters:** Public buckets remain a leading source of data exposure.

- [ ] **Public buckets are documented exceptions**: Static website buckets are in a separate account or project, contain no private data, and are tagged with an owner.

- [ ] **ACLs disabled**: S3 Object Ownership "Bucket owner enforced". GCP uniform bucket-level access.

- [ ] **Bucket policies deny insecure transport**: Condition `aws:SecureTransport = false` denied. Azure "secure transfer required".

- [ ] **Versioning and object lock on critical data**: Protects against deletion and ransomware.

- [ ] **Pre-signed and SAS URLs short-lived**: Minutes, not days. Azure SAS uses user delegation keys, not account keys.

- [ ] **Storage account keys not used by apps (Azure)**: Disable shared key access where possible and use Entra ID auth.

## 4. Data protection

- [ ] **Encryption at rest everywhere**: Databases, volumes, snapshots, buckets, queues, and backups.

- [ ] **Customer-managed keys for sensitive data**: Separate keys per data class or tenant. Key policies limit who can decrypt.

- [ ] **Key rotation on**: Automatic rotation for KMS keys. Documented rotation for any app-level keys.

- [ ] **Secrets in a secrets manager**: AWS Secrets Manager, Azure Key Vault, GCP Secret Manager. Not in env files, AMIs, container images, or Terraform state.
  - **How to verify:** Secret scanning on infrastructure repos. Check Terraform state storage is encrypted and access-restricted.

- [ ] **Snapshots and AMIs private**: No public snapshots, images, or database snapshots.
  - **How to verify:** Provider config rules (AWS Config `ebs-snapshot-public-restorable-check`, `rds-snapshots-public-prohibited`).

- [ ] **Backups isolated and tested**: Cross-account or vault-locked backups. Restore tested on a schedule.

## 5. Network

- [ ] **No admin ports open to the internet**: SSH (22), RDP (3389), database ports (3306, 5432, 1433, 27017, 6379, 9200) not open to `0.0.0.0/0` or `::/0`.
  - **Secure default:** SSM Session Manager, Azure Bastion, GCP IAP TCP forwarding.

- [ ] **Default VPC and default security groups unused**: Default security group has no rules.

- [ ] **Private subnets for data stores**: Databases, caches, and internal services have no public IPs.

- [ ] **Private endpoints for cloud services**: VPC endpoints / Private Link / Private Service Connect for S3, KMS, Secrets Manager, and similar.

- [ ] **Egress filtering on sensitive workloads**: Outbound through NAT plus a proxy or firewall with domain allowlists.

- [ ] **WAF in front of public apps**: Managed rule sets on load balancers or CDN, tuned in count mode first.

- [ ] **DDoS protection on public endpoints**: Provider standard DDoS protection at minimum.

- [ ] **VPC flow logs on**: At least for production VPCs, shipped to the log archive account.

## 6. Compute

- [ ] **IMDSv2 required (AWS)**: `HttpTokens=required` on all instances and launch templates. Hop limit 1 unless containers need it.
  - **Why it matters:** IMDSv1 is easy to reach through SSRF.
  - **How to verify:** AWS Config rule `ec2-imdsv2-check`.

- [ ] **Metadata protection on other providers**: GCP and Azure metadata requires the `Metadata-Flavor` / `Metadata` header. Do not proxy these headers from user input.

- [ ] **Hardened base images**: CIS-hardened or minimal images. Rebuilt and redeployed at least monthly.

- [ ] **Patch management**: Automated OS patching (SSM Patch Manager, Azure Update Manager, OS Config).

- [ ] **No secrets in user data or startup scripts**.

- [ ] **Serverless functions least privilege**: One role per function. Timeouts and concurrency limits set. Function URLs require auth unless intentionally public.

## 7. Containers and Kubernetes

- [ ] **Managed control plane endpoint private or IP-restricted**.

- [ ] **RBAC least privilege**: No `cluster-admin` for apps or CI. Namespaced roles.

- [ ] **Pod Security Standards enforced**: `restricted` profile for app namespaces: non-root, no privilege escalation, read-only root filesystem, dropped capabilities, no host namespaces.

- [ ] **Network policies**: Default deny between namespaces, then allow what is needed.

- [ ] **Workload identity instead of node roles**: IRSA / EKS Pod Identity, Azure Workload Identity, GKE Workload Identity.

- [ ] **Image provenance enforced**: Admission control allows only signed images from your registries.

- [ ] **Secrets encrypted in etcd**: KMS envelope encryption enabled.

## 8. Logging and detection

- [ ] **Audit logs on in all regions and accounts**: Org-wide CloudTrail (management events at minimum, data events for sensitive buckets), Azure Activity Log and Entra sign-in logs, GCP Admin Activity and Data Access logs.

- [ ] **Logs immutable**: Log bucket in the archive account with object lock and restricted delete.

- [ ] **Threat detection on**: GuardDuty, Defender for Cloud, Security Command Center.

- [ ] **Alerts on high-risk events**: Root sign-in, IAM policy changes, new access keys, security group opened to the world, audit logging disabled, KMS key scheduled for deletion.

- [ ] **Config and posture monitoring**: AWS Config / Azure Policy compliance / Security Command Center posture against the CIS Benchmark.

## 9. Infrastructure as code

- [ ] **All infra in code**: Terraform, Bicep, CloudFormation, or Pulumi. Console changes are the exception and get reconciled.

- [ ] **IaC scanning in CI**: Checkov, tfsec / Trivy config, or KICS on every PR.

- [ ] **State files protected**: Remote state encrypted, versioned, access-restricted, with locking.

- [ ] **Drift detection**: Scheduled plan or drift detection to catch manual changes.

## 10. DNS and domains

- [ ] **No dangling DNS records**: CNAMEs and alias records point only to resources you still own.
  - **Why it matters:** Deleted cloud resources with live DNS records allow subdomain takeover.
  - **How to verify:** Script that resolves every record in your zones and checks the target still exists in your accounts.

- [ ] **Registrar locked with MFA**: Domain transfer lock on. Registrar account uses MFA.

- [ ] **DNSSEC where supported**.
