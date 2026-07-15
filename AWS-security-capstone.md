# Capstone Project: Securing JRT Learn's AWS Landing Zone

## Case Study Background

**JRT Learn** is a fictional 150-person digital health & wellness company. Their flagship product is a subscription app + telehealth platform: users book virtual consultations, store health records, track fitness data, and pay via a Stripe-tokenized card flow (so raw PAN never touches AWS, but the environment is still PCI-*adjacent*).

**Why security matters right now (the narrative driving this project):**
- JRT Learn just closed a Series B and signed its first enterprise wellness client, who is requiring a **SOC 2 Type II** report within 6 months.
- Because they store telehealth records, they are in scope for **HIPAA** (PHI: patient names, diagnoses, appointment notes, video call metadata).
- A close competitor had a public S3 bucket breach last quarter that leaked customer PII — the board has asked security to prove "that can't happen to us."
- The engineering team ships fast (multiple deploys/day) and has historically had broad IAM permissions in a single AWS account. You've just finished migrating them to a multi-account **Landing Zone** (Control Tower or equivalent) and now own the security build-out.

**Assumed account structure** (adjust to match what you actually built):

| OU | Accounts | Purpose |
|---|---|---|
| Security | Log Archive, Audit/Security Tooling | Central CloudTrail/Config logs, GuardDuty/Security Hub delegated admin |
| Infrastructure | Shared Services, Network | Transit Gateway, DNS, shared VPC endpoints |
| Workloads | Dev, Staging, Prod | Application accounts, environment-separated |
| Sandbox | Individual dev sandboxes | Cost/blast-radius isolation for experimentation |

**Data classification:**
- **PHI** — telehealth notes, appointment records (Prod only)
- **PII** — names, emails, addresses, DOB (Prod, some in Staging with masked data)
- **Confidential** — internal financials, employee data (Shared Services)
- **Public** — marketing site assets

---

## Project Objective

Design and implement the security controls a real healthtech company would need to pass a SOC 2 audit, demonstrate HIPAA-reasonable safeguards, and survive a credible incident — **on top of** the Landing Zone you've already built. You're not just enabling services; you're producing the artifacts an auditor or CISO would actually ask for (policies-as-code, runbooks, evidence).

---

## Phase 1 — Identity & Access Foundations

**Objective:** Nobody has standing access they don't need; every human/system identity is traceable to a person or workload.

