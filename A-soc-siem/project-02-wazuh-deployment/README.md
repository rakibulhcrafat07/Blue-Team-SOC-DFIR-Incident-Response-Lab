# Project 02 — Wazuh SIEM Deployment

**Status:** 🟡 In Progress (core build, detections and tuning done; SOC Overview dashboard screenshot and final report pending)
**Track:** A — SOC & SIEM Operations
**Case ID:** CB-02-001
**Date:** 2026-10-25

> Evidence screenshots are referenced as `[img NN]` and live in [`images/`](images/). Every file is listed with its size and hash in the Evidence Inventory.

---

## 1. Summary

This project builds a working SIEM from scratch on a single laptop. A Wazuh 4.14.9 all-in-one server (manager, indexer and dashboard) runs in one VirtualBox VM on an isolated lab network. One Ubuntu endpoint is enrolled as an agent. On top of that base I turned on and tested the main Wazuh capabilities a SOC analyst uses every day: File Integrity Monitoring (FIM), Security Configuration Assessment (SCA), Vulnerability Detection, detection of two simulated ATT&CK techniques (T1110 Brute Force and T1136 Create Account), one custom detection rule, and three dashboard visualizations.

All attacks are simulated, run only between my own lab VMs, and the account names are fake.

**Scope change, stated up front:** the original plan also included a Windows endpoint (`soc-win-endpoint`). I dropped it in this run to save time and RAM. This project therefore has **one Linux endpoint only**, and there is no Windows evidence. Windows-specific detections (Sysmon, Windows event logs) are not covered here.

## 2. Objectives

1. Deploy a single-node Wazuh server and reach the dashboard from the host.
2. Enroll an endpoint as an agent and confirm it reports as Active.
3. Enable and test FIM, SCA and Vulnerability Detection on that endpoint.
4. Generate known attack activity (T1110, T1136) and confirm Wazuh raises the expected rules.
5. Tune detection with one custom rule and prove it fires without duplicating the original alert.
6. Build dashboard visualizations that summarise the lab for a SOC overview.

## 3. Lab architecture

```
 Windows laptop (host, VirtualBox)           192.168.60.1
        |
        |  Host-only network 192.168.60.0/24  (DHCP off)
        |
   +----+-----------------------+---------------------------+
   |                            |                           |
 soc-wazuh-mgr              soc-linux-endpoint        (soc-win-endpoint
 192.168.60.10              192.168.60.21              planned, not built)
 Wazuh manager +            Wazuh agent 4.14.9
 indexer + dashboard        Ubuntu 24.04.5
 Ubuntu 24.04 LTS
```

Every VM has two adapters: adapter 1 is NAT (internet, for package downloads), adapter 2 is the host-only network (the lab network). The lab network has no gateway, so lab traffic never leaves the laptop.

| Item | Value |
|---|---|
| Host | Windows laptop, Ryzen 7 3750H, 24 GB RAM, VirtualBox 7.x |
| Manager VM | `soc-wazuh-mgr`, Ubuntu Server 24.04 LTS, 7 GB RAM, 3 vCPU, 192.168.60.10 |
| Endpoint VM | `soc-linux-endpoint`, Ubuntu Server 24.04.5 LTS, 2 GB RAM, 1 vCPU, 192.168.60.21 |
| Tool versions | Wazuh 4.14.9 (installer `wazuh-install.sh`), wazuh-agent 4.14.9 |
| Time zone | VMs and logs use UTC. The dashboard shows browser time (UTC+6), so dashboard times are 6 hours ahead of log times. |

## 4. Evidence inventory

All files are in [`images/`](images/). `SHA256SUMS.txt` in the same folder holds the same SHA-256 list for a quick check with `sha256sum -c SHA256SUMS.txt`. The installer screenshot has the admin password blacked out. A separate screenshot that showed all internal user passwords was not kept and is not published.

