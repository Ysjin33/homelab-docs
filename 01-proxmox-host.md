# Proxmox Host: John

## 1. Goal

I run Proxmox VE as a home hypervisor so I can build and break virtual machines in a safe environment. It’s how I’m learning how operating systems, networks and servers work in practice, and it’s building the hands-on skills I need for an IT career. It’s also just a lot of fun.


## 2. Environment

| Item | Value |
|---|---|
| Hostname | John |
| Proxmox VE version | pve-manager/9.2.21 |
| Hardware | Dell OptiPlex 7090 |
| CPU | 11th Gen Intel(R) Core(TM) i5-11500T 6 Core |
| RAM | 32 GB |
| Boot disk | BC711 NVMe SK hynix 256GB |
| Management IP | 192.168.0.50 |
| Gateway | 192.168.0.1 |
| DNS server | 100.100.100.100 |
| Web UI | https://192.168.0.50:8006 |
| Install date | 26-08-2026 |

## 3. Steps taken

> Write your own steps in order. Include the commands or menu paths you actually used.

1. **Prepared the installer:** <ISO version, tool used to make the USB, e.g. Rufus or balenaEtcher>
2. **BIOS/UEFI settings:** <virtualisation enabled (Intel VT-x / AMD-V), boot order, anything else you changed>
3. **Installed Proxmox VE:** <filesystem chosen (ext4/ZFS), hostname, IP, gateway, DNS>
4. **First login:** <how you reached the web UI, certificate warning accepted>
5. **Updated the host:** <repositories used (e.g. no-subscription), update commands, reboot>
6. **Configured networking:** <see section 5>
7. **Added storage and uploaded ISOs:** <see section 6>


## 5. Network configuration

| Item | Value |
|---|---|
| Bridge | vmbr0 |
| Bridge port (physical NIC) | nic0 |
| Bridge address | 192.168.0.50/24 |
| Gateway | 192.168.0.1 |
| VLANs | None |
| Isolated bridge | vmbr1 (no bridge port, no host IP), created 20-11-2026 for the honeypot lab |


## 6. Storage

| Storage ID | Type | Status | Used for | Total | Used | Available | Usage |
|---|---|---|---|---|---|---|---|
| local | dir | active | ISO images, backups, container templates | 67.7 GiB | 10.9 GiB | 53.3 GiB | 16.12% |
| local-lvm | lvmthin | active | VM disks | 141.2 GiB | 10.7 GiB | 130.5 GiB | 7.56% |


## 7. Virtual machines on this host

| VM ID | Name | OS | Role | IP | Doc |
|---|---|---|---|---|---|
| 100 | DC01 | Windows Server | AD DS + DNS (lab.local) | ... | [02-ad-dc01.md](02-ad-dc01.md) |
| 101 | honeypot | Ubuntu Server 24.04.3 LTS | Honeypot lab (Cowrie + Python honeypots) | DHCP (admin NIC) | [03-honeypot.md](03-honeypot.md) |

## 8. Snapshots and backups

| VM | Snapshot | Taken | Why |
|---|---|---|---|
| 100 DC01 | pre-lockout-policy | 19-11-2026 | Before changing the account lockout policy |
| 100 DC01 | pre-honeypot-lab | 20-11-2026 | Before starting the honeypot lab (no RAM) |
| 101 honeypot | clean-base | 20-11-2026 | Hardened base: updated, firewall on, SSH on 22222, Docker installed, trap NIC added (no RAM) |
| 101 honeypot | cowrie-running | 25-11-2026 | Cowrie honeypot running in Docker on the trap NIC and logging |

No scheduled backups yet.

## Changelog for this page

| Date | Change |
|---|---|
| 19-10-2026 | Created page |
| 20-11-2026 | Added isolated bridge vmbr1; recorded snapshots |
| 20-11-2026 | Recorded VM 101 snapshot clean-base |
| 25-11-2026 | Recorded VM 101 snapshot cowrie-running |