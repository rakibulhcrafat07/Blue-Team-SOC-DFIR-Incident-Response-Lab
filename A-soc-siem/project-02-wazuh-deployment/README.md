# Project 02 — Wazuh SIEM Deployment

**Status:** 🟡 In Progress (core build and detections done; SOC Overview dashboard screenshot and final report pending)
**Track:** A — SOC & SIEM Operations
**Case ID:** CB-02-001

## Overview

A single-node Wazuh 4.14.9 deployment (manager, indexer, dashboard) on an isolated VirtualBox lab network, with one Linux endpoint enrolled as an agent. The lab then exercised File Integrity Monitoring, SCA, Vulnerability Detection, two simulated ATT&CK techniques (T1110, T1136), one custom detection rule and three dashboard visualizations. All attacks are simulated, lab-only, against my own VMs.

**Scope note:** the original plan also had a Windows endpoint (`soc-win-endpoint`). I dropped it for this run to save time and RAM, so this project is **Linux-endpoint only**. No Windows evidence exists and none is claimed.

## Environment

| Item | Value |
|---|---|
| Case ID | CB-02-001 |
| Host | Windows laptop, Ryzen 7 3750H, 24 GB RAM, VirtualBox 7.x |
| Lab network | Host-only 192.168.60.0/24 (DHCP off, host 192.168.60.1) plus NAT for internet |
| Manager VM | `soc-wazuh-mgr`, Ubuntu Server 24.04 LTS, 7 GB RAM / 3 vCPU, 192.168.60.10 |
| Endpoint VM | `soc-linux-endpoint`, Ubuntu Server 24.04.5 LTS, 2 GB RAM / 1 vCPU, 192.168.60.21 |
| Tool versions | Wazuh 4.14.9 (all-in-one installer, `-a`), wazuh-agent 4.14.9 |
| Time zone | VMs and logs in UTC; the dashboard shows browser time (UTC+6), so dashboard times are 6 hours ahead of log times |
| Date | 2026-10-25 |

## Tools

Wazuh manager, indexer and dashboard; VirtualBox; OpenSSH; tmux; the Wazuh dashboard (Threat Hunting, FIM, SCA, Vulnerability Detection, Visualize).

## Evidence Inventory

All screenshots are in [`images/`](images/). The installer screenshot has the admin password blacked out; a separate screenshot that listed all internal user passwords was deliberately not kept.

