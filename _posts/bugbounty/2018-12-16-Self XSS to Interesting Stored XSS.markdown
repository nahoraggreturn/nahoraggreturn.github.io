---
layout: post
title:  "Self XSS to Interesting Stored XSS"
date:   2018-12-15
categories: BugBounty
---

**T**his is my first bug bounty write-up, so kindly go easy on me!

So i found this XSS in a program on Hackerone. The interesting thing about this Stored XSS is the **place** where it's reflected which i found by luck while searching a way to escalate from self XSS.

Also i can't disclose the program name as they have asked not to but if you find it somehow, i'll not be amazed.

So let's dive in to this site, we'll call it `redacted.com`

While sitting on `redacted.com` for hours and trying to get XSS on it, i thought it was impossible to get XSS on this site as it was properly encoding everything. And even if i find one, it's gonna be Self XSS. 

It wasn't a big site and after trying for XSS at every endpoint, i gave up and moved on to find other vulnerabilities.

So the next day, i read a report on H1 about [template injection](https://hackerone.com/reports/250837) in AngularJS and i was like hmmmmmm, i didn't knew about that. Turns out that `redacted.com` was also running AngularJS. 

So i tried a simple expression like \{\{4\*4\}\}, which if not encoded will reflect `16` and finally found a place where it wasn't encoding. Now i can give this payload **\{\{constructor.constructor('alert("XSS")')()\}\}** for XSS.

**YAY!!! i found XSS, after a minute realizes......DAMN, it's a Self!!!**

![Unsubscribe-link]({{site.baseurl}}/assets/bugbounty/self-stored-xss/xss-stored-meme-277x300.png){:class="img-responsive"}

What NOW????

So after hours of searchinng around, i found an **interesting place** where it was executing and doesn't need any authentication.

To give a background about this application, it has a feature to email the reports(of whatever this site does) and we can give a custom name to the report. These reports are sensitive and can only be viewed by authenticated user. The Self XSS i found was in the name of this report and since the reports can only be viewed by authenticated user, there's no way of getting it executed on other users. **Really???** 

So i used this feature and emailed a report to my mail and found a small unsubscribe link down in the corner hiding. 

![meme]({{site.baseurl}}/assets/bugbounty/self-stored-xss/writeup2_1.png){:class="img-responsive"}

Opened it and BOOM!!! It was reflecting the report name without any authentication.

![Unsubscribe]({{site.baseurl}}/assets/bugbounty/self-stored-xss/writeup2_2.png){:class="img-responsive"}

Time to test whether it's encoding the curly brackets \{\{\}\}

So i quickly went to my report page, gave the name as **\{\{constructor.constructor('alert("XSS")')()\}\}**	 and saved it. Opened that unsubscribe link again and BOOM!! It's **Stored XSS** baby.

![Alert]({{site.baseurl}}/assets/bugbounty/self-stored-xss/writeup2_3.png){:class="img-responsive"}

Now when anyone opens that unsubscribe link, XSS will be executed. This will work on anyone irrespective of whether the victim is authenticated or not.


Lesson Learned:
==

1) See what technologies are running on the application and find exploits specific to them.

2) Read old hackerone reports that are disclosed whenever you feel like bored.

3) **Try Harder On Everything In Application** - I read reports and write-ups alot but never came across one that got XSS in email unsubscription link. I could've just reported that self XSS and got ended up as Informative but i gave more time and got lucky.

---

  

Timeline:
==

**09/10/2018 -** *Submitted Report*

**10/10/2018 -** *Triaged*

**11/10/2018 -** *Rewarded*

**22/10/2018 -** *Resolved*

*Thanks for reading.*
