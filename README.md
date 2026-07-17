# Infrastructure Security Hardening — Multi-VM Lab

![Ubuntu](https://img.shields.io/badge/Ubuntu-25.10%20%2F%2026.04-E95420?logo=ubuntu)
![OpenSSH](https://img.shields.io/badge/OpenSSH-key--only%20auth-black?logo=openssh)
![UFW](https://img.shields.io/badge/UFW-firewall-0078D4)
![Docker](https://img.shields.io/badge/Docker-containers-2496ED?logo=docker)
![Zabbix](https://img.shields.io/badge/Zabbix-8.0-D81F26?logo=zabbix)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-17-4169E1?logo=postgresql)
![Wazuh](https://img.shields.io/badge/Wazuh-4.14-3AB6E6?logo=wazuh)
![License](https://img.shields.io/badge/License-MIT-green)

Hands-on security hardening and monitoring work across a multi-VM lab environment (application server, monitoring server, S3-compatible storage). Covers SSH/firewall hardening, cloud architecture analysis, database performance testing, and security event monitoring with Wazuh.

Every command shown was executed against real, running infrastructure; every measurement and finding was verified, not simulated — including bugs found, hypotheses that turned out wrong, and fixes that had to be revised after testing.

---

## Architecture

```
infra-security-hardening/
├── Server_Security_Hardening_Report.md          # SSH + firewall hardening, Docker/UFW bypass finding
├── Cloud_Security_Architecture_Analysis.md       # Annotated architecture, network flows, gap analysis
├── Performance_Monitoring_Report.md              # PostgreSQL performance testing (2M rows) vs. Zabbix
├── Wazuh_Security_Monitoring_Report.md           # Wazuh deployment, event detection, rootcheck bug fix
├── carburoam_security_architecture.svg           # Referenced architecture diagram
└── README.md
```

```
                 ┌──────────────────────┐
                 │  External operator    │
                 │  SSH :22, key only     │
                 └──────────┬───────────┘
                            │
        ┌───────────────────┼────────────────────┐
        ▼                   ▼                    ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────────┐
│    App VM      │   │  Zabbix VM     │   │   Storage VM        │
│  Carburoam      │──▶│  Zabbix server │   │  SeaweedFS (S3 API) │
│  Agent + SNMP   │   │  PostgreSQL    │   │  Backup bucket      │
│  UFW            │   │  Apache + PHP  │   │                     │
└───────────────┘   └──────┬────────┘   └─────────▲──────────┘
                            │                        │
                            ▼                        │
                    ┌───────────────┐                │
                    │   Discord      │                │
                    │  alert channel │                │
                    └───────────────┘                │
                    (daily backup) ────────────────────┘
```

See [`carburoam_security_architecture.svg`](./carburoam_security_architecture.svg) for the full annotated diagram.

---

## Technology stack

| Component | Technology |
|---|---|
| Operating system | Ubuntu 25.10 / 26.04 (ARM64) |
| Remote access | OpenSSH — key-only authentication, no root login |
| Firewall | UFW, minimal per-service ruleset |
| Containerization | Docker |
| Performance monitoring | Zabbix 8.0 (Server + Agent2 + SNMP) |
| Database under test | PostgreSQL 17 |
| Security monitoring | Wazuh 4.14 (indexer + manager + dashboard + agent) |
| Object storage | SeaweedFS (S3-compatible) |
| Alerting | Discord webhooks (both performance and security channels) |

---

## Contents

| Document | Covers |
|---|---|
| [`Server_Security_Hardening_Report.md`](./Server_Security_Hardening_Report.md) | SSH authentication hardening, firewall rule review and cleanup, and a structural finding: Docker silently bypassing UFW's declared ruleset via `iptables` |
| [`Cloud_Security_Architecture_Analysis.md`](./Cloud_Security_Architecture_Analysis.md) | Annotated architecture diagram, network flow mapping, protection mechanisms, and gap analysis (inconsistent hardening across VMs, unauthenticated storage backend) |
| [`Performance_Monitoring_Report.md`](./Performance_Monitoring_Report.md) | PostgreSQL performance testing on a 2-million-row dataset — simple vs. joined query execution time, cross-checked against an existing Zabbix deployment |
| [`Wazuh_Security_Monitoring_Report.md`](./Wazuh_Security_Monitoring_Report.md) | Wazuh server + agent deployment, custom PostgreSQL log decoder, real security event generation with MITRE ATT&CK classification, and a genuine false-positive flood bug found and fixed in `rootcheck` |

---

## Key findings

| Finding | Report |
|---|---|
| Docker publishes container ports directly to `iptables`, bypassing UFW's `deny` policy entirely — confirmed via `iptables -t nat -L DOCKER` | Server Security Hardening |
| An `authorized_keys` entry with no corresponding private key on the current machine can create a false sense of access redundancy | Server Security Hardening |
| A "more mature" Debian base image is not automatically more secure — a direct A/B scan comparison showed the opposite in one case | Cloud Security Architecture |
| Zabbix's 1-minute polling interval misses short database query bursts entirely — a monitoring blind spot, not a misconfiguration | Performance Monitoring |
| Wazuh's `OS_Regex` engine silently fails to match `{n}` quantifier syntax (e.g. `\d{4}`) that looks valid by analogy with standard regex — requires literal repetition (`\d\d\d\d`) instead | Wazuh Security Monitoring |
| A 216,500-alert flood traced to two legacy `rootcheck` signature checks (`check_trojans`, `check_dev`) producing false positives against modern binaries and virtualized `/dev/` — root-caused and disabled at the source, not just filtered downstream | Wazuh Security Monitoring |

---

## Methodology

Each report follows the same structure: commands used → raw output/measurements → analysis. Where a first attempt failed or a hypothesis was wrong, that path is documented rather than smoothed over — the debugging chain is treated as part of the evidence, not noise to remove.

No credentials, tokens, or private keys are included in any document. Internal IP addresses have been replaced with placeholders (`<APP-VM-IP>`, `<ZABBIX-VM-IP>`, `<STORAGE-VM-IP>`) prior to publication.

---

## Author

**Natália Helen Ferreira**
PhD in Biological Chemistry | Data Engineer & AI (RNCP Level 7, in progress)
[LinkedIn](https://linkedin.com/in/ferreiranh) · [GitHub](https://github.com/NatyFerreira)

---

## License

MIT License — see [LICENSE](LICENSE) for details.
