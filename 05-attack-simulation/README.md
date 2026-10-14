# Attack Simulation — Network Reconnaissance & Authentication Testing

## Objective

Conduct controlled attack simulations within an isolated cybersecurity homelab to understand how reconnaissance and authentication attacks interact with Windows security controls and generate detectable security events.

The objective was to simulate realistic attacker behaviour against a monitored Windows endpoint and validate that the resulting authentication activity could be observed through the Elastic SIEM.

The simulations focused on:

- **Network reconnaissance:** Using Nmap to identify accessible services and assess Windows Firewall restrictions.
- **Controlled SMB exposure:** Configuring a restricted firewall rule to allow authentication testing over TCP port 445.
- **Remote authentication testing:** Using SMBClient to generate unsuccessful network authentication attempts.
- **Password-guessing simulation:** Using Hydra to test a controlled password list against a temporary Windows account.
- **Security telemetry generation:** Producing Windows Event IDs 4625 and 4624 for subsequent analysis.

## Attack Simulation Scope

All testing was conducted within a private VirtualBox homelab using three virtual machines.

| System | Role | IP Address |
|---|---|---|
| Kali-Attack | Simulated attacker | `192.168.56.102` |
| WindowsEndpoint | Target and monitored endpoint | `192.168.56.101` |
| Ubuntu SOC Server | Security monitoring infrastructure | `192.168.56.10` |

The attack simulations were directed exclusively at the designated WindowsEndpoint.

Temporary test credentials and a source-restricted SMB firewall exception were used to control the testing environment.

Following the simulations, the temporary account and firewall exception were removed.

The attack activities and resulting evidence are documented in this section, while the detailed alert investigation and analyst response are covered separately in Section 6 — Incident Investigation.

## Network Reconnaissance Using Nmap

### Objective

Perform network reconnaissance from Kali Linux to identify accessible TCP services on WindowsEndpoint and evaluate the effectiveness of Windows Firewall restrictions.

### Initial Network Scan

The following Nmap command was executed from Kali Linux:

```bash
sudo nmap -sS -sV -T3 192.168.56.101
```

**Command breakdown:**

- `sudo` — executes Nmap with elevated privileges.
- `nmap` — launches the network scanning tool.
- `-sS` — performs a TCP SYN scan.
- `-sV` — attempts to identify service versions on accessible ports.
- `-T3` — uses the normal timing template.
- `192.168.56.101` — specifies WindowsEndpoint as the target.

### Initial Findings

The scan confirmed that WindowsEndpoint was reachable across the Host-Only network.

However, all 1,000 TCP ports scanned were reported as **filtered**.

This indicated that Windows Firewall was restricting inbound network connections, preventing Nmap from establishing whether the scanned services were open or closed.

The results demonstrated the endpoint's initial network protection before any firewall configuration changes were made.

### Controlled SMB Firewall Configuration

To support the authentication testing, a temporary Windows Defender Firewall rule was created to permit inbound SMB traffic exclusively from the Kali attacker VM.

The following PowerShell command was executed on WindowsEndpoint:

```powershell
New-NetFirewallRule -DisplayName "SOC Lab - Allow SMB from Kali" -Direction Inbound -Protocol TCP -LocalPort 445 -RemoteAddress 192.168.56.102 -Action Allow
```

The rule specified:

| Parameter | Configuration |
|---|---|
| Direction | Inbound |
| Protocol | TCP |
| Destination Port | 445 (SMB) |
| Permitted Source IP | `192.168.56.102` |
| Target System | WindowsEndpoint |
| Action | Allow |

This enabled the required testing without disabling Windows Firewall or granting unrestricted inbound access.

### Follow-Up Network Scan

After configuring the firewall exception, a targeted scan was performed:

```bash
sudo nmap -sS -sV -p 445 192.168.56.101
```

The scan confirmed that TCP port 445 was accessible:

