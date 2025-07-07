---
layout: post
title: "Exploiting an ORM Injection to Steal Cryptocurrency from an Online Shooter"
description: "blog writeup"
categories: writeup
image: /public/crypto-shooter-embed.png
---

Earlier this month, [Sam Curry](https://x.com/samwcyo) and I found one of our first exploitable ORM injection vulnerabilities that we've seen in the wild and leveraged it to steal cryptocurrency from an online game.

All of us being gamers working in crypto, one of the things we were always interested in were games that integrate crypto in one form or another. We found an upcoming "pay-to-spawn" shooter, a battle royale game where you have to put up some amount of crypto to spawn and the winner takes all.
Sadly for us, the game was not fully released and it's in closed beta with no way to get invites. 
We also could not find the actual game binary to poke around on the website anywhere, but our friend [Justin Rhinehart](https://x.com/sshell_) found it was uploaded to VirusTotal.

After acquiring a copy of the binary we were met with essentially a blank screen as the game servers were not currently running. So we had a closer look at the actual website powering the game. Account signups were enabled and after making an account and playing around with the website for a little bit, we made some progress. The age old match-and-replace "false" with "true" popped up another, previously hidden, menu section in the dashboard.

<figure style="text-align: center;">
  <img src="{{ '/public/no-admin.png' | relative_url }}" alt="No admin panel" class="imgCenter">
  <figcaption style="font-size: 0.8em; color: #666; margin-top: -25px;">Accessing the website normally</figcaption>
</figure>

<figure style="text-align: center;">
  <img src="{{ '/public/admin-mr.png' | relative_url }}" alt="admin panel option visible" class="imgCenter">
  <figcaption style="font-size: 0.8em; color: #666; margin-top: -25px;">Accessing the website with the client-side match and replace for "isAdmin" set to "true"</figcaption>
</figure>

Looks like the admin panel for the game is just a hidden menu on the main site. While the match-and-replace trick gave us visual access to the admin panel, no data was actually accessible and most relevant API calls threw a 401 unauthorized, as our user accounts obviously lacked admin permissions. While this gave us a good target to aim for, getting permissions for the admin panel, we didn't find any way to escalate our privileges on the main site.

<figure style="text-align: center;">
  <img src="{{ '/public/admin-panel.png' | relative_url }}" alt="admin panel" class="imgCenter">
  <figcaption style="font-size: 0.8em; color: #666; margin-top: -25px;">A look at the admin panel with no admin permissions</figcaption>
</figure>

Taking a step back and seeing if there were any other subdomains accessible, we noticed a "dev." domain that appeared to have the exact same content as the main site. Signups worked there as well, with one major difference: Django debug mode was enabled. This meant API calls that failed were throwing verbose errors which allowed us to quickly gather more information about the inner working of the site. 
While Django debug mode is usually a quick win, this time all the potential secrets and passwords were redacted by Django.
We tried gathering more information from the little source code snippets that are in the stack traces, also without any luck.
```
POST /api/race/queue HTTP/2
Host: dev.xxx.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:138.0) Gecko/20100101 Firefox/138.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: text/plain;charset=UTF-8
Content-Length: 120

{"queue":"default","action":"remove_users","query":{"start_filters":{"id":"123"},"filters":{},"order":[]}}
```
Thats when I noticed that the API request above threw an error told us that we were controlling the filter for an ORM query on the backend.
Remembering the great blogpost [plORMbing your Django ORM](https://www.elttam.com/blog/plormbing-your-django-orm/) by Alex Brown, we got to work trying to get our admin permissions. 

The error leaked a long list of models that we could try to juice for more information. Throwing `password` with the `__contains` filter at the request, to try and leak the user's password hash worked!
```
POST /api/race/queue HTTP/2
Host: dev.xxx.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:138.0) Gecko/20100101 Firefox/138.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: text/plain;charset=UTF-8
Content-Length: 120

{"queue":"default","action":"remove_users","query":{"start_filters":{"password__contains":"a"},"filters":{},"order":[]}}
```
Gave us the following response.
```
{"queue": "default", "deleted": 140}
```
This means that 140 users had an "a" in their password hash. Django by default uses PBKDF2, which are pretty slow to crack and cracking passwords is boring anyways.
Other obvious targets to look for in these situations are session cookies stored in the database, or other access tokens like JWTs or password reset tokens. 
Sadly, none of these were present. 
We did however have the models `is_superuser` and `is_staff` to figure out all the admin accounts. 
By combining multiple filters, we could now query for all `is_superuser` or `is_staff` accounts and figure out their email's by bruteforcing the `email` with a `__startswith` query.
Sam cooked up a quick script to automate the bruteforcing of the emails.

<figure style="text-align: center;">
  <img src="{{ '/public/script.png' | relative_url }}" alt="scripting the exploit" class="imgCenter">
  <figcaption style="font-size: 0.8em; color: #666; margin-top: -25px;">Exploit script trying to find an admin user</figcaption>
</figure>

Now we had the emails of the admins, but no way to actually log in. Looking back at the list of models, we noticed the `mail_log` model.
This had multiple fields, like `subject` and `message`. This sounded suspiciously close to emails that the system sends to users.
To quickly test that theory, I requested a password reset for my account and using the ORM injection queried the `mail_log.message` for my account for a string that was present in the email, like the actual link to reset my password.

<figure style="text-align: center;">
  <img src="{{ '/public/pw-reset.png' | relative_url }}" alt="scripting the exploit" class="imgCenter">
  <figcaption style="font-size: 0.8em; color: #666; margin-top: -25px;">Exploit script leaking a password reset link</figcaption>
</figure>

Jackpot! Now all we had to do was combine everything to get our admin access.
We requested a password reset for the admin email that we leaked previously, then we leaked the password reset link from the email stored in the `mail_log` and simply used the link to reset the password of the admin so we can log in.

<figure style="text-align: center;">
  <img src="{{ '/public/admin-access.png' | relative_url }}" alt="accessing the admin panel" class="imgCenter">
  <figcaption style="font-size: 0.8em; color: #666; margin-top: -25px;">Full access to the admin panel</figcaption>
</figure>

Now the last thing to check was if we had access to the crown jewels that is the payout wallet for the game. We tried sending a small amount of USDC that was in the wallet to us, which worked. We had full access to all admin functionality, including the ability to drain the wallets that are running the game!

<figure style="text-align: center;">
  <img src="{{ '/public/jackpot.png' | relative_url }}" alt="moving game wallet funds" class="imgCenter">
  <figcaption style="font-size: 0.8em; color: #666; margin-top: -25px;">Moving funds from the game's crypto wallet</figcaption>
</figure>

Everything was responsibly disclosed to the companies involved and fixes were deployed for the issues that we found.

Shoutouts for helping with the hacking and blogpost:
- Samuel Curry ([@samwcyo](https://x.com/samwcyo))
- Justin Rhinehart ([@sshell_](https://x.com/sshell_))