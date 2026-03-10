---
layout: post
title: ApoorvCTF 2026 - Days of Future Past (Web)
date: 2026-03-08
description: How the "Days of Future Past" challenge was solved at the ApoorvCTF 2026 event.
tags: writeups web cryptography
categories: writeups web cryptography
author: "Anonymous Student"
---

## Challenge Overview

- **Name of the CTF Event:** ApoorvCTF 2026
- **Challenge Name:** Days of Future Past
- **Category:** Web, Cryptography
- **Description:** CryptoVault - Secure Message Storage Platform. So can you get the secure message from the military grade security provided by our platform.
- **Provided Files / URL:** http://chals1.apoorvctf.xyz:8001/
- **Goal:** Get the secure messages which contain the flag.

## Initial Analysis

When visiting the provided website one is presented with the following home page:

{% include figure.liquid loading="eager" path="assets/img/posts/2026-03-08-days-of-future-past/cryptoVault_home.png"
class="img-fluid rounded z-depth-1" max_width="500px"%}

This page already includes some useful information. The service seems to use a proprietary XOR stream cipher to encrypt messages and provides a RESTful API to access the messages. After clicking on "Get Started" one is presented with a registration page, where an account can be created. Then, one can sign in. After the successful login, one is presented with a JWT token, but otherwise no further access to a new page is provided. The sign in page also includes the hint that we have the role "viewer".

{% include figure.liquid loading="eager" path="assets/img/posts/2026-03-08-days-of-future-past/cryptoVault_signin.png"
class="img-fluid rounded z-depth-1" max_width="500px"%}

Back on the home page I had a look at the HTML code of the page. There I found the following information:

{% include figure.liquid loading="eager" path="assets/img/posts/2026-03-08-days-of-future-past/cryptoVault_home_html.png"
class="img-fluid rounded z-depth-1" max_width="300px"%}

It seems that there is an API available at `/api/v1/` and a backup of a config. Furthermore, the file `/static/js/app.js` seems to include some more useful information:

```js
// API Configuration
const CONFIG = {
    apiBase: '/api/v1',
    version: '1.0.3',
    // TODO: Remove hardcoded backup path reference before production
    // The config backup at /backup/config.json.bak should be deleted
    backupConfig: '/backup/config.json.bak',
};
...
// Debug info (requires API key)
debug: function(apiKey) {
    return this.call('/debug', {
        headers: { 'X-API-Key': apiKey }
    });
},
...
// Get vault messages (requires admin JWT)
getMessages: function() {
    return this.call('/vault/messages');
}
```

The config backup seems to be reachable at `/backup/config.json.bak` and there are additional interesting looking endpoints `/debug` (which seems to require an API key) and `/vault/messages` (which probably delivers the messages we are looking for and which requires an admin JWT).

## Solution Path

First, I had a close look at the config backup, reachable at `/backup/config.json.bak`:

{% include figure.liquid loading="eager" path="assets/img/posts/2026-03-08-days-of-future-past/cryptoVault_backup.png"
class="img-fluid rounded z-depth-1" max_width="300px"%}

It includes an API key (probably the one needed for the debug endpoint), again some information on the internal endpoints `/api/v1/debug` and `/api/v1/vault/messages`, which seem to be important, as well as the hint that the JWT uses the HS256 algorithm.

After trying to access the endpoint `/api/v1/debug` without an API key I got the following message, which was expected:

{% include figure.liquid loading="eager" path="assets/img/posts/2026-03-08-days-of-future-past/cryptoVault_debug_unauthorized.png"
class="img-fluid rounded z-depth-1" max_width="300px"%}

It seems like the API key we found in the backup is the one needed to access the debug endpoint. After adding the X-API-Key header with the value `d3v3l0p3r_acc355_k3y_2024` to the request I got the following information:

{% include figure.liquid loading="eager" path="assets/img/posts/2026-03-08-days-of-future-past/cryptoVault_debug.png"
class="img-fluid rounded z-depth-1" max_width="300px"%}

Here we got a lot of valuable information. The first section `auth_config` includes a hint to derive a secret key as well as a secret key hash. Regarding the additional information in this section, i.e. the algorithm HS256 and the available roles (viewer, editor, admin), it seems like this information is concerning the JWT token. The JWT token we got after the login has the role viewer and we already found out that we need the role admin to access the messages. So, the secret key mentioned may be the key which was used to sign the JWT and which we could use to edit the role in our JWT token. The hint says the key is the company name (lowercase) concatenated with the founding year. The second section `company_info` in the debug information includes all this information. The company name is "CryptoVault" and the founding year is 2026. Therefore, the key would probably be `cryptovault2026`. Since we also know the secret key hash, I tried hashing `cryptovault2026` with SHA256 and this hash matched the one provided in the debug information exactly, i.e. it should be the correct key. The last section `vault_info` mentions again that we need admin rights to access the endpoint `/api/v1/vault/messages`, that it includes 15 encrypted messages, and that the messages were encrypted with an XOR stream cypher. So, the next step was to change our role from viewer to admin.

