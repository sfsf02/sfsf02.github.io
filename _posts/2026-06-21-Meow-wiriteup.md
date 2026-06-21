---

layout: post
title: "HTB Starting Point: Purr-fecting the Basics with Meow"
date: 2026-06-21 15:21:25 +0200
categories: [CTF , Hack The Box]
tags: [HTB, Starting Point, Beginner, Nmap, Telnet, Reconnaissance]
excerpt: "A beginner-friendly walkthrough of Hack The Box's Meow machine, covering basic enumeration, Telnet access, and the core cybersecurity concepts behind each step."
---

## Introduction

Every cybersecurity journey starts with the basics, and Hack The Box’s **Starting Point** machines are designed exactly for that. They don’t just throw you at a vulnerable target and say *“good luck”* — they teach the habits, terminology, and thought process behind real-world enumeration and exploitation.

In this write-up, I’ll walk through **Meow**, one of the first machines in the Starting Point path. It’s a very beginner-friendly box, but it still introduces several core ideas you’ll use over and over again in penetration testing:

* working inside a **virtual machine**
* using the **terminal** comfortably
* connecting to the lab with **OpenVPN**
* verifying connectivity with **ping**
* scanning services with **Nmap**
* identifying insecure remote access via **Telnet**
* and finally logging in to capture the flag

Rather than just listing the answers, I want to explain **why each question matters** and how it fits into a pentester’s workflow.

---

## Box Information

| Category   | Value                                           |
| ---------- | ----------------------------------------------- |
| Platform   | Hack The Box                                    |
| Path       | Starting Point                                  |
| Machine    | Meow                                            |
| Difficulty | Very Easy                                       |
| Focus      | Basic Linux access, service enumeration, Telnet |

---

## Step 1: Understanding the Environment

Before touching the target machine, HTB starts with a few foundational questions. These may feel simple, but they’re there for a reason: they reinforce the environment and tools you’ll be using throughout the rest of the platform.

---

## The Safety Net: What is a VM?

### Question 1

**In cybersecurity, isolated environments—like Pwnbox or the vulnerable target machines—are often VMs. What does VM stand for?**

**Answer:** `Virtual Machine`

Virtual Machines are one of the most important building blocks in cybersecurity. A VM gives you an isolated environment where you can safely test tools, inspect malware, break things, and make mistakes without destroying your main operating system.

For pentesting and CTF work, that isolation matters a lot. You want a lab where you can freely experiment, run scans, connect to targets, and even crash a box without worrying about your real machine.

> **Why this matters:**
> In security, we intentionally interact with unstable, vulnerable, or even malicious systems. VMs give us a safe playground to do that.

---

## Our Digital Home: The Command Line

### Question 2

**What tool do we use to interact with the operating system in order to issue commands via the command line, such as the one to start our VPN connection? It's also known as a console or shell.**

**Answer:** `Terminal`

If you want to work in cybersecurity, the terminal becomes home. Most of the tools used for enumeration, exploitation, scripting, and automation are command-line based, so being comfortable in the shell is not optional — it’s essential.

The terminal is where you’ll launch scans, inspect files, connect to services, transfer payloads, and troubleshoot issues. The earlier you get comfortable with it, the better.

> **Takeaway:**
> Graphical interfaces are nice, but the terminal is where most of the real work happens.

---

## Step 2: Connecting to the HTB Lab

## Tunneling In: Connecting to Hack The Box

### Question 3

**What service do we use to form our VPN connection into HTB labs?**

**Answer:** `openvpn`

Hack The Box labs are accessed through a VPN connection, and **OpenVPN** is the tool used to establish that tunnel. Without it, your machine won’t be able to communicate with the target box inside the HTB environment.

In practice, this means importing your `.ovpn` configuration file and connecting from the terminal.

```bash
sudo openvpn <your_config_file.ovpn>
```

