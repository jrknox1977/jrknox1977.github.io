---
title: "Using autorecon"
date: 2022-07-26 10:00:00
catagories: [article,tools]
tags: [recon]
---
My favorite tool for inital recon of boxes is: [autorecon](https://github.com/Tib3rius/AutoRecon) by [Tib3rius](https://twitter.com/0xTib3rius) 

I create a new folder for the box and then a command that looks like:
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
