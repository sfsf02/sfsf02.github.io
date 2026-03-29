---
layout: post
title: "TryHackMe: Pickle Rick"
date: 2026-03-29 09:48:14 +0200
categories:
  - CTF
  - TryHackMe
tags:
  - tryhackme
  - gobuster
  - privilege-escalation
  - gtfobins
  - linux
---

Welcome back, fellow hackers! Today, we’re tackling a super fun machine based on one of my absolute favorite shows. It has a heavy *Rick and Morty* theme, and if you haven't seen the iconic episode where Rick turns himself into a pickle... drop everything and go watch it! 

Alright, now that you're caught up, let's dive in. 

### The Mission Briefing

First, let's look at the description of the machine:

> "This Rick and Morty-themed challenge requires you to exploit a web server and find three ingredients to help Rick make his potion and transform himself back into a human from a pickle."

So, Rick has turned himself into a pickle *again* and needs our help to revert back. To do this, we need to hunt down three specific ingredients hidden across the system. 

Since we know it's a web server, let's open up our very first, highly advanced hacker tool (duh, obviously the browser!).

![web server's homepage](https://safsecmedia.blob.core.windows.net/images/Pickle%20Rick/homePageRick.png){: .shadow }

Ohhhh, Rick really did turn himself into a picccccccccccccccccccccccckle again!

### Step 1: Basic Reconnaissance 

As a hacker, you don't need to overcomplicate things right out of the gate. Let's start with the basics and check out the page's source code. You never know what developers (or mad scientists) leave behind in the comments.

![source code revealing the username](https://safsecmedia.blob.core.windows.net/images/Pickle%20Rick/sourceCode.png){: .shadow }

Naughty Rick! We found a hidden username. Ohhhh, this is going to be good.

### Step 2: Brute-Forcing Directories

Let's fire up our second tool: `gobuster`. We need to crawl the web server and do some technical directory brute-forcing. 

I'm using the `-t` flag to increase the speed (threads) and the `-x` flag to specifically look for files with `.php`, `.js`, `.txt`, and `.html` extensions. Why are we looking for these? Because we found a username, which means there's probably a login page somewhere!

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt -t 50 -x php,js,txt,html
```

![Gobuster terminal results](https://safsecmedia.blob.core.windows.net/images/Pickle%20Rick/GobusterRsults.png){: .shadow }

Oohh, look at this treasure! We found two yummy files: `login.php` and `robots.txt`!

### Step 3: Gaining Access

Now that the login page has been found, we need a password. 

I took a look at the `robots.txt` file and found a really strange word just sitting there. I figured, why not try it as the password? 

![robots.txt file contents](https://safsecmedia.blob.core.windows.net/images/Pickle%20Rick/rebotsTxt.png){: .shadow }

I plugged the username we found in the source code and the weird word from `robots.txt` into the login page... and *voilà!* We are in, baby!

![successful login/command panel](https://safsecmedia.blob.core.windows.net/images/Pickle%20Rick/commandPanel.png){: .shadow }

### Step 4: Hunting for Ingredients

As you can see, we have access to a command panel. Let's dive right in, list the files, and see if we can spot the ingredients we need.

```bash
ls -la
```

![issuing the `ls -la` command](https://safsecmedia.blob.core.windows.net/images/Pickle%20Rick/Ls-laCommnadResult.png){: .shadow }

Ohhhh, we found the first ingredient file! Let's read it using the standard `cat` command:

```bash
cat Sup3rS3cretPickl3Ingred.txt
```

![issuing the `cat` command and failing](https://safsecmedia.blob.core.windows.net/images/Pickle%20Rick/CommandDisable.png){: .shadow }

Da*****n it, Rick! He disabled the `cat` command. Ugh.

### Step 5: Bypassing Restrictions with GTFOBins

Time to go to our third tool. I don't know if I'd strictly call it a "hacking tool," but I use it *a lot*. It's the fantastic website [GTFOBins](https://gtfobins.github.io/). It saves me so much time when I need to figure out how to bypass local security restrictions using standard Linux binaries.

![GTFOBins Website](https://safsecmedia.blob.core.windows.net/images/Pickle%20Rick/GTFOBins.png){: .shadow }

Let's see what other binaries can give us file-reading abilities. Fortunately, the `less` command is available, which works perfectly for reading files. (There are other binaries you could try too, like `more`, `tail`, or `head`, and plenty of others!!).

```bash
less Sup3rS3cretPickl3Ingred.txt
```

![using `less` to read the first ingredient](https://safsecmedia.blob.core.windows.net/images/Pickle%20Rick/FirstIncredient.png){: .shadow }

Awesome, we got the first one! 

Let's look around for the other two ingredients. If we navigate over to Rick's home folder, we strike gold again. Yep, we got the second one!

![finding and reading the second ingredient in Rick's home folder](https://safsecmedia.blob.core.windows.net/images/Pickle%20Rick/SecondIncredient.png){: .shadow }

### Step 6: The Final Ingredient (Privilege Escalation)

Okay, now we only have one left. I was looking around the file system, taking a bit too much time, until I remembered something crucial: the `root` user directory.

![Root Directory Permission](https://safsecmedia.blob.core.windows.net/images/Pickle%20Rick/rootPermission.png){: .shadow }

As you can see, we need to be `root`. *\*Burrrp\** It's never easy. 

Let's try the easiest privilege escalation check in the book: the `sudo -l` command. This will tell us what binaries our current user is allowed to run with root privileges.

```bash
sudo -l
```

![the `sudo -l` results](https://safsecmedia.blob.core.windows.net/images/Pickle%20Rick/Sudo-lResult.png){: .shadow }

Ahhhhh! Look at that. We can run *everything* as root without even needing a password! Let's elevate our privileges, dive into that `/root` directory, and grab that final ingredient.

```bash
sudo ls -la /root
sudo less /root/3rd.txt
```

![the 3rd ingredient in the root directory](https://safsecmedia.blob.core.windows.net/images/Pickle%20Rick/findign3rdIncre.png){: .shadow }

Woooh! We found our 3rd and final ingredient. Let's get it.

![he final ingredient](https://safsecmedia.blob.core.windows.net/images/Pickle%20Rick/3rdIncredient.png){: .shadow }

Mission accomplished. Rick is human again. 

Salute, *\*Burrrrp\** safsec

---

### 🛠️ Command Cheat Sheet & Tool Summary

For my fellow students and CTF players, here is a quick wrap-up of the tools and commands used to crack this box:

* **View Page Source:** `Ctrl + U` (in most browsers) - *Always check the source code for hidden comments or credentials!*
* **Gobuster:** `gobuster dir -u http://<TARGET_IP> -w <WORDLIST> -t 50 -x php,js,txt,html` - *Used for fast directory and file brute-forcing.*
* **List Files:** `ls -la` - *Lists all files in a directory, including hidden ones.*
* **Read Files (Cat alternative):** `less <filename>` - *Used to read file contents when `cat` is blocked.*
* **GTFOBins:** [gtfobins.github.io](https://gtfobins.github.io/) - *The ultimate cheat sheet for bypassing local security restrictions using misconfigured binaries.*
* **Check Sudo Permissions:** `sudo -l` - *Lists the commands the current user is allowed to run as root.*
* **Execute as Root:** `sudo <command>` - *Runs the specified command with elevated privileges.*
