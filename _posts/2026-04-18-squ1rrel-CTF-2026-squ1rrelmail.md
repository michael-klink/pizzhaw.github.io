---
layout: post
title: squ1rrel CTF 2026 - squ1rrelmail (Web)
date: 2026-04-18
description: How the "squ1rrelmail" challenge was solved at the squ1rrel CTF 2026 event.
tags: writeups web squ1rrelctf
categories: writeups web
author: "Anonymous Student"
---

## Challenge Overview

- **Name of the CTF Event:** squ1rrel CTF 2026
- **Challenge Name:** squ1rrelmail
- **Category:** Web
- **Description:** The underground squirrel messaging network has been shut down by campus IT. They say the site is gone for good, but rumors say the moderators left a few doors open. Can you find a way back in?
- **Provided Files / URL:** `https://squ1rrelmail.squ1rrel.dev/`
- **Goal:** Find the flag.

## Initial Analysis

{% include figure.liquid loading="eager" path="assets/img/posts/2026-04-18-squ1rrelmail/landingpage.png"
class="img-fluid rounded z-depth-1" max_width="500px"%}

After opening the challenge website I was greated with the above page. Since it didn't contain any useful information, I had a look at the page's source code. In there I found the following comment:

```html
<!-- TODO: disable /login endpoint before public takedown page goes live -->
```

The comment hints at an endpoint `/login` which maybe still exist, so I tried to access it. Opening `https://squ1rrelmail.squ1rrel.dev/login` revealed the following page:

{% include figure.liquid loading="eager" path="assets/img/posts/2026-04-18-squ1rrelmail/login.png"
class="img-fluid rounded z-depth-1" max_width="500px"%}

I entered the tree name "titanwood" (which I just made up) and clicked on "Burrow In", which opened the next page `https://squ1rrelmail.squ1rrel.dev/dashboard`:

{% include figure.liquid loading="eager" path="assets/img/posts/2026-04-18-squ1rrelmail/dashboard.png"
class="img-fluid rounded z-depth-1" max_width="500px"%}

Entering something in the textfield on the dashboard and submitting it didn't do anything. I quickly checked the page source code, which confirmed that the form didn't have any functionality, i.e. nothing was done with the input.

The text below the form contained some useful information. Apparently, the dashboard functionality is restricted to admin accounts and the session seems to be secured with a weak key.

## Solution Path

