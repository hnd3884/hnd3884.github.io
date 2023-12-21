---
title: Vulhub - EvilBox One
author: HoangND
date: 2023-01-03 00:00:00 +0700
categories: [Vulhub]
tags: [vulhub]
---

EvilBox One writeup

# Set up
Download ova file from this [link](https://www.vulnhub.com/entry/evilbox-one,736/) and import it to Oracle VirtualBox.

# Recon
Firstly, we use `arp` command to explore target's ip address

```bash
arp -a
```

The target's ip address is 192.168.56.101

![image](https://user-images.githubusercontent.com/61985236/210299714-ff3664da-58ff-46f2-bd4e-ece93bddbb97.png)

Then we use `nmap` to discover open ports in the target

```bash
nmap -Pn 192.168.56.101 -sV
```

![image](https://user-images.githubusercontent.com/61985236/210299884-91d5558c-ee6b-42ac-b183-1ac31b3d674f.png)

so the target is running apacher version 2.4.38 and SSH server is enabled. Go to http://192.168.56.101 and see the default page of apache

![image](https://user-images.githubusercontent.com/61985236/210300062-b3cbe343-e45a-49be-8500-e1eabbe7400f.png)

Use `fuff` to fuzz endpoints of Apache, the following command use `-c` for color, `-e` for file extension and `-recursion` to enable recursive fuzzing. I used to use seclist web-discovery for word list.

```bash
ffuf -c -u http://192.168.56.101/FUZZ -w /usr/share/seclists/Discovery/Web-Content/raft-small-words-lowercase.txt -e '.php' -recursion
```

Apache has endpoint `/secret`, and because of `-recursion` option, ffuf adds fuzzing from `/secret` to the queue, then it found `evil.php` file in this route.

![image](https://user-images.githubusercontent.com/61985236/210381413-51a8b0d1-138a-40d3-b548-b5c423daea89.png)

![image](https://user-images.githubusercontent.com/61985236/210381389-1953451d-112e-4971-abd4-48be485edcab.png)

Go to `/secret/evil.php`, it is blank page

![image](https://user-images.githubusercontent.com/61985236/210381578-2553bc0b-d222-4f6d-8240-e5f0d1c0d4cb.png)

Next we will use __ffuf__ to fuzz parameters of `evil.php`, `-X` option for request method, `-fs` to set negative size to 0

```bash
ffuf -c -u http://192.168.56.101/secret/evil.php?FUZZ=/etc/passwd -X POST -w /usr/share/seclists/Discovery/Web-Content/raft-small-words-lowercase.txt -fs 0
```

and we see that `evil.php` handles `command` from request parameters

![image](https://user-images.githubusercontent.com/61985236/210383145-4269b9ba-f33a-4068-967a-b0e315ccfa6b.png)

Open `http://192.168.56.101/secret/evil.php?FUZZ=/etc/passwd` in the browser and it shows `passwd` content

![image](https://user-images.githubusercontent.com/61985236/210384877-f29a42e8-c5e1-47fb-bf61-a66b09d6a42b.png)

# RCE

From this path traversal, we use the seclist to dump sensitive files from host. But some files are in user's home directory, so first we fuzz current user by using `.bashrc` file

```bash
ffuf -c -u http://192.168.56.101/secret/evil.php?command=/home/FUZZ/.bashrc -w users.txt -fs 0 -v
```

![image](https://user-images.githubusercontent.com/61985236/210389326-8f08ce01-5870-4327-9261-2368392cfbdd.png)

so here we see the current user running Apache is `mowree`, from this we can dump private ssh key from `/home/mowree/.ssh/id_rsa`

![image](https://user-images.githubusercontent.com/61985236/210389611-7f8fba8c-9858-432a-9847-ab7de4b3ff7c.png)

Change permission of id_rsa to 600, then try to use this SSH private key to login `mowree` but openssh asks for passphrase for key. Maybe we can crack this private key? Save id_rsa to kali machine, use the following commands to crack private key

```bash
# create hash file from key
ssh2john key.pk > key.hash
# crack hash
john --wordlist=darkweb2017-top10.txt id_rsa.hash
```

![image](https://user-images.githubusercontent.com/61985236/210398619-c7900a4a-271a-45a2-8901-6db26eb69996.png)

See that the passphrase of private key is `unicorn`, use it to access target machine

![image](https://user-images.githubusercontent.com/61985236/210401904-019f741d-5c4d-49a5-b8a6-1fdf4ae410ea.png)

# Privilege escalation

Use `find` command to find files that `mowree` has write permission

```bash
find / -type f -writable 2>/dev/null
```

![image](https://user-images.githubusercontent.com/61985236/210418248-ec5dcefa-e37d-4584-8c44-05410aa0f7f8.png)

You see that `mowree` can write to `/etc/passwd`, easily use `openssl` to generate hash for password. For example, the following command is used to generate hash for password `root`

```bash
openssl passwd root
```

Put the output hash to passwd file

![image](https://user-images.githubusercontent.com/61985236/210627201-b4e2c2c7-506e-4c4e-8cdd-e25fcefca631.png)

and then access to root user with password `root`

```bash
su root
```

![image](https://user-images.githubusercontent.com/61985236/210627509-6e00b31a-6776-46ba-a658-07319fd43e45.png)




