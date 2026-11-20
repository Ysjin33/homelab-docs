# Honeypot Lab

## 1. Goal

I'm building an isolated honeypot network in the Proxmox lab, attacking it from my own Kali VM, and detecting and reporting on what it records. The full plan is the Obsidian note "Honeypot Lab Plan".

## 2. Environment

| Item | Value |
|---|---|
| Isolated bridge | vmbr1 (no bridge port, no host IP) |
| Honeypot VM | ID 101, `honeypot`, Ubuntu Server 24.04.3 LTS, 2 vCPU, 2 GB RAM, 20 GB disk on local-lvm (root volume extended to 18 GB) |
| Admin NIC | ens18 on vmbr0, DHCP, currently 192.168.0.x (no router reservation yet) |
| Trap NIC | ens19 on vmbr1, static 10.10.10.10/24 (`/etc/netplan/60-trap.yaml`), nothing listening on it yet |
| Real SSH | port 22222, password login, set through an `ssh.socket` override |
| Firewall | ufw: deny incoming, allow outgoing, allow 22222/tcp in on ens18 only |
| Installed | docker.io, qemu-guest-agent, ufw, jq, tcpdump, python3-venv |
| Snapshot | `clean-base` (20-11-2026, no RAM) |
| Kali VM | _TBD_ (planned ID 102, vmbr1 only, 10.10.10.20/24) |

## 3. Steps taken

1. **Isolated bridge (20-11-2026):** created `vmbr1` under Datacenter -> John -> System -> Network with no bridge port and no IP, then applied the configuration.
2. **Snapshot:** took a snapshot of DC01 called `pre-honeypot-lab` (no RAM) before starting.
3. **Honeypot VM:** created VM 101 on `vmbr0` (it needs the internet for updates and Docker), installed Ubuntu Server 24.04.3 LTS with OpenSSH server, then set the CD/DVD drive to no media.
4. **Updates and tools:** upgraded all packages, installed `qemu-guest-agent docker.io ufw jq tcpdump python3-venv`, checked Docker with `hello-world`, and extended the root volume with `sudo lvextend -r -l +100%FREE /dev/ubuntu-vg/ubuntu-lv`.
5. **Real SSH moved to port 22222:** on Ubuntu 24.04 the port is owned by `ssh.socket`, so I added `/etc/systemd/system/ssh.socket.d/override.conf`:
```ini
[Socket]
ListenStream=
ListenStream=0.0.0.0:22222
ListenStream=[::]:22222
```
   then ran `sudo systemctl daemon-reload` and `sudo systemctl restart ssh.socket`. IPv4 and IPv6 need separate lines, otherwise IPv4 clients are refused.
6. **Firewall:**
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow in on ens18 to any port 22222 proto tcp
sudo ufw enable
```
7. **Trap NIC:** added a second network device on `vmbr1` in Proxmox (VirtIO), then gave it a static address in `/etc/netplan/60-trap.yaml` (`ens19: addresses: [10.10.10.10/24]`, file mode 600) and ran `sudo netplan apply`.
8. **Snapshot:** took `clean-base` of VM 101 (no RAM) as the restore point for a hardened base.
9. **Still to do:** pin the admin address (router reservation or static), Kali VM, Docker run of Cowrie on `10.10.10.10:22`, attacks, my own Python honeypot, detection and write-up.

## 4. Verification

```bash
sudo ufw status verbose      # active, deny (incoming), 22222/tcp on ens18 ALLOW IN, IPv4 and IPv6
sudo ss -tlnp | grep ':22'   # 0.0.0.0:22222 and [::]:22222, owned by systemd
ip -br a                     # ens18 192.168.0.x/24, ens19 10.10.10.10/24
ip route                     # one default route via 192.168.0.1 on ens18
```

After the firewall was enabled, a fresh SSH login on port 22222 from the Mac still worked.

## 5. Lab vs production

- Real SSH uses password login on a home LAN. In production I'd use key-only logins and rate limiting.
- The admin address comes from DHCP. In production it would be a fixed reservation or a static address.
- The trap NIC and the admin NIC share one VM. In production the honeypot would sit on its own VLAN, with no path to management systems.

## 6. Rollback

Revert VM 101 to `clean-base`, or delete the VM. To remove the isolated bridge, delete `vmbr1` under System -> Network and apply the configuration.

## 7. Problems met

| Problem | Cause | Fix |
|---|---|---|
| Console blank for about 5 minutes after the install reboot | Unknown, cleared on its own or after a reset | Waited and reset |
| `ssh` to 192.168.0.x refused | DHCP had changed the VM's address to 192.168.0.x | Used the new address |
| Changing `Port` in sshd config did nothing | `ssh.socket` owns the port on Ubuntu 24.04 | Socket override (step 5) |
| SSH stopped listening completely | I masked `ssh.socket`, but the service depends on it | `systemctl unmask ssh.socket`, then the override |
| New port refused over IPv4 | Override listed only `22222`, which bound IPv6 only | List `0.0.0.0:22222` and `[::]:22222` separately |
| `ens18` showed no IPv4 right after `netplan apply` | DHCP lease takes a few seconds | Waited about 20 seconds |
| `ssh` timed out | Connected to port 22 without `-p 22222`; ufw drops it silently | Used `-p 22222` |
