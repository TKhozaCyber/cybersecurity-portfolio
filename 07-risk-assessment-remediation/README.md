# Security Risk Assessment, Remediation & ISO 27001 Mapping

## Objective

Conduct a structured security risk assessment based on the findings from the SOC homelab's network reconnaissance, SMB authentication testing and Hydra password-guessing simulation.

The objective was to identify potential security weaknesses, evaluate their likelihood and impact, recommend appropriate remediation measures, and map relevant security practices to ISO/IEC 27001:2022 controls.

The assessment focuses on:

- Network service exposure and firewall configuration.
- Authentication security and password-guessing risks.
- Account access controls and authentication monitoring.
- Security logging and detection capabilities.
- Remediation measures and verification evidence.

The assessment uses findings from the controlled lab environment to demonstrate practical risk analysis and security improvement.

## Risk Assessment Methodology

### Assessment Approach

A qualitative risk assessment was used to evaluate security weaknesses identified during the project.

Each finding was assessed using the following criteria:

| Criterion | Description |
|---|---|
| Security Finding | The identified weakness or potential exposure |
| Supporting Evidence | Observations from reconnaissance, authentication testing or security logs |
| Likelihood | The estimated probability of the weakness being exploited |
| Impact | The potential consequences of successful exploitation |
| Risk Rating | The overall risk based on likelihood and impact |
| Recommended Remediation | Security measures intended to reduce the identified risk |
| Verification | Evidence used to confirm whether the remediation was implemented and effective |

### Risk Rating System

Likelihood and impact were each assessed on a three-point scale.

| Rating | Likelihood | Impact |
|---|---|---|
| 1 — Low | Exploitation is unlikely under the assessed conditions | Limited disruption or security consequences |
| 2 — Medium | Exploitation is plausible under certain conditions | Moderate operational or security consequences |
| 3 — High | Exploitation is likely when the required conditions exist | Significant potential compromise or disruption |

An overall risk score was calculated using:

**Risk Score = Likelihood × Impact**

| Score | Risk Level |
|---|---|
| 1–2 | Low |
| 3–4 | Medium |
| 6–9 | High |

The ratings are qualitative estimates intended for this educational lab assessment, rather than measurements of risk in a production organisation.

### Assessment Limitations

The testing was performed within an isolated VirtualBox homelab using temporary accounts and controlled network permissions.

Findings were classified according to the available evidence. Observed behaviour was distinguished from potential vulnerabilities that would require additional configuration checks to confirm.

The assessment does not claim that the controlled laboratory activity represents a compromise of a production environment.

Recommendations are intended to demonstrate how the identified risks could be mitigated using appropriate security controls.

## Identified Security Risks

### Risk 1 — SMB Network Service Exposure

**Risk Description**

During reconnaissance, WindowsEndpoint initially reported all 1,000 scanned TCP ports as filtered.

A temporary firewall exception was subsequently configured to permit SMB traffic on TCP port 445 exclusively from the Kali attacker VM (`192.168.56.102`).

A follow-up Nmap scan confirmed that TCP port 445 was accessible, while the remaining 999 scanned ports continued to be filtered.

Although this access was intentionally configured for testing, exposing SMB to an untrusted or compromised source could increase the opportunity for unauthorised network authentication attempts.

**Risk Assessment**

| Criterion | Assessment |
|---|---|
| Identified Risk | SMB network service exposure |
| Likelihood | 2 — Medium |
| Impact | 2 — Medium |
| Risk Score | 4 |
| Overall Risk | **Medium** |
| Supporting Evidence | Initial and follow-up Nmap scans |
| Status | Temporary lab exception removed after testing |

**Potential Impact**

If equivalent access existed in an inadequately protected environment, attackers could attempt credential guessing, identify accessible shares or exploit applicable SMB vulnerabilities.

The actual risk would depend on the service configuration, network restrictions, system patching and authentication controls.

**Recommended Remediation**

- Restrict inbound SMB connections to explicitly authorised systems.
- Avoid exposing TCP port 445 to untrusted networks.
- Apply least-privilege firewall rules.
- Disable unnecessary SMB services and maintain security updates.
- Review firewall exceptions periodically and remove temporary rules after testing.

**Remediation Performed**

The temporary firewall exception allowing TCP 445 from Kali was removed following the attack simulations.

**Verification Evidence**

The initial and follow-up Nmap screenshots demonstrate the effect of the controlled firewall change.

The removal of the temporary exception was completed during lab cleanup. However, a post-remediation Nmap screenshot was not collected, so restoration of the original filtered state has not been independently evidenced.

---

### Risk 2 — Password Guessing and Successful Authentication

