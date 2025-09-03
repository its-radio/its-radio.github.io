---
published: True
layout: post
title: HTB Write-Up | medium Sherlock | Heartbreaker
date: 2025-09-03
description: A write-up for Hack The Box's forensics challenge 'Heartbreaker'. This challenge focuses primarily on analysis of an evidence dump from a machine that was compromised following an employee opening a malicious email.
tags: forensics medium sherlock HackTheBox dfir windows email
categories: Write-Ups
thumbnail: assets/img/heartbreaker/thumb.png
---

<style>
    .small-margin > * {
        margin-bottom: 0.1rem; /* to make image figure titles be a reasonable distance from the image. */
    }
</style>

## Tools & Setup

For this investigation, I used a machine running Fedora 42, meaning that I had access to all the standard Linux tools like `grep`, `strings`, etc. Of course, there are Windows equivalents for most of these tools, and it is possible to repeat any of these steps on Windows.

## Starting Out

I always start by thoroughly reading the description. It reads:

*Delicate situation alert! The customer has just been alerted about concerning reports indicating a potential breach of their database, with information allegedly being circulated on the darknet market. As the Incident Responder, it's your responsibility to get to the bottom of it. Your task is to conduct an investigation into an email received by one of their employees, comprehending the implications, and uncovering any possible connections to the data breach. Focus on examining the artifacts provided by the customer to identify significant events that have occurred on the victim's workstation.*

**Important points from the description:**

1. There has been a data breach, with sensitive information being sold on the darknet.
2. A potentially malicious email is involved, possibly kicking off the incident.
3. We will be focusing on the workstation of the employee who received the malicious email.

## Download ultimatum.zip and decompress it

As usual, I used 7z to unzip the archive and enter the password `hacktheblue`.