Tasks:
- Set up **IAM Identity Center** (or confirm it's federated) with permission sets mapped to job functions: `DeveloperReadOnly-Dev`, `DeveloperPowerUser-Dev`, `SecurityAuditor-ReadOnly-AllAccounts`, `BreakGlassAdmin`.
- Write **Service Control Policies (SCPs)** at the OU level:
  - Deny root user actions except in Management account.
  - Deny disabling of CloudTrail, Config, GuardDuty.
  - Deny creation of IAM users/access keys outside a designated `Legacy` account (force workloads onto roles).
  - Deny leaving the AWS Organization.
  - Region restriction (e.g., deny all actions outside `us-east-1`/`us-west-2` if that's your compliance boundary).
- Implement a **break-glass procedure**: an emergency IAM role with strong MFA + CloudTrail alerting, documented and tested once.
- Enable **IAM Access Analyzer** in the Audit account, run it against Prod, and remediate any external/cross-account access findings.

**Deliverable:** SCP JSON files in a repo, an IAM permission-set matrix (who can do what, where), and a one-page break-glass runbook.

---

## Phase 2 — Detective Controls (the "auditor walks in" layer)

**Objective:** Every account emits logs to one place, and JRT Learn can answer "what happened and when" for any resource.

Tasks:
- Confirm **organization-wide CloudTrail** is on, logging to the Log Archive account, with log file validation enabled.
- Enable **AWS Config** with an organization aggregator; deploy a **conformance pack** for HIPAA or CIS AWS Foundations Benchmark.
- Enable **GuardDuty** org-wide with delegated admin in the Audit account (include S3, EKS, RDS protection if relevant).
- Enable **Security Hub** org-wide, aggregate findings into the Audit account, enable the CIS and AWS Foundational Security Best Practices standards.
- Enable **Amazon Macie** in Prod and Staging, run a discovery job, and confirm it correctly flags the PHI-bearing S3 buckets.
- Set a **CloudWatch/EventBridge** rule that forwards HIGH/CRITICAL Security Hub findings to a Slack/SNS channel.

**Deliverable:** Screenshot/export of Security Hub compliance score before and after this phase, plus a short "what we can now detect" writeup.

---

## Phase 3 — Preventive Controls

**Objective:** Make the risky thing hard to do by accident.

Tasks:
- **S3:** Account-level Block Public Access everywhere except an explicitly approved public-assets bucket; bucket policies requiring `aws:SecureTransport`; default encryption with KMS on PHI/PII buckets.
- **KMS:** Customer-managed keys per data classification (one for PHI, one for general app data), key policies scoped to specific roles, rotation enabled.
- **Networking:** Prod app tier in private subnets only; security groups scoped to specific ports/sources (no `0.0.0.0/0` except the ALB); NACLs as a second layer for the PHI database subnet.
- **WAF + Shield:** Attach AWS WAF to the public ALB/CloudFront distribution with managed rule groups (SQLi, common vulnerabilities) plus rate-based rules; enable Shield Standard (or Advanced if you want to simulate that decision).
- **Secrets:** Move any hardcoded DB credentials into **Secrets Manager** with rotation; confirm no secrets in Parameter Store as plaintext.

**Deliverable:** Before/after network diagram, and an SCP or Config rule proving public S3 access is now structurally prevented, not just discouraged.

---

## Phase 4 — Data Protection Deep Dive (HIPAA-flavored)

**Objective:** Prove PHI is protected at rest, in transit, and during backup/retention.

Tasks:
- Confirm encryption in transit end-to-end (ALB→app with TLS, app→RDS with TLS enforced).
- Enable **RDS/Aurora encryption** with the PHI-specific KMS key; enable automated backups + a defined retention period; test a point-in-time restore.
- Set an **S3 Lifecycle policy** for PHI logs matching your (invented) retention policy, e.g., 6 years to satisfy a HIPAA-like requirement, then archive to Glacier.
- Document data flow: where does PHI enter, where does it rest, where does it leave (e.g., third-party telehealth video vendor) — a simple data flow diagram is one of the most commonly requested SOC 2 artifacts.

**Deliverable:** A one-page data flow diagram + an encryption/retention control matrix (data type → at rest → in transit → backup → retention).

---

## Phase 5 — Automated Remediation

**Objective:** Reduce mean-time-to-remediate so findings don't just pile up in Security Hub.

Tasks:
- Build one **EventBridge + Lambda** auto-remediation: e.g., if Config detects a security group open to `0.0.0.0/0` on port 22, automatically revoke it and notify.
- Build a **Security Hub custom action** that a human can click to trigger a Lambda (e.g., "quarantine this EC2 instance" by moving it to an isolation security group).
- Document which findings are auto-remediated vs. which require human judgment, and why.

**Deliverable:** Working Lambda + a short design note on your auto-remediation philosophy (what's safe to automate vs. not).

---

## Phase 6 — Incident Response Simulation (the capstone centerpiece)

This is what separates "I turned on some services" from "I can actually respond to a breach." Pick **one** scenario (or run both if you have time) and treat it like a tabletop exercise you execute for real in your sandbox.

### Scenario A: Leaked credentials → cryptomining
Simulate: an IAM access key (belonging to a low-privilege CI role) gets "leaked" — you manually trigger this by having GuardDuty generate a finding, or by intentionally launching an EC2 instance from an unusual region/IP using that key, mimicking `UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration` or `CryptoCurrency:EC2/BitcoinTool.B`.
1. GuardDuty fires a finding.
2. Trace it in CloudTrail: who/what used the key, from where, what API calls followed.
3. Contain: disable/rotate the key, isolate the instance (SG with no rules), snapshot the volume for forensics.
4. Eradicate: terminate the compromised resource, review Access Analyzer for further exposure.
5. Recover: redeploy clean infrastructure from IaC.
6. Write a **post-incident report**: timeline, root cause, impact assessment (was PHI/PII touched?), and 3 corrective actions.

### Scenario B: Public S3 bucket exposing PHI (the "competitor" scenario)
1. Intentionally misconfigure a test bucket (in a sandbox, clearly labeled) to be public.
2. Confirm Macie/Config/Security Hub detect it — measure time-to-detection.
3. Contain: re-apply Block Public Access, rotate any exposed credentials if applicable.
4. Assess: what was exposed, for how long, does this trigger a HIPAA breach-notification obligation (research the 500-person/60-day rule as a thought exercise)?
5. Write the same style of post-incident report, plus a simulated "breach notification memo" to leadership.

**Deliverable:** A timestamped incident timeline, root-cause analysis, and post-incident report — this single artifact is one of the most valuable things you can show in an interview.

---

## Phase 7 — Compliance Mapping & Audit Readiness

**Objective:** Translate what you built into what an auditor actually reads.

Tasks:
- Map each control you built to a **SOC 2 Trust Services Criteria** (Security, Availability) — e.g., "SCP denying CloudTrail deletion" → CC7.2 (detection of security events).
- Use **AWS Audit Manager** (optional) with the SOC 2 or HIPAA framework to auto-generate an evidence folder.
- Produce a **control matrix**: Control | AWS Service | Framework Mapping | Evidence Location | Owner.

**Deliverable:** A control matrix spreadsheet — this is a real, portfolio-worthy artifact.

---

## Final Deliverables Checklist

- [ ] Architecture diagram (accounts, OUs, network)
- [ ] SCP + IAM permission-set repo
- [ ] Security Hub / Config compliance score screenshots (before/after)
- [ ] Data flow diagram + encryption/retention matrix
- [ ] One working auto-remediation Lambda
- [ ] Incident timeline + post-incident report from your simulation
- [ ] Compliance control matrix (SOC 2 or HIPAA mapped)
- [ ] Executive summary (1 page): "Here's the risk we reduced and how"

## Suggested Pacing

| Week | Focus |
|---|---|
| 1 | Phase 1 (IAM/SCPs) |
| 2 | Phase 2 (Detective controls) |
| 3 | Phase 3 (Preventive controls) |
| 4 | Phase 4 (Data protection) + Phase 5 (Automation) |
| 5 | Phase 6 (Incident simulation) |
| 6 | Phase 7 (Compliance mapping) + final writeup |

## Cost & Safety Notes
- GuardDuty, Config, Security Hub, and Macie all have ongoing costs proportional to activity — set a budget alarm before you start.
- Do incident simulations only in Sandbox/Dev accounts with resources you clearly label as test data — never use real PHI-shaped data, even fake-realistic data, in a bucket you intentionally expose.
- Tear down WAF/Shield Advanced and any NAT Gateways when not actively using them if cost is a concern.
