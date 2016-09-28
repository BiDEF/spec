# Pack-Man
Packed Message Notation

Design Goals
------------
* Rich user defined types (for expressive self-describing data)
* Reduce (re)occurance of full types (for small message size)
* Stable for all future (no versioning, no build in special purpose types that might be "standard" today)
* Any possible message is formally correct by design (to not have to validate on a non-semantic level)
* Statically typed (type once) and dynamically typed (type for each value) data can be mixed freely


Base Types
----------

The format has 8 base types:

1. `symbol`
2. `bits`
3. `uint`
4. `int`
5. `char`
6. `decimal`
7. `array`
8. `record`

* A _boolean_ equals `bits` with 1 byte using 1 bit.
* _Binary_ would be an `array` of `uint` of 1 byte width. 
* A _text_ or _string_ is an `array` of `char`s or in case of UTF-8 _binary_.
* An _object_ is a `record` with tagged field types for the field names.
* A _map_ could be a `record` with a keys `array` and values `array`.
* A _date_ could be a `uint` with 8 bytes width or a `record` of day, month and year.
* A _fraction_ could be a `record` of an `int` numerator and an `uint` denominator.


Encoding
--------
Data is described by a pair of a type followed by a value.

		+-----------+===========+
		|   type    |    data   |
		+-----------+===========+

The first byte of such a pair encodes what type it is.
Depending on the type the type has additional bytes
before the value bytes follow. 
The count of value bytes is either a fix number given
by the type or encoded explicitly as part of the type.


#### Symbols

		+-----------+
		|0 000 ssss | (see table of symbols below)
		+-----------+
		
  Symbols are type and value in a single byte.

 `ssss` | Symbol    | `ssss`| Symbol
 ------ | --------- |------ | --------- |
 `0000` | numeric 0 | `1000` | numeric 8
 `0001` | numeric 1 | `1001` | numeric 9
 `0010` | numeric 2 | `1010` | `null`
 `0011` | numeric 3 | `1011` | `undefined`
 `0100` | numeric 4 | `1100` | `false`
 `0101` | numeric 5 | `1101` | `true`
 `0110` | numeric 6 | `1110` | `dynamic`
 `0111` | numeric 7 | `1111` | `tag` 
 
  The symbols `null`, `undefined`, `true` and `false` are only used in connection with `dynamic` or `v`arying typed values.
 
  The `dynamic` type is used for array element types or record field types to indicate that the type is explicitly encoded for each value. 
  
  When `dynamic` is used as a top level type (a type not further refined later) the value is by convention one byte long.
  It is up to the receiver to make sense of it.

  The `tag` symbol is followed by the type-value pair for the tag data.

		+-----------+-------------    --+===========    ==+
		|0 000 1111 | tag's type / * /  | tag data / * /  |
		+-----------+------------    ---+==========    ===+

 * Tags precede the type they are attached to. 
 * Multiple tags follow each other before the type they all are attached to.
 * Tags can be attached to any type including array element and record field types as well as dynamic types.

#### Bits

		+-----------+------------+=======        ==+
		|0 001 v bbb| data bytes | data / 1-255 /  |
		+-----------+------------+======        ===+

The bits `bbb` (0-7) give the number of used bits in the last byte: 1-8 (1 is always added). Bits are used LSB to MSB.

* `bool` is equal to `bits(1)` what i is encoded as `00010001 0000001`. 
* the `bits` value for _true_ is `00000001`.
* the `bits` value for _false_ is `00000000`.

#### Unsigned Integers

		+-----------+=======      ==+
		|0 010 v www| data / 1-8 /  |
		+-----------+======      ===+

The bits `www` (0-7) give the _width_ of the value: 1-8 bytes (1 is always added)

#### Signed Integers

		+-----------+=======      ==+
		|0 011 v www| data / 1-8 /  |
		+-----------+======      ===+

The bits `www` (0-7) give the _width_ of the value: 1-8 bytes (1 is always added)

#### Characters

		+-----------+=======      ==+
		|0 100 v www| data / 1-8 /  |
		+-----------+======      ===+

The bits `www` (0-7) give the _width_ of the value: 1-8 bytes (1 is always added)

#### Decimals

		+-----------+==============      ==+==========+
		|0 101 v www| coefficient / 0-7 /  | exponent |
		+-----------+=============      ===+==========+

The bits `www` (0-7) give the _width_ of the value: 1-8 bytes (1 is always added)
The exponent always uses the last byte. The coefficient uses 0-7 bytes. If the coefficient has no bytes it is 1. 

#### Arrays

		+-----------+---------------    --+=========      ==+===========    ==+
		|0 110 v lll| element type / * /  | length / 1-8 /  | elements / * /  |
		+-----------+--------------    ---+========      ===+==========    ===+

  The bits `lll` (0-7) give the amount of bytes for the `length` information, 1-8 bytes (1 is always added).

  The array values will only have an explicitly stated type if the array is `v`arying or its `element type` is `dynamic`.
  Otherwise it is a sequence of values only.
  
  If a `v`arying array type is predefined and later referenced the element type is stated explicitly for each value.
  This allows to change details of the element type like the width or make use of `null` elements.