| Item | File | Size | SHA-256 | Acquired |
|---|---|---|---|---|
| Wazuh dashboard home, no agents yet | `images/01-wazuh-dashboard-home.png` | 181,303 B | `45e38b893e50ae117961e37b903ae6c6f32948c5592ccf6bbaaebe4e7ee0bf1b` | 2026-10-25 |
| soc-wazuh-mgr hostname and 192.168.60.10/24 on enp0s8 | `images/02-wazuh-mgr-hostname-ip.png` | 35,803 B | `52b51b3cba3c14252f87f374eda9ebc7eb7c5ca4408d6749634cdac8dd286d26` | 2026-10-25 |
| VirtualBox base VM with Base-clean snapshot | `images/03-virtualbox-base-snapshot.png` | 145,280 B | `a6c81013fff2d21dcb4180c4160d8256597afaeebab236e675b6a2190fb371ee` | 2026-10-25 |
| Host-only network 192.168.60.1 before Apply | `images/04-host-only-network-before-apply.png` | 62,500 B | `3c329fbb549ca8c135d21c11ef9d50137ed7209ce7d640369ea2bb1cd40f2194` | 2026-10-25 |
| Wazuh installer finished (admin password blacked out) | `images/05-wazuh-install-finished-password-redacted.png` | 41,416 B | `76c04038f408c52b6c23a599258fa99da7e6bbc71c829cec4a976e10a9982411` | 2026-10-25 |
| Deploy new agent wizard, DEB amd64 | `images/06-deploy-agent-wizard-deb-amd64.png` | 39,413 B | `d2b519775bb4bd8c474c269529ef38ea6bdf972fc145635ebb269add73b1141f` | 2026-10-25 |
| wazuh-agent 4.14.9 installed on soc-linux-endpoint | `images/07-linux-agent-install-4.14.9.png` | 98,019 B | `90fdff5f54f5bc7f993ea7e2fd9755b1f00dc781f34f88f6ff8373b1d953fae1` | 2026-10-25 |
| wazuh-agent service active (running) | `images/08-linux-agent-service-running.png` | 50,280 B | `d05bd1d2215c47a3bb8c36c0ba79fbe945f11c99d9fb463e34d69db56634b8b4` | 2026-10-25 |
| Agents list, soc-linux-endpoint Active | `images/09-agents-list-linux-active.png` | 107,380 B | `93e20236b9aa9fa3fc5564f477fe9750db87803639daddaa9624d2f167a72bba` | 2026-10-25 |
| FIM dashboard, added and modified | `images/10-fim-dashboard-added-modified.png` | 96,550 B | `7664e6bd36f410ba1e8472b1950a7f35a6bb2895280e118da382004909cf8742` | 2026-10-25 |
| FIM events for /opt/fim-test/a.txt, rules 554 and 550 | `images/11-fim-events-a-txt-rule554-550.png` | 152,283 B | `a0c982cd8e1e2d8c7b78546143799c39ae1296966cc2eff9c4029ca9bd97cd8c` | 2026-10-25 |
| T1110 simulation terminal, 12 failed SSH logins | `images/12-t1110-bruteforce-terminal.png` | 138,384 B | `da1583c3664ccb2e1ac14a33153fc9bb2f814f0c52111bd1c9f585748d12f58a` | 2026-10-25 |
| Rule 5712 definition (frequency 8, timeframe 120) | `images/13-rule-5712-threshold-frequency8-timeframe120.png` | 42,010 B | `c5f7971b3ebbe6977c729e4b9b0ff3b671fb41ba80d2cb25a6d0a24b8b47eeed` | 2026-10-25 |
| T1136 simulation, useradd suspicious_user | `images/14-t1136-useradd-suspicious-user-terminal.png` | 8,201 B | `891613a07f4bf4ee8277f71f4583890c793c52dd90146b6c2f04865a40151646` | 2026-10-25 |
| Threat Hunting events 5710, 5503, 5902, 5901 | `images/15-threat-hunting-events-5710-5503-5902-5901.png` | 311,607 B | `ea5de72ceae16fe002a9216bb9700b2019108fafed0bfeb79b36801c46ee7e62` | 2026-10-25 |
| Second brute-force run, 5710/5503/5551 | `images/16-t1110-second-run-5710-5503-5551.png` | 297,110 B | `8550d76324d78bcb51eb9878f4ce8ce3ed50bda1e042f0bbe1371993cf03a155` | 2026-10-25 |
| alerts.json counts, 5710=62, 5712=2 | `images/17-alerts-json-count-5710-62-5712-2.png` | 80,538 B | `9ac0129d23475aad8f8467d0cf39eb8dbe0d0e5cda482d28cae5ea9f7f6f9d74` | 2026-10-25 |
| Custom rule 100100 in local_rules.xml, manager restarted | `images/18-custom-rule-100100-local_rules-and-manager-restart.png` | 47,645 B | `89fb2cf98d2cdc814d6aa69a66d24aa7a5a77081794b264e72cefdd92b6f66d7` | 2026-10-25 |
| T1136 simulation, useradd suspicious_user2 | `images/19-useradd-suspicious-user2-terminal.png` | 16,027 B | `5b3b2f6c820771fc10cd395b63e63ed5698f52194b0fc6a7013852c1189e4c38` | 2026-10-25 |
| Rule 100100 firing vs 5902 | `images/20-custom-rule-100100-fires-vs-5902.png` | 165,637 B | `2d9f80ec6fdfe89a6e7a10245b3dc8cc3baa4cceccadc818b5c95f20f3da0c65` | 2026-10-25 |
| SCA, CIS Ubuntu 24.04 benchmark, score 50% | `images/21-sca-cis-ubuntu-2404-score-50.png` | 192,782 B | `c02354162d5efaf433f6d2ffc62c113e7afdccde35e607b9acc186d3312c727c` | 2026-10-25 |
| Vulnerability Detection dashboard | `images/22-vulnerability-detection-dashboard.png` | 151,675 B | `19c433f1b6b30807c2df982c2df1f47e6f1408fef73bdd5874dae5d5623fe960` | 2026-10-25 |
| Pie visualization config, alerts by agent | `images/23-viz-alerts-by-agent-pie-config.png` | 122,102 B | `6cdfc7c90e5b1c9e5108e296dfecf1d3177224dfb8aa6dbccc72a3f5e62c37fb` | 2026-10-25 |
| Three saved SOC visualizations | `images/24-visualizations-list-three-saved.png` | 65,161 B | `4ac99dc64a40afacf9b19a79ecf346a6be06996e9ae1a31b2753bd5567c3bf5e` | 2026-10-25 |

