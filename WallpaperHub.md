# 📝 Proving Grounds Practice: WallpaperHub

## 🔍 Target Overview
* **Box Name:** WallpaperHub
* **Platform:** OffSec Proving Grounds (Practice)
* **OS:** Linux (Debian)
* **Target IP:** 192.168.58.204

## 1. Reconnaissance

### Nmap Scan
[cite_start]We start with a standard Nmap scan to enumerate open ports and services[cite: 71].

```bash
nmap -p- -sC -sV 192.168.60.204
