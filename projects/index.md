# Projects


{{< figure src="/img/projectscover.jpg">}}

# SUBway
[{{< fa github 3x>}}](https://github.com/sam-lane/subway)
{{< admonition type=info open=true >}}
CLI tool for subdomain discovery
{{< /admonition >}}
Enumerate subdomains by either using DNS lookup or by virtual hosting HTTP requests. Useful for things like Hack The Box or Try Hack Me when no DNS server is available.

vhost lookups as simple as supply a list of domains and the IP.

```bash
subway -h example.com -w subdomains.txt -v -ip 127.0.0.1
====================================
   ______  _____                  
  / __/ / / / _ )_    _____ ___ __
 _\ \/ /_/ / _  | |/|/ / _ '/ // /
/___/\____/____/|__,__/\_,_/\_, / 
                           /___/  
====================================

host: example.com
wordlist: subdomains.txt
====================================
www.example.com
wild.example.com
mail.example.com
server.example.com
dev1.example.com
100:100
done
```
Includes features to filter out reponses of certain length or filter out reponse codes.

# Gissue
[{{< fa github 3x>}}](https://github.com/sam-lane/gissue) [{{< fa python 3x>}}](https://pypi.org/project/gissue/)
{{< admonition type=info open=true >}}
A CLI for managing GitHub issues in your project
{{< /admonition >}}

Gissue is a CLI program for managing your Github issues for your project in the command line. It allows you to view all issues in a project and create new issues all without leaving your comfy terminal session.


# Shell-Search
[{{< fa github 3x>}}](https://github.com/sam-lane/shell-search) [{{< fa npm 3x>}}](https://www.npmjs.com/package/shell-search)
{{< admonition type=info open=true >}}
**Quickly** search google and open your browser window from the command line.
{{< /admonition >}}

I think I spend more time reading Stackoverflow than I do writing code in my text editor. When I'm working I always have three things open - VScode, a web browser and a terminal. I write my code go to my terminal run it and if an error pops up I am unfamilar with to my web browser I go in the hopes of finding a stackoverflow question that fufills my needs. The annoying part of all this, I have to tab over to my browser open another tab, you can't have too many tabs open, and enter my query. I wanted to streamline this _ever so dedious_ process. I had recently been learning a bit of Javascript and thought I would have ago and writing a command line program that I can type my query into hit enter the browser googles that query for me and makes it my active window.

Shell-search is easy to use. Install it (instructions are on the github) and issue a search with 
```bash
search <query here>
```