---

## 5. Build steps

### Step 1 — Prepare the lab network and VMs

**Goal:** a repeatable base so that every VM starts from the same clean state.

1. Created one Ubuntu Server 24.04 VM called `soc-ubuntu-base`, installed with OpenSSH, and extended the root volume to use the whole disk:

   ```bash
   sudo lvextend -r -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
   ```

2. Took a snapshot named `Base-clean` of that VM [img 03], then made **Full Clones** for each lab machine with a new MAC address for every network card.
3. Created the host-only network `192.168.60.0/24` with the host at `192.168.60.1` and DHCP turned off [img 04].
4. On each clone, set a static IP on the host-only adapter (`enp0s8`) through `/etc/netplan/60-lab.yaml` and applied it with `sudo netplan apply`. The manager got `192.168.60.10/24` [img 02]. Because clones share the identity of their parent, I also regenerated `machine-id` and the SSH host keys on every clone.
5. Used SSH from the host (PowerShell) instead of the VM console, because the VirtualBox console has no clipboard.

**Result:** manager and endpoint reachable from the host on the lab network, each with its own identity.

### Step 2 — Install Wazuh (all-in-one) on `soc-wazuh-mgr`

**Goal:** manager, indexer and dashboard on one VM.

I ran the installer inside `tmux` over SSH, so the install survives a dropped connection:

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a -i
```

`-a` installs all three components, `-i` ignores the OS and hardware checks (Ubuntu 24.04 with 7 GB RAM triggers warnings on a lab machine). The installer printed the generated admin credentials at the end [img 05, password blacked out]. The credentials archive left on disk was deleted after the password was saved elsewhere.

The dashboard opened at `https://192.168.60.10` from the host browser after accepting the self-signed certificate [img 01]. At this point there were no agents.

**Result:** Wazuh 4.14.9 running, dashboard reachable from the host.

### Step 3 — Enroll the Linux endpoint

