# ASN0

ASN0 is an attempt at a degenerate encoding: strings of octets and string of strings of octets, ... Here's the high level schema[^abnf]:

```abnf
message  = OCTET* / message*
```

Fundamentally the goal is **not** to encode types such as integers etc but to allow an incoming octet stream to be **tokenised** and outputted into a tree of sub strings. How those outputted octet strings are mapped to types is **outside** the scope of this specification.


## High level example

Let's take a simple message in JSON[^base64]:

```json
{ "Hello": [ "World", 3, true, "AQIDBA==" ] }
```
The simplest mapping of types it to keep all the value data the same but rewrap that data as an ASN0 message[^plist]:

```source.plist
( "Hello", ( "World", "3", "T", <01020304> ) )
```


or just using hex data values:

```source.plist
( <48656C6C 6F>, ( <576F726C 64>, <33>, <54>, <01020304> ) )
```

```source.plist
<B4>
    <85> <4865 6C6C6F>
    <AD>
        <85><576F72 6C64>
        <33>
        <54>
        <84> <01020304>
```

Now we have something in the correct form, we can encode it using the wire format ...

## Wire format

### Overview

byte<br>(in binary) | byte<br>(in hex) | Description
------------------- | ---------------- | -----------
0xxx xxxx           | 0x00 - 0x7f      | single octet where MSB is 0
10xx xxxx           | 0x80 - 0x9f      | octet array
11xx xxxx           | 0xa0 - 0xff      | message array

### Arrays


Layout<br>(in binary) | Description
--------------------- | -----------
0x xxxx | plain array (octet count 0-31)
10 1000 | plain array (octet count in next byte)
10 1001 | plain array (octet count in next 2 bytes)
10 1010 | plain array (octet count in next 3 bytes)
10 1011 | plain array (octet count in next 8 bytes)
10 1100 | **reserved &times; 1**
10 1101 | gzip compressed array (octet count in next 2 bytes)
10 1110 | gzip compressed array (octet count in next 3 bytes)
10 1111 | gzip compressed array (octet count in next 8 bytes)
11 xxxx | **reserved &times; 16**


### Notes

1. No single length byte option exists for gzip compressed octet message arrays as gzip just isn't worth it below for blobs of 1024 bytes of less.
2. All sizes are little endian
3. Single 7 bit [ASCII](https://en.wikipedia.org/wiki/ASCII) values are encoded in without extra space usage


[^abnf]: [Augmented Backus–Naur form](https://en.wikipedia.org/wiki/Augmented_Backus–Naur_form)


[^base64]: JSON does not allow octet strings so octet array data is encoded as base64.

[^plist]: Shown here as a [NeXTSTEP Property list](https://en.wikipedia.org/wiki/Property_list#NeXTSTEP) as it will accept octet array data represented as hexadecimal codes in ASCII, message arrays, and strings.
