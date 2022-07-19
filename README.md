# ASN0

ASN0 is an attempt at a degenerate encoding. Here's the high level schema[^abnf]:

```abnf
message = OCTET* / message* / NULL
```

```swift
enum Value {
    case message([Value])
    case octets([UInt8])
    case null
}
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
1xxx xxxx           | 0x80 - 0xfb      | octet array with length 0-124
1111 1100           | 0xfc             | octet array continuation
1111 1101           | 0xfd             | null
1111 1110           | 0xfe             | begin message array
1111 1111           | 0xff             | end message array

### Notes

1. Single 7 bit [ASCII](https://en.wikipedia.org/wiki/ASCII) values are encoded in without extra space usage

### Examples

Note that all value to octet encodings are **NOT** no part of this specification


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
["a", {"b": "c"}] | (61, (62, 63)) | FE61FE62 63FFFF
{"b": "B", "a": "A", "d": "D", "e": "E", "c": "C"} | (65, 45, 63, 43, 64, 44, 61, 41, 62, 42) | FE654563 43644461 416242FF
true | 74727565 | 84747275 65
false | 66616C73 65 | 8566616C 7365
null | NULL | FD
[1, [2, 3]] | (02, (04, 06)) | FE02FE04 06FFFF
[1, [2, 3], [4, 5]] | (02, (04, 06), (08, 0A)) | FE02FE04 06FFFE08 0AFFFF
[1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25] | (02, 04, 06, 08, 0A, 0C, 0E, 10, 12, 14, 16, 18, 1A, 1C, 1E, 20, 22, 24, 26, 28, 2A, 2C, 2E, 30, 32) | FE020406 080A0C0E 10121416 181A1C1E 20222426 282A2C2E 3032FF
{"b": [2, 3], "a": 1} | (62, (04, 06), 61, 02) | FE62FE04 06FF6102 FF

[^abnf]: [Augmented Backus–Naur form](https://en.wikipedia.org/wiki/Augmented_Backus–Naur_form)


[^base64]: JSON does not allow octet strings so octet array data is encoded as base64.

[^plist]: Shown here as a [NeXTSTEP Property list](https://en.wikipedia.org/wiki/Property_list#NeXTSTEP) as it will accept octet array data represented as hexadecimal codes in ASCII, message arrays, and strings.
