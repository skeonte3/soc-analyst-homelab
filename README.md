# SOC Analyst Home Lab

A fully functional Security Operations Center lab built on Proxmox, simulating a real-world detection and response environment. Used to practice SIEM workflows, attack detection, and threat hunting on Windows endpoints.

## Overview

This lab provides hands-on experience with the core technologies and workflows of a SOC analyst:

- Endpoint telemetry collection (Sysmon)
- SIEM log aggregation and search (Splunk Enterprise)
- SIEM and XDR platform (Wazuh)
- Adversary simulation (Atomic Red Team)
- Network segmentation and isolation
- Detection engineering
- DNS filtering and monitoring (Pi-hole)


## Architecture
[Internet] → [Spectrum Router] → [Proxmox Host (Dell OptiPlex 7020)]
│
┌──────────────────┴──────────────────┐
│ vmbr0 (LAN 192.168.1.0/24)          │
│  ├── Pi-hole LXC (CT 100)           │
│  ├── Splunk Enterprise VM (101)     │
│  └── Wazuh SIEM VM (104)            │
│                                     │
│ vmbr1 (Isolated 10.10.10.0/24)      │
│  ├── Kali Linux VM (102)            │
│  └── Windows 10 Victim VM (103)     │
└─────────────────────────────────────┘
The Windows VM has a second NIC on vmbr0 to forward logs to Splunk while remaining isolated from the home network for attack simulations on vmbr1. Kali also has a second NIC on vmbr0 for Wazuh agent connectivity.

## Tech Stack

| Component | Purpose |
|---|---|
| **Proxmox VE 8.4** | Hypervisor |
| **Splunk Enterprise** | SIEM and log analysis |
| **Splunk Universal Forwarder** | Log shipping from endpoints |
| **Wazuh 4.14** | SIEM, XDR, and endpoint monitoring |
| **Sysmon** (SwiftOnSecurity config) | Windows endpoint telemetry |
| **Atomic Red Team** | MITRE ATT&CK-aligned adversary simulation |
| **Pi-hole** | DNS filtering and query logging |
| **Kali Linux 2026.1** | Attacker platform |
| **Windows 10 Pro 22H2** | Target endpoint |
| **Ubuntu Server 22.04** | Splunk and Wazuh host OS |


## Lab Specifications

- **Host hardware:** Dell OptiPlex 7020, 32GB RAM, 1.8TB SSD
- **Network:** Dual bridges (LAN + isolated lab network)
- **VM resources:**
  * Splunk: 4 cores, 8GB RAM, 50GB disk
  * Wazuh: 4 cores, 8GB RAM, 80GB disk
  * Kali: 2 cores, 4GB RAM, 40GB disk
  * Windows 10: 4 cores, 8GB RAM, 80GB disk


## Lab Components

### 🔵 Splunk SIEM
Splunk Enterprise receives Sysmon logs from the Windows 10 endpoint via the Universal Forwarder. Used for detection engineering, SPL queries, and adversary simulation validation.

### 🟠 Wazuh SIEM
Wazuh monitors all lab endpoints simultaneously — Proxmox host, Pi-hole, Kali, Windows 10, and Splunk. Provides file integrity monitoring, vulnerability detection, configuration assessment, and MITRE ATT&CK-mapped alerting.

→ See [/wazuh/README.md](./wazuh/README.md) for full deployment details.

### 🟢 Pi-hole
DNS-level ad and tracker blocking with query logging. Configured as the primary DNS server for the lab via Proxmox's local resolver.

### 🔴 Kali Linux
Attack platform on an isolated network segment (vmbr1). Used for adversary simulation against the Windows 10 target.

### 🪟 Windows 10
Target endpoint with Sysmon (SwiftOnSecurity config), Splunk Universal Forwarder, and Wazuh agent installed. Dual-NIC for isolated attack surface and log forwarding.


## Setup Highlights

