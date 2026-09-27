# Findings

Each of these was found because something broke, or because the evidence
disagreed with what had been written down — not because a planned check passed.
Roughly in the order they changed the design.

---

## 1. The fix for the oldest switches was stricter than it needed to be

The first attempt to reach the fleet from the control node failed on three of
the five SSH devices, before any password was sent:

```text
paramiko 5.0.0 -> 10.99.20.1  (3560CG-2, IOS 15.2)  no acceptable kex algorithm
paramiko 5.0.0 -> 10.99.20.20 (C2940,    IOS 12.1)  no acceptable kex algorithm
```

The two sides never agreed on a key exchange. What these switches offer,
captured with a transport that stops after key exchange and never logs in:

```text
C2940      kex: diffie-hellman-group1-sha1                                          cipher agreed: 3des-cbc
3560CG-2   kex: diffie-hellman-group-exchange-sha1, group14-sha1, group1-sha1       cipher agreed: aes128-ctr
3560CG-1   kex: diffie-hellman-group-exchange-sha1, group14-sha1, group1-sha1       cipher agreed: aes128-cbc
```

Every option is SHA-1, and `paramiko` 5 has removed every SHA-1 key exchange
class — not disabled by default, deleted, so there is nothing to re-enable. The
two "identical" 3560s aren't identical here either: the 15.0 image can't do CTR
mode.

The fix was `paramiko<3`. Then `pip install ncclient` quietly upgraded paramiko
back to 5.0, because `ncclient` 0.7 requires `paramiko>=3.2`, so `ncclient` was
pinned below 0.7 as well. Both pins went into
[`requirements.txt`](../requirements.txt) so a rebuild couldn't repeat it. The
alternative — OpenSSH with legacy algorithms for three devices — would have meant
Ansible's `raw` module there, and giving up idempotence, diff and check mode on
half the fleet.

**A few days later, the pin turned out to be wrong.** It had only ever been
tested against 5.0 and 2.12. Tested against the versions in between:

```text
paramiko 3.5.1  SHA-1 kex offered by default   C2940 OK/3des-cbc   3560CG-2 OK/aes128-ctr   3560CG-1 OK/aes128-cbc
paramiko 4.0.0  SHA-1 kex offered by default   C2940 OK/3des-cbc   3560CG-2 OK/aes128-ctr   3560CG-1 OK/aes128-cbc
```

Only 5.0 broke them. So `paramiko<3` was costing two things it never needed to:
the fix for Terrapin (CVE-2023-48795, added in 3.4) on every connection, and a
current `ncclient`. `>=3.4,<5` satisfies both constraints. It stays unchanged
until a full playbook run proves it, because a handshake is not a playbook.

One more prediction didn't survive contact with the evidence. `paramiko` prints a
deprecation warning at import saying `TripleDES` "will be removed" from the
`cryptography` package in 48.0.0, which would have left the C2940 — whose only
cipher is 3DES — unreachable from Python. The control node runs `cryptography`
50.0.1, and 3DES is still there. The warning was an intention, not a date.

*A pin records what was tested, not what is true. Write down what it costs, so
somebody has a reason to test it again.*

---

## 2. NETCONF rode the port that was already open

Ansible's Junos modules need NETCONF, not a CLI session. Port 830 timed out —
which, on Junos, is what a filter drop looks like: the device discards silently
rather than refusing.

Opening 830 would have meant adding a second management protocol to the permit
list of the device that terminates the site-to-site IPsec tunnel. But
`set system services netconf ssh` also exposes NETCONF as an SSH **subsystem on
port 22**, which was already permitted:

```bash
ssh -s casey@10.99.0.2 netconf      # returns a <hello> - NETCONF over port 22
```

So the inventory sets `ansible_port: 22` for the `junos` group, with no firewall
change and no new exposure. The hello message also showed the AAA design at
work, over a protocol that design never mentions:

```text
<!-- user netadmin, class j-super-user -->
```

Logged in as a named AD user; the TACACS+ server mapped them to a local template
user; Junos applied that template's login class.

*Check whether a capability rides a path that's already permitted before opening
a new one.*

---

## 3. Uniform failure across independent platforms meant the tests were wrong

The first audit failed the same four checks on every IOS and EOS device:

```text
3560CG-1         8/12   Login authenticates to TACACS+, Commands are authorized by TACACS+, Commands are accounted to TACACS+, Syslog...
3560CG-2         8/12   (same four)
C2940-LAB        8/12   (same four)
ARISTA710P-LAB   5/9    (same four)
```

The EOS and IOS checks live in different files, with different regexes. For
both to fail identically, either the whole fleet was misconfigured the same way
across two vendors, or the tests shared a wrong assumption.

