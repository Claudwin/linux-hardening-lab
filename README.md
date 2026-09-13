# Linux Hardening Lab

CIS Benchmark Level 1 hardening of a Rocky Linux 9 server in Microsoft Azure, measured with OpenSCAP against the official SCAP Security Guide content. Controls were applied manually first, then via the vendor-generated remediation script, with compliance scored at each phase.

**Approach:** the manual phase is deliberately small. Its purpose is to build the understanding needed to reason about a control when it breaks something, gets challenged by an auditor, or conflicts with a business requirement — the situations where an automated fix is the start of the conversation rather than the end of it. The automated phase is how this work is actually done at scale. The gap between what each phase achieved is itself part of the finding.

---

## Compliance progression

| Phase | What changed | Score |
|---|---|---|
| **Baseline** [Report](https://raw.githack.com/Claudwin/linux-hardening-lab/refs/heads/main/reports/baseline-report.html) | Default Rocky Linux 9.8 image, unmodified | **71.47%** |
| **Manual — SSH access control** | Root SSH login disabled; authentication attempts capped at 4 | **71.51%** |
| **Manual — permissions, accounts, auditing** | No change — controls 03, 04, 07 verified already compliant | **71.51%** |
| **Automated remediation** | OpenSCAP-generated Bash script, 100+ rules, applied and rebooted | **96.85%** |

520 rules evaluated per scan. Two controls applied by hand moved the score 0.04 points; one generated script moved it 25. That contrast is the point — manual work builds the understanding, automation delivers the coverage, and they are different skills.

---

## Controls

| # | Control | CIS concern | File | State found | Status |
|---|---|---|---|---|---|
| [01](docs/methodology.md#control-01--disable-ssh-root-login) | Disable SSH root login | Access control | `/etc/ssh/sshd_config` | `prohibit-password` | ✅ Hardened → `no` |
| [02](docs/methodology.md#control-02--limit-ssh-authentication-attempts) | Limit SSH authentication attempts | Authentication | `/etc/ssh/sshd_config` | `#MaxAuthTries 6` (commented default) | ✅ Hardened → `4` |
| [03](docs/methodology.md#control-03--restrict-sshd_config-permissions) | Restrict `sshd_config` permissions | Access control | `/etc/ssh/sshd_config` | `600 root root` | ☑️ Verified compliant |
| [04](docs/methodology.md#control-04--verify-root-is-the-only-uid-0-account) | Verify root is the only UID 0 account | Access control | `/etc/passwd` | single UID 0 | ☑️ Verified compliant |
| [05](docs/methodology.md#note-on-controls-05-and-06--runtime-value-versus-persistent-configuration) | Disable IP forwarding | Network parameters | `/etc/sysctl.d/` | runtime `0`, no persistent declaration | ⚙️ Automated |
| [06](docs/methodology.md#note-on-controls-05-and-06--runtime-value-versus-persistent-configuration) | Enable TCP SYN cookies | Network parameters | `/etc/sysctl.d/` | runtime `1`, no persistent declaration | ⚙️ Automated |
| [07](docs/methodology.md#control-07--enable-audit-daemon) | Enable audit daemon | Logging & auditing | `auditd` | installed, enabled, active | ☑️ Verified compliant |

**Status key** — ✅ Hardened: non-compliant, changed by hand. ☑️ Verified compliant: already met, confirmed and documented. ⚙️ Automated: gap identified manually, remediated in the automated phase.

Rationale and command sequence for each: **[docs/methodology.md](docs/methodology.md)**

---

## What the manual phase surfaced

Three findings where the obvious check returns the wrong answer. Each is the kind of thing an automated fix handles silently, leaving nothing learned.

**A correct value is not an enforced value.** `MaxAuthTries` was commented out and `net.ipv4.ip_forward` was unset in every `sysctl` config file — yet both held compliant values at runtime, inherited as compile-time defaults. A kernel upgrade or a package dropping a file into `/etc/sysctl.d/` would reverse the posture with no corresponding change in configuration. This is why CIS checks the persistent declaration rather than the running value.

**Drop-in files override the main config, and order is inverted.** Rocky 9 places `Include /etc/ssh/sshd_config.d/*.conf` at the top of `sshd_config`, and `sshd` takes the **first** occurrence of a directive — the opposite of most configuration systems. A drop-in setting `PermitRootLogin yes` would silently defeat an edit to the main file.

**File-level verification is not effective-configuration verification.** After automated remediation, both hand-applied directives had vanished from `/etc/ssh/sshd_config` — a `grep` of that file alone would suggest the work had been reverted. The script had relocated them to `00-complianceascode-hardening.conf`, prefixed `00-` precisely so they cannot be overridden. `sshd -T` resolves includes and reports what the daemon is actually running:

```bash
sudo sshd -T | grep -E "permitrootlogin|maxauthtries"
permitrootlogin no
maxauthtries 4
```

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
| **[Methodology](docs/methodology.md)** | Change procedure, scan commands, per-control rationale, automated remediation |
| **[Problems encountered](docs/problems-encountered.md)** | Documented failures and root cause analysis |
| `reports/` | OpenSCAP HTML reports and XML results for all three scans |
| `remediate.sh` | Generated remediation script, as applied |
| `screenshots/` | Scan scores, config before/after, NSG rule |

---

## Skills demonstrated

Azure CLI VM provisioning · Network Security Group configuration · OpenSCAP compliance scanning and remediation · CIS Benchmark interpretation · SSH and kernel parameter hardening · Change management with backup and verified rollback · Linux root cause analysis · Technical documentation
