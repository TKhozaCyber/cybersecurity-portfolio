# SOC Incident Report — SMB Authentication & Password-Guessing Investigation

## Incident Information

| Field | Details |
|---|---|
| Incident Type | Repeated Failed Authentication / Password Guessing |
| Investigation Date | 12 October 2026 |
| Environment | Isolated VirtualBox SOC Homelab |
| Detection Platform | Elastic Security SIEM |
| Detection Rule | Repeated Windows Failed Logons |
| Alert Severity | Medium |
| Risk Score | 47 |
| Affected System | WindowsEndpoint — `192.168.56.101` |
| Source System | Kali-Attack — `192.168.56.102` |
| Primary Accounts Investigated | `FakeSOCUser`, `HydraLabUser` |
| Investigation Status | Closed |
| Final Classification | True-positive detections of authorised laboratory testing |

## 1. Executive Summary

On 12 October 2026, multiple Windows authentication failures were generated during controlled security testing within an isolated SOC homelab.

The activity originated from a Kali Linux virtual machine and targeted a monitored Windows 11 endpoint using SMB over TCP port 445.

Elastic Security detected repeated failed authentication attempts through a custom threshold-based detection rule named **Repeated Windows Failed Logons**.

The investigation involved two primary remote authentication scenarios:

- **SMB authentication testing:** Three unsuccessful authentication attempts targeting `FakeSOCUser`.
- **Hydra password-guessing simulation:** Three unsuccessful authentication attempts targeting `HydraLabUser`, followed by one successful network authentication.

### Detection and Investigation

Windows recorded the failed authentication attempts as Security Event ID 4625 and the successful authentication as Event ID 4624.

These events were collected through Elastic Agent, indexed in Elasticsearch and analysed using Kibana Discover.

The custom detection rule generated Medium-severity alerts after the configured threshold of three failed authentication events was reached.

Further investigation established that the Hydra simulation resulted in successful authentication after three failed attempts.

Windows Security records confirmed that the authentication activity originated from `192.168.56.102`, used Logon Type 3 and involved NTLM authentication.

### Findings and Response

The investigation confirmed that the activity was intentionally generated as part of authorised security testing.

Although the Hydra simulation successfully authenticated to the temporary test account, no unauthorised access or privilege escalation was demonstrated.

The investigation findings were documented in Elastic Security, and the resulting alerts were closed.

Following testing, the temporary Hydra account, password list and SMB firewall exception were removed.

### Security Significance

The investigation highlighted two important security considerations:

1. Exposing SMB services can create additional opportunities for network authentication attacks.
2. Repeated authentication failures followed by successful authentication may indicate password-guessing activity requiring investigation.

The exercise also identified an opportunity to improve the existing detection rule by correlating failed authentication attempts with subsequent successful logons.

### Final Outcome

The monitoring environment successfully collected, detected and supported the investigation of controlled authentication attack activity.

The incident was classified as authorised laboratory testing rather than a genuine security breach.

The investigation demonstrated practical experience in SIEM monitoring, security alert triage, Windows event correlation, incident documentation and response verification.

## 2. Incident Timeline

### Overview

The investigation involved two controlled authentication scenarios conducted on 12 October 2026.

The first scenario validated remote SMB authentication monitoring and detection. The second simulated password guessing using Hydra, resulting in three failed authentication attempts followed by a successful network authentication.

The timeline was reconstructed using Nmap results, SMBClient and Hydra output, Windows Security Event Logs and Elastic Security investigation records.

### Chronological Timeline

