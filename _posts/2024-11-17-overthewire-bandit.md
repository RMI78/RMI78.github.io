---
title: Breaking Bandit from Over The Wire
date: 2024-11-22 00:00:00 +0000
categories: [CHALLENGES]
tags: [cybersecurity, Linux, beginner, SSH, writeup, git]
---
# Introduction
Back in the days when I started pentesting, I did the bandit track from Over The Wire and while I find that Over The Wire's website I not enough recommended to beginners, I really wanted to feel some nostalgia by doing the track again with all the knowledge I have acquired over the years. Moreover, there is already tons of tutorials for them so I just feel like mine is missing here.

For those who don't know, the bandit challenges are a series of more than 30 levels to test yourself if you are good enough with a command line. It starts with some basic stuff such as hidden files and ends with complex git layers in which you have to seek the solution. The password for level 0 is given and each following level is hidding the SSH password of the following one.

### Disclaimer
In this walkthrough, I will disclose most of the password, so I highly encourage you to try the challenges yourself before starting to read this. By the way, this also emphasis the notes you should take when pentesting stuff around. I got a lot of complains about people staying stuck at some level, calling it a day and completly forgetting the passwords when they come back which force them to start over. Hopefully you can eventually reach here for the different passwords.

# Levels

## Level 0-1

Level 0 is basically just a text telling you to initiate an SSH connection to a server with the given informations (including password). 

```
ro@Parrot:~/$ ssh -p 2220 bandit0@bandit.labs.overthewire.org
```

And then typing the given password "bandit0" (same as login)

Once you are in, as specified in the website, you have to print a file called readme on the screen:

```
bandit0@bandit:~$ cat readme 
Congratulations on your first steps into the bandit game!!
Please make sure you have read the rules at https://overthewire.org/rules/
If you are following a course, workshop, walkthrough or other educational activity,
please inform the instructor about the rules as well and encourage them to
contribute to the OverTheWire community so we can keep these games free!

The password you are looking for is: ZjLjTmM6FvvyRnrb2rfNWOZOTa6ip5If
```

## Level 2

So the concept is the same, we SSH into the same machine, same port, but different user, `bandit1` give access to the password of `bandit2`, which itself gives access to the password of `bandit3` and so on...

>The password for the next level is stored in a file called - located in the home directory

Here the password is stored in a file called "-". The catch is "-" is also a character used to give commands to programs, so doing 

```
bandit1@bandit:~$ cat -
```
Won't work as the command is considered as unterminated. You have to explicitly specify to `cat` that "-" is a file by doing so 

```
bandit1@bandit:~$ cat ./-
263JGJPfgU6LtdEvgfWU1XP5yac29mFx
```

Here, we prefix the filename by `./` with `.` being a reference to the current folder in which we are located. Thus, expliciting the existence of "-" as a file.

## Level 3

>The password for the next level is stored in a file called spaces in this filename located in the home directory

Here it is mentionned that the password is in a file called spaces. Using quotes here allow to explicitly telling `cat` that we want a file with spaces as name 

```
bandit2@bandit:~$ cat "spaces in this filename" 
MNk8KNH3Usiio41PRUEoDFPqfxLPlSmx
```

## Level 4

>The password for the next level is stored in a hidden file in the inhere directory.

 In Linux, every file starting with a dot is a hidden file not showing up with a simple `ls`. That is why when pentesting or investigating it is always useful to setup an alias: `alias ls="ls -a"` where the `-a` option means "all". (using the `-a` option is the bare minimum, you can also use `-l` which gives more informations)

```
bandit3@bandit:~$ ls 
inhere
bandit3@bandit:~$ ls inhere/
bandit3@bandit:~$ ls -a inhere/
.  ..  ...Hiding-From-You
bandit3@bandit:~$ cat inhere/...Hiding-From-You 
2WmrDFRmJIq3IPxneAaMGhap0pFhF3NJ
```

## Level 5

> he password for the next level is stored in the only human-readable file in the inhere directory. 

In Linux, you have the tool called `file` which will try to identify the type of file you give it as argument regardless of its extention name (relying on the file header). If you combine that with a wildcard, `file` will list you all files with their file type. 

```
bandit4@bandit:~$ ls 
inhere
bandit4@bandit:~$ ls inhere/
-file00  -file01  -file02  -file03  -file04  -file05  -file06  -file07  -file08  -file09
bandit4@bandit:~$ file inhere/*
inhere/-file00: data
inhere/-file01: data
inhere/-file02: data
inhere/-file03: data
inhere/-file04: data
inhere/-file05: data
inhere/-file06: data
inhere/-file07: ASCII text
inhere/-file08: data
inhere/-file09: data
bandit4@bandit:~$ cat inhere/-file07 
4oQYVPkxZOOEOO5pTW81FB8j8lxXGUQw
```

Here, data means generally giberrish, and "ASCII text" is human readable. `file` is not included on every operating system and may become unreliable when trying to identify broken files, and niche file formats. 

## Level 6

> The password for the next level is stored in a file somewhere under the inhere directory and has all of the following properties:
>- human-readable
>- 1033 bytes in size
>- not executable

This is the moment to show off your `find` command skills. For this level you don't have to respect all the criteria. My approach was to use `find` then reuse `file` through piping to identify the "human-readable" part but it turned out that a simple find based on the 2 other criteria would do the job.

```
bandit5@bandit:~$ find inhere/ -type f -size 1033c ! -executable 
inhere/maybehere07/.file2
bandit5@bandit:~$ find inhere/ -type f -size 1033c ! -executable | xargs cat 
HWasnPhtq9AVKe0dmk45nxy20cvUa6EG
```

I am sure a `find` option is a single google search away from me for the first criteria but why bother. the `c` in the size of find corresponding to bytes is not very intuitive but `b` is already taken by `bit` 

Here I used a pipe `|` to redirect the output of my command to `xargs` which will iter through the lines (here a single line) of my output by repeating the command given as argument.

## Level 7

Again find here but with different criterias. 
>The password for the next level is stored somewhere on the server and has all of the following properties:
>- owned by user bandit7
>- owned by group bandit6
>- 33 bytes in size

```
bandit6@bandit:~$ find / -size 33c -user bandit7 -group bandit6 2> /dev/null | xargs cat 
morbNTDkSW6jIlUc0ymOdMaLnOlFVAaj
```

Here, the file was somewhere on the server so I had to mention "/" as the root location of the filesystem. the options are pretty straightforward. I also reused the xargs trick but I know that as "bandit6" I do ont have access to all the server and `find` can become really noisy regarding stuff you do not have access to. That's why I used `2> /dev/null` to redirect the errors in the output to a file called `/dev/null` which is the blackhole of flux in Linux as everything sent to it will be forgotten.

## Level 8

> The password for the next level is stored in the file data.txt next to the word millionth.

Here we have an easy `grep` command to filter out the password from the file without going through it manually. 

```
bandit7@bandit:~$ cat data.txt | grep millionth 
millionth	dfwvzFQi4mU0wfNbFOe9RoWskMLg7eEc
```

## Level 9

> The password for the next level is stored in the file data.txt and is the only line of text that occurs only once.

So in this situation you can use the command `uniq` but it only detect same __adjacent__ lines which is not garantee here but we can also make the use of `sort` which can lexically sort strings for us. Then give the output to `uniq`. Technicaly, once sorted, all the same lines will be adjacent.

```
bandit8@bandit:~$ cat data.txt | sort |  uniq -u 
4CKMh1JI91bUIZZPXDqGanal4xvAg0JM
```

## Level 10

> The password for the next level is stored in the file data.txt in one of the few human-readable strings, preceded by several ‘=’ characters.

Here we go again with `grep` and using regular expression, in addition to that, it is mentionned that some part of the file are read-only. So it is a 2 step process. We can use `strings` which is a command extracting all human readable characters from a file, and then redirecting the output to `grep` with a regular expression. 


```
strings data.txt | grep -E "=+"
}========== the
p\l=
;c<Q=.dEXU!
3JprD========== passwordi
qC(=	
~fDV3========== is
7=oc
zP=	
~de=
3k=fQ
~o=0
69}=
%"=Y
=tZ~07
D9========== FGUW5ilLVJrxX9kMYMmlN4MgbpfMiqey
N=~[!N
zA=?0j
```

We are only looking for one to multiple "=" signs but I am sure there are better Regex out there if you want to fine tune this.

## Level 11

> The password for the next level is stored in the file data.txt, which contains base64 encoded data.

On this level we can use the `base64` command to decode the file:

```
bandit10@bandit:~$ base64 -d data.txt 
The password is dtR173fZKb0RRsDFSGsg2RWnpNVj3qRr
```

## Level 12

> The password for the next level is stored in the file data.txt, where all lowercase (a-z) and uppercase (A-Z) letters have been rotated by 13 positions

