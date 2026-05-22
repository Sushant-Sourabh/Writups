# 📝 Proving Grounds Practice: ClamAV

## 🔍 Target Overview
* **Box Name:** ClamAV
* **Platform:** OffSec Proving Grounds (Practice)
* **OS:** Linux (Debian)
* **Target IP:** 192.168.58.42

## 📊 Executive Summary
I initiated my enumeration by methodically investigating each open port identified in the Nmap scan. Upon checking the web service on port 80, I encountered a webpage displaying a block of ASCII characters. I decoded this block into plain text, which revealed the hidden string 'ifyoudontpwnmeuran0b'. Unfortunately, this turned out to be a rabbit hole and yielded no further actionable intelligence.

Pivoting my attack path away from the web server, I began researching known vulnerabilities associated with the SMTP service. My research led me to a public Remote Code Execution (RCE) exploit targeting a specific ClamAV configuration. I executed the exploit against the target, which surprisingly bypassed the need for a low-privileged foothold entirely—the payload executed successfully and dropped me directly into a root-level shell, fully compromising the machine.

---

## ⚔️ Phase 1: Enumeration

My methodology always begins with a comprehensive port scan to map the external attack surface. 

```bash
# Nmap Initial Scan
nmap -p- -sV -sC -T4 192.168.58.42
```

<img width="861" height="424" alt="image" src="https://github.com/user-attachments/assets/aff36a12-348e-464f-8cd5-695ff7dbd0a5" />