| Time (UK) | Event | Evidence / Outcome |
|---|---|---|
| Morning | Initial Nmap reconnaissance against WindowsEndpoint | All 1,000 scanned TCP ports reported as filtered |
| Morning | Restricted Windows Firewall exception configured | SMB/TCP 445 permitted from Kali (`192.168.56.102`) |
| Morning | Follow-up Nmap reconnaissance | TCP 445 confirmed open |
| Morning | SMBClient authentication testing | Invalid credentials rejected with `NT_STATUS_LOGON_FAILURE` |
| 06:50:25 | First recorded SMB authentication failure | Windows Event ID 4625 |
| 06:50:29 | Second recorded authentication failure | Windows Event ID 4625 |
| 06:50:33 | Third recorded authentication failure | Windows Event ID 4625 |
| Approximately 06:51 | Elastic Security alert generated | Custom threshold rule detected three failures |
| Following alert generation | SMB investigation conducted | Kali source identified, findings documented and alert closed |
| Evening | Hydra password-guessing simulation initiated | Four candidate passwords tested against `HydraLabUser` |
| 21:31:36 | First Hydra authentication failure | Windows Event ID 4625 |
| 21:31:38 | Second Hydra authentication failure | Windows Event ID 4625 |
| 21:31:41 | Third Hydra authentication failure | Windows Event ID 4625 |
| 21:31:43 | Successful network authentication | Windows Event ID 4624 |
| Following authentication | Elastic Security investigation | Failed and successful events correlated |
| Following investigation | Analyst documentation and alert closure | Investigation note recorded; alert marked Closed |
| After testing | Environment cleanup | Temporary account, password list and SMB firewall exception removed |

*Exact times are included where available from captured event records. Other activities are presented in their observed sequence without assigning unsupported timestamps.*

### Phase 1 — Reconnaissance and Initial Authentication Testing

The investigation began with network reconnaissance from Kali Linux against WindowsEndpoint.

Initial results showed that inbound TCP ports were filtered. A temporary source-restricted firewall exception was then configured to permit SMB authentication testing over TCP port 445.

SMBClient generated three failed network authentication attempts against `FakeSOCUser`.

Windows recorded these events as Event ID 4625, identifying Kali as the originating workstation.

Elastic Security subsequently generated an alert through the custom failed-logon detection rule.

The activity was investigated and classified as expected laboratory testing.

### Phase 2 — Hydra Password-Guessing Simulation

Later that day, Hydra was used to test a controlled list of four candidate passwords against the temporary account `HydraLabUser`.

Windows recorded three failed authentication attempts followed by one successful network authentication.

The successful logon occurred approximately seven seconds after the first recorded failure.

The events shared the same targeted account and originated from the designated Kali attacker VM.

### Phase 3 — Detection, Investigation and Response

Elastic Security generated an alert for the repeated authentication failures.

Further investigation using Kibana Discover established that the Hydra simulation had also produced a successful Event ID 4624.

The analyst reviewed the authentication sequence, identified the source system and documented the findings within Elastic Security.

The alert was closed after confirming the activity was an authorised test.

Following the investigation, temporary testing permissions and credentials were removed.

### Timeline Findings

The reconstructed sequence established that:

1. Network access restrictions initially prevented direct SMB connectivity.
2. A controlled firewall exception enabled authentication testing.
3. Failed authentication attempts generated identifiable Windows Security events.
4. The custom detection rule successfully identified repeated failures.
5. The Hydra simulation resulted in successful network authentication.
6. Correlation of Events 4625 and 4624 provided additional context beyond the original alert.
7. Investigation findings were documented and the alerts closed.
8. Temporary testing access was removed following the simulations.

### Timeline Conclusion

The event timeline demonstrated the progression from initial reconnaissance through authentication testing, automated detection, event correlation and investigation closure.

The sequence also illustrated how combining attacker-side observations with endpoint and SIEM telemetry provides a more complete understanding of security activity.

## 3. Technical Investigation Findings

### 3.1 Detection and Alert Analysis

The investigation began with alerts generated by the custom **Repeated Windows Failed Logons** rule in Elastic Security.

The rule monitored Windows Event ID 4625 and generated a threshold-based alert when three failed authentication events were detected against the same targeted username.

The generated alerts contained the following attributes:

