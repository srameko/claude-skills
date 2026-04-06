---
name: system-hardening
description: >
  System hardening for Ubuntu servers following NIST and CISA guidelines.
  Use this skill whenever the user works on server hardening, SSH configuration,
  sysctl tuning, firewall rules (UFW), fail2ban, user management, or any
  security configuration of the OCI ARM or Intel VPS. Also trigger when
  reviewing or improving the hardening Ansible role, or when checking compliance
  against NIST SP 800-53 or CISA hardening guides. Always use this skill for
  security configuration work.
---

# System Hardening Skill

## Scope
Ubuntu 22.04 servers on OCI Free Tier (ARM primary, Intel secondary).
Hardening is applied via the `hardening` Ansible role in the infrastructure repo.
See Ansible skill for deployment workflow.

---

## Hardening Areas

### 1. SSH (`roles/hardening/tasks/ssh.yml`)

Config deployed to `/etc/ssh/sshd_config.d/99-hardening.conf`

**Current defaults:**
```yaml
ssh_port: 22
ssh_permit_root_login: "no"
ssh_password_authentication: "no"
ssh_pubkey_authentication: "yes"
ssh_max_auth_tries: 2
ssh_max_startups: "10:30:60"

ssh_kex_algorithms:
  - curve25519-sha256
  - curve25519-sha256@libssh.org
  - sntrup761x25519-sha512@openssh.com

ssh_ciphers:
  - chacha20-poly1305@openssh.com
  - aes256-gcm@openssh.com
  - aes128-gcm@openssh.com

ssh_macs:
  - hmac-sha2-512-etm@openssh.com
  - hmac-sha2-256-etm@openssh.com

ssh_hostkey_algorithms:
  - ssh-ed25519
  - rsa-sha2-512
  - rsa-sha2-256
```

**NIST/CISA alignment:**
- Disable root login → NIST AC-6 (least privilege)
- Key-only auth → NIST IA-2 (strong authentication)
- Modern algorithms only → NIST SC-8, SC-28 (transmission/data protection)
- MaxAuthTries 2 → NIST AC-7 (unsuccessful login attempts)

---

### 2. Kernel / sysctl (`roles/hardening/tasks/sysctl.yml`)

Config deployed to `/etc/sysctl.d/99-hardening.conf`

**Current settings + NIST mapping:**

| Setting | Value | NIST Control |
|---|---|---|
| `net.ipv4.tcp_syncookies` | 1 | SC-5 (DoS protection) |
| `net.ipv4.conf.all.rp_filter` | 1 | SC-7 (IP spoofing) |
| `net.ipv4.conf.default.rp_filter` | 1 | SC-7 |
| `net.ipv4.icmp_echo_ignore_broadcasts` | 1 | SC-5 |

**Recommended additions (CISA/NIST):**
```yaml
# Disable IP forwarding (unless routing)
net.ipv4.ip_forward: "0"
net.ipv6.conf.all.forwarding: "0"

# Ignore ICMP redirects
net.ipv4.conf.all.accept_redirects: "0"
net.ipv4.conf.default.accept_redirects: "0"
net.ipv6.conf.all.accept_redirects: "0"

# Disable source routing
net.ipv4.conf.all.accept_source_route: "0"
net.ipv4.conf.default.accept_source_route: "0"

# Log martian packets
net.ipv4.conf.all.log_martians: "1"

# Disable magic sysrq
kernel.sysrq: "0"

# Restrict dmesg to root
kernel.dmesg_restrict: "1"

# Restrict ptrace
kernel.yama.ptrace_scope: "1"

# Prevent core dumps with setuid
fs.suid_dumpable: "0"
```

---

### 3. Firewall (UFW) (`roles/ufw/`)

Default policy: deny incoming, allow outgoing.

Add rules only for required services:
```yaml
# Example ufw defaults
ufw_rules:
  - { rule: allow, port: 22, proto: tcp }   # SSH
  - { rule: allow, port: 80, proto: tcp }   # HTTP
  - { rule: allow, port: 443, proto: tcp }  # HTTPS
```

**NIST alignment:** SC-7 (boundary protection), CA-3 (network access control)

---

### 4. fail2ban (`roles/fail2ban/`)

Template: `jail.local.j2`

**Key jails to have active:**
- `sshd` — ban after failed SSH attempts
- Custom jails for nginx if exposed

**NIST alignment:** SI-3, AC-7 (intrusion detection, failed login lockout)

---

### 5. Admin Users (`roles/hardening/tasks/users.yml`)

```yaml
admin_user: admin
admin_user_shell: /bin/bash
admin_user_groups: [sudo]
admin_user_github: "srameko"   # SSH keys fetched from GitHub
```

SSH public keys are pulled from `https://github.com/srameko.keys` — no manual key management needed.

**NIST alignment:** AC-2 (account management), AC-6 (least privilege)

---

## Hardening Checklist (per new server)

- [ ] SSH: root login disabled, password auth disabled, modern algorithms only
- [ ] SSH: port changed from 22 if exposed to internet
- [ ] sysctl: syncookies, rp_filter, redirect/source route disabled
- [ ] UFW: default deny, only required ports open
- [ ] fail2ban: sshd jail active
- [ ] Admin user created, sudo group, SSH keys from GitHub
- [ ] Unattended upgrades enabled (security updates)
- [ ] No unnecessary services running (`systemctl list-units --type=service`)
- [ ] `/tmp` mounted noexec if possible
- [ ] Auditd installed and running (for NIST AU-2 audit events)

---

## NIST SP 800-53 Key Controls Reference

| Control | Area | Implementation |
|---|---|---|
| AC-2 | Account management | Admin user role, no shared accounts |
| AC-6 | Least privilege | No root login, sudo only |
| AC-7 | Failed login lockout | fail2ban, MaxAuthTries 2 |
| AU-2 | Audit events | auditd |
| IA-2 | Authentication | Key-only SSH |
| SC-5 | DoS protection | syncookies, ICMP broadcast ignore |
| SC-7 | Boundary protection | UFW, rp_filter |
| SC-8 | Transmission confidentiality | Modern SSH ciphers only |
| SI-2 | Flaw remediation | Unattended upgrades |
| SI-3 | Malware protection | fail2ban, limited attack surface |

---

## OCI-Specific Notes

- OCI has its own network security groups (NSGs) — UFW and OCI NSG must both allow traffic
- ARM instance (primary): nginx + Docker + GoPhish
- Intel instance (secondary): supplementary workloads
- OCI instances use `ubuntu` as default user — admin user is created on top
