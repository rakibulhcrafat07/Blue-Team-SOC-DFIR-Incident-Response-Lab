# Runbook - Project 02: Wazuh SIEM Deployment

A planned step-by-step reference for the 6 build steps in the README. This is a runbook to follow, not evidence - real commands run, screenshots, and findings belong in the README's own Build Steps / Evidence Inventory / Findings sections once this project is actually executed.

Uses the lab from ../LAB-ARCHITECTURE.md: soc-wazuh-mgr (192.168.60.10, 6GB/3vCPU), soc-win-endpoint (192.168.60.20), soc-linux-endpoint (192.168.60.21).

## Step 1 - Sizing and deployment (the manager)

On soc-wazuh-mgr, run:

curl -sO https://packages.wazuh.com/4.9/wazuh-install.sh
sudo bash wazuh-install.sh -a

The installer prints an admin password at the end - save it immediately, it is shown only once. Takes 10-15 minutes. Log in at https://192.168.60.10

## Step 2 - Agent enrollment

Linux agent (soc-linux-endpoint):

curl -sO https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.9.0-1_amd64.deb
sudo WAZUH_MANAGER=192.168.60.10 dpkg -i ./wazuh-agent_4.9.0-1_amd64.deb
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent

Windows agent (soc-win-endpoint): install the .msi from the Wazuh downloads page, set WAZUH_MANAGER to 192.168.60.10 during install.

Confirm both hosts show Active under Agents in the dashboard.

## Step 3 - File Integrity Monitoring (FIM / syscheck)

In ossec.conf on each agent, add syscheck directories for /etc and /bin,/sbin with check_all and realtime enabled.

Restart the agent, then test: touch /etc/test-fim.txt and confirm an FIM alert appears.

## Step 4 - SCA and vulnerability detection

On the manager, in ossec.conf, enable the vulnerability-detector module with a 5 minute interval and run_on_start yes.

SCA (CIS-style checks) runs automatically once an agent is enrolled - check SCA under each agent's detail page.

## Step 5 - Test event generation (ATT&CK mapping)

T1110 Brute Force: from soc-win-endpoint or another host, attempt repeated SSH logins with a bad password against soc-linux-endpoint (192.168.60.21), about 10 attempts.

T1136 Create Account: on Linux run useradd to add a test user; on Windows run net user to add a test account.

Confirm both surface as alerts; note the rule ID each one triggers.

## Step 6 - Rule tuning and SOC overview dashboard

Check the default sshd rule threshold in the Wazuh ruleset before changing anything. Add one custom rule in local_rules.xml (for example, flag useradd specifically), restart the manager, confirm it fires without duplicating the default alert. Build a dashboard with 2-3 visualizations: alert count over time, top rule IDs, agent status.

## Wrapping up

Move results into the README: Evidence Inventory (every screenshot or log, with hash or timestamp), Findings (SCA/vuln results, rule IDs), IOCs (test account name/IP used, clearly marked simulated), Deliverables checklist, then flip Status from Planned to Done.
