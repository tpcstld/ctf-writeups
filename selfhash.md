# Introspective CRC

## Problem

> We saw https://shells.aachen.ccc.de/~spq/md5.gif and were inspired.

> Challenge running at selfhash.ctfcompetition.com:1337

## Overview

In this challenge, we are given the following domain and port.

```
selfhash.ctfcompetition.com:1337
```

As is the norm in this competition, let's try connecting to it via TCP.

```
$ nc selfhash.ctfcompetition.com 1337
Give me some data:
```

Seems like this server wants some data! Let's type in a few characters.

```
Give me some data: asdf

Check failed.
Expected:
    len(data) == 82
Was:
    4
```

And the connection closes. Judging by how `asdf` is 4 characters long, seems
like our input becomes the `data` variable in this (presumably) Python server.

Also, it seems like they want exactly 82 characters! We can provide that.

```
$ nc selfhash.ctfcompetition.com 1337
Give me some data: 1234567890123456789012345678901234567890123456789012345678901234567890123456789012

Check failed.
Expected:
    set(data) <= set("01")
Was:
    set(['1', '0', '3', '2', '5', '4', '7', '6', '9', '8'])
```

Oh, they only accept ones and zeros.

```
$ nc selfhash.ctfcompetition.com 1337
Give me some data: 0000000000000000000000000000000000000000000000000000000000000000000000000000000000

Check failed.
Expected:
    crc_82_darc(data) == int(data, 2)
Was:
    1021102219365466010738322L
    0
```

Interesting!

Some Googling reveals that `crc_82_darc` most likely corresponds
to a function in [pwntools](http://docs.pwntools.com/en/stable/util/crc.html#pwnlib.util.crc.crc_82_darc).
Let's verify that:

```
$ pip install pwntools
...
$ python
Python 2.7.10 (default, Feb  6 2017, 23:53:20)
[GCC 4.2.1 Compatible Apple LLVM 8.0.0 (clang-800.0.34)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> from pwnlib.util.crc import crc_82_darc
>>> crc_82_darc('0000000000000000000000000000000000000000000000000000000000000000000000000000000000')
1021102219365466010738322L
>>> crc_82_darc('0' * 82) # This is the same.
1021102219365466010738322L
```

Now we're sure.

Reading the documentation of pwntools, we find that `crc_82_darc` computes a [CRC](https://en.wikipedia.org/wiki/Cyclic_redundancy_check) from `data` with some constants.
Our goal is to find a value of `data` such that the CRC of `data` matches its base-2, numeric interpretation.
Reading the Wikipedia article on CRC, we learn the following:

1. CRC treats the input as bits, so we need to find the binary representations of ASCII `1` and `0`.

```python
>>> bin(ord('0'))
'0b110000'
>>> bin(ord('1'))
'0b110001'
```

2. CRC is linear, or formally `crc(x xor y) = crc(x) xor crc(y)`.

Linearity seems like a very interesting property! Let's try it out:

```
>>> crc_82_darc('0' * 81 + '1')
1530552403008155235290126L
>>> crc_82_darc('0' * 81 + '1') ^ crc_82_darc('0' * 82)
1946231125162404815852188L
```

We use the fact that `'0' xor '1' = 0b00000001`, since their ASCII representations differ only in the last bit.

```
>>> chr(ord('0') ^ ord('0'))
'\x00'
>>> chr(ord('0') ^ ord('1'))
'\x01'
>>> crc_82_darc('\x00' * 81 + '\x01')
1946231125162404815852188L
```

Cool, Wikipedia isn't lying.

Since CRC is linear, something like `crc_82_darc("1010")` is equivalent to `crc_82_darc("\x01\x00\x01\x00") xor crc_82_darc("0000")`. This is because `"1010" == "\x01\x00\x01\x00" xor "0000"`.

We're trying to find `data` such that `crc_82_darc(data) = int(data, 2)`. `data` is an ASCII string, but now we can interpret the problem as finding a `\x00`-`\x01` string `x` such that `crc_82_darc(x) xor crc_82_darc('0' * 82) = to_num(x)`. `to_num` means that you interpret the string as a binary number, where `'\x00' == 0` and `'x01' == 1`.

To find the `x` that satisfies the condition, we turn to linear algebra!

## Solution

In this environment, we're going to work in [GF(2)](https://en.wikipedia.org/wiki/GF(2)) so that we can represent XOR as addition. We represent `x` as 82-element vector, where the i-th element in `x` is 1 if the i-th character is `\x01`, and 0 otherwise. Also, we define
`to_num(x) = x` since we no longer need to "interpret" the string.

`crc_82_darc('0' * 82)` returns a 82-bit constant, so let's also model it as a 82-element vector named `b`. Specifically, the i-th element of `b` is 1 if and only if the i-th bit in `crc_82_darc('0' * 82)` is also 1.

Lastly, we need to transform `crc_82_darc(x)`. Recall that we can write the CRC of any `\x00`-`\x01` string as a linear combination (i.e. XORs) of the CRCs of the basic strings (i.e.  strings with only 1 `\x01`). Therefore, we can construct a matrix 82x82 matrix `A` such that the i-th column is populated with the CRC of the `\x00`-`\x01` basic string that whose `\x01` character is in the i-th position. We can see that `A * x = crc_82_darc(x)`.

Now, we have turned the expression `crc_82_darc(x) xor crc_82_darc('0' * 82) = to_num(x)` into the linear equation `A * x + b = x`.
We can rewrite this as:

```
b = x - A * x
b = I * x - A * x
b = (I - A) * x
```

We can solve this equation using a linear algebra solver, and we decided to use [sage](http://doc.sagemath.org/html/en/tutorial/tour_algebra.html) since it supports GF(2). Below is the code for our solution:

``` python
#!/usr/bin/sage -python
import numpy as np
import sage.all

from pwnlib.util.crc import crc_82_darc

def binary(x):
    """
    Convert a number to its binary representation as a list of 82 0's and 1's.
    >>> binary(0b10110)[-5:]
    [1, 0, 1, 1, 0]
    >>> len(binary(1021102219365466010738322L))
    82
    """
    return map(int, '{0:082b}'.format(x))

def transpose(A):
    """
    Return the transpose of a rectangular matrix.
    >>> transpose([[1, 2, 3], [4, 5, 6]])
    [[1, 4], [2, 5], [3, 6]]
    """
    return map(list, zip(*A))

def main():
    # Construct b
    b = binary(crc_82_darc('0' * 82))

    # Construct A
    A = []
    for i in xrange(82):
        string_with_1_at_i = '\x00' * i + '\x01' + '\x00' * (82 - i - 1)
        A.append(binary(crc_82_darc(string_with_1_at_i)))

    # Transpose A
    A = transpose(A)

    # Compute (I - A) (subtraction is XOR  in GF2)
    for i in xrange(82):
        A[i][i] = A[i][i] ^ 1
    
    # Solve for x where Ax = b
    GF2 = sage.all.GF(2)
    A = sage.all.matrix(GF2, A)
    b = sage.all.vector(GF2, b)

    answer = A.solve_right(b)
    print 'Answer:', ''.join(map(str, answer))

if __name__ == '__main__':
    main()
```

```
$ sage -python crc.py
Answer: 1010010010111000110111101011101001101011011000010000100001011100101001001100000000
$ nc selfhash.ctfcompetition.com 1337
Give me some data: 1010010010111000110111101011101001101011011000010000100001011100101001001100000000

CTF{i-hope-you-like-linear-algebra}
```
