# Home Detection Engineering Lab

A self-contained detection engineering environment built on commodity laptop hardware for hands-on rule development, threat intelligence operationalization, and ATT&CK coverage analysis. Built by a cybersecurity practitioner with ~25 years of industry experience as exercises for a focused pivot into detection engineering and threat intelligence roles.

---

## Goals

| Area | Purpose |
|------|---------|
| **Detection rule development & testing** | Write and validate Sigma rules and KQL/EQL detections against realistic telemetry. Replay recorded attack datasets (EVTX samples, PCAPs) to confirm rule logic without waiting for live attacks. |
| **Threat intelligence operationalization** | Ingest structured IOC feeds into Elasticsearch. Build and tune indicator match rules. Bridge the gap between raw feed data and actionable detections. |
| **Network visibility** | Protocol-aware logging (Zeek) and signature-based alerting (Suricata) across all lab network traffic. Community ID enables cross-tool flow correlation. |
| **Windows endpoint telemetry** | Sysmon-instrumented Windows endpoints feed process creation, network connections, registry activity, and LSASS access events into the stack via Elastic Agent. Mirrors enterprise SOC telemetry profiles. |

The core detection engineering loop this lab supports:  
**identify technique → generate telemetry → write detection → validate → map to ATT&CK → identify coverage gaps → repeat**

---

## Hardware

| Device | Role |
|--------|------|
| ThinkPad X1 Carbon Gen 7 | ELK server / sensor host · Debian 13 · headless · static IP · 16 GB RAM |
| ThinkPad X1 Carbon Gen 8 | Daily driver / analyst workstation · Debian 13 · Kibana browser · SSH client |
| ASUS ROG Zephyrus G14 | Windows VM host for endpoint telemetry workloads |
| Home Wi-Fi (802.11ac) | Lab interconnect · all traffic on 192.168.1.0/24 |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                     HOME NETWORK  192.168.1.0/24                    │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │         elastic-lab  (ThinkPad X1 Carbon Gen 7)              │   │
│  │         Debian 13 · Headless · Static IP 192.168.1.50        │   │
│  │                                                              │   │
│  │  ┌─────────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐  │   │
│  │  │Elasticsearch│  │ Logstash │  │  Kibana  │  │  Fleet  │  │   │
│  │  │   :9200     │  │  :5044   │  │  :5601   │  │  :8220  │  │   │
│  │  └──────┬──────┘  └────┬─────┘  └────┬─────┘  └────┬────┘  │   │
│  │         └──────────────┴──────────────┴─────────────┘        │   │
│  │                    Docker bridge network                      │   │
│  │                                                              │   │
│  │  ┌──────────────────┐    ┌──────────────────────────────┐   │   │
│  │  │  Zeek (host)     │    │  Suricata (host)             │   │   │
│  │  │  passive · wlp*  │    │  IDS mode · EVE JSON         │   │   │
│  │  │  conn/dns/http/  │    │  ET Open + abuse.ch rules    │   │   │
│  │  │  ssl/files logs  │    │  daily auto-update           │   │   │
│  │  └──────────────────┘    └──────────────────────────────┘   │   │
│  │                                                              │   │
│  │  ┌──────────────────┐    ┌──────────────────────────────┐   │   │
│  │  │  MISP  :8080     │    │  Elastic Agent (local)       │   │   │
│  │  │  Threat Intel    │    │  System · Auditd · Docker    │   │   │
│  │  │  Docker          │    │  Zeek · Suricata logs        │   │   │
│  │  └──────────────────┘    └──────────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────┐    ┌──────────────────────────────┐   │
│  │  daily-driver           │    │  Windows endpoint            │   │
│  │  ThinkPad X1C Gen 8     │    │  (physical / VM)             │   │
│  │  Debian 13              │    │  Sysmon + Elastic Agent      │   │
│  │  Kibana · SSH access    │    │  → Fleet :8220               │   │
│  └─────────────────────────┘    └──────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

**Data flows:**
- Zeek and Suricata run on the host OS, reading the network interface passively. Their logs are shipped to Elasticsearch by the local Elastic Agent.
- Windows endpoints enroll via Fleet Server and stream Sysmon, Security, PowerShell, and Defender events directly into the pipeline.
- Threat intelligence integrations poll Abuse.ch, OTX, and MISP on a schedule, populating IOC indices that indicator match rules cross-reference against all incoming events.
- The daily driver accesses Kibana on port 5601 over Wi-Fi — no display or keyboard required on the server.

