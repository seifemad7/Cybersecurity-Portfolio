# Simple CTF – TryHackMe

## Target
A basic Linux machine with common web and privilege escalation issues.

## Steps
1. Nmap scan: Found HTTP and SSH
2. Directory brute-force: Found /robots.txt → hidden dir
3. Exploited weak password login for user
4. Found user.txt flag
5. Escalated via SUID binary
6. Root flag captured

## Flags
- user.txt: THM{userflag}
- root.txt: THM{rootflag}
