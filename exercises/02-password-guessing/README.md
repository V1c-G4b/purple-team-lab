# 02: Hardening SSH against brute force

| | |
|---|---|
| Date | 2026-11-14 |
| Technique (MITRE ATT&CK) | T1110.001 Brute Force: Password Guessing |
| Target | Ubuntu Server 26.04 (`10.10.10.3`) |
| Attacker | Kali Linux 2026.3 (`10.10.10.4`) |
| Outcome | TODO |

> Exercise performed in my isolated lab, against machines I own.

## Goal
Re-run the attack from [exercise 01](../01-ssh-brute-force/README.md)
against one defense at a time, and see where each one stops it.

## Baseline
Same victim and same attack as exercise 01: `suporte` / `princess`,
Hydra with the top 50 of `rockyou.txt`. Without defenses, the password
is found in a few seconds.

## Round 1: restrict who can use SSH
### Defense
File `/etc/ssh/sshd_config.d/10-allowusers.conf`:

```
AllowUsers v1c-g4b
```

Validated with `sudo sshd -t` and applied with `sudo systemctl restart ssh`.
### Prediction
Hydra would find the password, but authorization would block the login
### Result
1 of 1 target completed, 0 valid password found.
Why? SSH treats unlisted accounts as invalid and answers the same way whether the password is right or wrong
### Log trail

```bash
sudo grep "AllowUsers" /var/log/auth.log | tail -3
```
```text
2026-11-14T03:46:15.878640+00:00 v1c sshd-session[6685]: User suporte from 10.10.10.4 not allowed because not listed in AllowUsers
2026-11-14T03:46:17.779494+00:00 v1c sshd-session[6689]: User suporte from 10.10.10.4 not allowed because not listed in AllowUsers
2026-11-14T03:46:19.753656+00:00 v1c sshd-session[6695]: User suporte from 10.10.10.4 not allowed because not listed in AllowUsers
```
```bash
 sudo grep "Failed password for invalid user suporte" /var/log/auth.log | wc -l
```
```text
59
```
### Limits
This defense is limited to SSH connections and the password is still weak. Also the allowed user in the list is the only target for attacks and with that the invader can focus on him.
## Round 2: keys instead of passwords
(mesmas cinco seções)

## Round 3: block repeated failures
(mesmas cinco seções)

## All defenses together

## Comparison
| Defense | Stops the attack? | Log signature | Side effects |
|---|---|---|---|

## Extra findings

## What I learned
