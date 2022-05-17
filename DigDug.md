## Dig Dug 

Room link: https://tryhackme.com/room/digdug

## Scanning 
A basic NMAP scan showed that only port 22 was open on the machine.
```
ajread@aj-ubuntu:~$ nmap -A [Remote IP] 
Starting Nmap 7.80 ( https://nmap.org ) at 2022-05-16 20:58 EDT
Nmap scan report for [Remote IP] 
Host is up (0.100s latency).
Not shown: 998 closed ports
PORT      STATE    SERVICE VERSION
22/tcp    open     ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 (Ubuntu Linux; protocol 2.0)
51493/tcp filtered unknown
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.80 seconds
```
Port 53, for DNS, was closed, which I thought was weird. 
```
ajread@aj-ubuntu:~$ nmap -p53 [Remote IP]
Starting Nmap 7.80 ( https://nmap.org ) at 2022-05-16 21:03 EDT
Nmap scan report for [Remote IP]
Host is up (0.100s latency).

PORT   STATE  SERVICE
53/tcp closed domain

Nmap done: 1 IP address (1 host up) scanned in 0.25 seconds
```
## Enumerating
I knew that I needed to use dig commands based on the challenge prompt. There are a variety of DNS record types. I figured the best one to try out first was TXT, since I have seen TXT on CTFs before. I also needed to make sure I pointed at the right DNS server using the ```@``` option in ```dig```.
```
ajread@aj-ubuntu:~$ dig givemetheflag.com @[Remote IP] txt

; <<>> DiG 9.16.15-Ubuntu <<>> givemetheflag.com @[Remote IP] txt
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 60721
;; flags: qr aa; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;givemetheflag.com.		IN	TXT

;; ANSWER SECTION:
givemetheflag.com.	0	IN	TXT	"[REDACTED]"

;; Query time: 100 msec
;; SERVER: [Remote IP]#53([Remote IP])
;; WHEN: Mon May 16 20:57:10 EDT 2022
;; MSG SIZE  rcvd: 86
```
And, I was right! The flag was located within the TXT record. 

But I could have also used ```nslookup``` to get the flag as well. 
```
ajread@aj-ubuntu:~$ nslookup -Type=TXT givemetheflag.com [Remote IP]
Server:		[Remote IP]
Address:	[Remote IP]#53

givemetheflag.com	text = "[REDACTED]"
```