**Goal:** a reporting agent with the right name and manager address.

1. In the dashboard I used **Agents → Deploy new agent**, selected DEB amd64, entered the manager address `192.168.60.10` and the agent name `soc-linux-endpoint` [img 06]. The wizard generates the exact install command.
2. On `soc-linux-endpoint` I ran the generated command, which downloads the package and installs it with the manager address and agent name as environment variables [img 07]:

   ```bash
   wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.9-1_amd64.deb \
     && sudo WAZUH_MANAGER='192.168.60.10' WAZUH_AGENT_NAME='soc-linux-endpoint' dpkg -i ./wazuh-agent_4.14.9-1_amd64.deb
   ```

3. Started the service:

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable --now wazuh-agent
   systemctl status wazuh-agent
   ```

   The service reported `active (running)` [img 08].
4. The agent list shows `soc-linux-endpoint`, IP 192.168.60.21, Ubuntu 24.04.5 LTS, version 4.14.9, status **active** [img 09]. This is the agent inventory for the project.

**Result:** one Active agent.

### Step 4 — File Integrity Monitoring (FIM)

**Goal:** see a change to a watched file in near real time.

1. On the endpoint I created a test directory `/opt/fim-test` and added it to the `<syscheck>` section of the agent's `ossec.conf` as a realtime-monitored directory. Realtime mode reports a change as it happens instead of at the next scheduled scan.
2. Restarted the agent, then created `a.txt` in that folder and edited it.
3. The FIM dashboard shows the *added* and *modified* events for the agent [img 10]. In the event list, two alerts appear about 20 ms apart for `/opt/fim-test/a.txt`: rule **554** (file added to the system, level 5) and rule **550** (integrity checksum changed, level 7) [img 11].

**Result:** realtime FIM works end to end. Dashboard time was 19:20:34 (UTC+6).

### Step 5 — Security Configuration Assessment (SCA)

**Goal:** a baseline hardening score for the endpoint.

The agent ran the built-in policy **CIS Ubuntu Linux 24.04 LTS Benchmark v1.0.0** against the endpoint. Result: **119 passed, 118 failed, 42 not applicable, score 50%** out of 279 checks [img 21]. A default Ubuntu install is not hardened to the CIS benchmark, so a low first score is expected. It gives a starting point for hardening; I did not remediate anything in this project.

### Step 6 — Vulnerability Detection

**Goal:** see which installed packages have known CVEs.

Vulnerability Detection is enabled by default since Wazuh 4.8 through the `<vulnerability-detection>` block, so no configuration change was needed. After the agent's package inventory was collected, the dashboard listed **243 critical, 1,379 high, 534 medium, 48 low and 1,612 pending** findings (3,816 in total) for `soc-linux-endpoint` [img 22]. Almost all findings (3,809) belong to one package, the kernel image `linux-image-6.8.0-147-generic`, so a kernel update would very likely remove most of them. I did not test that.

### Step 7 — Simulated attack 1: T1110 Brute Force

**Goal:** trigger the SSH brute-force detections.

From `soc-wazuh-mgr` I sent repeated failed SSH logins as a user that does not exist (`baduser`) to the endpoint, using `sshpass` with a wrong password [img 12]:

```bash
sudo apt -y install sshpass
for i in $(seq 1 12); do sshpass -p wrongpass ssh -o StrictHostKeyChecking=no \
  -o PreferredAuthenticationTypes=password -o PubkeyAuthentication=no baduser@192.168.60.21; done
