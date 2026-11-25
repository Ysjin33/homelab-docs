# Homelab Documentation

A Proxmox-based lab for learning Windows Server, Active Directory and Cybersecurity.

**New here?** Read the pages in order: the host first, then the domain controller, then the honeypot lab. Each page says what I did, how I checked it worked, and what went wrong. This repo is public, so admin addresses, login details and credentials are deliberately left out.

## What I've built
| Project | Description | Status |
|---|---|---|
| [Proxmox host](01-proxmox-host.md) | Hypervisor running the lab | Done |
| [DC01](02-ad-dc01.md) | Domain controller (AD DS + DNS) for lab.local | Done |
| [Honeypot lab](03-honeypot.md) | Isolated honeypot network: Cowrie plus my own Python honeypots | In progress (Week 1 of 3: Cowrie running, Kali VM being built) |

## Network diagram

The honeypot lab (see [03-honeypot.md](03-honeypot.md)):

```
 Home LAN (vmbr0) ───────────────────────────────┐
                                                 │ admin only: real SSH on port 22222
 Proxmox host "John"                             │
   ├─ DC01 (vmbr0)                               │
   ├─ VM 101 "honeypot"                          │
   │     ├─ ens18 (vmbr0)  admin NIC ────────────┘
   │     └─ ens19 (vmbr1)  trap NIC 10.10.10.10  <- Cowrie listens here on port 22
   └─ VM 102 "kali" (created, install in progress)
         └─ vmbr1 only, 10.10.10.20
 vmbr1 = isolated bridge: no physical port and no address on the host
```

## IP table
| Host | Role | IP |
|---|---|---|
| Router | Gateway | 192.168.0.1 |
| John | Proxmox host | 192.168.0.50 |
| DC01 | Domain controller | 192.168.0.x |
| honeypot | Honeypot VM admin NIC (DHCP) | assigned by DHCP, not recorded here |
| honeypot (trap) | Honeypot VM trap NIC on isolated vmbr1 | 10.10.10.10 |
| kali | Attacker VM on isolated vmbr1 only (address set after the install) | 10.10.10.20 |

## Runbooks
- [Reset a user's password](runbooks/reset-user-password.md)
- [Unlock an account](runbooks/unlock-account.md)
- [Add a user to a group](runbooks/add-user-to-group.md)