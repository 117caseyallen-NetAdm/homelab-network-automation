# Verification

The output behind each claim in the README. Captured 2026-09-22 to 2026-09-27;
all times UTC. Authorization-log lines are trimmed to device, user, profile,
verdict, service and command.

## Contents

1. [The fleet across the build](#1-the-fleet-across-the-build)
2. [The allowlist, with no device involved](#2-the-allowlist-with-no-device-involved)
3. [The allowlist, on the devices](#3-the-allowlist-on-the-devices)
4. [The first write, seen three ways](#4-the-first-write-seen-three-ways)
5. [Convergence](#5-convergence)
6. [An EOS configuration session](#6-an-eos-configuration-session)
7. [Syslog, by arrival](#7-syslog-by-arrival)
8. [The backup system saw every change](#8-the-backup-system-saw-every-change)
9. [What the legacy switches negotiate](#9-what-the-legacy-switches-negotiate)
10. [What is deliberately not here](#10-what-is-deliberately-not-here)

---

## 1. The fleet across the build

Checks were added as the build found things worth checking, so the totals grow.

| Stage | 3560CG-1 | 3560CG-2 | C2940 | Arista | SRX |
| --- | --- | --- | --- | --- | --- |
| First run, wrong regexes | 8/12 | 8/12 | 8/12 | 5/9 | 0/0 — never audited |
| Regexes corrected | 14/17 | 14/17 | 13/17 | 9/10 | 10/10 |
| Exception recorded (C2940, MD5) | 14/17 | 14/17 | 13/17 + 1 excepted | 9/10 | 10/10 |
| `config-commands` check added and fixed | 15/18 | 15/18 | 14/18 + 1 | 9/10 | 10/10 |
| **Payload 1:** HTTP server off | 16/18 | 16/18 | 15/18 + 1 | 9/10 | 10/10 |
| HTTPS and running = startup checks added | 18/20 | 18/20 | 17/20 + 1 | 10/11 | 10/10 |
| **Payload 2:** syslog forwarded | **19/20** | **19/20** | **18/20 + 1** | **11/11** | **10/10** |

The fleet table the last run printed:

```text
3560CG-1         19/20 pass  1 fail  0 excepted   Local secrets are not type 5 (MD5)
3560CG-2         19/20 pass  1 fail  0 excepted   Local secrets are not type 5 (MD5)
C2940-LAB        18/20 pass  1 fail  1 excepted   No leftover Administrator account
ARISTA710P-LAB   11/11 pass  0 fail  0 excepted
SRX345-LAB       10/10 pass  0 fail  0 excepted
```

### What to notice

- The **SRX's 0/0** in the first row is the result the first version of the gate
  would have passed as compliant.
- **Both remaining failures are credentials**, kept out of the pipeline
  deliberately.
- The C2940's exception is **stable** across every stage, and it expires on
  2027-09-24.

---

## 2. The allowlist, with no device involved

The TACACS+ validator in
[homelab-tacacs-aaa](https://github.com/117caseyallen-NetAdm/homelab-tacacs-aaa/blob/main/scripts/tac-validate.sh)
asks the running server for each answer through a loopback device entry. The
deploy account's 21 checks — the other 13 cover the admin and read-only
profiles:

```text
  OK    testdeploy authenticates                 PASS
  OK    testdeploy authenticates (PAP)           PASS
  OK    testdeploy shell start                   PASS
  OK    testdeploy may show run                  PASS
  OK    testdeploy may configure                 PASS
  OK    testdeploy may disable http              PASS
  OK    testdeploy may set logging host          PASS
  OK    testdeploy may write memory              PASS
  OK    ...disable http, with <cr>               PASS
  OK    ...configure terminal, with <cr>         PASS
  OK    testdeploy REFUSED add user              FAIL
  OK    testdeploy REFUSED erase startup         FAIL
  OK    testdeploy REFUSED reload                FAIL
  OK    testdeploy REFUSED interface             FAIL
  OK    testdeploy REFUSED no aaa                FAIL
  OK    testdeploy REFUSED tacacs key            FAIL
  OK    testdeploy may open EOS session          PASS
  OK    testdeploy may commit                    PASS
  OK    testdeploy may abort                     PASS
  OK    testdeploy REFUSED EOS bash              FAIL
  OK    testdeploy REFUSED other session         FAIL

  34 passed, 0 failed
```

`FAIL` in the right-hand column is the expected answer for a refusal; `OK` means
the server gave the answer the check wanted. The two `<cr>` rows were `FAIL`
against `PASS` until the anchors were fixed
([finding 6](findings.md#6-the-allowlist-had-two-defects-failing-in-opposite-directions)).

---

## 3. The allowlist, on the devices

A session as `svc-ansible-rw` on 3560CG-2, before `aaa authorization
config-commands`. It entered an interface, set `ip http server`, and created a
local user. The server heard about none of it:

```text
10.99.20.1  svc-ansible-rw  deploy  permit  shell
10.99.20.1  svc-ansible-rw  deploy  permit  shell  configure terminal <cr>
```

After:

```text
10.99.20.1  svc-ansible-rw  deploy  permit  shell
10.99.20.1  svc-ansible-rw  deploy  permit  shell  configure terminal <cr>
10.99.20.1  svc-ansible-rw  deploy  permit  shell  show running-config <cr>
10.99.20.1  svc-ansible-rw  deploy  deny    shell  clear counters <cr>
10.99.20.1  svc-ansible-rw  deploy  permit  shell  configure terminal <cr>
10.99.20.1  svc-ansible-rw  deploy  deny    shell  interface GigabitEthernet 0 8 <cr>
10.99.20.1  svc-ansible-rw  deploy  permit  shell  write memory <cr>
```

And what the operator saw:

```text
LAB-CISCO-3560CG-2#clear counters
Command authorization failed.

LAB-CISCO-3560CG-2(config)#interface GigabitEthernet0/8
Command authorization failed.
```

The same test on the other platforms:

```text
C2940-LAB(config)#interface FastEthernet0/1
Command authorization failed.

ARISTA710P-LAB(config)#interface Ethernet1
% Authorization denied for command 'interface Ethernet1'
```

---

## 4. The first write, seen three ways

**The playbook**, dry run and then real, on one device:

```text
TASK [Show the commands that were sent]
ok: [3560CG-2] => {
    "http_off.commands": [
        "no ip http server",
        "no ip http secure-server"
    ]
}
PLAY RECAP
3560CG-2   : ok=2    changed=1    unreachable=0    failed=0
```

**The AAA server**, authorizing each command from the control node at
`10.99.20.33`:

```text
10.99.20.1  svc-ansible-rw  deploy  permit  shell  show running-config <cr>
10.99.20.1  svc-ansible-rw  deploy  permit  shell  configure terminal <cr>
10.99.20.1  svc-ansible-rw  deploy  permit  shell  no ip http server <cr>
10.99.20.1  svc-ansible-rw  deploy  permit  shell  no ip http secure-server <cr>
10.99.20.1  svc-ansible-rw  deploy  permit  shell  copy running-config startup-config <cr>
10.99.20.1  svc-ansible-rw  deploy  permit  shell  show running-config <cr>
```

**The device**:

```text
LAB-CISCO-3560CG-2#show running-config | include http
no ip http server
no ip http secure-server
```

And the fleet run, where the C2940 failed halfway and left its fix unsaved:

```text
fatal: [C2940-LAB]: FAILED! => {"changed": false, "msg": "Task failed: ...no ip http secure-server
                                                             ^ % Invalid input detected at '^' marker."}

C2940-LAB#show running-config | include http
no ip http server
C2940-LAB#show startup-config | include http
ip http server
```

---

## 5. Convergence

One fleet dry run after the first device had been fixed. Action only where
reality differed from intent:

```text
changed: [3560CG-1]
ok:      [3560CG-2]
changed: [C2940-LAB]
```

And two consecutive real runs on a device that was already correct:

```text
3560CG-2   : ok=1    changed=0    unreachable=0    failed=0    skipped=1
3560CG-2   : ok=1    changed=0    unreachable=0    failed=0    skipped=1
```

---

## 6. An EOS configuration session

The Arista's first dry run, refused. The error the playbook printed was the
cleanup, not the cause:

```text
fatal: [ARISTA710P-LAB]: FAILED! => {"msg": "b'end\r\n% Invalid input\r\nARISTA710P-LAB#'"}
```

The cause, in the authorization log:

```text
10.99.10.14  svc-ansible-rw  deploy  deny    shell  bash timeout 5 cat /mnt/flash/boot-config <cr>
10.99.10.14  svc-ansible-rw  deploy  permit  shell  show configuration sessions <cr>
10.99.10.14  svc-ansible-rw  deploy  permit  shell  show running-config <cr>
10.99.10.14  svc-ansible-rw  deploy  deny    shell  configure session ansible_179049026576 <cr>
```

With sessions on the allowlist, a dry run — every line inside the session still
authorized on its own, and the session aborted at the end:

```text
10.99.10.14  svc-ansible-rw  deploy  deny    shell  bash timeout 5 cat /mnt/flash/boot-config <cr>
10.99.10.14  svc-ansible-rw  deploy  permit  shell  show configuration sessions <cr>
10.99.10.14  svc-ansible-rw  deploy  permit  shell  show running-config <cr>
10.99.10.14  svc-ansible-rw  deploy  permit  shell  configure session ansible_179052911183 <cr>
10.99.10.14  svc-ansible-rw  deploy  permit  shell  logging host 10.99.20.32 <cr>
10.99.10.14  svc-ansible-rw  deploy  permit  shell  show session-config diffs <cr>
10.99.10.14  svc-ansible-rw  deploy  permit  shell  configure session ansible_179052911183 <cr>
10.99.10.14  svc-ansible-rw  deploy  permit  shell  abort <cr>
10.99.10.14  svc-ansible-rw  deploy  permit  shell  show running-config <cr>
10.99.10.14  svc-ansible-rw  deploy  permit  shell  show startup-config <cr>
```

The diff the playbook displayed came from the switch:

```text
--- system:/running-config
+++ session:/ansible_179052911183-session-config
+logging host 10.99.20.32
```

After the refused attempt, `show configuration sessions` on the Arista listed
nothing pending, so the failure left nothing half-built. And `bash` stays
refused; the collection carries on without it.

---

## 7. Syslog, by arrival

From the collector, after the real run. Each device reported the change that
made it report:

```text
3560cg-2.casey.corp   %SYS-5-CONFIG_I: Configured from console by svc-ansible-rw on vty0 (10.99.20.33)
3560cg-2.casey.corp   %SYS-6-LOGGINGHOST_STARTSTOP: Logging to host 10.99.20.32 port 514 started - CLI initiated
c2940-lab.casey.corp  %SYS-5-CONFIG_I: Configured from console by svc-ansible-rw on vty0 (10.99.20.33)
10.255.10.2           %SYS-5-CONFIG_I: Configured from console by svc-ansible-rw on vty0 (10.99.20.33)
ARISTA710P-LAB        Aaa: %AAA-4-CMD_AUTHZ_FAILED: User svc-ansible-rw failed authorization to execute
                      command 'bash timeout 5 cat /mnt/flash/boot-config'
```

The collector's per-device directories, an hour later:

```text
10.255.10.2            3560cg-2.casey.corp    ARISTA710P-LAB
c2940-lab.casey.corp   PA440-LAB              SRX345-LAB
```

Six devices, all forwarding. `10.255.10.2` is 3560CG-1, on its transit link
toward the SRX ([finding 11](findings.md#11-configured-isnt-delivered)). The
firewall's own lines aren't reproduced: PAN-OS puts its serial number in every
syslog message.

---

## 8. The backup system saw every change

The lab's Oxidized instance polls every device hourly and commits the running
configuration to Git when it changes. It knows nothing about Ansible. Its
history over the same two days, filtered to the lines this work touched:

```text
09-26 16:04  3560CG-1, 3560CG-2, C2940   +aaa authorization config-commands   (by hand, finding 6)
09-26 17:04  3560CG-1                    -ip http server  -ip http secure-server  +no ip http server  +no ip http secure-server
09-26 17:05  3560CG-2                    (the same four lines)
09-26 17:05  C2940                       -ip http server  +no ip http server
09-27 17:24  3560CG-1, 3560CG-2          +logging host 10.99.20.32
09-27 17:24  C2940                       +logging 10.99.20.32
09-27 17:25  ARISTA710P-LAB              +logging host 10.99.20.32
```

Each device's change is in that device's own syntax: two independent systems
agree on what the pipeline did. The lab's plan called this loop "deploy,
detect, commit". The switches' own configuration headers even name who made the
change:

```text
! Last configuration change at 16:53:18 UTC Sat Sep 26 2026 by svc-ansible-rw
```

Two more C2940 commits in that window changed nothing but those header lines.
The first records the manual `write memory` that saved the half-applied fix from
[finding 7](findings.md#7---check-proves-intent-not-feasibility). The second
records the unsaved change of
[finding 8](findings.md#8-an-unsaved-change-fooled-an-audit-that-only-read-running-config)
being reverted and saved, and then a session that entered configuration mode only
to ask the parser for its `logging` syntax. Nothing in the configuration
changed, but leaving configuration mode moved IOS's "last configuration change"
stamp all the same. Oxidized keeps that line deliberately, and only when it names
a user, as a record of who touched the device.

---

## 9. What the legacy switches negotiate

A transport that stops after key exchange — no authentication — against each
legacy switch:

```text
C2940      kex offered: diffie-hellman-group1-sha1                                         cipher: 3des-cbc
3560CG-2   kex offered: diffie-hellman-group-exchange-sha1, group14-sha1, group1-sha1      cipher: aes128-ctr
3560CG-1   kex offered: diffie-hellman-group-exchange-sha1, group14-sha1, group1-sha1      cipher: aes128-cbc

paramiko 2.12.0 (pinned)   all three OK
paramiko 3.5.1             all three OK   SHA-1 key exchange still offered by default
paramiko 4.0.0             all three OK   SHA-1 key exchange still offered by default
paramiko 5.0.0             no acceptable kex algorithm
```

`cryptography` on the control node: 50.0.1, with `TripleDES` still importable
([finding 1](findings.md#1-the-fix-for-the-oldest-switches-was-stricter-than-it-needed-to-be)).

---

## 10. What is deliberately not here

- **Device configurations.** The backup system holds those, privately. Every
  excerpt here was read by a person before it was published.
- **The vault.** Not even encrypted — publishing it would invite an offline
  attack on the passphrase. [`vault/deploy.yml.example`](../vault/deploy.yml.example)
  shows its shape.
- **Serial numbers.** Device facts are gitignored, the facts playbook no longer
  prints them, and the firewall's syslog lines are excluded above.