```

Each attempt returned `Permission denied`. Wazuh raised these rules (Threat Hunting → Events) [img 15, img 16]:

| Rule | Meaning | Level |
|---|---|---|
| 5710 | sshd: attempt to login using a non-existent user | 5 |
| 5503 | PAM: user login failed | 5 |
| 5551 | PAM: multiple failed logins in a small period of time | 10 |
| 5712 | sshd: brute force trying to get access to the system, non-existent user | 10 |

Rule 5712 did **not** appear on the dashboard after the first run. Its definition in `0095-sshd_rules.xml` shows what it needs: it is a correlation rule that wants `frequency="8"` matches of 5710 inside `timeframe="120"` seconds from the same source IP, with `ignore="60"` [img 13]. My first batch of 12 attempts took only about 6 seconds, and I did not establish why 5712 was missing afterwards. I ran a second, longer batch of 20 attempts. After that, counting lines in `alerts.json` gave **62 for rule 5710 and 2 for rule 5712** [img 17], so the brute-force rule fired after the second run.

**Result:** T1110 detected through 5710, 5503, 5551 and 5712.

### Step 8 — Simulated attack 2: T1136 Create Account

**Goal:** detect a new local account on the endpoint.

On `soc-linux-endpoint`:

```bash
sudo useradd -m suspicious_user
```

[img 14]. Wazuh raised **5902** (new user added to the system, level 8) and **5901** (new group added to the system, level 8), because `useradd` creates a user and its group [img 15]. The account name is a simulated indicator and was only used in this lab.

### Step 9 — Tune detection with a custom rule

**Goal:** raise the severity of local account creation and label it with the right MITRE technique, without producing duplicate alerts.

On the manager I added a child rule of 5902 to `/var/ossec/etc/rules/local_rules.xml`:

```xml
<rule id="100100" level="10">
  <if_sid>5902</if_sid>
  <description>Lab: new local user created on Linux host</description>
  <mitre>
    <id>T1136.001</id>
  </mitre>