```text
PORT    STATE SERVICE
445/tcp open  microsoft-ds
```

The result verified that the firewall exception permitted SMB traffic from the designated Kali attacker system.

### Security Findings

The reconnaissance demonstrated that:

- WindowsEndpoint was reachable on the private lab network.
- Windows Firewall initially filtered inbound TCP probes.
- A specific firewall exception could expose SMB without disabling the entire firewall.
- Network access controls directly influenced the available attack surface.
- TCP port 445 could subsequently be used for controlled remote authentication testing.

### Outcome

The reconnaissance successfully identified the endpoint's initial firewall restrictions and validated a controlled configuration change that enabled SMB authentication testing.

This established the network conditions required for the subsequent SMBClient and Hydra simulations.

The temporary firewall rule was removed after testing to restore the endpoint's original access restrictions.

## SMB Authentication Testing

### Objective

Simulate unsuccessful remote authentication attempts from Kali Linux against WindowsEndpoint using SMB (Server Message Block) over TCP port 445.

The objective was to generate Windows Security Event ID 4625 records originating from a separate system and verify that the authentication failures were captured by Elastic SIEM.

### Initial SMB Authentication Test

Following confirmation that TCP port 445 was accessible, SMBClient was used from Kali Linux to attempt authentication against WindowsEndpoint.

The following command was executed:

```bash
smbclient -L //192.168.56.101 -U 'FakeSOCUser%WrongPassword123!'
```

**Command breakdown:**

- `smbclient` — launches the SMB client utility.
- `-L` — requests a list of available SMB shares on the target.
- `//192.168.56.101` — identifies WindowsEndpoint as the target.
- `-U` — specifies the username and password for authentication.
- `FakeSOCUser` — the designated test username.
- `WrongPassword123!` — the deliberately incorrect test password.

The authentication attempt returned:

`NT_STATUS_LOGON_FAILURE`

This confirmed that the supplied credentials were rejected by Windows.

### Repeated Authentication Attempts

To generate multiple authentication failures, the SMBClient command was repeated three times using a Bash loop:

```bash
for i in {1..3}; do
    smbclient -L //192.168.56.101 -U 'FakeSOCUser%WrongPassword123!'
    sleep 2
done
```

Each unsuccessful attempt produced a Windows Security Event ID 4625.

The controlled sequence provided repeatable authentication telemetry for testing the existing Elastic detection rule.

### Windows Security Event Verification

The generated authentication failures were examined through Windows Security Event Logs.

The following information was identified:

| Event Attribute | Observed Value |
|---|---|
| Event ID | 4625 — Failed Logon |
| Target Username | `FakeSOCUser` |
| Logon Type | 3 — Network |
| Source Workstation | `KALI` |
| Source IP Address | `192.168.56.102` |
| Authentication Protocol | NTLM |
| Destination | WindowsEndpoint (`192.168.56.101`) |
| Destination Port | TCP 445 |

**Logon Type 3** confirmed that the activity involved network authentication rather than an interactive Windows sign-in.

The source IP address and workstation name provided evidence linking the authentication failures to the Kali attacker VM.

### Elastic SIEM Verification

The Windows Security events were collected through Elastic Agent and forwarded to Elasticsearch.

Kibana Discover was used to confirm that three Event ID 4625 records generated during the test were available in the `logs-*` data view.

The authentication activity also satisfied the conditions of the custom **Repeated Windows Failed Logons** detection rule.

Elastic Security successfully generated a threshold-based alert for the targeted username.

Detailed alert investigation and response activities are documented separately in Section 6 — Incident Investigation.

### Outcome

The SMB authentication simulation successfully demonstrated how unsuccessful remote authentication attempts generate security telemetry on WindowsEndpoint.

The test verified:

- Network authentication failures originating from Kali Linux.
- Windows Event ID 4625 generation.
- Identification of the source IP address and workstation.
- Recognition of network authentication through Logon Type 3.
- Successful ingestion of the events into Elastic SIEM.
- Automated detection of repeated authentication failures.

