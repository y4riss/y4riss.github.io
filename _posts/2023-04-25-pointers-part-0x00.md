---
layout: post
title: C programming - Pointers 0x00
date: "2023-04-25 00:12:43 +0100"
categories: [Articles, C programming - Pointers]
tags: [c, pointers]
---

Hello there, dear reader.

In this article, we are going to get a glimpse of pointers in the C programming language.

Now I'm not a C wizard I can assure you that, but I think i have a good grasp of it and I'm here to share the knowledge !

## What is a pointer ?

To make it quick, pointers are just `variables`.

The special thing about them is that instead of storing a `value`, they store `addresses`.

### But how ?

Let's take this simple example

```c

int main(void)
{
  int a;
  int *p;

  a = 1337;
  p = &a;
}
```

Pointers are defined by prepending an asterisk (\*) to the variable name.

As we said, pointers point to data, they hold addresses.

To get an address out of a variable, we append `&` to it.

To get the value of data pointed by a pointer, we use `*` to dereference our pointer.

> Important Note : the asteriks `*` used to declare a pointer and dereference it are different.
>
> - The first tells the compiler that the variable is a pointer, after that we don't use it again.
> - The second is when we need to access the value of data that our pointer is pointing to (which is called dereferencing)

Here is a simple illustration to make it a bit clear
<img src="/../assets/c_pointers0_img1.png" />

- The left column represents the identifiers (the name of our variables ).
- The middle column is our memory.
- The right column represents addresses - where those variables live in the memory.

As you can see in the illustration, when we initialized `a = 1337` , our `a` has been reserved `4 bytes` in memory, which I will represent by `0x04` for the sake of understanding.

> Note : 4 bytes since our variable is an int, and int is often 4 bytes.
> You can check its size by printing `sizeof(int)`

As for `p`, you can notice 2 things :

- memory reserved for `p` is almost doubled compared to `a`
- `p` stores the address of `a`

Our **pointer** will always get allocated `8` or `4` bytes of memory depending on the processor's architecture ( 4 bytes for x86 processors, and 8 bytes for x64 processors ) regardless of the type it points to (char, int , ...).

### But why ?

Why do we need pointers 😕 ??

Jokes on you, they are made for one purpose : to make our lives harder 😨.

<img src="https://media.pinatafarm.com/protected/B183D0EF-49B8-47BF-A523-E72FD0CFFAAC/Grumpy-Cat-Reverse.3.meme.webp" />

Okey jokes aside, Pointers are awesome 😈 (Yeah i sound crazy ikr), they have many use cases and are very powerful when it comes to memory management.

Now it would be boring 🥱 if I start listing pointers use cases and giving definitions, that's not my goal.

Instead, I want to keep you entertained and challenged at the same time, so here is a problem for you :

<span style="color : #ff5b5b">**Write a `c function` that swaps the values of two integers.**</span>

```c
#include <stdio.h>

void swap(...)
{
  // your code goes here
}

int main(void)
{
  int a;
  int b;

  a = 1;
  b = 2;

  printf("before swapping...\n");
  printf("a : %d, b : %d\n",a,b); //a : 1, b : 2
  //swap(...)

  printf("after swapping...\n");
  printf("a : %d, b : %d\n",a,b); //a: 2, b : 1
}
```

I'll wait for you to finish...

Got it? good job 😁.

You used pointers? Excellent JOB 😍.

Have I made my point ? HELL YEAH 😍.

You can find the solution at the end of the article.

## Pointers - a bit trippy ?

Let's dive a little bit deeper.

### Trippy 0x00

You know that, when declaring an array, you are just making a pointer that points to the first element of that array.

```c
#include<stdio.h>

int main(void)
{
        int arr[] = {1,2,3,4,5};

        printf("%d",*arr);

        return 0;
}
```

After compiling and executing our program, what do you think the output would be?

```bash
1
```

As I said, `arr` is just a pointer to the first element of the array, since the first element is 1, when dereferencing we get 1.

### Trippy 0x01

Trippy enough? little do you know 🤭

