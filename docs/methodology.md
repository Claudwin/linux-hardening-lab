## Control 03 — Restrict `sshd_config` permissions

**CIS concern:** access control
**File:** `/etc/ssh/sshd_config`
**Required:** mode `600`, owned `root:root`
**Found:** `600 root root`
**Outcome:** verified compliant, no change made

The file controls who can access the system and how. World-readable permissions expose the SSH posture to any local user — which authentication methods are enabled, which users are permitted, whether root login is allowed. That is reconnaissance material. Write access by any non-root user would allow those rules to be changed outright.

```bash
stat -c "%a %U %G %n" /etc/ssh/sshd_config
```

**Drop-in directory.** Rocky 9 ships an `Include /etc/ssh/sshd_config.d/*.conf` directive at the top of `sshd_config`, so files in that directory form part of the effective configuration and fall under the same control:

```bash
sudo ls -la /etc/ssh/sshd_config.d/
```

Directory found at `700 root root`, containing `50-cloud-init.conf` and `50-redhat.conf`, both `600 root root`.

Order matters here in a way that is easy to get wrong: within `sshd_config`, **the first occurrence of a directive wins**, which is the opposite of most configuration systems. Because the `Include` sits at the top of the file, drop-ins take precedence over the main config below. A drop-in setting `PermitRootLogin yes` would have silently defeated the change made in control 01. Verifying the drop-in contents is therefore part of verifying the control, not an optional extra.

---

## Control 04 — Verify root is the only UID 0 account

**CIS concern:** access control
**File:** `/etc/passwd`
**Required:** exactly one account with UID 0
**Found:** `root` only
**Outcome:** verified compliant, no change made

UID 0 *is* root as far as the kernel is concerned — the account name is only a label. An account named something innocuous carrying UID 0 holds full root privilege while appearing ordinary in a casual user listing. This is a well-established persistence mechanism following a compromise.

```bash
awk -F: '($3 == 0) { print $1 }' /etc/passwd
```

No `sudo` required: `/etc/passwd` is world-readable by design, since normal operations such as `ls -l` need to resolve UIDs to names. The password hashes live in `/etc/shadow`, which is not world-readable.

**If a second UID 0 account were found**, the remediation would not be immediate deletion. The account would first be identified and its dependencies established — a running service authenticating as that account would break on removal. The fix is then either reassignment to a non-zero UID or removal, depending on what the account is for.

---

## Control 05 — Disable IP forwarding

**CIS concern:** kernel network parameters
**Required:** `net.ipv4.ip_forward = 0`, persistently configured
**Found:** runtime value `0`; no persistent declaration
**Outcome:** deferred — see note below

IP forwarding causes the host to pass traffic between interfaces, functioning as a router. A server that is not a router should not do this. Where it does, an attacker with a foothold on the host can use it to pivot into networks reachable from the host but not from their own position.

```bash
sysctl net.ipv4.ip_forward
sudo grep -rs "ip_forward" /etc/sysctl.conf /etc/sysctl.d/
```

---

## Control 06 — Enable TCP SYN cookies

**CIS concern:** kernel network parameters
**Required:** `net.ipv4.tcp_syncookies = 1`, persistently configured
**Found:** runtime value `1`; no persistent declaration
**Outcome:** deferred — see note below

A TCP handshake is SYN, SYN-ACK, ACK. A SYN flood sends large volumes of SYN packets and never completes the handshake, filling the connection table so that legitimate connections are refused. SYN cookies allow the kernel to encode connection state into the sequence number rather than allocating a table entry, leaving nothing to exhaust.

```bash
sysctl net.ipv4.tcp_syncookies
sudo grep -rs "tcp_syncookies" /etc/sysctl.conf /etc/sysctl.d/
```

---

### Note on controls 05 and 06 — runtime value versus persistent configuration

Both parameters hold the correct value at runtime, but neither is declared in `/etc/sysctl.conf` or anywhere under `/etc/sysctl.d/`. The values are kernel compile-time defaults — correct by inheritance rather than by policy.

This is the same pattern as `MaxAuthTries` in control 02: a correct value that nothing enforces. A kernel upgrade that changes a default, or any package dropping a file into `/etc/sysctl.d/` that sets `ip_forward = 1`, would reverse the system's posture with no corresponding change in its configuration and no obvious trace.

CIS checks the persistent configuration rather than the running value for exactly this reason, so **OpenSCAP will continue to report both rules as failing until they are explicitly declared**, notwithstanding the correct runtime values. These controls are therefore recorded as deferred rather than compliant, so that the documentation and the scan output agree.

Planned remediation:

```bash
sudo nano /etc/sysctl.d/60-cis-hardening.conf
sudo sysctl --system
```

The `60-` prefix places the file after the `50-` range conventionally used by distribution packages. Files under `/etc/sysctl.d/` are read in lexical order with later files taking precedence, so the prefix determines whether local policy overrides vendor defaults or is overridden by them.

---

## Control 07 — Enable audit daemon

**CIS concern:** logging and auditing
**Required:** `audit` package installed, `auditd` enabled and running
**Found:** `audit-3.1.5-8.el9.x86_64`, enabled, active
**Outcome:** verified compliant, no change made

`auditd` records security-relevant kernel events: file access, permission and ownership changes, privilege escalation, and configuration modification. Without it there is no forensic record of activity on the host, and no way to reconstruct what occurred after an incident.

```bash
rpm -q audit
systemctl is-enabled auditd
systemctl is-active auditd
```

The package is named `audit`; the service is `auditd`. Querying `rpm -q auditd` returns not-installed on a system where the daemon is present and running.

Enablement and running state are independent conditions: a service may be running now but not configured to start at boot, or configured to start at boot but currently stopped. Both must hold.