So here you can use some online tools such as [cyberchef](https://cyberchef.io/) but there is a way to do it only from the terminal using the `tr` command. This command is used to translate or delete characters from a string, you just need to input the alphabet in the right order and tell `tr` to start from "n" which is the 13th letter of the alphabet, all the way to M by cycling through "ZA" when the offset reaches the end. And doing this fro lowercases and uppercases letter.

```
bandit11@bandit:~$ cat data.txt | tr 'A-Za-z' 'N-ZA-Mn-za-m'
The password is 7x16WNeHIi5YkIhWsfFIqoognUTyj9Q4
```
## Level 13

> The password for the next level is stored in the file data.txt, which is a hexdump of a file that has been repeatedly compressed. For this level it may be useful to create a directory under /tmp in which you can work. Use mkdir with a hard to guess directory name. Or better, use the command “mktemp -d”. Then copy the datafile using cp, and rename it using mv.

So it gets complicated here as there are more steps involved. 

```
bandit12@bandit:~$ DIR=$(mktemp -d) && cd $DIR
bandit12@bandit:/tmp/tmp.pqUBpFkety$ xxd -r $HOME/data.txt > binfile 
bandit12@bandit:/tmp/tmp.pqUBpFkety$ file binfile 
binfile: gzip compressed data, was "data2.bin", last modified: Thu Sep 19 07:08:15 2024, max compression, from Unix, original size modulo 2^32 574
bandit12@bandit:/tmp/tmp.pqUBpFkety$ mv binfile data2.gz
bandit12@bandit:/tmp/tmp.pqUBpFkety$ gzip -d data2.gz
bandit12@bandit:/tmp/tmp.pqUBpFkety$ file data2 
data2: bzip2 compressed data, block size = 900k
bandit12@bandit:/tmp/tmp.pqUBpFkety$ bzip2 -d data2
bzip2: Can't guess original name for data2 -- using data2.out
bandit12@bandit:/tmp/tmp.pqUBpFkety$ file data2.out 
data2.out: gzip compressed data, was "data4.bin", last modified: Thu Sep 19 07:08:15 2024, max compression, from Unix, original size modulo 2^32 20480
bandit12@bandit:/tmp/tmp.pqUBpFkety$ mv data2.out data2.gz 
bandit12@bandit:/tmp/tmp.pqUBpFkety$ gzip -d data2.gz 
bandit12@bandit:/tmp/tmp.pqUBpFkety$ file data2 
data2: POSIX tar archive (GNU)
bandit12@bandit:/tmp/tmp.pqUBpFkety$ mv data2 data2.tar 
bandit12@bandit:/tmp/tmp.pqUBpFkety$ tar xvf data2.tar 
data5.bin
bandit12@bandit:/tmp/tmp.pqUBpFkety$ file data5.bin 
data5.bin: POSIX tar archive (GNU)
bandit12@bandit:/tmp/tmp.pqUBpFkety$ mv data5.bin data5.tar 
bandit12@bandit:/tmp/tmp.pqUBpFkety$ tar xvf data
data2.tar  data5.tar  
bandit12@bandit:/tmp/tmp.pqUBpFkety$ tar xvf data5.tar 
data6.bin
bandit12@bandit:/tmp/tmp.pqUBpFkety$ file data6.bin 
data6.bin: bzip2 compressed data, block size = 900k
bandit12@bandit:/tmp/tmp.pqUBpFkety$ bzip2 -d data6.bin 
bzip2: Can't guess original name for data6.bin -- using data6.bin.out
bandit12@bandit:/tmp/tmp.pqUBpFkety$ file data6.bin.out 
data6.bin.out: POSIX tar archive (GNU)
bandit12@bandit:/tmp/tmp.pqUBpFkety$ mv data6.bin.out data6.tar 
bandit12@bandit:/tmp/tmp.pqUBpFkety$ tar xvf data6.tar 
data8.bin
bandit12@bandit:/tmp/tmp.pqUBpFkety$ file data8.bin 
data8.bin: gzip compressed data, was "data9.bin", last modified: Thu Sep 19 07:08:15 2024, max compression, from Unix, original size modulo 2^32 49
bandit12@bandit:/tmp/tmp.pqUBpFkety$ mv data8.bin data8.gz 
bandit12@bandit:/tmp/tmp.pqUBpFkety$ gzip -d data8.gz
bandit12@bandit:/tmp/tmp.pqUBpFkety$ file data8
data8: ASCII text
bandit12@bandit:/tmp/tmp.pqUBpFkety$ cat data8 
The password is FO5dwFsc0cbaIiH0h8J2eUks2vdTDwAn
```

This is the kind of challenges you can often find in capture the flags as it is easy to make, pretty dumb and time-consuming. It is a 8-layered compression archive with popular tools (sometimes you get unlucky and end up using archive tools made and used by a single guy in the world and it is a pain to make it work, lucky us, we only get easy ones identifiable by `file` everytime). Plus an additionnal hexdump on top of it. To get back to a regular file from an hexdump you can use the command `xxd -r mydump` which will reverse the hexdump to a regular binary file. Here it gives us an archive as expected.

So the archive tools here are roughly the same:
- `tar`
- `bzip2`
- `gzip`

You have to know at least those compression utilities and how to compress/decompress them. After that the process is simple and can be quite automated:
1) You identify the file using `file`
2) You rename the file using `mv` (used to move files but if you move a file to a name that doesn't exist, it is renaming) as some tools above refuse to work if the file doesn't have the right extension. This does not apply to all the tools (and I am not doing it everytime) but if you want to be safe, always use it.
3) Decompress the file using the various command lines (Google is your friend or manpages for the purists)
4) repeat the process until you identify an ASCII text file. 

## Level 14

> The password for the next level is stored in /etc/bandit_pass/bandit14 and can only be read by user bandit14. For this level, you don’t get the next password, but you get a private SSH key that can be used to log into the next level. 

So here we are already logged in as "bandit13" but what is expected is to get access to bandit14's password with a specific SSH key. SSH can use different authentication method. One is the login/password that we are using. Another one is through asymetrical keys.

To do that we can reuse SSH inside our current session by tweaking the options and mentionning the SSH key. 


From here you have multiple solutions. Either you login as bandit14 and look for level 15 without getting the password for bandit14:
```
ssh -p 2220 -i sshkey.private bandit14@localhost 
```
Either you execute the command directly after `ssh` to print the password, automatically get logged out and relog with login anf password:

```
ssh -p 2220 -i sshkey.private bandit14@localhost cat /etc/bandit_pass/bandit14
# blablabla, message of the day, blablabla
MU4VWeTyJk8ROof1qqmcBPaLh7lDCPvS
```

## Level 15

> The password for the next level can be retrieved by submitting the password of the current level to port 30000 on localhost.

