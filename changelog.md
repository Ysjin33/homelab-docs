# Changelog

| Date | Change |
|---|---|
| 25-11-2026 | Rewrote the honeypot page so it can be followed from scratch: added a status box, design diagram, the Cowrie steps, the login rules, a larger problems table, next steps, safety rules and a glossary. Filled in the README network diagram |
| 25-11-2026 | Started Cowrie 3.1.8 in Docker on the honeypot VM, published only on the trap address (10.10.10.10:22), read its log with jq, and took snapshot cowrie-running. Problems met: the image has no find, docker export hides volumes, docker logs shows no events, and a client config file made ssh on the VM use port 22222 |
| 20-11-2026 | This repo is public, so I removed the honeypot's admin IP and login-method details from the docs; admin addresses are kept in my private notes only |
| 20-11-2026 | Hardened VM 101: updates, Docker and tools, root volume extended, SSH moved to port 22222, ufw enabled, trap NIC (ens19, 10.10.10.10/24) added on vmbr1, snapshot clean-base taken |
| 20-11-2026 | Created VM 101 (honeypot, Ubuntu Server 24.04.3 LTS) on vmbr0 for the honeypot lab; not hardened yet |
| 20-11-2026 | Created isolated bridge vmbr1 (no ports, no IP) and took DC01 snapshot pre-honeypot-lab for the honeypot lab |
| 19-11-2026 | Configured account lockout policy (5 attempts, 15 min lockout) in the Default Domain Policy and tested with a throwaway user |
| 19-11-2026 | Unlocked an account and reset a password for a throwaway user, and wrote the unlock and reset runbooks |
| 19-11-2026 | Created test security group SG-Lab-Tester, added and removed a test user (GUI and PowerShell), and wrote the add-user-to-group runbook |