# Git Happens

Room [link](https://tryhackme.com/room/githappens). 

## Scanning 
I ran an NMAP scan to check out what was located on the machine. 
```
ajread@ajread-laptop:~/ctf/tryhackme$ nmap -A [TARGET IP]
Starting Nmap 7.92 ( https://nmap.org ) at 2023-02-27 21:14 EST
Nmap scan report for [TARGET IP]
Host is up (0.078s latency).
Not shown: 999 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
80/tcp open  http    nginx 1.14.0 (Ubuntu)
|_http-title: Super Awesome Site!
|_http-server-header: nginx/1.14.0 (Ubuntu)
| http-git: 
|   [TARGET IP]:80/.git/
|     Git repository found!
|_    Repository description: Unnamed repository; edit this file 'description' to name the...
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.61 seconds
```
I navigated to the webpage hosted on port 80 and it was a basic login page. There was some obfuscated code at the bottom within the script section. However, it was more interesting to notice the git repo within ```/.git``` from the nmap scan. 

## Enumeration 
I wanted to pull down the git repo. So, I used [git-dumper](https://github.com/arthaud/git-dumper) and I was able to pull down the entire repo to my local machine. 
```
ajread@ajread-laptop:~/ctf/tryhackme$ ls -la
total 44
drwxrwxr-x 4 ajread ajread 4096 Feb 27 21:06 .
drwxrwxr-x 4 ajread ajread 4096 Feb 27 21:00 ..
drwxrwxr-x 2 ajread ajread 4096 Feb 27 21:06 css
-rw-rw-r-- 1 ajread ajread 3775 Feb 27 21:06 dashboard.html
-rw-rw-r-- 1 ajread ajread 1115 Feb 27 21:06 default.conf
-rw-rw-r-- 1 ajread ajread  120 Feb 27 21:06 Dockerfile
drwxrwxr-x 7 ajread ajread 4096 Feb 27 21:06 .git
-rw-rw-r-- 1 ajread ajread  792 Feb 27 21:06 .gitlab-ci.yml
-rw-rw-r-- 1 ajread ajread 6890 Feb 27 21:06 index.html
-rw-rw-r-- 1 ajread ajread   54 Feb 27 21:06 README.md
```
I used a recursive grep to search for something with the word "password," but I was unsuccessful. Sometimes developers forget to remove key information when using git. So, I used ```git log``` to check out previous commits. In one of the previous commits, I found the password! 
```
<script>
-      function login() {
-        let form = document.getElementById("login-form");
-        console.log(form.elements);
-        let username = form.elements["username"].value;
-        let password = form.elements["password"].value;
-        if (
-          username === "admin" &&
-          password === "[REDACTED]"
-        ) {
-          document.cookie = "login=1";
-          window.location.href = "/dashboard.html";
-        } else {
-          document.getElementById("error").innerHTML =
-            "INVALID USERNAME OR PASSWORD!";
-        }
-      }

```
And I was able to submit the flag! 
