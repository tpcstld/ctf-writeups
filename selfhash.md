To start this challenge, we are given the following domain and port.

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

2. CRC is linear, which means that `crc(xor(x, y)) = xor(crc(x), crc(y))`.

(2) seems like a very interesting property! Let's try it out:

```
>>> crc_82_darc('0' * 81 + '1')
1530552403008155235290126L
>>> crc_82_darc('0' * 81 + '1') ^ crc_82_darc('0' * 82)
1946231125162404815852188L
```

We use the fact that `xor('0', '1') = 0b00000001`, since their ASCII representations differ only in the last bit.

```
>>> chr(ord('0') ^ ord('0'))
'\x00'
>>> chr(ord('0') ^ ord('1'))
'\x01'
>>> crc_82_darc('\x00' * 81 + '\x01')
1946231125162404815852188L
```

Cool, the property seems to hold.

Observe that any binary number can be written as the `xor` of its bits. For example, `1010` can be written as `xor(1000, 0010)`. We can also `xor` it with `0000` as well. `1010 = xor(1000, 0010, 0000)`.

ASCII strings of zeros and ones also have a similar property. For example, the string
`"1010" == xor("0000", "\x01\x00\x00\x00", "\x00\x00\x01\x00")`.

Here's the relationship between the binary and ASCII representations of a number.

`data`  |ASCII representation                       |`int(data, 2)`  
--------|-------------------------------------------|--------------
`"0000"`|0b0011000<span style="color:red">0</span> 0b0011000<span style="color:red">0</span> 0b0011000<span style="color:red">0</span> 0b0011000<span style="color:red">0</span>| 0b<span style="color:red">0</span><span style="color:red">0</span><span style="color:red">0</span><span style="color:red">0</span>
`"1010"`|0b00110001 0b00110000 0b00110001 0b00110000| 0b1010
`"0001"`|0b00110000 0b00110000 0b00110000 0b00110001| 0b0000


