# Track A — SOC & SIEM Operations: Lab Architecture

A from-scratch VM + network design for Track A (Projects 02–10). Host has 32 GB RAM total — this design caps VM usage at ~70% (≈22 GB), leaving ~10 GB for the host OS, VirtualBox overhead, and anything else running. Kept deliberately separate from the Red-Team-Penetration-Testing-Lab attack network.

## Design principles

- 70% RAM ceiling, always. Target <=22 GB allocated to running VMs at any one time.
- One dedicated internal network for the whole SOC lab, isolated from the red team lab and from the production LAN.
- Every VM gets two adapters: Adapter 1 = NAT (internet, installs/updates only), Adapter 2 = the SOC lab's Host-only network (all internal traffic).
- Static IPs on the Host-only adapter for every VM, so configs (agent to manager, case management to SIEM, etc.) don't break on reboot.
- Never run everything at once — all 8 VMs together is ~27 GB, over the 22 GB ceiling. Power on in clusters matching whichever project is active.

## Network setup

1. VirtualBox -> File -> Host Network Manager -> Create a new Host-only adapter.
2. Subnet: 192.168.60.0/24, DHCP server disabled — static IPs assigned manually per VM.
3. Name convention: prefix every SOC-lab VM `soc-` to keep it visually distinct from the red-team lab VMs.

## VM inventory (trimmed to fit a 22 GB ceiling)

| VM name | Role | Project(s) | OS | RAM | vCPU | Disk | Static IP |
|---|---|---|---|---|---|---|---|
| soc-wazuh-mgr | Wazuh manager + indexer + dashboard | 02, 03, 05, 09, 10 | Ubuntu Server 22.04/24.04 | 6 GB | 3 | 50 GB | 192.168.60.10 |
| soc-elastic | Elastic Stack (Elasticsearch, Kibana, Fleet) | 04 | Ubuntu Server 22.04/24.04 | 6 GB | 3 | 50 GB | 192.168.60.11 |
| soc-thehive | TheHive + Cortex (case management) | 06 | Ubuntu Server 22.04/24.04 | 3 GB | 2 | 35 GB | 192.168.60.12 |
| soc-shuffle | Shuffle (SOAR, Docker-based) | 07 | Ubuntu Server 22.04/24.04 | 2 GB | 2 | 30 GB | 192.168.60.13 |
| soc-misp | MISP (threat intel platform) | 08 | Ubuntu Server 22.04/24.04 | 3 GB | 2 | 35 GB | 192.168.60.14 |
| soc-win-endpoint | Windows telemetry target (Sysmon, Advanced Audit Policy) | 03, 05, 09, 10 | Windows 10/11 eval | 3 GB | 2 | 50 GB | 192.168.60.20 |
| soc-linux-endpoint | Linux telemetry target (auditd) | 03, 05, 09, 10 | Ubuntu Server 22.04 | 2 GB | 1 | 25 GB | 192.168.60.21 |
| soc-caldera | Caldera — adversary emulation for hunting/purple-team | 09, 10 | Ubuntu Server 22.04 | 2 GB | 2 | 25 GB | 192.168.60.30 |

Total if every VM ran at once: ~27 GB — still over the 22 GB ceiling, so power-on groups (not "run everything") are how this actually stays within budget. These are lab-minimum specs, not comfortable-production — expect Wazuh/Elastic to feel a bit slow under load.

## Power-on groups

| Group | VMs powered on | RAM used | Headroom (of 22 GB) | For |
|---|---|---|---|---|
| A — SIEM core | soc-wazuh-mgr, soc-win-endpoint, soc-linux-endpoint | 11 GB | 11 GB | Projects 02, 03 |
| B — SIEM comparison | Group A + soc-elastic | 17 GB | 5 GB | Project 04 |
| C — Detection engineering | Group A only | 11 GB | 11 GB | Project 05 |
| D — Case management | soc-thehive, soc-wazuh-mgr | 9 GB | 13 GB | Project 06 |
| E — Automation | soc-shuffle, soc-thehive | 5 GB | 17 GB | Project 07 |
| F — Threat intel | soc-misp, soc-wazuh-mgr | 9 GB | 13 GB | Project 08 |
| G — Hunting & validation | soc-wazuh-mgr, both endpoints, soc-caldera | 13 GB | 9 GB | Projects 09, 10 |

Shut down VMs not actively in use — VirtualBox doesn't reclaim host RAM from an idle-but-running guest. Group B is the tightest fit — close unrelated host apps while it's running.

## Build order

1. Create the Host-only network (192.168.60.0/24) first.
2. Build soc-wazuh-mgr, soc-win-endpoint, soc-linux-endpoint — unblocks Projects 02 and 03.
3. Build the rest one VM at a time, right before the project that needs it.

## Next immediate step

Create the Host-only network and the three Group-A VMs. Once soc-wazuh-mgr is up, follow `project-02-wazuh-deployment/runbook.md` using 192.168.60.10 as the manager's address.