This level is about basic networking. (This is also something I am still doing nowadays in CTF. If you don't know what a random open port is doing, just go talk to it !) In this case we know we have to submit the current password so to do that you can use `netcat` to directly connect to the port.

```
bandit14@bandit:~$ netcat localhost 30000
MU4VWeTyJk8ROof1qqmcBPaLh7lDCPvS
Correct!
8xCjnmgoKbGLhHFAZlGE5Tmu4M2tKJQo
```

## Level 16

> The password for the next level can be retrieved by submitting the password of the current level to port 30001 on localhost using SSL/TLS encryption.

This is practically the same challenge as before except you have to know about SSL/TLS and be familiar with it as this is the standard encryption for web communications. No need to open a browser here, you can just use `openssl` to establish a connection and get the password.

```
bandit15@bandit:~$ openssl s_client -connect localhost:30001
CONNECTED(00000003)
Can't use SSL_get_servername
depth=0 CN = SnakeOil
verify error:num=18:self-signed certificate
verify return:1
depth=0 CN = SnakeOil
verify return:1
---
Certificate chain
 0 s:CN = SnakeOil
   i:CN = SnakeOil
   a:PKEY: rsaEncryption, 4096 (bit); sigalg: RSA-SHA256
   v:NotBefore: Jun 10 03:59:50 2024 GMT; NotAfter: Jun  8 03:59:50 2034 GMT
---
Server certificate
-----BEGIN CERTIFICATE-----
MIIFBzCCAu+gAwIBAgIUBLz7DBxA0IfojaL/WaJzE6Sbz7cwDQYJKoZIhvcNAQEL
BQAwEzERMA8GA1UEAwwIU25ha2VPaWwwHhcNMjQwNjEwMDM1OTUwWhcNMzQwNjA4
MDM1OTUwWjATMREwDwYDVQQDDAhTbmFrZU9pbDCCAiIwDQYJKoZIhvcNAQEBBQAD
ggIPADCCAgoCggIBANI+P5QXm9Bj21FIPsQqbqZRb5XmSZZJYaam7EIJ16Fxedf+
jXAv4d/FVqiEM4BuSNsNMeBMx2Gq0lAfN33h+RMTjRoMb8yBsZsC063MLfXCk4p+
09gtGP7BS6Iy5XdmfY/fPHvA3JDEScdlDDmd6Lsbdwhv93Q8M6POVO9sv4HuS4t/
jEjr+NhE+Bjr/wDbyg7GL71BP1WPZpQnRE4OzoSrt5+bZVLvODWUFwinB0fLaGRk
GmI0r5EUOUd7HpYyoIQbiNlePGfPpHRKnmdXTTEZEoxeWWAaM1VhPGqfrB/Pnca+
vAJX7iBOb3kHinmfVOScsG/YAUR94wSELeY+UlEWJaELVUntrJ5HeRDiTChiVQ++
wnnjNbepaW6shopybUF3XXfhIb4NvwLWpvoKFXVtcVjlOujF0snVvpE+MRT0wacy
tHtjZs7Ao7GYxDz6H8AdBLKJW67uQon37a4MI260ADFMS+2vEAbNSFP+f6ii5mrB
18cY64ZaF6oU8bjGK7BArDx56bRc3WFyuBIGWAFHEuB948BcshXY7baf5jjzPmgz
mq1zdRthQB31MOM2ii6vuTkheAvKfFf+llH4M9SnES4NSF2hj9NnHga9V08wfhYc
x0W6qu+S8HUdVF+V23yTvUNgz4Q+UoGs4sHSDEsIBFqNvInnpUmtNgcR2L5PAgMB
AAGjUzBRMB0GA1UdDgQWBBTPo8kfze4P9EgxNuyk7+xDGFtAYzAfBgNVHSMEGDAW
gBTPo8kfze4P9EgxNuyk7+xDGFtAYzAPBgNVHRMBAf8EBTADAQH/MA0GCSqGSIb3
DQEBCwUAA4ICAQAKHomtmcGqyiLnhziLe97Mq2+Sul5QgYVwfx/KYOXxv2T8ZmcR
Ae9XFhZT4jsAOUDK1OXx9aZgDGJHJLNEVTe9zWv1ONFfNxEBxQgP7hhmDBWdtj6d
taqEW/Jp06X+08BtnYK9NZsvDg2YRcvOHConeMjwvEL7tQK0m+GVyQfLYg6jnrhx
egH+abucTKxabFcWSE+Vk0uJYMqcbXvB4WNKz9vj4V5Hn7/DN4xIjFko+nREw6Oa
/AUFjNnO/FPjap+d68H1LdzMH3PSs+yjGid+6Zx9FCnt9qZydW13Miqg3nDnODXw
+Z682mQFjVlGPCA5ZOQbyMKY4tNazG2n8qy2famQT3+jF8Lb6a4NGbnpeWnLMkIu
jWLWIkA9MlbdNXuajiPNVyYIK9gdoBzbfaKwoOfSsLxEqlf8rio1GGcEV5Hlz5S2
txwI0xdW9MWeGWoiLbZSbRJH4TIBFFtoBG0LoEJi0C+UPwS8CDngJB4TyrZqEld3
rH87W+Et1t/Nepoc/Eoaux9PFp5VPXP+qwQGmhir/hv7OsgBhrkYuhkjxZ8+1uk7
tUWC/XM0mpLoxsq6vVl3AJaJe1ivdA9xLytsuG4iv02Juc593HXYR8yOpow0Eq2T
U5EyeuFg5RXYwAPi7ykw1PW7zAPL4MlonEVz+QXOSx6eyhimp1VZC11SCg==
-----END CERTIFICATE-----
subject=CN = SnakeOil
issuer=CN = SnakeOil
---
No client certificate CA names sent
Peer signing digest: SHA256
Peer signature type: RSA-PSS
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 2103 bytes and written 373 bytes
Verification error: self-signed certificate
---
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
Server public key is 4096 bit
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
Early data was not sent
Verify return code: 18 (self-signed certificate)
---
---
Post-Handshake New Session Ticket arrived:
SSL-Session:
    Protocol  : TLSv1.3
    Cipher    : TLS_AES_256_GCM_SHA384
    Session-ID: F77834A48B344389601292314C1C46273F13F699687827358D6BE7045DB0E570
    Session-ID-ctx: 
    Resumption PSK: 8554EC709C88E04C1942C7C8AE54AD19A3B9612FA325CCC99F23D0FB6064F581CD63F37D7EE74D2B0ACD2122EDCC484B
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    TLS session ticket lifetime hint: 300 (seconds)
    TLS session ticket:
    0000 - ad 67 51 83 e2 e6 b7 93-8b 25 99 84 27 67 26 9e   .gQ......%..'g&.
    0010 - 40 da 41 ea 8d c5 19 c0-ba 98 53 d2 7d f2 f4 44   @.A.......S.}..D
    0020 - 3d d7 7c e3 05 c9 3d a5-f8 cb cf 5c 30 0e 53 a9   =.|...=....\0.S.
    0030 - 90 4c 8c c9 28 3b f7 b6-50 94 a2 56 4c 82 79 55   .L..(;..P..VL.yU
    0040 - d4 e5 1f 14 fb a7 29 ae-41 f7 b0 58 af 12 50 94   ......).A..X..P.
    0050 - 45 f7 60 b9 77 d0 8c a6-a2 da 51 a8 a6 59 79 c9   E.`.w.....Q..Yy.
    0060 - 45 e1 43 c2 21 bf b5 b3-dc d9 b5 81 37 24 bc 6d   E.C.!.......7$.m
    0070 - 87 6d b5 37 a3 a6 9b fd-fc 3d 12 4a 42 e9 f0 77   .m.7.....=.JB..w
    0080 - 7d bd 66 6b 02 b0 65 d1-c1 3f dc 08 37 2a 72 61   }.fk..e..?..7*ra
    0090 - fa 62 ce 3e 6d 90 59 d9-b2 97 50 7e 2a 13 e4 da   .b.>m.Y...P~*...
    00a0 - bf c4 1e b7 3a 60 98 8b-69 d1 63 e9 45 8d e3 45   ....:`..i.c.E..E
    00b0 - 22 56 c6 06 50 70 fc 48-8a 57 8e ae 9a 19 f7 06   "V..Pp.H.W......
    00c0 - f8 19 89 c1 80 a9 a3 50-3a 8a 0d f5 11 61 bc f4   .......P:....a..
    00d0 - fb 3a c0 13 b8 b8 fe ed-0e 4d 11 85 d9 a7 85 9e   .:.......M......

    Start Time: 1731869955
    Timeout   : 7200 (sec)
    Verify return code: 18 (self-signed certificate)
    Extended master secret: no
    Max Early Data: 0
---
read R BLOCK
---
Post-Handshake New Session Ticket arrived:
SSL-Session:
    Protocol  : TLSv1.3
    Cipher    : TLS_AES_256_GCM_SHA384
    Session-ID: 3B73FA3C267EC16BC6D7DA7533FAC64E0C2285A3F2B572D2E92D983D15FC9245
    Session-ID-ctx: 
    Resumption PSK: 2EA3B0FDA94E24574C55589446A1DC4C7F4F904E4D11BD8A5FF1D27D818CC04EBFDC944392ED48711D6DB5CDCA2BAD4D
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    TLS session ticket lifetime hint: 300 (seconds)
    TLS session ticket:
    0000 - ad 67 51 83 e2 e6 b7 93-8b 25 99 84 27 67 26 9e   .gQ......%..'g&.
    0010 - 25 4a fa b6 6e 54 36 3e-ce 25 8c 6e 62 86 84 50   %J..nT6>.%.nb..P
    0020 - ae 57 94 62 75 59 30 85-7a 9d fc af bd 39 d7 57   .W.buY0.z....9.W
    0030 - d5 68 4a 83 81 01 26 53-5e fe 96 69 57 70 41 df   .hJ...&S^..iWpA.
    0040 - 57 76 b7 f8 af dc 07 72-c7 5e eb 78 07 37 03 a4   Wv.....r.^.x.7..
    0050 - 5b b5 15 8d 8e b5 87 be-d0 1c df 79 aa e5 93 ed   [..........y....
    0060 - 84 30 52 37 c2 30 34 00-a1 b9 fd 07 79 12 2c 9c   .0R7.04.....y.,.
    0070 - 72 5f c4 33 7b 84 0a 48-3d 6d da ce af dc e7 a1   r_.3{..H=m......
    0080 - 9c 07 08 e1 d5 1a b4 f6-77 a3 a3 55 96 c1 80 6c   ........w..U...l
    0090 - d3 7d 43 d0 22 71 05 62-2a 68 24 5e 5b ee 73 c2   .}C."q.b*h$^[.s.
    00a0 - 6c 1b 04 79 ec 9e 7f 50-46 4d 87 d0 46 98 f0 86   l..y...PFM..F...
    00b0 - ca 3a 8e 33 74 6f 66 5c-3d a1 70 ff ee 19 06 b8   .:.3tof\=.p.....
    00c0 - c9 7d d7 bc e8 65 c2 1d-94 0b 20 f6 44 28 9d ee   .}...e.... .D(..
    00d0 - c1 9a 2d 7e bd 02 e1 55-1d a3 27 b8 c1 94 64 67   ..-~...U..'...dg

    Start Time: 1731869955
    Timeout   : 7200 (sec)
    Verify return code: 18 (self-signed certificate)
    Extended master secret: no
    Max Early Data: 0
---
read R BLOCK
```
And the prompt will just wait for you to input the password of the current level. 
```
8xCjnmgoKbGLhHFAZlGE5Tmu4M2tKJQo
Correct!
kSkvUpMQ7lBYyCM4GBPvCvT1BfWRy0Dx
``` 

## Level 17

> The credentials for the next level can be retrieved by submitting the password of the current level to a port on localhost in the range 31000 to 32000. First find out which of these ports have a server listening on them. Then find out which of those speak SSL/TLS and which don’t. There is only 1 server that will give the next credentials, the others will simply send back to you whatever you send to it.

So the good thing about this machine is that it comes with `nmap` preinstalled on it. `nmap` is a network tool used to scan servers's open ports, protocols, and much more. Using `nmap`, we can find the port and dumb down the problem to the same difficulty of level 16. 

```
bandit16@bandit:~$ nmap -p 31000-32000 localhost 
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-17 19:05 UTC
Nmap scan report for localhost (127.0.0.1)
Host is up (0.00014s latency).
Not shown: 996 closed tcp ports (conn-refused)
PORT      STATE SERVICE
31046/tcp open  unknown
31518/tcp open  unknown
31691/tcp open  unknown
31790/tcp open  unknown
31960/tcp open  unknown
```

So we went from 1000 ports to 5 port. But we can reduce that even more. `nmap` has a builtin script allowing us to see which SSL/TLS encryptions are supported. We can use it to reduce the number of ports we have to try.