Based on my initial analysis, I had a look at the cookies in the browser web storage, where I found a JSON Web Token. To inspect the JWT I used [jwt.io](https://jwt.io):

{% include figure.liquid loading="eager" path="assets/img/posts/2026-04-18-squ1rrelmail/jwt-decode.png"
class="img-fluid rounded z-depth-1" max_width="500px"%}

The JWT includes our chosen username and our role, which is set to `user`. In order to verify the JWT signature and change the role to `admin`, I had to find the key. The hint on the dashboard mentioned that the secret isn't hard to guess, so I tried different options like "secret", "acorn", "tree" etc. until I finally guessed the correct key, which was "squirrel". Knowing the correct secret then allowed me to change the role from `user` to `admin` and sign the new token:

{% include figure.liquid loading="eager" path="assets/img/posts/2026-04-18-squ1rrelmail/jwt-encode.png"
class="img-fluid rounded z-depth-1" max_width="500px"%}

I copied the new JWT, saved it in the token cookie on the challenge website and reloaded the page, which forwarded me to the next page `https://squ1rrelmail.squ1rrel.dev/acorn-inbox`:

{% include figure.liquid loading="eager" path="assets/img/posts/2026-04-18-squ1rrelmail/acorn-inbox.png"
class="img-fluid rounded z-depth-1" max_width="500px"%}

Since the form on the dashboard didn't provide any functionality, I checked the source code of this new page first, to see if this one actually did something. Fortunately, this form actually sent a GET request to the backend. Apart from that, I couldn't find any hints in the source code.

First, I tried simple inputs like "test", "squirrel", or "flag". Submitting these just returned them unchanged.

Next, I tried `${7*7}` and what is shown in the picture below (I actually wanted to include the second payload right here, but our website simply won't display it correctly...). The first one was returned unchanged, but the second one returned the following:

{% include figure.liquid loading="eager" path="assets/img/posts/2026-04-18-squ1rrelmail/acorn-7x7.png"
class="img-fluid rounded z-depth-1" max_width="500px"%}

The input was evaluated by the server, which indicated that we were dealing with a server-side template injection (SSTI) vulnerability. Using the following decision tree and some more information I found in an online article by [PortSwigger](https://portswigger.net/web-security/server-side-template-injection), and after trying specific inputs, I came to the conclusion that the website used the "Jinja" templating engine.

{% include figure.liquid loading="eager" path="assets/img/posts/2026-04-18-squ1rrelmail/ssti-decision-tree.png"
class="img-fluid rounded z-depth-1" max_width="500px"%}

My guess was that the flag was stored either in an environment variable or in a file. Trying to access the config returned the following (all payloads I used from now on are included as screenshots, since our website doesn't display them if I include them here directly):

{% include figure.liquid loading="eager" path="assets/img/posts/2026-04-18-squ1rrelmail/payload-config.png"
class="img-fluid rounded z-depth-1" max_width="500px"%}

Unfortunately, the flag wasn't included here, so I decided to move on and try to find the flag in a file on the server. To get a list of available classes, which I could then use to access the file system of the server, I constructed the following input:

{% include figure.liquid loading="eager" path="assets/img/posts/2026-04-18-squ1rrelmail/payload-list-subclasses.png"
class="img-fluid rounded z-depth-1" max_width="500px"%}

Here is a quick explanation of the different parts of the input:

```python
''.__class__              # <class 'str'>
''.__class__.__mro__      # Method Resolution Order
''.__class__.__mro__[1]   # object (the base class of all Python objects)
```

The complete command calls `object.__subclasses__()` and returns all the classes that have ever been loaded in the current Python process. This is a "shadow list" of all imported modules and classes.

Submitting this payload returned a list with more than 500 classes. Finding the correct indexes of the classes I wanted to use was too cumbersome with this list, so I crafted the following input, which returned a list of the class names together with their index:

{% include figure.liquid loading="eager" path="assets/img/posts/2026-04-18-squ1rrelmail/payload-list-subclasses-index.png"
class="img-fluid rounded z-depth-1" max_width="500px"%}

The class I was specifically looking for was `subprocess.Popen`, which would allow me to spawn new processes and execute commands on the server. This class could be found at index 361. Then, I crafted the following input, which would allow me to execute any command in a shell on the server and return the output of that command:

{% include figure.liquid loading="eager" path="assets/img/posts/2026-04-18-squ1rrelmail/payload-shell-command-execution.png"
class="img-fluid rounded z-depth-1" max_width="500px"%}

Next, I used this input to have a look at which files and directories existed on the server:

{% include figure.liquid loading="eager" path="assets/img/posts/2026-04-18-squ1rrelmail/payload-ls.png"
class="img-fluid rounded z-depth-1" max_width="500px"%}

After some trial and error I found the following files and directories in the root directory of the server:

- app
- bin
- boot
- dev
- etc
- **flag.txt**
- home
- lib
- lib64
- media
- mnt
- opt
- proc
- root
- run
- sbin
- srv
- sys
- tmp
- usr
- var

The file `flag.txt` looked promising, so I tried to print the content of it:

{% include figure.liquid loading="eager" path="assets/img/posts/2026-04-18-squ1rrelmail/payload-cat-flag.png"
class="img-fluid rounded z-depth-1" max_width="500px"%}

The response finally contained the flag: `squ1rrel{...}`

{% include figure.liquid loading="eager" path="assets/img/posts/2026-04-18-squ1rrelmail/flag.png"
class="img-fluid rounded z-depth-1" max_width="500px"%}

## Conclusion

Overall, this was a very nice CTF challenge. It clearly demonstrates just how dangerous server-side template injection can be. SSTI allows for remote code execution and direct access to server resources, all without any form of authentication.

A software security course I took last semester briefly touched on the topic of server-side template injection, but we only looked at the basic concept. So I knew that such vulnerabilities existed, but I had to work out for myself how to actually exploit them in this challenge. This took a lot of time and patience, but it was great to delve deeper into a concept I had encountered in my studies and pick up some new knowledge along the way. It was also cool that the challenge wasn't just about the server-side template injection vulnerability, but that you had to solve a few other puzzles first.
