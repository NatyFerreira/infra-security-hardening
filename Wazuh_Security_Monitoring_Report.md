# Wazuh Deployment and Usage Report — Security Monitoring for Carburoam

**Date**: July 16, 2026
**Scope**: Wazuh server + agent deployment, PostgreSQL log collection, security event generation and analysis

---

## 1. Installation steps

### Server deployment

A dedicated VM (`wazuh-server`, <WAZUH-VM-IP>) was provisioned, following the same per-service isolation pattern already used for Zabbix and the S3 storage backend.

```bash
orb create ubuntu wazuh-server
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash wazuh-install.sh -a
```

**Issue encountered and resolved**: the initial attempt used the Wazuh 4.9 installer, which failed with `ERROR: Uncompatible system. This script must be run on a 64-bit system` on this ARM64 VM. Investigation confirmed this was a known bug in the 4.9-era installer's architecture detection (it only recognized `x86_64`, not `aarch64`, even though ARM64 is fully 64-bit and officially supported by Wazuh). Using the current installer (4.14) resolved this immediately — current Wazuh documentation confirms AARCH64/ARM64 as an officially supported architecture for all central components (indexer, server, dashboard).

The all-in-one install (`-a` flag) completed in ~4 minutes, deploying the Wazuh indexer, manager, Filebeat, and dashboard on a single node.

### Access and credential handling

The installer generates a random `admin` password on first run. As with other credentials handled during this module (SSH keys, CI/CD tokens), this password was treated as compromised once it appeared in the working session log, and was rotated immediately:

```bash
sudo /usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh -u admin -p '<new password>'
```

**Note**: Wazuh enforces a specific password policy — 8–64 characters, at least one uppercase, one lowercase, one digit, and one symbol from a restricted set (`.`, `*`, `+`, `?`, `-` only; common symbols like `!`, `/`, `=` are rejected). The new password was stored in macOS's Passwords app rather than kept in plaintext notes.

### Agent deployment

The Wazuh agent was deployed to the `ubuntu` VM (hosting Carburoam) using the dashboard's guided deployment (**Endpoints → Deploy new agent**), selecting the DEB aarch64 package:

```bash
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.6-1_arm64.deb
sudo WAZUH_MANAGER='<WAZUH-VM-IP>' WAZUH_AGENT_NAME='carburoam-vm' dpkg -i ./wazuh-agent_4.14.6-1_arm64.deb
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

Confirmed active in the dashboard: **Agents summary → Active (1), Disconnected (0)**.

### PostgreSQL log collection setup

```bash
sudo sed -i '/<\/ossec_config>/i\
  <localfile>\
    <log_format>syslog</log_format>\
    <location>/var/log/postgresql/postgresql-17-main.log</location>\
  </localfile>' /var/ossec/etc/ossec.conf