No successful authentication or unauthorised access occurred during this SMBClient test.

The simulation established the foundation for the subsequent Hydra password-guessing scenario.

## Hydra Password-Guessing Simulation

### Objective

Simulate a controlled password-guessing attack against WindowsEndpoint using Hydra from the Kali Linux attacker VM.

The objective was to demonstrate how repeated authentication attempts generate Windows Security telemetry and how a successful authentication can be identified following multiple failures.

### Test Account Preparation

A temporary Windows local account named `HydraLabUser` was created specifically for the simulation.

The account was configured as a standard local user without administrative privileges.

A controlled password list containing four candidate passwords was prepared on Kali Linux:

- Three deliberately incorrect passwords.
- One valid password assigned to the temporary test account.

This provided a predictable testing sequence without targeting legitimate user accounts.

### Hydra Attack Execution

Hydra was used to perform SMB authentication attempts against WindowsEndpoint over TCP port 445.

The following command was executed from Kali Linux:

```bash
hydra -l HydraLabUser -P hydra-lab-passwords.txt -t 1 -f -V -m 'workgroup:{WORKGROUP}' smb2://192.168.56.101
```

**Command breakdown:**

| Parameter | Purpose |
|---|---|
| `hydra` | Launches the password-testing utility |
| `-l HydraLabUser` | Specifies the targeted test account |
| `-P hydra-lab-passwords.txt` | Supplies the controlled password list |
| `-t 1` | Limits testing to one concurrent task |
| `-f` | Stops testing after identifying a valid credential |
| `-V` | Displays individual authentication attempts |
| `-m 'workgroup:{WORKGROUP}'` | Supplies the SMB module's workgroup configuration |
| `smb2://192.168.56.101` | Targets WindowsEndpoint using SMB2 |

The test was restricted to the designated Windows virtual machine within the isolated network.

### Observed Attack Results

Hydra attempted authentication using the controlled password list.

The resulting sequence was:

| Attempt | Authentication Result | Windows Event ID |
|---|---|---|
| 1 | Failed | 4625 |
| 2 | Failed | 4625 |
| 3 | Failed | 4625 |
| 4 | Successful | 4624 |

Hydra reported that it had identified a valid credential for `HydraLabUser`.

The successful authentication was independently corroborated through Windows Security Event Logs rather than relying solely on Hydra's output.

### Windows Security Event Verification

The authentication events were examined on WindowsEndpoint.

The following information was confirmed:

| Attribute | Observed Value |
|---|---|
| Target Username | `HydraLabUser` |
| Source IP Address | `192.168.56.102` |
| Source Workstation | `KALI` |
| Destination IP Address | `192.168.56.101` |
| Network Service | SMB / TCP 445 |
| Logon Type | 3 — Network |
| Authentication Protocol | NTLM |
| Failed Authentication Events | 3 × Event ID 4625 |
| Successful Authentication Events | 1 × Event ID 4624 |

The successful Event ID 4624 confirmed that Windows accepted the credentials for a network logon.

This demonstrated successful authentication to the designated lab account, not privilege escalation or unrestricted access to the endpoint.

### Elastic SIEM Verification

The generated authentication records were collected by Elastic Agent and indexed in Elasticsearch.

Kibana Discover confirmed four relevant authentication events for `HydraLabUser`:

**4625 → 4625 → 4625 → 4624**

The existing **Repeated Windows Failed Logons** detection rule identified the three failed authentication events and generated a security alert.

The rule did not automatically detect the subsequent successful authentication as a separate failure-to-success correlation. That relationship was established through analysis of the Windows Security events.

The detailed investigation and alert response are documented in Section 6 — Incident Investigation.

### Testing Cleanup

Following the simulation:

- The temporary `HydraLabUser` account was deleted.
- The source-restricted SMB firewall exception was removed.
- The temporary password list was deleted from Kali Linux.
- Historical Windows Security events and Elastic investigation evidence were retained.

### Outcome

The Hydra simulation successfully demonstrated a controlled password-guessing sequence that progressed from three failed authentication attempts to one successful network authentication.

The exercise provided practical experience in:

- Conducting controlled authentication security testing.
- Understanding SMB and NTLM authentication behaviour.
- Identifying failed and successful Windows logon events.
- Validating SIEM telemetry generated by attacker activity.
- Recognising the difference between detecting repeated failures and correlating a subsequent successful login.
- Restoring the lab's security configuration following testing.

The simulation was performed exclusively against a temporary account within the authorised homelab environment.

## Attack Sequence & Results

### Attack Simulation Timeline

The testing was conducted in a controlled sequence, progressing from network reconnaissance to remote authentication testing.

| Stage | Activity | Observed Result |
|---|---|---|
| 1 | Nmap reconnaissance against WindowsEndpoint | All 1,000 scanned TCP ports reported as filtered |
| 2 | Restricted Windows Firewall exception configured | SMB access permitted from Kali on TCP 445 |
| 3 | Targeted Nmap scan | TCP 445 confirmed open |
| 4 | SMBClient authentication testing | Three failed authentication attempts generated |
| 5 | Windows Security Log verification | Event ID 4625 identified with Kali source information |
| 6 | Hydra password-guessing simulation | Three failed attempts followed by one successful authentication |
| 7 | Windows authentication event analysis | Three Event ID 4625 records followed by Event ID 4624 |
| 8 | Elastic SIEM verification | Authentication events successfully collected and searched |
| 9 | Security detection | Custom failed-logon rule generated alerts |
| 10 | Environment cleanup | Temporary account, password list and firewall exception removed |

### Key Findings

**1. Windows Firewall restricted the initial attack surface**

Initial reconnaissance demonstrated that inbound TCP traffic was filtered. Controlled SMB testing became possible only after the designated firewall exception was configured.

**2. Remote authentication attempts generated identifiable security telemetry**

Windows recorded failed network authentication events containing the targeted username, Logon Type 3, the Kali workstation name and its source IP address.

**3. Password guessing resulted in successful authentication**

The Hydra simulation demonstrated that a valid credential could be identified after multiple failed attempts against the temporary lab account.

This was confirmed through Windows Event ID 4624.

**4. Security monitoring captured the attack activity**

Elastic Agent successfully forwarded the Windows authentication events to Elasticsearch, allowing them to be analysed through Kibana Discover.

**5. Automated detection and event correlation serve different purposes**

The custom detection rule identified repeated authentication failures.

However, recognising that those failures were followed by successful authentication required additional event correlation during the investigation.

## Lessons Learned

The simulations provided practical experience in several cybersecurity areas:

- **Network security:** Understanding how firewall configurations control service exposure and influence the available attack surface.
- **Authentication security:** Understanding SMB network authentication and the risks associated with password guessing.
- **Windows event analysis:** Interpreting Event IDs 4625 and 4624 to establish authentication outcomes.
- **Security monitoring:** Verifying that simulated attacker activity produces telemetry that can be collected and analysed.
- **Detection validation:** Testing custom security detections against controlled attack activity.
- **Secure testing practices:** Using dedicated lab accounts, restricted network permissions and post-testing cleanup.

## Conclusion

The attack simulations successfully demonstrated the relationship between network reconnaissance, remote authentication attempts and the generation of Windows Security events.

The controlled testing produced measurable evidence of both failed and successful authentication activity, which was collected through the Elastic SIEM.

The resulting events and security alerts provided the foundation for **Section 6 — Incident Investigation**, where the activity was correlated, investigated, documented and closed.

This exercise demonstrated how offensive security testing can support defensive monitoring, detection validation and SOC analyst development.
