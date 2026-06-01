<div align="center">

```
 ██████╗ ██████╗ ███████╗ ██████╗██╗   ██╗██████╗  █████╗
██╔═══██╗██╔══██╗██╔════╝██╔════╝██║   ██║██╔══██╗██╔══██╗
██║   ██║██████╔╝███████╗██║     ██║   ██║██████╔╝███████║
██║   ██║██╔══██╗╚════██║██║     ██║   ██║██╔══██╗██╔══██║
╚██████╔╝██████╔╝███████║╚██████╗╚██████╔╝██║  ██║██║  ██║
 ╚═════╝ ╚═════╝ ╚══════╝ ╚═════╝ ╚═════╝ ╚═╝  ╚═╝╚═╝  ╚═╝
```

**Obscura Threat Investigation**

*Open-source threat intelligence investigations — from raw signals to structured findings.*

[![TLP:WHITE](https://img.shields.io/badge/TLP-WHITE-white?style=flat-square)](https://www.first.org/tlp/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)](LICENSE)
[![Maintained by](https://img.shields.io/badge/maintained%20by-sudo3rs-black?style=flat-square)](https://github.com/Masriyan)
[![Investigations](https://img.shields.io/badge/investigations-active-red?style=flat-square)](#investigations)

</div>

---

## What Is This

**Obscura** is a public threat intelligence investigation repository. Every folder here is a standalone case — a real threat, infrastructure cluster, malware campaign, or attacker operation that was investigated and documented in full.

The goal is simple: turn raw threat signals into structured, reusable intelligence that defenders can actually use. Not marketing. Not vendor reports. Just the work.

Each investigation covers the full chain — initial discovery, infrastructure analysis, behavioral breakdown, IOC extraction, MITRE ATT&CK mapping, and defensive recommendations. Everything is TLP:WHITE unless noted otherwise.

---

## Investigations

| # | Case | Category | Status | Date |
|---|------|----------|--------|------|
| 001 | [ClickFix Malvertising — vid30s.com & vidara.to](./001-clickfix-vid30s-vidara/) | Malvertising / InfoStealer | ✅ Complete | 2026-06-01 |

> More cases added as investigations complete. Watch the repo to get notified.

---

## Case Structure

Every investigation follows the same layout:

```
NNN-case-name/
├── README.md          # Executive summary and key findings
├── report/
│   ├── full-report.md         # Complete investigation narrative
│   ├── full-report-id.md      # Bahasa Indonesia version (where available)
│   └── full-report.docx       # Formatted document export
├── iocs/
│   ├── domains.txt            # One domain per line, defanged
│   ├── ips.txt                # IP addresses
│   ├── urls.txt               # Full URLs
│   └── ioc-master.csv         # All IOCs with type, value, confidence, notes
├── artifacts/
│   ├── decoded/               # Deobfuscated scripts, decoded payloads
│   ├── pcap/                  # Network captures (sanitized)
│   └── samples/               # File samples (password-protected, pw: infected)
├── rules/
│   ├── yara/                  # YARA detection rules
│   └── sigma/                 # Sigma rules for SIEM/EDR
└── notes/
    └── raw-audit-log.txt      # Raw tool output and session logs
```

---

## IOC Format

All IOCs are defanged for safe distribution:

```
# Defanging conventions used in this repo
example[.]com          # domains
hxxps://example[.]com  # URLs
192[.]168[.]1[.]1      # IPs
```

The `ioc-master.csv` in each case uses this schema:

```
type,value,confidence,first_seen,last_seen,tags,notes
domain,vid30s[.]com,HIGH,2026-05-30,2026-06-01,bait;clickfix;namecheap,2-day-old churn domain
ip,69.41.166.29,HIGH,2026-06-01,2026-06-01,injector;clickfix-payload,pinderecphory.com origin IP
```

---

## Detection Rules

YARA and Sigma rules are published in each case's `rules/` directory. They're written to be immediately deployable — tested against the observed behavior, not theoretical signatures.

**YARA rules** cover file-based and memory detection patterns.  
**Sigma rules** target process creation, network, and web proxy log sources.

---

## Methodology

Investigations in this repo follow a consistent methodology:

```
1. Signal Collection     Passive recon, WHOIS, DNS, urlscan, passive DNS
2. Infrastructure Map    IP pivoting, cert transparency, GA ID correlation
3. Behavioral Analysis   Source code review, dynamic analysis, JS deobfuscation
4. IOC Extraction        Domains, IPs, URLs, campaign IDs, cookies, hashes
5. TTP Mapping           MITRE ATT&CK technique identification
6. Rule Development      YARA + Sigma for detection
7. Report + Publish      Full writeup, IOC export, rule release
```

Tools commonly used across investigations: `whois`, `dig`, `curl`, `urlscan.io`, `SecurityTrails`, `PassiveDNS`, `PublicWWW`, Wireshark, Ghidra, custom deobfuscation scripts.

---

## Using This Repo

**Defenders / SOC teams** — grab the IOC list from `iocs/` and the detection rules from `rules/` for each case. Everything is ready to import.

**Threat hunters** — start with the full report for context, then pivot from the campaign identifiers and infrastructure anchors documented in each case.

**Researchers** — raw audit logs and decoded artifacts are in `notes/` and `artifacts/`. The methodology section of each report documents every tool call and pivot decision.

**TI platforms** — IOCs are available in plain text and CSV. STIX/TAXII-formatted exports are planned for future cases.

---

## Contributions

If you have additional findings, overlapping IOCs, or attribution data that extends any case here, open an issue or PR. Community pivots are welcomed and credited.

**When contributing:**
- Defang all indicators before committing
- Don't commit live malware samples without password-protecting the archive (`pw: infected`)
- Include your source/methodology notes
- Follow the existing case structure

---

## Disclaimer

All investigations in this repository are conducted for **defensive purposes only** — to identify, document, and help block malicious infrastructure.

No offensive activity was performed against any target. All behavioral analysis was conducted in isolated sandbox environments. IOCs are published to enable detection and blocking, not to facilitate attacks.

If you are a legitimate operator of any infrastructure mentioned in these reports and believe it was flagged in error, open an issue with supporting evidence.

---

## Author

**!!!* (sudo3rs)  
Threat Researcher & Hunter  

[![GitHub](https://img.shields.io/badge/GitHub-Masriyan-black?style=flat-square&logo=github)](https://github.com/Masriyan)

> *"Omnia hostium vestigia sequor."*  
> — Follow every trace of the enemy.

---

<div align="center">
<sub>TLP:WHITE — This repository and its contents may be distributed freely.</sub>
</div>
