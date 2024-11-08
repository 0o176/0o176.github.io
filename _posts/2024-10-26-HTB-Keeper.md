---
layout: post
title: "HTB: Keeper"
---

# Introduction

Keeper is an easy Linux box on HackTheBox, and is based on finding default credentials to gain initial access to admin area and using user credentials found there to move forward.

![header](/docs/assets/img/HTB-Keeper/header.PNG)

# Recon

## nmap

As always, the first thing to do is to run a `nmap` scan.

```powershell
nmap -sV -sC -Pn -p- --min-rate=1000 {BOX_IP}
```

![nmap](/docs/assets/img/HTB-Keeper/nmap.PNG)

Two interesting open ports were found, 22 and 80. While **ssh** might be useful later, first focus on the website.

## Site Exploration

We are redirected to the login screen.

![web](/docs/assets/img/HTB-Keeper/web.PNG)

With no credentials we can try some common combinations but easy Google search for default credentials of the **Request Tracker** gives us the answer.

![creds](/docs/assets/img/HTB-Keeper/creds.PNG)

After login we can start to look for new hints. In the **Admin** section we can list users, and are lucky to find one with password listed right there.

![user](/docs/assets/img/HTB-Keeper/user.PNG)

# SSH

Now when we have valid credentials we can take a look at the port 22 and try to get in.

![ssh](/docs/assets/img/HTB-Keeper/ssh.PNG)

We are in, let’s grab the **user flag** and look what else is there.

![user_flag](/docs/assets/img/HTB-Keeper/user_flag.PNG)

Not much this user can do, but there is interesting **zip** file.

![unzip](/docs/assets/img/HTB-Keeper/unzip.PNG)

Looks promissing, download the files to our machine and explore further.

![download_files](/docs/assets/img/HTB-Keeper/download_files.PNG)

# KeePass

After googling for a bit I found what to do with the files.

There is documented vulnerability [CVE-2023–32784](https://nvd.nist.gov/vuln/detail/CVE-2023-32784) and an existing functional exploit available [HERE](https://github.com/vdohney/keepass-password-dumper).

![pass1](/docs/assets/img/HTB-Keeper/pass1.PNG)

There was some issue with .NET version, just switch to **6.0** to run smoothly.

![pass2](/docs/assets/img/HTB-Keeper/pass2.PNG)

The first character is unknown, the second character has multiple options and after that the characters are _“dgrød med fløde”_. So we need first two characters to obtain the password.

Taking a quick look on the word _“dgrød med fløde”_, we can see search result showing _“rødgrød med fløde”_.

![ss](/docs/assets/img/HTB-Keeper/ss.PNG)

We can use it to get into the **passcodes.kdbx** file we downloaded earlier.

![keepass](/docs/assets/img/HTB-Keeper/keepass.PNG)

It worked!

Once we are inside we can take a look at root user. In the notes there is a `PuTTY-User-Key-File`.

![keepass2](/docs/assets/img/HTB-Keeper/keepass2.PNG)

![keepass3](/docs/assets/img/HTB-Keeper/keepass3.PNG)

# Root

That can be used to generate private **SSH key**. Let’s save it to .ppk file and generate the key using `puttygen`.

![keepass4](/docs/assets/img/HTB-Keeper/keepass4.PNG)

![id_rsa](/docs/assets/img/HTB-Keeper/id_rsa.PNG)

With the **id_rsa** we get we can log in as a root user and get the flag.

![root](/docs/assets/img/HTB-Keeper/root.PNG)

~
