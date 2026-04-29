# Initial Access Pot

Room link [here](https://tryhackme.com/room/initialaccesspot) 

1. Which web page did the attacker attempt to brute force?
**Answer**: ```wp-login.php```
* Look at /var/log/apache2/access.log for details on the brute forece. Will look like ```167.172.41.141 - - [27/Jun/2025:21:20:42 +0000] "POST /wp-login.php HTTP/1.0" 200 5244 "-" "Mozilla/5.0 (Hydra)"```

2. What is the absolute path to the backdoored PHP file?
**Answer**: ```/var/www/html/wordpress/wp-content/themes/blocksy/404.php```. 
* Look through the web traffic in access.log and see: 
	```167.172.41.141 - - [27/Jun/2025:21:31:51 +0000] "POST /wp-admin/admin-ajax.php HTTP/1.1" 200 595 "http://demo-web.deceptitech.thm/wp-admin/theme-editor.php?file=404.php&theme=blocksy" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/138.0.0.0 Safari/537.36"```. The data shows the theme update for the blocksy theme and file 404.php. 


3. Which file path allowed the attacker to escalate to root?
*Answer*: ```/etc/ssh/id_ed25519.bak```
* Look through the /etc folder for ssh creds. I was able to read through them as regular user. This readability can be confirmed using ```ls -la ``` within the directory. Can see authentication logs as root as well within access.log to confirm that it is the pair that was used: ```2025-06-27T22:08:07.701837+00:00 deceptipot-demo sshd[3134]: Accepted publickey for root from 127.0.0.1 port 40710 ssh2: ED25519 SHA256:V3R2H0epBUGDZ7A64uK15wJdhyhH9ifINoxFUWxR6KQ```. 

4. Which IP was port-scanned after the privilege escalation?
*Answer*: 172.16.8.216
* Bash history for root can show the scannning below: 
```
for ip in 172.16.8.{200..254}; do ping -c1 ${ip} & done
nc -w 2 -v 172.16.8.216 22
nc -w 2 -v 172.16.8.216 80
nc -w 2 -v 172.16.8.216 3389
nc -v 172.16.8.216 3389
```
* Guessed on this one based on the background information provided and the network map. Didnt find any evidence or logs. It should be from the kworker logs. Bash history for root can show the scannning below: 


5. What is the MD5 hash of the malware persisting on the host?
*Answer*: d6f2d80e78f264aff8c7aea21acb6ca6
* Focusing on kworker, can find that it is run within sbin which is not a normal location for the binary. In order to get the MD5, just need to run ```md5sum```. 

6. Can you access the DeceptiPot in recovery mode?
*Answer* : Run with ```sudo /usr/bin/deceptipot -r Em1lyR0ss_DeCePti!```. 
* Find the deceptipot.conf file and check for the recovery key. The conf file should also contain the instructions for bringing the system up and running. 