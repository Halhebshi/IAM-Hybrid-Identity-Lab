# Enterprise Identity and Access Management Lab

A self-directed lab built around a simple idea: organize identity work the way an identity actually lives in an enterprise, not as a pile of disconnected features.

An identity is born in a system of record, provisioned, governed, continuously verified, defended when attacked, and eventually deprovisioned. This lab builds that entire lifecycle hands-on across a hybrid environment: on-prem Active Directory, Entra Connect, Entra ID P2, Defender for Identity, Sentinel, and a SailPoint developer tenant. Where a capability genuinely requires a paid enterprise platform (Workday, CyberArk), it is covered conceptually and marked as such rather than faked.

## Environment

- Windows Server 2022 domain controller running on-prem Active Directory
- Entra Connect with Password Hash Sync, scoped to a dedicated OU
- Entra ID P2 tenant with full M365 security licensing
- Hybrid-joined endpoint enrolled in Intune
- Microsoft Defender for Identity sensor on the DC
- Microsoft Sentinel workspace with Azure Logic Apps for SOAR
- SailPoint Identity Security Cloud developer tenant
- GitHub Actions for workload identity federation

## What Was Built

### Part 1: Hybrid Identity Foundation

AD as the system of record, Entra Connect as the bridge, and a documented attribute ownership model. The interesting problem solved here: Lifecycle Workflows trigger on employeeHireDate and employeeLeaveDateTime, but these are cloud-only attributes that cannot be written directly on synced users via the portal or Graph. The working solution stores values in msDS-cloudExtensionAttribute1/2 in Generalized-Time format and maps them through a cloned inbound sync rule with Direct flow type. Getting this right took real troubleshooting; conversion expressions are the most common failure point and turn out to be unnecessary when the source format is correct.

### Part 2: Identity Lifecycle (JML)

Full joiner, mover, leaver automation:

- **Joiner**: account created in AD with complete attributes, Lifecycle Workflow fires the day before start date, provisioning license, groups, Temporary Access Pass, and manager notification
- **Mover**: department attribute change in AD drives dynamic group re-evaluation and entitlement management auto-assignment, removing old access and granting new with no manual steps
- **Leaver**: two models built deliberately. A Lifecycle Workflow handles planned offboarding with a per-task audit trail. A separate SOAR chain (Sentinel detection rule plus Logic App with managed identity) revokes sessions and strips licenses within seconds of any account disable, covering emergency containment of compromised accounts

### Part 3: Access Governance

- Three-RBAC-model implementation: Entra roles, Azure RBAC, and app roles, with Administrative Units scoping helpdesk roles so a Password Administrator cannot touch admin accounts
- PIM with eligible (not permanent) Global Admin, approval gates, MFA on activation, and break-glass accounts excluded by design
- Entitlement management with access packages, manager approval, expiry, and auto-assignment policies that revoke on transfer
- Recurring access reviews with fail-secure defaults (no response means removal)
- Segregation of Duties enforced through incompatible access packages

### Part 4: Authentication and Zero Trust

- Conditional Access baseline: MFA for all, legacy auth blocked, compliant or hybrid-joined device required
- Phishing-resistant authentication strengths (FIDO2, Windows Hello for Business) enforced for admin roles, addressing the adversary-in-the-middle token theft pattern that standard push MFA cannot stop
- Token Protection and Continuous Access Evaluation enabled
- SAML SSO and SCIM automatic provisioning to a gallery app, closing the leaver gap across SaaS

### Part 5: Identity Threat Detection and Response

- Defender for Identity sensor on the DC with sensitive account tagging and a honeytoken account
- Three attacks simulated and detected: Kerberoasting, LDAP reconnaissance, and DCSync
- Identity Protection risk policies: sign-in risk triggers MFA, user risk triggers secure password change
- Sentinel KQL detections correlating on-prem and cloud identity signals
- SOAR playbook that auto-contains high-risk privileged accounts: revoke sessions, disable account, notify SOC. Deliberately scoped to admin role holders only so it does not override the gentler self-service remediation for standard users

### Part 6: Enterprise IGA and PAM

- SailPoint Identity Security Cloud: source aggregation from a delimited file, identity warehouse, certification campaign with revocations flowing back to the source
- Cross-application Segregation of Duties policy, the capability Entra cannot provide and the main reason regulated enterprises buy IGA platforms
- Workload identity federation: GitHub Actions authenticating to Azure via OIDC with zero stored secrets
- CyberArk covered conceptually, mapping each PIM concept to its PAM equivalent (vaulting, session recording, rotation)

## Lessons That Cost Time

- employeeHireDate on synced users has exactly one supported path and it is not the portal or Graph. The AD attribute plus sync rule approach is documented above because nothing online lays it out end to end.
- Sync timing matters more than expected. employeeLeaveDateTime reached the metaverse quickly but the export to Entra ID took far longer than the standard sync cycle suggests. The lesson: verify each stage of the pipeline (AD, metaverse, Entra ID) separately before assuming a configuration problem, because sometimes the answer is patience.
- Disabling an account does not kill active tokens. Sessions stay valid up to an hour unless explicitly revoked, which is why every leaver path in this lab revokes sessions and why Continuous Access Evaluation matters.
- A cloned sync rule at the wrong precedence can take down attribute flow for every synced user. The bad clone created during troubleshooting was disabled and kept as evidence rather than deleted.

## Documentation

Full lab guide with configuration steps, PowerShell, KQL detections, and interview scenario mapping: [Lab04_Enterprise_IAM.pdf]


