---
layout: post
title: Creating a base64 encoder/decoder with Python - Part 1
date: "2023-04-29 13:48:19 +0100"
categories: [Tutorials, Python scripting]
tags: [python, algorithms]
---

Hello friend,

In this series, we are going to be creating a simple `base64` encoder/decoder with `python`.

We are going to do so from scratch, the aim is to understand the underlying process of how it works.

## What is base64 ?

Before attacking base64, we need to understand what a numbering system is.

We, as humans, use `base10` as our standard numbering system. If I ask you to spit out all of the digits you know, you are going to answer back with :

`0 1 2 3 4 5 6 7 8 9`

`10` defines how many symbols are in that numbering system.

There is also `base2` or `Binary`, which , if you guess, has only 2 symbols, `0` and `1`, and is pretty much a fundamental component of computing, electronics, digital technology...

Okey, what about `base64` now ?

Following the same logic, base64 is a numerical system which uses `64` symbols to encode data.

Those symbols are `[A-Z,a-z]` , `[0-9]` ,`+` and `/`

Here is the complete symbol table

<img src="/../assets/base64-table.png" />

## Creating the symbol set

Now, first of all let's build this table using python.

Here is a very simple way of doing int it:

```python
# creating the base64 symbol set
A_Z = [chr(i) for i in range(65,91)]
a_z = [chr(i) for i in range(97,123)]
zero_nine = [chr(i) for i in range(48,58)]
additional_chars = ['+','/']
base64_symbol_table = A_Z + a_z + zero_nine + additional_chars
```

```python
print(base64_symbol_table)
['A', 'B', 'C', 'D', 'E', 'F', 'G', 'H', 'I', 'J', 'K', 'L', 'M', 'N', 'O', 'P', 'Q', 'R', 'S', 'T', 'U', 'V', 'W', 'X', 'Y', 'Z', 'a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j', 'k', 'l', 'm', 'n', 'o', 'p', 'q', 'r', 's', 't', 'u', 'v', 'w', 'x', 'y', 'z', '0', '1', '2', '3', '4', '5', '6', '7', '8', '9', '+', '/']
```

Let's break down how the above code works. I'll explain the first line and you do the math:

- `range(65,91)` creates a range of numbers from 65 to 90 (inclusive), which represent the ASCII values for the uppercase letters A to Z.
- `chr(i)` is a built-in Python function that takes an integer argument and returns the corresponding ASCII character. So `chr(65)` returns the character `'A'`, `chr(66)` returns '`B'`, and so on.
- `[chr(i) for i in range(65,91)] `is a list comprehension that applies the chr() function to each integer in the range 65 to 90, and creates a new list with the resulting characters. The end result is a list of strings containing the uppercase letters from A to Z.

## Creating the encoder

### A bit of theory

Now before attacking this part, we need to understand how the encoding part actually works.

To start, let's say we have `data = "Hi"`, how would we proceed to encode it?

- Each [ascii](https://www.asciitable.com/) character is going to be represented in its binary format.

In our case `H` is `72` and `i` is `105`.

Which gives `01001000` `01101001`

-Next, we group the bits together.

`01001000` `01101001` becomes `0100100001101001`.

- Each symbol in base64 is representend as `6 bits` not `8 bits`, so we are going to divide the above result in chunks of `6 bits`

`0100100001101001` becomes `010010` `000110` `1001`

- Now the last byte `1001` needs to have `6 bits`, we need some sort of padding.

`010010` `000110` `1001` becomes `010010` `000110` `100100`

> Note : we added 00 at the end of 1001 to complete 6 bits

- We revert back to decimal

`010010` `000110` `100100` becomes `18` `6` `36`

- Those numbers represent the indexes in our symbol table,
  where `18` corresponds to `S`...

`18` `6` `36` becomes `SGk`

- Now a base64 string needs its length to be a multiple of `4`, we accomplish that by adding `=` as a padding.

The final result is `SGk=`

Here is a little diagram a created to help you visualize the algorithm
<img src="/../assets/base64-encoding-algorithm.png"/>

### The code

Now let's get to actually coding that.

##### Step 1 :

```python
 # converting the word into its binary format
    binary = []
    for c in word:
        binary.append(bin(ord(c)).split("b")[-1].rjust(8,'0'))
