# Server Security Hardening Report — Carburoam Application VM

## 1. SSH Configuration

### 1.1 Baseline (before hardening)

```bash
sudo sshd -T | grep -E "permitrootlogin|passwordauthentication|pubkeyauthentication|port|permitemptypasswords|maxauthtries"
```

| Setting | Value | Assessment |
|---|---|---|
| `port` | 22 | Default port — first target of automated scanners, but changing it is security-by-obscurity, not a real control; not changed here in favor of stronger authentication controls |
| `permitrootlogin` | `without-password` | Root login allowed with a key, not fully disabled |
| `pubkeyauthentication` | `yes` | Key-based auth already available |
| `passwordauthentication` | **`yes`** | Main weakness: password login accepted |
| `permitemptypasswords` | `no` | Already safe |
| `maxauthtries` | 6 | Acceptable but not tightened |

No overrides existed in `/etc/ssh/sshd_config.d/` prior to this exercise — all values above were implicit defaults, not explicit choices. This in itself was a finding: nothing had been deliberately configured.

### 1.2 Hardening applied

A dedicated override file was created (safer than editing the main `sshd_config` directly, and easier to review/revert):

```bash
sudo tee /etc/ssh/sshd_config.d/99-hardening.conf << 'EOF'
PasswordAuthentication no
PermitRootLogin no
PermitEmptyPasswords no
MaxAuthTries 6
X11Forwarding no
AllowTcpForwarding no
ClientAliveInterval 300
ClientAliveCountMax 2
EOF
sudo sshd -t          # syntax validation before applying
sudo systemctl restart sshd
```

| Setting | New Value | Rationale |
|---|---|---|
| `PasswordAuthentication` | `no` | Eliminates password brute-force as an attack vector entirely |
| `PermitRootLogin` | `no` | No direct root login, even with a key — forces privilege escalation via `sudo` from a named user, which is auditable |
| `PermitEmptyPasswords` | `no` | Unchanged, already safe |
| `MaxAuthTries` | 6 | Kept moderate after an incident (see 1.3) showed a too-low value can cause legitimate lockouts |
| `X11Forwarding` / `AllowTcpForwarding` | `no` | Disables unused features that expand the attack surface |
| `ClientAliveInterval` / `ClientAliveCountMax` | 300 / 2 | Idle sessions are automatically dropped after ~10 minutes of inactivity |

### 1.3 Incident during hardening — root cause and resolution

**What happened**: on the first hardening attempt (with `MaxAuthTries 3`), a *second* terminal session testing the new configuration was locked out with `Permission denied (publickey)` for both the personal user and `root`. The original, still-open session was used to immediately revert the change (`rm` the override file, restart `sshd`), following the standard safe-hardening practice of never closing the current session before validating a new SSH configuration.

**Root cause investigation**:
1. Initial hypothesis (SSH agent offering too many keys, tripping a low `MaxAuthTries`) was tested and ruled out — `ssh-add -l` showed the agent had no loaded identities at all.
2. A verbose connection attempt (`ssh -v`) revealed the actual cause: the client was offering `~/.ssh/id_ed25519`, but the server's `authorized_keys` only listed an **RSA** key under the comment `natyferreira@ubuntu` — a key whose **private half no longer existed on this Mac** (`ls ~/.ssh/id_rsa` → "No such file or directory"). This orphaned key had been authorized at some point in the past (on a different machine, or since deleted) and was never actually usable from the current workstation.
3. **Resolution**: the current, actively-used `id_ed25519` public key was appended to `authorized_keys` on the VM. A duplicate-entry artifact from a repeated paste was cleaned up with `sort -u`.
4. Hardening was reapplied and validated successfully (see 1.4).

**Lesson**: an `authorized_keys` file should be periodically audited for orphaned keys — a key with no corresponding usable private key anywhere gives a false sense of access redundancy while providing none in practice.

### 1.4 Post-hardening validation

```bash
# Confirm password auth is rejected
ssh -o PubkeyAuthentication=no natyferreira@ubuntu.orb.local
# → Permission denied (publickey)  ✔ expected

# Confirm key auth works
ssh -o IdentitiesOnly=yes -i ~/.ssh/id_ed25519 natyferreira@ubuntu.orb.local
# → prompts for the key's passphrase, connects successfully  ✔ expected

# Confirm effective configuration
sudo sshd -T | grep -E "passwordauthentication|permitrootlogin|permitemptypasswords|maxauthtries|x11forwarding|allowtcpforwarding|clientaliveinterval|clientalivecountmax"
```

Result:
```
maxauthtries 6
clientaliveinterval 300
clientalivecountmax 2
permitrootlogin no
passwordauthentication no
x11forwarding no
permitemptypasswords no
allowtcpforwarding no
```

All 8 hardening settings confirmed active.

---

## 2. Firewall Configuration

### 2.1 Baseline review

```bash
sudo ufw status verbose
```

