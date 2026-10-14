# Detection Engineering — Windows Failed Logon Detection

## Objective

Design, configure and validate a custom security detection rule in Elastic Security to identify repeated failed Windows authentication attempts.

The objective was to develop a threshold-based detection capable of identifying potentially suspicious authentication activity using Windows Security Event ID 4625.

The detection was designed to:

- Monitor failed authentication events collected from WindowsEndpoint.
- Identify multiple failed logon attempts targeting the same user account.
- Generate a security alert when a defined threshold was reached.
- Provide relevant event information to support SOC alert triage and investigation.
- Validate detection effectiveness through controlled authentication testing.

A custom rule named **Repeated Windows Failed Logons** was created using Kibana's Elastic Security detection engine.

The rule was configured to generate a Medium-severity alert when three or more failed authentication events were observed for the same target account within the detection window.

This implementation demonstrates the process of translating suspicious authentication behaviour into automated detection logic, validating that logic against real security telemetry and investigating the resulting alerts.

## Detection Rule Configuration

### Rule Overview

A custom threshold-based detection rule was created in Elastic Security to identify repeated failed authentication attempts against Windows user accounts.

| Configuration | Value |
|---|---|
| Rule Name | Repeated Windows Failed Logons |
| Rule Type | Threshold |
| Data Source | `logs-*` |
| Severity | Medium |
| Risk Score | 47 |
| Execution Interval | Every 5 minutes |
| Additional Look-back | 1 minute |
| Threshold | 3 failed authentication events |
| Group By | `winlog.event_data.TargetUserName` |
| Alert Suppression | Disabled |

### KQL Detection Query

The following Kibana Query Language (KQL) query was configured:

`agent.name:"WindowsEndpoint" AND winlog.event_id:4625`

The query filters incoming security telemetry to identify Windows Event ID 4625 records originating from the monitored WindowsEndpoint.

### Threshold Logic

The rule groups matching authentication failures by the targeted username using:

`winlog.event_data.TargetUserName`

An alert is generated when **three or more matching failed authentication events** are identified for the same targeted account within the rule's evaluation window.

Grouping by username allows the detection to distinguish between authentication failures affecting different accounts.

### Detection Schedule

The rule was configured to execute every five minutes with an additional one-minute look-back period.

This allows the detection engine to evaluate recently collected security events while accounting for a small amount of ingestion delay.

### Severity and Risk Classification

The rule was assigned:

- **Severity:** Medium
- **Risk Score:** 47

These values represent the initial classification of repeated authentication failures as potentially suspicious activity requiring investigation.

An alert does not independently prove malicious intent. Further analysis is required to establish whether the failures resulted from legitimate user error, misconfiguration or attempted unauthorised access.

## Detection Rule Testing & Validation

### Testing Methodology

Following the creation of the custom detection rule, controlled authentication failures were generated on WindowsEndpoint to validate the detection logic.

The test used the account name `FakeSOCUser` and deliberately incorrect credentials to produce Windows Security Event ID 4625.

The objective was to verify that:

- Windows recorded the failed authentication attempts.
- Elastic Agent successfully collected the security events.
- Elasticsearch indexed the events.
- The custom threshold rule identified repeated failures targeting the same account.
- Elastic Security generated an alert when the configured threshold was met.

### Test Execution

Repeated authentication failures were generated using the Windows `net use` command with invalid credentials.

Three failed authentication events were produced for the `FakeSOCUser` account.

The events were subsequently verified in Kibana Discover using the `logs-*` data view.

The following KQL query was used to identify the relevant events:

`agent.name:"WindowsEndpoint" AND winlog.event_id:4625`

### Detection Results

The custom **Repeated Windows Failed Logons** rule successfully generated an alert after the configured threshold was reached.

The resulting alert contained the following information:

| Alert Attribute | Observed Result |
|---|---|
| Detection Rule | Repeated Windows Failed Logons |
| Targeted Account | `FakeSOCUser` |
| Event ID | 4625 |
| Threshold Count | 3 |
| Severity | Medium |
| Risk Score | 47 |
| Detection Status | Alert generated successfully |

The threshold result was verified using the Elastic alert field:

`kibana.alert.threshold_result.count = 3`

This confirmed that the rule grouped the failed authentication events and triggered when the threshold condition was satisfied.

### Alert Investigation

The generated alert was reviewed through the Elastic Security interface.

The investigation confirmed that:

- The authentication failures targeted the designated test account.
- The observed activity matched the configured detection criteria.
- The events were deliberately generated during controlled testing.
- No unauthorised access occurred.

An investigation note was added to document the findings, and the alert was closed as expected test activity.

### Validation Outcome

The test successfully demonstrated the complete detection workflow:

**Windows Authentication Failures → Security Event Logs → Elastic Agent → Elasticsearch → Detection Rule → Security Alert**

The successful validation established confidence that the rule could identify repeated failed authentication attempts.

The detection was subsequently tested against remote authentication activity originating from the Kali Linux VM, as documented in the attack simulation and incident investigation sections.

## Detection Limitations & Potential Improvements

### Current Detection Limitations

Although the custom detection rule successfully identified repeated Windows authentication failures, several limitations were identified.

| Limitation | Explanation | Potential Improvement |
|---|---|---|
| False positives | Legitimate users entering incorrect passwords repeatedly may trigger alerts. | Correlate failures with account behaviour, source information and surrounding events. |
| Fixed threshold | The rule requires three failed logons, meaning fewer attempts may not generate an alert. | Evaluate different thresholds based on normal authentication activity. |
| Username-based grouping | Events are grouped by targeted username without distinguishing their source IP addresses. | Incorporate source information to improve investigation context and detection precision. |
| Successful logons | The rule detects Event ID 4625 but does not automatically correlate subsequent Event ID 4624 successes. | Develop correlation logic to identify successful authentication following repeated failures. |
| Detection delay | The five-minute execution schedule means alerts are not necessarily generated immediately. | Evaluate shorter execution intervals where operational resources permit. |

### False Positive Considerations

Repeated failed authentication attempts do not always indicate malicious activity.

Potential legitimate causes include:

- Users entering incorrect passwords.
- Applications attempting authentication with outdated credentials.
- Misconfigured services or scheduled tasks.
- Temporary account or authentication configuration issues.

An analyst must review the surrounding context before classifying an alert as malicious.

### Potential Detection Enhancements

Future improvements could include:

1. **Source correlation:** Analysing source IP addresses and workstation names alongside targeted accounts.
2. **Failure-to-success correlation:** Identifying successful authentication events following repeated failures.
3. **Account monitoring:** Applying additional scrutiny to privileged or sensitive accounts.
4. **Baseline analysis:** Comparing authentication activity against normal endpoint behaviour.
5. **Detection tuning:** Adjusting thresholds and execution intervals to improve detection effectiveness while reducing unnecessary alerts.

### Evaluation

The detection rule achieved its intended objective by identifying repeated failed authentication events and generating actionable security alerts.

However, the rule represents an initial detection capability rather than a complete authentication threat-detection solution.

The identified improvements provide opportunities for future detection engineering and security monitoring development.

## Results & Lessons Learned

### Key Achievements

- Developed and implemented a custom threshold-based detection rule using Elastic Security.
- Applied Kibana Query Language (KQL) to filter Windows authentication failures.
- Configured username-based grouping to identify repeated failures against individual accounts.
- Successfully generated security alerts when the configured threshold of three failed logons was reached.
- Verified alert attributes, including threshold count, targeted username, severity and risk score.
- Investigated test alerts, documented findings and completed alert closure.
- Identified detection limitations and opportunities for future improvement.

### Lessons Learned

**Detection engineering:** Gained practical experience translating suspicious authentication behaviour into automated detection logic.

**Rule validation:** Demonstrated the importance of generating controlled test events and confirming that the resulting alerts match the expected detection criteria.

**Alert triage:** Developed experience reviewing security alerts, examining supporting telemetry and documenting investigation outcomes.

**Detection limitations:** Recognised that a threshold-based rule can identify suspicious patterns without necessarily establishing whether authentication activity is malicious.

**Continuous improvement:** Identified opportunities to improve detection accuracy through source correlation, behavioural analysis and failure-to-success authentication correlation.

### Conclusion

The custom **Repeated Windows Failed Logons** rule successfully detected controlled authentication failures on WindowsEndpoint.

The implementation demonstrated a complete initial detection-engineering workflow, from rule configuration and testing to alert investigation and validation.

The rule subsequently supported the investigation of remote authentication attempts generated from Kali Linux.

These activities established the foundation for the next project stages: **Attack Simulation** and **Incident Investigation**, where the detection was evaluated against more realistic authentication attack scenarios.