#### Records

		+-----------+---------      --+--------------    --+=======    ==+
		|0 111 v fff| fields / 1-8 /  | field types / * /  | data / * /  |
		+-----------+--------      ---+-------------    ---+======    ===+

  The bits `fff` (0-7) give the amount of bytes that encode the `fields` count, 1-8 bytes (1 is always added).

  When the record type is predefined and later referenced only those fields will have an explicit type that are `v`arying or `dynamic`. 
  If the whole record is `v`arying the field types are not part of the type definition but repeated before each value.
  This is used to as a form of _union_ type.
  
  To name fields tags are attached to the field types.

#### Type Reference

		+-----------+
		|1 rrrrrrr  | 
		+-----------+

  Uses the type given by the 7 bit id `rrrrrrr`. If the referenced type is `v`arying or has partial types that are `dynamic` the actual values for these types are stated before the value.

#### Type Definition

		+-----------+-----------+============    ==+
		|1 1111111  | 0 rrrrrrr |  type def / * /  |
		+-----------+-----------+===========    ===+

  Defines a type with the given 7-bit id for the **duration of the conversaion**.
  A type might however be redefined in any message during a conversaion (including the same message that origianlly defined it first). **The defined type has no value**.
  
  The ids `1xxxxxxx` or `x1111111` should not be used as they cannot be referenced.
  It is however not illegal to do so. The receiver is free to ignore such definitions or do some custom interpretation that might be meaningful between particular senders and receivers.


Messages
--------
A message is a sequence of typed values. A values type always preceeds the value bytes.

Typically a message starts by some type definitions followed by one or more typed values that reference to the previously defined types. But type definitions and usual typed values can be mixed at will.
As the "type table" (list of types as defined most recently) is valid for the entire conversation (message exchange) subsequent messages can refer to types defined in previous messages. Therefor both sender and receiver have to keep track of
of the type table as communicated to the other side.

Dynamic Typing
--------------
Most _"typed"_ binary formats are _dynamically_ typed. 
This is to say that each value has an explicit type tag that can be any of the possible types.
The types of values are encountered right before the value. 
No prediction are possible, no guarantees made about any of subsequent types or values.
This is even true for values that are "members" like array elements or the "fields" of an object or record.
While this is flexible is does not match the reality of strongly typed languages very well where data has to be homogenous and of a type that is known upfront. 
In addition the types are thereby unnecessarily repeated what both makes larger messages and gives room for inconsistencies.
Therefore you have to opt into this kind of dynamic typing - it is a *choice* not the default. 
The default is to not expect dynamic (or varying) types for members.

The text _"Hi"_ encoded using an `array` of `char` (say something like UTF-16 with 2 bytes per char) with one byte for the length of the value looks like this:

		array     char     length 2         H                 i
		001100001 01000010 00000010 00000000 01001000 00000000 01101001

Each character is encoded using 2 data bytes (UTF-16). No type bytes occur before the characters as elements of the array.

If the `char`s are varying, say like for UTF-8, this could be encoded as:

		array     char* 6  length 2 char   1 H        char   1 i
		001100001 01001110 00000010 01000001 01001000 01000001 01101001

The `v`arying bit is set (look for `*`) to indicate that each element in the array is a `char` of some sort (width a maxmimum width of 6 bytes per character; this is UTF-8 specific). The exact type is encoded for each value/element.
Here both `H` as well as `i` are ASCIIs (`char 1`) encoded as a single byte.

Arrays that are completely `dynamic` on the other hand a meant to contain different kinds of types as used in dynamically typed languages like javascript or clojure. The following clojure list `(1 "a" -2)` can be encoded as follows:

		array     dynamic  length 3 uint   1 1        char   1 a        int    1 -2
		001100001 00001110 00000011 00100001 00000001 01000001 01100001 00110001 11111110

`dynamic` might also be used to compress uniform arrays of a wide number type that are just filled with small numbers that could be encoded by a single byte. 


Shared Type Tags
----------------
Any type can be tagged with abitrary values. One useage is to _type tag_ values for a particular target languages or standards. This makes the type of a value even more precise so that an decoder knows what kind of value should be constructed from the data.

Shared tags are by convention encoded as `uint` values 
(for vendor, language or product specific tags use some other type).

The table below shows the meaning of tag values agreed upon so far:

TODO table


Initial Type Presets
--------------------
To save even more type definitions the possible 127 referable types can be predefine by convention.
If a type if refered to without having defined it in the same conversation it is defined by a commonly agreed preset table.

TODO table


Motivation
----------
Hundreds of binary data encodings already exist. 
None (known to me) provides a solution that in effect
allows to be precise about the semantics of the data
transfered without being both limited and excessive
at the same time. 

A really descriptive format must allow for rich 
composition from a basic set as narrow (and thereby 
general) as possible.
The format has to allow to attach arbitrary meta data
to enrich the information about the data.
This in turn makes effective type references impotent
to not having to repeat excessive type and meta data
over and over again. 
