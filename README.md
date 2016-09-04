# BiDEF - Binary Data Exchange Format

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
 
  The `dynamic` type is used for array element types or record field types to indicate that the type is explicitly encoded for each value. When `dynamic` is used as a top level type the receiver is asked to interprete the value bits/bytes and determine the target type from it.

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

* `bool` is equal to `bits(1)` what i is encoded as `00010001 0000001`. 
* the `bits` value for _true_ is `00000001`.
* the `bits` value for _false_ is `00000000`.

#### Unsigned Integers

		+-----------+=======      ==+
		|0 010 v www| data / 1-8 /  |
		+-----------+======      ===+

#### Signed Integers

		+-----------+=======      ==+
		|0 011 v www| data / 1-8 /  |
		+-----------+======      ===+

#### Characters

		+-----------+=======      ==+
		|0 100 v www| data / 1-8 /  |
		+-----------+======      ===+

#### Decimals

		+-----------+==============      ==+==========+
		|0 101 v www| coefficient / 0-7 /  | exponent |
		+-----------+=============      ===+==========+

#### Arrays

		+-----------+---------------    --+---------      --+=======    ==+
		|0 110 v lll| element type / * /  | length / 1-8 /  | data / * /  |
		+-----------+--------------    ---+--------      ---+======    ===+

  Elements will only have an explicit type if the array is `v`arying or its element type is `dynamic`.
  
  If a `v`arying array type is predefined and later referenced the element type is (re)stated explicitly for each value.
  This allows to change details of the element type like the width or if it is varying.

#### Records

		+-----------+---------      --+--------------    --+=======    ==+
		|0 111 v lll| fields / 1-8 /  | field types / * /  | data / * /  |
		+-----------+--------      ---+-------------    ---+======    ===+

  When the record type is predefined and later referenced only those fields will have an explicit type that are `v`arying or `dynamic`. 
  
  If the whole record is `v`arying the field types are not part of the type definition but repeated before each value.
  This is used to as a form of _union_ type.
  

#### Type Reference

		+-----------+
		|1 rrrrrrr  | (7 bit id)
		+-----------+

**Type Definition** (followed by a type without a value!): 

		+-----------+-----------+============    ==+
		|1 1111111  | 0 rrrrrrr |  type def / * /  |
		+-----------+-----------+===========    ===+

  Defines a type with the given 7-bit id for the **duration of the session**.
  A type might however be redefined in any message during a session (including the same message that origianlly defined it first). 
  The ids `1xxxxxxx` or `x1111111` should not be used as they cannot be referenced.
  It is however not illegal to do so. The receiver is free to ignore such definitions or do some custom interpretation that might be meaningful between particular senders and receivers.


Messages
--------
A message is a sequence of type-value pairs.
It starts typically by some type definitions followed by one or more type-value pairs that reference to the previously defined types. But type definitions and usual typed values can be mixed at will.


Dynamic Typing
--------------
Most typed binary formats are dynamically typed. 
This is to say that each value has an explicit type tag that can be any of the possible types.
The types of values are encountered right before the value. 
This is even true for values that are "members" like array elements or "fields".
While this is flexible is does not match the reality of strongly typed languages very well.
In addition the types are thereby unnecessarily repeated.
Therefore this kind of typing is a *choice* in BiDEF. 
The default is to not expect dynamic (varying) types for members.

An `array` of `char` (say something like UTF-16 with 2 bytes per char) with one byte for the length of the value has the following type:

		array     char     length 2         H                 i
		001100001 01000010 00000010 00000000 01001000 00000000 01101001

The text data _"Hi"_ has **no** type tag before each character.
Now if the `char`s are varying, say like for UTF-8 this could be encoded as:

		array     char* 6  length 2 char   1 H        char   1 i
		001100001 01001110 00000010 01000001 01001000 01000001 01101001

The text now uses `char`s with a _maximum_ of 6 bytes per char. Note also that the `v`arying bit is set (right where the `*` is) to indicate that each element in the array is again type encoded. Here these are both 1 byte long `char`s.


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
