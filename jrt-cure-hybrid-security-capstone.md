# Capstone Project (Hybrid): Securing JRT Cure's AWS + Microsoft 365 / Entra ID / AVD Environment

## Updated Case Study Background

Everything from the original case study still holds — JRT Cure is a 150-person telehealth company, PHI/PII in AWS Prod, SOC 2 deadline, HIPAA exposure, board-level nervousness after a competitor's S3 breach.

**New layer:** JRT Cure runs its corporate identity and productivity stack on Microsoft 365, and just standardized on this hybrid model:

- **Entra ID is the single Identity Provider (IdP)** for the whole company — employees, contractors, and clinicians authenticate once into Entra ID, which then federates into both M365 (Exchange Online, SharePoint, Teams) and AWS (via IAM Identity Center as a SAML service provider). There is no such thing as an "AWS-only" identity anymore.
- **Microsoft 365** hosts internal comms (Teams), the intranet/knowledge base (SharePoint), and email (Exchange Online) — some of which touches PHI-adjacent content (e.g., a clinician emailing a colleague about a case, or a SharePoint site with de-identified patient volume reports).
- **Azure Virtual Desktop (AVD)** is the mandated access path for anyone touching Prod. Remote clinicians and on-call engineers don't get direct laptop access to the AWS console or Prod resources — they RDP into an Intune-managed, non-persistent AVD session host, and *from there* reach AWS. This is JRT Cure's answer to "how do we control BYOD risk without banning remote work."

**Why this matters for the story:** the board's real fear after the competitor breach wasn't just "an S3 bucket was public" — it was "we don't actually know who our identities are anymore across two clouds." This hybrid phase is about closing that gap: one IdP, one conditional access policy set, one place both AWS and Microsoft 365 logs get correlated.

The AWS account structure, SCPs, GuardDuty/Security Hub/Config setup, networking, and data protection controls from the original project **do not change** — you're adding an identity and access layer in front of them, not rebuilding them.

---

## Phase 1 (Revised) — Hybrid Identity Foundation

**Objective:** Entra ID is the single source of truth for "who is this person and what should they be allowed to do," in both Microsoft 365 and AWS.