Before this review, 6 rules were active (12 counting IPv6 duplicates): ports 22, 80, 443, 81, 10050/tcp, and 161/udp, all `ALLOW IN` from `Anywhere`.

### 2.2 Justification review — is each open port still needed?

| Port | Service | Still in use? | Evidence | Decision |
|---|---|---|---|---|
| 22/tcp | SSH | Yes | Active administration access | Keep |
| 80/tcp | HTTP (nginx-proxy-manager) | **No** | `docker ps -a` showed the nginx-proxy-manager container was already removed in a prior maintenance session (Iteration 2 cleanup); two unrelated stopped `nginx` containers (`Exited (0) 2 weeks ago`) were found and removed during this review, confirming nothing currently listens on this port | **Removed** |
| 443/tcp | HTTPS (nginx-proxy-manager) | No | Same as above | **Removed** |
| 81/tcp | nginx-proxy-manager admin UI | No | Same as above | **Removed** |
| 10050/tcp | Zabbix Agent | Yes | Actively polled by the Zabbix Server VM | Keep |
| 161/udp | SNMP | Yes | Actively polled for the custom SNMP process-count item | Keep |

### 2.3 Correction applied

```bash
sudo ufw delete allow 80/tcp
sudo ufw delete allow 443/tcp
sudo ufw delete allow 81/tcp
```

Result — final ruleset:
```
22/tcp       ALLOW IN    Anywhere
10050/tcp    ALLOW IN    Anywhere
161/udp      ALLOW IN    Anywhere
22/tcp (v6)      ALLOW IN    Anywhere (v6)
10050/tcp (v6)   ALLOW IN    Anywhere (v6)
161/udp (v6)     ALLOW IN    Anywhere (v6)
```

Three unjustified rules removed, reducing the declared attack surface from 6 to 3 distinct services.

### 2.4 Critical finding — Docker bypasses UFW entirely

While investigating why port **8501** (Carburoam's HTTP port, reachable from the Zabbix VM for the existing HTTP monitoring check) did **not** appear anywhere in the `ufw` ruleset despite clearly being reachable, the following was found:

```bash
sudo iptables -L DOCKER -n --line-numbers
sudo iptables -t nat -L DOCKER -n | grep 8501
```

```
ACCEPT tcp -- 0.0.0.0/0  172.22.0.2  tcp dpt:8501
DNAT   tcp -- 0.0.0.0/0  0.0.0.0/0   tcp dpt:8501 to:172.22.0.2:8501
```

**Explanation**: this is a well-documented, long-standing Docker behavior: any port published via `-p` (or a Compose `ports:` entry) is inserted by the Docker daemon directly into `iptables`'s `DOCKER` chain, which is evaluated **before** UFW's own rules. This means **UFW's `deny (incoming)` default policy does not apply to Docker-published ports at all** — they are exposed to `0.0.0.0/0` (any source) regardless of what UFW's ruleset shows.

**Practical risk in this environment**: currently low, since this VM is only reachable from the internal OrbStack network, not the public internet. However, this is a real, well-known class of misconfiguration: if this VM were ever given a public IP or exposed via port forwarding, port 8501 would be open to the entire internet with zero firewall protection, even though `ufw status` would never have listed it as an open rule — creating a dangerous false sense of security for anyone auditing the firewall alone without checking `iptables` directly.

**Recommended corrective action (documented, not applied in this session to avoid breaking the live Zabbix↔Carburoam HTTP check integration)**:
- Bind the container port to a specific internal interface/IP instead of all interfaces (e.g. `<APP-VM-IP>:8501:8501` instead of `8501:8501` in `docker-compose.yml`), restricting reachability to the internal network by design rather than by firewall rule; or
- Deploy `ufw-docker` (a maintained community tool that reconciles Docker's iptables rules with UFW's policy), so that UFW's ruleset becomes an accurate, trustworthy source of truth again.

---

## 3. Internal vs. External Access Summary

| Access Type | Services | Justification |
|---|---|---|
| **Internal only** (OrbStack private network, `192.168.x.x`) | Zabbix Agent (10050), SNMP (161), Carburoam HTTP (8501, via Docker-published port — see 2.4 finding) | These services are only meaningfully useful to other VMs on the same internal network (Zabbix Server polling the app VM); none are intended for public/internet access |
| **Administrative access** | SSH (22) | Required for human operator access; hardened to key-only authentication (Section 1) |
| **Removed / no longer applicable** | HTTP/HTTPS/nginx-admin (80, 443, 81) | Belonged to a decommissioned nginx-proxy-manager + WordPress stack, already removed at the container level in a prior maintenance pass; firewall rules had been left behind as stale artifacts until this review |

**Conclusion**: this review confirmed that firewall rules had drifted out of sync with actual running services (three rules for a decommissioned stack), and surfaced a structural finding independent of any specific misconfiguration — that Docker's own port-publishing mechanism silently bypasses UFW, meaning firewall audits on Docker hosts must always cross-check `iptables -L DOCKER` in addition to `ufw status` to get an accurate picture of real exposure.
