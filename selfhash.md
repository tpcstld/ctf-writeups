In this challenge, we are given the following domain and port.

```
selfhash.ctfcompetition.com:1337
```

As is the norm in this CTF competition, let's try connecting to it via TCP.

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

Some Googling reveals that the `crc_82_darc` function most likely corresponds
to the corresponding function in [pwntools](http://docs.pwntools.com/en/stable/util/crc.html).
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

Reading the documentation of pwntools, we find that `crc_82_darc` computes a [CRC](https://en.wikipedia.org/wiki/Cyclic_redundancy_check) of `data` with some constants.
Our goal is to find a value of `data` such that the CRC of `data` matches its base-2, numeric interpretation.
Reading the Wikipedia article on CRC, we learn the following:

1. CRC treats the input as bits, so we need to find the binary representations of ASCII `1` and `0`.

```python
>>> bin(ord('0'))
'0b110000'
>>> bin(ord('1'))
'0b110001'
```

2. CRC is linear, which means that `crc(x xor y) = crc(x) xor crc(y)`.

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

In this enviroment, we're going to work in [GF(2)](https://en.wikipedia.org/wiki/GF(2)) so that we can represent XOR as addition. We represent `x` as 82-element vector, where the i-th element in `x` is 1 if the i-th character is `\x01`, and 0 otherwise. Also, we define
`to_num(x) = x` since we no longer need to "interpret" the string.

`crc_82_data('0' * 82)` returns a 82-bit constant, so let's also model it as a 82-element vector named `b`. Specifically, the i-th element of `b` is 1 if and only if the i-th bit in `crc_82_data('0' * 82)` is also 1.

Lastly, we need to transform `crc_82_data(x)`. Recall that we can write the CRC of any `\x00`-`\x01` string as a linear combination (i.e. XORs) of the CRCs of the basic strings (i.e.  strings with only 1 `\x01`). Therefore, we can construct a matrix 82x82 matrix `A` such that the i-th column is populated with the CRC of the `\x00`-`\x01` basic string that whose `\x01` character is in the i-th position. We can see that `A * x = crc_82_data(x)`.

Now, we have turned the expression `crc_82_darc(x) xor crc_82_data('0' * 82) = to_num(x)` into the linear equation `Ax + b = x`.
We can rewrite this as:

```
b = x - Ax
b = Ix - Ax
b = (I - A)x
```

We can solve this equation using a linear algebra solver, and we decided to use [sage](http://doc.sagemath.org/html/en/tutorial/tour_algebra.html) since it supports GF(2). Below is the code for our solution:

``` python
import numpy as np
import sage.all

from pwnlib.util.crc import crc_82_darc

def binary_recursive(n):
    return n>0 and [n&1]+binary_recursive(n>>1) or []

def binary(n):
    answer = binary_recursive(n)
    while len(answer) < 82:
        answer.append(0)

    return answer[::-1]

def main():
    # Construct b
    b = binary(crc_82_darc('0' * 82))

    # Construct A
    A = []
    for x in xrange(82):
        string = '\x00' * (x) + '\x01' + '\x00' * (82 - x - 1)
        crcs.append(crc_82_darc(string))

    A = [binary(n) for n in crcs]
    A = zip(*crcs)
    A = [list(x) for x in crcs]

    # (I - A)
    for x in xrange(82):
        A[x][x] = A[x][x] ^ 1
    
    # Solve
    GF2 = sage.all.GF(2)
    A = sage.all.matrix(GF2, A)
    b = sage.all.vector(GF2, b)

    answer = np.array(A.solve_right(b), dtype=np.uint8)
    print 'Answer:', answer

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

<!--

For example, `crc_82_darc('\x01\x00\x01\x01') == 1*crc_82_darc('\x01\x00\x00\x00') + 0*crc_82_darc('x00\x01\x00\x00') + 1*crc_82_darc('\x00\x00\x01\x00') + 0*crc_82_darc('\x00\x00\x00\x01')`. (Remember that addition is the same as XOR now.)

Let's instead find a `\x00`-`\x01` string such that
`crc_82_darc(x) xor crc_82_darc("0" * 82) == to_num(x)`

The challenge asks us to find `data` such that `crc_82_darc(data) = int(data, 2)`. Instead of finding this directly, we'll instead find a string `x` of `\x00` and `\x01` such that `crc_82_darc(x) xor crc_82_darc("0" * 82) == x`

Observe that any binary number can be written as the `xor` of its bits. For example, `1010` can be written as `xor(1000, 0010)`. We can also `xor` it with `0000` as well. `1010 = xor(1000, 0010, 0000)`.

ASCII strings of zeros and ones also have a similar property. For example, the string
`"1010" == xor("0000", "\x01\x00\x00\x00", "\x00\x00\x01\x00")`.

Here's the relationship between the binary and ASCII representations of a number.

`data`  |ASCII representation                       |`int(data, 2)`  
--------|-------------------------------------------|--------------
`"0000"`|0b0011000<span style="color:red">0</span> 0b0011000<span style="color:red">0</span> 0b0011000<span style="color:red">0</span> 0b0011000<span style="color:red">0</span>| 0b<span style="color:red">0</span><span style="color:red">0</span><span style="color:red">0</span><span style="color:red">0</span>
`"1010"`|0b00110001 0b00110000 0b00110001 0b00110000| 0b1010
`"0001"`|0b00110000 0b00110000 0b00110000 0b00110001| 0b0000

Now, given a CRC, we can use its linearity property to break it up into a combination of XORs just like how we can break up
ASCII strings. For example,
`crc_82_darc("1010") == xor(crc_82_darc("0000"), crc_82_darc("\x01\x00\x00\x00"), crc_82_darc("\x00\x00\x01\x00")`.
Similarly, recall that `0b1010 == xor(0b0000, 0b1000, 0b0010)`.

To solve this problem, we need the CRC of the ASCII representation to be the CRC. We will model this as a system of equations and solve for the CRC.

To do this, we need to mathematically represent:

* a binary number
* the XOR of binary numbers
* the `crc_82_darc` of a binary number
* the equality of two binary numbers





we need to find a set `(0000, 1000, 0010)` such that `xor(crc_82_darc("0000"), ...) = xor(0b0000, ...)`

we need both sets of XORs to represent the same number. This will get us a string So, we can consider this problem as a system of equations -->
