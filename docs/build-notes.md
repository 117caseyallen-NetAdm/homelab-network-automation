# Build notes

Each stage ends with the check that proved it. Nothing was allowed to change a
device until the reading side was trusted, and nothing was given a write
credential until the AAA server had been shown to refuse what it shouldn't
allow.

---

## 0 — Prerequisites

The TACACS+ build ([homelab-tacacs-aaa](https://github.com/117caseyallen-NetAdm/homelab-tacacs-aaa)):
every device authenticates administrators against Active Directory, authorizes
them per command, and accounts what they do. Everything here runs through that.

## 1 — The control node

`CA-ANS-LAB`, an unprivileged Debian 12 LXC on Proxmox at `10.99.20.33`, on the
management VLAN alongside the other tooling. Ansible in a virtualenv at
`/opt/ansible`, the project at `/opt/netauto`, pushed to a private repository in
the lab's Gitea.

Built by hand rather than with Terraform, deliberately: learning two tools at
once means learning neither properly.

**Check:** `ansible --version`. First it needed `LANG=C.UTF-8` — a minimal
container ships without a locale, and Ansible refuses to start without one.

## 2 — Reaching the fleet

The first connection attempt failed on three of five devices on key exchange
([finding 1](findings.md#1-the-fix-for-the-oldest-switches-was-stricter-than-it-needed-to-be)).
Pinned `paramiko<3`, then `ncclient<0.7` when installing it upgraded paramiko
straight back.

**Check:** a facts playbook against all five, one line per device, in about 31
seconds.

## 3 — Junos over NETCONF

`junos_facts` refused a CLI connection. Port 830 timed out. The inventory now
uses `ansible.netcommon.netconf` on port 22 for the `junos` group
([finding 2](findings.md#2-netconf-rode-the-port-that-was-already-open)).

**Check:** `ssh -s casey@10.99.0.2 netconf` returns a `<hello>`, and the facts
playbook completes for the SRX.

## 4 — The audit

`compliance.yml` and the checks in `group_vars`. The first run failed identically
everywhere, which was the tests' fault
([finding 3](findings.md#3-uniform-failure-across-independent-platforms-meant-the-tests-were-wrong)),
and its gate passed a device it never audited
([finding 4](findings.md#4-the-gate-passed-a-device-it-never-audited)).

Exceptions with a reason and an expiry date came next, so a permanent,
accepted failure — MD5 secrets on a 2003 image that supports nothing stronger —
could be recorded without leaving the report permanently red.

**Check:** a fleet table in which every remaining failure was real: syslog not
forwarded, the IOS HTTP server enabled, an `enable secret` the credential
rotation had missed, and a leftover local account.

## 5 — Accounts and an allowlist

In AD: `svc-ansible-ro` in the existing `NetOps` group, reusing the read-only
profile the backup system already proves daily, and `svc-ansible-rw` in a new
group, `NetDeploy`.

On the TACACS+ server: a `deploy` profile that permits session setup, the
diff Ansible reads first, and a short list of configuration commands — and
refuses everything else.

**Check:** the TACACS+ validator, with no device involved, grown from 13 checks
to 34 — including refusals for creating a user, erasing the startup
configuration, reloading, entering an interface, removing AAA, setting a
TACACS+ key, opening a shell, and opening a configuration session under any
name but Ansible's. It found the `<cr>` defect on the way
([finding 6](findings.md#6-the-allowlist-had-two-defects-failing-in-opposite-directions)).

## 6 — The devices have to ask

A real session as the deploy account on each platform it may change, with a
second session open in case anything locked the account out.

**Check:** on IOS, the first attempt failed open — configuration commands were
never sent to the server. After `aaa authorization config-commands`:
`clear counters` and `interface GigabitEthernet0/8` refused, `write memory`
permitted. The same test on the C2940 and the Arista, refused on both.

## 7 — The vault, and the first write

`ansible-vault create` holding the deploy credential, created through the
editor so the plaintext never touched disk. The encrypted file is committed to
the private repository only.

Payload 1, `no ip http server`, chosen because the audit had already proved it
was needed and it can't cut the SSH session it travels over.

**Check:** `--check --diff` on one device; the real run on that device; a second
run reporting `changed=0`; the authorization log showing read, change, save,
read back. Then the fleet — where the C2940 failed halfway
([finding 7](findings.md#7---check-proves-intent-not-feasibility)).

## 8 — Running config must equal startup

A check comparing the two, with the noise filter built from observed output
([finding 8](findings.md#8-an-unsaved-change-fooled-an-audit-that-only-read-running-config)).

**Check:** it passed on every saved device, failed on the C2940 with a
deliberately unsaved change while nothing else moved, and passed again once the
change was reverted and saved.

## 9 — Payload 2: syslog

The C2940's parser was asked for its `logging` syntax before the playbook was
written. The Arista's first dry run was refused over configuration sessions
([finding 9](findings.md#9-the-visible-error-wasnt-the-failure)), and over a
request for a shell that stays refused
([finding 10](findings.md#10-the-automation-library-asked-for-a-root-shell)).

**Check:** log files on the collector for every device, each containing the
change that created it
([finding 11](findings.md#11-configured-isnt-delivered)). Then the audit:

```text
3560CG-1         19/20   the remaining failure is the enable secret
3560CG-2         19/20   the same
C2940-LAB        18/20   a leftover local account, plus one accepted exception
ARISTA710P-LAB   11/11
SRX345-LAB       10/10
```

## Next

A Gitea Actions runner: lint and syntax check, the audit on every push and
every night, a `--check --diff` whose output is the review, and an apply behind
a button. The vault has to move into a task first
([finding 12](findings.md#12-the-vault-is-loaded-when-a-playbook-is-parsed)),
and the audit needs its own credential under its own vault ID.
