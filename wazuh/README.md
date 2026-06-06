Wazuh SIEM Deployment

Added as Phase 2 of the SOC Analyst Home Lab. Wazuh provides centralized security monitoring across all lab endpoints — giving visibility into the hypervisor, DNS layer, attack platform, and both SIEM stacks simultaneously.

## Overview

Wazuh is an open-source SIEM and XDR platform used in real-world SOC environments. This deployment adds a second detection layer alongside Splunk, enabling:

- Real-time alert correlation across all lab VMs and containers
- File integrity monitoring (FIM) on every endpoint
- Vulnerability detection and configuration assessment
- MITRE ATT&CK-mapped alerting
- Centralized visibility into Pi-hole DNS activity

## Architecture

```
Proxmox Host (192.168.1.200)
│
├── CT 100 - Pi-hole (192.168.1.53)            ← Wazuh Agent (pihole)
├── VM 101 - Splunk Enterprise (192.168.1.205) ← Wazuh Agent (splunk)
├── VM 102 - Kali Linux (192.168.1.9)          ← Wazuh Agent (kali-vm)
├── VM 103 - Windows 10 Pro                    ← Wazuh Agent (win10)
└── VM 104 - Wazuh Server (192.168.1.201)      ← Manager + Dashboard
     │
     └── Wazuh Indexer (OpenSearch)
         Wazuh Manager
         Wazuh Dashboard (HTTPS :443)
```

The Proxmox host itself (pve) also runs a Wazuh agent, making the hypervisor a monitored endpoint.

## Tech Stack

| Component | Details |
|---|---|
| **Wazuh Version** | 4.14.1 (server) / 4.14.5 (agents) |
| **Server OS** | Ubuntu Server 22.04 LTS |
| **VM Resources** | 4 vCPU, 8GB RAM, 80GB disk |
| **Installation Method** | Single-node all-in-one (`wazuh-install.sh -a`) |
| **Dashboard** | HTTPS at `192.168.1.201:443` |

## Monitored Endpoints

| Agent ID | Name | IP | OS | Status |
|---|---|---|---|---|
| 002 | pve | 192.168.1.200 | Debian GNU/Linux 12 | Active |
| 004 | pihole | 192.168.1.53 | Debian GNU/Linux 12 | Active |
| 011 | kali-vm | 192.168.1.9 | Kali GNU/Linux 2026.1 | Active |
| 013 | splunk | 192.168.1.205 | Ubuntu 22.04 LTS | Active |
| 005 | win10 | 192.168.1.x | Windows 10 Pro | Active (when VM running) |

## Setup Highlights

1. **Single-node all-in-one install** — Wazuh indexer, manager, and dashboard deployed on a single Ubuntu 22.04 VM using the official installer script.

2. **Dual-NIC for Kali** — Kali is isolated on `vmbr1` (10.10.10.0/24) for attack simulations. A second NIC on `vmbr0` was added to allow the Wazuh agent to reach the manager without exposing Kali to the main network through the isolated interface.

3. **Proxmox host monitoring** — The Proxmox hypervisor itself runs a Wazuh agent, providing visibility into VM lifecycle events, host logins, and system changes at the hypervisor layer.

4. **Pi-hole agent via apt repo** — The Pi-hole LXC required adding the Wazuh apt repository rather than downloading the `.deb` directly, due to CDN access restrictions inside the LXC.

5. **Agent registration via agent-auth** — Each agent was enrolled using `agent-auth` against port 1515 on the Wazuh manager after resolving duplicate agent name conflicts caused by auto-registration on service start.

## Installation Summary

**Wazuh Server (Ubuntu 22.04):**
```bash
curl -4 -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

**Linux Agents (Debian/Ubuntu/Kali):**
```bash
curl -4 -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg \
  --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
sudo chmod 644 /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] \
  https://packages.wazuh.com/4.x/apt/ stable main" | \
  sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt update
sudo WAZUH_MANAGER='192.168.1.201' apt install wazuh-agent -y
sudo systemctl enable --now wazuh-agent
sudo /var/ossec/bin/agent-auth -m 192.168.1.201 -A <agent-name>
```

**Windows Agent:**
```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.5-1.msi `
  -OutFile $env:tmp\wazuh-agent
msiexec.exe /i $env:tmp\wazuh-agent /q `
  WAZUH_MANAGER='192.168.1.201' WAZUH_AGENT_NAME='win10'
NET START WazuhSvc
& "C:\Program Files (x86)\ossec-agent\agent-auth.exe" -m 192.168.1.201
```

## Troubleshooting Notes

**CDN Access Denied (XML response instead of script)**
```bash
# Fails:
curl -sO https://packages.wazuh.com/4.x/wazuh-install.sh
# Works:
curl -4 -sO https://packages.wazuh.com/4.14/wazuh-install.sh
```

**Duplicate Agent Name on Registration**
```bash
sudo /var/ossec/bin/manage_agents -l        # find the ID
sudo /var/ossec/bin/manage_agents -r <ID>   # remove it
sudo systemctl stop wazuh-agent
sudo sh -c 'echo "" > /var/ossec/etc/client.keys'
sudo /var/ossec/bin/agent-auth -m 192.168.1.201 -A <name>
sudo systemctl start wazuh-agent
```

**Missing lsb-release on Proxmox**
```bash
apt install lsb-release -y
```

**ossec.conf Manager Address Not Set**
```bash
nano /var/ossec/etc/ossec.conf
# Change: <address>MANAGER_IP</address>
# To:     <address>192.168.1.201</address>
systemctl restart wazuh-agent
```

## Wazuh Capabilities in Use

| Feature | Status |
|---|---|
| Log collection | ✅ Active on all agents |
| File Integrity Monitoring (FIM) | ✅ Active |
| Vulnerability Detection | ✅ Active |
| Configuration Assessment (SCA) | ✅ Active |
| MITRE ATT&CK mapping | ✅ Active |
| Active Response | 🔲 Planned |
| Pi-hole log forwarding | 🔲 Planned |

## Future Work

- Configure Pi-hole log forwarding to Wazuh via `ossec.conf` localfile block
- Run Atomic Red Team attacks and document Wazuh alert mappings vs. Splunk detections
- Build a Wazuh dashboard showing cross-platform alert trends
- Set up Wazuh Active Response to auto-block IPs triggering brute force rules
- Integrate Wazuh alerts into a SOAR workflow (Shuffle)
