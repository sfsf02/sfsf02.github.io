---
layout: post
title: "TryHackMe Support Writeup: From BOLA to Command Injection RCE"
date: 2026-07-20 09:32:42 +0200
categories: [CTF, TryHackMe]
tags: [thm, tryhackme, penetration-testing, bola, lfi, rce, command-injection, web-exploitation]
description: "A complete walkthrough of the TryHackMe 'Support' room, chaining Cookie Manipulation, BOLA, Local File Inclusion, and Command Injection to gain RCE."
excerpt: "Join me as I tackle the TryHackMe 'Support' challenge! We'll start with basic enumeration, stumble into some BOLA vulnerabilities, forge cookies, and eventually chain our way to a reverse shell."
---

Welcome back, fellow geeks! Today, we are diving into a fun TryHackMe (THM) room called [**Support**](https://tryhackme.com/room/support). 

Before we jump in, let's read the challenge description to understand our target:

> "A new internal Support Operations Platform has been deployed to assist IT and helpdesk teams. The application handles user management, internal APIs, and system-level operations. However, security was not the primary focus during development. Several features rely on user-controlled input and weak trust boundaries."

Our ultimate task? 
> "Can you pentest the platform and escalate your access to achieve RCE on the server?"

Let's process our thoughts here. We have a new web platform handling internal APIs and system operations. Because this is purely a web application pentest, it narrows our focus nicely. We don't need to worry about heavy infrastructure exploitation—we just need to poke at this website until it breaks. Remember to stick to the scope (scope creep is real, folks!).

Let's dive in!

### Initial Reconnaissance & Enumeration

First things first, I navigated to the target IP in my browser to see what we were dealing with.

![Support login page.](https://safsecmedia.blob.core.windows.net/images/THM/Support/loginpage.png){: .shadow }

Gotcha. We are greeted by a Help Desk login page, and conveniently, the page provides us with a valid helpdesk email address! 

My immediate instinct was to try brute-forcing the login with this email. However, as any good pentester knows, thorough enumeration is the key to success. Before launching loud attacks, we need to map the application's surface area. 

While I manually inspected the page's source code (which unfortunately yielded nothing interesting), I fired up Gobuster in the background to hunt for hidden directories.

```bash
gobuster dir -u http://TARGET_IP:80 -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -x bak,txt,html,php,js -t 20
```
*Why we run it:*
* `-u` → The target URL.
* `-w` → The path to our wordlist (using SecLists).
* `-x` → File extensions to search for (very important for finding hidden backups or configuration files).
* `-t 20` → Use 20 concurrent threads to speed up the scan.

![Gobuster output.](https://safsecmedia.blob.core.windows.net/images/THM/Support/GobusterOutput.png){: .shadow }

Gobuster struck gold! Here is what we found under the hood:
* `api.php` (Visiting this directly redirected me back to the login page)
* `config.php`
* `dashboard.php`
* `/includes` directory (Contains `header.php` and `skin.php`)
* `/js` directory
* `/layout` directory
* `/skins` directory (Contains PHP files related to UI colors)

> 💡 **Pentesting Tip:** Whenever a web application has a feature to change "skins" or "themes" via local files, your hacker senses should tingle. This is a classic setup for a Local File Inclusion (LFI) vulnerability!

### Brute-Forcing Initial Access

I like to manage my time efficiently. Since I already tested the login page manually five times and noticed there was no account lockout mechanism, it was time to bring in `ffuf` to brute-force the password for our known helpdesk email.

```bash
ffuf -w ~/Downloads/rockyou.txt -X POST -d "email=help%40support.thm&password=FUZZ" -H "Content-Type: application/x-www-form-urlencoded" -u http://TARGET_IP -fc 200
```
*Why we run it:*
* `-w` → Path to our trusty `rockyou.txt` password list.
* `-X POST` → Specifies that we are sending a POST request.
* `-d` → The data payload we are sending (notice `password=FUZZ`, which tells ffuf where to inject the passwords).
* `-H` → Sets the content type to standard form data.
* `-u` → The target URL.
* `-fc 200` → Filters out HTTP 200 OK responses (useful if a failed login returns a 200, so we can spot the anomaly that indicates a success, like a 302 redirect).

While `ffuf` was running, I poked around the directories Gobuster found. The `/js` and `/layout` folders mostly just contained the Bootstrap framework. The `/includes` folder had `header.php` and `skin.php`. 

![The /includes directory.](https://safsecmedia.blob.core.windows.net/images/THM/Support/IncludeDirectory.png){: .shadow }
![The /skins directory.](https://safsecmedia.blob.core.windows.net/images/THM/Support/SkinsDirectory.png){: .shadow }

Checking back on `ffuf`, we had a massive hit! 

![ffuf successful output.](https://safsecmedia.blob.core.windows.net/images/THM/Support/FfufOutput.png){: .shadow }

Never doubt `rockyou.txt`! We now had a valid password for the helpdesk account.

### Cookie Manipulation & Discovering BOLA

 Armed with our credentials, I logged into the application and made sure Burp Suite was running in the background to catch all the traffic.

![the dashboard page.](https://safsecmedia.blob.core.windows.net/images/THM/Support/DashboardPage.png){: .shadow }
![the Burp Suite response headers.](https://safsecmedia.blob.core.windows.net/images/THM/Support/DateRequest.png){: .shadow }

The dashboard wasn't empty; it had a functional feature for changing the theme (e.g., `http://TARGET_IP/dashboard.php?skin=green`). 

But more importantly, looking at the server's response in Burp Suite, I noticed a very suspicious `Set-Cookie` header. It assigned a cookie named `isITUser` with a value that looked suspiciously like an MD5 hash.

I headed over to [CrackStation](https://crackstation.net/) to see if I could decrypt it. Sure enough, the hash cracked perfectly. The value was simply the word: `false`.

If `isITUser` is currently set to `false`, what happens if we change it to `true`?

I generated an MD5 hash for the word "true", intercepted my next request with Burp Suite, and replaced the cookie's value. *(Note: You can also easily do this using the Storage tab in your browser's Developer Tools!)*

![the dashboard with the new IT Admin Panel section.](https://safsecmedia.blob.core.windows.net/images/THM/Support/ITPanal.png){: .shadow }

Oh la la! By forging our cookie, we tricked the application into revealing a hidden "IT admin panel" section. Let's check it out!

![the api page.](https://safsecmedia.blob.core.windows.net/images/THM/Support/APiPage.png){: .shadow }

Clicking the "View API" button revealed how the application was fetching user data:
`GET /user/3`

This endpoint returned JSON information about user #3. Whenever you see an API fetching data via an incremental ID, it is time to test for **Broken Object Level Authorization (BOLA)**—also known as Insecure Direct Object Reference (IDOR). 

> 📝 **Note on BOLA:** BOLA occurs when an application fails to check if the user requesting a specific resource actually has the permissions to view it. If we can change `/user/3` to `/user/1` and see someone else's data, the application is vulnerable.

I fired up Burp Suite's Intruder, set the user ID as the payload position, and fuzzed it from 1 to 50 to see who else was on the system.

![the Burp Suite Intruder setup.](https://safsecmedia.blob.core.windows.net/images/THM/Support/IntruderSetup.png){: .shadow }
![Burp Suite showing the BOLA results.](https://safsecmedia.blob.core.windows.net/images/THM/Support/IntruderOutput.png){: .shadow }

Bingo! We discovered three users on the platform, and one of them had an `admin` role along with their email address.

### Hitting the Jackpot: Local File Inclusion (LFI)

Remember the theme-changing functionality (`dashboard.php?skin=green`)? And the `config.php` file we found earlier? Let's combine them.

I suspected the `skin` parameter was directly loading PHP files from the `/skins` directory. Since we want to read the configuration file, I tried a directory traversal payload, omitting the `.php` extension (assuming the backend code appends it automatically).

Payload: `http://TARGET_IP/dashboard.php?skin=../config`

![URL output showing the LFI success.](https://safsecmedia.blob.core.windows.net/images/THM/Support/LFIConfigOutput.png){: .shadow }

Jackpot! The Local File Inclusion worked perfectly. Not only did we read the file, but it revealed a `MASTER_PASSWORD` variable! 

I also used the LFI to read the source code of `dashboard.php` to confirm how it was processing our inputs. By reviewing the code, it appeared that we could only read `.php` files located within the `html` directory.
![the dashboard.php source code fetched via LFI.](https://safsecmedia.blob.core.windows.net/images/THM/Support/LFIDashboardOutput.png){: .shadow }

### The "Facepalm" Moment: Logging in as Admin

With the admin email from our BOLA attack and the `MASTER_PASSWORD` from our LFI attack, I confidently went to log in.

*Invalid credentials.*

I tried again. And again. I was stuck here for over an hour. Was there a different login page? Was the password hashed? 

Eventually, I swallowed my pride and looked up a walkthrough. Huge shoutout to [Djalil Ayed on YouTube](https://youtu.be/qrg0cObFVIc?si=mzYrAw1rDdW2CkWi) for his fantastic video on this room. 

It turns out, you simply have to remove the `@` symbol from the master password before submitting it. Why? Well, Djalil mentioned in his video that he couldn't find any logical reason for it either, admitting it was just a **lucky guess**! I even asked an AI about it, and it suggested it might just be a typo by the room creator. Regardless, I never would have guessed that on my own.

![Admin Dashboard.](https://safsecmedia.blob.core.windows.net/images/THM/Support/Adminlogin.png){: .shadow }

We got our first flag! 

### Escalating to RCE via Command Injection

The admin dashboard featured a new utility that displayed the current date and time. Setting up my Burp Suite intercept again, I investigated the request. 

The application was sending a parameter named `sys` in the body of a POST request with the value `date`. 

![The request for the date functionality.](https://safsecmedia.blob.core.windows.net/images/THM/Support/DateRequest.png){: .shadow }

If the application is taking our input and passing it directly to the system's command line, it might be vulnerable to Command Injection. I tried changing the payload to `sys=whoami`.

![the "only date command allowed" error message.](https://safsecmedia.blob.core.windows.net/images/THM/Support/errorCommandInjection.png){: .shadow }

The server threw an error: `"only date command allowed"`. 
Okay, there is a filter in place. But filters can often be bypassed. I decided to try injecting a pipe character (`|`), which in Linux tells the terminal to execute the first command, and pass its output to the second command.

Payload: `sys=date|whoami`

![the successful whoami command execution.](https://safsecmedia.blob.core.windows.net/images/THM/Support/successfulInjection.png){: .shadow }

It worked! The filter only checked if the word "date" was present at the beginning of the string, completely ignoring the chained command. We have Remote Code Execution!

From here, I could just use `cat` to read the root flag, but the TryHackMe debrief explicitly mentioned the importance of getting a proper reverse shell.

I headed over to [RevShells.com](https://www.revshells.com/) to generate a payload. First, I needed to know what tools were on the target machine. I used our command injection to run `which python3`, which confirmed Python 3 was installed.

I copied a Python 3 reverse shell payload, pasted it into my Burp Suite request, and set up a netcat listener on my attack machine.

![catching the reverse shell and reading the root flag.](https://safsecmedia.blob.core.windows.net/images/THM/Support/SuccessfullReverseShell.png){: .shadow }

We are in! Root flag captured.

---

### ⚠️ Lessons Learned

This was a fantastic room that perfectly highlighted why we chain vulnerabilities together:
* **Cookie Security:** Never trust client-side data for authorization. Using a simple MD5 hash of the word `false` to determine privilege levels is incredibly dangerous.
* **BOLA / IDOR:** Always validate that the user requesting a resource is actually authorized to view it. Just because an API is hidden doesn't mean it's secure.
* **Input Sanitization:** The LFI and Command Injection both stemmed from trusting user input (`skin` and `sys`). Blacklisting words (like requiring the word "date") is rarely as effective as strict whitelisting and proper input escaping.
* **Don't Overthink It:** Sometimes a password issue really is just a weird typo!

---

### 🛠️ Command Cheat Sheet

* **Gobuster:** Used to brute-force hidden directories and files.
  ```bash
  gobuster dir -u http://TARGET_IP:80 -w /path/to/wordlist.txt -x bak,txt,html,php,js -t 20
  ```
* **ffuf:** A fast web fuzzer, used here to brute-force a login form via POST request.
  ```bash
  ffuf -w /path/to/wordlist.txt -X POST -d "email=target@email.com&password=FUZZ" -H "Content-Type: application/x-www-form-urlencoded" -u http://TARGET_IP -fc 200
  ```
* **Checking for Python:** Useful during command injection to see if we can use Python for a reverse shell.
  ```bash
  which python3
  ```

---

### Final Thoughts

The Support room is a brilliant exercise in manual enumeration, manipulating HTTP requests, and understanding how one small vulnerability (like a forgeable cookie) can unravel an entire application's security posture. 

Cheers,  
SafSec
