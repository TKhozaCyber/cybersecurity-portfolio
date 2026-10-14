# Elastic SIEM Deployment & Configuration

## Objective

Deploy and configure a centralised Security Information and Event Management (SIEM) platform using the Elastic Stack on Ubuntu Server.

The objective was to establish a functioning security monitoring infrastructure capable of collecting Windows endpoint telemetry, centrally managing Elastic Agents and supporting security event analysis through Kibana.

The deployment involved configuring Elasticsearch, Kibana and Fleet Server, establishing secure connectivity between the Ubuntu SOC Server and WindowsEndpoint, and troubleshooting service, certificate and resource-management issues.

This infrastructure provides the foundation for the log monitoring, detection engineering and incident investigation activities documented in subsequent sections.

## Deployment Environment

The Elastic SIEM infrastructure was deployed on a dedicated Ubuntu Server virtual machine within the VirtualBox homelab.

| Configuration | Details |
|---|---|
| Operating System | Ubuntu Server 24.04 LTS |
| Virtual Machine | Ubuntu-server-clean |
| Host-Only IP Address | `192.168.56.10` |
| Allocated RAM | 6 GB |
| Virtual CPUs | 4 |
| Virtual Disk | 40 GB |
| Elasticsearch Version | 8.19.22 |
| Kibana Version | 8.19.22 |

The Ubuntu server provides centralised storage, management and analysis of security telemetry collected from the monitored Windows endpoint.

## Elastic Stack Components

### Elasticsearch

Elasticsearch acts as the central data storage and search engine for the SOC environment.

It indexes and stores security telemetry collected by Elastic Agent, allowing authentication events and other system activity to be searched and analysed.

### Kibana

Kibana provides the web-based interface for interacting with Elasticsearch.

It was configured to support:

- Log searching and analysis through Discover.
- Security event investigation.
- Custom detection rules.
- Security alert monitoring.
- Fleet and Elastic Agent management.

### Fleet Server

Fleet Server was deployed on the same Ubuntu virtual machine as Elasticsearch and Kibana.

It provides centralised management of enrolled Elastic Agents, including policy distribution, integration configuration and agent status monitoring.

### Elastic Agent

Elastic Agent was installed on WindowsEndpoint (`192.168.56.101`) and enrolled into Fleet.

The agent collects Windows event logs and sends the telemetry to Elasticsearch for analysis.

### Log Collection Architecture

The deployment uses the following data flow:

**Windows Event Logs → Elastic Agent → Elasticsearch → Kibana**

Fleet Server separately manages the Elastic Agent through its assigned policy.

This architecture enables centralised collection, storage and analysis of Windows security telemetry within the SOC environment.

## Installation & Initial Configuration

### Ubuntu SOC Server Preparation

Ubuntu Server 24.04 LTS was installed in Oracle VirtualBox to provide a dedicated Linux environment for hosting the Elastic SIEM infrastructure.

The virtual machine was configured with NAT and Host-Only network adapters, allowing internet access for software installation and communication with other systems on the private lab network.

A static lab IP address of `192.168.56.10` was used for communication with the monitored Windows endpoint.

### Elasticsearch Deployment

Elasticsearch was installed on the Ubuntu SOC Server to provide centralised indexing, storage and searching of security telemetry.

The service was configured and tested to ensure it could run successfully and support communication with Kibana and the Elastic Agents.

### Kibana Deployment

Kibana was installed on the same Ubuntu server and connected to Elasticsearch.

The web interface was made accessible from the host computer through:

`http://192.168.56.10:5601`

This provided access to Discover, Fleet, Elastic Security, detection rules and alert investigations.

### Fleet Server Deployment

Fleet Server was deployed using Elastic Agent on the Ubuntu SOC Server.

Fleet provided the central management interface needed to enrol, configure and monitor Elastic Agents across the lab environment.

The Windows endpoint was subsequently enrolled into Fleet using an assigned agent policy.

### Service Verification

Following installation and configuration, the operational status of the core services was verified using Linux system administration commands:

- `sudo systemctl status elasticsearch`
- `sudo systemctl status kibana`
- `sudo systemctl status elastic-agent`

These checks were used to confirm that Elasticsearch, Kibana and Elastic Agent were running successfully.

### Initial Deployment Outcome

The core Elastic infrastructure was successfully deployed on Ubuntu Server, with Kibana accessible through the host computer and Fleet available for centralised endpoint management.

Additional troubleshooting and configuration work was subsequently performed to address TLS connectivity, Kibana alerting configuration and resource optimisation.

## TLS Certificates & Fleet Connectivity Troubleshooting

### Issue Identified

During the configuration of the Windows Elastic Agent, a TLS certificate validation issue prevented successful communication with Elasticsearch.

