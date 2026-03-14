---
layout: post
title: UTCTF 2026 - Half Awake (Forensics)
date: 2026-03-13
description: How the "Half Awake" challenge was solved at the UTCTF 2026 event.
tags: writeups forensics utctf
categories: writeups forensics
author: "Anonymous Student"
---

## Challenge Overview

- **Name of the CTF Event:** UTCTF 2026
- **Challenge Name:** Half Awake
- **Category:** Forensics
- **Description:** Our SOC captured suspicious traffic from a lab VM right before dawn. Most packets look like ordinary client chatter, but a few are pretending to be something they are not.
- **Provided Files / URL:** `half-awake.pcap`
- **Goal:** Find the flag.

## Initial Analysis

After opening the provided `.pcap` file in Wireshark I saw the following traffic:

{% include figure.liquid loading="eager" path="assets/img/posts/2026-03-13-half-awake/wireshark.png"
class="img-fluid rounded z-depth-1" max_width="500px"%}

At the top we have some TCP and HTTP traffic, followed by some MDNS traffic. Then there is again mostly TCP traffic with some TLS in between.

First, I had a look at the HTTP request (packet nr. 4), which is a GET request to an endpoint `/instructions.hello`:
```
GET /instructions.hello HTTP/1.1
Host: awake.utctf.local
User-Agent: half-awake-client/1.0
Accept-Encoding: gzip, xor
```
This looked quite promising, since the name of the endpoint includes "instructions", so hopefully the response contains some useful information.

In the corresponsing response (packet nr. 6) I then found the following information:
```
Read this slowly:
1) mDNS names are hints: alert.chunk, chef.decode, key.version
2) Not every 'TCP blob' is really what it pretends to be
3) If you find a payload that starts with PK, treat it as a file
```

## Solution Path

The first hint probably refers to the three MDNS packets (nr. 8 - 10). Looking at the individual packets in more detail I couldn't find any useful information, so probably only the names themselves are relevant.

The name `alert.chunk` looked interesting. Combining it with hint number 3, I then thought that there may be a packet "alert" with some useful payload. Looking through the entire traffic I found an "Encrypted Alert" in packet nr. 36. The hex dump of the payload looked like this:
```
0030                                    50 4b 03 04 14
0040   00 00 00 08 00 aa ba 6b 5c eb 92 16 71 2e 00 00
0050   00 29 00 00 00 0a 00 00 00 73 74 61 67 65 32 2e
0060   62 69 6e 01 29 00 d6 ff 75 c3 66 db 61 d0 7b df
0070   34 db 66 e8 61 c0 34 dc 33 e8 73 84 33 e8 74 df
0080   33 e8 70 c5 30 c3 30 d4 30 db 5f c3 72 86 63 dc
0090   7d 50 4b 03 04 14 00 00 00 08 00 aa ba 6b 5c 9c
00a0   fe 88 9d 2e 00 00 00 2e 00 00 00 0a 00 00 00 72
00b0   65 61 64 6d 65 2e 74 78 74 05 c1 41 0e 00 10 0c
00c0   04 c0 bb 57 ec d7 84 8d f6 a0 a4 1a d2 df 9b b1
00d0   15 e0 a5 67 88 da 80 d0 09 3d a0 35 cf 1d ec 08
00e0   21 4e 9d c4 ab 59 3e 50 4b 01 02 14 03 14 00 00
00f0   00 08 00 aa ba 6b 5c eb 92 16 71 2e 00 00 00 29
0100   00 00 00 0a 00 00 00 00 00 00 00 00 00 00 00 80
0110   01 00 00 00 00 73 74 61 67 65 32 2e 62 69 6e 50
0120   4b 01 02 14 03 14 00 00 00 08 00 aa ba 6b 5c 9c
0130   fe 88 9d 2e 00 00 00 2e 00 00 00 0a 00 00 00 00
0140   00 00 00 00 00 00 00 80 01 56 00 00 00 72 65 61
0150   64 6d 65 2e 74 78 74 50 4b 05 06 00 00 00 00 02
0160   00 02 00 70 00 00 00 ac 00 00 00 00 00

```

