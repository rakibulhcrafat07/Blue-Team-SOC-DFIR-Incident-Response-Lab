<h1 align="center">Project 02 — Wazuh SIEM Deployment</h1>

<p align="center">
  <img alt="Status" src="https://img.shields.io/badge/status-in%20progress-yellow">
  <img alt="Wazuh" src="https://img.shields.io/badge/Wazuh-4.14.9-blue">
  <img alt="Track" src="https://img.shields.io/badge/track-A%20SOC%20%26%20SIEM-informational">
  <img alt="ATT&CK" src="https://img.shields.io/badge/ATT%26CK-T1110%20%7C%20T1136-red">
  <img alt="Endpoint" src="https://img.shields.io/badge/endpoint-Linux%20only-lightgrey">
</p>

<p align="center"><b>Case ID:</b> CB-02-001 · <b>Date:</b> 2026-10-25 · <b>Analyst:</b> Rafat</p>

> **Scope note.** The plan also had a Windows endpoint (`soc-win-endpoint`). I dropped it in this run to save time and RAM, so this project has **one Linux endpoint only** and there is no Windows evidence. All attacks are simulated, run only between my own lab VMs.

## Contents

1. [Summary](#1-summary)
2. [Lab architecture](#2-lab-architecture)
3. [What I did, in order](#3-what-i-did-in-order)
4. [Build steps](#4-build-steps) — host prep, base VM, clones, Wazuh, agent, FIM, SCA, vulnerabilities, attacks, tuning, dashboards
5. [Problems encountered](#5-problems-encountered)
6. [Findings](#6-findings)
7. [MITRE ATT&CK](#7-mitre-attck) and [Indicators](#8-indicators-all-simulated)
8. [Deliverables](#9-deliverables) and [Key learnings](#10-key-learnings)
9. [Evidence inventory](#11-evidence-inventory)

---

## 1. Summary

A working SIEM built from scratch on one laptop. A **Wazuh 4.14.9 all-in-one server** (manager, indexer, dashboard) runs in a VirtualBox VM on an isolated lab network. One Ubuntu endpoint is enrolled as an agent. On top of that I enabled and tested the Wazuh features a SOC analyst uses daily: **File Integrity Monitoring (FIM)**, **Security Configuration Assessment (SCA)**, **Vulnerability Detection**, detection of two simulated ATT&CK techniques (**T1110 Brute Force**, **T1136 Create Account**), **one custom detection rule**, and **three dashboard visualizations**.

Screenshots are referenced as `[img NN]` (the 24 main captures) and `[S NN]` (setup-phase captures) and live in [`images/`](images/). Click any thumbnail to open it full size.

## 2. Lab architecture

```mermaid
flowchart LR
    H["Windows laptop (host)<br/>VirtualBox<br/>192.168.60.1"]
    subgraph LAB["Host-only network 192.168.60.0/24 (DHCP off)"]
        M["soc-wazuh-mgr<br/>192.168.60.10<br/>Wazuh manager + indexer + dashboard<br/>Ubuntu 24.04 LTS"]
        E["soc-linux-endpoint<br/>192.168.60.21<br/>Wazuh agent 4.14.9<br/>Ubuntu 24.04.5"]
        W["soc-win-endpoint<br/>planned, not built"]
    end
    H --- M
    H --- E
    E -- "agent traffic" --> M
    M -. "simulated SSH attacks" .-> E
    style W stroke-dasharray: 5 5,color:#888
```

Every VM has two adapters: **adapter 1 = NAT** (internet for package downloads), **adapter 2 = host-only** (the lab network, no gateway, so lab traffic never leaves the laptop).

| Item | Value |
|---|---|
| Host | Windows laptop "Rafat-PC", Ryzen 7 3750H, 24 GB RAM, VirtualBox 7.x [S01] |
| Manager VM | `soc-wazuh-mgr`, Ubuntu Server 24.04 LTS, 7 GB RAM (7168 MB), 3 vCPU, 192.168.60.10 |
| Endpoint VM | `soc-linux-endpoint`, Ubuntu Server 24.04.5 LTS, 2 GB RAM, 1 vCPU, 192.168.60.21 |
| Base VM (template) | `soc-ubuntu-base`, 2 GB RAM, 2 vCPU, 50 GB disk, snapshot `Base-clean` |
| Tool versions | Wazuh 4.14.9 (`wazuh-install.sh -a`), wazuh-agent 4.14.9 |
| Time zone | VMs and logs use **UTC**. The dashboard shows browser time (**UTC+6**), so dashboard times are 6 hours ahead of log times. |

## 3. What I did, in order

| # | Phase | Result | Evidence |
|---|---|---|---|
| 1 | Check host resources, download Ubuntu Server ISO | 24 GB RAM, ~16 GB usable for VMs | S01 |
| 2 | Create the base VM and install Ubuntu Server | `soc-ubuntu-base` running | S04–S11 |
| 3 | SSH into the VM from the host (NAT port forward 2222 → 22) | PowerShell SSH works | S12–S15 |
| 4 | Host-only lab network, second adapter, `Base-clean` snapshot | 192.168.60.0/24 ready | img 04, S16, img 03 |
| 5 | Full-clone the base into manager and endpoint, give each a new identity and IP | Two VMs on the lab network | S17–S23, S25, img 02 |
| 6 | Install Wazuh 4.14.9 all-in-one | Dashboard at https://192.168.60.10 | S24, img 05, img 01 |
| 7 | Enroll the Linux agent | Agent Active | img 06–09 |
| 8 | FIM, SCA, Vulnerability Detection | Alerts, 50% SCA score, 3,816 CVE findings | S26–S27, img 10–11, 21–22 |
| 9 | Simulate T1110 and T1136 | Rules 5710/5503/5551/5712, 5902/5901 | img 12–17, 19, S28–S29 |
| 10 | Custom rule 100100 | Level 10, no duplicate alert | img 18, 20 |
| 11 | Dashboard visualizations | 3 saved; SOC Overview dashboard still to do | S30–S31, img 23–24 |

---

## 4. Build steps

### Step 1 — Prepare the host

Checked the host resources: AMD Ryzen 7 3750H, **24 GB RAM** (about 23.4 GB usable), 1.14 TB disk [S01]. With a ceiling of roughly 70% of RAM for VMs (about 16 GB), a three-VM plan fits: manager 7 GB, endpoint 2 GB, plus the 2 GB base VM that is only powered on to build clones. Downloaded the **Ubuntu Server 24.04.3 LTS live ISO** (the ISO used for the base VM is listed in the VM summary [S06]).

<p>
<img src="images/S01-host-specs.png" width="380" title="S01 · Host: Ryzen 7 3750H, 24 GB RAM">
</p>


### Step 2 — Create the base VM and install Ubuntu Server

**Goal:** one clean template that every lab VM is cloned from.

In VirtualBox I created `soc-ubuntu-base` with the unattended install **skipped**, so I picked the settings myself: Ubuntu (64-bit), **2048 MB RAM, 2 vCPU**, EFI off, **50 GB dynamically allocated disk** [S04, S05, S06]. Then ran the Ubuntu Server installer: the first NIC (`enp0s3`) took 10.0.2.15 by DHCP from VirtualBox NAT [S07], the guided storage layout used **LVM** (the installer only allocates half of the volume group to the root logical volume, about 24 GB) [S08], profile name/server name `soc-ubuntu-base` with user `soc` [S09]. The installer finished [S10] and the first login worked on the second try (the first password attempt was mistyped) [S11].


<p>
<img src="images/S04-vbox-new-vm-disk-50gb.png" width="380" title="S04 · New VM: 50 GB disk">
<img src="images/S05-vbox-new-vm-hardware-2gb-2cpu.png" width="380" title="S05 · New VM: 2048 MB RAM, 2 CPUs">
<img src="images/S06-vbox-new-vm-summary.png" width="380" title="S06 · New VM: summary">
<img src="images/S07-ubuntu-installer-network-dhcp.png" width="380" title="S07 · Installer: network (enp0s3 DHCP)">
<img src="images/S08-ubuntu-installer-storage-lvm.png" width="380" title="S08 · Installer: storage (LVM)">
<img src="images/S09-ubuntu-installer-profile.png" width="380" title="S09 · Installer: profile (password masked)">
<img src="images/S10-ubuntu-installing-system.png" width="380" title="S10 · Installer: installing system">
<img src="images/S11-first-login-base-vm.png" width="380" title="S11 · First login on the base VM">
</p>


After the first login I extended the root logical volume so the VM could use the full 50 GB disk. My first attempt had a typo in the volume name (`ubbuntu-lv`), the command was corrected to `ubuntu-lv`:

```bash
sudo lvextend -r -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
df -h /
```

`df -h /` then showed about 48 GB on `/` [S15].

### Step 3 — SSH access from the host

**Goal:** paste commands from the host instead of typing in the VirtualBox console, which has no working clipboard.

The VM was on NAT only, so I used a VirtualBox **port-forward rule**: host `127.0.0.1:2222` → guest `10.0.2.15:22`. The sshd service was running and listening on port 22 inside the VM, but SSH from the host got `Connection refused` [S12], because the rule had not been saved in the GUI [S13]. I added the rule again [S14] (the command-line form is in section 5) and SSH from PowerShell worked: `ssh -p 2222 soc@127.0.0.1`. When I finished the base setup I powered the VM off cleanly [S15].


<p>
<img src="images/S12-ssh-port-22-listening-connection-refused.png" width="380" title="S12 · sshd listening on :22, host gets Connection refused">
<img src="images/S13-vbox-port-forwarding-empty.png" width="380" title="S13 · Port forwarding rules: empty">
<img src="images/S14-vbox-port-forwarding-ssh-2222-to-22.png" width="380" title="S14 · Rule added: ssh TCP 127.0.0.1:2222 → 10.0.2.15:22">
<img src="images/S15-base-vm-poweroff-df-48g.png" width="380" title="S15 · df -h after lvextend, then poweroff">
</p>


### Step 4 — Host-only lab network, second adapter, base snapshot

**Goal:** a private network for the lab machines.

Created the VirtualBox host-only network `192.168.60.0/24` with the host at `192.168.60.1` and **DHCP turned off** [img 04]. Enabled **adapter 2 = Host-only** on the base VM (adapter 1 stays NAT). Inside the VM the new interface `enp0s8` existed but was `DOWN` with no address [S16]. I gave it a static address through netplan (`/etc/netplan/60-lab.yaml`, `enp0s8`, `192.168.60.5/24`, `dhcp4: false`, no gateway) and applied it with `sudo netplan apply`. Finally I took the snapshot **`Base-clean`** of the powered-off base VM [img 03].


<p>
<img src="images/04-host-only-network-before-apply.png" width="380" title="04 · Host-only network 192.168.60.1/24, DHCP off">
<img src="images/S16-ip-a-enp0s8-down.png" width="380" title="S16 · ip a: enp0s8 present but DOWN">
<img src="images/03-virtualbox-base-snapshot.png" width="380" title="03 · Base VM with snapshot Base-clean">
</p>


### Step 5 — Clone the base into the lab VMs

**Goal:** two independent VMs from one template, each with its own identity.

From the `Base-clean` snapshot I made two **Full Clones** [S17, S18]: `soc-wazuh-mgr` (set to **3 vCPU, 7168 MB RAM** [S19, S20]) and `soc-linux-endpoint` (1 vCPU, 2048 MB). The clone dialog's defaults (snapshots: *Everything*, MAC policy: *include only NAT network adapter MAC addresses*) were accepted, and I checked afterwards that the clone's network cards had new MAC addresses [S21].

A clone still carries its parent's hostname, machine-id, SSH host keys and IP, so on each clone I changed them. For the manager [S23]:

```bash
sudo hostnamectl set-hostname soc-wazuh-mgr
sudo sed -i 's/soc-ubuntu-base/soc-wazuh-mgr/g' /etc/hosts
sudo rm -f /etc/machine-id /var/lib/dbus/machine-id
sudo systemd-machine-id-setup
sudo rm -f /etc/ssh/ssh_host_*
sudo dpkg-reconfigure openssh-server
sudo sed -i 's/192.168.60.5/192.168.60.10/' /etc/netplan/60-lab.yaml
sudo reboot
```

The endpoint got the same treatment with the name `soc-linux-endpoint` and the address `192.168.60.21` [S25]. After the reboot, SSH to the new address worked from PowerShell; the host asked to trust the new key, because the key had just been regenerated [S22]. The manager's hostname and address are visible on the VM itself [img 02].


<p>
<img src="images/S17-vbox-clone-dialog-base-clean.png" width="380" title="S17 · Clone dialog on the Base-clean snapshot">
<img src="images/S18-vbox-clone-soc-wazuh-mgr-full-clone.png" width="380" title="S18 · Clone soc-wazuh-mgr: Full Clone">
<img src="images/S19-wazuh-mgr-3-cpu.png" width="380" title="S19 · soc-wazuh-mgr: 3 CPUs">
<img src="images/S20-wazuh-mgr-7168mb-ram.png" width="380" title="S20 · soc-wazuh-mgr: 7168 MB RAM">
<img src="images/S21-wazuh-mgr-adapter2-host-only.png" width="380" title="S21 · soc-wazuh-mgr: adapter 2 host-only, new MAC">
<img src="images/S22-first-ssh-to-clone.png" width="380" title="S22 · First SSH to the clone at 192.168.60.10">
<img src="images/S23-clone-identity-hostname-machine-id-ssh-keys-ip.png" width="380" title="S23 · Manager: new hostname, machine-id, SSH keys, IP">
<img src="images/S25-endpoint-clone-identity-and-ssh.png" width="380" title="S25 · Endpoint: same steps, IP 192.168.60.21">
<img src="images/02-wazuh-mgr-hostname-ip.png" width="380" title="02 · Manager hostname and 192.168.60.10/24">
</p>


**Result:** `soc-wazuh-mgr` (192.168.60.10) and `soc-linux-endpoint` (192.168.60.21) reachable from the host, each with its own identity.

### Step 6 — Install Wazuh (all-in-one) on `soc-wazuh-mgr`

**Goal:** manager, indexer and dashboard on one VM.

I ran the installer inside `tmux` over SSH so it survives a dropped connection:

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

`-a` installs all three components. The installation assistant started with Wazuh version **4.14.9** and logged to `/var/log/wazuh-install.log` [S24]. The install ran for about 16 minutes (about 12:26 to 12:42 UTC) and finished by printing the generated admin credentials [img 05, password blacked out]. I saved the credentials and then deleted the credentials archive that was left on disk. The dashboard opened at `https://192.168.60.10` after accepting the self-signed certificate [img 01]; at this point there were no agents.


<p>
<img src="images/S24-wazuh-install-assistant-started-4.14.9.png" width="380" title="S24 · wazuh-install.sh -a started, Wazuh 4.14.9">
<img src="images/05-wazuh-install-finished-password-redacted.png" width="380" title="05 · Installer finished (password blacked out)">
<img src="images/01-wazuh-dashboard-home.png" width="380" title="01 · Wazuh dashboard home, no agents yet">
</p>


### Step 7 — Enroll the Linux endpoint

**Goal:** a reporting agent with the right name and manager address.

In the dashboard I used **Agents → Deploy new agent**, selected DEB amd64, entered the manager address `192.168.60.10` and the agent name `soc-linux-endpoint` [img 06]. The wizard generates the install command, which I ran on the endpoint [img 07]:

```bash
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.9-1_amd64.deb \
  && sudo WAZUH_MANAGER='192.168.60.10' WAZUH_AGENT_NAME='soc-linux-endpoint' dpkg -i ./wazuh-agent_4.14.9-1_amd64.deb
sudo systemctl daemon-reload
sudo systemctl enable --now wazuh-agent
systemctl status wazuh-agent
```

The service was `active (running)` [img 08] and the agent list shows `soc-linux-endpoint`, 192.168.60.21, Ubuntu 24.04.5 LTS, version 4.14.9, status **active** [img 09]. This is the agent inventory for the project.


<p>
<img src="images/06-deploy-agent-wizard-deb-amd64.png" width="380" title="06 · Deploy new agent wizard (DEB amd64)">
<img src="images/07-linux-agent-install-4.14.9.png" width="380" title="07 · wazuh-agent 4.14.9 installed">
<img src="images/08-linux-agent-service-running.png" width="380" title="08 · wazuh-agent service running">
<img src="images/09-agents-list-linux-active.png" width="380" title="09 · Agents list: soc-linux-endpoint Active">
</p>


### Step 8 — File Integrity Monitoring (FIM)

**Goal:** see a change to a watched file in near real time.

On the endpoint I created `/opt/fim-test` and added it to the `<syscheck>` section of the agent config `/var/ossec/etc/ossec.conf` with `sed`, right after the default directories [S26]:

```xml
<directories realtime="yes" check_all="yes">/opt/fim-test</directories>
```

My first `grep` on `ossec.conf` failed with `Permission denied` because the file is root-only; with `sudo` it showed the new line 115 [S27]. After restarting the agent (`active (running)` since 13:20:08 UTC) I created and edited the test file:

```bash
sudo touch /opt/fim-test/a.txt
echo "changed" | sudo tee /opt/fim-test/a.txt
```

The FIM dashboard shows *added* and *modified* events [img 10]. In the events list, two alerts appear about 20 ms apart for `/opt/fim-test/a.txt`: rule **554** (file added, level 5) and rule **550** (integrity checksum changed, level 7) [img 11].


<p>
<img src="images/S26-fim-test-dir-ossec-conf-sed.png" width="380" title="S26 · mkdir fim-test + sed adds the directory to ossec.conf">
<img src="images/S27-fim-config-agent-running-touch-a-txt.png" width="380" title="S27 · grep shows the config line; agent running; touch/echo a.txt">
<img src="images/10-fim-dashboard-added-modified.png" width="380" title="10 · FIM dashboard: added and modified">
<img src="images/11-fim-events-a-txt-rule554-550.png" width="380" title="11 · FIM events: rules 554 and 550 for a.txt">
</p>


**Result:** realtime FIM works end to end. Dashboard time was 19:20:34 (UTC+6).

### Step 9 — Security Configuration Assessment (SCA)

The agent ran the built-in policy **CIS Ubuntu Linux 24.04 LTS Benchmark v1.0.0**. Result: **119 passed, 118 failed, 42 not applicable, score 50%** out of 279 checks [img 21]. A default Ubuntu install is not hardened to the CIS benchmark, so a low first score is expected. It is the baseline for later hardening; I did not remediate anything in this project.

### Step 10 — Vulnerability Detection

Enabled by default since Wazuh 4.8 (the `<vulnerability-detection>` block), so no configuration change was needed. After the agent's package inventory was collected, the dashboard listed **243 critical, 1,379 high, 534 medium, 48 low and 1,612 pending** findings (3,816 in total) for `soc-linux-endpoint` [img 22]. Almost all findings (3,809) belong to one package, the kernel image `linux-image-6.8.0-147-generic`, so a kernel update would very likely remove most of them. I did not test that.


<p>
<img src="images/21-sca-cis-ubuntu-2404-score-50.png" width="380" title="21 · SCA: CIS Ubuntu 24.04, score 50%">
<img src="images/22-vulnerability-detection-dashboard.png" width="380" title="22 · Vulnerability Detection dashboard">
</p>


### Step 11 — Simulated attack 1: T1110 Brute Force

**Goal:** trigger the SSH brute-force detections.

From `soc-wazuh-mgr` I sent repeated failed SSH logins as a user that does not exist (`baduser`) to the endpoint, using `sshpass` with a wrong password [img 12]:

```bash
sudo apt -y install sshpass
for i in $(seq 1 12); do sshpass -p wrongpass ssh -o StrictHostKeyChecking=no \
  -o PreferredAuthenticationTypes=password -o PubkeyAuthentication=no baduser@192.168.60.21; done
```

Every attempt returned `Permission denied`. Wazuh raised these rules (Threat Hunting → Events) [img 15, img 16]:

| Rule | Meaning | Level |
|---|---|---|
| 5710 | sshd: attempt to login using a non-existent user | 5 |
| 5503 | PAM: user login failed | 5 |
| 5551 | PAM: multiple failed logins in a small period of time | 10 |
| 5712 | sshd: brute force trying to get access to the system, non-existent user | 10 |

Rule 5712 did **not** appear on the dashboard after the first run. Its definition in `0095-sshd_rules.xml` shows what it needs: a correlation rule that wants `frequency="8"` matches of 5710 inside `timeframe="120"` seconds from the same source IP, with `ignore="60"` [img 13]. My first batch of 12 attempts took only about 6 seconds, and I did not establish why 5712 was missing afterwards. I ran a second, longer batch of **20 attempts** [S28]. After that, counting lines in `alerts.json` gave **62 for rule 5710 and 2 for rule 5712** [img 17], so the brute-force rule fired after the second run.


<p>
<img src="images/12-t1110-bruteforce-terminal.png" width="380" title="12 · First run: 12 failed SSH logins">
<img src="images/13-rule-5712-threshold-frequency8-timeframe120.png" width="380" title="13 · Rule 5712: frequency 8, timeframe 120">
<img src="images/S28-t1110-second-run-20-attempts.png" width="380" title="S28 · Second run: 20 attempts">
<img src="images/16-t1110-second-run-5710-5503-5551.png" width="380" title="16 · Second run: 5710, 5503, 5551">
<img src="images/17-alerts-json-count-5710-62-5712-2.png" width="380" title="17 · alerts.json counts: 5710 = 62, 5712 = 2">
</p>


**Result:** T1110 detected through 5710, 5503, 5551 and 5712.

### Step 12 — Simulated attack 2: T1136 Create Account

**Goal:** detect a new local account on the endpoint.

On `soc-linux-endpoint`:

```bash
sudo useradd -m suspicious_user
```

[img 14]. Wazuh raised **5902** (new user added, level 8) and **5901** (new group added, level 8), because `useradd` creates a user and its group [img 15]. The account name is a simulated indicator and was only used in this lab.


<p>
<img src="images/14-t1136-useradd-suspicious-user-terminal.png" width="380" title="14 · useradd suspicious_user">
<img src="images/15-threat-hunting-events-5710-5503-5902-5901.png" width="380" title="15 · Events: 5710, 5503, 5902, 5901">
</p>


### Step 13 — Tune detection with a custom rule

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

Custom rule IDs start at 100000, so 100100 does not collide with built-in rules. I restarted `wazuh-manager` (status `active (running)` since 13:48:56 UTC) [img 18], then created two more test users, `suspicious_user2` [img 19] and `suspicious_user3` [S29]. A query for `rule.id: (100100 or 5902)` shows the result [img 20]:

| Time (dashboard, UTC+6) | Agent | Rule | Level |
|---|---|---|---|
| 19:33:52.569 | soc-linux-endpoint | 5902 (before the custom rule existed) | 8 |
| 19:50:00.749 | soc-linux-endpoint | 100100 | 10 |
| 19:51:28.771 | soc-linux-endpoint | 100100 | 10 |

For the two new users only rule 100100 fires; it replaces 5902 instead of firing next to it. The same query also lists a 5902 on `soc-wazuh-mgr` at 18:37:28, which is earlier than the planned tests and not part of them.


<p>
<img src="images/18-custom-rule-100100-local_rules-and-manager-restart.png" width="380" title="18 · local_rules.xml rule 100100, manager restarted">
<img src="images/19-useradd-suspicious-user2-terminal.png" width="380" title="19 · useradd suspicious_user2">
<img src="images/S29-t1136-useradd-suspicious-user3.png" width="380" title="S29 · useradd suspicious_user3">
<img src="images/20-custom-rule-100100-fires-vs-5902.png" width="380" title="20 · Rule 100100 fires; 5902 does not duplicate">
</p>


**Result:** custom rule works, level 10, tagged T1136.001, no duplicate alert.

### Step 14 — Dashboard visualizations

In the dashboard's **Visualize** app I created visualizations on the `wazuh-alerts-*` index [S30, S31] and saved three [img 23, img 24]:

- **SOC - Alerts by agent** — pie chart, terms aggregation on `agent.name`, size 5 (slices for `soc-linux-endpoint` and `soc-wazuh-mgr`).
- **SOC - Alerts over time** — area chart over time (date histogram on `timestamp`).
- **SOC - Top rule IDs** — data table of the most frequent rule IDs.


<p>
<img src="images/S30-new-visualization-types.png" width="380" title="S30 · New visualization: choose type">
<img src="images/S31-area-visualization-built.png" width="380" title="S31 · Area chart: date histogram on timestamp">
<img src="images/23-viz-alerts-by-agent-pie-config.png" width="380" title="23 · Pie chart: alerts by agent">
<img src="images/24-visualizations-list-three-saved.png" width="380" title="24 · Three saved visualizations">
</p>


Combining the three into a single **SOC Overview** dashboard is **not done yet**, so there is no dashboard screenshot.

---

## 5. Problems encountered

| Problem | What happened | Root cause | Fix | Lesson |
|---|---|---|---|---|
| SSH to the VM refused | `Connection refused` on the forwarded port while sshd was listening [S12] | The NAT port-forward rule was not saved from the GUI [S13] | Added the rule again [S14]; command-line form: `VBoxManage controlvm "soc-ubuntu-base" natpf1 "ssh,tcp,127.0.0.1,2222,10.0.2.15,22"` | Check the rule exists before blaming SSH |
| `lvextend` failed | `Logical volume ubuntu-lv not found` | Typo in the volume name (`ubbuntu-lv`) | Re-ran with `/dev/ubuntu-vg/ubuntu-lv` | Read the name from `lvs` before typing |
| Second network card missing at first start | Adapter 2 (host-only) was not enabled when the VM first started | An adapter cannot be added while the VM is running | Powered off, enabled the host-only adapter; `enp0s8` then appeared but was `DOWN` with no address [S16], so I set the static IP in netplan | Enable both adapters before first boot |
| Cloning with default options | Clone dialog defaults accepted without reading them | Clicked through | Checked that the clone's MAC addresses are new [S21] and regenerated hostname, `machine-id` and SSH host keys [S23, S25] | Check the clone options every time |
| Password rejected | First admin password refused by `wazuh-passwords-tool.sh` | The tool only accepts the symbols `.*+?-` (8–64 characters with upper, lower, digit and symbol) | Chose a password using an allowed symbol | Read the policy first |
| Credentials archive extracted to disk | `tar -o` extracted the archive instead of printing a file | Typo for `-O` | Read the file with `sudo cat`, then removed `wazuh-install-files` | Delete files that hold credentials once they are saved |
| Dashboard not loading in the built-in browser | Certificate error page | Self-signed certificate that the pane cannot click through | Used a normal browser and accepted the certificate | Lab certificates are self-signed by design |
| `grep` on `ossec.conf` denied | `Permission denied` [S26] | File is root-only | Re-ran with `sudo` [S27] | Use `sudo` for files under `/var/ossec` |
| Brute-force rule 5712 not seen | Only 5710 and 5503 visible after the first run | Not established; 5712 needs 8 matches inside 120 s (see Step 11) | Ran 20 attempts, confirmed with `alerts.json` counts | Read the rule definition first and confirm in the raw log |
| Windows endpoint dropped | No Windows evidence | Chose to save time and RAM | Documented a Linux-only scope | Do not claim what was not built |

## 6. Findings

| # | Observation | Evidence | Interpretation | Confidence |
|---|---|---|---|---|
| 1 | Agent `soc-linux-endpoint` (192.168.60.21, Ubuntu 24.04.5, 4.14.9) is Active | img 09 | Enrollment works over the host-only network | High |
| 2 | FIM raised rule 554 and rule 550 about 20 ms apart for `/opt/fim-test/a.txt` | img 10, 11, S27 | Realtime syscheck works | High |
| 3 | Failed logins for a non-existent user raised 5710 and 5503; 5551 (level 10) fired on the second run; 5712 fired twice in total | img 15, 16, 17 | The brute-force pattern is detected once the rule threshold is met | High |
| 4 | `useradd` produced 5902 and 5901 at level 8 | img 15 | New-account activity is visible | High |
| 5 | Custom rule 100100 (level 10) fired for `suspicious_user2` and `suspicious_user3`, with no 5902 next to it | img 20 | Tuning raised severity without duplicates | High |
| 6 | SCA score is 50% (119 passed, 118 failed) on a default Ubuntu install | img 21 | A default install is far from the CIS benchmark; this is the hardening baseline | High |
| 7 | 3,816 vulnerability findings, 3,809 of them in one kernel package | img 22 | Updating the kernel would likely clear most of them (not tested) | Medium |

## 7. MITRE ATT&CK

| Technique | Name | How it was tested | Detected by |
|---|---|---|---|
| T1110 | Brute Force | Failed SSH logins as `baduser` against 192.168.60.21 | 5710, 5503, 5551, 5712 |
| T1136.001 | Create Account: Local Account | `useradd` of `suspicious_user`, `suspicious_user2`, `suspicious_user3` | 5902, 5901, custom 100100 |

## 8. Indicators (all simulated)

| Type | Value | Context |
|---|---|---|
| Username | `baduser` | Non-existent user in the simulated brute force |
| Username | `suspicious_user`, `suspicious_user2`, `suspicious_user3` | Accounts created on purpose in the lab |
| IP (source) | 192.168.60.10 | `soc-wazuh-mgr`, attacker side of the simulation |
| IP (target) | 192.168.60.21 | `soc-linux-endpoint` |

## 9. Deliverables

- [x] Investigation notes (sections 3–6)
- [x] Screenshots in `images/` (24 main captures + 29 setup captures, hashes in `SHA256SUMS.txt`)
- [x] Agent inventory (img 09, one Linux agent)
- [x] Tuning log (custom rule 100100, Step 13)
- [ ] Dashboard screenshot (three visualizations saved, the SOC Overview dashboard is not assembled yet)
- [ ] Windows endpoint (dropped, not built)
- [ ] Final report in `report/`

## 10. Key learnings

1. A correlation rule such as 5712 depends on `frequency` and `timeframe`. Read the rule file before you design the test, and confirm results in the raw `alerts.json`, not only on one dashboard page.
2. A VM clone is not a new machine until it has its own MAC address, hostname, `machine-id` and SSH host keys.
3. A template VM plus Full Clones turns a multi-VM lab into minutes of work, and the `Base-clean` snapshot is the way back if a build goes wrong.
4. Dashboard time (browser zone) and log time (UTC) differ by 6 hours here, so every timestamp in a report needs its zone written next to it.
5. A child rule with `if_sid` replaces the parent alert for matching events, which is the clean way to raise severity without double alerts.
6. Credentials do not belong in screenshots or leftover files. Black them out or do not keep the image. Several setup screenshots that showed passwords were left out of this repository on purpose.
7. Writing down what was **not** done (Windows endpoint, dashboard assembly) keeps the evidence honest.

## 11. Evidence inventory

`images/SHA256SUMS.txt` holds the same list for `sha256sum -c SHA256SUMS.txt`. The installer screenshot (img 05) has the admin password blacked out. Screenshots that showed passwords were not kept and are not published.

### Main captures (img 01–24)

| Ref | File | Size | SHA-256 |
|---|---|---|---|
| `01` | `images/01-wazuh-dashboard-home.png` | 181,303 B | `45e38b893e50ae117961e37b903ae6c6f32948c5592ccf6bbaaebe4e7ee0bf1b` |
| `02` | `images/02-wazuh-mgr-hostname-ip.png` | 35,803 B | `52b51b3cba3c14252f87f374eda9ebc7eb7c5ca4408d6749634cdac8dd286d26` |
| `03` | `images/03-virtualbox-base-snapshot.png` | 145,280 B | `a6c81013fff2d21dcb4180c4160d8256597afaeebab236e675b6a2190fb371ee` |
| `04` | `images/04-host-only-network-before-apply.png` | 62,500 B | `3c329fbb549ca8c135d21c11ef9d50137ed7209ce7d640369ea2bb1cd40f2194` |
| `05` | `images/05-wazuh-install-finished-password-redacted.png` | 41,416 B | `76c04038f408c52b6c23a599258fa99da7e6bbc71c829cec4a976e10a9982411` |
| `06` | `images/06-deploy-agent-wizard-deb-amd64.png` | 39,413 B | `d2b519775bb4bd8c474c269529ef38ea6bdf972fc145635ebb269add73b1141f` |
| `07` | `images/07-linux-agent-install-4.14.9.png` | 98,019 B | `90fdff5f54f5bc7f993ea7e2fd9755b1f00dc781f34f88f6ff8373b1d953fae1` |
| `08` | `images/08-linux-agent-service-running.png` | 50,280 B | `d05bd1d2215c47a3bb8c36c0ba79fbe945f11c99d9fb463e34d69db56634b8b4` |
| `09` | `images/09-agents-list-linux-active.png` | 107,380 B | `93e20236b9aa9fa3fc5564f477fe9750db87803639daddaa9624d2f167a72bba` |
| `10` | `images/10-fim-dashboard-added-modified.png` | 96,550 B | `7664e6bd36f410ba1e8472b1950a7f35a6bb2895280e118da382004909cf8742` |
| `11` | `images/11-fim-events-a-txt-rule554-550.png` | 152,283 B | `a0c982cd8e1e2d8c7b78546143799c39ae1296966cc2eff9c4029ca9bd97cd8c` |
| `12` | `images/12-t1110-bruteforce-terminal.png` | 138,384 B | `da1583c3664ccb2e1ac14a33153fc9bb2f814f0c52111bd1c9f585748d12f58a` |
| `13` | `images/13-rule-5712-threshold-frequency8-timeframe120.png` | 42,010 B | `c5f7971b3ebbe6977c729e4b9b0ff3b671fb41ba80d2cb25a6d0a24b8b47eeed` |
| `14` | `images/14-t1136-useradd-suspicious-user-terminal.png` | 8,201 B | `891613a07f4bf4ee8277f71f4583890c793c52dd90146b6c2f04865a40151646` |
| `15` | `images/15-threat-hunting-events-5710-5503-5902-5901.png` | 311,607 B | `ea5de72ceae16fe002a9216bb9700b2019108fafed0bfeb79b36801c46ee7e62` |
| `16` | `images/16-t1110-second-run-5710-5503-5551.png` | 297,110 B | `8550d76324d78bcb51eb9878f4ce8ce3ed50bda1e042f0bbe1371993cf03a155` |
| `17` | `images/17-alerts-json-count-5710-62-5712-2.png` | 80,538 B | `9ac0129d23475aad8f8467d0cf39eb8dbe0d0e5cda482d28cae5ea9f7f6f9d74` |
| `18` | `images/18-custom-rule-100100-local_rules-and-manager-restart.png` | 47,645 B | `89fb2cf98d2cdc814d6aa69a66d24aa7a5a77081794b264e72cefdd92b6f66d7` |
| `19` | `images/19-useradd-suspicious-user2-terminal.png` | 16,027 B | `5b3b2f6c820771fc10cd395b63e63ed5698f52194b0fc6a7013852c1189e4c38` |
| `20` | `images/20-custom-rule-100100-fires-vs-5902.png` | 165,637 B | `2d9f80ec6fdfe89a6e7a10245b3dc8cc3baa4cceccadc818b5c95f20f3da0c65` |
| `21` | `images/21-sca-cis-ubuntu-2404-score-50.png` | 192,782 B | `c02354162d5efaf433f6d2ffc62c113e7afdccde35e607b9acc186d3312c727c` |
| `22` | `images/22-vulnerability-detection-dashboard.png` | 151,675 B | `19c433f1b6b30807c2df982c2df1f47e6f1408fef73bdd5874dae5d5623fe960` |
| `23` | `images/23-viz-alerts-by-agent-pie-config.png` | 122,102 B | `6cdfc7c90e5b1c9e5108e296dfecf1d3177224dfb8aa6dbccc72a3f5e62c37fb` |
| `24` | `images/24-visualizations-list-three-saved.png` | 65,161 B | `4ac99dc64a40afacf9b19a79ecf346a6be06996e9ae1a31b2753bd5567c3bf5e` |

### Setup-phase captures (S01–S31)

| Ref | File | Size | SHA-256 |
|---|---|---|---|
| `S01` | `images/S01-host-specs.png` | 31,633 B | `74d19fb92b935fbe8022dba2c07f13b5f001ea5525e1732201b5a908de6a74f2` |
| `S04` | `images/S04-vbox-new-vm-disk-50gb.png` | 96,081 B | `67303d913f70c905eefd4183006e4b8e4c884e28482273dea26a372f60ce7795` |
| `S05` | `images/S05-vbox-new-vm-hardware-2gb-2cpu.png` | 91,353 B | `7a40a4edeb35568591badaa07a0814f927a659955d61367fbd2e775f8533fb6b` |
| `S06` | `images/S06-vbox-new-vm-summary.png` | 107,442 B | `64a302078cd70ad45863e73f81ea6347a4147107cc818598a97f79f597d6d6ac` |
| `S07` | `images/S07-ubuntu-installer-network-dhcp.png` | 16,032 B | `64f5e9fa074b47c53c720f046626c11d077059269623f21c0abe4a31e47dcb32` |
| `S08` | `images/S08-ubuntu-installer-storage-lvm.png` | 29,935 B | `c693d33d3d2936aa88d29ad15e21108e4a137cab4862e4c5cd11e6119d512960` |
| `S09` | `images/S09-ubuntu-installer-profile.png` | 19,137 B | `824035318ab499535100a5404217c423ef371fcf11ff6666b2c9ebd8e5640e5f` |
| `S10` | `images/S10-ubuntu-installing-system.png` | 25,282 B | `4d96d6a7419481c610f483f5c7507a7762e079ab4104673e9f2a56715ed22add` |
| `S11` | `images/S11-first-login-base-vm.png` | 19,012 B | `48a9ffad5f3905e4f2645110574a907bc51c3c44a1c8aa250dd5421ab1d90f7d` |
| `S12` | `images/S12-ssh-port-22-listening-connection-refused.png` | 2,241 B | `74a6e772897455f2b0c8bbe21d0295304b2f6dd0f08ca0fd98f3edd23291a9fd` |
| `S13` | `images/S13-vbox-port-forwarding-empty.png` | 55,609 B | `77405a0ae11f73dc27817e8e574d12f76b8201f6d4714c28abd461b451724deb` |
| `S14` | `images/S14-vbox-port-forwarding-ssh-2222-to-22.png` | 10,663 B | `1a1e4bf1ab845e37fc95aac6c644e08006669ee086b9bb847931049860ac98de` |
| `S15` | `images/S15-base-vm-poweroff-df-48g.png` | 20,330 B | `d0b6db8a342e102c4d6caeed32df3018769b040db846ddb9c85072033ace8195` |
| `S16` | `images/S16-ip-a-enp0s8-down.png` | 36,048 B | `6bc030da42e30fb865fc62a90ddc64ca85642c4847aca8e784cbeabe756e85f4` |
| `S17` | `images/S17-vbox-clone-dialog-base-clean.png` | 154,879 B | `6f9c38573293376158116168c6997ea0507fe02dc18b70096e587bdbb0f141ad` |
| `S18` | `images/S18-vbox-clone-soc-wazuh-mgr-full-clone.png` | 105,558 B | `a3cd3ed944d0c61f797f325c1081b255d65764a03f0ddf19efc073d0010c7c5f` |
| `S19` | `images/S19-wazuh-mgr-3-cpu.png` | 40,549 B | `4a7152a90bb7a0824601d2694bf78badc6d765427437651d90d58af07782dbbd` |
| `S20` | `images/S20-wazuh-mgr-7168mb-ram.png` | 50,807 B | `30e2b8a8f9168114f280dbe4ef0f4fdfd431907f4de25bc670ca8c25886af3f7` |
| `S21` | `images/S21-wazuh-mgr-adapter2-host-only.png` | 47,949 B | `a46a2807d9f98df83a74170c331c937c615af90af8d301a473aeabb837bf37ed` |
| `S22` | `images/S22-first-ssh-to-clone.png` | 82,574 B | `456a9fddfdde76341e00a35a7944380b8a5ce415edb01a266163555fdd461685` |
| `S23` | `images/S23-clone-identity-hostname-machine-id-ssh-keys-ip.png` | 123,673 B | `54e0af3238c18a8d2c70b3ba4677c29e421b58b311efad8bfb5b62817e8923e1` |
| `S24` | `images/S24-wazuh-install-assistant-started-4.14.9.png` | 6,054 B | `c6758fef7b6a240c77cd146d1513aa3d1ccdd2b30bedf5aa33177529b43ccd4d` |
| `S25` | `images/S25-endpoint-clone-identity-and-ssh.png` | 118,740 B | `80a0d12ec0e35001e6535a0e8e1d7595817f896f3ee26def0641499f23648d53` |
| `S26` | `images/S26-fim-test-dir-ossec-conf-sed.png` | 24,503 B | `6a9eeaf74c2e21e51aa31d99c1fbb0926bf2e535e9cca3d90669da556bb4ba1f` |
| `S27` | `images/S27-fim-config-agent-running-touch-a-txt.png` | 46,700 B | `5ccc591d93e8184eb11933185b63adce0b43efe213ffb843022d3f4ce977c321` |
| `S28` | `images/S28-t1110-second-run-20-attempts.png` | 54,295 B | `e793ee293722c94dccfa5596fdc8bc67ec392da48286dea439380985720a0152` |
| `S29` | `images/S29-t1136-useradd-suspicious-user3.png` | 7,094 B | `d12dcc92a7b5a3681de24aeec292a14e1f162ba46e75c18686d9a297b1b5e557` |
| `S30` | `images/S30-new-visualization-types.png` | 111,089 B | `c4c84f5b496166ae23af593bd637e148ecd028fd7206954faa593f51b45b6cfe` |
| `S31` | `images/S31-area-visualization-built.png` | 140,543 B | `e5c26025b4f5b471bab0ae2c1024362d7dbef131dfa374ef9a964d098d18d719` |