The evidence against the fleet was the audit itself: it had authenticated to
every device through TACACS+ to read the configurations. The devices used a
**named** server group — `aaa authentication login default group TAC-GROUP
local` — and the regexes demanded the literal keyword `group tacacs+`.

The group name moved into the baseline, so the check now also asserts that every
device uses the same group. One of the four was genuine: nothing forwarded
syslog, which became the second remediation.

*Before fixing a fleet, ask whether one wrong test explains the failure better
than many wrong devices.*

---

## 4. The gate passed a device it never audited

`-e compliance_gate=true` is meant for CI: exit non-zero if anything is
non-compliant. Its first version asserted that each device had no failed checks.

During the same first run, the SRX failed to return its configuration at all:

```text
SRX345-LAB       0/0
```

Zero failures out of zero checks. The assertion was true, and a device that
couldn't be audited would have been reported compliant.

The gate now requires that checks actually ran, before it looks at whether they
passed:

```yaml
- hostvars[item].results | default([]) | length > 0
- hostvars[item].results | default([]) | rejectattr('ok') | rejectattr('excused') | list | length == 0
```

*Absence of failure is not evidence of compliance. Make the test demand
evidence.*

---

## 5. The first useful run corrected the documentation

The facts playbook gathered model, version and interfaces from all five devices
in about 31 seconds:

```text
LAB-CISCO-3560CG-1 / WS-C3560CG-8PC-S / 15.0(2)SE5    / 15 interfaces
LAB-CISCO-3560CG-2 / WS-C3560CG-8PC-S / 15.2(2)E5     / 16 interfaces
C2940-LAB          / WS-C2940-8TT-S   / 12.1(22)EA13  / 13 interfaces
SRX345-LAB         / srx345           / 20.2R3-S2.5   / 45 interfaces
ARISTA710P-LAB     / CCS-710P-12      / 4.30.4M       / 16 interfaces
```

The two 3560s are not on the same image. The documentation had said "IOS 15.x"
for both, as a hedge. That difference later mattered (finding 1: 15.0 can't do
CTR mode).

Two smaller things from the same run. The Junos facts module crashed on its
`hardware` subset (`'Hardware' object has no attribute '_module'`), so Junos
gathers `min` and `interfaces` only. The module that crashed turned out to be
Juniper's own, reached through a redirect nobody could see (finding 13).
And Junos reports 45 interfaces where the switches report physical ports,
because it counts logical units. Anything that feeds these numbers into a
source of truth has to normalise them first.

*A source of truth earns its keep the first time it contradicts a document.*

---

## 6. The allowlist had two defects, failing in opposite directions

The deploy account's permissions are an allowlist on the TACACS+ server.
Proving it took two stages: a validator that asks the server directly, with no
device involved, then a real session on real switches. Each stage caught a
different defect.

**The first failed closed.** IOS appends `<cr>` as the last argument of every
authorization request, and the server matches patterns against the full string.
So a pattern that looks obviously right —

```text
if (cmd =~ /^(no )?ip http (secure-)?server$/) permit
```

— never matches `no ip http server <cr>`, and the command is refused. It was
caught because the validator tested each anchored pattern twice, with and
without the terminator:

```text
OK    testdeploy may disable http              PASS
FAIL  ...disable http, with <cr>               want PASS got FAIL
```

The fix makes the terminator optional (`( ?<cr>)?$`) rather than dropping the
anchor, which would have let `^configure( terminal)?` also permit
`configure replace`.

**The second failed open, and only hardware could show it.** Logged in as the
deploy account, a session typed two commands the allowlist doesn't contain:
entering an interface, and creating a local user. Both were accepted. The authorization log had two entries for that session:

```text
10.99.20.1  svc-ansible-rw  deploy  permit  shell
10.99.20.1  svc-ansible-rw  deploy  permit  shell  configure terminal <cr>
```

Nothing after `configure terminal` was ever sent to the server.
`aaa authorization commands 15` covers exec commands; configuration mode is a
separate setting, `aaa authorization config-commands`, and on these switches its
absence meant **off**. The account could create users, and it reconfigured the
AAA subsystem itself — closing the hole with the privilege the hole gave it.

With the line added, the same account:

```text
10.99.20.1  svc-ansible-rw  deploy  deny    shell  clear counters <cr>
10.99.20.1  svc-ansible-rw  deploy  deny    shell  interface GigabitEthernet 0 8 <cr>
10.99.20.1  svc-ansible-rw  deploy  permit  shell  write memory <cr>
```

The 2003 image has `config-commands` too. Arista's
`aaa authorization commands all` already covers configuration mode, which was
tested rather than assumed after IOS had disproved one assumption. Note the
interface name: IOS sent `GigabitEthernet 0 8`, the slash as a separate argument.
A pattern written from what the command looks like when typed would not match
it.

