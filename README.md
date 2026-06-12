# IAM Hybrid Identity and Governance Lab

A self-directed lab built around a simple idea: organize identity work the way an identity actually lives in an enterprise, not as a pile of disconnected features.

An identity is born in a system of record, provisioned, governed, continuously verified, defended when attacked, and eventually deprovisioned. This lab builds that entire lifecycle hands-on across a hybrid environment: on-prem Active Directory, Entra Connect, Entra ID P2, Defender for Identity, Sentinel, and Global Secure Access. Where a capability genuinely requires a paid enterprise platform (Workday, CyberArk, SailPoint), it is covered conceptually and marked as such rather than faked.

## Environment

- Windows Server 2022 domain controller running on-prem Active Directory
- Entra Connect with Password Hash Sync, scoped to a dedicated OU
- Entra ID P2 tenant with full M365 E5 security licensing
- Hybrid-joined endpoint enrolled in Intune
- Microsoft Defender for Identity sensor on the DC
- Microsoft Sentinel workspace with Azure Logic Apps for SOAR
- GitHub Actions with workload identity federation
- Global Secure Access with private network connector for Application Proxy and Private Access
- SailPoint Identity Security Cloud (conceptual — hands-on tenant requires customer license or partner status; actively working toward SailPoint ambassador status to get access)

## What Was Built

### Part 1: Hybrid Identity Foundation

AD as the system of record, Entra Connect as the bridge, and a documented attribute ownership model. The interesting problem here: Lifecycle Workflows trigger on employeeHireDate and employeeLeaveDateTime, but these are cloud-only attributes that cannot be written directly on synced users via the portal or Graph. The working solution stores values in msDS-cloudExtensionAttribute1/2 in Generalized-Time format and maps them through a cloned inbound sync rule with Direct flow type. Conversion expressions are the most common failure point and turn out to be unnecessary when the source format is correct.

### Part 2: Identity Lifecycle (JML)

Full joiner, mover, leaver automation:

- **Joiner**: account created in AD with complete attributes, Lifecycle Workflow fires the day before start date, provisioning license, groups, Temporary Access Pass, and manager notification
- **Mover**: department attribute change in AD drives dynamic group re-evaluation and entitlement management auto-assignment, removing old access and granting new with no manual steps
- **Leaver**: two models built deliberately. A Lifecycle Workflow handles planned offboarding with a per-task audit trail. A separate SOAR chain (Sentinel detection rule plus Logic App with managed identity) revokes sessions and strips licenses within seconds of any account disable, covering emergency containment of compromised accounts. CAE handles session revocation near-instantly; the SOAR chain handles everything else — license removal, group cleanup, manager notification, and audit trail

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
- SAML SSO configured to Salesforce with Entra as the identity provider; SCIM automatic provisioning requires a paid Salesforce instance and is documented conceptually
- App Registrations, service principals, OAuth 2.0 consent frameworks, and the defensive review of third-party app permissions

### Part 5: Identity Threat Detection and Response

- Defender for Identity sensor on the DC with sensitive account tagging and a honeytoken account
- Attack simulations run against AD: Kerberoasting, LDAP reconnaissance, and DCSync. LDAP recon confirmed captured in IdentityQueryEvents. MDI honeytoken alert fired automatically confirming end-to-end sensor detection
- Identity Protection risk policies: sign-in risk triggers MFA, user risk triggers secure password change
- Sentinel KQL detections correlating on-prem and cloud identity signals
- SOAR playbook that auto-contains high-risk privileged accounts: revoke sessions, disable account, notify SOC — scoped to admin role holders only so it does not override self-service remediation for standard users

### Part 6: Enterprise IGA and PAM

- SailPoint Identity Security Cloud: understood deeply as a platform — source aggregation, identity warehouse, certification campaigns, role mining, and cross-application SoD policy enforcement across SAP, Oracle, and other non-Microsoft systems. Hands-on access requires a customer license or SailPoint partner status; actively pursuing ambassador status in the SailPoint developer community to get a tenant
- Cross-application SoD (SAP vendor-create conflicting with Oracle payment-approve) is the capability Entra cannot provide and the main reason regulated enterprises buy a dedicated IGA platform
- Workload identity federation: GitHub Actions authenticating to Azure via OIDC with zero stored secrets — repo holds only IDs, nothing to rotate or leak
- CyberArk covered conceptually, mapping each PIM concept to its PAM equivalent (vaulting, session recording, rotation)

### Part 7: Network Identity Controls

Three controls that extend Zero Trust beyond the directory to how resources are accessed over the network:

- **Application Proxy**: published an internal IIS site as an enterprise application through Entra ID without opening firewall ports. Pre-authentication set to Microsoft Entra ID so no request reaches the backend without a valid token and Conditional Access policies enforced at the edge
- **Entra Private Access**: configured Global Secure Access Quick Access with an application segment targeting the DC's internal IP, enabled the Private Access traffic forwarding profile, and installed the GSA client on the hybrid-joined endpoint. Internal resources accessible without a VPN, with identity-based access controls on every connection
- **Private Endpoints**: deployed a Key Vault with public access disabled, connected to a dedicated VNet subnet via private endpoint, with Azure-managed private DNS zone resolving the vault hostname to its private IP. Credentials used by app registrations and managed identities are only reachable from trusted network paths

## Lessons That Cost Time

- employeeHireDate on synced users has exactly one supported path and it is not the portal or Graph. The AD attribute plus sync rule approach is documented in the project guide because nothing online lays it out end to end
- Sync timing matters more than expected. The export to Entra ID took far longer than the standard sync cycle suggests. Verify each pipeline stage (AD, metaverse, Entra ID) separately before assuming a configuration problem
- Disabling an account does not kill active tokens. Sessions stay valid up to an hour unless explicitly revoked, which is why every leaver path revokes sessions and why CAE matters
- A cloned sync rule at the wrong precedence can break attribute flow for every synced user. The bad clone was disabled and kept as evidence rather than deleted
- MDI attack detections (Kerberoasting, DCSync) require the sensor to build a behavioral baseline over time. LDAP recon and honeytoken detections are more immediate in a fresh lab environment

## Documentation

Full project write-up with configuration detail and screenshots: [Enterprise_IAM_Project.pdf](docs/Enterprise_IAM_Project.pdf)

---
