---
layout: post
title: EHAX CTF 2026 - Borderline Personality (Web)
date: 2026-02-28
description: How the "Borderline Personality" challenge was solved at the EHAX CTF 2026 event.
tags: writeups web
categories: writeups web
pseudocode: true
author: "Anonymous Student"
featured: false
---

## Challenge Overview

- **Challenge Name:** Borderline Personality
- **Category:** Web
- **Description:** The proxy thinks it's in control. The backend thinks it's safe. Find the space between their lies and slip through.
- **Provided Files / URL:**
    - URL http://chall.ehax.in:9098/
    - Files (sources of the web server):
        - docker-compose.yml
        - haproxy.cfg
        - Backend app.py
- **Goal:** Get the flag from the backend server.

## Initial Analysis
First, I examined the provided website. It offers a textbox to enter a payload and a button to submit the payload to the API. When one inputs some text and submits it, we get as a response a message, that the data was successfully received.

Then, there is a section "Restricted Administrative Operations" with a button to access `/admin/flag` (internal only). When one clicks this button, we get as a response the message "HTTP 403 Forbidden: HAProxy WAF blocked this request. Only local-origin traffic allowed."

After that I had a look at the different source files which were provided.

The `docker-compose.yml` had the following content:
```
services:
  haproxy:
    image: haproxy:1.9.1-alpine
    volumes:
      - ./haproxy/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
    ports:
      - '9098:8080'
    depends_on:
      - backend

  backend:
    build: ./backend
    expose:
      - '5000'
    restart: always
```

The `haproxy.cfg` had the following content:
```
global
    log stdout format raw local0
    maxconn 2000

defaults
    log     global
    mode    http
    option  httplog
    timeout connect 5000ms
    timeout client  50000ms
    timeout server  50000ms

frontend http-in
    bind *:8080
    
    acl restricted_path path -m reg ^/+admin
    http-request deny if restricted_path
    
    default_backend application_backend

backend application_backend
    server backend1 backend:5000
```

And the backend `app.py` had the following content:
```python
from flask import Flask, request, jsonify, render_template   
   
app = Flask(__name__)   
    
@app.route('/')  
def index():  
    return render_template('index.html')  
    
   
@app.route('/api/submit', methods=['POST'])  
def submit():  
    data = request.get_data()   
    return jsonify({"status": "success", "message": "Data received."}), 200   
    
    
@app.route('/admin/flag', methods=['GET', 'POST'])   
def flag():   
    return "EHAX{TEST_FLAG}\n", 200   
    
    
@app.errorhandler(404)   
def not_found(e):   
    return "Not Found\n", 404   
```

In the `app.py` we can see, that when we submit data to `/api/submit`, we always get the same response, no matter what data we submit, because the data isn't used for anything. Then, we can also see, that the route `/admin/flag` isn't protected at all. When the backend receives a request to `/admin/flag` it returns the flag.

The route `/admin/flag` is only protected by the proxy, which defines any path that matches the regex `^/+admin` as a restricted path and denies any request to a restricted path. Request to paths which are not restricted are forwarded to the backend.

## Solution Path
As we saw in the analysis above, the proxy is the reason for why we can't get the flag, because it blocks requests to `/admin/flag`. We somehow have to trick the proxy into forwarding our request to `/admin/flag` to the backend. If we manage to do this, the backend will return the flag.

Looking closer at the regex which is used by the proxy to filter restricted paths, we see that it matches any path that starts with one or many `/` followed by `admin`. To pass this check, which would result in our request being forwarded to the backend, we have to modify our request path, such that it doesn't match this pattern. On the other hand, our request path still has to be interpreted as `/admin/flag` by the backend, otherwise we don't get the flag. This means that we have to construct a path that isn't recognised by the proxy as containing `/admin` but is by the backend.

After thinking about it I thought that many backend frameworks like Flask, which is used here, probably decode and normalise paths, but the proxy may not. So, we could try to construct a path which doesn't include `/admin` directly (to pass the check by the proxy), but is normalised by the backend to `/admin` (to still reach the flag endpoint). To achieve this I had two ideas:
- Add a second `/` in front of `admin`, but in the encoded form `%2F`, i.e. `/%2Fadmin/flag`. Here the idea is that the proxy doesn't decode and normalise the path, which would lead to the regex not matching the path and the request being forwarded to the backend. The backend then decodes and normalises the path, which results in the path `/admin/flag`, which is exactly what we want.
- Add a `./` after the first `/` in the path, i.e. `/./admin/flag`. Here the idea is again that the proxy doesn't normalise the path, which would also lead to the regex not matching the path and the request being forwarded to the backend. The backend then normalises the path, which results again in the path `/admin/flag`.

In the dev tools of my browser I then copied the original GET request to `/admin/flag`, changed the path to `/%2Fadmin/flag` and sent the request. A bit surprised, this directly worked out and I got the flag in the response.

## Conclusion
In the end, the solution to this challenge seemed quite easy and straight forward. But, since this was my first CTF challenge that I ever played, it took me quite a long time to solve. When first analysing the website and the source files, everything seemed correct and secure. On first glance, the proxy correctly checks the request path and blocks requests to `/admin`. The challenge was then to find the right place where the vulnerability was located and to come up with a good idea of how to exploit it.

My key takeaways from this challenge are that small details are often the key to success and that you always need to carefully analyse situations that appear correct or harmless at first glance.