```

- Here, we loop through each character `c` of our string
- we grab its ascii value with `ord()`
- we convert it to binary with `bin()`

```python
print(bin(36))
0b100100
```

`bin()` appends `0b` at the start of our desired value, we split by "b" and take the right half.

- To make sure that our `byte` is `8 bits`, we fill the start of our string with zeros to complete 8 charars using `rjust()`.

##### Step 2 :

```python
binary = "".join(binary)
```

Here, we just group what we got.

##### Step 3 :

```python
new_word = []

    while len(binary):
        new_word.append(binary[0:6])
        binary = binary[6:]
```

We form groups of 6 as explained earlier.

##### Step 4 :

```python
new_word[-1] = new_word[-1].ljust(6, '0')
```

We grab the last element, and as long as it is not 6 digits, we add zeros.

##### Step 5 :

```python
result = ""
for byte in new_word:
    #byte is a string representing the 6 bit chunk
    index = bin_to_int(byte)
    result += base64_symbol_table[index]
```

Here we loop through each byte, which is 6 bits long,
we convert it to `int` using `bin_to_int()` which is a small function a created.

The result is the index of our character in the symbol table, we access it and we store the result in `result`.

Here is the `bin_to_int()` function

```python
# binary to decimal custom function
def bin_to_int(byte):
        result = 0
        k = 1
        for i in range(0,len(byte)):
            result += int(byte[len(byte) - 1 - i]) * k
            k = k * 2
        return result
```

<img src="/../assets/bin_to_int.png" />

##### Step 6 ~ Last step :

```python
def get_multiple_of_4(num):
    while num % 4 != 0:
        num += 1
    return num

result = result.ljust(get_multiple_of_4(len(result)),'=')
return result
```

In this step, we make sure the resulted string has a length that is multiple of 4.

To do so, we grab the length of our string, we get the closest multiple of 4 greater or equal to it, and thats the length of our new string.

We use `ljust()` which adds `=` as long as our string doesnt fit the length specified in the first parameter.

## Wrapping Up

Putting all of the above together :

```python
# creating base64 table index
A_Z = [chr(i) for i in range(65,91)]
a_z = [chr(i) for i in range(97,123)]
zero_nine = [chr(i) for i in range(48,58)]
additional_chars = ['+','/']
base64_symbol_table = A_Z + a_z + zero_nine + additional_chars

# binary to decimal custom function
def bin_to_int(byte):
        #110101
        result = 0
        k = 1
        for i in range(0,len(byte)):
            result += int(byte[len(byte) - 1 - i]) * k
            k = k * 2
        return result

def b64_encode(word):

    # converting the word into its binary format
    binary = []
    for c in word:
        binary.append(bin(ord(c)).split("b")[-1].rjust(8,'0'))

    binary = "".join(binary)

    # splitting the binary format into chunks of 6 bits
    new_word = []

    while len(binary):
        new_word.append(binary[0:6])
        binary = binary[6:]

    # adding padding to the last elem
    new_word[-1] = new_word[-1].ljust(6, '0')


    # encrypting the text
    result = ""
    for byte in new_word:
        index = bin_to_int(byte)
        result += base64_symbol_table[index]

    # adding padding of "="
    def get_multiple_of_4(num):

        while num % 4 != 0:
            num += 1
        return num
    result = result.ljust(get_multiple_of_4(len(result)),'=')
    return result
```

Let's test it 😄

```python
word = "Yassir"
print(f"{word} ---> {b64_encode(word)}")
#Program stdout
Yassir ---> WWFzc2ly
```

Now we should get our word if we decode the result.

```bash
echo "WWFzc2ly" | base64 -d
Yassir
```

It works !

In the next post, we are going to create a decoder, and a main function to nicely handle the input.

See you soon

## Resources

[Base64 encoding: What sysadmins need to know](https://www.redhat.com/sysadmin/base64-encoding)

[https://medium.com/swlh/base64-encoding-algorithm-42abb929087d#:~:text=%E2%80%9CIn%20programming%2C%20Base64%20is%20a,number%20system%2C%20radix%20means%20base.](https://medium.com/swlh/base64-encoding-algorithm-42abb929087d#:~:text=%E2%80%9CIn%20programming%2C%20Base64%20is%20a,number%20system%2C%20radix%20means%20base.)