The first two bytes `50 4b` translated into ASCII symbols is actually `PK`, so according to hint number 3 this payload should be treated as a file. Now the question arose, which file type? A quick research on the internet revealed that a file that starts with `50 4b 03 04` is a ZIP file. So, I copied the payload and created a new file `alert.zip`. This file contained two different files. A `readme.txt` and a `stage2.bin`. The readme contained just the following information:
```
not everything here is encrypted the same way
```

For now I didn't know exactly what to do with this information, but maybe it will become important later.

The file `stage2.bin` contained the following hex values:
```
75 c3 66 db 61 d0 7b df 34 db 66 e8 61 c0 34 dc 33 e8 73 84 33 e8 74 df 33 e8 70 c5 30 c3 30 d4 30 db 5f c3 72 86 63 dc 7d
```

Translated into ASCII symbols I got the following:
```
uÃfÛaÐ{ß4ÛfèaÀ4Ü3ès3ètß3èpÅ0Ã0Ô0Û_ÃrcÜ}
```

On the first glace this didn't make much sense. The only thing that looked promising are the `{` and `}`, which may be a hint that it includes the flag. Looking back at the information in the readme, I came to the conclusion that the content of `stage2.bin` is probably encrypted somehow. Since I didn't have a key yet, I went back to Wireshark and the initial hints.

Hint number two states that not every TCP packet is what it pretends to be, so I had a look at all of the TCP packets. All packets have length 54, except for packet nr. 30, which has length 99, so I had a detailed look at that packet first. In there I found the following information:
```
*client_hello-ish_bytes___Agbogbloshie___
```

To make sure the other TCP packets didn't include some useful information too, I also looked through all of them, but actually didn't find anything useful in them.

Additionally, I also looked into the different TLS packets which appear between the many TCP packets. Inside these "Encrypted Handshake Messages" I found the following 5 pieces of information:
```
1) Golden Showers Far East 
2) Agbogbloshie forwarding
3) inventory_status=OK.....
4) randomized_tls_payload_block_01!
5) randomized_tls_payload_block_02!
```

Interestingly, this is the second time "Agbogbloshie" is mentioned. Since I have never heard of this before, I did some research online. According to Wikipedia Agbogbloshie was a slum and commercial district in Accra, Ghana, where electronic scrap was collected, processed and exported. Unfortunately, this information didn't really help me to solve this CTF challenge. Regarding the other pieces of information, I also couldn't make much sense of them or didn't see how they help solving the challenge.

So, I had another look at the first hint concerning the MDNS packets. There I realized that there is a fourth MDNS packet I didn't see before. Packet nr. 11 is the response to one of the above three MDNS requests, namely for `key.version.local`. This packet includes an answer in the TXT format with the value `00b7`. This value probably has something to do with the key which we need to decrypt the content of `stage2.bin`, or maybe it even is the key.

Looking again at the ASCII representation of the content of `stage2.bin` I realized that the characters nr. 1, 3, 5, ... are "normal" characters, while the other characters are more "cryptic" looking. So, maybe only every second character has to be transformed somehow while the other characters stay untouched. This could also explain the hint "not everything here is encrypted the same way" from the `readme.txt`. If we assume the value `00b7` I found before to be the key, XOR encryption/decryption would fit, because when we XOR something with `00b7`, only every second character will be changed. I therefore took the hex representation of the content of `stage2.bin` and XORed it with `00b7`, which finally revealed the flag:
```
utflag{...}
```

## Conclusion

This CTF was the first forensics challenge I ever solved and I really liked it. It was a lot of fun to investigate the provided traffic, finding useful information and combining everything to reach the final goal. My main takeaways from this challenge are that in forensic tasks, individual bytes of information are often key to solving a task and that not everything that looks interesting or promising is useful or even necessary to solve such a challenge.
