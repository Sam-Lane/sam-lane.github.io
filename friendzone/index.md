# Friendzone


# HTB Writeup: friendzone

## Introduction
Friendzone was my third box to own on HackTheBox. I learnt **alot** from this box. Such as exploiting Local File Inclusion (LFI) to have PHP execute my reverse shell, to understanding more about DNS and the ways python imports libraries.


## Enumeration

### nmap
Running nmap on a server will give a lot of information on what ports are open on the machine and what services those ports belong to. 

```bash
nmap -sS -sC -sV -oA friendzone -p- -T4 10.10.10.123
```
From running the above command we see the following services and ports running on friendzone.

- ftp 21
- ssh 22
- domain 53
- http 80
- samba 139
- https 443
- samba 445

### Looking into the samba share.
Kali comes with a great tool smbmap for doing recon on samba shares.

```bash
smbmap -H 10.10.10.123
```
Running it we get this result.
```bash
root@kali:~# smbmap -H 10.10.10.123
[+] Finding open SMB ports....
[+] Guest SMB session established on 10.10.10.123...
[+] IP: 10.10.10.123:445	Name: administrator1.friendzone.red                     
	Disk                                                  	Permissions
	----                                                  	-----------
	print$                                            	NO ACCESS
	Files                                             	NO ACCESS
	general                                           	READ ONLY
	Development                                       	READ, WRITE
	IPC$ 

```

This tells use we have read only access to general and can read and write to Development. Lets start of by seeing what can be found in general. To do this I used smbclient on kali:

```bash
smbclient //10.10.10.123/general
```

When prompted for a password you can just hit enter and get a smb shell. Hitting ```ls``` there is a file called creds.txt lets grab that with ```get creds.txt``` putting it onto the local machine and using cat to read it we get:

> creds for the admin THING:
> 
> admin:WORKWORKHhallelujah@#

Lets have a quick look into Development using the same smbclient command as before and see if there is anything interesting. It just contains rubbish but lets remember for the future we can write to this directory and that can be incredible useful for exploitation.

### The Website
So we have admin credential that do not work for ssh so maybe the website contains. I used dirb which is a peice of software that takes a list of common urls and trys them against a target. Dirb proved to not find anything other than a mocking robots.txt and an index page:

{{< figure src="/img/friendzonewebsite.png" alt="friendzone website" position="center" style="border-radius: 8px;" caption="/index.html" captionPosition="right" captionStyle="color: red;" >}}

That is when I remembered the server was running a DNS service...

### Digging through DNS.
DNS is the service that resolves a human readable website url such as www.google.com to an ip address. Thanks to <cite>ippsec[1]</cite> video were he uses a range of tools to enumerate DNS. The only command he used that was any use was dig. Running:

```bash
dig axfr friendzoneportal.red @10.10.10.123
```
Getting the domain friendzoneportal.red from the home page email address returns:

```bash
root@kali:~/HTB/FriendZone# dig axfr friendzoneportal.red @10.10.10.123

; <<>> DiG 9.11.5-P4-3-Debian <<>> axfr friendzoneportal.red @10.10.10.123
;; global options: +cmd
friendzoneportal.red.	604800	IN	SOA	localhost. root.localhost. 2 604800 86400 2419200 604800
friendzoneportal.red.	604800	IN	AAAA	::1
friendzoneportal.red.	604800	IN	NS	localhost.
friendzoneportal.red.	604800	IN	A	127.0.0.1
admin.friendzoneportal.red. 604800 IN	A	127.0.0.1
files.friendzoneportal.red. 604800 IN	A	127.0.0.1
imports.friendzoneportal.red. 604800 IN	A	127.0.0.1
vpn.friendzoneportal.red. 604800 IN	A	127.0.0.1
friendzoneportal.red.	604800	IN	SOA	localhost. root.localhost. 2 604800 86400 2419200 604800
;; Query time: 28 msec
;; SERVER: 10.10.10.123#53(10.10.10.123)
;; WHEN: Thu Jul 11 15:17:22 BST 2019
;; XFR size: 9 records (messages 1, bytes 309)
```

> admin.friendzoneportal.red

Great! we have found the admin page. Lets update our /etc/hosts file to use these new domains. Unfortuantly you cannot use the servers dns service as they all resolve to localhost. Initially visiting admin.friendzoneportal.red just shows the same index page. So lets try https

Excellent we have a login form but when we log in we are just told

> Admin page is not developed yet !!! check for another one

so more digging is required. Checking the SSL certificate from the website can see the issue is from a friendzone.red lets run dig again but this time zone transfer without the portal in the domain.

```bash
root@kali:~/HTB/FriendZone# dig axfr friendzone.red @10.10.10.123

; <<>> DiG 9.11.5-P4-3-Debian <<>> axfr friendzone.red @10.10.10.123
;; global options: +cmd
friendzone.red.		604800	IN	SOA	localhost. root.localhost. 2 604800 86400 2419200 604800
friendzone.red.		604800	IN	AAAA	::1
friendzone.red.		604800	IN	NS	localhost.
friendzone.red.		604800	IN	A	127.0.0.1
administrator1.friendzone.red. 604800 IN A	127.0.0.1
hr.friendzone.red.	604800	IN	A	127.0.0.1
uploads.friendzone.red.	604800	IN	A	127.0.0.1
friendzone.red.		604800	IN	SOA	localhost. root.localhost. 2 604800 86400 2419200 604800
;; Query time: 25 msec
;; SERVER: 10.10.10.123#53(10.10.10.123)
;; WHEN: Thu Jul 11 16:01:52 BST 2019
;; XFR size: 8 records (messages 1, bytes 289)
```

Update /etc/hosts and visit https://administrator1.friendzone.red

{{< figure src="/img/te.png" alt="Admin Login" position="center" style="border-radius: 8px;" caption="/index.html" captionPosition="right" captionStyle="color: red;" >}}

Login using the username and password from creds.txt we got earlier from the samba share. On logging in we are told to visit /dashboard.php


## Exploiting the LFI.

## 





## References
[1]:https://youtu.be/JRPWFSzFaG0?t=75
1. [ippsec](https://youtu.be/JRPWFSzFaG0?t=75)
[2]:http://pentestmonkey.net/tools/web-shells/php-reverse-shell
2. [php-reverse-shell](http://pentestmonkey.net/tools/web-shells/php-reverse-shell)