```
bandit16@bandit:~$ nmap --script ssl-enum-ciphers -p 31000-32000 localhost 
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-17 19:09 UTC
Nmap scan report for localhost (127.0.0.1)
Host is up (0.00013s latency).
Not shown: 996 closed tcp ports (conn-refused)
PORT      STATE SERVICE
31046/tcp open  unknown
31518/tcp open  unknown
| ssl-enum-ciphers: 
|   TLSv1.2: 
|     ciphers: 
|       TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA384 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_ARIA_128_GCM_SHA256 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_ARIA_256_GCM_SHA384 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_CAMELLIA_128_CBC_SHA256 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_CAMELLIA_256_CBC_SHA384 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256 (secp256r1) - A
|       TLS_RSA_WITH_AES_128_CBC_SHA (rsa 4096) - A
|       TLS_RSA_WITH_AES_128_CBC_SHA256 (rsa 4096) - A
|       TLS_RSA_WITH_AES_128_CCM (rsa 4096) - A
|       TLS_RSA_WITH_AES_128_CCM_8 (rsa 4096) - A
|       TLS_RSA_WITH_AES_128_GCM_SHA256 (rsa 4096) - A
|       TLS_RSA_WITH_AES_256_CBC_SHA (rsa 4096) - A
|       TLS_RSA_WITH_AES_256_CBC_SHA256 (rsa 4096) - A
|       TLS_RSA_WITH_AES_256_CCM (rsa 4096) - A
|       TLS_RSA_WITH_AES_256_CCM_8 (rsa 4096) - A
|       TLS_RSA_WITH_AES_256_GCM_SHA384 (rsa 4096) - A
|       TLS_RSA_WITH_ARIA_128_GCM_SHA256 (rsa 4096) - A
|       TLS_RSA_WITH_ARIA_256_GCM_SHA384 (rsa 4096) - A
|       TLS_RSA_WITH_CAMELLIA_128_CBC_SHA (rsa 4096) - A
|       TLS_RSA_WITH_CAMELLIA_128_CBC_SHA256 (rsa 4096) - A
|       TLS_RSA_WITH_CAMELLIA_256_CBC_SHA (rsa 4096) - A
|       TLS_RSA_WITH_CAMELLIA_256_CBC_SHA256 (rsa 4096) - A
|     compressors: 
|       NULL
|     cipher preference: client
|     warnings: 
|       Key exchange (secp256r1) of lower strength than certificate key
|   TLSv1.3: 
|     ciphers: 
|       TLS_AKE_WITH_AES_128_GCM_SHA256 (ecdh_x25519) - A
|       TLS_AKE_WITH_AES_256_GCM_SHA384 (ecdh_x25519) - A
|       TLS_AKE_WITH_CHACHA20_POLY1305_SHA256 (ecdh_x25519) - A
|     cipher preference: client
|_  least strength: A
31691/tcp open  unknown
31790/tcp open  unknown
| ssl-enum-ciphers: 
|   TLSv1.2: 
|     ciphers: 
|       TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA384 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_ARIA_128_GCM_SHA256 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_ARIA_256_GCM_SHA384 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_CAMELLIA_128_CBC_SHA256 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_CAMELLIA_256_CBC_SHA384 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256 (secp256r1) - A
|       TLS_RSA_WITH_AES_128_CBC_SHA (rsa 4096) - A
|       TLS_RSA_WITH_AES_128_CBC_SHA256 (rsa 4096) - A
|       TLS_RSA_WITH_AES_128_CCM (rsa 4096) - A
|       TLS_RSA_WITH_AES_128_CCM_8 (rsa 4096) - A
|       TLS_RSA_WITH_AES_128_GCM_SHA256 (rsa 4096) - A
|       TLS_RSA_WITH_AES_256_CBC_SHA (rsa 4096) - A
|       TLS_RSA_WITH_AES_256_CBC_SHA256 (rsa 4096) - A
|       TLS_RSA_WITH_AES_256_CCM (rsa 4096) - A
|       TLS_RSA_WITH_AES_256_CCM_8 (rsa 4096) - A
|       TLS_RSA_WITH_AES_256_GCM_SHA384 (rsa 4096) - A
|       TLS_RSA_WITH_ARIA_128_GCM_SHA256 (rsa 4096) - A
|       TLS_RSA_WITH_ARIA_256_GCM_SHA384 (rsa 4096) - A
|       TLS_RSA_WITH_CAMELLIA_128_CBC_SHA (rsa 4096) - A
|       TLS_RSA_WITH_CAMELLIA_128_CBC_SHA256 (rsa 4096) - A
|       TLS_RSA_WITH_CAMELLIA_256_CBC_SHA (rsa 4096) - A
|       TLS_RSA_WITH_CAMELLIA_256_CBC_SHA256 (rsa 4096) - A
|     compressors: 
|       NULL
|     cipher preference: client
|     warnings: 
|       Key exchange (secp256r1) of lower strength than certificate key
|   TLSv1.3: 
|     ciphers: 
|       TLS_AKE_WITH_AES_128_GCM_SHA256 (ecdh_x25519) - A
|       TLS_AKE_WITH_AES_256_GCM_SHA384 (ecdh_x25519) - A
|       TLS_AKE_WITH_CHACHA20_POLY1305_SHA256 (ecdh_x25519) - A
|     cipher preference: client
|_  least strength: A
31960/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 0.55 seconds
```

So our potentials ports to test are 31790 and 31518