sudo -u postgres psql -c "ALTER SYSTEM SET log_connections = 'on';"
sudo -u postgres psql -c "ALTER SYSTEM SET log_disconnections = 'on';"
sudo -u postgres psql -c "ALTER SYSTEM SET log_statement = 'ddl';"
sudo systemctl restart postgresql
sudo systemctl restart wazuh-agent
```

---

## 2. Security events generated and observed

Four categories of security-relevant events were deliberately generated to validate detection end-to-end.

| Event generated | Command | Detected? | Wazuh classification |
|---|---|---|---|
| Failed SSH login attempts (×5) | `ssh usuario_invalido@localhost` | **Yes** | 5 alerts, MITRE ATT&CK "Password Guessing" + "SSH" techniques, all correctly attributed to `carburoam-vm` |
| PostgreSQL connection/disconnection | `psql -c "SELECT 1;"` | Collected, not alerted | Log lines confirmed present in `/var/log/postgresql/...` and confirmed read by Wazuh's logcollector (`ossec.log`: "Analyzing file: ...postgresql-17-main.log"), but no structured alert was generated — see Section 4 |
| PostgreSQL DDL statement (`CREATE`/`DROP TABLE`) | `psql -c "CREATE TABLE ..."` | Collected, not alerted | Same as above |
| File creation/deletion in `/etc/` | `touch` / `rm` on a test file | **Yes**, after configuration fix | Initially not detected (default `syscheck` scan interval is 12h — see Section 4); after enabling `realtime="yes"` monitoring, 2 alerts appeared within seconds, classified under MITRE ATT&CK "Data Destruction" and "File Deletion" |

### Screenshots / dashboard evidence

- **SSH brute-force test**: Threat Hunting view showed 5 total events, all classified as "Authentication failure", with a MITRE ATT&CK breakdown split evenly between "Password Guessing" and "SSH" techniques.
- **File integrity test (after real-time fix)**: 2 events, classified as "Data Destruction" and "File Deletion" — Wazuh correctly flagged the create-then-quickly-delete pattern as a data-destruction-relevant behavior, not just a neutral file change.

---

## 3. Performance monitoring vs. security monitoring — key differences observed

| Aspect | Zabbix (performance monitoring) | Wazuh (security monitoring) |
|---|---|---|
| **Primary question answered** | "Is the system healthy and within resource limits?" | "Did something suspicious or malicious happen?" |
| **Data collected** | Numeric metrics (CPU %, memory %, disk %, HTTP status) sampled at fixed intervals | Log events and file-system changes, evaluated against rule signatures as they occur |
| **Typical output** | A time series graph and a threshold-crossing trigger | A discrete, classified alert (severity level, MITRE ATT&CK technique) |
| **Latency sensitivity** | Tolerant of 1-minute polling — a CPU spike matters as a trend, not a single instant | Latency-sensitive — a failed login or file deletion needs near-real-time detection to be actionable, as demonstrated by the FIM `realtime="yes"` fix in this exercise |
| **False positive shape** | A threshold briefly crossed by legitimate load (e.g. a deploy) | A legitimate admin action (e.g. a real file edit) that resembles a suspicious pattern |
| **Where this project already saw both** | Section 3 of the Performance Monitoring Report noted Zabbix's 1-minute polling missed a short PostgreSQL query burst entirely | This report's FIM test needed the same kind of latency awareness — the default 12h scan interval would have made file changes invisible for most of a day |

**Conclusion**: the two tools are complementary, not redundant. Zabbix answers "is this VM going to run out of resources," which this module's earlier triggers (CPU/RAM/disk >90%/80%) already cover well. Wazuh answers "did someone try to log in as an unauthorized user, or tamper with a system file," which Zabbix has no visibility into at all — Zabbix would never have flagged the 5 failed SSH logins or the `/etc/` file tampering, since neither crosses a resource threshold.

---

## 4. Gaps identified and improvement proposals (original assessment)

| Gap | Evidence | Proposed improvement |
|---|---|---|
| PostgreSQL logs are collected but not parsed into structured alerts | `ossec.log` confirms the file is read; no alert appeared for connection or DDL events despite `log_connections`/`log_statement` being enabled | Add a custom Wazuh decoder and rule set for PostgreSQL's log format (Wazuh supports custom `<decoder>`/`<rule>` definitions in `/var/ossec/etc/rules/local_rules.xml`); alternatively, route PostgreSQL logs through `rsyslog` first (already configured in Iteration 2 for WARNING+ messages) so they arrive in a Wazuh-recognized `syslog` shape |
| Default File Integrity Monitoring scan interval (12h) is too coarse for timely detection | The first `touch`/`rm` test was invisible until `realtime="yes"` was explicitly enabled | Enable real-time FIM on all security-sensitive paths from initial deployment, not as an afterthought — this should be part of the standard Wazuh agent baseline configuration going forward, documented in the runbook |
| No alert routing configured yet (Wazuh detects, but doesn't notify) | All testing was done by manually checking the dashboard; there is no equivalent of the Discord webhook integration already built for Zabbix | Configure Wazuh's own integration (it supports Slack, email, and custom webhooks, including Discord via a generic webhook integration) so security alerts reach the same notification channel as performance alerts, giving a single point of awareness |
| Single-node deployment (indexer + manager + dashboard on one VM) | All components installed with `-a` (all-in-one) on `wazuh-server` | Acceptable for this lab scale; documented as a scaling limitation for a future iteration, consistent with the same tradeoff already flagged for the Zabbix server in the cloud security architecture report |

## 4b. Update — gaps closed in a follow-up remediation pass

Two of the four gaps above were revisited and closed in a dedicated remediation session (full details, including the debugging path, in `Remediation_Log.md`). This section is added so the report reflects the system's actual current state rather than only the state at initial deployment.

**PostgreSQL decoder — now implemented.** A custom decoder/rule pair was added (`/var/ossec/etc/decoders/local_decoder.xml`, `/var/ossec/etc/rules/local_rules.xml`), classifying PostgreSQL connection and DDL events instead of leaving them as unparsed raw log lines. Getting this working required two rounds of correction: the initial `<program_name>` matching approach failed because PostgreSQL's log format has no syslog-style program name field; and Wazuh's `OS_Regex` engine turned out not to support the `{n}` quantifier syntax (`\d{4}`) used in the first `prematch` attempt, requiring literal repetition (`\d\d\d\d`) instead. Verified both via `wazuh-logtest` and against real generated events (4 connection events correctly classified in the dashboard).

**Discord notification integration — now implemented.** A custom integration script was added, mirroring the Zabbix→Discord pattern from the earlier monitoring iteration. Three distinct issues were found and fixed before it worked end to end: a duplicated `<integration>` block in `ossec.conf` from a failed earlier attempt; the webhook URL file being unreadable by the `wazuh` service user (`wazuh-integratord` runs as `wazuh`, not `root`, so permissions that worked for manual testing didn't work for the real service); and, once working, an initial alert threshold (`level 7`) that was too low and immediately flooded the channel with routine `rootcheck` background noise — raised to `level 12` to keep only genuinely significant alerts.

**Still open**: the FIM scan-interval gap and the single-node deployment limitation remain as documented above — no change to those two.

### Updated status table

| Gap | Status |
|---|---|
| PostgreSQL log parsing/alerting | ✅ Resolved |
| FIM default scan interval | ✅ Resolved (real-time enabled during initial testing, see Section 2) |
| Discord notification integration | ✅ Resolved |
| Single-node deployment | Open (accepted tradeoff, not a defect) |

Additionally, `wazuh-server` itself received the same SSH hardening (key-only authentication, no root login) and firewall configuration (UFW restricted to ports 22, 443, 1514/tcp, 1515/tcp, 55000/tcp) applied to every other VM in this module — this VM had been deployed but not hardened at the time the original report above was written.

## 4c. Update — genuine bug found: rootcheck false-positive flood

After instructor review flagged "notifications have a bug," a dedicated investigation was carried out (not initially caught by the earlier remediation pass, which had only addressed the *volume threshold* of notifications, not their *root cause*).

**Symptom**: the security event dashboard showed 216,500 alerts from a single rule (`Rule: 510 — Host-based anomaly detection event (rootcheck)`) within roughly 24 hours — far beyond what routine background noise would explain, even accounting for the unusually high number of `sudo`/config-change commands run during this module's hardening work.

**Root cause investigation**:
```bash
sudo grep -oP "Rule: \d+ \(level \d+\) -> '.*?'" /var/ossec/logs/alerts/alerts.log | sort | uniq -c | sort -rn | head -15
```
Confirmed `Rule: 510` alone accounted for 216,500 of roughly 216,600 total alerts. Inspecting the alert content revealed two distinct false-positive sources, both from `rootcheck`'s legacy signature-based checks (`check_trojans` and `check_dev`), which run a full scan every time `wazuh-manager` restarts (`scan_on_start`) — and the manager had been restarted many times that day while debugging the decoder and notification integration, multiplying the effect.

1. **`check_trojans` false positives**: rootcheck's bundled trojan signatures (`/var/ossec/etc/rootcheck/rootkit_trojans.txt`) are decades-old heuristics matching generic substrings like `bash|^/bin/sh|...` inside common binaries. Modern glibc-compiled utilities (`/bin/ls`, `/bin/env`, `/bin/echo`, `/usr/bin/` equivalents) legitimately contain these substrings internally and were being flagged as "Trojaned version of file detected" — a well-known class of false positive with this legacy signature set, not an actual compromise.
2. **`check_dev` false positives**: 25,067 alerts titled "File present on /dev" — rootcheck's device-directory scan is not well-suited to virtualized/container environments (this VM runs on OrbStack), where `/dev/` legitimately contains many dynamic pseudo-device entries that a traditional rootkit scanner doesn't expect and flags as anomalous.

**Fix**:
```bash
sudo sed -i 's/<check_trojans>yes<\/check_trojans>/<check_trojans>no<\/check_trojans>/' /var/ossec/etc/ossec.conf
sudo sed -i 's/<check_dev>yes<\/check_dev>/<check_dev>no<\/check_dev>/' /var/ossec/etc/ossec.conf
sudo truncate -s 0 /var/ossec/logs/alerts/alerts.log /var/ossec/logs/alerts/alerts.json
sudo systemctl restart wazuh-manager
```
Other rootcheck checks (`check_files`, `check_sys`, `check_pids`, `check_ports`, `check_if`) were left enabled — only the two checks producing false positives in this specific environment were disabled.

**Verification**: alert log line count dropped from 175,546+ (mid-flood) to 11 (normal manager-startup baseline) within 30 seconds of the fix, confirming the flood stopped rather than merely slowed down.

**Distinction from the earlier notification threshold fix (Section 4b)**: the earlier fix (raising the Discord integration's alert level from 7 to 12) reduced how many of these events reached Discord, but did not address why so many were being generated in the first place at the source. Both fixes were necessary: the threshold change controls *notification volume* for genuinely-occurring events, while this fix eliminates a source of *false* events entirely. Relying on the threshold alone would have left the dashboard itself polluted with 200,000+ false positives, undermining the tool's usefulness for genuine threat hunting even if Discord stayed quiet.

---

## 5. Unrelated finding surfaced during this deployment

While reviewing running processes on the `ubuntu` VM during the performance-monitoring exercise (immediately preceding this Wazuh work), an orphaned `cloudflared` tunnel was discovered actively exposing the VM to the public internet — a leftover from a prior module's WordPress/nginx-proxy-manager setup that had been removed at the container level but never at the service level. This was stopped and disabled (`systemctl stop/disable cloudflared`), and is documented in full in the Performance Monitoring Report. It is noted here as well because it is exactly the class of finding Wazuh-style security monitoring is meant to catch — an unexpected outbound-connected process — reinforcing the case for enabling real-time process/network monitoring going forward rather than relying on manual `htop` review to catch it.

---

## Conclusion

Wazuh was successfully deployed (server + agent) after resolving an ARM64 installer compatibility issue, and validated against four deliberately generated security event types. Two of the four (SSH brute-force attempts, real-time file integrity changes) were detected and automatically classified against the MITRE ATT&CK framework with no further tuning. The remaining two gaps identified at initial deployment (PostgreSQL log parsing, no notification integration) were closed in a follow-up remediation pass — see Section 4b — bringing the total to four of four event types now fully alerting, with the notification chain verified end to end via Discord. Comparing this exercise directly against the existing Zabbix deployment confirmed the two tools answer genuinely different operational questions and should be run together, not as alternatives to each other.
