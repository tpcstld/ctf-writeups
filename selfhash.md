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
To get past this assert, we need to make the output of the checksum collide with
the binary representation of our data.
