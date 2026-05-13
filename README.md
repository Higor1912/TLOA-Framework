# TLOA — Threat-Led Offensive Audit

> A personal cybersecurity framework built around threat intelligence-driven attack simulation, detection validation, and incident response — grounded in MITRE ATT&CK and designed to expose what your SIEM misses.

---

## What is TLOA?

Most security practitioners learn tools in isolation: one day it's nmap, another it's Metasploit, another it's Wazuh. But in real attacks, adversaries don't follow tutorials — they follow objectives. And a good defender needs to understand exactly that reasoning.

**TLOA (Threat-Led Offensive Audit)** is a structured framework for bridging that gap: start from real threat intelligence, build attack scenarios based on documented MITRE ATT&CK TTPs, execute them in a controlled environment, and validate what the SIEM detects — and what it misses.

Detection gaps are not failures of the lab. They are its most valuable output.

---

## Core Principles

| # | Principle | Description |
|---|---|---|
| 1 | **Threat-oriented** | No test without a documented TTP basis |
| 2 | **Offensive mindset** | Think like the attacker to test like the defender |
| 3 | **Practical validation** | Technical evidence, not theory |
| 4 | **Risk-based prioritization** | Severity informed by real threat context, not CVSS alone |
| 5 | **Business impact focus** | Every finding framed in terms of organizational consequence |

---

## Methodology — 7 Phases

```
Phase 0 — Scoping        Define scope and authorization → Rules of Engagement
Phase 1 — Context        Map the target environment → Asset inventory, attack surface
Phase 2 — Threat Intel   Identify relevant threat actors → Threat profile, TTPs
Phase 3 — Modeling       Transform threats into scenarios → Documented attack paths
Phase 4 — Validation     Test security in practice → Evidence, detection gaps
Phase 5 — Prioritization Classify risks by real threat context → Risk ranking
Phase 6 — Reporting      Communicate risk clearly → Executive and technical report
```

---

## Lab Architecture

### Network Topology

```
VMware Workstation — Host-Only Network (VMnet1) — 192.168.204.0/24
│
├── DC01 — Windows Server 2025 (Domain Controller)
│     IP: 192.168.204.132 | Domain: tloa.local
│     Services: AD DS, DNS, Sysmon (SwiftOnSecurity), Wazuh Agent 4.7.5
│
├── WS01 — Windows 10 Pro (Domain Workstation / Victim)
│     IP: 192.168.204.130
│     Services: Sysmon (SwiftOnSecurity), Wazuh Agent
│
├── KALI — Kali Linux 2025.4 (Attacker)
│     IP: 192.168.204.128
│     Tools: Atomic Red Team, BloodHound, Netexec, Mimikatz
│
└── SIEM — Ubuntu Server 24.04 (Wazuh Manager)
      IP: 192.168.204.129
      Stack: Wazuh 4.7.5 — SIEM/XDR, ATT&CK-mapped alert dashboards
```

### Component Stack

| Component | Version | Role |
|---|---|---|
| Windows Server 2025 | DC01 | Domain Controller, AD DS, DNS |
| Windows 10 Pro | WS01 | Domain workstation (victim) |
| Kali Linux | 2025.4 | Attacker — TTP simulation |
| Wazuh | 4.7.5 | SIEM/XDR — detection and alerting |
| Sysmon | SwiftOnSecurity config | Endpoint telemetry enrichment |
| Atomic Red Team | Latest | ATT&CK-mapped technique execution |
| BloodHound | Community Edition | AD attack path enumeration |

### Active Directory — Intentional Attack Surface

| Account | Type | Misconfiguration | Related Technique |
|---|---|---|---|
| `jsmith` | Standard user | Weak password, credential reuse | T1078, T1110 |
| `sconnor` | Standard user | Member of sensitive groups | T1069, T1087 |
| `svc-backup` | Service account | Kerberoastable (SPN: `MSSQLSvc/dc01.tloa.local:1433`) | T1558.003 |

> **Note on Windows Server 2025 hardening:** WS2025 deprecates RC4 and enforces LDAP signing by default. Kerberoasting attempts against `svc-backup` are blocked in this environment. These were documented as realistic technical findings — not bypassed — reflecting the behavior of modern production environments.

---

## ATT&CK Coverage & Detection Results

| TTP | Technique | Executed | Detected | Notes |
|---|---|---|---|---|
| T1046 | Network Service Discovery | ✅ | ❌ | Windows Firewall blocked scan — network-layer gap |
| T1053.005 | Scheduled Task/Job | ✅ | ✅ | Wazuh Rule 61104 — real-time |
| T1112 | Modify Registry | ✅ | ⚠️ | Detected with 12h syscheck latency — Sysmon EID 13 recommended |
| T1548.002 | UAC Bypass (Event Viewer) | ✅ | ✅ | Rules 594, 750 — ATT&CK mapped |
| T1003.001 | OS Credential Dumping — LSASS | ✅ | ❌ | No native Wazuh rule — critical gap |
| T1550.002 + T1021.006 | Pass-the-Hash / WinRM Lateral Movement | ✅ | ❌ | DC01 blocked via SMB signing; WS01 successfully exploited |

**Effective detection rate: 40–60%** — expected for a Wazuh-only stack without a dedicated EDR. The gaps are the point.

### Identified Gaps & Recommendations

| Gap | Technique | Recommendation |
|---|---|---|
| Network recon | T1046 | Deploy Suricata or Zeek for network-layer visibility |
| Registry modification latency | T1112 | Use Sysmon Event ID 12/13 for real-time FIM |
| LSASS credential dump | T1003.001 | Custom Wazuh rules for Sysmon EID 10, filtering unauthorized lsass.exe access |

---

## IR Case Structure

Every investigation in `TLOA-IR-Cases` follows a consistent 4-phase format:

```
Phase 1 — Attack Execution     Technique selection, execution steps, artifacts generated
Phase 2 — Detection & Triage   Alert analysis, Wazuh rule mapping, severity assessment
Phase 3 — Investigation        IOC extraction, timeline reconstruction, log correlation
Phase 4 — Reporting            Executive summary, ATT&CK mapping, remediation
```

---

## Framework Repositories

| Repository | Description |
|---|---|
| [`TLOA-Framework`](https://github.com/Higor1912/TLOA-Framework) | This repository — framework documentation and architecture |
| [`TLOA-Lab-Exercises`](https://github.com/Higor1912) | ATT&CK-mapped exercises with execution logs, Wazuh detections, and gap analysis |
| [`TLOA-IR-Cases`](https://github.com/Higor1912) | Structured IR investigations following the 4-phase format |
| [`CTI-Toolkit`](https://github.com/Higor1912) | Python modules for offline TTP mapping (STIX 2.1), IOC enrichment, and report generation |
| [`Threat-Intel-Reports`](https://github.com/Higor1912) | Structured threat intelligence reports with ATT&CK coverage and IOC tables |

---

## Related

- 📄 Medium: [TLOA — Threat-Led Offensive Audit](https://medium.com/@higorsilva1816/tloa-threat-led-offensive-audit-8863d9962e63)
- 🔗 LinkedIn: [linkedin.com/in/higor-silva-sec](https://linkedin.com/in/higor-silva-sec)
- 💻 GitHub: [github.com/Higor1912](https://github.com/Higor1912)

---

## Language

This README is written in English for broader accessibility. Architecture and methodology documentation under `/docs` is written in Brazilian Portuguese.

---

*TLOA is a personal learning framework — not affiliated with any organization. All simulations are conducted in an isolated lab environment.*