A compliance check now asserts `aaa authorization config-commands` on every IOS
device, because nothing in a configuration without it says anything is missing.

*A server-side test proves the ruleset. Only a device proves the device asks.*

---

## 7. `--check` proves intent, not feasibility

The first write turned off the IOS HTTP server. The dry run across the fleet
listed two commands for each switch, including the C2940:

```text
"http_off.commands": ["no ip http server", "no ip http secure-server"]
```

The real run failed on the C2940's second line:

```text
no ip http secure-server
             ^
% Invalid input detected at '^' marker.
```

A 2003 image has no HTTPS management server, so the command doesn't exist in
its parser. On IOS, check mode computes the difference between desired and
running configuration **locally** and never asks the device whether it would
accept the commands. A clean `--check` there is a statement of intent.

Then it got worse. IOS configuration isn't transactional, so line 1 had already
been applied. Ansible reported `changed: false`; the device disagreed. And
because the task failed, `save_when` never ran:

```text
show running-config | include http   ->  no ip http server
show startup-config  | include http  ->  ip http server
```

An audit reading the running configuration would have gone green, and the next
reload would have undone the fix.

Three changes came out of it: the HTTPS line became a separate task, skipped
where a host variable says the device has no HTTPS server; `save_when` became
`modified`, which saves whenever running differs from startup rather than only
when this run changed something; and a compliance check now compares running
against startup (finding 8).

The variable is a **capability** flag, not a compliance exception. An exception
says a device is non-compliant and that's been accepted; this device has
nothing to disable, so it is compliant by construction.

The habit that came out of it paid off on the next payload. Before writing the
syslog playbook, the C2940's parser was asked:

```text
C2940-LAB(config)#logging ?
  Hostname or A.B.C.D  IP address of the logging host
```

No `host` keyword on 12.1: it's `logging 10.99.20.32`. That went into a
second capability flag before any playbook tried the wrong syntax.

*Ask the device's parser before writing the playbook, not after the playbook
fails.*

---

## 8. An unsaved change fooled an audit that only read running-config

Finding 7 left one device where running and startup configuration disagreed, and
the audit couldn't see it. The fix was a check that compares the two. The
difficulty is noise: the two commands print different headers, and some lines
drift on their own.

Rather than guess at a filter, a diagnostic printed every differing line on
three devices known to be saved:

```text
3560CG-1   "Building configuration..."   "Current configuration : 3658 bytes"   "Using 3658 out of 524288 bytes"
3560CG-2   (the same three)
C2940-LAB  (the same three)  +  "ntp clock-period 17180193"  /  "ntp clock-period 17180192"
```

The drifting `ntp clock-period` appeared on the 2003 switch only. A filter
written from the newer switches would have failed the C2940 on every run
forever, and a check that always fails gets ignored.

The check was then made to fail deliberately — an unsaved description on an
unused port of the C2940:

```text
C2940-LAB   17/20  ->  16/20   Running config is saved to startup   (every other device unchanged)
```

— and recovered when the change was reverted and saved. A check that has never
been seen to fail isn't one yet.

*Write filters from observed output, from every platform, and make the check
fail on purpose before trusting it.*

---

## 9. The visible error wasn't the failure

The syslog playbook's first dry run against the Arista failed with this:

```text
end
% Invalid input
ARISTA710P-LAB#
```

The prompt is the clue: `#`, not `(config)#`. The session was never in
configuration mode, so `end` was invalid. That was the playbook's cleanup
tripping over an earlier failure, and EOS's parser rejected it before the AAA
server was ever consulted — which is why it isn't in the authorization log.

The log had the real failure:

```text
10.99.10.14  svc-ansible-rw  deploy  permit  shell  show configuration sessions <cr>
10.99.10.14  svc-ansible-rw  deploy  permit  shell  show running-config <cr>
10.99.10.14  svc-ansible-rw  deploy  deny    shell  configure session ansible_179049026576 <cr>
```

Ansible's EOS modules don't use `configure terminal`. They open a
**configuration session**, stage the change, and commit or abort it. The
allowlist was widened by exactly the strings the log showed —
`configure session ansible_[0-9]+` and `commit` / `abort` — and no wider.

That didn't widen what the account can change, because each line inside a
session is still authorized on its own:

```text
10.99.10.14  svc-ansible-rw  deploy  permit  shell  configure session ansible_179052911183 <cr>
10.99.10.14  svc-ansible-rw  deploy  permit  shell  logging host 10.99.20.32 <cr>
10.99.10.14  svc-ansible-rw  deploy  permit  shell  show session-config diffs <cr>
10.99.10.14  svc-ansible-rw  deploy  permit  shell  abort <cr>
```

