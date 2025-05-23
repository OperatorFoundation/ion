[TOC]

# The Operator Foundation

 [Operator](https://operatorfoundation.org/) makes useable tools to help people around the world with censorship, security, and privacy.

# ion

ion, a lightweight, portable, and cross-platform binary data exchange format

# Specification

ion supports one universal data type, with several subtypes.

Every ion data structure is of type Storage.

Every Storage has a StorageType, of which there are 5 options:

- WORD (0)
- FLOAT (1)
- WORD_ARRAY (2)
- FLOAT_ARRAY (3)
- MIXED_ARRAY (4)

Every Storage has a NounType, which is an integer.

Every Storage has a value, of which there are five subtypes mapping onto the StorageTypes. The specific instantiations of these five subtypes are specific to each implementation, but they are defined semantically as follows:

- WORD - a signed integer of a machine-specific size
- FLOAT - a floating point number of a machine-specific size
- WORD_ARRAY - an unboxed list of words
- FLOAT_ARRAY - an unboxed list of floats
- MIXED_ARRAY - a boxed, mixed-type list of any Storage types

## Nouns

Each Storage has a NounType, which is an integer. Arbitrary noun types can be defined by the user, but there are also some basic builtin types:

- INTEGER (0) - this can be either a WORD or a BigInt represented as a WORD_ARRAY
- REAL (1) - a floating point number stands in awkwardly for a real number
- CHARACTER (2) - a unicode character
- STRING (3) - a unicode string
- LIST (4) - a general-purpose list, which could be specialized into a WORD_ARRAY, FLOAT_ARRAY, or MIXED_ARRAY
- DICTIONARY (5) - a mapping from Storage keys to Storage values

## Encoding

Storage follows a standard encoding, which is specifically cross-platform. It does not rely on machine word size or endianness.

### Squeeze Encoding of Unsigned Integers

A fundamental building block of the ion encoding is the cross-platform squeeze format for unsigned integers. The encoding for each unsigned integer has two parts:

- length - 1 byte
- value - variable

The length is a single byte that specifies how many bytes are used to encode the value portion The value is the big endian encoding of the unsigned integer, dropping all leading zero bytes.

Sample encodings use decimal numbers to represent sequences of bytes. For instance, here is the hexadecimal sequence 0x01 0xFF 0x00:

```
1 255 0
```

Zero has a special encoding:

```
0
```

Small integers have small encodings. Note that the size of the encoding does not depend on the machine word size or endianness

For instance, here is the encoding for 1:

```
1 1
```

The above encoding consists of a length of 1, followed by a variable value of size 1, containing the byte 1.

Larger integers have larger encodings, here is the encoding for 2147483648:

```
4 128 0 0 0
```

This has a length of 4, followed by a 4-byte value. This encoding is the same whether its on a 32-bit or 64-bit machine architecture and whether the machine architecture is little endian or big endian.

Integers larger than a machine word size are automatically handled as bigints. Here is the encoding for 9223372036854775807:

```
127 255 255 255 255 255 255 255
```

This number is too big to be a word on a 32-bit machine, but it's no problem as our format is cross-platform.

### Squeeze Encoding of Signed Integers

Squeeze encoding can be extended to support signed integers. When an integer is negative, the high bit of the length is set to 1, otherwise it is set to 0. This means that the length has a practical range of 0 to 127.

### Encoding Storage Values

Every encoded Storage starts with two bytes:

- StorageType - 1 byte
- NounType - 1 byte

The rest of the encoding is determined by the StorageType.

#### WORD

A word is represented as a squeezed signed integer. Here is a WORD with a NounType of INTEGER and a value of 7

```
0 0 1 7
```

StorageType is 0 (WORD). NounType is 0 (INTEGER). Squeezed length is 1, followed by a variable array of bytes of size 1 containing the value 7.

#### FLOAT

A float is similar to a WORD, but the length will either by 4 for a 32-bit float or 8 for a 64-bit float. The floating point value is specified in big endian IEEE floating point format.

#### WORD_ARRAY

An array of words is represented by a squeezed unsigned integer specifying how many items are in the list, followed by a squeezed sign integer for each of those items. Here is the list [3, 4]:

```
3 4 1 2 1 3 1 4
```

First we have a StorageType 3 for WORD_ARRAY and a NounType 4 for LIST. Then we have a squeezed length "1 2", specifying 1 byte is used for the length, and the length specified is 2, meaning that there are 2 words in the list. Then we have two more squeezed integers, "1 3" uses 1 byte to encode the value 3 and "1 4" uses 1 byte to encode the value 4. Please note that the individual WORDs are unboxed. They do not carry StorageType and NounType information, just the array itself does.

#### FLOAT_ARRAY

An array of floats is the same as an array of words, except that the individual elements of the list will be FLOAT values instead of WORD values. Like in a WORD_ARRAY, the individual FLOAT values are unboxed. They do not carry StorageType or NounType information, just the array itself does.

#### MIXED_ARRAY

A mixed array is a boxed type. Each element in the list contains its own StorageType and NounType information, just like the array itself. This is a recursive data structure that can be used to build trees.

### Encoding Noun Values

#### INTEGER

The INTEGER NounType can be implemented either as a WORD, in which case it is of machine word size, or a WORD_ARRAY to represent a BigInt larger than machine word size.

#### REAL

The REAL NounType is implemented as a FLOAT.

#### CHARACTER

A CHARACTER is implemented as a WORD. This is in contrast to other representations, such as UTF-8, which is based on arrays of arrays of bytes (grapheme clusters) serialized into a complex data structure that is represented by an array of bytes. In UTF-8, individual characters are of variable length. In ion, a CHARACTER is always a WORD.

#### STRING

Just as a CHARACTER is a WORD in ion, a STRING of CHARACTERS is a WORD_ARRAY. On 32-bit machines, this is the equivalent of a UTF-32 encoding.

#### LIST

There are three types of lists: WORD_ARRAY for unboxed machine words, FLOAT_ARRAY for unboxed floating point numbers of a machine-specific size, and MIXED_ARRAY for general-purpose boxed arrays.

#### DICTIONARY

A DICTIONARY is implemented as a MIXED_ARRAY contains two elements, both of which are LISTs representing parallel arrives. The first array contains keys and the second array, of the same length, contains associated values.

# Implementations

ion is portable not just machine architectures, but also programming languages.

- [ion-swift](https://github.com/OperatorFoundation/ion-swift) - Swift 5.9
- [iota-cpp](https://github.com/blanu/iota-cpp) - ion is included inside the iota implementation for C++17 for both Arduino cores and desktop operating systems
- [iota-python](https://github.com/blanu/iota-python) - ion is include inside the iota implementation for CircuitPython for both Arduino cores and desktop operating systems