1. **Network isolation** — Created an additional Linux bridge (vmbr1) with no gateway to fully isolate attack traffic from the home network and internet.
2. **Dual-NIC Windows VM** — One NIC on the isolated network for receiving attacks, one on the LAN for forwarding telemetry to Splunk and Wazuh.
3. **Dual-NIC Kali VM** — Isolated NIC on vmbr1 for attacks, second NIC on vmbr0 for Wazuh agent connectivity.
4. **Sysmon with SwiftOnSecurity config** — Industry-standard config that captures meaningful events without overwhelming the SIEM with noise.
5. **Splunk receiver on port 9997** — Standard forwarder ingestion endpoint configured to receive logs from Windows endpoints.
6. **Wazuh all-in-one deployment** — Single-node install covering indexer, manager, and dashboard on one Ubuntu VM.
7. **Atomic Red Team** for repeatable, MITRE-mapped attack simulations.


## Sample Detections

Real Splunk searches built and tested in this lab:

**Process creation by PowerShell:**
index=main EventCode=1 Image="powershell"
**Encoded PowerShell commands (suspicious):**
index=main EventCode=1 Image="powershell" CommandLine="-EncodedCommand"
**All Windows endpoint activity (last hour):**
index=main host=win10
## Techniques Tested

| MITRE ATT&CK ID | Technique | Status |
|---|---|---|
| T1059.001 | PowerShell | ✅ Detected |
| (more to come) | | |


## Lessons Learned

- **Physical layer matters first.** A bad ethernet cable forced the NIC to negotiate at 10 Mbps with constant link flapping, which presented as application-layer symptoms (ERR_CONTENT_LENGTH_MISMATCH on the Proxmox web UI). Always check `dmesg` for link speed and stability before assuming software issues.
- **Config file encoding can silently fail.** Splunk's Universal Forwarder ignored an `inputs.conf` written with Notepad's default encoding. Fixed by rewriting via PowerShell with explicit ASCII encoding.
- **Local System doesn't get Sysmon log access by default.** The forwarder service needed to run under an admin account (or have the Sysmon channel SDDL updated) to subscribe to the Sysmon event log.
- **Tamper Protection blocks Defender PowerShell commands.** Atomic Red Team requires disabling both Tamper Protection (in the Windows Security UI) and real-time protection.
- **SwiftOnSecurity's Sysmon config is the right starting point.** It produces high-signal events without flooding the SIEM with normal Windows noise.
- **CDN access restrictions affect agent downloads inside LXC containers.** Wazuh's generic 4.x URL returned Access Denied from inside the Pi-hole LXC. Fixed by using the version-specific URL or adding the apt repository instead.
- **Wazuh agents auto-register on startup.** This causes duplicate agent name conflicts when doing manual registration. Fix by stopping the agent, clearing `client.keys`, then re-registering before restarting.


## Repository Structure
soc-analyst-homelab/
├── README.md               ← This file
├── wazuh/
│   └── README.md           ← Wazuh deployment guide and agent setup
└── screenshots/            ← Lab screenshots
## Future Work

- ✅ Wazuh SIEM deployed — see [/wazuh](./wazuh/README.md)
- Build dashboards visualizing endpoint activity in Splunk and Wazuh
- Install Splunk Security Essentials and ES Content Update
- Practice Kali-to-Windows network attacks (nmap, CrackMapExec, Metasploit)
- Set up alerting and email notifications in Wazuh
- Configure Pi-hole log forwarding to Wazuh
- Run Atomic Red Team and compare detections between Splunk and Wazuh
- Add Wazuh Active Response for automated blocking
- Document full MITRE ATT&CK technique coverage achieved


## Acknowledgments

- [SwiftOnSecurity Sysmon Config](https://github.com/SwiftOnSecurity/sysmon-config)
- [Atomic Red Team by Red Canary](https://github.com/redcanaryco/atomic-red-team)
- [Splunk Free Trial](https://www.splunk.com/)
- [Wazuh Open Source](https://wazuh.com/)

## Screenshots

[First Detection]

![Running T1059 001-1 Mimikatz Atomic Test](Running%20T1059.001-1%20Mimikatz%20Atomic%20Test.png)

[PowerShell Output Second Detection]

![Powershell output](Powershell%20output.png)

[Splunk Outputs from Detection]

![Screenshot 2026-05-27 210511](Screenshot%202026-05-27%20210511.png)
![Screenshot 2026-05-27 210553](Screenshot%202026-05-27%20210553.png)

[Splunk Creation Events]

![Screenshot 2026-05-27 210648](Screenshot%202026-05-27%20210648.png)

[Splunk Process Execution Counts]

![Screenshot 2026-05-27 210710](Screenshot%202026-05-27%20210710.png)