The agent attempted to connect to Elasticsearch using the Ubuntu SOC Server's Host-Only IP address:

`https://192.168.56.10:9200`

However, the Elasticsearch TLS certificate did not include this IP address in its Subject Alternative Name (SAN) entries.

This resulted in an **x509 certificate validation error**, preventing the Windows agent from securely publishing telemetry.

### Investigation

The troubleshooting process involved:

- Reviewing Elastic Agent status and connectivity errors.
- Investigating the Elasticsearch HTTPS configuration.
- Identifying the mismatch between the connection IP address and the certificate's valid identities.
- Confirming that the certificate supported the hostname `ubuntu-soc-lab`.

The investigation established that the issue was related to TLS hostname verification rather than general network connectivity.

### Resolution

Instead of regenerating the Elasticsearch certificates, the configuration was adjusted to use the hostname already supported by the certificate.

The following entry was added to the WindowsEndpoint hosts file:

`192.168.56.10 ubuntu-soc-lab`

The Fleet-managed Elasticsearch output was then configured to use:

`https://ubuntu-soc-lab:9200`

This allowed WindowsEndpoint to resolve the hostname to the Ubuntu server's Host-Only IP address while using an identity recognised by the TLS certificate.

### Verification

Following the configuration changes:

- The Windows endpoint successfully resolved `ubuntu-soc-lab`.
- Elastic Agent established a working connection.
- Fleet reported the Windows agent as **Healthy** and **Connected**.
- Windows Security events were successfully ingested into Elasticsearch and became searchable in Kibana Discover.

### Outcome & Lessons Learned

The issue was resolved without disabling TLS certificate validation or replacing the existing certificates.

This troubleshooting exercise demonstrated the importance of:

- Understanding TLS certificate identity verification.
- Distinguishing network connectivity issues from certificate validation failures.
- Configuring hostname resolution in isolated lab environments.
- Verifying successful agent communication and data ingestion after configuration changes.

The hosts-file solution was appropriate for the controlled homelab. A production environment would typically use centrally managed DNS and certificates containing the required hostnames or IP addresses.

## Kibana Encryption Configuration & Detection Engine Troubleshooting

### Issue Identified

Following the initial Elastic Stack deployment, Kibana's security detection and alerting functionality could not be fully initialised.

Investigation identified a configuration problem involving the `xpack.encryptedSavedObjects.encryptionKey` setting, which Kibana requires for encrypted saved objects used by alerting and other security features.

Attempts to configure the encryption key directly within `kibana.yml` resulted in configuration validation errors and prevented Kibana from starting successfully.

### Investigation

The troubleshooting process included:

- Reviewing Kibana service logs and configuration validation errors.
- Investigating the encrypted saved objects plugin and its configuration requirements.
- Verifying the installed Elastic Stack versions.
- Upgrading Elasticsearch and Kibana from version 8.19.13 to 8.19.22.
- Testing alternative methods of providing the required encryption key.

The investigation established that the required setting could be supplied through the Kibana keystore.

### Resolution

The encryption key was configured securely using the Kibana keystore rather than placing its value directly in the YAML configuration file.

The following command was executed on Ubuntu:

`sudo KBN_PATH_CONF=/etc/kibana /usr/share/kibana/bin/kibana-keystore add xpack.encryptedSavedObjects.encryptionKey`

The encryption key value was entered through the command's interactive prompt.

The keystore configuration was verified using:

`sudo KBN_PATH_CONF=/etc/kibana /usr/share/kibana/bin/kibana-keystore list`

The original encryption key entry in `kibana.yml` was left commented out to avoid conflicting configuration.

Kibana was subsequently restarted.

### Verification

Following the changes:

- The Kibana service started successfully.
- The encrypted saved objects functionality initialised.
- Elastic Security detection rules became available.
- A custom Windows failed-logon detection rule was successfully created and enabled.
- Controlled authentication testing generated alerts that were investigated through Elastic Security.

### Outcome & Lessons Learned

The configuration issue was successfully resolved, restoring the alerting capabilities required for the SOC lab.

This exercise demonstrated practical experience in:

- Troubleshooting Linux service startup failures.
- Interpreting application configuration validation errors.
- Managing sensitive application settings through a keystore.
- Performing compatible Elastic Stack version upgrades.
- Verifying application functionality following configuration changes.

The successful resolution enabled the detection engineering and incident investigation activities documented later in the project.

## Elasticsearch Memory Optimisation & Storage Management

### Issue Identified

During the deployment and configuration of the Elastic Stack, the Ubuntu SOC Server experienced resource constraints.

The environment initially allocated 8 GB of RAM to Ubuntu, creating pressure on the physical host when running the Windows and Kali virtual machines simultaneously.

