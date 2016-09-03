# BiDEF - Binary Data Exchange Format

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


Data Encoding
-------------
Data is described by a pair of a type followed by a value.

		+-----------+===========+
		|   type    |    data   |
		+-----------+===========+

The first byte of such a pair encodes what type it is.
Depending on the type the type has additional bytes
before the value bytes follow. 
The count of value bytes is either a fix number given
by the type or encoded explicitly as part of the type.


**0 Symbols**:

		+-----------+
		|0 000 ssss | (see table of symbols below)
		+-----------+

Tag symbol (is followed by a complete tag-value pair):

		+-----------+-------------    --+===========    ==+
		|0 000 1111 | tag's type / * /  | tag data / * /  |
		+-----------+------------    ---+==========    ===+

 Tags can be attached to any type including array
 element and record field types as well as dynamic
 types. This allows foreign language type tags or
 property names etcetera. 

**1 Bits**:

		+-----------+------------+=======        ==+
		|0 001 v bbb| data bytes | data / 1-255 /  |
		+-----------+------------+======        ===+

`bool` is equal to `bits(1)`, this type is encoded as `0 001 0 001  0000001`.

**2 Unsigned Integers**:

		+-----------+=======      ==+
		|0 010 v www| data / 1-8 /  |
		+-----------+======      ===+

**3 Signed Integers**:

		+-----------+=======      ==+
		|0 011 v www| data / 1-8 /  |
		+-----------+======      ===+

**4 Characters**:

		+-----------+=======      ==+
		|0 100 v www| data / 1-8 /  |
		+-----------+======      ===+

**5 Decimals**:

		+-----------+==============      ==+==========+
		|0 101 v www| coefficient / 0-7 /  | exponent |
		+-----------+=============      ===+==========+

**6 Arrays**:

		+-----------+---------------    --+---------      --+=======    ==+
		|0 110 v lll| element type / * /  | length / 1-8 /  | data / * /  |
		+-----------+--------------    ---+--------      ---+======    ===+

**7 Records**:

		+-----------+---------      --+--------------    --+=======    ==+
		|0 111 v lll| fields / 1-8 /  | field types / * /  | data / * /  |
		+-----------+--------      ---+-------------    ---+======    ===+

**Type Reference**:

		+-----------+
		|1 rrrrrrr  | (7 bit id)
		+-----------+

**Type Definition** (followed by a type without a value!): 

		+-----------+-----------+============    ==+
		|1 1111111  | 0 rrrrrrr |  type def / * /  |
		+-----------+-----------+===========    ===+


Table of Symbols
----------------

		 0 0000 numeric 0
		 1 0001 numeric 1
		 2 0010 numeric 2
		 3 0011 numeric 3
		 4 0100 numeric 4
		 5 0101 numeric 5
		 6 0110 numeric 6
		 7 0111 numeric 7
		 8 1000 numeric 8
		 9 1001 numeric 9
		10 1010 null
		11 1011 undefined
		12 1100 false
		13 1101	true
		14 1110	dynamic (marks elements within arrays and records as explicitly encoded for each value)
		15 1111 tag (followed by a complete type-value pair that is meant to be attached)

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