It also changes what `--check` means on this platform. EOS check mode builds a
real session, asks the switch for the difference, and aborts:

```text
--- system:/running-config
+++ session:/ansible_179052911183-session-config
+logging host 10.99.20.32
```

That diff came from the switch. The failure in finding 7 couldn't have happened
here.

*The error message tells you what broke last. The authorization log tells you
what broke first.*

---

## 10. The automation library asked for a root shell

In the same log, before any configuration was attempted:

```text
10.99.10.14  svc-ansible-rw  deploy  deny  shell  bash timeout 5 cat /mnt/flash/boot-config <cr>
```

During device discovery, Arista's Ansible collection asks to run a command in
`bash`. On EOS, `bash` is a shell on the underlying Linux system. Permitting it
would permit everything, including editing the switch's own AAA configuration
from underneath the CLI that enforces it.

Nobody would have written "don't let the automation library open a shell" as a
rule. It didn't need writing: the allowlist is deny-by-default, the request was
refused, and the collection carried on without the file. The switch logged the
refusal independently too:

```text
ARISTA710P-LAB Aaa: %AAA-4-CMD_AUTHZ_FAILED: User svc-ansible-rw failed authorization to execute command 'bash timeout 5 cat /mnt/flash/boot-config'
```

The refusal is now an asserted check in the TACACS+ validator. If the profile is
ever loosened in a way that lets it through, the validator fails.

*An allowlist catches what nobody thought to forbid. A denylist would have let
this through.*

---

## 11. Configured isn't delivered

The syslog payload was verified on the collector, not in the device
configurations. Every device should have produced a message as a side effect of
being reconfigured, and did:

```text
3560cg-2.casey.corp  %SYS-5-CONFIG_I: Configured from console by svc-ansible-rw on vty0 (10.99.20.33)
3560cg-2.casey.corp  %SYS-6-LOGGINGHOST_STARTSTOP: Logging to host 10.99.20.32 port 514 started - CLI initiated
c2940-lab.casey.corp %SYS-5-CONFIG_I: Configured from console by svc-ansible-rw on vty0 (10.99.20.33)
10.255.10.2          %SYS-5-CONFIG_I: Configured from console by svc-ansible-rw on vty0 (10.99.20.33)
```

Three devices were filed under their names. One arrived as `10.255.10.2` — the
3560 at the far site, whose route to the collector runs through the IPsec
tunnel, so its syslog leaves from its transit link toward the SRX rather than
from its management address. Reverse DNS can't name that address.

It's the same class of problem the TACACS+ build found with the SRX's accounting,
which left through the tunnel from an address the server didn't recognise. The
fix is `logging source-interface`, which isn't on the allowlist yet.

*A configuration line is a claim. A log file on the collector is evidence.*

---

## 12. The vault is loaded when a playbook is parsed

The deploy credential lives in an `ansible-vault` file, loaded with
`vars_files`. A syntax check failed without the vault password:

```text
[ERROR]: Invalid vars_files file '/opt/netauto/playbooks/../vault/deploy.yml': Attempting to decrypt but no vault secrets found.
```

`vars_files` is read when the playbook is parsed, so even `--syntax-check`
needs the key. That matters for the next stage: the plan for a CI runner puts
linting first, precisely because it needs no credentials. With `vars_files` it
does.

The fix is to load the vault inside a task (`include_vars`), so that parsing
never touches it. Not done yet.

*Know when your secrets are read, not just where they are stored.*

---

## 13. A suppressed warning was hiding a deadline

Before this repository was published, `ansible-lint` flagged the two Junos
modules:

```text
fqcn[canonical]: You should use canonical module name `juniper.device.junos_facts`
                 instead of `junipernetworks.junos.junos_facts`.
```

The `junipernetworks.junos` collection is deprecated. Its modules are redirects
to Juniper's own `juniper.device`, which was already installed as a dependency,
so every Junos task had been running Juniper's code all along. And the
collection's runtime metadata sets a date:

```yaml
junos_facts:
  redirect: juniper.device.junos_facts
  deprecation:
    removal_date: "2028-04-01"
```

Ansible had been ready to say so on every run. It never did, because
`ansible.cfg` had carried `deprecation_warnings = False` since the control node
was built. With warnings on, a syntax check of the original playbooks shows it:

```text
[DEPRECATION WARNING]: junipernetworks.junos.junos_facts has been deprecated.
The junipernetworks.junos collection is deprecated. Use juniper.device.junos_facts instead.
```

The playbooks now use the canonical names — the same code that was already
running — and the same syntax check shows no deprecations at all. So the
warnings are back on: they cost nothing now, and the next deadline will be
announced instead of hidden.

*Turning off a class of warnings turns off the one that matters too. Silence the
specific noise, not the channel.*
