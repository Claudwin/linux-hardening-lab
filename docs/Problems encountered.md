# Problems encountered

> Failures hit during this build, recorded with what was observed, what was concluded, and what was actually true. Added to as work progresses.

---

## 1. NSG rule accepted a valid address of the wrong family

**Observed:** SSH to the VM timed out rather than being refused.

**Diagnosis:** A timeout indicates a dropped packet at a filtering layer; a refusal indicates the packet reached the host and something declined it. That distinction pointed at the Network Security Group rather than at `sshd`.

**Root cause:** The NSG rule restricting SSH had been populated from `curl https://ifconfig.me`, which resolved over IPv6 and returned an IPv6 address. The SSH connection was made to the VM's IPv4 address and therefore originated from the client's IPv4 address, which matched nothing in the rule.

The rule was syntactically valid and Azure enforced it exactly as written. Nothing in the tooling flags an IPv6 source prefix on a rule governing an IPv4 path.

**Fix:** Force IPv4 resolution and inspect the value before applying it.

```bash
MYIP=$(curl -s -4 https://ifconfig.me)
echo "Setting NSG source to: $MYIP"
az network nsg rule update -g <rg> --nsg-name <nsg> \
  --name default-allow-ssh --source-address-prefixes $MYIP
```

**Prevention:** Never populate a firewall rule from a variable without echoing it first.

---

## 2. Connectivity test run from the wrong network position

**Observed:** After hardening `sshd`, a test SSH connection timed out. This was read as evidence the change had broken remote access, and the configuration was rolled back from backup.

**What was actually true:** The change was working correctly. It had passed `sshd -t`, the restart had succeeded, and both values had been confirmed by `grep`. A working configuration was reverted for no reason.

**Root cause:** The test was run from a shell **on the VM itself**, connecting to the VM's own public IP. That traffic egressed to Azure's network and returned to the NSG from the VM's address, which is not on the allow list, and was dropped.

**Fix:** Re-apply the configuration and test from the administrative workstation, whose address is on the allow list.

**Prevention:** A connectivity test is only meaningful when run from the network position that real users occupy. Testing from inside the host answers a different question than the one being asked, and here it produced a false negative that caused a correct change to be reverted.

---

## 3. Case corruption introduced during cross-platform authoring

**Observed:** A script drafted on macOS and transferred to the Linux host contained capitalisation introduced by the editor — `DATE` for `date`, `REPORTS` for `reports`, and uppercase terminators in ANSI escape sequences.

**Why it wasn't caught:** `bash -n` passed the file cleanly. Syntax validation confirms a script parses; it does not confirm that the commands exist or that the values are meaningful. Linux command and path names are case-sensitive, so the file would have failed at runtime despite validating.

**Prevention:** Author files on the target platform where practical. Where cross-platform authoring is necessary, grep for known-suspect patterns after transfer rather than relying on syntax validation alone.

---

## 4. Miscounting scan results by parsing XML with grep

**Observed:** Rule counts taken with `grep -c 'result>pass'` against the results XML differed implausibly between two scans of the same system — roughly half the rules appeared to vanish after a two-control change.

**Diagnosis:** A number that moves that far in response to that small a change indicates the measurement broke, not the system. The instinct to verify rather than record the number was correct even though the specific hypothesis (a truncated scan) was wrong.

**Root cause:** The results file embeds the full datastream including all profile definitions, not only the profile evaluated. A naive line count is not a reliable measure of rule outcomes.

**Fix:** Read the `<score>` element written by the scanner, which is the authoritative value.

**Prevention:** Don't parse structured data with line-oriented tools when the format provides an authoritative field.