**Risk Description**

The Hydra simulation demonstrated how an attacker with SMB connectivity could attempt multiple passwords against a Windows account.

During testing, three unsuccessful authentication attempts against `HydraLabUser` were followed by one successful authentication.

Windows recorded:

- Three Event ID 4625 authentication failures.
- One Event ID 4624 successful network authentication.
- Logon Type 3.
- Source IP address `192.168.56.102`.

The successful authentication demonstrated that possession of valid credentials could allow network authentication to the exposed service.

The password was intentionally included in a controlled test list. Therefore, this result does not independently establish that a production password policy was weak or that account lockout protections were absent.

**Risk Assessment**

| Criterion | Assessment |
|---|---|
| Identified Risk | Password-guessing attempts leading to valid credential use |
| Likelihood | 2 — Medium |
| Impact | 3 — High |
| Risk Score | 6 |
| Overall Risk | **High** |
| Supporting Evidence | Hydra output and Windows Events 4625/4624 |
| Status | Temporary lab account deleted after testing |

The risk rating represents the potential consequences of comparable activity against a real account in an exposed environment, rather than the impact of this authorised laboratory simulation.

**Potential Impact**

Successful unauthorised authentication could potentially allow an attacker to access network resources available to the compromised account.

Depending on account permissions and additional security weaknesses, this could result in unauthorised data access, further account compromise or lateral movement.

The laboratory evidence confirmed successful network authentication only; it did not demonstrate privilege escalation or lateral movement.

**Recommended Remediation**

- Enforce strong password policies and block commonly used or compromised passwords.
- Configure and verify appropriate account lockout or authentication throttling controls.
- Apply least-privilege permissions to user accounts.
- Restrict unnecessary network authentication access.
- Monitor repeated authentication failures and subsequent successful logons.
- Investigate unusual authentication patterns originating from untrusted systems.

**Remediation Performed**

The temporary `HydraLabUser` account was deleted after testing, and the temporary SMB firewall exception was removed.

**Verification Evidence**

Hydra output and Windows Security logs confirmed three failed authentications followed by one successful network authentication.

The Elastic SIEM successfully recorded the authentication sequence.

The laboratory cleanup was completed, but the Windows account lockout policy was not independently assessed, and no post-remediation account or firewall verification screenshot was retained.

These remain opportunities for further security validation.

## Security Controls & Remediation Verification

### Security Controls Implemented

Several security measures were applied during the SOC homelab to maintain controlled access and support security monitoring.

| Security Control | Implementation | Purpose |
|---|---|---|
| Network isolation | Dedicated VirtualBox Host-Only network | Restrict laboratory attack traffic to the designated virtual environment |
| Firewall access control | SMB access permitted only from `192.168.56.102` | Limit access to the service required for testing |
| Least-privilege account | Standard local account used for Hydra testing | Avoid granting unnecessary administrative privileges |
| Centralised security logging | Windows Security logs collected through Elastic Agent | Provide visibility into authentication activity |
| Automated detection | Custom threshold rule monitoring Event ID 4625 | Identify repeated authentication failures |
| Investigation and response | Alerts investigated, documented and closed | Establish an auditable security investigation process |

### Remediation Actions Performed

**1. Temporary SMB Firewall Exception Removal**

Following the authentication simulations, the temporary Windows Firewall rule permitting SMB traffic from Kali was removed.

This removed the deliberately configured inbound access exception.

**2. Temporary Test Account Removal**

The `HydraLabUser` account was deleted following the password-guessing simulation.

Removing the temporary account prevented its continued use with the credentials employed during testing.

**3. Temporary Password File Removal**

The controlled password list used by Hydra was deleted from Kali Linux.

Password values were also redacted from the screenshot prepared for the public GitHub portfolio.

**4. Continued Security Monitoring**

The Elastic detection rule was retained to support future monitoring and additional detection-engineering exercises.

### Remediation Verification

The project distinguished between actions completed during testing and evidence independently captured to verify their effectiveness.

| Remediation or Control | Available Evidence | Verification Status |
|---|---|---|
| Initial inbound firewall restrictions | Nmap scan showing 1,000 filtered TCP ports | Verified before testing |
| Restricted SMB access | Follow-up Nmap scan showing only TCP 445 open | Verified during testing |
| Temporary firewall exception removal | Cleanup action completed | Post-removal scan not captured |
| Temporary account deletion | Cleanup action completed | Post-deletion account check not captured |
| Security log collection | Windows authentication events visible in Kibana Discover | Verified |
| Failed-logon detection | Elastic alert with threshold count of 3 | Verified |
| Failed-to-success event correlation | Three Event ID 4625 records followed by Event ID 4624 | Verified through investigation |