</rule>
```

Custom rule IDs start at 100000, so 100100 does not collide with built-in rules. I restarted the manager so the rule would load, and checked the service (`active (running)`) [img 18]. Then I created two more test users, `suspicious_user2` [img 19] and `suspicious_user3`.

A query for `rule.id: (100100 or 5902)` shows the result [img 20]:

| Time (dashboard, UTC+6) | Agent | Rule | Level |
|---|---|---|---|
| 19:33:52.569 | soc-linux-endpoint | 5902 (before the custom rule existed) | 8 |
| 19:50:00.749 | soc-linux-endpoint | 100100 | 10 |
| 19:51:28.771 | soc-linux-endpoint | 100100 | 10 |

For the two new users only rule 100100 fires; 5902 is replaced by it instead of firing next to it. The same query also lists a 5902 on `soc-wazuh-mgr` at 18:37:28, which is earlier than the planned tests and is not part of them.

**Result:** custom rule works, level 10, tagged T1136.001, no duplicate alert.

### Step 10 — Dashboard visualizations

**Goal:** the pieces of a SOC overview.

In the dashboard's Visualize app I built three visualizations on the `wazuh-alerts-*` index and saved them [img 23, img 24]:

- **SOC - Alerts by agent** — pie chart, terms aggregation on `agent.name`, size 5 (slices for `soc-linux-endpoint` and `soc-wazuh-mgr`).
- **SOC - Alerts over time** — area chart over time.
- **SOC - Top rule IDs** — data table of the most frequent rule IDs.

Combining the three into a single **SOC Overview** dashboard is not done yet, so there is no dashboard screenshot.

---

## 6. Problems encountered

| Problem | What happened | Root cause | Fix | Lesson |
|---|---|---|---|---|
| SSH to the VM refused | `Connection refused` on the forwarded port | The NAT port-forward rule was not actually saved from the GUI | `VBoxManage controlvm "soc-ubuntu-base" natpf1 "ssh,tcp,127.0.0.1,2222,10.0.2.15,22"` | Check that the rule exists before blaming SSH |
| Second network card missing | No host-only adapter after boot | An adapter cannot be added while the VM is running, and it was not enabled before the first start | Powered off, enabled the host-only adapter, confirmed network 192.168.60.1/24 with DHCP off | Enable both adapters before first boot |
| Clone kept its parent's identity | Same MAC policy and identity after cloning | Clone dialog defaults were clicked through | Re-cloned with new MAC addresses, regenerated `machine-id` and SSH host keys | Check the clone options every time |
| Password rejected | First admin password refused by `wazuh-passwords-tool.sh` | The tool only accepts the symbols `.*+?-` (8–64 characters with upper, lower, digit and symbol) | Chose a password using an allowed symbol | Read the policy first |
| Credentials archive extracted to disk | `tar -o` extracted the archive instead of printing a file | Typo for `-O` | Read the file with `sudo cat`, then removed `wazuh-install-files` | Delete files that hold credentials once they are saved |
| Dashboard not loading in the built-in browser | Certificate error page | Self-signed certificate that the pane cannot click through | Used a normal browser and accepted the certificate | Lab certificates are self-signed by design |
| Brute-force rule 5712 not seen | Only 5710 and 5503 visible after the first run | Not established. 5712 needs 8 matches inside 120 s (see Step 7) | Ran 20 attempts, confirmed with `alerts.json` counts | Read the rule definition before testing, and confirm in the raw log |
| Windows endpoint dropped | No Windows evidence | Chose to save time and RAM | Documented a Linux-only scope | Do not claim what was not built |

## 7. Findings

| # | Observation | Evidence | Interpretation | Confidence |
|---|---|---|---|---|
| 1 | Agent `soc-linux-endpoint` (192.168.60.21, Ubuntu 24.04.5, 4.14.9) is Active | img 09 | Enrollment works over the host-only network | High |
| 2 | FIM raised rule 554 and rule 550 about 20 ms apart for `/opt/fim-test/a.txt` | img 10, img 11 | Realtime syscheck works | High |
| 3 | Failed logins for a non-existent user raised 5710 and 5503; 5551 (level 10) fired on the second run; 5712 fired twice in total | img 15, 16, 17 | The brute-force pattern is detected once the rule threshold is met | High |
| 4 | `useradd` produced 5902 and 5901 at level 8 | img 15 | New-account activity is visible | High |
| 5 | Custom rule 100100 (level 10) fired for `suspicious_user2` and `suspicious_user3`, with no 5902 next to it | img 20 | Tuning raised severity without duplicates | High |
| 6 | SCA score is 50% (119 passed, 118 failed) on a default Ubuntu install | img 21 | A default install is far from the CIS benchmark; this is the hardening baseline | High |
| 7 | 3,816 vulnerability findings, 3,809 of them in one kernel package | img 22 | Updating the kernel would likely clear most of them (not tested) | Medium |

## 8. MITRE ATT&CK

| Technique | Name | How it was tested | Detected by |
|---|---|---|---|
| T1110 | Brute Force | Failed SSH logins as `baduser` against 192.168.60.21 | 5710, 5503, 5551, 5712 |
| T1136.001 | Create Account: Local Account | `useradd` of `suspicious_user`, `suspicious_user2`, `suspicious_user3` | 5902, 5901, custom 100100 |

## 9. Indicators (all simulated)

| Type | Value | Context |
|---|---|---|
| Username | `baduser` | Non-existent user in the simulated brute force |
| Username | `suspicious_user`, `suspicious_user2`, `suspicious_user3` | Accounts created on purpose in the lab |
| IP (source) | 192.168.60.10 | `soc-wazuh-mgr`, attacker side of the simulation |
| IP (target) | 192.168.60.21 | `soc-linux-endpoint` |

## 10. Deliverables

- [x] Investigation notes (sections 5–7)
- [x] Screenshots in `images/` (24 files, hashes in `SHA256SUMS.txt`)
- [x] Agent inventory (img 09, one Linux agent)
- [x] Tuning log (custom rule 100100, Step 9)
- [ ] Dashboard screenshot (the three visualizations are saved, the SOC Overview dashboard is not assembled yet)
- [ ] Windows endpoint (dropped, not built)
- [ ] Final report in `report/`

## 11. Key learnings

1. A correlation rule such as 5712 depends on `frequency` and `timeframe`. Read the rule file before you design the test, and confirm results in the raw `alerts.json`, not only on one dashboard page.
2. A VM clone is not a new machine until it has its own MAC address, `machine-id` and SSH host keys.
3. Dashboard time (browser zone) and log time (UTC) differ by 6 hours here, so every timestamp in a report needs its zone written next to it.
4. A child rule with `if_sid` replaces the parent alert for matching events, which is the clean way to raise severity without double alerts.
5. Credentials do not belong in screenshots or leftover files. Black them out or do not keep the image.
6. Writing down what was **not** done (Windows endpoint, dashboard assembly) keeps the evidence honest.
