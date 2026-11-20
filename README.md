# Homelab Documentation

A Proxmox-based lab for learning Windows Server, Active Directory and Cybersecurity.

## What I've built
| Project | Description | Status |
|---|---|---|
| [Proxmox host](01-proxmox-host.md) | Hypervisor running the lab | Done |
| [DC01](02-ad-dc01.md) | Domain controller (AD DS + DNS) for lab.local | Done |
| [Honeypot lab](03-honeypot.md) | Isolated honeypot network: Cowrie plus my own Python honeypots | In progress |

## Network diagram

## IP table
| Host | Role | IP |
|---|---|---|
| Router | Gateway | 192.168.0.1 |
| John | Proxmox host | 192.168.0.50 |
| DC01 | Domain controller | 192.168.0.x |
| honeypot | Honeypot VM admin NIC (DHCP) | assigned by DHCP, not recorded here |
| honeypot (trap) | Honeypot VM trap NIC on isolated vmbr1 | 10.10.10.10 |

## Runbooks
- [Reset a user's password](runbooks/reset-user-password.md)
- [Unlock an account](runbooks/unlock-account.md)
- [Add a user to a group](runbooks/add-user-to-group.md)