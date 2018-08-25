---
layout: post
title:  "HTB:Celestial Write-Up"
date:   2018-08-25
categories: HTB
---

If you somehow stumbled upon this blog, then you probably know what [HTB](https://hackthebox.eu) is (May god bless you if you **don't**).

`_Disclaimer: I’m a noob._`

Hey everyone, this is my first blog and i'll not mind if you start judging my english while reading cause i know it is very **good**....lol. 

Let's dive in!
===

For me, getting the **user** on this box was more fun and challenging than getting **root**. I didn't had much exposure in **deserialization** vulnerability before, like whenever someone talk about it, i'm like...**WTF** is "deserialization". But after doing this machine, I am much more confident in exploiting it. So let's start **pentesting**.

First, I like to go directly to the browser and see if any website is hosted on the machine because most of the time it is. But this time, it wasn't. So I started with **Nmap** scan.

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

Turns out i was partially right, http service was running but on port 3000 and the server was **Node.js** (I hate nodejs websites). Going to the site shows this 404 **text**.

![index]({{site.baseurl}}/assets/celestial/index.png){:class="img-responsive"}

Yup, Nothing here and source code is also clean. 

_"When there is nothing in front, there is something going on in the back"_. So let's intercept the request using **burp**.

On the first request, the server was **setting** the cookie in profile parameter (see below)

![cookei-set]({{site.baseurl}}/assets/celestial/cookie-set.png){:class="img-responsive"}

And on the subsequent requests, browser was sending this cookie in profile parameter and returning something interesting.

![cookie-send]({{site.baseurl}}/assets/celestial/cookie-send.png){:class="img-responsive"}

As you can see above, we didn't get that 404 with the cookie but a different response. So just to experiment, i removed the cookie and sent the request to server and i got that same 404 text. _"Hmmm, something is going on with the cookie"_. Taking a closer look at cookie, we can see that it's **base64** encoded.

![ini_b64]({{site.baseurl}}/assets/celestial/ini_b64.png){:class="img-responsive"}

After decoding, we can see above that the data is in JSON format. Looking at the decoded json data and the response together, I noticed that the server is first decoding the profile parameter of cookie, then parsing the json data and displaying it in a specific format. So i replaced `Dummy` with `admin` and `2` with `45` in json data, encoded and fired it. The server responded as expected (below).

![admin]({{site.baseurl}}/assets/celestial/admin.png){:class="img-responsive"}

So just for curiosity, I tried **simple xss** (`<script>alert(1)</script>`) and succeded but that's not what we wanted (wish there was a **bug bounty**). We wanted a reverse-shell. After banging my head for a while, i accidently sent a cookie with bad format (deleted one character by mistake) and the server dumped this error.

![dump]({{site.baseurl}}/assets/celestial/dump.png){:class="img-responsive"}

Damn, that's a lot of info to digest but if you try to understand it, you can see there is an error while **unserializing** the data. The word `unserialize` gave me a big hint.

Let's sum up whatever we've gathered till now. So basically the server is decoding the cookie, unserialzing, and simply displaying it. The only thing that can go wrong, is during `unserialization` of **user** data. So I started Googl-fu about nodejs serialization and found a module called [serialize](https://www.npmjs.com/package/node-serialize). To exploit it, i first have to understand it. I downloaded **nodejs**, installed the **serialize** module and started playing with it. First, i understood how serialization and unserialization happens and then i tried **exploiting** it.

![concept]({{site.baseurl}}/assets/celestial/concept.png){:class="img-responsive"}

So just to understand the basics of deserialization vulnerability I created an object, add a simple `exec()` (which **exec**utes system commands) method with "malicious" string inside and serialized it. Then I put that serialized string inside `unserialize()` method, executed it and the `exec()` method got executed(above screenshot). So I assumed this is what is happening on the server and without wasting any time, I created a nodejs reverse shell payload using this [tool](https://github.com/ajinabraham/Node.Js-Security-Course/blob/master/nodejsshell.py), which gave this output. 

![revshell]({{site.baseurl}}/assets/celestial/revshell.png){:class="img-responsive"}

Then I **serialized** the above output manually (if we don't serialize it, then the paylod will not get executed while deserialization), put it in "username" field of json data and encoded it.

![finalpayload]({{site.baseurl}}/assets/celestial/finalpayload.png){:class="img-responsive"}

Started `nc`, sent the encoded data in the profile parameter of cookie(see the final request below) and BAM!! We got a **reverse shell**.

![burp]({{site.baseurl}}/assets/celestial/burp.png){:class="img-responsive"}

![gotshell]({{site.baseurl}}/assets/celestial/gotshell.png){:class="img-responsive"}

So after getting a shell, I first **enumerate** the user directory and luckily i found something in it.

There were 2 interesting files, `output.txt`(owned by root) in user dir and `script.py` in Documents dir(see the contents below of both files).

![2files]({{site.baseurl}}/assets/celestial/2files.png){:class="img-responsive"}

As the output of `script.py` was shown inside `output.txt`, i thought there must be a **cron job** running this python script as **root** and putting the output in txt file. So I overwrite the script with my own payload, which reads the root.txt from root dir.

 Surprisingly, the script got executed as `root` and outputted the flag in `output.txt`

![pscrip]({{site.baseurl}}/assets/celestial/pscrip.png){:class="img-responsive"}

pwn3d!!