# ASN0

ASN0 is an attempt at a degenerate encoding: strings of octets and string of strings of octets, ... Here's the high level schema[^abnf]:

```abnf
message  = OCTET* / message* / NULL
```

Fundamentally the goal is **not** to encode types such as integers etc but to allow an incoming octet stream to be **tokenised** and outputted into a tree of sub strings. How those outputted octet strings are mapped to types is **outside** the scope of this specification.


## High level example

Let's take a simple message in JSON[^base64]:

```json
{ "Hello": [ "World", 3, true, "AQIDBA==" ] }
```
The simplest mapping of types it to keep all the value data the same but rewrap that data as an ASN0 message[^plist]:

```source.plist
( "Hello", ( "World", "3", "true", <01020304> ) )
```


or just using hex data values:

```source.plist
( <48656C6C 6F>, ( <576F726C 64>, <33>, <74727565>, <01020304> ) )
```

Now we have something in the correct form, we can encode it using the wire format ...

## Wire format

### Overview

byte<br>(in binary) | byte<br>(in hex) | Description
------------------- | ---------------- | -----------
0xxx xxxx           | 0x00 - 0x7f      | single octet where MSB is 0
10xx xxxx           | 0x80 - 0x9f      | octet array
11xx xxxx           | 0xc0 - 0xff      | message array
1010 1100           | 0xAC             | null

### Arrays

Layout<br>(in binary) | Description
--------------------- | -----------
0x xxxx | plain array (octet count 0-31)
10 1000 | plain array (octet count in next byte) `value + 32`
10 1001 | plain array (octet count in next 2 bytes) `value + 288
10 1010 | plain array (octet count in next 3 bytes) `value + 65_824`
10 1011 | plain array (octet count in next 8 bytes) `value + 16_843_040`
10 1100 | **reserved &times; 1**
10 1101 | gzip compressed array (octet count in next 2 bytes)
10 1110 | gzip compressed array (octet count in next 3 bytes)
10 1111 | gzip compressed array (octet count in next 8 bytes)
11 xxxx | **reserved &times; 16**

### Notes

1. All sizes are little endian
2. Single 7 bit [ASCII](https://en.wikipedia.org/wiki/ASCII) values are encoded in without extra space usage
3. No single length byte option exists for gzip compressed octet message arrays as gzip just isn't worth it below blobs of 1024 bytes of less.


Diagnostic | Encoded | Wire
---------- | ------- | ----
0 | 00 | 00
1 | 02 | 02
10 | 14 | 14
23 | 2E | 2E
24 | 30 | 30
25 | 32 | 32
100 | C8 | 81C8
1000 | D007 | 82D007
1000000 | 80841E | 8380841E
1000000000000 | 00204AA9 D101 | 8600204A A9D101
18446744073709551615 | FFFFFFFF FFFFFFFF | 88FFFFFF FFFFFFFF FF
-1 | 01 | 01
-10 | 13 | 13
-100 | C7 | 81C7
-1000 | CF07 | 82CF07
1.0 | 312E30 | 83312E30
1.1 | 312E31 | 83312E31
"IETF" | 49455446 | 84494554 46
"" |  | 80
["a", {"b": "c"}] | (61, (62, 63)) | C461C262 63
{"d": "D", "b": "B", "a": "A", "c": "C", "e": "E"} | (61, 41, 64, 44, 63, 43, 65, 45, 62, 42) | CA614164 44634365 456242
true | 54 | 54
false | 46 | 46
null | NULL | AC
[1, [2, 3]] | (02, (04, 06)) | C402C204 06
[1, [2, 3], [4, 5]] | (02, (04, 06), (08, 0A)) | C702C204 06C2080A
[1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25] | (02, 04, 06, 08, 0A, 0C, 0E, 10, 12, 14, 16, 18, 1A, 1C, 1E, 20, 22, 24, 26, 28, 2A, 2C, 2E, 30, 32) | D9020406 080A0C0E 10121416 181A1C1E 20222426 282A2C2E 3032
{"b": [2, 3], "a": 1} | (62, (04, 06), 61, 02) | C662C204 066102

[^abnf]: [Augmented Backus–Naur form](https://en.wikipedia.org/wiki/Augmented_Backus–Naur_form)


[^base64]: JSON does not allow octet strings so octet array data is encoded as base64.

[^plist]: Shown here as a [NeXTSTEP Property list](https://en.wikipedia.org/wiki/Property_list#NeXTSTEP) as it will accept octet array data represented as hexadecimal codes in ASCII, message arrays, and strings.