```
bandit16@bandit:~$ openssl s_client -nocommands -connect bandit.labs.overthewire.org:31790
CONNECTED(00000003)
depth=0 CN = SnakeOil
verify error:num=18:self-signed certificate
verify return:1
depth=0 CN = SnakeOil
verify return:1
---
Certificate chain
 0 s:CN = SnakeOil
   i:CN = SnakeOil
   a:PKEY: rsaEncryption, 4096 (bit); sigalg: RSA-SHA256
   v:NotBefore: Jun 10 03:59:50 2024 GMT; NotAfter: Jun  8 03:59:50 2034 GMT
---
Server certificate
-----BEGIN CERTIFICATE-----
MIIFBzCCAu+gAwIBAgIUBLz7DBxA0IfojaL/WaJzE6Sbz7cwDQYJKoZIhvcNAQEL
BQAwEzERMA8GA1UEAwwIU25ha2VPaWwwHhcNMjQwNjEwMDM1OTUwWhcNMzQwNjA4
MDM1OTUwWjATMREwDwYDVQQDDAhTbmFrZU9pbDCCAiIwDQYJKoZIhvcNAQEBBQAD
ggIPADCCAgoCggIBANI+P5QXm9Bj21FIPsQqbqZRb5XmSZZJYaam7EIJ16Fxedf+
jXAv4d/FVqiEM4BuSNsNMeBMx2Gq0lAfN33h+RMTjRoMb8yBsZsC063MLfXCk4p+
09gtGP7BS6Iy5XdmfY/fPHvA3JDEScdlDDmd6Lsbdwhv93Q8M6POVO9sv4HuS4t/
jEjr+NhE+Bjr/wDbyg7GL71BP1WPZpQnRE4OzoSrt5+bZVLvODWUFwinB0fLaGRk
GmI0r5EUOUd7HpYyoIQbiNlePGfPpHRKnmdXTTEZEoxeWWAaM1VhPGqfrB/Pnca+
vAJX7iBOb3kHinmfVOScsG/YAUR94wSELeY+UlEWJaELVUntrJ5HeRDiTChiVQ++
wnnjNbepaW6shopybUF3XXfhIb4NvwLWpvoKFXVtcVjlOujF0snVvpE+MRT0wacy
tHtjZs7Ao7GYxDz6H8AdBLKJW67uQon37a4MI260ADFMS+2vEAbNSFP+f6ii5mrB
18cY64ZaF6oU8bjGK7BArDx56bRc3WFyuBIGWAFHEuB948BcshXY7baf5jjzPmgz
mq1zdRthQB31MOM2ii6vuTkheAvKfFf+llH4M9SnES4NSF2hj9NnHga9V08wfhYc
x0W6qu+S8HUdVF+V23yTvUNgz4Q+UoGs4sHSDEsIBFqNvInnpUmtNgcR2L5PAgMB
AAGjUzBRMB0GA1UdDgQWBBTPo8kfze4P9EgxNuyk7+xDGFtAYzAfBgNVHSMEGDAW
gBTPo8kfze4P9EgxNuyk7+xDGFtAYzAPBgNVHRMBAf8EBTADAQH/MA0GCSqGSIb3
DQEBCwUAA4ICAQAKHomtmcGqyiLnhziLe97Mq2+Sul5QgYVwfx/KYOXxv2T8ZmcR
Ae9XFhZT4jsAOUDK1OXx9aZgDGJHJLNEVTe9zWv1ONFfNxEBxQgP7hhmDBWdtj6d
taqEW/Jp06X+08BtnYK9NZsvDg2YRcvOHConeMjwvEL7tQK0m+GVyQfLYg6jnrhx
egH+abucTKxabFcWSE+Vk0uJYMqcbXvB4WNKz9vj4V5Hn7/DN4xIjFko+nREw6Oa
/AUFjNnO/FPjap+d68H1LdzMH3PSs+yjGid+6Zx9FCnt9qZydW13Miqg3nDnODXw
+Z682mQFjVlGPCA5ZOQbyMKY4tNazG2n8qy2famQT3+jF8Lb6a4NGbnpeWnLMkIu
jWLWIkA9MlbdNXuajiPNVyYIK9gdoBzbfaKwoOfSsLxEqlf8rio1GGcEV5Hlz5S2
txwI0xdW9MWeGWoiLbZSbRJH4TIBFFtoBG0LoEJi0C+UPwS8CDngJB4TyrZqEld3
rH87W+Et1t/Nepoc/Eoaux9PFp5VPXP+qwQGmhir/hv7OsgBhrkYuhkjxZ8+1uk7
tUWC/XM0mpLoxsq6vVl3AJaJe1ivdA9xLytsuG4iv02Juc593HXYR8yOpow0Eq2T
U5EyeuFg5RXYwAPi7ykw1PW7zAPL4MlonEVz+QXOSx6eyhimp1VZC11SCg==
-----END CERTIFICATE-----
subject=CN = SnakeOil
issuer=CN = SnakeOil
---
No client certificate CA names sent
Peer signing digest: SHA256
Peer signature type: RSA-PSS
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 2107 bytes and written 409 bytes
Verification error: self-signed certificate
---
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
Server public key is 4096 bit
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
Early data was not sent
Verify return code: 18 (self-signed certificate)
---
---
Post-Handshake New Session Ticket arrived:
SSL-Session:
    Protocol  : TLSv1.3
    Cipher    : TLS_AES_256_GCM_SHA384
    Session-ID: CA7BC7476320BCB49E2F31488BA19D18516B35DA2672D77E21E2886911A0E9B4
    Session-ID-ctx: 
    Resumption PSK: C1BC0792D7CDCED9AC7843FF6BA8B91736DCCF908B77DF57043BD8846387494C43F5578C7FD77562A5938E4589FA370B
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    TLS session ticket lifetime hint: 300 (seconds)
    TLS session ticket:
    0000 - 2f 6f 99 c6 de eb 14 f4-e8 9a 8f 20 2c be 48 ee   /o......... ,.H.
    0010 - 1d f8 3b 30 2c ca 4a df-9e dc 8c 93 2a 36 4f 74   ..;0,.J.....*6Ot
    0020 - 4d bd 45 f4 71 09 6e b4-0c ae d1 12 45 ff 8a 49   M.E.q.n.....E..I
    0030 - 16 e7 68 22 a2 a1 5f b3-0d 70 ba ab 3a f6 7b f1   ..h".._..p..:.{.
    0040 - 56 8e 7c 23 a1 7d 55 06-47 1a 55 d6 d3 e8 8a ae   V.|#.}U.G.U.....
    0050 - d9 54 18 f3 f4 09 d2 7e-d2 b0 d9 a0 60 5e b3 62   .T.....~....`^.b
    0060 - 0b 34 e2 da 47 e9 f5 00-da ba 35 ae 79 94 5a f1   .4..G.....5.y.Z.
    0070 - db 09 98 9b b2 7a c6 a8-f7 0a 39 9a e2 58 ca d3   .....z....9..X..
    0080 - bf 06 33 f5 60 ab 8c ab-14 24 89 8f 6b c6 d3 48   ..3.`....$..k..H
    0090 - eb 7c 69 16 ac 42 1f 23-27 19 3d 28 39 f1 cc 44   .|i..B.#'.=(9..D
    00a0 - 9a 4b 66 db 81 52 c8 5f-93 8f 98 17 1a ac cd 76   .Kf..R._.......v
    00b0 - c1 0e 19 e7 85 21 ab c2-93 c5 93 ba 03 bc 9c f0   .....!..........
    00c0 - 74 68 ba ca 2e 27 cb ea-f5 e6 8d a7 00 d6 3e 0f   th...'........>.
    00d0 - 14 7b 88 b9 86 4b 60 19-4a e5 2d c1 2c 03 ba 82   .{...K`.J.-.,...
    00e0 - 1d be a0 7f bf ec 90 a2-8a ad 88 5b 9a 66 06 05   ...........[.f..
    00f0 - 19 bc 8f c5 09 34 79 54-a2 fc 17 d9 b0 8e f1 67   .....4yT.......g

    Start Time: 1731872507
    Timeout   : 7200 (sec)
    Verify return code: 18 (self-signed certificate)
    Extended master secret: no
    Max Early Data: 0
---
read R BLOCK
---
Post-Handshake New Session Ticket arrived:
SSL-Session:
    Protocol  : TLSv1.3
    Cipher    : TLS_AES_256_GCM_SHA384
    Session-ID: C330F4FC3D4068EB725EC62FD875A129C6C53FC33F90D0815002BA73BFBA03D4
    Session-ID-ctx: 
    Resumption PSK: FA7A68141BE51A3412BC2FBC48676FE9080FB25B98700EE8D0BC306A4164F7A8A55B8BD802376D3BA0EAE996B832ED96
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    TLS session ticket lifetime hint: 300 (seconds)
    TLS session ticket:
    0000 - 2f 6f 99 c6 de eb 14 f4-e8 9a 8f 20 2c be 48 ee   /o......... ,.H.
    0010 - 46 28 54 4a 32 27 4c e1-91 bf e0 7b f7 01 89 97   F(TJ2'L....{....
    0020 - de 74 e1 7e 92 48 87 40-5b c2 25 be 30 a1 5a 02   .t.~.H.@[.%.0.Z.
    0030 - fb 0d 20 09 ca c9 5c aa-eb 1f 6f 91 49 0b ab 84   .. ...\...o.I...
    0040 - 64 2e 53 3e bc 1e 12 97-ae c8 b2 8f 2f f0 24 78   d.S>......../.$x
    0050 - e4 4f fb 3a 06 97 a3 40-15 99 8d 5a 7d e1 d1 e9   .O.:...@...Z}...
    0060 - ba e8 6b be fb 77 ec ee-2a 69 dd d1 1d 64 f5 5c   ..k..w..*i...d.\
    0070 - f5 ca 32 6e ad 5c 70 53-21 c2 01 ab 3b 95 31 94   ..2n.\pS!...;.1.
    0080 - 04 7b f7 4a 03 de 72 f0-0c 8d a6 cb 3f 07 a9 6b   .{.J..r.....?..k
    0090 - f1 d9 43 7c 99 d4 f3 9a-75 10 75 b1 e4 75 7d 91   ..C|....u.u..u}.
    00a0 - bf cc 59 59 0b d6 97 ad-87 fa c9 91 36 e4 25 7a   ..YY........6.%z
    00b0 - a5 f8 4e 2c 6a 4f c2 88-94 e8 66 4e 14 ef 6b 3c   ..N,jO....fN..k<
    00c0 - 17 8c 3c 40 b0 74 ab a4-18 64 da 4b 81 66 22 af   ..<@.t...d.K.f".
    00d0 - 10 c1 c5 f4 08 5e c0 d2-6d 34 17 a3 d0 dd e3 44   .....^..m4.....D
    00e0 - e5 24 67 ad 35 78 74 22-f0 09 af 78 cd 37 c3 c4   .$g.5xt"...x.7..
    00f0 - ba d8 fa a9 de 2a 2f 29-2b 26 e4 24 90 de a0 cf   .....*/)+&.$....

    Start Time: 1731872507
    Timeout   : 7200 (sec)
    Verify return code: 18 (self-signed certificate)
    Extended master secret: no
    Max Early Data: 0
---
read R BLOCK
kSkvUpMQ7lBYyCM4GBPvCvT1BfWRy0Dx
Correct!
-----BEGIN RSA PRIVATE KEY-----
MIIEogIBAAKCAQEAvmOkuifmMg6HL2YPIOjon6iWfbp7c3jx34YkYWqUH57SUdyJ
imZzeyGC0gtZPGujUSxiJSWI/oTqexh+cAMTSMlOJf7+BrJObArnxd9Y7YT2bRPQ
Ja6Lzb558YW3FZl87ORiO+rW4LCDCNd2lUvLE/GL2GWyuKN0K5iCd5TbtJzEkQTu
DSt2mcNn4rhAL+JFr56o4T6z8WWAW18BR6yGrMq7Q/kALHYW3OekePQAzL0VUYbW
JGTi65CxbCnzc/w4+mqQyvmzpWtMAzJTzAzQxNbkR2MBGySxDLrjg0LWN6sK7wNX
x0YVztz/zbIkPjfkU1jHS+9EbVNj+D1XFOJuaQIDAQABAoIBABagpxpM1aoLWfvD
KHcj10nqcoBc4oE11aFYQwik7xfW+24pRNuDE6SFthOar69jp5RlLwD1NhPx3iBl
J9nOM8OJ0VToum43UOS8YxF8WwhXriYGnc1sskbwpXOUDc9uX4+UESzH22P29ovd
d8WErY0gPxun8pbJLmxkAtWNhpMvfe0050vk9TL5wqbu9AlbssgTcCXkMQnPw9nC
YNN6DDP2lbcBrvgT9YCNL6C+ZKufD52yOQ9qOkwFTEQpjtF4uNtJom+asvlpmS8A
vLY9r60wYSvmZhNqBUrj7lyCtXMIu1kkd4w7F77k+DjHoAXyxcUp1DGL51sOmama
+TOWWgECgYEA8JtPxP0GRJ+IQkX262jM3dEIkza8ky5moIwUqYdsx0NxHgRRhORT
8c8hAuRBb2G82so8vUHk/fur85OEfc9TncnCY2crpoqsghifKLxrLgtT+qDpfZnx
SatLdt8GfQ85yA7hnWWJ2MxF3NaeSDm75Lsm+tBbAiyc9P2jGRNtMSkCgYEAypHd
HCctNi/FwjulhttFx/rHYKhLidZDFYeiE/v45bN4yFm8x7R/b0iE7KaszX+Exdvt
SghaTdcG0Knyw1bpJVyusavPzpaJMjdJ6tcFhVAbAjm7enCIvGCSx+X3l5SiWg0A
R57hJglezIiVjv3aGwHwvlZvtszK6zV6oXFAu0ECgYAbjo46T4hyP5tJi93V5HDi
Ttiek7xRVxUl+iU7rWkGAXFpMLFteQEsRr7PJ/lemmEY5eTDAFMLy9FL2m9oQWCg
R8VdwSk8r9FGLS+9aKcV5PI/WEKlwgXinB3OhYimtiG2Cg5JCqIZFHxD6MjEGOiu
L8ktHMPvodBwNsSBULpG0QKBgBAplTfC1HOnWiMGOU3KPwYWt0O6CdTkmJOmL8Ni
blh9elyZ9FsGxsgtRBXRsqXuz7wtsQAgLHxbdLq/ZJQ7YfzOKU4ZxEnabvXnvWkU
YOdjHdSOoKvDQNWu6ucyLRAWFuISeXw9a/9p7ftpxm0TSgyvmfLF2MIAEwyzRqaM
77pBAoGAMmjmIJdjp+Ez8duyn3ieo36yrttF5NSsJLAbxFpdlc1gvtGCWW+9Cq0b
dxviW8+TFVEBl1O4f7HVm6EpTscdDxU+bCXWkfjuRb7Dy9GOtt9JPsX8MBTakzh3
vBgsyi/sN3RqRBcGU40fOoZyfAMT8s1m/uYv52O6IgeuZ/ujbjY=
-----END RSA PRIVATE KEY-----

closed
```

So the right port was 31790 and I have been tinkering my mind a bit about that since in interactive mod, `openssl s_client` have the differents commands specified in Over The Wire but they have been shortened to the letters "d,r,k" which are checked eveytime you input something and as soon as one of the letter is the first of the string, the rest of the string is ignored and the comand is executed. Having a password starting by "k" I constantly sent "KEYUPDATE" commands, a simple solution is to use the `-nocommands` options which gives us a private key so I guess we can SSH to level 18 using that key. 

```
bandit16@bandit:~$ cd $(mktemp -d)
bandit16@bandit:/tmp/tmp.o9pD6IkpuZ$ nano sshkey.priv
bandit16@bandit:/tmp/tmp.o9pD6IkpuZ$ chmod 600 sshkey.priv 
```
It is important to change the permissions of the file having the key otherwise SSH will ignore it telling you that it has "too open" permissions.

I copied and pasted that key into a file to test just like level 14 and it worked.


## Level 18

> There are 2 files in the homedirectory: passwords.old and passwords.new. The password for the next level is in passwords.new and is the only line that has been changed between passwords.old and passwords.new

This level is clearly about using  the `diff` tool to make a difference between 2 files. By using this tool we can extract the password to bandit18.

```
bandit17@bandit:~$ diff passwords.old passwords.new 
42c42
< ktfgBvpMzWKR5ENj26IbLGSblgUG9CzB
---
> x2gLTTjFwMOhQ8oWNbMN362QKxfRqGlO
```

So the line removed in "password.old" compared to "password.new" is "ktfgBvpMzWKR5ENj26IbLGSblgUG9CzB", and have been replaced by "x2gLTTjFwMOhQ8oWNbMN362QKxfRqGlO" making it the password for the next level.

## Level 19

> The password for the next level is stored in a file readme in the homedirectory. Unfortunately, someone has modified .bashrc to log you out when you log in with SSH.

Well we can try inputing the command right after our `ssh` command just like we did before.

```
ro@Parrot:~/$ ssh -p 2220 bandit18@bandit.labs.overthewire.org cat readme
                         _                     _ _ _   
                        | |__   __ _ _ __   __| (_) |_ 
                        | '_ \ / _` | '_ \ / _` | | __|
                        | |_) | (_| | | | | (_| | | |_ 
                        |_.__/ \__,_|_| |_|\__,_|_|\__|
                                                       

                      This is an OverTheWire game server. 
            More information on http://www.overthewire.org/wargames

bandit18@bandit.labs.overthewire.org's password: 
cGWpMaKXVwDUNgPAVJbWYuGHVn9zl3j8
```

## Level 20

> To gain access to the next level, you should use the setuid binary in the homedirectory. Execute it without arguments to find out how to use it. The password for this level can be found in the usual place (/etc/bandit_pass), after you have used the setuid binary.

To overcome this challenge, you have to be aware of setuid binaries. In other words, binaries being executed as another user in the system. This is pretty much useful if you are trying to do some privileges escalation just like what is expected us to do here. 

```
bandit19@bandit:~$ ls -l 
total 16
-rwsr-x--- 1 bandit20 bandit19 14880 Sep 19 07:08 bandit20-do
bandit19@bandit:~$ ./bandit20-do 
Run a command as another user.
  Example: ./bandit20-do id
```

Once here, we have to find a way to exploit this binary by using it in another way as it is specified in the help section. We can see that whatever we are doing, it execute code as bandit20. Maybe the `id` argument is being executed somewhere in the code as an actual command. Let's give it a try.

```
bandit19@bandit:~$ ./bandit20-do  cat /etc/bandit_pass/bandit20
0qXahG8ZjOVMN9Ghs7iOWsCfZyXOUbYO
```

Bingo ! 

## Level 20

> There is a setuid binary in the homedirectory that does the following: it makes a connection to localhost on the port you specify as a commandline argument. It then reads a line of text from the connection and compares it to the password in the previous level (bandit20). If the password is correct, it will transmit the password for the next level (bandit21).

```
bandit20@bandit:~$ ./suconnect 
Usage: ./suconnect <portnumber>
This program will connect to the given port on localhost using TCP. If it receives the correct password from the other side, the next password is transmitted back.
```

This one involve having 2 terminals because it relies on a basic client/server architecture. The client will be `suconnect` and you have to setup a server somewhere (on internet or on the machine) using `netcat` to do the procedure. Make sure to have 2 terminals open. In a terminal you will setup `netcat` to listen on a specific, arbitrary choosen port:

### Terminal 1/server
```
netcat -lvnp 40568 
Listening on 0.0.0.0 40568
```

And then initiate a connection with suconnect on the other terminal:

### Terminal 2/client
```
bandit20@bandit:~$ ./suconnect 40568
```

Then you go back to Terminal 1, and you will see that a connection have been accepted. Past the password of the actual user, it will be compared in suconnect which will send the password of the next level in return.

### Terminal 1/server
```
netcat -lvnp 40568 
Listening on 0.0.0.0 40568
0qXahG8ZjOVMN9Ghs7iOWsCfZyXOUbYO
EeoULMCra2q0dSkYj561DX7s1CpBuOBt
```

This is a typical Out-Of-Band exfiltration where you either have to make a server listen locally or outside because you cannot reach the data locally so you need to have a third instance involved (other that you and the SSH server you are talking to).

## Level 21

> A program is running automatically at regular intervals from cron, the time-based job scheduler. Look in /etc/cron.d/ for the configuration and see what command is being executed.

Here we will take advantage of a cronjob. 

```
ls /etc/cron.d/ 
cronjob_bandit22  cronjob_bandit23  cronjob_bandit24  e2scrub_all  otw-tmp-dir  sysstat
bandit21@bandit:~$ cat  /etc/cron.d/cronjob_bandit22
@reboot bandit22 /usr/bin/cronjob_bandit22.sh &> /dev/null
* * * * * bandit22 /usr/bin/cronjob_bandit22.sh &> /dev/null
bandit21@bandit:~$ cat /usr/bin/cronjob_bandit22.sh 
#!/bin/bash
chmod 644 /tmp/t7O6lds9S0RqQh9aMcz6ShpAoZKF7fgv
cat /etc/bandit_pass/bandit22 > /tmp/t7O6lds9S0RqQh9aMcz6ShpAoZKF7fgv
bandit21@bandit:~$ cat /tmp/t7O6lds9S0RqQh9aMcz6ShpAoZKF7fgv
tRae0UfB9v0UzbCdn9cY0gQnds9GF58Q
```

Pretty straight forward, the cronjob is executing a bash script at each reboot. We have access to read this bash script which output the content of "/etc/bandit_pass/bandit22" in a temporary file under "/tmp", we use `cat` to read the file and get the password. 

## Level 22

> A program is running automatically at regular intervals from cron, the time-based job scheduler. Look in /etc/cron.d/ for the configuration and see what command is being executed.

Same goes on here with bash, we can be confident guessing the cron file associated with this level.

```
bandit22@bandit:~$ cat /etc/cron.d/cronjob_bandit23
@reboot bandit23 /usr/bin/cronjob_bandit23.sh  &> /dev/null
* * * * * bandit23 /usr/bin/cronjob_bandit23.sh  &> /dev/null
bandit22@bandit:~$ cat /usr/bin/cronjob_bandit23.sh 
```
```Bash
#!/bin/bash

myname=$(whoami)
mytarget=$(echo I am user $myname | md5sum | cut -d ' ' -f 1)

echo "Copying passwordfile /etc/bandit_pass/$myname to /tmp/$mytarget"

cat /etc/bandit_pass/$myname > /tmp/$mytarget
```

Okay so now a bit of Bash skills helps us o understand that the content of the file at "/etc/bandit_pass/\$myname" is outputed to a temporary file have the content of the "\$mytarget" name.

- "\$myname" is the result of whoami. In other words the user executing the script and we can clearly see in the cronjob that the script is executed as "bandit23" so is the content of "\$myname"
- "\$mytarget" is a string having "\$myname" hashed in MD5 (see `md5sum` command) and trimmed using `tr`
Now that we have all the pieces of the puzzle, we can output the password:

```
bandit22@bandit:~$ cat /tmp/$(echo "I am user bandit23"| md5sum | cut -d ' ' -f 1)
0Zf11ioIjMVN551jX3CmStKLYqjk54Ga
```

## Level 23

> A program is running automatically at regular intervals from cron, the time-based job scheduler. Look in /etc/cron.d/ for the configuration and see what command is being executed.

Same principle as the 2 previous challenges.

```
bandit23@bandit:~$ cat /etc/cron.d/cronjob_bandit24 
@reboot bandit24 /usr/bin/cronjob_bandit24.sh &> /dev/null
* * * * * bandit24 /usr/bin/cronjob_bandit24.sh &> /dev/null
bandit23@bandit:~$ cat /usr/bin/cronjob_bandit24.sh
```
```Bash
#!/bin/bash

myname=$(whoami)

cd /var/spool/$myname/foo
echo "Executing and deleting all scripts in /var/spool/$myname/foo:"
for i in * .*;
do
    if [ "$i" != "." -a "$i" != ".." ];
    then
        echo "Handling $i"
        owner="$(stat --format "%U" ./$i)"
        if [ "${owner}" = "bandit23" ]; then
            timeout -s 9 60 ./$i
        fi
        rm -f ./$i
    fi
done
```

So this script is being executed everytime as bandit24. We do not really need to break it down if it really does what the `echo` command says. We just have to create a bash script and copy it to the right directory so it gets executed and deleted. 

Let's first place us in a writeable directory

```
bandit23@bandit:~$ cd $(mktemp -d)
bandit23@bandit:/tmp/tmp.eCxpiGDiOE$
```

```Bash
#!/bin/bash

mydir="/tmp/tmp.eCxpiGDiOE" # our temporary directory
myuser="bandit23" # test localy with bandit23 before sending
cat /etc/bandit_pass/$myuser > $mydir/extracted_password.txt
```

We write the content using `nano` and make the file executable. and test it with "bandit23"

```
bandit23@bandit:/tmp/tmp.eCxpiGDiOE$ nano myscript.sh 
Unable to create directory /home/bandit23/.local/share/nano/: No such file or directory
It is required for saving/loading search history or cursor positions.

bandit23@bandit:/tmp/tmp.eCxpiGDiOE$ chmod +x myscript.sh
bandit23@bandit:/tmp/tmp.eCxpiGDiOE$ ./myscript.sh 
bandit23@bandit:/tmp/tmp.eCxpiGDiOE$ cat extracted_password.txt 
0Zf11ioIjMVN551jX3CmStKLYqjk54Ga
```

It seems to work so we just have to adjust the variable "\$myname" and copying it to "/var/spool/$myname/foo" where "\$myname" is bandit24. Cron takes a couple of minutes to work so do not expect immediate results. 

Also you can either remove extracted_password.txt, either set its permissions to 777 so bandit24 is allowed to write in. Same goes on for the directory you are in.

```
bandit23@bandit:/tmp/tmp.eCxpiGDiOE$ chmod 777 $(pwd)
bandit23@bandit:/tmp/tmp.eCxpiGDiOE$ ls
extracted_password.txt  myscript.sh
bandit23@bandit:/tmp/tmp.eCxpiGDiOE$ cat extracted_password_bandit24.txt 
gb8KRRCsshuZXI0tUuR6ypOFjiZbf3G8
```

This is a good level to get around permissions

## Level 24

> A daemon is listening on port 30002 and will give you the password for bandit25 if given the password for bandit24 and a secret numeric 4-digit pincode. There is no way to retrieve the pincode except by going through all of the 10000 combinations, called brute-forcing.
You do not need to create new connections each time

Looks like it is time to create a small Bash bruteforcer for this use-case. 

```Bash
#!/bin/bash 
for i in $(seq 0 9999);
do
    out=$(printf "gb8KRRCsshuZXI0tUuR6ypOFjiZbf3G8 %04d\n" $i | nc -q 0 localhost 30002)
    if ! [[ $out =~ "Wrong!" ]]; then
        echo "$out" 
        break
    fi  
done

```

There may be other way to bruteforce the thing as I am initiating a new connection each time but I am more focused on small develoment time rather than execution time. So we use `nano` to paste the code, and execute it. 

```
bandit24@bandit:/tmp/tmp.XTXqGXilXb$ nano brute.sh && chmod +x brute.sh
bandit24@bandit:/tmp/tmp.XTXqGXilXb$ ./brute.sh 
I am the pincode checker for user bandit25. Please enter the password for user bandit24 and the secret pincode on a single line, separated by a space.
Correct!
The password of user bandit25 is iCi86ttT4KSNe1armKiwbQNmB3YJP3q4
```

## Level 25
> Logging in to bandit26 from bandit25 should be fairly easy… The shell for user bandit26 is not /bin/bash, but something else. Find out what it is, how it works and how to break out of it.

So we connect and find an ssh key for bandit26; we attemtps a connection from here and we get:

```
bandit25@bandit:~$ ls 
bandit26.sshkey
bandit25@bandit:~$ ssh -p 2220 -i bandit26.sshkey bandit26@localhost
# The usual stuff about key confirmation + message of the day...
  _                     _ _ _   ___   __  
 | |                   | (_) | |__ \ / /  
 | |__   __ _ _ __   __| |_| |_   ) / /_  
 | '_ \ / _` | '_ \ / _` | | __| / / '_ \ 
 | |_) | (_| | | | | (_| | | |_ / /| (_) |
 |_.__/ \__,_|_| |_|\__,_|_|\__|____\___/ 
Connection to localhost closed.
```

I get immediatly disconnected from the connection. Something is in the way. 

The statement says it uses a custom shell, we can figure out which shell by using "/etc/passwd" if we have access.

```
bandit25@bandit:~$ cat /etc/passwd | grep bandit26
bandit26:x:11026:11026:bandit level 26:/home/bandit26:/usr/bin/showtext
bandit25@bandit:~$ file /usr/bin/showtext 
/usr/bin/showtext: POSIX shell script, ASCII text executable
bandit25@bandit:~$ cat /usr/bin/showtext 
#!/bin/sh

export TERM=linux

exec more ~/text.txt
exit 0
```

So now we know that the shell we are running into is a shell script. We have to find a way to break it. 

The usage of more is tricky here, by resizing your actual window, you can stop the script because `more` will page the output for you to see. There is actually a way to start a shell from there but it will start "/usr/bin/showtext" again which is useless. Another tricky workaround is to go through vi which is runnable from more with the command ":v". Once in vi, you can use vi commands to pop up a nice shell.

```
:set shell=/bin/bash
:shell
bandit26@bandit:~$ 
bandit26@bandit:~$ cat /etc/bandit_pass/bandit26
s0773xxkk0MXfdqOfPRVr9L3jJBUOgCZ
```
At this point you should have started to understand that each password is stored in /etc/bandit_pass/banditXX. 
The catch is each file is only readable by its user, we can still note the password for bandit 26 though.

## Level 26

> Good job getting a shell! Now hurry and grab the password for bandit27!

This level is the same as level 20:
```
bandit26@bandit:~$ ls 
bandit27-do  text.txt
bandit26@bandit:~$ ./bandit27-do cat /etc/bandit_pass/bandit27
upsNCc7vzaRDx6oZC6GiR6ERwe1MowGB
```
## Level 27

> There is a git repository at ssh://bandit27-git@localhost/home/bandit27-git/repo via the port 2220. The password for the user bandit27-git is the same as for the user bandit27.

For this level we will be using the command `git` which is a well known versionning tool made by Linus Torvald for coding project and keeping track of changes in the code. We clone the project in a temporary directory and we can investigate further.

```
bandit27@bandit:~$ cd $(mktemp -d)
bandit27@bandit:/tmp/tmp.7SrYbycTuO$ git clone ssh://bandit27-git@localhost:2220/home/bandit27-git/repo
```
This welcome you with the message of the day and ask you for the password which will be the same as bandit27 as written in the statement, then you have to go to the directory and look around. 

```
bandit27@bandit:/tmp/tmp.7SrYbycTuO$ cd repo/
bandit27@bandit:/tmp/tmp.7SrYbycTuO/repo$ ls 
README
bandit27@bandit:/tmp/tmp.7SrYbycTuO/repo$ cat README 
The password to the next level is: Yz9IpL0sBcCeuG7m9uQFt8ZNpS4HZRcN
```
Prettty easy.

## Level 28

> There is a git repository at ssh://bandit28-git@localhost/home/bandit28-git/repo via the port 2220. The password for the user bandit28-git is the same as for the user bandit28.


Using the same stuff we seen in the previous level...

```
bandit28@bandit:~$ cd $(mktemp -d) && git clone ssh://bandit28-git@localhost:2220/home/bandit28-git/repo 
[Extra blabla and password prompt then cloning the repo]
bandit28@bandit:/tmp/tmp.USgMJFA1SO$ cd repo/
bandit28@bandit:/tmp/tmp.USgMJFA1SO/repo$ ls 
README.md
bandit28@bandit:/tmp/tmp.USgMJFA1SO/repo$ cat README.md 
# Bandit Notes
Some notes for level29 of bandit.

## credentials

- username: bandit29
- password: xxxxxxxxxx

bandit28@bandit:/tmp/tmp.USgMJFA1SO/repo$ git log --oneline 
817e303 (HEAD -> master, origin/master, origin/HEAD) fix info leak
3621de8 add missing data
0622b73 initial commit of README.md
```
`git log --oneline` is a command that sums up the number of commits (changes) brought to the project. Here we can see that it had 3 commits. In this scenario we want to make us believe that the developper who coded this made an initial commit, then added "missing data" which should be the password, but realised it was a bad idea, then did another commit on top of it to remove it, which is bad practice and should never be done this way because we can actually rollback to the commit when he added the password.

```
bandit28@bandit:/tmp/tmp.USgMJFA1SO/repo$ git checkout 3621de8 
Note: switching to '3621de8'.

You are in 'detached HEAD' state. You can look around, make experimental
changes and commit them, and you can discard any commits you make in this
state without impacting any branches by switching back to a branch.

If you want to create a new branch to retain commits you create, you may
do so (now or later) by using -c with the switch command. Example:

  git switch -c <new-branch-name>

Or undo this operation with:

  git switch -

Turn off this advice by setting config variable advice.detachedHead to false

HEAD is now at 3621de8 add missing data
bandit28@bandit:/tmp/tmp.USgMJFA1SO/repo$ cat README.md 
# Bandit Notes
Some notes for level29 of bandit.

## credentials

- username: bandit29
- password: 4pT1t5DENaYuqnqvadYs1oE4QLCdjmJ7
```
There we go.

## Level 29

> There is a git repository at ssh://bandit29-git@localhost/home/bandit29-git/repo via the port 2220. The password for the user bandit29-git is the same as for the user bandit29.

Again a git challenge, I am skipping the cloning part, see what you have to do in the previous levels 

```
bandit29@bandit:/tmp/tmp.UtNvVml7pg$ cd repo/
bandit29@bandit:/tmp/tmp.UtNvVml7pg/repo$ ls 
README.md
bandit29@bandit:/tmp/tmp.UtNvVml7pg/repo$ cat README.md 
# Bandit Notes
Some notes for bandit30 of bandit.

## credentials

- username: bandit30
- password: <no passwords in production!>

```

Here comes the notion of branches in `git`. When you work on a project with multiple people, sometimes you need to add a functionnality requiring to have your own version of the code that doesn't depend on other people's changes, version that you will merge later when your functionnality is fully operationnal... Or if it doesn't work and you do not want to do it anymore, you can just throw it away without ruining the rest of the code. This is what banches are made for. Sometimes developpers have a production branch and a dev branch for different purposes. So let's see our branches here

```
git branch -la
* master
  remotes/origin/HEAD -> origin/master
  remotes/origin/dev
  remotes/origin/master
  remotes/origin/sploits-dev
```

Looks like we found our dev branch, let's `git switch` to it and we can get this done. 

```
bandit29@bandit:/tmp/tmp.UtNvVml7pg/repo$ git switch dev 
branch 'dev' set up to track 'origin/dev'.
Switched to a new branch 'dev'
bandit29@bandit:/tmp/tmp.UtNvVml7pg/repo$ cat README.md 
# Bandit Notes
Some notes for bandit30 of bandit.

## credentials

- username: bandit30
- password: qp30ex3VLz5MDG1n91YowTv4Q8l7CDZL
```

## Level 30

> There is a git repository at ssh://bandit30-git@localhost/home/bandit30-git/repo via the port 2220. The password for the user bandit30-git is the same as for the user bandit30.


Again `git`, same procedure. 

```
bandit30@bandit:/tmp/tmp.WqZDOiL0Pu/repo$ cat README.md 
just an epmty file... muahaha
bandit30@bandit:/tmp/tmp.WqZDOiL0Pu/repo$ git branch -la 
* master
  remotes/origin/HEAD -> origin/master
  remotes/origin/master
bandit30@bandit:/tmp/tmp.WqZDOiL0Pu/repo$ git logs --oneline 
git: 'logs' is not a git command. See 'git --help'.

The most similar command is
	log
bandit30@bandit:/tmp/tmp.WqZDOiL0Pu/repo$ git log --oneline 
acfc3c6 (HEAD -> master, origin/master, origin/HEAD) initial commit of README.md
bandit30@bandit:/tmp/tmp.WqZDOiL0Pu/repo$ git status 
On branch master
Your branch is up to date with 'origin/master'.

nothing to commit, working tree clean
```
Eveything is disapointing so far because none of ouf strategies here works. We can also take a look as the stash (place where you can put crappy changes awaiting to be commited and you are switching between branches/commits, etc.)
```
bandit30@bandit:/tmp/tmp.WqZDOiL0Pu/repo$ git stash list
```

But this gives us nothing either. Then while inspecting the references in the ".git" directory, I remembered that you can actually use tags with commits on `git` (see tags as "Milestones" reached in a project, often associated with a version number, which wasn't the case here).

```
bandit30@bandit:/tmp/tmp.WqZDOiL0Pu/repo/.git$ cat packed-refs 
# pack-refs with: peeled fully-peeled sorted 
acfc3c67816fc778c4aeb5893299451ca6d65a78 refs/remotes/origin/master
84368f3a7ee06ac993ed579e34b8bd144afad351 refs/tags/secret
bandit30@bandit:/tmp/tmp.WqZDOiL0Pu/repo$ git switch --detach 84368f3a7ee0
fatal: reference is not a tree: 84368f3a7ee0
```

Here we can see that I am doing some nonsense as the has associated with the tag is not a commit hash but it helped me to figure out which kind of git object it was because I started out from the basics, that is to say print an object by its hash no matter what it is:

```
bandit30@bandit:/tmp/tmp.WqZDOiL0Pu/repo$ git cat-file -p 84368f3a7ee06ac993ed579e34b8bd144afad351
fb5S2xb7bRyFmAvQYQGEqsbhVyJqhnDy
```

I wasn't sure it was the password but it looked like it (same 32 characters lenght string). So I tried it and it worked.

## Level 31 

>There is a git repository at ssh://bandit31-git@localhost/home/bandit31-git/repo via the port 2220. The password for the user bandit31-git is the same as for the user bandit31.

Here is another `git` challenge. Again.

```
bandit31@bandit:/tmp/tmp.K7JjBWxHmj$ cd repo/
bandit31@bandit:/tmp/tmp.K7JjBWxHmj/repo$ cat README.md 
This time your task is to push a file to the remote repository.

Details:
    File name: key.txt
    Content: 'May I come in?'
    Branch: master
```

This seems pretty straightforward for someone knowing `git`:
1) Write a file
2) Adding it to git 
3) Commiting the changes 
4) Pushing to the remote branch

Obviously there was a catch:

```
andit31@bandit:/tmp/tmp.K7JjBWxHmj/repo$ echo "May I come in?" > key.txt
bandit31@bandit:/tmp/tmp.K7JjBWxHmj/repo$ git add key.txt 
bandit31@bandit:/tmp/tmp.K7JjBWxHmj/repo$ git commit 
On branch master
Your branch is ahead of 'origin/master' by 1 commit.
  (use "git push" to publish your local commits)

nothing to commit, working tree clean
```

Nothing to commit even though I added the file. In git there is a hidden file called gitignore preventing you from adding files by mistakes:

```
bandit31@bandit:/tmp/tmp.K7JjBWxHmj/repo$ cat .gitignore 
*.txt
```
Here we can see that all added the files ending up by ".txt" are ignored. And git will ignore all changes made to ".gitignore" unless they are commited.
A simple workaround is to actually force-add the file and commit using `git add -f key.txt`.

```
bandit31@bandit:/tmp/tmp.K7JjBWxHmj/repo$ git add -f key.txt 
bandit31@bandit:/tmp/tmp.K7JjBWxHmj/repo$ git commit 
On branch master
Your branch is ahead of 'origin/master' by 1 commit.
  (use "git push" to publish your local commits)

nothing to commit, working tree clean
bandit31@bandit:/tmp/tmp.K7JjBWxHmj/repo$ git push 
The authenticity of host '[localhost]:2220 ([127.0.0.1]:2220)' can't be established.
ED25519 key fingerprint is SHA256:C2ihUBV7ihnV1wUXRb4RrEcLfXC5CXlhmAAM/urerLY.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Could not create directory '/home/bandit31/.ssh' (Permission denied).
Failed to add the host to the list of known hosts (/home/bandit31/.ssh/known_hosts).
                         _                     _ _ _   
                        | |__   __ _ _ __   __| (_) |_ 
                        | '_ \ / _` | '_ \ / _` | | __|
                        | |_) | (_| | | | | (_| | | |_ 
                        |_.__/ \__,_|_| |_|\__,_|_|\__|
                                                       

                      This is an OverTheWire game server. 
            More information on http://www.overthewire.org/wargames

bandit31-git@localhost's password: 
Enumerating objects: 4, done.
Counting objects: 100% (4/4), done.
Delta compression using up to 2 threads
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 320 bytes | 160.00 KiB/s, done.
Total 3 (delta 0), reused 0 (delta 0), pack-reused 0
remote: ### Attempting to validate files... ####
remote: 
remote: .oOo.oOo.oOo.oOo.oOo.oOo.oOo.oOo.oOo.oOo.
remote: 
remote: Well done! Here is the password for the next level:
remote: 3O9RfhqyAlVBEZpVb6LYStshZoqoSx5K 
remote: 
remote: .oOo.oOo.oOo.oOo.oOo.oOo.oOo.oOo.oOo.oOo.
remote: 
To ssh://localhost:2220/home/bandit31-git/repo
 ! [remote rejected] master -> master (pre-receive hook declined)
error: failed to push some refs to 'ssh://localhost:2220/home/bandit31-git/repo'
```
We can see that our remote got rejected but it doesn't matter because we dumped the key.

## Level 32

> After all this git stuff, it’s time for another escape. Good luck!

FINALLY we are done with `git` ! but the statement doesn't give us any clue on what to do. 

```
WELCOME TO THE UPPERCASE SHELL
>> 
```
Okay so this look nothing like any shell, we are most likely in a "jail" in which we have to escape. first step are discovery to see what we are able to do or not by testing out some commands. It it written "UPPERCASE SHELL". We have to get every single little detail about this shell to know how to escape.

```
>> help 
sh: 1: HELP: Permission denied
>> test 
sh: 1: TEST: Permission denied
>> echo AA
sh: 1: ECHO: Permission denied
>> cd
sh: 1: CD: Permission denied
>> .
>> ..
sh: 1: ..: Permission denied
>> README
sh: 1: README: Permission denied
>> man cat 
sh: 1: MAN: Permission denied
>> man 
sh: 1: MAN: Permission denied
>> whoami
sh: 1: WHOAMI: Permission denied
>> /bin/bash 
sh: 1: /BIN/BASH: not found
```
Here you can see that every command that I type is uppercased and we are getting a different error if I actually type a command or if I type a path

```
>> ^[[C^[[D^[[A^[[B^[[D
sh: 1:: Permission denied
>> 
```

By actually trying to move the cursor with arrows, it turned out there was characters, which you can typically find on shells such as `sh`, which turned out to be the case since we had our error showing up `sh`

```
>> "
sh: 2: Syntax error: Unterminated quoted string
>> " /bin/bash
sh: 2: Syntax error: Unterminated quoted string
```
Quotes seems to be handled correctly

```
>> VAR=3
>> VAR=$(echo TEST)
sh: 1: ECHO: Permission denied
>> #test
```
And we can initiate variables but the prompt is uppercased before anything is executed; we also have comments. 

Still looking for variables, we can see them in the errors when they are trying to be executed as commands:

```
>> $ENV
>> $PATH
sh: 1: /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin: not found
```

We can try the different variables but most of them doesn't work. That said I tried to expand the shell using "$0" which contains the first argument of the command. And...

```
>> $0
$ ls
uppershell
$ bash
bandit33@bandit:~$ ls 
uppershell
bandit33@bandit:~$ cat /etc/bandit_pass/bandit33 
tQdtbs5D5i2vJwkO8mEyYEyTL8izoeJ0
```
I just escaped and now in Level 33. As of today, there is no Level 34 yet which means we reached the end. 

# Conclusion
This was more tough than expected. Especially level 33 and 25 where it took me a bit more time to figure out what was going on but it is still a great experience, something fun to play with and to come back to when you start getting a bit rusty with the command line. 