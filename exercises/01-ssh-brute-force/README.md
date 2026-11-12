# 01: SSH brute force

| | |
|---|---|
| Date | 2026-11-11 |
| Technique (MITRE ATT&CK) | [T1110.001 Brute Force: Password Guessing](https://attack.mitre.org/techniques/T1110/001/) |
| Target | Ubuntu Server 26.04 (`10.10.10.3`) |
| Attacker | Kali Linux 2026.3 (`10.10.10.4`) |
| Outcome | Account `suporte` compromised in 4 seconds, contained and investigated |

> Exercise performed in my isolated lab, against machines I own. Architecture in [`lab/architecture.md`](../../lab/architecture.md).

## Scenario

A support account (`suporte`, Portuguese for "support") created "just for a test" and forgotten on the server, with a weak password. This is common in companies: nobody remembers the account exists, and it stays reachable over SSH like any other.

| Item | Configuration |
|---|---|
| Network | VirtualBox NAT Network `labnet` (`10.10.10.0/24`), isolated from the home network |
| Victim account | `suporte`, password `princess` (weak on purpose) |
| Exposed service | SSH on port 22, accepting password login, no attempt limit |

## Attack (red side)

**Idea:** a dictionary attack. Instead of trying every possible combination, the attacker tries the passwords that show up most often in real leaks, starting with the most popular.

**1. Wordlist.** I used the first 50 passwords from `rockyou.txt`, a list of 14 million real passwords leaked in 2009, sorted by popularity. `princess` is the sixth.

```bash
sudo gunzip /usr/share/wordlists/rockyou.txt.gz
head -n 50 /usr/share/wordlists/rockyou.txt > top50.txt
```

**2. Attack with Hydra.**

```bash
hydra -l suporte -P top50.txt -t 4 ssh://10.10.10.3
```

| Parameter | Purpose |
|---|---|
| `-l suporte` | Targets a single fixed account |
| `-P top50.txt` | Tries the passwords in the list |
| `-t 4` | 4 parallel attempts (above that, SSH starts refusing connections) |

**Result:** password found in a few seconds.

## Detection (blue side)

**Live**, during the attack, in a second session on Ubuntu:

```bash
sudo tail -f /var/log/auth.log
```

The successful login is buried among the failures. With a few lines you can spot it by eye; with thousands, you can't. That's why the investigation uses filters:

```bash
# successful logins
sudo grep "Accepted password" /var/log/auth.log

# how many failures the account had
sudo grep "Failed password for suporte" /var/log/auth.log | wc -l
# 8
```

**Evidence** (excerpt from `/var/log/auth.log`, times in UTC):

```
17:42:38 sshd-session[1578]: Failed password for suporte from 10.10.10.4 port 33976 ssh2
17:42:38 sshd-session[1579]: Failed password for suporte from 10.10.10.4 port 33988 ssh2
17:42:40 sshd-session[1577]: Failed password for suporte from 10.10.10.4 port 33972 ssh2
17:42:40 sshd-session[1581]: Failed password for suporte from 10.10.10.4 port 34002 ssh2
17:42:40 sshd-session[1578]: Accepted password for suporte from 10.10.10.4 port 33976 ssh2
17:42:40 sshd-session[1578]: pam_unix(sshd:session): session opened for user suporte(uid=1001)
17:42:42 sshd-session[1581]: Failed password for suporte from 10.10.10.4 port 34002 ssh2
17:42:42 sshd-session[1577]: Failed password for suporte from 10.10.10.4 port 33972 ssh2
17:42:42 sshd-session[1578]: pam_unix(sshd:session): session closed for user suporte
17:42:43 sshd-session[1579]: Failed password for suporte from 10.10.10.4 port 33988 ssh2
```

**What the trail shows:**

| Signal | Why it indicates an attack |
|---|---|
| 8 failures in 7 seconds on the same account | Nobody mistypes their own password that fast |
| All from the same IP | A single source trying repeatedly |
| `Accepted password` in the middle of the failures | Not just an attempt: the account was compromised |
| Failures **after** the success | Parallel attempts that were already in flight (`-t 4`) |

**Why 8 failures and not 5?** `princess` was the sixth password, so there should be 5 misses. Since Hydra tries 4 passwords at once, other attempts were already in progress when the right one hit, and they failed too. The trail even reveals the tool's configuration.

## Timeline

| UTC | Local (UTC-3) | Event |
|---|---|---|
| 17:13:20 | 14:13:20 | Account `suporte` created |
| 17:42:36 | 14:42:36 | First attempt from `10.10.10.4` |
| 17:42:40 | 14:42:40 | Password cracked, session opened |
| 17:42:42 | 14:42:42 | Session closed by the attacker (2 seconds inside) |
| 17:42:44 | 14:42:44 | Last attack connections closed |
| 17:51:57 | 14:51:57 | Account locked |

**Response time:** 9 minutes between compromise and containment.

## Incident response

**1. Preserve the evidence** before changing anything:

```bash
sudo cp /var/log/auth.log ~/evidence-auth.log
```

**2. Contain.**

```bash
loginctl list-sessions        # is the intruder still logged in? (they weren't)
sudo passwd -l suporte        # lock the password
sudo usermod -e 1 suporte     # expire the account
```

`passwd -l` alone isn't enough: it locks the password, but SSH would still accept a key planted by the attacker. Expiring the account blocks every form of login.

**3. Investigate.**

| Question | Answer | How |
|---|---|---|
| How long was the intruder in? | 2 seconds | `session opened` and `session closed` in the log |
| Did they leave persistence? | No | `/home/suporte/.ssh` doesn't exist, no planted key |
| Are they still inside? | No | `loginctl list-sessions` |

Hydra only confirms the password and leaves. A real attacker would come back later with the cracked password, in a clean login with no prior failures, which a brute force rule wouldn't catch.

## Root causes and fixes

Any one of these fixes, on its own, would have stopped the attack:

| Cause | Fix |
|---|---|
| Weak password (6th on the world's most-used list) | Strong password policy |
| Any account can log in over SSH | Allow SSH only for those who need it |
| Unlimited attempts | Block the IP after a few failures |
| SSH accepts password login | Require SSH keys |
| Forgotten account on the server | Remove unused accounts |

## Extra findings

- **IP alone doesn't tell attacker from admin.** My legitimate access goes through Kali as a bastion, so it also shows up in the log as `10.10.10.4`. What sets them apart is the pattern: I log in on the first try with my own account; the attacker failed 8 times on a support account.
- **Tools disappear between versions.** On Ubuntu 26.04, `who` returned nothing and `last` isn't installed. The alternative was `loginctl` and `auth.log` itself.


## What I learned

This exercise improved my log analysis and interpretation. When Hydra
easily broke into the `suporte` account, I understood the method, the
parameters it uses, and how to tell from the logs when an attack has
succeeded.

From what I read about incident response, I learned the order of the
steps: preserve the evidence, isolate the attacker, investigate, and make
sure nothing was infected or taken.

From the attacker's side, I noticed how short the attack was: the logs
show many failures in just a few seconds. Human mistakes don't look like
that, because people need time to try again and figure out what went
wrong. From the defender's side, I saw the guards that were missing
before the attack reached its goal: rate limiting, SSH keys, and
managing unused accounts.

## Next steps

- [ ] Apply the fixes one at a time and repeat the attack to see where it gets blocked
- [ ] Teach log-scanner to read sshd's `auth.log`
- [ ] Create a rule for the clean login that follows a successful brute force
