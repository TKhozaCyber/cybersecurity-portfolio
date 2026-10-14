# Windows Security Log Monitoring & Analysis

## Objective

Configure and validate centralised Windows Security log monitoring using Elastic Agent and Elasticsearch, enabling authentication activity to be analysed through Kibana Discover.

The objective was to understand how Windows records authentication events, identify relevant security information within those records, and establish a reliable source of telemetry for threat detection and incident investigation.

The monitoring focused on two Windows Security Event IDs:

- **Event ID 4625** — Failed logon attempts.
- **Event ID 4624** — Successful logon attempts.

These events provide valuable information about authentication activity, including targeted user accounts, logon types, source IP addresses and authentication methods.

The collected telemetry would subsequently support custom detection rules and controlled security investigations within the SOC homelab.

## Windows Event Log Collection & Configuration

### WindowsEndpoint Configuration

A Windows 11 virtual machine was configured as the monitored endpoint within the SOC homelab.

| Configuration | Details |
|---|---|
| Hostname | `WindowsEndpoint` |
| Operating System | Windows 11 |
| Host-Only IP Address | `192.168.56.101` |
| Log Collection Agent | Elastic Agent |
| Agent Management | Elastic Fleet |
| SIEM Server | Ubuntu SOC Server (`192.168.56.10`) |

### Elastic Agent Integration

Elastic Agent was installed on WindowsEndpoint and enrolled into Fleet, allowing the Ubuntu SOC Server to centrally manage its monitoring configuration.

The System integration was used to collect native Windows Event Logs, including:

- **Security:** Authentication activity and security-related events.
- **System:** Operating system and service-related events.
- **Application:** Application-generated events and errors.

This provided visibility into endpoint activity without requiring manual log exports.

### Centralised Log Collection

Windows Event Logs were collected by Elastic Agent and forwarded to Elasticsearch for centralised storage and analysis.

The monitoring process followed this workflow:

**Windows Event Logs → Elastic Agent → Elasticsearch → Kibana Discover**

Within Kibana, the `logs-*` data view was used to search and inspect the ingested event records.

### Collection Verification

Successful log collection was confirmed by searching Kibana Discover for Windows Security Event ID `4625`.

The following KQL query was used:

`winlog.event_id : 4625`

The search returned a Windows failed-logon event, confirming that authentication telemetry was being collected and indexed successfully.

The availability of these events established the logging foundation required for subsequent detection engineering and incident investigations.

## Windows Authentication Event Analysis

### Understanding Windows Authentication Events

Windows records authentication activity in the Security Event Log. These records provide information that security analysts can use to investigate suspicious login behaviour.

Two event types were examined during the SOC homelab:

| Event ID | Description | Security Relevance |
|---|---|---|
| 4625 | Failed account logon | Identifies unsuccessful authentication attempts that may indicate incorrect credentials, password guessing or unauthorised access attempts |
| 4624 | Successful account logon | Records successful authentication and can be correlated with earlier failed attempts |

A failed authentication event does not automatically indicate malicious activity. Analysts must consider the account involved, source system, frequency and surrounding events before determining whether activity is suspicious.

### Important Authentication Fields

The following fields were examined within Windows Security event records:

| Field | Purpose |
|---|---|
| `winlog.event_id` | Identifies the Windows event type |
| `winlog.event_data.TargetUserName` | Identifies the account targeted by the authentication attempt |
| `winlog.event_data.LogonType` | Indicates the type of authentication session |
| `winlog.event_data.IpAddress` | Records the source network address when available |
| `winlog.event_data.WorkstationName` | Identifies the originating workstation when available |
| `winlog.event_data.AuthenticationPackageName` | Identifies the authentication package used |
| `@timestamp` | Provides the event timestamp used for investigation and correlation |

These fields help establish **who attempted authentication, where the activity originated, how authentication was performed and when the event occurred**.

### Understanding Logon Types

Windows assigns logon types to distinguish different authentication methods.

Examples include:

