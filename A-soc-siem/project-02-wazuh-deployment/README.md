# Project 02 — Wazuh SIEM Deployment

**Status:** ⚪ Planned
**Track:** SOC & SIEM

## Objective
Deploy a working SIEM and onboard every lab host.

## Scenario
The company has no central logging. Stand up Wazuh and get Windows and Linux telemetry flowing with alerts the SOC can actually use.

## Environment
| Item | Value |
|---|---|
| Case ID | CB-02-001 |
| Host OS | Ubuntu Server 24.04 (manager); Windows 10/11 eval (endpoint); Ubuntu Server 22.04 (endpoint) |
| VMs used | soc-wazuh-mgr (192.168.60.10), soc-win-endpoint (192.168.60.20), soc-linux-endpoint (192.168.60.21) |
| Tool versions | Wazuh 4.9.x (confirm exact version at install time) |
| Evidence source | Wazuh dashboard, agent enrollment log, ossec.conf diffs |
| Time zone used | UTC |

## Tools
- Wazuh manager / indexer / dashboard
- Wazuh agents

## Reference
- Lab design (network, VM specs, power-on groups): see ../LAB-ARCHITECTURE.md
- Step-by-step deployment runbook (commands for each build step below): see ./runbook.md

## Evidence Inventory
| Item | Size | MD5 | SHA-256 | Source | Acquired (UTC) |
|---|---|---|---|---|---|
| | | | | | |

## Investigation / Build Steps

### Step 1 — Plan sizing and deploy Wazuh (all-in-one or distributed)
_Commands, screenshots and observations._

### Step 2 — Enroll Windows DC, endpoints and Linux server agents
_Commands, screenshots and observations._

### Step 3 — Enable FIM on sensitive paths, SCA and vulnerability detection
_Commands, screenshots and observations._

### Step 4 — Generate test events (failed logons, new user, file change) and confirm alerts
_Commands, screenshots and observations._

### Step 5 — Tune noisy rules; document every change and why
_Commands, screenshots and observations._

### Step 6 — Build a SOC overview dashboard
_Commands, screenshots and observations._

## Problems Encountered
| Problem | What happened | Root cause | Fix | Lesson |
|---|---|---|---|---|
| | | | | |

## Findings
| # | Observation | Evidence location | Interpretation | Confidence |
|---|---|---|---|---|
| 1 | | | | |

## ATT&CK Mapping
- T1110 Brute Force
- T1136 Create Account

## IOCs
| Type | Value | Context |
|---|---|---|
| | | |

## Deliverables
- [ ] Investigation notes
- [ ] Screenshots in images/
- [ ] Agent inventory
- [ ] Tuning log (rule, change, reason)
- [ ] Dashboard screenshots
- [ ] Final report in report/

## Key Learnings
1. 
2. 
3.
