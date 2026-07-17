# Cloud Security Architecture — Annotated Analysis

## 1. Architecture identified

Three isolated VMs (OrbStack), each with a distinct role:

| VM | IP | Role |
|---|---|---|
| `ubuntu` (application) | <APP-VM-IP> | Runs the Carburoam Docker container, the Zabbix Agent2, and the SNMP daemon |
| `zabbix-server` (monitoring) | <ZABBIX-VM-IP> | Runs Zabbix Server, PostgreSQL (Zabbix's own database), Apache + PHP-FPM (web frontend) |
| `s3-storage` (backup) | <STORAGE-VM-IP> | Runs SeaweedFS, exposing an S3-compatible API and the `carburoam-backups` bucket |

This is a deliberate isolation choice — each major service runs on its own VM rather than being co-located on the application host, so a compromise or resource exhaustion on one component doesn't directly threaten the others.

## 2. Network flows represented

| Flow | Direction | Port / Protocol | Purpose |
|---|---|---|---|
| Operator → app VM | External → internal | TCP 22 (SSH, key-only) | Human administrative access |
| App VM → storage VM | Internal → internal | HTTP :8333 (S3 API) | Daily backup of `db.sqlite3` |
| App VM ↔ Zabbix VM | Internal ↔ internal | TCP 10050 (Agent), UDP 161 (SNMP) | Metric polling |
| Zabbix VM → Discord | Internal → external | HTTPS (webhook) | Alert delivery on trigger |

No flow in this architecture is directly reachable from the public internet — all VMs sit on the internal OrbStack network. The only path in from outside is SSH to the application VM.

## 3. Protection mechanisms identified and analyzed

| Mechanism | Where | Analysis |
|---|---|---|
| **SSH key-only authentication** | App VM | Confirmed working (Section 1 of the server hardening report); eliminates password brute-force as an attack vector for the only VM reachable from outside the internal network |
| **UFW firewall, minimal ruleset** | App VM | Reduced to exactly 3 justified rules (22, 10050, 161) after removing 3 stale rules for a decommissioned stack — see hardening report |
| **`PermitRootLogin no`** | App VM | Direct root login disabled; privilege escalation must go through a named, auditable user via `sudo` |
| **Network segmentation (per-service VM)** | All 3 VMs | Limits blast radius: compromising the app container does not directly grant access to the Zabbix database or the backup store, since those are separate hosts |
| **Zabbix + Discord alerting** | Zabbix VM | Provides detection (not prevention) — an active trigger notifies quickly if a monitored service goes down or a resource threshold is crossed |

### Gap identified: **only the application VM was hardened**

The SSH/firewall hardening performed in this exercise was applied **only to the `ubuntu` VM**. The `zabbix-server` and `s3-storage` VMs were never audited for the same baseline (explicit SSH configuration, UFW rule justification). This is a real, current gap in this architecture, not a hypothetical one.

### Gap identified (carried over from the hardening report): **Docker bypasses UFW**

As documented during the SSH/firewall hardening exercise, any port published via Docker's `-p` flag (or Compose `ports:`) is inserted directly into `iptables`, bypassing UFW's declared ruleset entirely. This affects port 8501 (Carburoam's HTTP port) on the app VM, and would equally affect any Docker-published port on the storage VM (SeaweedFS's 8333/9333/8080) if that VM is not separately checked with `iptables -L DOCKER`.

### Gap identified: **no authentication on the S3-compatible storage**

Per the SeaweedFS PoC report, the storage service currently accepts arbitrary access key/secret pairs (`test`/`test` was used and worked). Combined with the fact that this VM was never SSH/firewall-hardened, the backup store is currently the weakest link in this architecture — anyone who can reach `<STORAGE-VM-IP>:8333` on the internal network can read or write the Carburoam backups without any credential check.

### Gap identified: **no encryption in transit**

None of the internal flows (SSH excluded) use TLS — the S3 API, the Zabbix agent/SNMP traffic, and the Discord webhook call (which does use HTTPS externally, but the Zabbix server itself doesn't authenticate that outbound connection with anything beyond the webhook URL as a shared secret) are not uniformly encrypted internally. This is a lower-severity finding given the traffic stays on an internal network, but it means anyone with access to that network segment could observe traffic in plaintext.

## 4. Recommendations for improvement

| Priority | Recommendation | Addresses |
|---|---|---|
| High | Apply the same SSH hardening (key-only auth, no root login, minimal UFW ruleset) to `zabbix-server` and `s3-storage`, not just the app VM | Gap: inconsistent hardening across the architecture |
| High | Configure real authentication on SeaweedFS (replace the placeholder `test`/`test` credentials with generated, rotated keys), and restrict bucket access via SeaweedFS's identity/policy configuration | Gap: unauthenticated backup store |
| Medium | Audit the storage VM for the same Docker/UFW bypass issue found on the app VM (`iptables -L DOCKER` on `s3-storage`), and apply the same mitigation (bind published ports to the internal interface only, or deploy `ufw-docker`) | Gap: Docker silently bypassing the firewall |
| Medium | Add a Zabbix agent to the storage VM (not yet deployed) so it is monitored with the same rigor as the other two VMs — currently it is a blind spot in the supervision architecture proposed in the S3 migration study | Consistency with the existing monitoring architecture |
| Low | Evaluate adding TLS for internal service-to-service traffic (S3 API in particular, since it carries backup data) if this architecture is ever extended beyond a closed lab network | Gap: no encryption in transit |

## 5. Summary

The three-VM segmentation itself is a sound architectural choice — it limits the blast radius of any single compromise and mirrors good production practice (separating application, monitoring, and storage concerns). However, the security posture is currently **uneven**: the application VM received deliberate hardening and a documented audit trail, while the monitoring and storage VMs were deployed functionally but never subjected to the same review. The most urgent gap is the storage VM's lack of authentication, since it holds the only backup copy of the application's data — an architecture whose backup layer is the least protected component defeats much of the purpose of having backups in the first place.
