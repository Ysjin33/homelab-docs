# Honeypot Lab

> **Status (26-11-2026): Week 1 of 3 in progress.** The isolated network, the honeypot VM and Cowrie are built and recording, and I've confirmed what Cowrie logs for a rejected login and for fake-shell commands. The Kali attacker VM is created and waiting for its install. Still to do: finish Kali, my own Python honeypot, a detection rule and the final write-up. See [section 8](#8-where-this-project-stands-and-whats-next).

## 1. Goal

I'm building an isolated honeypot network in the Proxmox lab, attacking it from my own Kali VM, and detecting and reporting on what it records. The plan is three weeks: Week 1 builds the lab and runs Cowrie, Week 2 builds my own honeypot in Python, Week 3 adds detection, analysis and a report.

**What is a honeypot?** A decoy computer that looks like a real target but has no real use. Nothing legitimate should ever connect to it, so every connection is suspicious, and everything an attacker tries gets logged. Security teams use honeypots to learn attacker addresses, passwords and commands.

**What is Cowrie?** An open-source "medium interaction" SSH honeypot. It accepts logins, shows a fake Linux shell and records every command typed, but nothing typed there runs on the real machine.

## 2. Design

```
 Home LAN (vmbr0) ───────────────────────────────┐
                                                 │ admin only: real SSH on port 22222
 Proxmox host "John"                             │
   ├─ DC01 (vmbr0)                               │
   ├─ VM 101 "honeypot"                          │
   │     ├─ ens18 (vmbr0)  admin NIC ────────────┘
   │     └─ ens19 (vmbr1)  trap NIC 10.10.10.10  <- Cowrie listens here on port 22 (Docker)
   └─ VM 102 "kali" (created, install in progress)
         └─ vmbr1 only, 10.10.10.20
 vmbr1 = isolated bridge: no physical port and no address on the host,
         so machines on it cannot reach the home LAN or the internet
```

One VM runs two different SSH servers on purpose:

| Port | Network card | Server | Purpose |
|---|---|---|---|
| 22222 | admin NIC (`ens18`) | the real OpenSSH server | how I manage the VM |
| 22 | trap NIC (`ens19`, `10.10.10.10`) | Cowrie, the fake one, in Docker | what attackers will find |

## 3. Environment

| Item | Value |
|---|---|
| Isolated bridge | vmbr1 (no bridge port, no host IP) |
| Honeypot VM | ID 101, `honeypot`, Ubuntu Server 24.04.3 LTS, 2 vCPU, 2 GB RAM, 20 GB disk on local-lvm (root volume extended to 18 GB) |
| Admin NIC | ens18 on vmbr0, DHCP on the home LAN (address not recorded here; no router reservation yet) |
| Trap NIC | ens19 on vmbr1, static 10.10.10.10/24 (`/etc/netplan/60-trap.yaml`) |
| Real SSH | port 22222, set through an `ssh.socket` override |
| Firewall | ufw: deny incoming, allow outgoing, allow 22222/tcp in on ens18 only |
| Honeypot software | Cowrie 3.1.8 (`cowrie/cowrie:latest`) in Docker, container `cowrie`, published only on `10.10.10.10:22` |
| Installed | docker.io, qemu-guest-agent, ufw, jq, tcpdump, python3-venv |
| Snapshots | `clean-base` (20-11-2026), `cowrie-running` (25-11-2026) |
| Kali VM | ID 102, `kali`, Kali Linux installer (amd64), 2 vCPU, 4 GB RAM, 40 GB disk on local-lvm. Created 26-11-2026 on vmbr0 for the install; install not finished yet. Planned: vmbr1 only, 10.10.10.20/24 |

## 4. Steps taken

Run the commands in steps 4 onward **on the honeypot VM**, not on your own computer. Check the prompt before pasting.

### Session 1 and 2 (20-11-2026): the lab and a hardened VM

1. **Isolated bridge:** created `vmbr1` under Datacenter -> John -> System -> Network with no bridge port and no IP, then applied the configuration.
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
   then ran `sudo systemctl daemon-reload` and `sudo systemctl restart ssh.socket`. IPv4 and IPv6 need separate lines, otherwise IPv4 clients are refused. Keep the first SSH session open until a second one works on the new port.
6. **Firewall:**
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow in on ens18 to any port 22222 proto tcp
sudo ufw enable
```
7. **Trap NIC:** added a second network device on `vmbr1` in Proxmox (VirtIO), then gave it a static address in `/etc/netplan/60-trap.yaml` (`ens19: addresses: [10.10.10.10/24]`, file mode 600) and ran `sudo netplan apply`.
8. **Snapshot:** took `clean-base` of VM 101 as the restore point for a hardened base.

### Session 3 (25-11-2026): Cowrie

9. **Started Cowrie**, published only on the trap address:
```bash
sudo docker run -d --name cowrie --restart unless-stopped \
  --log-opt max-size=10m --log-opt max-file=3 \
  -p 10.10.10.10:22:2222 cowrie/cowrie:latest
