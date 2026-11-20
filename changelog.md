# Changelog

| Date | Change |
|---|---|
| 20-11-2026 | Hardened VM 101: updates, Docker and tools, root volume extended, SSH moved to port 22222, ufw enabled, trap NIC (ens19, 10.10.10.10/24) added on vmbr1, snapshot clean-base taken |
| 20-11-2026 | Created VM 101 (honeypot, Ubuntu Server 24.04.3 LTS) on vmbr0 for the honeypot lab; not hardened yet |
| 20-11-2026 | Created isolated bridge vmbr1 (no ports, no IP) and took DC01 snapshot pre-honeypot-lab for the honeypot lab |
| 19-11-2026 | Configured account lockout policy (5 attempts, 15 min lockout) in the Default Domain Policy and tested with a throwaway user |
| 19-11-2026 | Unlocked an account and reset a password for a throwaway user, and wrote the unlock and reset runbooks |
| 19-11-2026 | Created test security group SG-Lab-Tester, added and removed a test user (GUI and PowerShell), and wrote the add-user-to-group runbook |