---
layout: post
title: "[HTB] Dog"
tags: [git-dumper, file-upload]
difficulty: Easy
status: Complete
updated_date: 2025-07-13
---
***

## Quick Hits:

| Info | Details |
| ---- | ------- |
| **IP Address** | 10.10.11.58 |
| **Difficulty** | 🟢 Easy |
| **Exploits** | File Upload |
| **Tools Used** | `git-dumper` |
| **Techniques Used** |  |
| **Interesting Files** |  |

#### Port Scan Results:

| Port | State | Service | Version |
| ---- | ----- | ------- | ------- |
| 22/tcp | open | ssh | OpenSSH 8.2p1 Ubuntu 4ubuntu0.12 (Ubuntu Linux; protocol 2.0) |
| 80/tcp | open | http | Apache httpd 2.4.41 ((Ubuntu)) |

***

## Walk-through

Browsing to the site gives us a Dog Contest website, very similar to a recent box aptly named Cat. On this previous box, we found a `.git` directory, and sure enough: we find the same thing here. It seems like it was made by the same people.

We can grab the `.git` directory contents by running the command below:

```
git-dumper http://dog.htb/.git doggit
```

Just our luck! We find a set of credentials in one of the files. `settings.php`

```
tiffany@dog.htb : BackDropJ2024DS2024
```

Unfortunately, we can’t use this to login to the box just yet. So, let’s dig into the app once again. Where can we use this? We find a login page for the backend Content Management System (CMS) called Backdrop CMS. And, score! We’re logged in as an admin of the CMS.

#### Unrestricted File Upload

Additionally, we find that the Backdrop CMS version is version v1.22, which seems to be vulnerable to an Unrestricted File Upload vulnerability in the site themes. Diving into the source, it seems that when we upload a theme, it unpacks the code into a directory which is accessible to us. Browsing the `themes` directory confirms this issue, as we can see the other themes currently installed.

To exploit, we can create a reverse shell `php` script and compress it into a `.zip` file. As a site admin, we can navigate to `Appearance` > `Install new themes` > `Manual installation` and then upload our `zip` archive through the `Upload a module, theme or layout archive to install` section. Once successfully installed, we can start the listener from our machine to accept the reverse shell connection. When we browse to the `themes` directory, we can find and open our reverse shell script and receive the connection in our listener. Browsing the `home` directory, it seems that we don’t have user access yet, so we can try switching to the `johncusack` user and use the only password we have from `tiffany`. **And, we’re in!**

#### Privilege Escalation

Running the `sudoers` command `sudo -l` gives us a lead: with our current user being able to run `/usr/local/bin/bee` with `root` permissions. This is a command-line utility for Backdrop CMS.

```
-bash-5.0$ sudo -l
[sudo] password for johncusack: 
Matching Defaults entries for johncusack on dog:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User johncusack may run the following commands on dog:
    (ALL : ALL) /usr/local/bin/bee
```

We are able to run `eval` using this utility and run `/bin/bash` as `root`. However, when I tried, I encountered an error. This can be resolved by running the command from the `/var/www/html` directory or by specifying it in the `—-root` option from the command. **And, we’re done!**