### Recommended Additional Verification

To further strengthen remediation assurance, the following checks would be appropriate:

- Confirm that the temporary SMB firewall rule no longer exists.
- Repeat the Nmap scan to verify that TCP 445 is no longer accessible from Kali under the original conditions.
- Confirm that the temporary `HydraLabUser` account no longer exists.
- Review Windows account lockout and password policies.
- Validate that authentication monitoring remains operational following changes.
- Test enhanced detection logic that correlates repeated failures with successful authentication.

These checks are recommendations for further validation and are not presented as completed tests.

### Supporting Evidence

The following screenshots provide evidence of the observed security controls and attack results:

- [Initial Nmap scan — filtered ports](../05-attack-simulation/screenshots/01-nmap-initial-filtered-ports.png)
- [Follow-up Nmap scan — SMB port 445 open](../05-attack-simulation/screenshots/02-nmap-smb-port-445-open.png)
- [Hydra password-guessing simulation](../05-attack-simulation/screenshots/06-hydra-password-guessing.png)
- [Windows authentication event sequence](../05-attack-simulation/screenshots/07-hydra-authentication-event-sequence.png)

### Remediation Outcome

The laboratory cleanup removed the temporary test account, password list and deliberately configured SMB firewall exception.

The investigation also demonstrated the effectiveness of centralised Windows authentication logging and threshold-based failed-logon detection.

Additional post-remediation verification would strengthen the evidence that the endpoint's original network restrictions were fully restored.

The assessment highlights the importance of combining preventative security controls, continuous monitoring, documented response procedures and verification of remediation actions.

## ISO/IEC 27001:2022 Annex A Control Mapping

### Purpose

The security risks, technical controls and investigation activities identified in this project were mapped to relevant ISO/IEC 27001:2022 Annex A controls.

The mapping demonstrates how practical SOC monitoring, network security and incident investigation activities can support broader information security objectives.

This is an educational control-mapping exercise and does not represent a formal ISO 27001 compliance assessment or certification.

### Relevant Security Controls

| ISO 27001:2022 Control | Security Objective | Homelab Implementation or Finding |
|---|---|---|
| **A.8.20 — Networks security** | Protect networks against unauthorised access and security threats. | Windows Firewall initially filtered inbound traffic. A temporary SMB exception was restricted to Kali and removed after testing. |
| **A.8.22 — Segregation of networks** | Separate systems and network environments where appropriate. | VirtualBox Host-Only networking separated laboratory VM traffic from direct access by external network devices. |
| **A.8.5 — Secure authentication** | Protect authentication processes against unauthorised access. | Hydra demonstrated password-guessing risk. Stronger password and account lockout controls were identified as recommendations, not verified implementations. |
| **A.8.15 — Logging** | Maintain security-relevant event records. | Windows Security Event IDs 4625 and 4624 were collected through Elastic Agent and indexed in Elasticsearch. |
| **A.8.16 — Monitoring activities** | Monitor systems for anomalous behaviour. | A custom Elastic Security rule detected repeated failed authentication attempts and generated security alerts. |
| **A.5.25 — Assessment and decision on information security events** | Assess security events and determine whether they represent security incidents. | Alerts were investigated, authentication events correlated and authorised laboratory activity distinguished from an actual compromise. |
| **A.5.28 — Collection of evidence** | Support the identification, collection and preservation of relevant evidence. | Windows event records, Elastic alerts, investigation notes and attack screenshots were retained as laboratory evidence. |
| **A.8.9 — Configuration management** | Maintain and manage system configurations securely. | Firewall configuration changes were controlled and temporary testing permissions removed. Additional post-remediation verification was identified as necessary. |

### Control Mapping Analysis

**Network Security — A.8.20 and A.8.22**

The homelab demonstrated how network configuration and firewall restrictions influence service accessibility.

The initial Nmap scan identified filtered TCP ports, while the follow-up scan confirmed that a restricted firewall exception allowed SMB access.

These findings highlight the importance of network isolation, controlled service exposure and least-privilege firewall configuration.

**Authentication Security — A.8.5**

The Hydra simulation demonstrated that repeated password attempts could result in successful network authentication when a valid credential was tested.

The exercise highlighted the importance of strong authentication controls, appropriate account lockout policies and monitoring for unusual login activity.

The effectiveness of the Windows account lockout policy was not independently verified during this project.

**Security Logging and Monitoring — A.8.15 and A.8.16**

Elastic Agent collected Windows authentication logs and forwarded them to Elasticsearch.

