# Cybersecurity-Lab-SIEM - Detection, Response & Automation

This lab simulates a real-world SOC environment from end-to-end: endpoint telemetry ingestion, custom detection engineering, automated case creation, IOC analysis and active response.

---

## Architecture

<p align="center">
  <img src="docs/architecture/architecture-overview.png">
</p>

<p align="center">
  <img src="docs/architecture/network-topology.png">
</p>

---

## Technologies

| Component | Role | Version |
|-----------|------|---------|
| **Wazuh** | SIEM / XDR - log ingestion, rule-based detection, alerting | 4.14 |
| **TheHive** | Case management - incident tracking and analyst collaboration | 5.6 |
| **Cortex** | Observables analysis and enrichment | 4.0 |
| **Shuffle** | SOAR - workflow automation and tool orchestration | 2.2 |
| **MISP** | Threat intelligence platform - IOC sharing and feed ingestion | Latest |
| **Sysmon** | Deep Windows endpoint telemetry (SwiftOnSecurity config) | 15.2 |
| **Atomic Red Team** | Adversary emulation mapped to MITRE ATT&CK | Latest |
| **VirtualBox** | Hypervisor - hosted isolated NAT network | 7.1 |

---

## Project Phases

### Phase 1 - Infrastructure
Deployed all services (TheHive, Cortex, Shuffle, MISP) utilising Docker containers in an Ubuntu Server VM (`soc-core`). Configured an isolated VirtualBox NAT network (`soc-net`, `10.0.5.0/24`) with port-forwarding rules to access the services on the host.

### Phase 2 - Endpoint Configuration
Deployed a **Windows 11** VM and an **Ubuntu Server** VM as Wazuh agents. Installed Sysmon with the SwiftOnSecurity config on the Windows machine and configured Wazuh to ingest `Microsoft-Windows-Sysmon/Operational` event channel logs.

### Phase 3 - Integrations
Connected all tools:
- **TheHive ↔ Cortex** - observables analysis within cases
- **TheHive ↔ MISP** - bidirectional IOC sharing 
- **Wazuh → Shuffle** - webhook-based alert forwarding for SOAR triggers
- **Cortex analyzers enabled:** VirusTotal, URLhaus, Abuse Finder, FileInfo

### Phase 4 - Detection Engineering
Wrote custom Wazuh rules in `local_rules.xml` targeting:
- Suspicious PowerShell download cradles (T1105)
- Scheduled task creation via command line (T1053.005)
- Registry Run key modification (T1547.001)

### Phase 5 - SOAR Automation
Built an end-to-end Shuffle workflow triggered by Wazuh webhooks:

```
Endpoints (Win / Ubuntu)
      ↓ sends events
Wazuh SIEM
      ↓ detects & alerts
Shuffle SOAR
      ↓ receives webhook — parses JSON (rule, agent, command data)
Regex node extracts SHA256 hash from Sysmon event
      ↓
Cortex queries VirusTotal for the hash
      ↓
TheHive alert created with host info and VT score
      ↓
Condition: VT malicious count > 3?
      ↓ YES
Wazuh active response isolates the host
      ↓
SOC analyst reviews the TheHive alert             
```

### Phase 6 - Adversary Simulation & Validation
Used **Atomic Red Team** to simulate real attacks mapped to MITRE ATT&CK:

```powershell
Invoke-AtomicTest T1105 -TestNumbers 10       # Suspicious PowerShell download
Invoke-AtomicTest T1547.001 -TestNumbers 1    # Registry Run key modification
Invoke-AtomicTest T1053.005 -TestNumbers 1    # Scheduled task creation
```

Validated the full pipeline: Wazuh detects → Shuffle triggers → Cortex enriches → TheHive alert fires → host isolated.


## Repository Structure

```
Cybersecurity-Lab-SIEM/
├── configs/
│   ├── local_rules.xml                  # Custom Wazuh detection rules
│   └── sigma_conversion.yaml            # Sigma conversion of custom rules
├── docs/
│   ├── architecture/
│   │   ├── architecture-overview.png
│   │   └── network-topology.png
│   ├── setup.md                         # Full deployment guide
│   ├── rules.md                         # Detection rule logic and explanations
│   ├── attack_simulations.md            # Atomic Red Team execution and results
│   ├── troubleshooting.md               # Known issues and fixes
│   └── screenshots/
│       ├── virtualbox-nat-network-a.png
│       ├── virtualbox-nat-network-b.png
│       ├── wazuh-agents-connected.png
│       ├── sysmon-telemetry-event.png
│       ├── wazuh-detection-alert.png
│       ├── shuffle-soar-workflow.png
│       ├── thehive-auto-case.png
│       └── wazuh-host-isolation.png
└── README.md
```

---

## Screenshots

| Screenshot | Description |
|------------|-------------|
| `virtualbox-nat-network.png` | NAT network configuration |
| `wazuh-agents-connected.png` | Agents connected to Wazuh |
| `sysmon-telemetry-event.png` | Sysmon event captured in Event Viewer |
| `wazuh-detection-alert.png` | Custom detection rule triggered |
| `shuffle-soar-workflow.png` | Shuffle automation workflow |
| `thehive-auto-case.png` | Case automatically created by Shuffle |
| `wazuh-host-isolation.png` | Endpoint isolated via active response |

> All screenshots are in [`docs/screenshots/`](docs/screenshots/)

---

## Skills Demonstrated

- **SIEM Engineering** - custom detection rule creation and tuning, log source integration, agent management
- **Incident Response** - end-to-end case lifecycle from detection to closure
- **SOAR Development** - conditional workflow logic, API chaining, automated host isolation
- **Threat Intelligence** - feed ingestion, IOC enrichment
- **Adversary Emulation** - MITRE ATT&CK simulation, detection and validation
- **Linux Administration** - Docker, systemd, networking

---

## Setup

Full deployment notes are in [`docs/setup.md`](docs/setup.md).

Requirements:
- Host machine with 16 GB+ RAM recommended
- VirtualBox 7.1
- Internet access

---

## References

- Wazuh Documentation: https://documentation.wazuh.com
- TheHive Project: https://thehive-project.org
- Shuffle SOAR: https://shuffler.io
- MISP Project: https://www.misp-project.org
- SwiftOnSecurity Sysmon Config: https://github.com/SwiftOnSecurity/sysmon-config
- Atomic Red Team: https://github.com/redcanaryco/atomic-red-team
- MITRE ATT&CK: https://attack.mitre.org