![OpenVPN successfully connecting to the Hack The Box lab network.](https://safsecmedia.blob.core.windows.net/images/HTB/Meow/openVpnConnection.png){: .shadow }
*OpenVPN successfully connecting to the Hack The Box lab network.*

> **Why this matters:**
> A VPN creates a secure tunnel between your machine and the lab network. In this case, it allows your traffic to reach HTB machines that aren’t exposed to the public internet.

---

## Step 3: Confirming the Target is Reachable

## Knock, Knock: Testing Connectivity

### Question 4

**What tool do we use to test our connection to the target with an ICMP echo request?**

**Answer:** `ping`

Before scanning or attacking anything, it’s worth checking whether the target is reachable at all. The quickest way to do that is with `ping`, which sends an ICMP echo request and waits for a reply.

```bash
ping <Target_IP_Address>
```

This doesn’t always guarantee a machine is offline if it doesn’t respond — many hosts block ICMP — but it’s still a fast and useful first check.

> **Tip:**
> If `ping` fails, don’t immediately assume the target is down. Firewalls often block ICMP while still allowing services like SSH, HTTP, or Telnet.

---

## Step 4: Enumeration with Nmap

## The Map Maker: Scanning for Open Doors

### Question 5

**What is the name of the most common tool for finding open ports on a target?**

**Answer:** `nmap`

If there’s one tool you absolutely need to know in early pentesting, it’s **Nmap**.

Nmap is used to discover hosts, identify open ports, detect services, fingerprint operating systems, and gather valuable reconnaissance data before exploitation. In many ways, it’s one of the first serious tools you’ll reach for on almost every box.

For the Meow machine, the next question focuses specifically on **port 23**, so I ran a targeted Nmap scan against that port.

---

## Peeking at Port 23

### Question 6

**What service do we identify on port 23/tcp during our scans?**

**Answer:** `telnet`

To identify the service on port `23`, I ran:

```bash
nmap -O -sV -sC -p 23 <Target_IP_Address>
```

### Breaking down the command

* `-O` — Attempts to identify the target operating system
* `-sV` — Detects the version of the running service
* `-sC` — Runs Nmap’s default NSE scripts
* `-p 23` — Scans only port `23`

This is a nice compact command for quick service enumeration when you already know which port you want to inspect.

![Nmap output showing port 23 open and identified as Telnet.](https://safsecmedia.blob.core.windows.net/images/HTB/Meow/nmapScan.png){: .shadow }
*Nmap output showing port 23 open and identified as Telnet.*

### Why Telnet stands out

Port `23` is commonly associated with **Telnet**, an old remote-access protocol that sends data in **clear text**. That means usernames, passwords, and session data can be exposed to anyone monitoring the traffic.

In modern environments, Telnet is considered highly insecure and has largely been replaced by SSH. So if you ever see Telnet exposed on a target, it immediately becomes interesting.

> **Red flag:**
> Telnet is a legacy protocol with no built-in encryption. Finding it exposed is often a sign that the system is weakly configured or intentionally vulnerable in a lab.

---

## Step 5: Finding the Login

## The “Duh” Moment: Privilege Escalation Without the Escalation

### Question 7

**What username is able to log into the target over telnet with a blank password?**

**Answer:** `root`

This was one of those moments where the answer feels obvious *after* you know it.

At first, I tried a few generic usernames, but nothing worked. Eventually I looked at the hint:

> *It is popularly known as the administrative account for any Linux-based Operating System, residing at the highest level of privilege on any such system.*

That points directly to the default Linux superuser: `root`.

What makes this machine beginner-friendly is that there’s no complicated exploit chain here. The system is intentionally vulnerable, and the challenge is really about recognizing the service, understanding what it does, and trying the obvious administrative account.

> **Lesson learned:**
> In CTFs — especially beginner labs — don’t overcomplicate things too early. Sometimes the intended path is there to teach a concept, not to trick you.

---

## Step 6: Logging In and Capturing the Flag

Once we know Telnet is exposed and the login username is `root`, the final step is to connect and grab the flag.

### Connecting to the service

```bash
telnet <Target_IP_Address> 23
```

![Telnet login prompt requesting a username.](https://safsecmedia%2Eblob%2Ecore%2Ewindows%2Enet/images/HTB/Meow/telnet%20login%2Epng){: .shadow }
*Telnet login prompt requesting a username.*

When prompted:

* enter `root` as the username
* leave the password blank
* press **Enter**

If the login succeeds, you’ll land in a shell on the target system.

---

## Basic Linux Commands Used

Once inside the machine, basic Linux navigation is enough to locate the flag:

* `pwd` — print the current directory
* `ls` — list files and directories
* `cd` — change directory
* `cat <filename>` — print the contents of a file

A typical workflow looks like this:

```bash
pwd
ls
cd /root
ls
cat flag.txt
```

![Successful Telnet session showing the flag being read from the target.](https://safsecmedia.blob.core.windows.net/images/HTB/Meow/loginSucc.png){: .shadow }
*Successful Telnet session showing the flag being read from the target.*

And with that, the box is complete.

---

## What This Box Teaches

Even though **Meow** is extremely simple, it introduces several habits that carry over into real pentesting labs:

1. **Connect to the environment properly** using a VPN.
2. **Verify reachability** before doing deeper enumeration.
3. **Use Nmap early** to understand the attack surface.
4. **Pay attention to legacy protocols** like Telnet.
5. **Try simple, logical authentication paths** before assuming the challenge is complicated.
6. **Be comfortable in a Linux shell**, because post-login navigation is part of the job.

That’s why this machine works so well as an introduction: it isn’t about flashy exploitation — it’s about building the workflow.

---

## Command Cheat Sheet

### OpenVPN

Used to connect to the Hack The Box lab network.

```bash
sudo openvpn <your_config_file.ovpn>
```

### Ping

Used to test basic connectivity to the target.

```bash
ping <Target_IP_Address>
```

### Nmap

Used to discover open ports, identify services, and gather reconnaissance.

```bash
nmap -sV -sC <Target_IP_Address>
```

### Telnet

Used to connect to the Telnet service running on the target.

```bash
telnet <Target_IP_Address> 23
```

### Basic Linux Navigation

```bash
ls
cd <directory>
pwd
cat <filename>
```

---

## Final Thoughts

**Meow** is one of those boxes that looks almost too easy at first glance, but that’s exactly the point. It introduces the beginner workflow in a controlled way: connect, verify, enumerate, identify the service, log in, and capture the flag.

If you’re just starting out on Hack The Box, don’t rush past these early machines. The concepts they teach — enumeration discipline, protocol awareness, and command-line comfort — are the same concepts that keep showing up later on much harder boxes.

And honestly, that’s one of the best lessons in cybersecurity:

> **The basics are not “beginner stuff.”**
> The basics are the foundation everything else is built on.

See you in the next write-up.

**— SafSec**