Let's create a simple program that accomplishes the following tasks:

- creating a simple static array of 5 integers.
- making a pointer that points to that array.
- print the elements of the array using the pointer.

Sounds simple enough...

```c
#include <stdio.h>

int main(void)
{
        int     arr[] = {1, 2, 3, 4 ,5};

        int     *ptr;
        int     i;

        ptr = arr;

        for(i = 0 ; i < 5; i++)
        {
                printf("arr[%d] : %d\n",i,*(ptr + i));
        }
        return 0;
}
```

```bash
#Program stdout
arr[0] : 1
arr[1] : 2
arr[2] : 3
arr[3] : 4
arr[4] : 5
```

<img src="/../assets/c_pointers0_img2.png" />

### Trippy 0x02

<img src="https://media.makeameme.org/created/why-you-doing-c9hea1.jpg"/>

Well, it's fun isn't it 🥰 ?

Let's use the same program we used above, and change one thing , **our pointer's type**.

```c
#include <stdio.h>

int main(void)
{
        int     arr[] = {1, 2, 3, 4 ,5};

        char    *ptr; // we changed int to char
        int     i;

        ptr = (char *)arr; // this is called typecasting, basically i just added it to avoid the compiler's warnings

        for(i = 0 ; i < 5; i++)
        {
                printf("arr[%d] : %d\n",i,*(ptr + i));
        }
        return 0;
}
```

Technically we just said to the compiler that ptr points to `char` instead of `int`.

What would happen in this case? Let's run our program and see.

```bash
#Program stdout
arr[0] : 1
arr[1] : 0
arr[2] : 0
arr[3] : 0
arr[4] : 2
```

<img src="/../assets/c_pointers0_meme2.png">
<img src="https://i.imgflip.com/5dlmso.png" />

Let me explain...

By definition, `char` is 1 byte long, and `int` is 4 bytes long.

Now if we take the first element of our array, which is `1`, and write it in binary, respecting the 4 bytes rule, we get this :

`00000000 00000000 00000000 00000001`

Each byte is 8 bits, and we need 4 of them.

Now let's rewrite this in hexadecimal.

`0x00000001 <=> 00 00 00 01`

Is the image getting clearer ? let's continue...

You are most likely using an **intel** / **amd** processor.Both those processors use `Little Endian byte order`.

Basically `little endian` would store bytes from the least significant byte to the most significant byte.

Meaning `00 00 00 01` will be stored as `01 00 00 00`

<img src="/../assets/c_pointers0_img3.png"/>

Do you get it now ? `01 00 00 00` are 4 bytes (no shit 😮).

And since our pointer is a pointer to `char`, it prints `1 byte` at a time inside our program's loop.

the first byte is 1, then 0, then 0 then 0.

### Trippy 0x03 ~ Challenge

Okey for this part, you are going to prove to me that you grasped what we've discussed earlier.

<span style="color : #ff5b5b">**Your task is to fill in the array with exactly `one element` to match the output.**</span>

> Note : use a pen ✍🏼 and paper 📄, it's really helpful.

```c
#include<stdio.h>

int main(void)
{

        int int_array[1] = {/*put one element here*/};
        char *ptr = (char *) int_array;

        for(int i = 0 ; i < 4 ; i++)
                printf("%d\n",*(ptr + i));
        return 0;
}
```

```bash
#Program stdout
1
2
3
4
```

The solution will be in the next article ~ Inshallah.

## Solutions

### exercise 1 - swap

```c
// the function
void swap(int *q, int *p)
{
  int tmp;

  tmp = *q;
  *q = *p;
  *p = tmp;
}

int main(void)
{
  //...
  swap(&a,&b) // since swap accepts pointers, you need to provide addresses
  //...
  return 0;
}
```

## Bibliography

📖 Hacking: The Art of Exploitation, 2nd Edition

[https://www.gnu.org/software/gnu-c-manual/gnu-c-manual.html#Pointers](https://www.gnu.org/software/gnu-c-manual/gnu-c-manual.html#Pointers)