Additionally, although the Ubuntu VM had a 40 GB virtual disk, its root logical volume initially provided only approximately 19 GB of usable filesystem space.

These limitations required memory and storage optimisation to support the three-VM SOC environment.

### Elasticsearch JVM Heap Optimisation

Elasticsearch uses the Java Virtual Machine (JVM), which requires dedicated heap memory for processing and managing data.

To reduce memory consumption, the following configuration file was created:

`/etc/elasticsearch/jvm.options.d/heap.options`

The JVM heap was configured as:

```text
-Xms2g
-Xmx2g
```

This assigned Elasticsearch a 2 GB initial and maximum JVM heap.

Following the adjustment, Elasticsearch's observed process memory consumption decreased from approximately 4.4 GB to around 2.6–2.9 GB.

The Ubuntu VM's allocated memory was subsequently reduced from 8 GB to 6 GB, allowing the three virtual machines to operate within the physical host's 16 GB RAM capacity.

### Ubuntu Storage Expansion

The Ubuntu VM's virtual disk was configured with 40 GB of storage, but the root logical volume initially used approximately 19 GB.

Logical Volume Manager (LVM) was used to extend the root filesystem into the remaining available space.

The following command was executed:

```bash
sudo lvextend -l +100%FREE -r /dev/ubuntu-vg/ubuntu-lv
```

This expanded the root logical volume and its filesystem to approximately 38 GB.

Following the expansion, approximately 21 GB of filesystem space was available.

### Log Storage Maintenance

System journal logs were also reviewed to identify recoverable disk space.

Older journal data was reduced to a 200 MB retention-size target, recovering approximately 577 MB of disk space.

This helped manage storage consumption while preserving the Elastic security telemetry stored in Elasticsearch.

### Verification

The optimisation work was verified through:

- Linux memory utilisation checks using `free -h`.
- Filesystem utilisation checks using `df -h`.
- Logical volume inspection using LVM utilities.
- Elasticsearch and Kibana service-status checks.
- Continued access to Kibana and successful security event ingestion.

### Outcome & Lessons Learned

The optimisation improved resource allocation across the homelab without requiring additional physical hardware.

Key technical lessons included:

- Configuring JVM heap settings to manage Elasticsearch memory consumption.
- Understanding the difference between JVM heap allocation and total process memory usage.
- Managing Linux filesystems using LVM.
- Monitoring disk utilisation and maintaining system logs.
- Balancing virtual machine resources on a memory-constrained host.

These changes helped maintain a functioning Elastic SIEM environment alongside the Windows endpoint and Kali attacker systems.

## Service Health & Deployment Verification

### Ubuntu Service Verification

Following deployment, troubleshooting and resource optimisation, the operational status of the Elastic services was checked using Linux system administration commands.

The following commands were used:

- `sudo systemctl status elasticsearch`
- `sudo systemctl status kibana`
- `sudo systemctl status elastic-agent`

These checks confirmed that Elasticsearch, Kibana and the Elastic Agent hosting Fleet Server were running successfully on the Ubuntu SOC Server.

### Windows Elastic Agent Verification

The WindowsEndpoint was configured with a Fleet-managed Elastic Agent to collect system and security telemetry.

The agent's Windows service status was checked using PowerShell:

`Get-Service "Elastic Agent"`

The agent's component status was also inspected using:

`& "C:\Program Files\Elastic\Agent\elastic-agent.exe" status`

The checks confirmed that the Windows Elastic Agent was operational, with Fleet reporting a **Healthy** and **Connected** status.

### Log Ingestion Verification

Successful telemetry collection was validated through Kibana Discover using the `logs-*` data view.

Windows authentication events were located using Event IDs:

- **4625:** Failed logon attempts.
- **4624:** Successful logon attempts.

The collected records contained information relevant to security investigations, including:

- Targeted user accounts.
- Authentication success or failure.
- Logon type.
- Source workstation and IP address.
- Authentication protocol.

During controlled authentication testing, Windows Security events were successfully collected and became searchable in Elasticsearch.

### Detection Engine Verification

Elastic Security was also used to confirm that the deployed infrastructure could support automated threat detection.

A custom threshold-based detection rule successfully generated security alerts when multiple failed Windows authentication attempts were recorded.

The resulting alerts were available for investigation, documentation and closure through Kibana.

Detailed detection logic and investigation evidence are documented in the later project sections.

### Deployment Outcome

The final verification confirmed that the Elastic infrastructure could:

- Maintain operational Elasticsearch, Kibana and Fleet services.
- Centrally manage the Windows Elastic Agent.
- Collect and index Windows Security telemetry.
- Support security event searches through Kibana Discover.
- Generate security alerts through custom detection rules.

This established a functioning SIEM platform capable of supporting the controlled attack simulations and incident investigations performed in the SOC homelab.
