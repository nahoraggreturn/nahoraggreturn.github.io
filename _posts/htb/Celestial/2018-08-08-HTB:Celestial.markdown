---
layout: post
title:  "HTB:Celestial"
date:   2018-08-08
categories: HTB
---

`_Disclaimer: I’m a noob._`

Hey everyone, this is my first blog and my english is very **good**....lol. 

Let's dive in!
===

For me, getting user on this box was more fun and somewhat difficult than getting root. I didn't had much exposure in **deserialization** vulnerability but after doing this machine, i can say that i am more confident than before in exploiting it. Let's start **pentesting**.

First, i like to go directly to the browser and see if any website is hosted on the machine because most of the time it is. But this time, it wasn't. So i started with **Nmap** scan.

```
root@kali:~/Documents/htb/boxes/Celestial# nmap -sV -sC -oA nmap 10.10.10.85

Starting Nmap 7.25BETA1 ( https://nmap.org ) at 2018-08-08 09:58 IST
Nmap scan report for 10.10.10.85
Host is up (0.24s latency).
Not shown: 999 closed ports
PORT     STATE SERVICE VERSION
3000/tcp open  http    Node.js Express framework
|_http-title: Site doesn't have a title (text/html; charset=utf-8).

``` 

Turns out i was partially right, http service is running but on port 3000 and the server is **Node.js**. Going to the site shows this

![index]({{site.baseurl}}/assets/celestial/index.png){:class="img-responsive"}

Yup, Nothing here and source code is also clean. When there is nothing on front, then there is something going on in the back.So let's intercept the request using **burp**.

On the first request, the server is **setting** the cookie with profile parameter

![cookei-set]({{site.baseurl}}/assets/celestial/cookie-set.png){:class="img-responsive"}

and on the next request, browser is sending the cookie in profile parameter and returning something intresting.

![cookie-send]({{site.baseurl}}/assets/celestial/cookie-send.png){:class="img-responsive"}

This time we got a different response.So i again send request to server without cookie and we got that same 404 text. Hmmm, something is going on with the cookie. Taking closer look at cookie, we can see that it's **base64** encoded.

![ini_b64]({{site.baseurl}}/assets/celestial/ini_b64.png){:class="img-responsive"}

After decoding, we can see the data is in JSON format. Looking at the decoded json data and the resonse, i noticed that the server is first decoding the profile parameter of cookie, then parsing the json data and displaying  the data in a specific format. So i replaced `Dummy` with `admin` and `2` with `45` in json data, encode it and fired it. The server responded as expected.

![admin]({{site.baseurl}}/assets/celestial/admin.png){:class="img-responsive"}

Tried **simple xss** (`<script>alert(1)</script>`) and succeded but that's not what we want (wish there was a **bug bounty**). We want a reverse-shell. After banging my head for a while, i accidently send a cookie with bad format (deleted one character by mistake) and the server dumped this error.

![dump]({{site.baseurl}}/assets/celestial/dump.png){:class="img-responsive"}

Damn, that's a lot of info to digest but if you try to understand it, you can see there is an error while **unserializing** the data. The word `unserialize` gave me a big hint.

Let's sum it up till now, so basically the server is decoding the cookie, unserialzing, and displaying it. The only thing that can go wrong in this, is during `unserialization` of **user** data. So i started googl-fu about nodejs serialization and found a module called [serialize](https://www.npmjs.com/package/node-serialize). To try to exploit it, i first have to understand it. I downloaded **nodejs**, installed the **serialize** module and started playing with it. First, i understood how serialization and unserialization happens and then i tried **exploiting** it.

![concept]({{site.baseurl}}/assets/celestial/concept.png){:class="img-responsive"}

As you can see i created an object, add a simple `exec()` method inside and serialized it. Then i put that serialized string inside `unserialize()` method, executed it and the `exec()` method got executed. So i assumed this is what is happening on the server and without wasting any time, i created a nodejs reverse shell using this [tool](https://github.com/ajinabraham/Node.Js-Security-Course/blob/master/nodejsshell.py), which gave this output. 

![revshell]({{site.baseurl}}/assets/celestial/revshell.png){:class="img-responsive"}

Then i **serialized** the above output manually, put it in "username" field of json data and encoded it.

![finalpayload]({{site.baseurl}}/assets/celestial/finalpayload.png){:class="img-responsive"}

Started `nc`, sent the encoded data in the profile parameter of cookie and BAM!! We got a **reverse shell**.

![burp]({{site.baseurl}}/assets/celestial/burp.png){:class="img-responsive"}

![gotshell]({{site.baseurl}}/assets/celestial/gotshell.png){:class="img-responsive"}

So after getting a shell, i first **enumerate** the user directory and luckily i found something in it.

There were 2 files, `output.txt` in user dir that is owned by root and `script.py` in Documents dir.

![2files]({{site.baseurl}}/assets/celestial/2files.png){:class="img-responsive"}

As the output of `script.py` was showing inside `output.txt`, i thought there must be a **cron job** running this python script as **root**. So i overwrite the script with my own which reads the root.txt from root dir.

Suprisingly, the script got executed as `root` and outputed the flag in `output.txt`

![pscrip]({{site.baseurl}}/assets/celestial/pscrip.png){:class="img-responsive"}

pwn3d!!