Tasks:
- Configure **AWS IAM Identity Center to use Entra ID as an external SAML 2.0 IdP** (Identity Center supports this directly, or via Entra ID's "AWS IAM Identity Center" enterprise app gallery entry). Set up the trust: Entra ID as IdP, IAM Identity Center as the SAML service provider/relying party.
- Turn on **SCIM provisioning** from Entra ID into IAM Identity Center so that user/group creation, updates, and *deprovisioning* are automatic — a terminated employee in Entra ID should lose AWS access within minutes, not at the next manual audit.
- Recreate your permission-set mapping, but now driven by **Entra ID security groups** (e.g., `sg-aws-dev-poweruser`, `sg-aws-prod-readonly`, `sg-aws-breakglass`) instead of IAM Identity Center-native groups. Group membership in Entra ID is now the single lever that grants AWS access.
- Configure **Conditional Access policies** in Entra ID:
  - Require MFA for all users, all apps.
  - Require a **compliant/managed device** (i.e., issued from an AVD session host or Intune-enrolled machine) for any sign-in to the AWS enterprise app.
  - Block AWS console/API federation entirely from non-AVD, non-corporate networks for anyone with a Prod-scoped group.
  - Block legacy authentication protocols org-wide.
- Set up **Privileged Identity Management (PIM)** in Entra ID for just-in-time elevation into the `sg-aws-breakglass` group — mirrors your AWS break-glass role, but now the *approval and time-bound activation* happens in Entra ID, with PIM's own audit trail.
- Produce an updated **identity matrix**: Entra ID group → Conditional Access requirements → AWS permission set → M365 access level. This single table is what an auditor will ask for first.

**Deliverable:** SAML federation config notes, the Entra-group-to-permission-set matrix, and screenshots of at least 3 Conditional Access policies in report-only or enforced mode.

---

## Phase 2 (Extended) — Detective Controls, Now Cross-Cloud

**Objective:** You should be able to answer "what did this person do" across *both* clouds from one timeline, not two separate consoles.

Everything from the original Phase 2 (org CloudTrail, Config, GuardDuty, Security Hub, Macie) stays as-is. Add:

- Enable **Entra ID sign-in logs and audit logs**, and export them (Diagnostic Settings → Event Hub or Log Analytics workspace, or directly into a SIEM if you're running one, e.g., Microsoft Sentinel or forwarding into your AWS Security account via a Lambda that reads from an Event Hub).
- Enable **Entra ID Identity Protection** for risky sign-in and risky user detections.
- The key exercise: pick one AWS `AssumeRoleWithSAML` CloudTrail event and **trace it backward to the originating Entra ID sign-in log entry** (same SAML assertion, same timestamp window, same user). This correlation is the whole point of the hybrid setup — practice doing it manually before you automate it.
- If using Sentinel (optional but a strong resume line): bring AWS CloudTrail/GuardDuty findings into Sentinel via the AWS connector, sitting alongside Entra ID and M365 signals in one workspace.

**Deliverable:** A short writeup + screenshots showing one full identity trace: Entra ID sign-in → Conditional Access decision → SAML assertion → AWS `AssumeRoleWithSAML` → the specific AWS API calls that identity made.

---

## Phase 3 (Extended) — Preventive Controls, AVD Edition

**Objective:** The AVD session host is the choke point — harden it like the crown jewel it is.

Original AWS preventive controls (S3, KMS, networking, WAF) stay as-is. Add:

- **AVD host pool hardening:** session hosts are Intune-managed, auto-patched, non-persistent (or FSLogix-profile-based) — no local admin for standard users, no personal AWS CLI credentials stored locally (this is exactly why federation via Entra ID matters — nothing long-lived to steal off the host).
- **FSLogix profile containers** encrypted at rest.
- **RDP/AVD connection security:** enforce AVD's built-in RDP Shortpath with TLS, restrict AVD access itself behind Conditional Access (MFA + compliant network).
- Confirm (via the Conditional Access policy from Phase 1) that the AWS console/API is **unreachable** from any device that isn't an AVD session host or a corporate-managed endpoint — test this by trying to federate in from an unmanaged personal browser and confirming it's blocked.

**Deliverable:** A network/access diagram showing the enforced path: `User → Entra ID (CA policy) → AVD session host → AWS IAM Identity Center → AWS account`, with the blocked direct path shown crossed out.

---

## Phase 4 (Extended) — Data Protection, Now Including M365

**Objective:** PHI/PII protection needs to hold in Microsoft 365, not just S3.

Original AWS encryption/KMS/retention controls stay as-is. Add:

- Apply **Microsoft Purview Information Protection sensitivity labels** (e.g., `PHI-Confidential`, `Internal`, `Public`) across Exchange, SharePoint, and Teams.
- Configure a **Purview DLP policy** that blocks or warns when a label like `PHI-Confidential` is about to leave the tenant via email or external sharing.
- Write a short **classification-alignment note**: map your Macie-discovered AWS data categories to your Purview labels so PHI/PII means the same thing whether it's sitting in an S3 bucket or a SharePoint site. This alignment is a real thing auditors ask about — "do you have one data classification standard or two."

**Deliverable:** Updated data flow diagram now showing M365 as a PHI-adjacent surface, plus the classification-alignment note.

---

## Phase 5 (Extended) — Automated Remediation, Cross-Cloud Triggers

Original AWS Lambda auto-remediation stays as-is. Add one hybrid automation:

- Build a flow (Logic App, or Lambda triggered via an Event Hub/webhook from Entra ID Identity Protection) so that when Entra ID flags a **risky sign-in for a Prod-scoped identity**, it automatically:
  1. Revokes that user's active Entra ID sessions.
  2. Disables their IAM Identity Center access (since SCIM sync will eventually catch it, but you want it immediate).
  3. Posts to the same Slack/SNS channel your AWS findings already go to.

**Deliverable:** Working automation (or a well-documented design if you can't fully wire cross-cloud auth in your sandbox) + a note on why identity risk should trigger both-cloud response, not just one.

---

## Phase 6 (Extended) — Incident Simulation: Scenario C (Hybrid Pivot)

This is the new centerpiece exercise. Run it in addition to (or instead of) Scenario A/B from the original project.

**Scenario C: Compromised Entra ID identity used to pivot into AWS**

1. Simulate a phishing/MFA-fatigue compromise: manually trigger Entra ID Identity Protection's risky sign-in (or simulate by signing in from an atypical location/impossible-travel pattern for a test account).
2. Assume, for the exercise, the attacker used the compromised session to federate into AWS via `AssumeRoleWithSAML` and touched a Prod resource.
3. **Detect:** find the Entra ID risk event, then trace forward to the corresponding CloudTrail `AssumeRoleWithSAML` event and everything that identity did afterward (same correlation skill from Phase 2, now under pressure).
4. **Contain:** revoke Entra ID sessions, disable the account, force password reset, and — critically — treat any AWS credentials/session tokens issued from that federated session as compromised (they're time-limited by SAML/STS design, but confirm and document that).
5. **Eradicate:** review PIM activation history and Conditional Access sign-in logs for anything else this identity touched; check if MFA methods were tampered with.
6. **Recover:** re-enable the account through normal identity verification, re-issue MFA.
7. Write the **post-incident report**: this time the timeline has two columns — Entra ID events and AWS events, side by side — plus a root-cause note on whatever let the phishing succeed (no Conditional Access requiring device compliance? No phishing-resistant MFA?).

**Deliverable:** Dual-timeline incident report — this is the artifact that most directly demonstrates hybrid-cloud IR skill, which is genuinely rare and valuable to show in an interview.

---

## Phase 7 (Extended) — Compliance Mapping, Hybrid Control Matrix

Extend your original control matrix with a new column for where the control actually lives:

| Control | Lives In | AWS Service / Entra Feature | Framework Mapping |
|---|---|---|---|
| MFA enforced for all access | Entra ID | Conditional Access | SOC 2 CC6.1 |
| Device compliance required for Prod access | Entra ID + Intune | Conditional Access | SOC 2 CC6.6 |
| Automatic deprovisioning on termination | Entra ID → AWS | SCIM provisioning | SOC 2 CC6.2 |
| PHI classification consistent across clouds | M365 + AWS | Purview labels + Macie | HIPAA Safeguards |
| Privileged access is time-boxed | Entra ID + AWS | PIM + break-glass role | SOC 2 CC6.3 |
| Cross-cloud audit trail | Entra ID + AWS | Sign-in logs + CloudTrail | SOC 2 CC7.2 |

**Deliverable:** Full control matrix spreadsheet, extended with the "Lives In" column — this is the artifact that makes your project read as genuinely hybrid rather than "AWS project with a login screen bolted on."

---

## What Changed vs. the Original Project (Quick Reference)

- **Unchanged:** account structure, SCPs, CloudTrail/Config/GuardDuty/Security Hub/Macie setup, S3/KMS/networking/WAF controls, RDS encryption/backup, auto-remediation Lambda pattern.
- **New:** Entra ID as IdP with SAML federation + SCIM into IAM Identity Center, Conditional Access as the real preventive layer, AVD as the mandated access path, Purview alongside Macie, PIM alongside AWS break-glass, and a hybrid identity-trace skill that threads through detection, prevention, and incident response.

## Updated Pacing

| Week | Focus |
|---|---|
| 1 | Phase 1 (Entra ID federation, Conditional Access, PIM) |
| 2 | Phase 2 (cross-cloud detection + log correlation) |
| 3 | Phase 3 (AVD hardening) + Phase 4 (Purview/data protection) |
| 4 | Phase 5 (cross-cloud automation) |
| 5 | Phase 6 (Scenario C hybrid incident simulation) |
| 6 | Phase 7 (hybrid control matrix) + final writeup |

## Notes on Sandbox Setup
- You'll need an Entra ID tenant (a free/dev tenant works) with IAM Identity Center configured for external IdP mode — this is a one-time trust configuration, not a code change.
- AVD requires an Azure subscription with at least one session host — a single `B2s`-class VM is enough to practice the access-path pattern; you don't need a large host pool.
- Keep the same cost/safety discipline as before: budget alarms, and no real-shaped PHI data anywhere, even in the "compromised" test account.
