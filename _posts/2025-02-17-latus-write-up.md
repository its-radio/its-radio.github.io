---
published: true
layout: post
title: HTB Write-Up | Hard Sherlock | Latus
date: 2024-12-09
description: A write-up for Hack The Box's forensics challange 'Latus'
tags: forensics hard sherlock HackTheBox 
categories: Write-Ups
thumbnail: assets/img/2025-02-14/thumb.webp
---

## Introduction

This is a write-up for Hack The Box's Sherlock challenge **Latus**, which is rated as **Hard**. It was relatively difficult, as reflected by the fact that I was only the 42nd person to solve it after it had been out for several months. For context, an easy Sherlock that I did a write-up for had something like 75 solves the day after it came out and now, a few months later, it has over 1000 solves.

That being said, I think this is a great challenge and I would encourage anyone to jump into a challenge like this one. I think diving in a little over your head in digital forensics is a fantastic way to learn and expose yourself to new topics.

For this demo I'm using a one system running Fedora 40 and and one running Windows 11 just depending on the ease of using different tools on different systems.

## Starting Out

### The Scenario

As always with CTFs, lets start by carefully reading the scenario description to see what we are working with.

*"Our customer discovered illegal RDP sessions without Privileged Access Management (PAM) in their system on June 28. They collected evidence on a server they suspected was an intermediary server to move laterally to others. Even though the attacker deleted the event log, I believe the few remaining artifacts are enough to help confirm the attack flow and trace the attacker's behavior."*

What do we learn here?

1. The activity we are interested in was discovered on June 28th, so we should start with evidence that fits that timeframe.
2. The customer suspects that the target server was being used as a pivot host, implying it wasn't the attacker's ultimate objective.
3. The event logs have been deleted, so we will be working with other types of forensic evidence, not logs.

### The Evidence

Now lets take a look at the evidence. If you are having trouble with the basics like downloading and decompressing the evidence, take a look at the beginning of [this other post](/blog/2024/Compromised-Write-Up/) where I describe how to do that.

After decompressing the evidence, we're left with a single file: `evidence.ad1`

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/latus/1.jpg" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>

Full transparency, even after completing the entire CDSA course from HTB and completing a masters degree in cybersecurity, I had not heard of .ad1 files. In fact, I double checked the CDSA course to make sure I hadn't missed it and ".ad1" doesn't come up in a search of the materials on HTB Academy.

Nothing that a quick google search can't fix though. It turns out that .ad1 is an extension used for a proprietary format, similar to a container according to [DIFR Science](https://dfir.science/2021/09/What-is-an-AD1.html). No need to get into the nitty gritty here, but it should suffice to say that it is a type of forensic image and we will need some specific software to be able to interact with it. 

To open `evidence.ad1` we will need to user FTK Imager, which can be found in Exterro's [downloads section](https://www.exterro.com/ftk-product-downloads). It needs to be installed on Windows. You could probably run it on Linux via Wine, but for simplicity's sake, I just booted up a windows system and installed it there.



<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/2025-02-14/18.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>


Thanks for reading! If you have any questions or comments, feel free to reach out or follow [me on X](https://x.com/its_rad_io)

