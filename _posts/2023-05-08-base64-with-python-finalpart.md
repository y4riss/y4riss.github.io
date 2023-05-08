---
layout: post
title: Creating a base64 encoder/decoder with Python - Part 2
date: '2023-05-08 23:59:39 +0100'
categories: [Tutorials, Python scripting]
tags: [python, algorithms]
---

Hello friend,

In this post, we are going to continue from where we left off, please check the [first part ](https://y4riss.github.io/posts/base64-with-python/)if you haven't.

In the previous post, we built a base64 encoder. Now we are going to be building a decoder, working our way backwards.

## Creating the decoder

### Theory
Here is how it is going to work :

<img src="/../assets/base64-encoding-algorithm_decoder.png" />

### Implementation
Here is the complete function, respecting the steps above.


```python
def b64_decode(word):

    #stripping away '='
    word = word.split("=")[0]

    #finding the index of each character in b64_index_table
    indexes = []
    for c in word:
        indexes.append(base64_symbol_table.index(c))

    # index to binary ( 6 bits)
    binary = []
    for i in indexes:
        binary.append(bin(i).split("b")[-1].rjust(6,'0'))

    # groupping the bits together
    binary = "".join(binary) # this is a string


    # forming group of 8 bits
    new_word = []
    while len(binary):
        new_word.append(binary[0:8])
        binary = binary[8:]

    # transform each byte to its ascii representation
    result = ""
    for c in new_word:
        result += chr(bin_to_int(c)) #you can use int(c,2)
        # instead of bin_to_int(c)
    return result
```

```python
word = "SGk="
print(b64_decode(word))
```

```bash
Program stdout
Hi
```

### Adding a main function

All working as expected, now let's add a main function to hundle user input.


Here is the expected behaviour:

```bash
python b64.py
[/_\] Usage : python b64.py <string> (-d | -e )
```

```bash
python b64.py Hi -e
[+] Hi ---> SGk=

python b64.py SGk= -d
[+] SGk= ---> Hi
```

You can have fun implementing your own functionalities, here is a simple one using the `sys` module.

```python
if __name__ == '__main__':
    
    len_args = len(sys.argv)
    if (len_args != 3):
        print(f"[] Usage : python {sys.argv[0]} <string> (-d | -e )")
        exit(0)
    option = sys.argv[2]
    if not (option == '-d' or option == '-e'):
        print(f"[] Usage : python {sys.argv[0]} <string> (-d | -e )")
        exit(0)

    word = sys.argv[1]
    if (option == '-d'):
        print(f"[+] {word} ---> {b64_decode(word)}")
    else:
        print(f"[+] {word} ---> {b64_encode(word)}")
```


## Wrapping up

Putting all of it together, we get something like this

```python
import sys

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

def b64_decode(word):

    #stripping away '='
    word = word.split("=")[0]

    #finding the index of each character in b64_index_table
    indexes = []
    for c in word:
        indexes.append(base64_symbol_table.index(c))

    # index to binary ( 6 bits)
    binary = []
    for i in indexes:
        binary.append(bin(i).split("b")[-1].rjust(6,'0'))

    # groupping the bits together
    binary = "".join(binary) # this is a string


    # forming group of 8 bits
    new_word = []
    while len(binary):
        new_word.append(binary[0:8])
        binary = binary[8:]

    # transform each byte to its ascii representation
    result = ""
    for c in new_word:
        result += chr(bin_to_int(c)) #you can use int(c,2)
        # instead of bin_to_int(c)
    return result

if __name__ == '__main__':
    
    len_args = len(sys.argv)
    if (len_args != 3):
        print(f"[/_\\] Usage : python {sys.argv[0]} <string> (-d | -e )")
        exit(0)
    option = sys.argv[2]
    if not (option == '-d' or option == '-e'):
        print(f"[/_\\] Usage : python {sys.argv[0]} <string> (-d | -e )")
        exit(0)

    word = sys.argv[1]
    if (option == '-d'):
        print(f"[+] {word} ---> {b64_decode(word)}")
    else:
        print(f"[+] {word} ---> {b64_encode(word)}")
```


That's it for this article, hope you learned something !

