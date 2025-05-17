---
published: True
layout: post
title: Chrome Browser History Plugin for Volatility 3
date: 2025-05-17
description: A discription of a plugin I wrote for Volatility 3. The plugin searches for, extracts, and parses Google Chrome history databases in forensic memory images.
tags: forensics memory browser chrome volatility
categories: Code
thumbnail: assets/img/chrome_hx_volatility_plugin/vol.jpg
toc:
    sidebar: left
---

<style>
    .small-margin > * {
        margin-bottom: 0.1rem; /* to make image figure titles be a reasonable distance from the image. */
    }
</style>

# Housekeeping

1. Get my chrome\_hx plugin from [my github](https://github.com/its-radio/volatility_plugins/)  
2. For quick plugin installation and usage instructions, see the readme on the github link, above.  
3. Get Volatility3 from the [Volatility Foundation's github](https://github.com/volatilityfoundation/volatility3)

# Introduction to the chrome\_hx plugin

A couple of months ago when I completed the Mellitus Sherlock (and wrote this post about it), I noticed that there wasn’t an easily accessible plugin available for Volatility 3 that could extract Chrome browser history in a single command. Now, the lack of such a plugin isn’t the end of the world. It’s not a difficult process to simply extract files related to a Chrome process, find the SQLite database called ‘History’, and then manually extract the browsing history. However, extracting browser history is a common task, so why not automate it via a plugin? 

I’ve never written a plugin before and since this seems like a fairly simple task, I thought this would be a good introduction to writing plugins for Volatility. In this post I’ll briefly describe how Volatility plugins work, how the plugin I wrote works, and how I might change it in the future.

# background

In searching for a history extraction plugin for Volatility 3, I came across [this post](https://blog.superponible.com/2014/08/31/volatility-plugin-chrome-history/), describing some plugins written for Volatility 2\. However, the post and plugins were written more than 10 years ago and I found that they are no longer functional. The writer had created plugins for extracting History, visits,  search terms, downloads, download chains, and cookies. While I may eventually write all of these (I think it would be fairly easy once I have a skeleton down), I started with just extracting URLs visited from instances of Chrome running when the memory image was taken.

# Goals for the plugin

I’m writing this post in hindsight, having already written a functional history extraction plugin, but I will try to cover my process and explain how the plugin works. First, I’ll go over what the plugin should do:

1. Find and extract the Chrome History Database from a memory image.  
2. Query the database and print the URLs visited to the terminal.  
3. Run on memory images from Windows systems.

# Basic architecture & Implementation

How do we implement that functionality into a Volatility plugin? Well, since I had never written a plugin for Volatility, for anything for that matter, I thought I should go to the source for some education. The Volatility foundation provides some [excellent documentation](https://volatility3.readthedocs.io/en/stable/development.html) describing how to get started with writing plugins for the framework. The sections on [writing plugins that run other plugins](https://volatility3.readthedocs.io/en/stable/complex-plugin.html#writing-plugins-that-run-other-plugins) and [writing plugins that output files](https://volatility3.readthedocs.io/en/stable/complex-plugin.html#writing-plugins-that-output-files) proved to be especially useful to me.

# How a Volatility plugin works

Read the link documentation for a more in depth description, but at a high level, a Volatility plugin consists of the following general parts:

1. **The plugin class**  
   Wrapping everything else in the plugin is a class named for the plugin (i.e. with my plugin being called with windows.chrome\_hx, I named this class Chrome\_Hx), that inherits from PluginInterface (a class that acts as an interface to the framework and signals that this is a plugin that can be used by the framework).  
2. **The requirements**  
   A list of required input variables like PID, memory addresses, etc. These requirements can then be provided by the framework if available. If other plugins are going to be used, they should also be listed here as requirements.  
3. **The run() method**  
   When the framework tries to make use of a plugin, it does so by importing it and calling its run() method, so this method functions kind of like a main() normally would. Additionally, the run() method is required to return a TreeGrid object. Whatever is in the TreeGrid object is what will actually get printed to the terminal as output from the plugin.  
4. **The \_generator() method**  
   Though not technically required, it is common to write a method called \_generator() that is used to populate the TreeGrid object that is returned by run(). This takes much of the heavy lifting out of run(), simplifying the population process.

And that’s it\! Just to briefly review, as long as the plugin class inherits from PluginInterface it gets recognized and imported by the framework. The requirements are checked and provided to the plugin. The run() method is run, which should return a TreeGrid object that is populated by the \_generator() method.

# How my plugin works

I won’t go over my entire development process, but I’ll try to give an overview of how it works and where it came from. If you want to follow along in my code, you can find it from [my github](https://github.com/its-radio/volatility_plugins/). In essence, all my plugin had to do is:

1. Find a specific process  
2. Find a specific file related to the process  
3. Dump that file  
4. Read that file  
5. Output some specific contents

Given that there are already plugins that find processes (`pslist`, `psscan`, etc.) and dump files (`dumpfiles`), I knew that I could borrow quite a bit of functionality from them. Given that my plugin needs to dump a file, I took a fair bit of inspiration from the `dumpfiles` plugin. In fact, my first proof of concept was to just take the `dumpfiles` plugin and add functionality to look for and parse the contents of SQLite databases that it outputs. After that worked, I knew I was on track, but I didn’t want to just copy the entire plugin since doing that would repeat a bunch of existing code. Then, I wrote a new plugin that included some code directly from `dumpfiles` and used methods from `pslist`, `handles`, and `dumpfiles`.

# The class

My plugin class is called `Chrome\_Hx`. It, of course, inherits from `PluginInterface`.

# The requirements

1. **Kernel**  
   This is required in basically every plugin. Since this is a windows plugin, it just gets assigned as Windows Kernel for Intel 32 & 64 bit architectures  
2. **PID**  
   I opted to have PID be the typical way that Chrome instances would be identified for extraction. If this requirement is omitted, the plugin still works, but it takes longer since it has to look for Chrome history DBs in every PID, even if it's not associated with Chrome.  
3. **Filter**  
   This requirement tells the plugin what pattern to look for in a filename that it is going to dump. This is the main method the plugin uses to identify Chrome history databases. It defaults to searching for a regex `Chrome.\*History$`. This regex will match filenames whose path consists of anything, then “Chrome”, then anything, and then end with “History”.  
   In my experience, this is a consistent naming convention for Chrome history databases. However, if the default regex doesn’t work, the user can specify a different regex for a specific situation.  
4. **Ignore-case**  
   This is a modifier for the Filter requirement. If the user needs to specify a different regex, this allows them to toggle case sensitivity on and off as desired.  
5. **`pslist` plugin**  
   The `pslist` plugin, used to filter for files with specific PIDs  
6. **`dumpfiles` plugin**  
   The `dumpfiles` plugin, used to process files found by windows.chrome\_hx  
7. **`handles` plugin**  
   Used to analyze and use the handles pointing to objects in use by the process(es) found by `pslist`.

# The run() method

The run method mainly does three things. First, it uses the `PsList` class to create a filter function to filter for processes of the PID specified by the user. Second, it uses the `PsList` class to create a generator object, procs, representing the various process objects that correspond to the process(es) matching the filter function. Third (and finally) it returns a TreeGrid while calling the `\_generator()` method (with `procs` as the only argument passed) to do the rest of the work and populate the TreeGrid.

# The \_generator method

This is where most of the functionality of the plugin is implemented, meaning it is the most important and most complex part (though it remains fairly straightforward). Here is a simplified explanation of what it does: For each process passed from `run()`, it generates a list of all the handles that process was using.   
Since handles are just a way for the computer to keep track of other files, processes, or anything else that the process has to communicate with, the plugin also needs to filter out any handles that aren’t to files, since we are only interested in files. Next the plugin applies the filter value that was previously specified as the default to search for Chrome history databases. It checks if it has previously dumped this file and, if it hasn’t, it adds an entry to a set to keep track of files it has dumped. Then it dumps the file’s contents and adds the name the file was written as to a list.

After all the filtered files have been dumped, the method goes through the list of names of dumped files. It checks if the files are SQLite databases by checking their magic bytes, then tries to use the `sqlitle3` python module to dump the contents of the `url` table. The resultant data from the table is formatted for the TreeGrid and then yielded back to be output by the framework. The portion that tries to do the SQLite work is wrapped in try-except syntax because there are often two databases that get dumped: One ending in `.vacb` (virtual address control block) and one ending in `.dat`. These are really the same database, but there are some important differences between them and they don’t always play nicely with SQLite after being dumped. The `.dat` file is the actual database that Chrome reads from the filesystem, while the `.vacb` one is the working version of the history that Chrome is updating in memory in real time, meaning it could have more up-to-date information. However, I have encountered cases in which one is malformed while the other is intact, so it's worthwhile to try to dump both of them.

**Note:** If they are both malformed for some reason, you can use these commands with [sqlite3](https://sqlite.org/index.html) installed on a Linux system to try to repair one or both of them:

```bash
sqlite3 \<malformed\_db\>.dat ".recover" \> salvaged.sql  \# parse recoverable portions of the db  
sqlite3 new\_fixed.db \< salvaged.sql  \# reconstruct a new database with the recovered portions  
sqlite3 new\_fixed.db  \# try to open the db in sqlite3 again  
```

# Output

This plugin outputs the URLs table from the Chrome history database as text, but it also outputs the files that came up as matches during its search. These files can be analyzed manually with any sqlite parser like `sqlite3` or `sqlitebrowser`, and they contain quite a bit more than this plugin actually prints to the terminal.

# Limitations

1. **I’m new\!**  
   I am rather new to writing plugins for Volatility. Even though I tried my best to stick to their recommendations and standards, it's possible that I fell short in these regards. If anyone more familiar with Volatility plugins has feedback, I would welcome it.  
2. **Lack of samples**  
   I tested this on several memory images collected from Windows 10 machines that I own, as well as a few from CTFs or Sherlocks from Hack The Box. All things considered, this isn’t a very large sample size so I can’t be sure how well it will work generalized across any image collected from a Windows machine. If you have any issues, please create an issue on github and ideally send me some output with `-vv` enabled.  
3. **Simple filtering approach**  
   The filtering approach I chose just tries to match the regex `Chrome.\*History$`. That could be too simple and might end up matching files that aren’t actually Chrome history databases. I thought about checking if the file was an SQLite before dumping to add an extra layer of filtering, but that would have meant messing around with the methods in `dumpfiles`, which I didn’t want to do.

# Future improvement

Since this is my first foray into writing a plugin, I've kept it fairly basic. However, in the future there are a few directions I might take this project:

1. Expand to work on Mac OS and Linux memory images  
2. Extract more types of information (e.g. downloads, search terms, etc.) based on user prompting  
3. Extract this type of data from different browsers other than Chrome  
4. Search for any Chrome instances to extract files from automatically without the user having to enter a PID  
5. Do more to validate extracted files as SQLite databases  
6. Add recovery to try to repair databases if they are dumped but malformed

# Conclusion

I created this plugin primarily as an educational project, but also in the hopes that someone else might find it useful. It's not perfect and it's not hard to access the database, but again, why not automate what we can? I learned a lot about Volatility during this project and I look forward to writing more plugins in the future.