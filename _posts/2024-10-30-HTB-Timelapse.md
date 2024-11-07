---
layout: post
title: "HTB: Timelapse"
toc: true
toc_sticky: true
---

{% include toc.html %}

Timelapse, an easy-rated Active Directory machine from Hack The Box, starts by finding a key on an SMB share. After cracking it open and authenticationg via Evil-WinRM, we’ll find credentials in the PowerShell history file for the next user. That useris a member of **LAPS_Readers** group, which allows us to pull out local administrator passwords. With that, we’ll get the administrator password and use Evil-WinRM to get a shell.

# Recon

## nmap

As always, start with nmap scan:
```powershell
sudo nmap -sC -sV -p- -v -oA nmap_scan/nmap_results 10.129.116.41
```
nmap finds 18 open TCP ports:
![nmap](/docs/assets/img/HTB-Timelapse/nmap.PNG)

The combination of ports (Kerberos + LDAP + DNS + SMB) suggest this should be a domain controller. Plus there is the name on cert on 5986 (dc01.timelapse.htb).

## SMB - TCP 445

Take a look at SMB, first use `netexec` or `smbclient` and find if null session is allowed.

```powershell
netexec smb 10.129.116.41 --shares -u 'guest' -p ''
```

![SMB](/docs/assets/img/HTB-Timelapse/smb1.png)

It is and we have 2 readable shares.
Next use `smbclient` to tak a closer look inside 

```powershell
smbclient -N //10.129.116.41/Shares
```
![SMB2](/docs/assets/img/HTB-Timelapse/smb2.png)

There is an interesting file in a **Shares** share, **winrm_backup.zip**. Download it and take a look at it on out machine.

![SMB3](/docs/assets/img/HTB-Timelapse/smb3.png)

There were few more files in the **\helpdesk** folder, but there were no use at the end.

# Shell as legacyy

## Exploring winrm_backup.zip

The .zip file we downloaded is password protected. We have to get good old **john** to help there.
Use `zip2john` to get a hash and crack it.

```powershell
zip2john winrm_backup.zip > ziphash.txt
```

![zip_hash](/docs/assets/img/HTB-Timelapse/zip_hash.png)

```powershell
john ziphash.txt --wordlist=~/Tools/rockyou.txt
```

![zip_hash2](/docs/assets/img/HTB-Timelapse/zip_hash2.png)

And we have out first password! **supremelegacy**

The zip contains single file: **legacyy_dev_auth.pfx**

![unzip](/docs/assets/img/HTB-Timelapse/unzip.png)

## Extracting certificate and key from a .pfx file

The .pfx file, which is in a PKCS#12 format, contains the SSL certificate (public keys) and the corresponding private keys. Sometimes, you might have to import the certificate and private keys separately in an unencrypted plain text format to use it on another system. This topic provides instructions on how to convert the .pfx file to .crt and .key files. [More here](https://www.ibm.com/docs/en/arl/9.7?topic=certification-extracting-certificate-keys-from-pfx-file).

When we try to extract ... there is a password again, and unfortunatelly not supremelegacy again :)

![pfx1](/docs/assets/img/HTB-Timelapse/pfx1.png)

We have to reach for `john` again, this time `pfx2john`

```powershell
pfx2john legacyy_dev_auth.pfx > pfx_hash
```
![pfx2](/docs/assets/img/HTB-Timelapse/pfx2.png)

```powershell
john pfx_hash --wordlist=~/Tools/rockyou.txt
```

![pfx3](/docs/assets/img/HTB-Timelapse/pfx3.png)

And we get second password **thuglegacy**

With that we can try to extract cert and key again:

```powershell
openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out pfx_key.key
```

![key1](/docs/assets/img/HTB-Timelapse/key1.png)

It will require you to input PEM pass phrase, you can put whatever you want there (at least 4 chars)

```powershell
openssl pkcs12 -in legacyy_dev_auth.pfx -clcerts -nokeys -out pfx_cert.crt
```

![key2](/docs/assets/img/HTB-Timelapse/key2.png)

## Logging in

We now have everything we need to finally get into the machine.

Use `evil-winrm` and freshly extracted `pfx_key.key` and `pfx_cert.crt` to get in.

*   `-S` Enable SSL, because its connecting to 5986;
*   `-c` pfx_cert.crt - provide the public key certificate
*   `-k` pfx_key.key - provide the private key
*   `-i` IP - host to connect to

```powershell
evil-winrm -i 10.129.116.41 -k pfx_key.key -c pfx_cert.crt -S
```

![winrm1](/docs/assets/img/HTB-Timelapse/winrm1.png)

And we can get our user flag

![user_flag](/docs/assets/img/HTB-Timelapse/user_flag.png)

# Shell as svc_deploy

## Looking around

As always start with checking privileges and group memberships.

```powershell
whoami /priv
```

![priv](/docs/assets/img/HTB-Timelapse/priv.png)

```powershell
net user legacyy
```

![groups](/docs/assets/img/HTB-Timelapse/groups.png)

Nothing too interesting there, maybe the **Development** group could be useful later..

## Powershell History

One of the very important places to check is the `PS History` ([More here](https://www.sharepointdiary.com/2020/11/powershell-command-history.html))

![ps_history1](/docs/assets/img/HTB-Timelapse/ps_history1.png)

```powershell
type C:\Users\legacyy\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

The file contains few lines, including connecting to the host using the creds for another user, **svc_deploy**

So we have new set of credentials: 
    **svc_deploy**:**E3R$Q62^12p7PLlC%KWaxuaV**

![ps_history2](/docs/assets/img/HTB-Timelapse/ps_history2.png)

## Shell

With the credential we get in previous step we can use `evil-winrm` to log in as another user and continue enumeration.

![winrm_svc](/docs/assets/img/HTB-Timelapse/winrm_svc.png)

# Privilege Escalation

## Enumeration 

Logged in as **svc_deploy** we need to check privileges and group memberships all over again.

![winrm_svc2](/docs/assets/img/HTB-Timelapse/winrm_svc2.png)

And we find something really important! User is member of **LAPS_Readers** ([More here](https://book.hacktricks.xyz/windows-hardening/active-directory-methodology/laps)).

This means we can dump passwords for local administrator.

There are 2 ways (that I know from top of my head), either use `netexec` or `PowerView`

## Dump Admin password

### Method 1: netexec

If there is no access to a powershell you can abuse the LAPS_Readers group privileges remotely through LDAP by using

```powershell
netexec ldap 10.129.116.41 -u svc_deploy -p 'E3R$Q62^12p7PLlC%KWaxuaV' -M laps
```
This will dump all the passwords that the user can read, allowing you to get a better foothold with a different user.

![adm_pass](/docs/assets/img/HTB-Timelapse/adm_pass.png)

### Method 1: PowerView

We can upload `PowerView.ps1` 

![powerview1](/docs/assets/img/HTB-Timelapse/powerview1.png)

Import it to PS

![powerview2](/docs/assets/img/HTB-Timelapse/powerview2.png)

And use `Get-ADComputer` module to get admin password.

![powerview3](/docs/assets/img/HTB-Timelapse/powerview3.png)

## Log in as admin

With the admin credentials we can once again use `evil-winrm` to log in and grab the root flag.

![admin_login](/docs/assets/img/HTB-Timelapse/admin_login.png)

![root_flag](/docs/assets/img/HTB-Timelapse/root_flag.png)

~