| Item | File | Size | MD5 | SHA-256 | Source | Acquired |
|---|---|---|---|---|---|---|
| Wazuh dashboard home, no agents yet | `images/01-wazuh-dashboard-home.png` | 181,303 B | `98ba2fa81ebd6294288904e04e858592` | `45e38b893e50ae117961e37b903ae6c6f32948c5592ccf6bbaaebe4e7ee0bf1b` | Dashboard/terminal capture | 2026-10-25 |
| soc-wazuh-mgr hostname and 192.168.60.10/24 on enp0s8 | `images/02-wazuh-mgr-hostname-ip.png` | 35,803 B | `962a2d76bca451e1972f3fdc502bde0e` | `52b51b3cba3c14252f87f374eda9ebc7eb7c5ca4408d6749634cdac8dd286d26` | Dashboard/terminal capture | 2026-10-25 |
| VirtualBox base VM with Base-clean snapshot | `images/03-virtualbox-base-snapshot.png` | 145,280 B | `81a6f292a5620b3d0c14d08af94c624e` | `a6c81013fff2d21dcb4180c4160d8256597afaeebab236e675b6a2190fb371ee` | Dashboard/terminal capture | 2026-10-25 |
| Host-only network 192.168.60.1 before Apply | `images/04-host-only-network-before-apply.png` | 62,500 B | `a9b13941f89aaca873dac242bd00f3ca` | `3c329fbb549ca8c135d21c11ef9d50137ed7209ce7d640369ea2bb1cd40f2194` | Dashboard/terminal capture | 2026-10-25 |
| Wazuh installer finished (admin password blacked out) | `images/05-wazuh-install-finished-password-redacted.png` | 41,416 B | `bed3944181414c71a49c9987dc6a7ccf` | `76c04038f408c52b6c23a599258fa99da7e6bbc71c829cec4a976e10a9982411` | Dashboard/terminal capture | 2026-10-25 |
| Deploy new agent wizard, DEB amd64 | `images/06-deploy-agent-wizard-deb-amd64.png` | 39,413 B | `81017fd7d6a2aebdc6ba248fb26f2f87` | `d2b519775bb4bd8c474c269529ef38ea6bdf972fc145635ebb269add73b1141f` | Dashboard/terminal capture | 2026-10-25 |
| wazuh-agent 4.14.9 installed on soc-linux-endpoint | `images/07-linux-agent-install-4.14.9.png` | 98,019 B | `5c56e1624bbdff5c3be07c5f0472e57b` | `90fdff5f54f5bc7f993ea7e2fd9755b1f00dc781f34f88f6ff8373b1d953fae1` | Dashboard/terminal capture | 2026-10-25 |
| wazuh-agent service active (running) | `images/08-linux-agent-service-running.png` | 50,280 B | `d9cafb573d8bc1af3b91f529e53c4f46` | `d05bd1d2215c47a3bb8c36c0ba79fbe945f11c99d9fb463e34d69db56634b8b4` | Dashboard/terminal capture | 2026-10-25 |
| Agents list, soc-linux-endpoint Active | `images/09-agents-list-linux-active.png` | 107,380 B | `879eb9276a1e5b133e0fb784552bd4cb` | `93e20236b9aa9fa3fc5564f477fe9750db87803639daddaa9624d2f167a72bba` | Dashboard/terminal capture | 2026-10-25 |
| FIM dashboard, added and modified | `images/10-fim-dashboard-added-modified.png` | 96,550 B | `6509101088c031ada169d744208d2618` | `7664e6bd36f410ba1e8472b1950a7f35a6bb2895280e118da382004909cf8742` | Dashboard/terminal capture | 2026-10-25 |
| FIM events for /opt/fim-test/a.txt, rules 554 and 550 | `images/11-fim-events-a-txt-rule554-550.png` | 152,283 B | `0763ab4eec63f593b9da05c8c406d42a` | `a0c982cd8e1e2d8c7b78546143799c39ae1296966cc2eff9c4029ca9bd97cd8c` | Dashboard/terminal capture | 2026-10-25 |
| T1110 simulation terminal, 12 failed SSH logins | `images/12-t1110-bruteforce-terminal.png` | 138,384 B | `2ee75128badd39581c6724773dbdfaa2` | `da1583c3664ccb2e1ac14a33153fc9bb2f814f0c52111bd1c9f585748d12f58a` | Dashboard/terminal capture | 2026-10-25 |
| Rule 5712 definition (frequency 8, timeframe 120) | `images/13-rule-5712-threshold-frequency8-timeframe120.png` | 42,010 B | `f4cf3e210686b84e381a79f4b4519cc7` | `c5f7971b3ebbe6977c729e4b9b0ff3b671fb41ba80d2cb25a6d0a24b8b47eeed` | Dashboard/terminal capture | 2026-10-25 |
| T1136 simulation, useradd suspicious_user | `images/14-t1136-useradd-suspicious-user-terminal.png` | 8,201 B | `03a2738f63fea285fa72efe4d54690a8` | `891613a07f4bf4ee8277f71f4583890c793c52dd90146b6c2f04865a40151646` | Dashboard/terminal capture | 2026-10-25 |
| Threat Hunting events 5710, 5503, 5902, 5901 | `images/15-threat-hunting-events-5710-5503-5902-5901.png` | 311,607 B | `6bcd29c889322172546dce65f2258d5e` | `ea5de72ceae16fe002a9216bb9700b2019108fafed0bfeb79b36801c46ee7e62` | Dashboard/terminal capture | 2026-10-25 |
| Second brute-force run, 5710/5503/5551 | `images/16-t1110-second-run-5710-5503-5551.png` | 297,110 B | `147659a6e5898a740fc0b072c4b7cf9d` | `8550d76324d78bcb51eb9878f4ce8ce3ed50bda1e042f0bbe1371993cf03a155` | Dashboard/terminal capture | 2026-10-25 |
| alerts.json counts, 5710=62, 5712=2 | `images/17-alerts-json-count-5710-62-5712-2.png` | 80,538 B | `5e5c3a96d2735d7901b0b034d07fe2c1` | `9ac0129d23475aad8f8467d0cf39eb8dbe0d0e5cda482d28cae5ea9f7f6f9d74` | Dashboard/terminal capture | 2026-10-25 |
| Custom rule 100100 in local_rules.xml, manager restarted | `images/18-custom-rule-100100-local_rules-and-manager-restart.png` | 47,645 B | `696f30d0e4179ed2ea3393151de7a600` | `89fb2cf98d2cdc814d6aa69a66d24aa7a5a77081794b264e72cefdd92b6f66d7` | Dashboard/terminal capture | 2026-10-25 |
| T1136 simulation, useradd suspicious_user2 | `images/19-useradd-suspicious-user2-terminal.png` | 16,027 B | `4a76625c3d687214d378b816e0a6b841` | `5b3b2f6c820771fc10cd395b63e63ed5698f52194b0fc6a7013852c1189e4c38` | Dashboard/terminal capture | 2026-10-25 |
| Rule 100100 firing vs 5902 | `images/20-custom-rule-100100-fires-vs-5902.png` | 165,637 B | `a12930b760cf8bb6584b5443f079d915` | `2d9f80ec6fdfe89a6e7a10245b3dc8cc3baa4cceccadc818b5c95f20f3da0c65` | Dashboard/terminal capture | 2026-10-25 |
| SCA, CIS Ubuntu 24.04 benchmark, score 50% | `images/21-sca-cis-ubuntu-2404-score-50.png` | 192,782 B | `e0201ae926f839d886da11bce9694c20` | `c02354162d5efaf433f6d2ffc62c113e7afdccde35e607b9acc186d3312c727c` | Dashboard/terminal capture | 2026-10-25 |
| Vulnerability Detection dashboard | `images/22-vulnerability-detection-dashboard.png` | 151,675 B | `e6980e3c3179ea34e7790110556b5ad6` | `19c433f1b6b30807c2df982c2df1f47e6f1408fef73bdd5874dae5d5623fe960` | Dashboard/terminal capture | 2026-10-25 |
| Pie visualization config, alerts by agent | `images/23-viz-alerts-by-agent-pie-config.png` | 122,102 B | `6a2b0cd6fec28edf0c190a79013356f7` | `6cdfc7c90e5b1c9e5108e296dfecf1d3177224dfb8aa6dbccc72a3f5e62c37fb` | Dashboard/terminal capture | 2026-10-25 |
| Three saved SOC visualizations | `images/24-visualizations-list-three-saved.png` | 65,161 B | `087b1ae3732361d56b809a25674d713a` | `4ac99dc64a40afacf9b19a79ecf346a6be06996e9ae1a31b2753bd5567c3bf5e` | Dashboard/terminal capture | 2026-10-25 |

