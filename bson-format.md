BSON Data Format
================

It is can be useful to be able to manipulate BSON entities without having to
decode them into objects.  Luckily, this happens to be pretty easy.

BSON Objects
------------

A BSON object is a byte-counted list of concatenated binary records, each
record being a typed, byte-counted value.

BSON Object:

- 4B total length
- length-4-1 B for list of records
- 1B zero byte NUL terminator

Each Record:

Each bson record is the concatenated binary fields of `[ length, id, name, value ]`

- 4B total length
- 1B type ie (see list below)
- *B field name (variable length, NUL terminated)
- 1B zero byte as name terminator NUL
- *B value (variable, based on type)
  - 0B null
  - 4B int32
  - 8B ieee float
  - 1B boolean
  - 4B len + len-1 B string + 1B NUL terminator
  - 4B len + len-5 B object of values + 1B NUL terminator
  - 4B len + len-5 B array of values + 1B NUL terminator (names are numbers)

It so happens that the layout of a BSON object is the same as the value part
of an object type record.

### Examples

        {p: null}

        [ 0c 00 00 00  07 00 00 00 0a 70 00  00 ]
          ^ 12B object, total, including NUL object terminator
          ^ 4B BSON object length field (12)
                                             ^ BSON object NUL byte terminator
                       ^ 7B null object length = 4B length + 1B type + 1B name + 1B NUL
                       ^ 4B entity length field (7)
                                   ^ entity type "NULL"
                                      ^ entity name "p"
                                         ^ entity name NUL byte terminator


        { p: null, a: 1.5 }

        [ 13 00 00 00 0a 70 00 01 61 00 00 00 00 00 00 00 f8 3f 00 ]
          ^ 19B bson total, including NUL object terminator
                                                                ^ NUL bson terminator
          ^ 4B BSON object length (19)
                      ^ field 1 type, 0x0A, null
                         ^ field 1 name, 'p' 0x70
                            ^ field 1 name NUL terminator byte
                               ^ field 2 type, 0x01 ieee floating-point
                                  ^ field 2 name, 'a' 0x61
                                     ^ field 2 name NUL terminator
                                        ^ 8 bytes of floating-point data
                                                             ^ last byte of float
                                                                ^ NUL bson terminator

Mongodump
---------

Mongodump outputs the documents in the collection as a concatenated list of
bson objects.  Reading the mongodump output file as a byte stream, it is
possible to separate the first object by reading the length from the first
four bytes, to separate the second object by reading the first four bytes
following the first object, and so on the rest of the objects.


BSON Types
----------

        0x01            64-bit IEEE 754 floating-point, 8 bytes
        0x02            NUL-terminated string, strlen+1 bytes
        0x03            NUL-terminated document
        0x04            array
        0x05            binary
        0x06            (deprecated, undefined)
        0x07            ObjectID, 12 bytes (4B be time, 3B system, 2B pid, 3B be seq)
        0x08            Boolean 1 byte
        0x09            datetime, 64-bit little-endian millisecond js timestamp
        0x0A            null, 0 bytes
        0x0B            RegExp, 2 NUL-terminated strings "ABC" and "im" of /ABC/im
        0x0C            (deprecated, db ref)
        0x0D            code
        0x0E            symbol
        0x0F            code with scope
        0x10            32-bit signed little-endian integer
        0x11            timestamp
        0x12            64-bit signed integer
        0xFF            min key
        0x7F            max key
