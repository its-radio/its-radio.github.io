---
published: False
layout: post
title: Windows Forensics Cheat Sheet
date: 2025-02-25
description: This is a cheat sheet for Windows forensics. It is a summary of techniques for obtaining various types of information from various forensic sources from Windows systems.
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

# How to view things

## Registry
[EZ-Tools](https://ericzimmerman.github.io/#!index.md) RegistryExplorer

# Where to find/get things {#locate}

## Registry Hives {#locate-reg}

### In the filesystem

System registry hives are located in:

```plain
C:\Windows\System32\config\
```

User-specific hives are located in:

```plain
C:\Users\<username>\NTUSER.DAT
```

# Network

## Local IP, DHCP, DNS, Default Gateway

Some values can be found in:

```plain
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters\Interfaces
```

While others can be found one level deeper in:
```plain
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters\Interfaces\{interface-GUID}
```

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0 small-margin">
        {% include figure.liquid loading="eager" path="assets/img/cs-windows/1.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">
    Caption
</div>

## FTP Connection Information & History

### 1. Windows Explorer (File Explorer) FTP History  
**Registry Path:**  

```plain
HKEY_CURRENT_USER\Software\Microsoft\FTP\Accounts
```
- Stores saved FTP credentials and connection details.

### 2. Internet Explorer / Windows Explorer FTP Sessions  
**Registry Path:**  

```
HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\ComDlg32\OpenSaveMRU\ftp
```
Contains a list of previously accessed FTP sites.

### 3. WinSCP  
**Registry Path:**  

```
HKEY_CURRENT_USER\Software\Martin Prikryl\WinSCP 2\Sessions
```
Stores connection details for WinSCP sessions.

### 4. FileZilla 
FileZilla does not store connections in the registry but keeps them in configuration files. These files can be small enough to be resident files in the MFT.
**File Paths:**
```
%APPDATA%\Roaming\FileZilla\recentservers.xml 
```
Contains recently connected FTP servers, usernames, and possibly passwords.
```
%APPDATA%\Roaming\FileZilla\sitemanager.xml
```
Stores saved FTP server credentials.

### 5. Windows Command Prompt FTP
The built-in Windows FTP client does not persist history in the registry, but temporary session logs may exist in:  

```
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters
```

# Credential Access

## Local Hashes & Credentials

### Dump Registry Hives with Impacket 

Check for registry files [here](#locate-reg)

Dump hashes using [Impacket](https://github.com/fortra/impacket)'s `secretsdump.py`

```bash
secretsdump.py -sam path/to/SAM -system path/to/SYSTEM -security path/to/SECURITY LOCAL > hashdump.txt
```

## DPAPI Master Key {#dpapi-masterkey}

### Prerequisites

1. The corresponding user's password.

### Description

You may need to dump the DPAPI Master key in order to access other credentials that you're after. Use [Mimikatz's](https://github.com/ParrotSec/mimikatz) `dpapi` module. Use this [unofficial documentaton](https://adsecurity.org/?page_id=1821) if you need to.

In some cases, you will have to first identify which Master Key GUID to specify. For example, when dumping Credential Manager caches, each one may require a different master key.

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0 small-margin">
        {% include figure.liquid loading="eager" path="assets/img/cs-windows/cred-manager-creds.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
    <div class="col-sm mt-3 mt-md-0 small-margin">
        {% include figure.liquid loading="eager" path="assets/img/cs-windows/master-key-guids.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">
    Multiple credentials cached by Credential Manager (left) and multiple master keys (right)
</div>

To ID which master key GUID corresponds to the credential you want to dump, use Mimikatz's `dpapi::cred` module:

```powershell
dpapi::cred /in:"C:\<path to evidence dir>\C\Users\<user>\AppData\Local\Microsoft\Credentials\<cred>"
```
The GUID will be in the output.

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0 small-margin">
        {% include figure.liquid loading="eager" path="assets/img/cs-windows/mimikatz-cred-id-masterkey.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">
    Identifying a DPAPI Master Key's GUID
</div>

### Dumping the key

Once you know the GUID of the master key you want to dump, use Mimikatz's `dpapi::masterkey` module:

```powershell
dpapi::masterkey /in:"C:\<path to evidence dir>C\Users\<user>\AppData\Roaming\Microsoft\Protect\<user SID>\<master key GUID>" /password:<user password> /protected
```
Here it is with dummy values subbed in:

```powershell
dpapi::masterkey /in:"C:\<path to evidence dir>C\Users\john.d\AppData\Roaming\Microsoft\Protect\S-1-5-21-1281496067-1440983016-2272511217-1000\ac986fb1-8431-4749-bc7b-92ecdf5d7d64" /password:Passw0rd! /protected
```

The master key will be included in the output.

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0 small-margin">
        {% include figure.liquid loading="eager" path="assets/img/cs-windows/mimikatz-masterkey-dump.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">
    Dumping a DPAPI Master Key
</div>

It seems like `/protected` isn't always needed but it doesn't hurt to add when its not needed.


## Windows Credential Manager Cached Credentials

Credential Manager caches credentials in this location:

```plain
C:\Users\<YourUsername>\AppData\Local\Microsoft\Credentials\
```
The credentials are encrypted by DPAPI and can be dumped with Mimikatz.

### Prerequisites

1. The [DPAPI master key](#dpapi-masterkey) corresponding to the credential you want to dump

### Dumping the cached credentials

Use Mimikatz's `dpapi::cred` module to dump the credential file.

```powershell
dpapi::cred /in:"C:\<path to evidence dir>C\Users\john.d\AppData\Local\Microsoft\Credentials\<credential>" /masterkey:<master key>
```

Here is the same command with dummy values:

```powershell
dpapi::cred /in:"C:\<path to evidence dir>C\Users\john.d\AppData\Local\Microsoft\Credentials\063D7EF36287654137F1E552FF79E61E" /masterkey:5902689a5601048b83a7858a842c20d79abff55d82c6d1a35148cc97533760b212d2354057fe3bbdb4d8fdf0ea6fdd1aa79d8bef0101136ebad6ce0eb73e93e8
```

If everything is in order, the credentials should be in the output.

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0 small-margin">
        {% include figure.liquid loading="eager" path="assets/img/cs-windows/mimikatz-cred-dump.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">
    Dumping a credential file from Credential Manager
</div>

# Binary Analysis

There are many types of binaries that require different techniques and tools for analysis. Below are some tools that could help with analysis of various binary types.

## Classifying Binaries

### File Command

Use the `file` command to get a preliminary idea of what type of exe you are dealing with.

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0 small-margin">
        {% include figure.liquid loading="eager" path="assets/img/cs-windows/file-exe.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">
    Revealing a Poertable Executable
</div>

### Detect It Easy

According to their own description [Detect It Easy (DiE)](https://github.com/horsicq/Detect-It-Easy) is a powerful tool for file type identification.

In this case, it didn't detect that this is a .NET Executable, but typically this is a very useful tool.

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0 small-margin">
        {% include figure.liquid loading="eager" path="assets/img/cs-windows/detectiteasy.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">
    Using DiE to analyze a .NET Executable
</div>

### Strings

`strings` is nearly always your friend when it comes to executables. Try dumping all the strings into a file, then grepping it or openning it it a text editor for manual searching.

```bash
strings smphost.exe > smphost.strings
```

```bash
grep -i '\.net' smphost.strings
```
From the output below, we can identify that it was built with **.NET 8.0.4**

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0 small-margin">
        {% include figure.liquid loading="eager" path="assets/img/cs-windows/net8PE.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">
    Using `strings` and `grep` to identify a .NET executable.
</div>

## Decompiling Binaries

This is a massive topic and I am by no means an expert, but here are some useful tools for decompilation.

### Ghidra
Ghidra is a general purpose reverse engineering tool. Get the NSA's Ghidra [here](https://ghidra-sre.org/).

### DotPeek
[DotPeek](https://www.jetbrains.com/decompiler/) is a free decompilation tool from JetBrains that allows easy decompilation of .NET executables.

Look for interesting parts that were written by the authors. Don't get distracted by all the libraries, etc.

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0 small-margin">
        {% include figure.liquid loading="eager" path="assets/img/cs-windows/dotpeek.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">
    Using DotPeek to decompile the important part a .NET executable.
</div>