| Attribute | Observed Value |
|---|---|
| Detection Rule | Repeated Windows Failed Logons |
| Rule Type | Threshold |
| Severity | Medium |
| Risk Score | 47 |
| Threshold Count | 3 |
| Monitored Endpoint | WindowsEndpoint |
| Relevant Event ID | 4625 |

The detection operated as configured and successfully identified repeated authentication failures during both remote authentication scenarios.

### 3.2 Source Attribution

Windows Security records were examined to establish the origin of the authentication activity.

The relevant records contained:

| Field | Observed Value |
|---|---|
| Source IP Address | `192.168.56.102` |
| Source Workstation | `KALI` |
| Destination System | WindowsEndpoint |
| Destination IP Address | `192.168.56.101` |
| Targeted Accounts | `FakeSOCUser`, `HydraLabUser` |
| Logon Type | 3 — Network |
| Authentication Package | NTLM |

The source IP address matched the Kali Linux virtual machine used for the controlled attack simulations.

This attribution was supported by the SMBClient and Hydra output captured on the attacker system.

### 3.3 Failed Authentication Analysis

Windows recorded unsuccessful authentication attempts using Event ID 4625.

The records identified the targeted accounts and provided evidence of network-based authentication attempts originating from Kali.

During the remote SMBClient scenario, three failed logon events were observed against `FakeSOCUser`.

During the Hydra scenario, three failed logon events were observed against `HydraLabUser`.

Both sequences triggered the configured detection rule.

### 3.4 Successful Authentication Analysis

The Hydra investigation identified a subsequent successful authentication event.

Windows Event ID 4624 confirmed that `HydraLabUser` successfully authenticated from the Kali attacker VM using a network logon.

The relevant sequence was:

| Event Time | Event ID | Result |
|---|---|---|
| 21:31:36 | 4625 | Failed authentication |
| 21:31:38 | 4625 | Failed authentication |
| 21:31:41 | 4625 | Failed authentication |
| 21:31:43 | 4624 | Successful authentication |

The successful Event ID 4624 was consistent with Hydra reporting that a valid credential had been identified.

The evidence confirmed successful network authentication to the designated test account.

It did not demonstrate administrative access, privilege escalation, lateral movement or an interactive desktop session.

### 3.5 Detection Capability and Limitations

The existing detection rule successfully identified repeated authentication failures.

However, the rule was configured specifically to monitor Event ID 4625. It did not automatically correlate those failures with the subsequent Event ID 4624.

The successful authentication was identified through additional investigation using Kibana Discover.

This distinction highlighted an opportunity to improve the detection capability by introducing failure-to-success authentication correlation.

### 3.6 Supporting Technical Evidence

The following evidence supports the investigation findings:

| Evidence | Purpose |
|---|---|
| [SMB failed authentication](../05-attack-simulation/screenshots/03-smbclient-failed-authentication.png) | Confirms unsuccessful remote SMB authentication |
| [Windows 4625 source information](../05-attack-simulation/screenshots/05-windows-4625-kali-source.png) | Identifies the Kali workstation and source IP |
| [Hydra simulation](../05-attack-simulation/screenshots/06-hydra-password-guessing.png) | Demonstrates controlled password-guessing activity |
| [Authentication event sequence](../05-attack-simulation/screenshots/07-hydra-authentication-event-sequence.png) | Shows three failed events followed by one successful event |
| [Successful Windows authentication](../05-attack-simulation/screenshots/08-windows-4624-successful-logon.png) | Confirms successful Logon Type 3 authentication |
| [Hydra investigation and alert closure](../06-incident-investigation/screenshots/02-hydra-investigation-note-closed.png) | Demonstrates documented investigation findings and closed alert |

### Technical Findings Conclusion

The collected evidence confirmed that the monitored endpoint successfully recorded authentication activity originating from Kali Linux.

Elastic Security detected repeated authentication failures, while further log analysis identified the successful authentication that followed the Hydra password-guessing attempts.

The investigation demonstrated the value of combining endpoint telemetry, automated security detection and manual event correlation to establish an accurate understanding of security activity.