Using the website [jwt.io](https://jwt.io) I had a closer look at our JWT. The secret key `cryptovault2026` also proved to be the correct key:

{% include figure.liquid loading="eager" path="assets/img/posts/2026-03-08-days-of-future-past/cryptoVault_jwt_decode.png"
class="img-fluid rounded z-depth-1" max_width="500px"%}

Using the same website I was able to change the role in the JWT from viewer to admin and sign the token again with the secret key:

{% include figure.liquid loading="eager" path="assets/img/posts/2026-03-08-days-of-future-past/cryptoVault_jwt_encode.png"
class="img-fluid rounded z-depth-1" max_width="500px"%}

Then, using our new JWT in the authorization header, I tried accessing the endpoint `/api/v1/vault/messages`, which returned the following information:

{% include figure.liquid loading="eager" path="assets/img/posts/2026-03-08-days-of-future-past/cryptoVault_messages.png"
class="img-fluid rounded z-depth-1" max_width="300px"%}

The individual messages were in the following format:

{% include figure.liquid loading="eager" path="assets/img/posts/2026-03-08-days-of-future-past/cryptoVault_messages_detail.png"
class="img-fluid rounded z-depth-1" max_width="300px"%}

The final step was to decrypt the messages, since the flag will probably be included in one of them. The problem was, that we didn't have the key to decrypt them. I tried using the key which worked for the JWT, but it didn't decrypt the messages. I went through all the information I had collected until now again in detail, but I couldn't find any key or hint regarding a key. Then, I thought again about the different hints regarding the encryption of the messages. It uses an XOR stream cypher, which seems to be not really secure (according to the hints). We also have the fact that we have 15 messages and not only one, so it may be that all these messages were encrypted with the same key. If we knew the plaintext of one of the messages, we could simply XOR the ciphertext with the plaintext and we would get the key, but we don't know any (entire) plaintext. After doing some research on the internet I came across a method called "Crib Dragging". This method uses the fact that when we XOR two ciphertexts we get the same result as when we would XOR the two plaintexts (the key cancles out). If we now know a part of one plaintext, we can get the part of the second plaintext at the same position in the message.

To use this method, I found the website [Crib Drag Attack Tool](https://gfuscht.cedricgasser.com/cribdrag). On this website you can add multiple ciphertexts and then guess a "crib" (a word or phrase which you suppose to appear in the plaintext of a message). The tool then XORs the guess at all possible positions and outputs the corresponding decrypted plaintexts. If the other messages decrypt to something readable, we found a correct part of the initial message. We can then analyse the different plaintext snippets, try to expand one of the snippets, input this new snippet into the tool and have another look at the new outputs. We then iterate until we decrypted the entire messages.

To use this tool to decrypt our messages I added all 15 of them. I then thought that one of the messages probably contained the word "flag", so I used this as my first crib. After trying this first guess on all 15 messages and after looking through all the decrypted parts, it seemed like message 13 position 9 could be correct, since for all 15 messages the corresponding sections decrypted to something (kind of) readable (other possible positions are not shown in the screenshot):

{% include figure.liquid loading="eager" path="assets/img/posts/2026-03-08-days-of-future-past/cryptoVault_crib_dragging_1.png"
class="img-fluid rounded z-depth-1" max_width="300px"%}

Looking at the plaintext snippets, the "hidd" in message 14 could maybe be expanded to "hidden". Trying this revealed the following plaintext snippets (only four out of 13 options shown in the screenshot):

{% include figure.liquid loading="eager" path="assets/img/posts/2026-03-08-days-of-future-past/cryptoVault_crib_dragging_2.png"
class="img-fluid rounded z-depth-1" max_width="300px"%}

Then, in message 7, the "nalysi" could be "analysis". Trying this revealed the following snippets (again, only 4 out of multiple options shown in the screenshot):

{% include figure.liquid loading="eager" path="assets/img/posts/2026-03-08-days-of-future-past/cryptoVault_crib_dragging_3.png"
class="img-fluid rounded z-depth-1" max_width="300px"%}

Continuing like this for a long time, I finally managed to fully decrypt all 15 messages. Message 13 included the flag we were looking for (not shown here):

1. Cryptography is not about hiding secrets but about understanding how systems fail
2. When the same encryption key is reused across messages the security guarantees collapse dramatically
3. Many students believe xor encryption is safe until they see how crib dragging breaks it completely
4. Attackers do not need to know the key if they can exploit patterns in human written language
5. The history of cryptography is filled with examples where small errors caused total compromise
6. Reusable keystreams turn a one time pad into a many time pad which is fundamentally insecure
7. Careful analysis of ciphertext pairs can slowly reveal plaintext through logical deduction
8. Natural language has structure and repetition which makes statistical attacks very effective
9. Once a portion of the key is recovered it can be reused to decrypt the remaining messages
10. Cryptographic challenges are designed to teach why proper key management is critical
11. Solving this problem requires patience intuition and a good understanding of xor operations
12. Some messages may appear to contain flags but only one will fully decrypt correctly
13. **The real flag is apoorvctf{...} and all others are distractions**
14. The flag hidden in this challenge is surrounded by misleading information and decoys
15. Security through obscurity may delay attackers but it never provides real cryptographic protection

## Conclusion

This CTF challenge was not a pure web challenge, but also included some cryptography. It was important to initially collect as much information as possible, then combine this information and move forward step by step. The most difficult part was definitely the decryption of the encrypted messages. For all previous steps there were different hints which helped, but for the decryption there were nearly none. This step required some background knowledge on cryptography and a lot of creativity. Decrypting the messages using the crib dragging method also highlights many important facts about cryptography, which can be found in the messages themselves ;)