- **Logon Type 2 — Interactive:** A user signs in directly to a computer.
- **Logon Type 3 — Network:** A user or remote system authenticates to a network-accessible resource.
- **Logon Type 10 — RemoteInteractive:** A remote interactive session, commonly associated with Remote Desktop.

During the homelab investigation, Logon Type 3 was observed in network authentication records.

### Event Analysis Using Kibana Discover

Kibana Discover was used to search the centralised Windows Security logs.

Failed authentication events were identified using:

`winlog.event_id : 4625`

Successful authentication events could be identified using:

`winlog.event_id : 4624`

Analysing the timestamps, targeted accounts, logon types and available source information made it possible to distinguish individual events and establish the context of authentication activity.

### Security Analysis

Repeated failed authentication attempts against the same account can indicate potential password-guessing activity, although legitimate user errors can produce similar events.

A successful authentication following repeated failures can warrant further investigation, especially when the events involve the same account and source system.

These observations informed the development of a custom failed-logon detection rule and the subsequent controlled incident investigation.

The detection rule configuration and attack simulation findings are documented separately in later sections of the project.

## Log Search, Filtering & Verification

### Searching Windows Security Events

Kibana Discover was used to search and analyse Windows Security events collected through Elastic Agent.

The `logs-*` data view was selected to access the ingested endpoint telemetry.

Kibana Query Language (KQL) was used to filter events by their characteristics.

The following searches were used during the project:

| KQL Query | Purpose |
|---|---|
| `winlog.event_id:4625` | Identify failed Windows logons |
| `winlog.event_id:4624` | Identify successful Windows logons |
| `agent.name:"WindowsEndpoint" AND winlog.event_id:4625` | Identify failed logons specifically from the monitored Windows endpoint |

These filters allowed the analysis to focus on relevant authentication activity rather than unrelated system events.

### Event Verification

On 23 September 2026, a failed Windows logon event was successfully located using Kibana Discover.

The verification demonstrated that:

- The `logs-*` data view contained Windows Security telemetry.
- The query `winlog.event_id:4625` returned a matching event.
- The event could be located by its timestamp.
- The record was available for further field-level investigation.

This confirmed that the monitoring pipeline was operational and that Windows authentication failures could be identified through centralised log searches.

### Investigative Approach

The log analysis workflow involved:

1. Selecting the relevant data view in Kibana Discover.
2. Applying KQL filters to locate authentication events.
3. Reviewing the event timestamps.
4. Inspecting available authentication fields, including the targeted account and logon type.
5. Comparing related events to establish the sequence and context of authentication activity.

### Outcome

Kibana Discover successfully provided searchable access to Windows authentication telemetry.

The ability to filter, inspect and correlate Windows Security events established the analytical foundation for creating threshold-based detection rules and investigating suspicious authentication activity.

Subsequent sections demonstrate how this telemetry was used to generate alerts and investigate controlled authentication attacks.

## Results & Lessons Learned

### Key Achievements

- Successfully configured centralised Windows Event Log collection using a Fleet-managed Elastic Agent.
- Verified that Windows Security events were ingested into Elasticsearch and searchable through Kibana Discover.
- Used KQL filtering to identify relevant authentication events.
- Examined authentication details including Event IDs, usernames, logon types, timestamps and source information.
- Established the logging foundation required for automated detection and incident investigation.

### Lessons Learned

**Understanding security telemetry:** Learned how Windows authentication events provide evidence of user activity and how individual event fields contribute to an investigation.

**Effective log analysis:** Developed practical experience using Kibana Discover and KQL to filter large volumes of system telemetry for security-relevant events.

**Context matters:** Recognised that failed authentication attempts do not automatically indicate malicious behaviour. Event frequency, source information, targeted accounts and surrounding activity must be considered.

**Monitoring before detection:** Confirmed the importance of reliable log collection and field verification before developing automated security detection rules.

### Conclusion

The Windows log monitoring implementation successfully provided visibility into endpoint authentication activity through the Elastic SIEM.

This established the foundation for the next stage of the project: **Detection Engineering**, where Windows authentication telemetry was used to create, test and validate a custom security detection rule.
