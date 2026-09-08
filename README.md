# Linux Hardening Lab

CIS Benchmark Level 1 hardening of a Rocky Linux 9 server in Microsoft Azure, measured with OpenSCAP against the official SCAP Security Guide content. Compliance is scored before and after each phase, and every manual control is applied by hand using a documented, reversible change procedure.

Ongoing project — the control set and compliance phases below expand as work progresses.

---

## Compliance progression

| Phase | What changed | Score |
|---|---|---|
| **Baseline** | Default Rocky Linux 9.8 image, unmodified | **71.47%** |
| **Manual — SSH access control** | Root SSH login disabled; authentication attempts capped at 4 | **71.51%** |
| **Manual — file permissions & accounts** | `sshd_config` ownership and mode; UID 0 account verification | *pending* |
| **Manual — kernel network parameters** | IP forwarding disabled; SYN cookies enabled | *pending* |
| **Manual — audit logging** | `auditd` installed and enabled | *pending* |
| **Automated remediation** | OpenSCAP-generated fix script applied to remaining rules | *pending* |

520 rules evaluated per scan. Two controls produce a fractional improvement — that is the expected arithmetic and is reported as measured. The manual phases demonstrate the change procedure against real controls; the automated phase is what moves the score materially.

---

## Controls

| # | Control | CIS concern | File | Before | After | Status |
|---|---|---|---|---|---|---|
| [01](docs/methodology.md#control-01--disable-ssh-root-login) | Disable SSH root login | Access control | `/etc/ssh/sshd_config` | `prohibit-password` | `no` | ✅ Implemented |
| [02](docs/methodology.md#control-02--limit-ssh-authentication-attempts) | Limit SSH authentication attempts | Authentication | `/etc/ssh/sshd_config` | `#MaxAuthTries 6` | `MaxAuthTries 4` | ✅ Implemented |
| 03 | Restrict `sshd_config` permissions | Access control | `/etc/ssh/sshd_config` | — | `600`, root-owned | ⬜ Planned |
| 04 | Verify root is the only UID 0 account | Access control | `/etc/passwd` | — | single UID 0 | ⬜ Planned |
| 05 | Disable IP forwarding | Network parameters | `sysctl` | — | `net.ipv4.ip_forward = 0` | ⬜ Planned |
| 06 | Enable TCP SYN cookies | Network parameters | `sysctl` | — | `net.ipv4.tcp_syncookies = 1` | ⬜ Planned |
| 07 | Enable audit daemon | Logging & auditing | `auditd` | — | installed, enabled | ⬜ Planned |
| 08 | Disable SSH password authentication | Authentication | `/etc/ssh/sshd_config` | — | `no` | ⬜ Planned |

Rationale and full command sequence for each implemented control: **[docs/methodology.md](docs/methodology.md)**

---

## Architecture

**Host**

| Component | Value |
|---|---|
| Platform | Microsoft Azure |
| Image | `resf:rockylinux-x86_64:9-base:9.8.20260525` |
| OS | Rocky Linux 9.8 (Blue Onyx) |
| Size | `Standard_B2s` (2 vCPU, 4 GB RAM) |
| Authentication | SSH key pair, password authentication disabled |

Image version pinned rather than `latest`, so the baseline is reproducible against a known OS build.

**Network**

SSH restricted at the Azure Network Security Group to a single source IP. The default rule created by `az vm create` opens port 22 to `0.0.0.0/0`; this was narrowed immediately after deployment. Network-layer control in front of host-layer controls — the two address different threats.

**Scanning**

| Component | Value |
|---|---|
| Scanner | `openscap-scanner` (`oscap`) |
| Content | `scap-security-guide` |
| Datastream | `ssg-rl9-ds.xml` |
| Profile | `xccdf_org.ssgproject.content_profile_cis_server_l1` |

Scanner and policy content are separate packages — `oscap` is the evaluation engine and ships with no policies of its own. The profile is the RHEL 9 CIS Level 1 Server benchmark; Rocky is a downstream RHEL rebuild, so the content applies directly. Level 1 over Level 2 because it is the baseline profile intended not to disrupt normal system function.

---

## Documentation

| Document | Contents |
|---|---|
| **[Methodology](docs/methodology.md)** | Five-step change procedure, scan commands, per-control rationale |
| **[Problems encountered](docs/problems-encountered.md)** | Documented failures and root cause analysis |
| `reports/` | Raw OpenSCAP HTML reports and XML results for each scan |
| `screenshots/` | Scan scores, config before/after, NSG rule |

---

## Skills demonstrated

Azure CLI VM provisioning · Network Security Group configuration · OpenSCAP compliance scanning · CIS Benchmark interpretation · SSH hardening · Change management with backup and verified rollback · Linux root cause analysis · Technical documentation