The observed activity was consistent with the authorised laboratory tests and did not establish any unauthorised access to the environment.

## 4. Risk Assessment, Remediation & Incident Resolution

### 4.1 Security Risk Assessment

The investigation identified two primary security risks associated with the controlled authentication simulations.

| Security Risk | Likelihood | Impact | Risk Rating |
|---|---|---|---|
| SMB service exposure through TCP 445 | Medium | Medium | **Medium (4/9)** |
| Password guessing resulting in successful network authentication | Medium | High | **High (6/9)** |

These ratings represent illustrative risks under the assessed conditions and do not indicate that a production environment was compromised.

**SMB Service Exposure**

The initial Nmap scan demonstrated that Windows Firewall filtered inbound connections.

A temporary firewall exception subsequently allowed SMB connectivity from Kali Linux, enabling remote authentication testing.

This highlighted how network service exposure can increase opportunities for unauthorised authentication attempts when access is insufficiently restricted.

**Password-Guessing Risk**

The Hydra simulation demonstrated that repeated password attempts could result in successful network authentication when valid credentials were supplied.

Although the activity was authorised, a comparable event involving an unauthorised actor could potentially result in access to resources available to the affected account.

The test did not establish that Windows account lockout protections were absent or that privilege escalation occurred.

### 4.2 Immediate Response Actions

Following the investigations, the following actions were completed:

| Response Action | Purpose | Status |
|---|---|---|
| Investigated Elastic Security alerts | Establish the cause and outcome of authentication activity | Completed |
| Correlated Windows Events 4625 and 4624 | Determine whether failed logons were followed by successful authentication | Completed |
| Documented investigation findings | Maintain an auditable record of the analyst's findings | Completed |
| Closed investigated alerts | Record the final disposition of authorised testing | Completed |
| Removed `HydraLabUser` | Eliminate the temporary test account | Completed |
| Removed the SMB firewall exception | Remove deliberately permitted testing access | Completed |
| Deleted the Hydra password file | Remove temporary plaintext test credentials from Kali | Completed |

The historical Windows authentication records and Elastic Security investigation evidence were retained for documentation and future analysis.

### 4.3 Remediation Verification

The investigation distinguished between completed remediation actions and independently verified outcomes.

The removal of the temporary account and firewall exception was completed during laboratory cleanup.

However, additional post-remediation verification was not captured.

Recommended checks include:

- Confirming that `HydraLabUser` no longer exists.
- Verifying that the temporary SMB firewall rule has been removed.
- Repeating the Nmap scan to confirm the endpoint's network exposure.
- Reviewing Windows authentication and account lockout policies.
- Confirming that Elastic Security monitoring remains operational.

These checks would strengthen assurance that the intended security configuration was restored.

### 4.4 Recommended Security Improvements

The following improvements were identified:

**Authentication Security**
- Review account lockout policies and password requirements.
- Apply least-privilege access controls.
- Monitor suspicious authentication patterns involving sensitive accounts.

**Network Security**
- Restrict SMB access to authorised systems.
- Remove unnecessary firewall exceptions.
- Maintain operating system and service security updates.

**Detection Engineering**
- Implement correlation between repeated failed logons and subsequent successful authentication.
- Incorporate source IP addresses into authentication investigations.
- Tune detection thresholds based on observed behaviour and false-positive rates.

**Incident Response**
- Standardise alert triage procedures.
- Maintain clear investigation timelines and supporting evidence.
- Verify remediation outcomes before formally confirming that identified risks have been reduced.

### 4.5 Incident Resolution

The investigated alerts were closed after the activity was confirmed as authorised security testing.

**Final Classification:** True-positive detections of controlled laboratory activity.

**Authentication Outcome:** Successful network authentication occurred during the Hydra simulation using the temporary test account.

**Confirmed Security Impact:** No unauthorised access, privilege escalation or lateral movement was demonstrated.

**Response Status:** Investigation completed, alerts closed and temporary testing access removed.