## Build Steps

### Step 1 — Plan and deploy Wazuh

1. Built a base VM (`soc-ubuntu-base`, Ubuntu 24.04), extended the LVM volume (`sudo lvextend -r -l +100%FREE /dev/ubuntu-vg/ubuntu-lv`), took a `Base-clean` snapshot (image 03), then made Full Clones with new MAC addresses.
2. Created the host-only network 192.168.60.0/24 with DHCP off (image 04). Gave `enp0s8` a static IP via `/etc/netplan/60-lab.yaml`, regenerated `machine-id` and the SSH host keys on each clone (image 02).
3. Installed Wazuh inside `tmux` over SSH:

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a -i
```

4. Installer finished and the dashboard answered at `https://192.168.60.10` (images 05, 01).

### Step 2 — Enroll the Linux agent

Used **Agents → Deploy new agent** (DEB amd64, server `192.168.60.10`, name `soc-linux-endpoint`; image 06), ran the generated command on the endpoint, then:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now wazuh-agent
systemctl status wazuh-agent
```

Agent installed at 4.14.9 (image 07), service running (image 08), shown **Active** in the agent list (image 09).

### Step 3 — FIM, SCA, Vulnerability Detection

- **FIM:** added `/opt/fim-test` to `<syscheck>` with realtime monitoring on the endpoint, created and edited `a.txt`. Result: rule 554 (file added, level 5) and rule 550 (integrity checksum changed, level 7) for `/opt/fim-test/a.txt` (images 10, 11).
- **SCA:** CIS Ubuntu Linux 24.04 LTS Benchmark v1.0.0 on `soc-linux-endpoint`: 119 passed, 118 failed, 42 not applicable, score 50% (image 21).
- **Vulnerability Detection:** enabled by default since Wazuh 4.8 (`<vulnerability-detection>` block). Dashboard shows 243 critical, 1,379 high, 534 medium, 48 low and 1,612 pending findings; most findings are in the kernel package `linux-image-6.8.0-147-generic` (image 22).

### Step 4 — Test events (simulated attacks)

- **T1110 Brute Force:** from `soc-wazuh-mgr`, repeated failed SSH logins as non-existent user `baduser` against `192.168.60.21` (12 attempts, then 20). Wazuh raised 5710 (non-existent user), 5503 (PAM login failed) and 5551 (PAM multiple failures, level 10). Rule 5712 needs `frequency=8` within `timeframe=120` s (image 13), and did not appear after the first 12-attempt run, which was only ~6 seconds. After the second run, `alerts.json` showed 62 lines for rule 5710 and 2 for 5712 (images 12, 16, 17).
- **T1136 Create Account:** `sudo useradd -m suspicious_user` on `soc-linux-endpoint` (image 14). Wazuh raised 5902 (new user added, level 8) and 5901 (new group added, level 8) (image 15).

### Step 5 — Tune rules (custom rule 100100)

Added a child rule of 5902 to `/var/ossec/etc/rules/local_rules.xml` that raises the level to 10 and tags MITRE T1136.001, then restarted the manager (image 18). Re-ran the test with `suspicious_user2` and `suspicious_user3` (image 19). Rule 100100 fired at level 10 for both, while the earlier `suspicious_user` event stayed at 5902 level 8, and no duplicate 5902 alert appeared next to 100100 (image 20).

### Step 6 — SOC dashboard

Saved three visualizations on `wazuh-alerts-*`: **SOC - Alerts by agent** (pie, terms on `agent.name`), **SOC - Alerts over time** (area), **SOC - Top rule IDs** (data table) (images 23, 24). Combining them into one "SOC Overview" dashboard is **not done yet**.

## Problems Encountered

| Problem | What happened | Root cause | Fix | Lesson |
|---|---|---|---|---|
| SSH to VM refused | `Connection refused` on the forwarded port | NAT port-forward rule was not actually saved from the GUI | `VBoxManage controlvm "soc-ubuntu-base" natpf1 "ssh,tcp,127.0.0.1,2222,10.0.2.15,22"` | Verify the rule exists before blaming SSH |
| Second NIC missing | Host-only adapter absent after boot | Adapter 2 cannot be added while the VM runs and was not enabled before first start | Powered off, enabled Host-only adapter, checked 192.168.60.1/24 DHCP off | Enable both adapters before first boot |
| Clone kept old identity | Clone dialog defaults (NAT-only MAC policy) | Defaults clicked through | Re-cloned with new MACs, regenerated `machine-id` and SSH host keys | Check clone options every time |
| Password rejected | First choice of admin password refused by `wazuh-passwords-tool.sh` | Tool allows only symbols `.*+?-` | Chose a password using an allowed symbol | Read the policy first |
| Extracted file instead of printing it | `tar -o` used instead of `-O` | Flag typo | Read with `sudo cat`, then removed `wazuh-install-files` | Remove files holding credentials after use |
| Dashboard not loading in built-in browser | Self-signed certificate | Pane cannot click through the warning | Opened the dashboard in a normal browser and accepted the certificate | Lab certs are self-signed by design |
| Rule 5712 not seen | Only 5710/5503 after first brute force | 12 attempts in ~6 s, and rule 5712 needs the threshold above | Ran 20 attempts, confirmed with `alerts.json` counts | Read the rule definition before testing |
| Windows endpoint dropped | No Windows evidence | Chose to save time and RAM | Documented Linux-only scope | Do not claim what was not built |

## Findings

| # | Observation | Evidence location | Interpretation | Confidence |
|---|---|---|---|---|
| 1 | Agent `soc-linux-endpoint` (192.168.60.21, Ubuntu 24.04.5, 4.14.9) is Active | image 09 | Enrollment works over the host-only network | High |
| 2 | FIM raised rules 554 and 550 for `/opt/fim-test/a.txt` about 20 ms apart (19:20:34, dashboard time) | image 11 | Realtime syscheck works | High |
| 3 | Failed logins for non-existent user raised 5710 and 5503; 5551 (level 10) fired on the second run; 5712 fired twice in total | images 15, 16, 17 | Brute-force pattern is detected once the threshold is met | High |
| 4 | `useradd` produced 5902 and 5901 (level 8) | image 15 | New-account activity is visible | High |
| 5 | Custom rule 100100 (level 10) fired for `suspicious_user2` and `suspicious_user3` | image 20 | Tuning raised severity without duplicates | High |
| 6 | SCA score 50% (119 passed, 118 failed) on a default Ubuntu install | image 21 | Default install is far from CIS; this is expected | High |
| 7 | 3,816 vulnerability findings, almost all in one kernel package | image 22 | A pending kernel update would remove most findings; not tested | Medium |

## MITRE ATT&CK

| Technique | Name | How tested | Detected by |
|---|---|---|---|
| T1110 | Brute Force | Failed SSH logins as `baduser` | 5710, 5503, 5551, 5712 |
| T1136.001 | Create Account: Local Account | `useradd suspicious_user*` | 5902, 5901, custom 100100 |

## IOCs

All values are simulated lab artifacts, not real threats.

| Type | Value | Context |
|---|---|---|
| Username | `baduser` | Non-existent user used in simulated brute force |
| Username | `suspicious_user`, `suspicious_user2`, `suspicious_user3` | Accounts created on purpose (simulated) |
| IP (source) | 192.168.60.10 | `soc-wazuh-mgr`, attacker side of the simulation |
| IP (target) | 192.168.60.21 | `soc-linux-endpoint` |

## Deliverables

- [x] Investigation notes (Build Steps, Problems, Findings above)
- [x] Screenshots in `images/` (24 files)
- [x] Agent inventory (image 09, one Linux agent)
- [x] Tuning log (custom rule 100100, Step 5)
- [ ] Dashboard screenshot (SOC Overview, three visualizations saved, dashboard not assembled yet)
- [ ] Windows endpoint (dropped, not built)
- [ ] Final report in `report/`

## Key Learnings

1. Read a rule's definition (`frequency`, `timeframe`) before testing it; a too-short simulation will not trigger a threshold rule.
2. A clone needs its own MAC, `machine-id` and SSH host keys, or it behaves like its parent.
3. Dashboard time (browser zone) and log time (UTC) differ, so write the zone next to every timestamp.
4. Never keep screenshots with credentials; black them out or do not keep them.