```bash
7z x HeartBreaker.zip
```
Looking at the C directory, this is some kind of forensic evidence gathered from a Windows system. At this point, I'm not sure if it is the entire filesystem or just certain files selected by the evidence gathering program. 

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0 small-margin">
        {% include figure.liquid loading="eager" path="assets/img/heartbreaker/1.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">
    Figure 1
</div>

No matter though, it's time to dive into the tasks.

## Task 1: The victim received an email from an unidentified sender. What email address was used for the suspicious email?

It looks like the user on this system, who is presumably the victim employee, is ash.williams. They have offline Outlook messages stored in `wb-ws-01/C/Users/ash.williams/AppData/Local/Microsoft/Outlook/ashwilliams012100@gmail.com.ost`, which I found with the `find` command.

```bash
find wb-ws-01/C/Users/ash.williams/AppData -iname "*mail*"
```
<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0 small-margin">
        {% include figure.liquid loading="eager" path="assets/img/heartbreaker/2.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">
    Figure 2
</div>

I can't directly view the `.ost` file, so I used `readpst` from `libpst` to convert the mail to mbox format.

```bash
mkdir mail_output_ash

readpst -r -o mail_output_ash wb-ws-01/C/Users/ash.williams/AppData/Local/Microsoft/Outlook/ashwilliams012100@gmail.com.ost
```

Now I can just open the various mbox files in a text editor. Opening the inbox file (`mail_output_ash/ashwilliams012100@gmail.com.ost/Inbox/mbox`) in a text editor, my attention was drawn to a message that seems quite suspicious. It claims to be from a secret admirer and urges the recipient to download an executable file. It is from `ImSecretlyYours@proton.me` (Figure 3).

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0 small-margin">
        {% include figure.liquid loading="eager" path="assets/img/heartbreaker/3.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">
    Figure 3
</div>

## Task 2: It appears there's a link within the email. Can you provide the complete URL where the malicious binary file was hosted?

See Figure 3. The link is included near the bottom in an href.

## Task 3: The threat actor managed to identify the victim's AWS credentials. From which file type did the threat actor extract these credentials?

This one took me much longer than I'd care to admit. I was looking in all the wrong places, but I'll quickly go over some of what I did that failed for educational purposes.

I looked around for some common places that AWS credentials might be stored, but found nothing. Here are some places where there wasn't evidence of AWS credentials:

**Files:**
- `%UserProfile%\.aws\credentials`
- `%UserProfile%\.aws\config`
- `%UserProfile%\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt`
- `.ps1` files
- `.env` files

**Logs:**
- No evidence of malicious access to `aws.exe`

**Registry:**
- No evidence of AWS credentials in `HKCU\Environment` or `HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Environment`

Having failed so far, I went back to reread the task question and noticed that Task 4 seems to be related to this one and mentions the "actual IAM credentials", referring to the AWS credentials we are searching for in Task 3. Based on this, I grepped for "IAM" and found a suspicious result (Figure 4).

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0 small-margin">
        {% include figure.liquid loading="eager" path="assets/img/heartbreaker/IAM_subject.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">
    Figure 4
</div>

Looking at that file in a text editor, I could see the AWS credentials (Figure 5).

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0 small-margin">
        {% include figure.liquid loading="eager" path="assets/img/heartbreaker/IAM_key.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">
    Figure 5
</div>

Though the file where I found the credentials was `mail_output_ash/ashwilliams012100@gmail.com.ost/\[Gmail\]/Drafts/mbox`, and has no extension, I previously extracted it from the `.ost` file, making that the answer.

## Task 4: Provide the actual IAM credentials of the victim found within the artifacts.

See Task 3, Figure 5. Make sure to format the credentials correctly.

## Task 5: When (UTC) was the malicious binary activated on the victim's workstation?
First, I used `chainsaw` to do a rough query for logs associated with the malicious binary (whose name was earlier observed in the malicious link in the email) to start getting a grasp on the execution timeline. The malicious binary is called `Superstar_MemberCard.tiff.exe`.

```bash
chainsaw search --timestamp 'Event.System.TimeCreated_attributes.SystemTime' --from "2024-03-13T00:00:00" --skip-errors 'Superstar_MemberCard.tiff.exe' wb-ws-01/C/Windows/System32/winevt
```
There were 128 results of logs related to the malicious binary. I used `grep`, `cut`, and `uniq` to parse the results and found that they are mostly clustered around 2024-03-13 10:44 to 10:48 with another few at 14:00 on the same day.

```bash
chainsaw search --timestamp 'Event.System.TimeCreated_attributes.SystemTime' --from "2024-03-13T00:00:00" --skip-errors 'Superstar_MemberCard.tiff.exe' wb-ws-01/C/Windows/System32/winevt | grep -i systemtime | cut -d ' ' -f 8 | cut -d '.' -f 1 | cut -d 'T' -f 2 | cut -d ':' -f 1,2 | uniq
```

Next, I used Chainsaw to bring up Sysmon logs in a tight window around the time the attack started, then found the first log with `Image: C:\Users\ash.williams\Downloads\Superstar_MemberCard.tiff.exe` and Sysmon EventID 1 (Process Creation) as shown in Figure 6.

```bash
chainsaw search --timestamp 'Event.System.TimeCreated_attributes.SystemTime' --from "2024-03-13T10:45:00" --to "2024-03-13T10:46:00" --skip-errors -t "Event.System.EventID: =1" wb-ws-01/C/Windows/System32/winevt/logs/Microsoft-Windows-Sysmon%4Operational.evtx
```

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0 small-margin">
        {% include figure.liquid loading="eager" path="assets/img/heartbreaker/binary_timestamp.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">
    Figure 5
</div>

## Task 6: Following the download and execution of the binary file, the victim attempted to search  for specific keywords on the internet. What were those keywords?

To find these keywords, I need to look through the user's internet history. They have a few browsers on their system, but since Firefox is the only one that would need to be intentionally installed on a Windows system, I think that's a good place to start. I used `sqlitebrowser` to open Firefox's history file.

```bash
sqlitebrowser ./wb-ws-01/C/Users/ash.williams/AppData/Roaming/Mozilla/Firefox/Profiles/hy42b1gc.default-release/places.sqlite
```

Then I opened the 'moz_places' table and found the keywords (Figure 7)

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0 small-margin">
        {% include figure.liquid loading="eager" path="assets/img/heartbreaker/keywords.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">
    Figure 7
</div>


## Task 7: At what time (UTC) did the binary successfully send an identical  malicious email from the victim's machine to all the contacts?

I first tried looking in the `Outbox` file (`mail_output_ash/ashwilliams012100@gmail.com.ost/Outbox/mbox`), but it was empty. It turns out that sent mail is not stored there. This made sense after I read about it, but to my young millennial mind, Outbox initially sounded like the opposite of Inbox.

Anyway, the sent mail is actually stored in a different file (`mail_output_ash/ashwilliams012100@gmail.com.ost/\[Gmail\]/Sent\ Mail`). I opened that file in a text editor and found the malicious email by searching for the malicious IP found in the link from Task 2 (Figure 8).

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0 small-margin">
        {% include figure.liquid loading="eager" path="assets/img/heartbreaker/email_to_contacts.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">
    Figure 8
</div>


## Task 8: How many recipients were targeted by the distribution of the said email excluding the victim's email account?

Ash.Williams' contacts are saved in the `mail_output_ash` directory that I made in Task 1. Each contact starts with the word "BEGIN", so I used that with `grep` to isolate the number of contacts and count them with `wc`.

```bash
cat mail_output_ash/ashwilliams012100@gmail.com.ost/Contacts\ \(This\ computer\ only\)/contacts | grep -i begin | wc -l
```
Alternatively, you could count the email addresses listed in the malicious email from Task 7.

## Task 9: Which legitimate program was utilized to obtain details regarding the domain controller?

Here I was interested in finding out what processes were accessed by the malicious binary. To do this, I used `chainsaw` to query the logs for Sysmon EventID 10, which indicates process access. There were only 16 results, so I could easily parse through them manually, looking for logs where the `SourceImage` is the malicious binary and the `TargetImage` is a legitimate program that can get details about the DC. I found a log that fit all the criteria (Figure 9).

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0 small-margin">
        {% include figure.liquid loading="eager" path="assets/img/heartbreaker/DC_info_access.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">
    Figure 9
</div>

`nltest.exe`, or [Net Logon Test](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/cc731935(v=ws.11)), is a tool that can, among other things, get a list of Domain Controllers and information about trust relationships within the domain.

## Task 10: Specify the domain (including sub-domain if applicable) that was used to download the tool for exfiltration.

I looked through some Sysmon EventID 11 (FileCreate) logs and found that the malicious binary writes a file `WinSCP.exe`, indicating that they downloaded [WinSCP](https://winscp.net/eng/index.php), which could be used for exfiltration (Figure 10).

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0 small-margin">
        {% include figure.liquid loading="eager" path="assets/img/heartbreaker/exfil_tool_creation.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">
    Figure 10
</div>

Using the file's creation time, I created a `chainsaw` query that searched for Sysmon DNS request logs (EventID 22) in a tight 10-second window around the `WinSCP.exe`'s creation time.

```bash
chainsaw search --timestamp 'Event.System.TimeCreated_attributes.SystemTime' --from "2024-03-13T10:45:20" --to "2024-03-13T10:45:30" --skip-errors -t "Event.System.EventID: =22" wb-ws-01/C/Windows/System32/winevt/logs/
```
Sure enough, the malicious binary made a DNS request to `us.softradar.com` just 2 seconds before it created the `WinSCP.exe` (Figure 11).

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0 small-margin">
        {% include figure.liquid loading="eager" path="assets/img/heartbreaker/exfil_tool_domain.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">
    Figure 11
</div>

## Task 11: The threat actor attempted to conceal the tool to elude suspicion. Can you specify the name of the folder used to store and hide the file transfer program?

I already found `WinSCP.exe` in Task 10. Its parent folder is what the question is asking for here.

## Task 12: Under which MITRE ATT&CK technique does the action described in question #11 fall?

The full path of `WinSCP.exe` is `C:\Users\Public\HelpDesk-Tools\WinSCP.exe`, meaning the exfiltration tool is being concealed in `C:\Users\Public\HelpDesk-Tools` to elude suspicion. In this case, the attacker isn't actually doing anything technical to hide their tools except naming them such that they could feasibly blend into a corporate environment. That is, the tool is not called `exfil.exe` and stored in `C:\Users\Public\Hacking-Tools`. This doesn't change or substantively hide the tools.

Certainly this would fall under Defense Evasion, and looking through the techniques at [MITRE's website](https://attack.mitre.org/tactics/TA0005/), it looks like it matches [Masquerading](https://attack.mitre.org/techniques/T1036/) well.

## Task 13: Can you determine the minimum number of files that were compressed before they were extracted?
I didn't really know where to start this one. Is that info stored in logs? Could I check the access times of the files via Eric Zimmerman's Timeline Explorer?

The question implies the existence of a compressed archive, so I thought I'd start by trying to find the archive that the files were compressed into to see if I could pivot from any information there. I used this `chainsaw` query:

```bash
chainsaw search --timestamp 'Event.System.TimeCreated_attributes.SystemTime' --from "2024-03-13T10:44:00" --to "2024-03-13T14:00:00" --skip-errors -t "Event.System.EventID: =11" wb-ws-01/C/Windows/System32/winevt/logs/ | grep -iC 25 zip
```

This only shows two zip files being created. One is the download of `WinSCP.zip`, and the other is probably what I am interested in: `WB-WS-01.zip` created by the malicious binary just before the network connection from the exfil tool (Figure 12).

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0 small-margin">
        {% include figure.liquid loading="eager" path="assets/img/heartbreaker/archive_creation.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">
    Figure 12
</div>

Naturally, the files compressed into this archive must have been accessed or gathered before the archive was created, so I queried again for Sysmon logs in the 19 seconds leading up to the creation of the archive. I also grepped for EventID to isolate the EventIDs and make it easier to glance through for a high-level overview of the events in those 19 seconds. What caught my eye here were all the EventID 11s. These are all files being created, possibly copied into the same directory before being compressed? (Figure 13)

```bash
chainsaw search --timestamp 'Event.System.TimeCreated_attributes.SystemTime' --from "2024-03-13T10:45:00" --to "2024-03-13T10:45:19" --skip-errors "" wb-ws-01/C/Windows/System32/winevt/logs/Microsoft-Windows-Sysmon%4Operational.evtx | grep EventID
```
<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0 small-margin">
        {% include figure.liquid loading="eager" path="assets/img/heartbreaker/archive_event_11s.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">
    Figure 13
</div>

Next I took a look at all the target filenames of the files being created with:

```bash
chainsaw search --timestamp 'Event.System.TimeCreated_attributes.SystemTime' --from "2024-03-13T10:45:00" --to "2024-03-13T10:45:19" --skip-errors -t "Event.System.EventID: =11" wb-ws-01/C/Windows/System32/winevt/logs/Microsoft-Windows-Sysmon%4Operational.evtx | grep -i targetfilename
```

Many of these look like files that an attacker would want to exfiltrate for later use (Figure 14).

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0 small-margin">
        {% include figure.liquid loading="eager" path="assets/img/heartbreaker/archive_files_for_exfil.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">
    Figure 14
</div>

Assuming these are the files the question is referring to, let's count them up with more `grep` and `wc`:

```bash
chainsaw search --timestamp 'Event.System.TimeCreated_attributes.SystemTime' --from "2024-03-13T10:45:00" --to "2024-03-13T10:45:19" --skip-errors -t "Event.System.EventID: =11" wb-ws-01/C/Windows/System32/winevt/logs/Microsoft-Windows-Sysmon%4Operational.evtx | grep -i targetfilename | grep "Public Files" | grep "\." | grep -v zip | wc -l
```
And the command output reported **26** results (excluding the directory itself and the archive), which was indeed the answer.


## Task 14: To exfiltrate data from the victim's workstation, the binary executed a command. Can you provide the complete command used for this action?

Since the question is asking about exfiltration, I suspect that the command executed by the malicious binary will include an invocation of the exfiltration tool `WinSCP.exe`, so I will query for that with `chainsaw` and filter for instances that include a `CommandLine` field. Only 4 lines are returned (Figure 15).

```bash
chainsaw search --timestamp 'Event.System.TimeCreated_attributes.SystemTime' --from "2024-03-13T10:45:00" --to "2024-03-13T13:00:00" --skip-errors 'WinSCP' wb-ws-01/C/Windows/System32/winevt/logs/ | grep -i commandline
```

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0 small-margin">
        {% include figure.liquid loading="eager" path="assets/img/heartbreaker/exfil_command_line.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">
    Figure 15
</div>

One of these CommandLines has the malicious binary as its ParentCommandLine, meaning that it was executed by the malicious binary.

## Conclusion

I enjoyed many aspects of this Sherlock. I liked the realism of the premise, the wide range of tasks that were assigned, and the simplicity of the attack. This is classic attack if there ever was one. It was nice to be able to complete so much of the investigation without resorting to many security tools. `Chainsaw` was really the only specialized tool that I had to use, and the rest was good-old linux terminal tools. Very fun! There is another Sherlock that seems to be a sequel of sorts to this one, *Heartbreaker-Continuum*, in which we can take a closer look at the malware from this challenge. Look out for that write-up coming up soon!

Thanks for reading! If you have any questions or comments, feel free to reach out or follow [me on X](https://x.com/its_rad_io)