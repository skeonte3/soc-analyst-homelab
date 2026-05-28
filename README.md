# SOC Analyst Home Lab

A fully functional Security Operations Center lab built on Proxmox, simulating a real-world detection and response environment. Used to practice SIEM workflows, attack detection, and threat hunting on Windows endpoints.

## Overview

This lab provides hands-on experience with the core technologies and workflows of a SOC analyst:
- Endpoint telemetry collection (Sysmon)
- SIEM log aggregation and search (Splunk Enterprise)
- Adversary simulation (Atomic Red Team)
- Network segmentation and isolation
- Detection engineering

## Architecture

[Internet] → [Spectrum Router] → [Proxmox Host (Dell OptiPlex 7020)]
│
├── vmbr0 (LAN 192.168.1.0/24)
│     ├── Pi-hole LXC (CT 100)
│     └── Splunk Enterprise VM (101)
│
└── vmbr1 (Isolated 10.10.10.0/24)
├── Kali Linux VM (102)
└── Windows 10 Victim VM (103)

The Windows VM has a second NIC on vmbr0 to forward logs to Splunk while remaining isolated from the home network for attack simulations on vmbr1.

## Tech Stack

| Component | Purpose |
|-----------|---------|
| **Proxmox VE 8.4** | Hypervisor |
| **Splunk Enterprise** | SIEM and log analysis |
| **Splunk Universal Forwarder** | Log shipping from endpoints |
| **Sysmon** (SwiftOnSecurity config) | Windows endpoint telemetry |
| **Atomic Red Team** | MITRE ATT&CK-aligned adversary simulation |
| **Kali Linux 2026.1** | Attacker platform |
| **Windows 10 Pro 22H2** | Target endpoint |
| **Ubuntu Server 22.04** | Splunk host OS |

## Lab Specifications

- **Host hardware:** Dell OptiPlex 7020, 32GB RAM, 1.8TB SSD
- **Network:** Dual bridges (LAN + isolated lab network)
- **VM resources:** 
  - Splunk: 4 cores, 8GB RAM, 50GB disk
  - Kali: 2 cores, 4GB RAM, 40GB disk
  - Windows 10: 4 cores, 8GB RAM, 80GB disk

## Setup Highlights

1. **Network isolation** — Created an additional Linux bridge (vmbr1) with no gateway to fully isolate attack traffic from the home network and internet.
2. **Dual-NIC Windows VM** — One NIC on the isolated network for receiving attacks, one on the LAN for forwarding telemetry to Splunk.
3. **Sysmon with SwiftOnSecurity config** — Industry-standard config that captures meaningful events without overwhelming the SIEM with noise.
4. **Splunk receiver on port 9997** — Standard forwarder ingestion endpoint configured to receive logs from Windows endpoints.
5. **Atomic Red Team** for repeatable, MITRE-mapped attack simulations.

## Sample Detections

Real Splunk searches built and tested in this lab:

**Process creation by PowerShell:**
```spl
index=main EventCode=1 Image="*powershell*"
```

**Encoded PowerShell commands (suspicious):**
```spl
index=main EventCode=1 Image="*powershell*" CommandLine="*-EncodedCommand*"
```

**All Windows endpoint activity (last hour):**
```spl
index=main host=win10
```

## Techniques Tested

| MITRE ATT&CK ID | Technique | Status |
|-----------------|-----------|--------|
| T1059.001 | PowerShell | ✅ Detected |
| (more to come) | | |

## Lessons Learned

- **Physical layer matters first.** A bad ethernet cable forced the NIC to negotiate at 10 Mbps with constant link flapping, which presented as application-layer symptoms (ERR_CONTENT_LENGTH_MISMATCH on the Proxmox web UI). Always check `dmesg` for link speed and stability before assuming software issues.
- **Config file encoding can silently fail.** Splunk's Universal Forwarder ignored an `inputs.conf` written with Notepad's default encoding. Fixed by rewriting via PowerShell with explicit ASCII encoding.
- **Local System doesn't get Sysmon log access by default.** The forwarder service needed to run under an admin account (or have the Sysmon channel SDDL updated) to subscribe to the Sysmon event log.
- **Tamper Protection blocks Defender PowerShell commands.** Atomic Red Team requires disabling both Tamper Protection (in the Windows Security UI) and real-time protection.
- **SwiftOnSecurity's Sysmon config is the right starting point.** It produces high-signal events without flooding the SIEM with normal Windows noise.

## Repository Contents

## Future Work

- Build dashboards visualizing endpoint activity
- Install Splunk Security Essentials and ES Content Update
- Practice Kali-to-Windows network attacks (nmap, CrackMapExec, Metasploit)
- Set up alerting and email notifications
- Add Linux endpoint with auditd
- Document the full MITRE ATT&CK technique coverage achieved

## Acknowledgments

- [SwiftOnSecurity Sysmon Config](https://github.com/SwiftOnSecurity/sysmon-config)
- [Atomic Red Team by Red Canary](https://github.com/redcanaryco/atomic-red-team)
- [Splunk Free Trial](https://www.splunk.com/)

[First Detection] <img width="609" height="168" alt="Running T1059 001-1 Mimikatz Atomic Test" src="https://github.com/user-attachments/assets/73ee9769-f797-4a9d-ad0f-df1fff25694c" />

[Powershell output second detection] <img width="617" height="227" alt="Powershell output" src="https://github.com/user-attachments/assets/6cb91095-8303-461a-bf21-6c73d2cd539c" /> <img width="618" height="489" alt="Screenshot 2026-05-27 210511" src="https://github.com/user-attachments/assets/af5109ab-619c-43e0-93b9-fdae4cecc539" />
<img width="612" height="484" alt="Screenshot 2026-05-27 210553" src="https://github.com/user-attachments/assets/39b29973-514b-4131-8f6c-3a6b6277cbc0" />

[Splunk Outputs from detection] <img width="1885" height="933" alt="Screenshot 2026-05-27 210648" src="https://github.com/user-attachments/assets/c8a82349-51fa-4887-8c6e-a02e2e86f014" /> <img width="1906" height="942" alt="Screenshot 2026-05-27 210710" src="https://github.com/user-attachments/assets/ffdea1bd-9ecb-40d4-900c-72a56a6c4558" />

[Splunk Creation] <img width="1410" height="944" alt="Splunk creation events" src="https://github.com/user-attachments/assets/80df2540-ba75-42e0-bd69-6df402c645f6" />

[Splunk Processiong Execution Counts] <img width="1412" height="954" alt="Splunk processes by execution count" src="https://github.com/user-attachments/assets/a338aaf0-5948-49bb-b58d-7b3f48012413" />

[Splunk Live Events] <img width="1415" height="956" alt="Splunk Live events" src="https://github.com/user-attachments/assets/5faae420-df49-414d-9abf-47fbc095a6a9" />







