# Purple Team Lab

A home lab where I play both sides of a security exercise: I run a real attack from Kali (red), then detect it in the logs, contain it and investigate it on the victim (blue). Each exercise is mapped to a MITRE ATT&CK technique and documented like an incident report: scenario, attack, detection, timeline, response, root causes and what I learned.

Everything runs on isolated VMs that I own, on a network the rest of my home network can't reach.

## Diagram

```mermaid
flowchart LR
    mac["MacBook<br/>workstation"]

    subgraph asus["Asus TUF: Windows + VirtualBox"]
        fw["Windows Firewall<br/>port 2223 only, private network"]
        subgraph labnet["labnet: 10.10.10.0/24"]
            kali["Kali Linux<br/>10.10.10.4<br/>attacker and bastion"]
            ubuntu["Ubuntu Server<br/>10.10.10.3<br/>victim"]
        end
    end

    mac -- "SSH :2223" --> fw --> kali
    kali -- "attack and admin access" --> ubuntu
```

## How each exercise works

```mermaid
flowchart LR
    A[Snapshot rollback] --> B[Red: attack]
    B --> C[Blue: detect in logs]
    C --> D[Contain and investigate]
    D --> E[Root causes and fixes]
    E --> F[Repeat with fixes applied]
```

## Exercises

| # | Exercise | Status |
|---|----------|--------|
| 01 | [SSH brute force](exercises/01-ssh-brute-force/README.md) | done |

## Lab documentation

- [Architecture](lab/architecture.md): the VMs, the labnet, the bastion and the reasoning behind each choice
- [Setup](lab/setup.md): step-by-step guide to build the same lab

## Notes

- [Week 01](notes/week-01.md)