Kibana Discover enabled security event analysis, while the custom detection rule generated alerts when repeated authentication failures were observed.

These activities demonstrated practical implementation of centralised security logging and automated monitoring.

**Security Event Assessment — A.5.25**

The investigation process involved evaluating alerts, examining the underlying authentication records and establishing whether the observed activity represented unauthorised access.

The alerts were confirmed as true-positive detections of controlled laboratory testing.

The activity was therefore documented as authorised simulation rather than an actual security breach.

**Evidence Collection — A.5.28**

Security evidence was retained through Windows event records, Kibana screenshots, detection alerts and investigation notes.

This supported the reconstruction of authentication activity and documentation of investigation findings.

The evidence was collected for educational purposes and was not managed under a formal forensic chain-of-custody procedure.

**Configuration Management — A.8.9**

Temporary configuration changes were introduced to support controlled security testing and removed when no longer required.

The assessment identified additional post-remediation verification as an improvement opportunity.

### Control Mapping Outcome

The ISO 27001 mapping demonstrated how the SOC homelab activities relate to network security, authentication, logging, monitoring, event assessment, evidence collection and configuration management.

The project provided practical examples of several security controls while also identifying areas requiring further validation.

The mapping is intended to show technical understanding and risk-based security thinking, rather than assert organisational compliance with ISO/IEC 27001.

## Risk Assessment Summary & Recommendations

### Summary of Findings

The assessment identified two primary security risks demonstrated through controlled testing.

| Security Risk | Likelihood | Impact | Risk Rating | Remediation Status |
|---|---|---|---|---|
| SMB network service exposure | Medium | Medium | **Medium (4/9)** | Temporary firewall exception removed; post-removal verification outstanding |
| Password guessing leading to successful authentication | Medium | High | **High (6/9)** | Temporary test account removed; authentication policy review outstanding |

These ratings represent potential risks under the assessed conditions rather than confirmed vulnerabilities in a production environment.

### Prioritised Security Recommendations

**Priority 1 — Strengthen Authentication Security**

- Review and enforce appropriate account lockout policies.
- Require strong passwords and prevent the use of known compromised credentials.
- Apply least-privilege permissions to user accounts.
- Monitor repeated authentication failures followed by successful logons.
- Consider stronger authentication mechanisms where supported.

**Priority 2 — Reduce Network Service Exposure**

- Restrict SMB connectivity to authorised systems and network segments.
- Remove unnecessary inbound firewall permissions.
- Review firewall configurations periodically.
- Keep Windows and network services updated.
- Verify that temporary testing exceptions are removed after use.

**Priority 3 — Improve Detection Capabilities**

- Develop correlation rules identifying successful authentication after repeated failures.
- Incorporate source IP addresses and workstation information into detection analysis.
- Review detection thresholds to balance visibility and false positives.
- Establish consistent alert triage and investigation procedures.

**Priority 4 — Strengthen Remediation Verification**

- Confirm temporary accounts have been removed.
- Verify the absence of temporary firewall exceptions.
- Perform post-remediation network scans.
- Retain verification evidence alongside risk assessment documentation.

### Residual Risk Considerations

Although the temporary account and firewall exception were removed, not every recommended security improvement was implemented or independently verified.

In particular, the Windows account lockout configuration was not formally assessed, and post-remediation network testing evidence was not captured.

Therefore, the assessment does not assign revised residual risk scores or claim that all identified risks have been eliminated.

Further verification would be required before determining the effectiveness of additional security controls.

## Lessons Learned

The assessment demonstrated several important security principles:

- **Risk-based thinking:** Security weaknesses should be evaluated according to both their likelihood and potential impact.
- **Defence in depth:** Firewall restrictions, secure authentication, endpoint logging and monitoring provide complementary protections.
- **Evidence-based assessment:** Security conclusions should be supported by observed technical findings rather than assumptions.
- **Remediation verification:** Implementing a fix is not the same as proving that the underlying risk has been sufficiently reduced.
- **Security governance:** Technical security activities can be related to recognised frameworks such as ISO/IEC 27001 to support wider information security objectives.

## Conclusion

The security risk assessment demonstrated how findings from network reconnaissance, authentication testing and SOC investigations can inform security improvement recommendations.

The project identified the potential consequences of SMB service exposure and password-guessing activity, evaluated the associated risks and documented appropriate remediation measures.

Mapping these findings to ISO/IEC 27001:2022 Annex A controls demonstrated how practical technical security activities relate to broader information security management practices.

The assessment also recognised the importance of distinguishing verified observations from potential vulnerabilities and outstanding remediation checks.

This completed the risk analysis supporting the final SOC incident report.
