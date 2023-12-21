---
title: Vulhub - CyberSploit2
author: HoangND
date: 2023-01-16 00:00:00 +0700
categories: [Vulhub]
tags: [vulhub]
---

CyberSploit2 writeup

# Link

[https://www.vulnhub.com/entry/cybersploit-2,511/](https://www.vulnhub.com/entry/cybersploit-2,511/)

# Description
![image](https://user-images.githubusercontent.com/61985236/212556697-ff5dcce5-8360-41fe-8ac3-2a40092b489d.png)


# Solve

Import the ova file to Virtual Box and firstly, we use `arp-scan` to determine the target's ip address

```bash
sudo arp-scan -I eth1 --localnet
```

![image](https://user-images.githubusercontent.com/61985236/212556789-f0824d88-795c-435e-add6-80bf567b80a5.png)

and then use `nmap` to scan open ports on the target

![image](https://user-images.githubusercontent.com/61985236/212556817-c8364d3c-8631-4c45-83f2-e10de6151587.png)

The target has apache web server running on port 80. When access the server we have a table that seems to be credentials. I noted the table and then do some fuzzing on web server.
I got no sensitive asset from webserver by using popular wordlist like seclist or rockyou, maybe we can find some interesting informations or hints from web source? Open the get request in BurpSuite and grep html comment, i saw a hint like bellow

![image](https://user-images.githubusercontent.com/61985236/212557053-b4c30b73-2fa6-4c6f-9c9e-fa232300e16a.png)

We got `ROT47` cipher, what does this mean? Maybe we have some `ROT47` encoded data somewhere? Get back to the table, i found the `ROT47`

![image](https://user-images.githubusercontent.com/61985236/212557260-be5c3f7b-cd9f-461a-bd5f-e13a877ba46a.png)

Decode 2 strings we got the ssh credential.
- `D92:=6?5C2` -> `shailendra`
- `` 4J36CDA=@:E` `` -> `cybersploit1`

Use the above credential to login `shailendra`

![image](https://user-images.githubusercontent.com/61985236/212557862-add31f7f-7df2-4b5e-b171-e711adcd0090.png)

# PRIVELEGE ESCALATION
We see that `shailendra` is not in sudo group. Firstly, we will find out that what kind of files we have write permission. The command i used to use is `find / -type f -writable 2>/dev/null`.
No writable file can used to get root here, but i got another interesting file in the result.

![image](https://user-images.githubusercontent.com/61985236/212558052-9c6d38d9-ecf4-4297-ad4f-853c0808b3aa.png)

That's hint file in the home folder

![image](https://user-images.githubusercontent.com/61985236/212558115-4f9d00fd-4226-489e-9ef2-99d6f1ff0c32.png)

`shailendra` user is really in `docker` group, easily mount `/etc` folder to one container and then edit shadow file to change root's password. An example for mouting `/etc` folder to volume of a container is as the following

```bash
docker run -v /etc:/mnt -it alpine
```

![image](https://user-images.githubusercontent.com/61985236/212558554-f80ba76f-db1f-4d2b-a076-5ac90c18ac8a.png)

