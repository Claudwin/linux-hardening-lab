## Automated remediation

**Phase outcome:** 71.51% → 96.85%

OpenSCAP can generate a Bash remediation script directly from scan results. Generating from the results file rather than the datastream produces a script scoped to the rules that actually failed on this host, rather than every rule in the profile.

```bash
sudo oscap xccdf generate fix \
  --profile xccdf_org.ssgproject.content_profile_cis_server_l1 \
  --fix-type bash \
  reports/hardened-results.xml \
  > remediate.sh
```

The script was reviewed before execution and applied with output captured:

```bash
sudo ./remediate.sh 2>&1 | tee reports/remediation-output.log
sudo reboot
```

The reboot is required before re-scanning. Several remediations — kernel parameters, module blacklists, mount options — only take effect at boot, and scanning before rebooting would undercount the result.

### What this phase demonstrates, and what it does not

Applying a vendor-generated script that changes 100+ settings at once is the opposite of the procedure used in the manual phase, where each control was inspected, backed up, and individually verified. The honest claim is that the remediation was applied and the result measured — not that every change it made is understood.

In production this would be staged rather than applied directly: run against a non-production host first, review the generated diff, and identify controls that conflict with application requirements before touching anything that serves traffic. The script hardens PAM, password policy, file permissions, SELinux, and firewall rules, and any of those can break a working application.

### Verifying the outcome

Three checks after remediation, in order of usefulness:

```bash
uptime                                                    # confirm the reboot preceded the scan
sudo sshd -T | grep -E "permitrootlogin|maxauthtries"     # confirm effective config, not file contents
sudo grep -A3 'urn:xccdf:scoring:default' reports/remediated-results.xml
```

The middle check matters more than it appears. Both manually applied directives were removed from `/etc/ssh/sshd_config` by the remediation script and rewritten into `/etc/ssh/sshd_config.d/00-complianceascode-hardening.conf`. A `grep` of the main config file alone would return nothing and suggest the manual work had been reverted.

The `00-` prefix is deliberate on the tooling's part: because `sshd` honours the **first** occurrence of a directive and the `Include` sits at the top of the main config, a `00-` prefixed drop-in cannot be overridden by anything below it. `sshd -T` resolves the includes and reports the effective configuration, which is the authoritative answer.

### Second-order effects

The umask control (default `027`) changed the permissions applied to files created after remediation. Scan reports subsequently written under `sudo` were created `root:root` at mode `640`, which broke file transfer off the host as the ordinary user:

```
scp: remote open "reports/remediated-report.html": Permission denied
```

Resolved with `chown`, but worth recording: hardening changed the behaviour of a workflow that had been functioning, and it did so in a way unrelated to the control's stated purpose. This is the ordinary case rather than an unusual one, and it is the reason blanket remediation is staged rather than applied directly in production.
