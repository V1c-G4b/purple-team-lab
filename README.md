# Purple Team Lab

What the lab is:
A home lab where I practice both sides of security: I attack my own
machines, then investigate the traces the attack left and fix what
allowed it.

I'm a full-stack developer moving into security, starting from zero.
Every exercise here follows the same cycle: attack, detect, respond,
harden, and document. The lab runs on VirtualBox, with a Kali attacker
and an Ubuntu victim on an isolated network.

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

## Exercises

| # | Exercise | Status |
|---|----------|--------|
| 01 | [SSH brute force](exercises/01-ssh-brute-force/README.md) | done |
| 02 | [TODO](exercises/02-new-exercise/README.md) | in progress |

## Lab documentation

- [Architecture](lab/architecture.md): the VMs, the labnet, the bastion and the reasoning behind each choice
- [Setup](lab/setup.md): step-by-step guide to build the same lab

## Notes

- [Week 01](notes/week-01.md)
