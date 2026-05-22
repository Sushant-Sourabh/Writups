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

**NMAP**
```bash
# Nmap Initial Scan
nmap -p- -sV -sC -T4 192.168.58.42
```

<img width="861" height="424" alt="image" src="https://github.com/user-attachments/assets/aff36a12-348e-464f-8cd5-695ff7dbd0a5" />

**Gobuster**
```
gobuster dir -u http://192.168.58.42 -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -x php,txt,bak
```

<img width="864" height="411" alt="image" src="https://github.com/user-attachments/assets/989e5530-5f13-4746-8bab-8fa87b317e79" />

**PORT 80 - Webpage**
No robots.txt available, neither anything mentioned in the view source only one ASCII code in the webpage

<img width="1300" height="153" alt="image" src="https://github.com/user-attachments/assets/eaa8c4ce-f5b0-4925-8e16-9ca59b24041d" />

changed ASCII to words 
<img width="642" height="260" alt="image" src="https://github.com/user-attachments/assets/1b602cff-9a93-496b-858a-fa7131e22984" />

## 🔓 Phase 2: Initial Foothold

**PORT 25 - SMTP - Sendmail 8.13.4**
```
Searchsploit Sendmail
```

<img width="879" height="642" alt="image" src="https://github.com/user-attachments/assets/603b1c82-f818-4638-890d-e1cca81222a7" />

Got one exploit with clamAV which has RCE

checked for its code
<img width="863" height="520" alt="image" src="https://github.com/user-attachments/assets/025d16d6-1bc2-484e-bb55-a852b8c2af91" />

THe code is mentioned to open the port 31337 with root access

---

## 💥 Phase 3: Exploitation

This looked promising, even though I don't have a ClamAV version, but the pieces are all there – ClamAV and Sendmail.
I copy the exploit to current directory and inspect the source code. It uses the ClamAV milter (filter for Sendmail), which appears to not validate inputs and run system commands. The exploit opens up a socket on 31337 and allows the attacker to send I/O through the socket. It only needs one argument -- the target IP

Started a netcat listner
```
nc -nv 192.168.58.42 31337
```
Then initiated the exploit 

<img width="860" height="407" alt="image" src="https://github.com/user-attachments/assets/9b0ad555-bf09-49a6-8292-8330b2840571" />

The connection was made and i got a shell. When checked it was a root shell.

<img width="595" height="82" alt="image" src="https://github.com/user-attachments/assets/77acf71c-905a-4ee1-999b-fbbad28a00ef" />