**Outstanding Verification:** Independent post-remediation account and firewall checks were not captured.

The laboratory investigation was therefore concluded, while the identified security improvements remain recommendations for future implementation and validation.

### 4.6 Risk and Remediation Conclusion

The investigation demonstrated how network exposure and authentication behaviour can create security risks requiring appropriate preventative and detective controls.

The laboratory response successfully documented the observed activity and removed the temporary testing configuration.

Further remediation verification and improved authentication correlation would strengthen the security posture and detection capabilities of the environment.

A more detailed assessment, including ISO/IEC 27001:2022 control mapping, is available in [Section 7 — Risk Assessment & Remediation](../07-risk-assessment-remediation/README.md).

## 5. Final Conclusions & Lessons Learned

### 5.1 Investigation Conclusion

The SOC investigation successfully demonstrated the detection, analysis and documentation of suspicious Windows authentication activity within a controlled cybersecurity homelab.

The custom **Repeated Windows Failed Logons** detection rule identified repeated authentication failures generated through SMBClient and Hydra testing.

Further analysis of Windows Security events established that the Hydra simulation resulted in three unsuccessful authentication attempts followed by one successful network authentication.

The investigation confirmed that:

- The activity originated from the designated Kali Linux attacker VM.
- Windows recorded the authentication attempts using Event IDs 4625 and 4624.
- Elastic Agent successfully collected the authentication events.
- The custom detection rule generated alerts for repeated authentication failures.
- Additional analysis identified the successful authentication that followed the failures.
- Investigation findings were documented and the alerts closed.

The activity was confirmed as authorised laboratory testing, with no unauthorised access or privilege escalation demonstrated.

### 5.2 Key Lessons Learned

**Detection is the beginning of an investigation**

An automated alert identifies behaviour matching a detection condition. Reviewing supporting events is necessary to determine what actually occurred.

**Authentication events must be correlated**

Analysing failed and successful authentication records together revealed information that the initial failed-logon detection did not capture automatically.

**Source attribution improves investigation accuracy**

The source IP address, workstation name, logon type and authentication protocol helped establish where the activity originated and how authentication occurred.

**Security controls must be verified**

Firewall restrictions and account controls influence exposure to authentication attacks. Following configuration changes, verification is necessary to confirm that the intended security posture has been restored.

**Clear documentation supports incident response**

Maintaining investigation notes, timelines, supporting evidence and alert dispositions makes the investigation understandable and repeatable.

### 5.3 Recommended Future Improvements

The investigation identified several improvements that could strengthen the SOC environment:

1. Develop detection logic correlating repeated authentication failures with subsequent successful logons.
2. Review Windows account lockout policies and strengthen authentication controls where appropriate.
3. Include source IP addresses and account context in detection tuning.
4. Perform and document post-remediation firewall and account verification.
5. Establish a repeatable SOC investigation checklist for future security events.

These recommendations represent opportunities for further development and were not all implemented during this project.

### 5.4 Final Incident Disposition

| Category | Final Outcome |
|---|---|
| Incident Classification | Authorised laboratory attack simulation |
| Detection Assessment | True-positive detections of repeated authentication failures |
| Authentication Outcome | Successful network authentication demonstrated during Hydra testing |
| Confirmed Unauthorised Access | None |
| Privilege Escalation | Not observed |
| Investigation Status | Completed |
| Alert Status | Closed |
| Temporary Testing Access | Removed |
| Outstanding Actions | Additional remediation verification and detection improvements |

### 5.5 Overall Project Outcome

The investigation demonstrated a practical SOC workflow, progressing from security monitoring and automated detection through event correlation, source attribution, risk assessment and documented response.

The project provided hands-on experience operating Elastic Security, interpreting Windows authentication telemetry and producing evidence-based investigation findings.

The completed investigation and supporting technical documentation demonstrate foundational capabilities applicable to SOC Analyst and Security Analyst responsibilities.

**Final Status: Investigation completed and documented.**
