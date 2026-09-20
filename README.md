# Azure SOC / SIEM Honeypot Lab

A hands-on home lab that builds a small Security Operations Center (SOC) in Microsoft Azure: a deliberately exposed "honeypot" VM feeds Windows Security Event logs into a Log Analytics Workspace / Microsoft Sentinel, where the failed-logon attempts are enriched with geolocation data and visualized on a live attack map.

![Windows VM Attack Map](screenshots/15-attack-map.png)

## Overview

| | |
|---|---|
| **Goal** | Stand up a cloud SIEM pipeline end-to-end and use it to observe real internet-background-noise attacks against an exposed VM |
| **Cloud** | Microsoft Azure |
| **SIEM** | Microsoft Sentinel (on a Log Analytics Workspace) |
| **Target** | Windows VM with RDP exposed to the internet and the local firewall disabled |
| **Enrichment** | GeoIP watchlist (~54,000 IP ranges) joined against failed logons via KQL |
| **Output** | Interactive Sentinel Workbook plotting attacker source locations on a world map |

## Architecture

```
 Internet ── (open NSG rule, all inbound) ──▶  Windows VM ("honeypot")
                                                     │  Windows Firewall disabled
                                                     │  Security Event Log (4625 = failed logon)
                                                     ▼
                                     Azure Monitor Agent (AMA) + Data Collection Rule
                                                     │
                                                     ▼
                                     Log Analytics Workspace (LAW)
                                                     │
                                                     ▼
                                          Microsoft Sentinel (SIEM)
                                                     │
                              ┌──────────────────────┴──────────────────────┐
                              ▼                                             ▼
                     KQL queries (4625 events)                 GeoIP Watchlist (ipv4_lookup)
                              │                                             │
                              └──────────────────────┬──────────────────────┘
                                                     ▼
                                        Sentinel Workbook: Attack Map
```

## Lab Steps

### Part 1 — Create the Azure environment

Created a resource group and virtual network to host the lab resources.

![Resource group - search](screenshots/01-resource-group-1.png)
![Resource group - review + create](screenshots/01-resource-group-2.png)
![Virtual network - configuration](screenshots/02-virtual-network-1.png)
![Virtual network - deployment complete](screenshots/02-virtual-network-2.png)

### Part 2 — Deploy the honeypot VM

Deployed a Windows VM, then intentionally weakened it so it would attract internet scanners and brute-force attempts:

- Created a **Network Security Group** inbound rule allowing **any** source, any port, any protocol
- Disabled the **Windows Defender Firewall** on all profiles (Domain, Private, Public)
- Verified reachability over RDP

![VM overview](screenshots/03-virtual-machine.png)
![NSG rule allowing all inbound traffic](screenshots/04-nsg-inbound-rule.png)
![RDP connection to the honeypot](screenshots/05-rdp-connection.png)
![Firewall disabled - Public profile](screenshots/06-firewall-off-1.png)
![Firewall disabled - Private profile](screenshots/06-firewall-off-2.png)
![Firewall disabled - Domain profile](screenshots/07-firewall-off-3.png)
![Successful ping confirming the honeypot is reachable](screenshots/08-ping-success.png)

> ⚠️ **This configuration is intentionally insecure.** It exists solely to attract unsolicited internet traffic for log analysis in an isolated lab. Never do this to a production system, and always shut down or delete the VM when the lab is finished to avoid cost/exposure.

### Part 3 — Generate and inspect logs

Triggered failed logon attempts and confirmed they appear in the local **Event Viewer** as **Event ID 4625**, before centralizing logging.

### Part 4 — Centralized log forwarding (Log Analytics + Sentinel)

- Created a **Log Analytics Workspace (LAW)**
- Added **Microsoft Sentinel** on top of the workspace
- Enabled the **Windows Security Events via AMA** data connector
- Created a **Data Collection Rule (DCR)** to ship Security Event logs from the VM to the workspace
- Queried the raw events with KQL (see [`queries/01-failed-logins.kql`](queries/01-failed-logins.kql))

![Log Analytics Workspace creation](screenshots/09-log-analytics-workspace.png)
![Adding Microsoft Sentinel to the workspace](screenshots/09-sentinel-to-law.png)
![Windows Security Events connector review](screenshots/10-windows-security-events.png)
![Windows Security Events via AMA content hub](screenshots/11-ama-connector.png)
![Data Collection Rule creation](screenshots/11-data-collection-rule.png)

### Part 5 — Enrich logs with GeoIP data

Raw `SecurityEvent` logs only contain the attacker's IP address, not location. To fix that:

- Imported a ~54,000-row CIDR-block-to-location CSV as a **Sentinel Watchlist** named `geoip` (search key: `network`)
- Used the `ipv4_lookup()` KQL plugin to join failed logons against the watchlist and resolve city/country/lat/long

See [`queries/02-geoip-enrichment.kql`](queries/02-geoip-enrichment.kql).

![Watchlist import for GeoIP data](screenshots/12-watchlist-geoip.png)
![KQL query joining failed logons to GeoIP data](screenshots/13-kql-geo-lookup.png)

### Part 6 — Build the attack map

Created a Sentinel **Workbook** with a Query element, pasted in custom JSON to render the enriched data as a heat map (bubble size/color = number of failed attempts per location).

See [`workbook/attack-map.json`](workbook/attack-map.json).

![Workbook query JSON](screenshots/14-workbook-json.png)
![Final attack map](screenshots/15-attack-map.png)

## Repository Structure

```
.
├── README.md
├── queries/
│   ├── 01-failed-logins.kql        # raw 4625 events
│   └── 02-geoip-enrichment.kql     # 4625 events joined to GeoIP watchlist
├── workbook/
│   └── attack-map.json             # Sentinel Workbook JSON for the map visualization
└── screenshots/                    # step-by-step screenshots referenced above
```

## Skills Demonstrated

- Azure resource provisioning (Resource Groups, VNets, VMs, NSGs)
- Windows security fundamentals (Event Viewer, Windows Defender Firewall, Event ID 4625)
- SIEM architecture (Log Analytics Workspace, Microsoft Sentinel, Azure Monitor Agent, Data Collection Rules)
- KQL (Kusto Query Language) for log querying and enrichment (`ipv4_lookup`, `evaluate`, `summarize`)
- Threat intelligence enrichment via Sentinel Watchlists
- Data visualization with Sentinel Workbooks

## Reproducing This Lab

1. Create a free or pay-as-you-go [Azure subscription](https://azure.microsoft.com/en-us/pricing/purchase-options/azure-account).
2. Deploy a Windows VM, open all inbound traffic on its NSG, and disable the Windows Firewall.
3. Fail a few logons against it, then confirm Event ID 4625 in the local Event Viewer.
4. Create a Log Analytics Workspace and add Microsoft Sentinel to it.
5. Enable the **Windows Security Events via AMA** connector and create a Data Collection Rule for the VM.
6. Import the [geoip-summarized.csv](https://raw.githubusercontent.com/joshmadakor1/lognpacific-public/refs/heads/main/misc/geoip-summarized.csv) as a Sentinel Watchlist named `geoip` (search key `network`).
7. Run the queries in [`queries/`](queries/) against the workspace.
8. Create a new Workbook, add a Query element, switch to Advanced Editor, and paste [`workbook/attack-map.json`](workbook/attack-map.json).