sudo ss -tlnp | grep -E ':22\b'
sudo docker ps
```
   - Cowrie listens on port 2222 inside the container; Docker maps `10.10.10.10:22` to it.
   - The address in `-p` matters: **Docker's published ports bypass ufw**, so the bind address is the real control. Binding to `10.10.10.10` keeps Cowrie off the admin NIC.
   - Expected result: one line showing `10.10.10.10:22` owned by `docker-proxy`, nothing on `0.0.0.0:22`, and `docker ps` showing `Up`. `--restart unless-stopped` brings it back after a VM reboot.
10. **Which logins Cowrie accepts.** There is no `userdb.txt`, so Cowrie uses its built-in defaults, which match `userdb.example` (copy it out to read it: `sudo docker cp cowrie:/cowrie/cowrie-git/src/cowrie/data/etc/userdb.example ./userdb-copy.txt`). Rules are read top to bottom and the first match wins:

    | Rule | Meaning |
    |---|---|
    | `root:x:!root`, `root:x:!123456`, `root:x:!/honeypot/i` | `root` is **rejected** with the passwords `root`, `123456`, or anything containing "honeypot" (case-insensitive) |
    | `root:x:*` | `root` is accepted with any other password |
    | `tomcat:x:*`, `oracle:x:*` | those users are accepted with any password |
    | `*:x:somepassword`, `*:x:*` | any username with any password is accepted |

    I predicted that `root` with a made-up password would be accepted, then tested it (step 12). It was.
11. **Where the log is and how to copy it out.** Events are written **only** to `cowrie.json` (one JSON object per line); `docker logs cowrie` shows start-up lines and nothing else. The file is `/cowrie/cowrie-git/var/log/cowrie/cowrie.json` inside the container, in an anonymous Docker volume. Copy it out and read it:
```bash
sudo docker cp cowrie:/cowrie/cowrie-git/var/log/cowrie/cowrie.json ~/cowrie.json
sudo chown "$USER": ~/cowrie.json
jq -r '[.timestamp, .eventid, .src_ip, .username, .password, .input] | map(. // "") | join("  ")' ~/cowrie.json
```
    Copy the log out **before** ever removing the container: a new container starts with a new, empty log volume.
12. **First test, from the VM itself.** `ssh -p 22 root@10.10.10.10` (the `-p 22` is needed, see problem 11 in section 7), a made-up password, then `exit`. Result and log contents are in section 5.
13. **Snapshot:** took `cowrie-running` of VM 101.

### Session 4 (26-11-2026): checking the log, then the Kali VM (in progress)

14. **Rejected login and fake-shell commands, from the VM itself.** `ssh -p 22 root@10.10.10.10`. First try: `root` with `123456` (predicted: rejected). Second try: a made-up password (predicted: accepted). Then in the fake shell: `uname -a`, `ls`, `cat /etc/passwd`, `exit`. Both predictions were right. I copied the log out and read it with the commands from step 11. Results are in section 5.
15. **Kali VM created (VM 102):** 2 vCPU, 4 GB RAM, 40 GB disk, network on `vmbr0` for the install, Kali installer ISO in the CD/DVD drive. My first ISO was the wrong build (arm64), see problem 13. I replaced it with the amd64 installer. The install and the move to the isolated bridge are the next steps (section 8).

## 5. Verification

```bash
sudo ufw status verbose        # active, deny (incoming), 22222/tcp on ens18 ALLOW IN, IPv4 and IPv6
sudo ss -tlnp | grep -E ':22\b|:22222'   # 22222 owned by sshd/systemd; 10.10.10.10:22 owned by docker-proxy
ip -br a                       # ens18 has its home-LAN DHCP address, ens19 10.10.10.10/24
sudo docker ps                 # cowrie is Up, ports 10.10.10.10:22->2222/tcp
```

**Is port 22 really Cowrie, not the real SSH server?** Compare host keys. Different fingerprints mean different servers:
```bash
ssh-keyscan -p 22 10.10.10.10 2>/dev/null | ssh-keygen -lf -
ssh-keyscan -p 22222 10.10.10.10 2>/dev/null | ssh-keygen -lf -
```

**What the first log showed** (30 events, `src_ip` is `10.10.10.10` because I tested from the VM itself; from Kali it will be `10.10.10.20`):

| Events | Cause |
|---|---|
| five short connections, each: `session.connect`, `client.version`, `client.kex`, `session.closed`, with no login | my `ssh-keyscan` probing the port. This is what a scanner looks like in a honeypot log |
| `session.connect`, `client.version`, `client.kex`, `login.success` (user `root`), `client.size`, `client.var`, `session.params`, `command.input` (`exit`), `log.closed`, `session.closed` | one manual login and logout |

**After the rejected-login test** (26-11-2026), counting the event types with `jq -r '.eventid' ~/cowrie.json | sort | uniq -c`:

| Event | Count |
|---|---|
| `cowrie.session.connect` | 7 |
| `cowrie.session.closed` | 7 |
| `cowrie.client.version` | 7 |
| `cowrie.client.kex` | 7 |
| `cowrie.command.input` | 6 |
| `cowrie.login.success` | 2 |
| `cowrie.session.params` | 2 |
| `cowrie.client.size` | 2 |
| `cowrie.client.var` | 2 |
| `cowrie.log.closed` | 2 |
| `cowrie.login.failed` | 1 |

The one `cowrie.login.failed` is `root` with `123456`, as predicted. Each line is JSON with `eventid`, `session`, `protocol`, `src_ip`, `src_port`, `dst_ip`, `dst_port`, `timestamp` and `message`, plus `username` and `password` on login events. Things worth knowing:

- `timestamp` is in UTC (it ends in `Z`), so convert it to local time when making charts.
- `dst_ip` and `dst_port` show the container's own address and port (2222), not the trap address.
- `src_ip` is still `10.10.10.10` because I tested from the VM itself. Docker's port forwarding should keep the real source address for a connection from Kali, but I haven't checked yet. That is the first test once Kali is on `vmbr1`.

Inside the fake shell, the prompt showed the hostname `svr04`, `uname -a` reported a Debian 6.1 kernel, and `cat /etc/passwd` listed a stock set of users including one called `phil`. That is Cowrie's built-in fake system, not the real VM, which runs Ubuntu.

Cowrie's configured fake banner is `SSH-2.0-OpenSSH_9.2p1 Debian-2+deb12u3` (from its bundled config; I have not yet seen it on the wire).

## 6. Lab vs production

- This is a home lab with no internet exposure. In production I'd use key-only logins, rate limiting and a management network separate from the honeypot.
- The admin address comes from DHCP. In production it would be a fixed reservation or a static address.
- The trap NIC and the admin NIC share one VM. In production the honeypot would sit on its own VLAN, with no path to management systems.
- Real internet exposure would bring real attackers, so it would need stricter isolation, monitoring and a legal check.
- Skilled attackers can recognise honeypots, for example by their behaviour or default settings. Cowrie's stock fake system (the hostname `svr04` and a user called `phil`) is one example. Changing it is a stretch goal.

## 7. Problems met

| # | Problem | Cause | Fix |
|---|---|---|---|
| 1 | Console blank for about 5 minutes after the install reboot | Unknown, cleared on its own or after a reset | Waited and reset |
| 2 | `ssh` refused on the address I'd noted | DHCP had given the VM a new address | Checked the current address with `ip -br a` on the VM console |
| 3 | Changing `Port` in sshd config did nothing | `ssh.socket` owns the port on Ubuntu 24.04 | Socket override (step 5) |
| 4 | SSH stopped listening completely | I masked `ssh.socket`, but the service depends on it | `systemctl unmask ssh.socket`, then the override |
| 5 | New port refused over IPv4 | Override listed only `22222`, which bound IPv6 only | List `0.0.0.0:22222` and `[::]:22222` separately |
| 6 | `ens18` showed no IPv4 right after `netplan apply` | DHCP lease takes a few seconds | Waited about 20 seconds |
| 7 | `ssh` timed out | Connected to port 22 without `-p 22222`; ufw drops it silently | Used `-p 22222` |
| 8 | `docker exec cowrie find ...` failed ("executable file not found") | The Cowrie image is stripped down: no `find` or `cat` inside | List files from outside with `sudo docker export cowrie \| tar -tf - \| grep <name>`, copy them with `docker cp` |
| 9 | `cowrie.json` missing from that file listing | `docker export` leaves out volumes, and the log folder is a volume | `sudo docker inspect -f '{{json .Mounts}}' cowrie` shows the volumes; `docker cp` can read from them. (The `GraphDriver` field is absent on this Docker version.) |
| 10 | `docker logs cowrie` showed no attacks | This Cowrie records events only in `cowrie.json` (`jsonlog` on, `textlog` off) | Read the JSON file with `jq` (step 11) |
| 11 | `ssh root@10.10.10.10` from the VM reached the real SSH server on 22222, not Cowrie: wrong banner, root refused, nothing logged | `/etc/ssh/ssh_config.d/10-port.conf` is a **client** config (`ssh_config.d`; the server's folder is `sshd_config.d`), so every `ssh` run on the VM defaults to port 22222 | Found with `ssh -v` ("Reading configuration data", "Connecting to ... port"). Use `ssh -p 22` for Cowrie tests from the VM. The file is still there and should be moved aside |
| 12 | I removed the container although its output was correct | Misread a correct result as a problem | Read the output first. `docker rm` also leaves a container's anonymous volumes behind, so an orphan volume with an old empty log remains; list with `sudo docker volume ls` |
| 13 | Kali VM would not boot the installer: "Boot failed: Could not read from CDROM (code 0009)", then it tried a network boot | I had downloaded the `arm64` Kali installer. The Proxmox host is an Intel machine, so it can't boot an ARM image. The download page can suggest the ARM build when browsing from an Apple Silicon Mac. The boot menu showed the disc as "Kali Linux arm64" | Removed that ISO and downloaded the `amd64` installer (the filename contains `amd64`), then selected it under VM 102, Hardware, CD/DVD Drive. Stop and start the VM afterwards so it picks up the change. Check the architecture in the filename before downloading |

## 8. Where this project stands and what's next

**Done**
- [x] Isolated bridge `vmbr1`, DC01 snapshot `pre-honeypot-lab`
- [x] Honeypot VM 101, hardened, real SSH on 22222, trap NIC on `vmbr1`, snapshot `clean-base`
- [x] Cowrie running on `10.10.10.10:22`, writing `cowrie.json`, snapshot `cowrie-running`
- [x] First login captured and read with `jq`
- [x] Rejected login and fake-shell commands tested, log fields confirmed (26-11-2026)
- [x] Kali VM 102 created with the amd64 installer attached (26-11-2026)

**Next (in order)**
1. Optional tidy: move `/etc/ssh/ssh_config.d/10-port.conf` aside; remove the leftover Docker volume from the first container (`sudo docker volume ls`, then remove it by name).
2. Week 1, Session 4 (in progress): VM 102 is created and the `amd64` installer is downloaded. Install Kali on `vmbr0`, update it, then switch its network card to `vmbr1` only and give it 10.10.10.20/24. Prove isolation: `ping 192.168.0.1` must **fail** and `ping 10.10.10.10` must work. Snapshot Kali (`clean-isolated`) and the honeypot (`pre-attack-tests`). Then make one login attempt from Kali and check that the log shows `10.10.10.20` as `src_ip`.
3. Attack the honeypot from Kali (`nmap`, then `hydra` with a wordlist that includes passwords Cowrie rejects, such as `root` and `123456`. With the default rules `hydra` stops at the first accepted password, so otherwise only one failure is logged and a brute-force rule would never fire).
4. Week 1, Session 5: read the evidence with `jq`.
5. Week 2: my own Python honeypot (SSH and HTTP), logging in the same JSON style. Week 3: detection rule, charts, MITRE ATT&CK mapping, attack day with a packet capture, and the final write-up.

## 9. Safety rules for this lab

- The honeypot and Kali stay on the isolated bridge `vmbr1`. No router port forwarding, no tunnels, no exposing it to the internet.
- Only attack `10.10.10.10` from Kali.
- Never type a real password into the honeypot. Cowrie records passwords in plain text. Use made-up ones.
- This repo is public: no admin addresses, login details, keys, real credentials or unredacted logs here.

## 10. Rollback

- Revert VM 101 to `cowrie-running` (Cowrie working) or `clean-base` (hardened, no Cowrie), or delete the VM.
- Remove Cowrie only: `sudo docker rm -f cowrie`. Copy the log out first. Add `-v` to also delete its volumes (this deletes the logs).
- Remove the isolated bridge: delete `vmbr1` under System -> Network and apply the configuration.

## 11. Glossary

| Term | Meaning |
|---|---|
| Bridge (`vmbr0`, `vmbr1`) | A virtual network switch inside Proxmox. `vmbr1` has no physical port, so it is cut off from everything else |
| NIC | Network interface card; here a virtual one |
| Trap NIC / admin NIC | The VM's card on the isolated bridge (where attackers connect) and its card on the home LAN (how I manage it) |
| Snapshot | A saved state of a VM that I can return to |
| Docker volume | Storage that lives outside a container's own filesystem. Cowrie's logs are in one |
| `jq` | A command-line tool for reading JSON |
| Honeypot / honeynet / honeytoken | A decoy system / a network of decoys / a decoy credential or file that should never be used |