---

## Stack

| Service | Port | Notes |
|---------|------|-------|
| Elasticsearch | 9200 | Single-node · 4 GB JVM heap · xpack.security enabled |
| Kibana | 5601 | Security app · Fleet · ATT&CK coverage map |
| Logstash | 5044 / 5000 | Beats input + JSON syslog input |
| Fleet Server | 8220 | Agent enrollment and policy management |
| Elastic Agent | — (local) | Ships Zeek, Suricata, Auditd, Docker, system logs |
| Zeek | wlp* passive | Community ID · conn / dns / http / ssl / files logs |
| Suricata | wlp* IDS mode | ET Open + Abuse.ch · EVE JSON · daily rule updates |
| MISP | 8080 | Threat intel platform · CIRCL, Botvrij, Abuse.ch feeds |
| SSH | 2222 | Key-based auth only · root disabled · UFW allowlist |

All ELK components and MISP run in Docker (Docker Compose). Zeek and Suricata run on the host OS for direct interface access.

---

## Threat Intelligence Sources

All free-tier:

- **Abuse.ch MalwareBazaar** — malware hashes (MD5 / SHA1 / SHA256) with family tags
- **Abuse.ch Feodo Tracker** — botnet C2 IPs and ports (Emotet, TrickBot, Dridex, QakBot)
- **Abuse.ch URLhaus** — malware distribution URLs
- **Abuse.ch SSL Blacklist** — JA3 / SHA1 fingerprints of known C2 TLS infrastructure
- **Abuse.ch ThreatFox** — multi-type IOCs across malware families
- **AlienVault OTX** — community pulse data (IPs, domains, hashes, URLs)
- **MISP community feeds** — CIRCL OSINT, Botvrij.eu, ESET IOC

Indicator match rules in Kibana Security cross-reference all incoming events against these feeds automatically.

---

## ATT&CK Coverage

| Tactic | Data Sources | Coverage |
|--------|-------------|----------|
| Initial Access | Suricata, Zeek http/files, Windows Security | Moderate |
| Execution | Sysmon EID 1, PowerShell/Operational, Auditd | High |
| Persistence | Sysmon EID 11/12/13, Windows Security, Auditd | High |
| Privilege Escalation | Sysmon EID 10, Windows Security 4672/4673 | Moderate |
| Defense Evasion | Sysmon EID 7/15, PowerShell logs | Moderate |
| Credential Access | Sysmon EID 10, Windows Security 4648/4768/4769 | Moderate |
| Discovery | Sysmon EID 1, Zeek conn.log, Auditd | Moderate |
| Lateral Movement | Zeek conn/smb/dce_rpc, Windows Security 4624/4625 | Moderate |
| Command & Control | Zeek dns/ssl/conn, Suricata C2 rules, TI indicator match | High |
| Exfiltration | Zeek files/conn volume anomalies, Suricata, DNS analytics | Low–Moderate |

**Known gaps:** Memory-resident techniques (process injection, reflective DLL loading) and encrypted channel evasion. Planned additions: Elastic Defend EDR mode, network decryption via mitmproxy.

---

## Sample Datasets Used

For detection development without waiting for live attacks:

| Dataset | Source | Use |
|---------|--------|-----|
| EVTX-ATTACK-SAMPLES | [sbousseaden/EVTX-ATTACK-SAMPLES](https://github.com/sbousseaden/EVTX-ATTACK-SAMPLES) | Windows EVTX mapped to ATT&CK techniques |
| Security Datasets (OTRF) | [OTRF/Security-Datasets](https://github.com/OTRF/Security-Datasets) | Structured host + network attack simulations |
| Malware Traffic Analysis | [malware-traffic-analysis.net](https://malware-traffic-analysis.net) | Real malware PCAPs → replay through Zeek/Suricata |
| BOTS v3 | [splunk/botsv3](https://github.com/splunk/botsv3) | Rich multi-source incident competition dataset |

---

## Related

- [Sigma](https://github.com/SigmaHQ/sigma) — vendor-neutral detection rule format
- [Elastic Detection Rules](https://github.com/elastic/detection-rules) — Elastic's prebuilt rule repository
- [Sysmon Modular](https://github.com/olafhartong/sysmon-modular) — tunable Sysmon configuration framework
- [MITRE ATT&CK](https://attack.mitre.org) — adversary technique framework used for coverage mapping
