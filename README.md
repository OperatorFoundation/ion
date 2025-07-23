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

Every Storage has a NounType, which is represented by an integer.

Every Storage has a value, of which there are five subtypes mapping onto the StorageTypes. The specific instantiations of these five subtypes are specific to each implementation, but they are defined semantically as follows:

- WORD - a signed integer of a machine-specific size
- FLOAT - a floating point number of a machine-specific size
- WORD_ARRAY - an unboxed list of words
- FLOAT_ARRAY - an unboxed list of floats
- MIXED_ARRAY - a boxed, mixed-type list of any Storage types

Taken altogether, every ion data structure is a tuple of (StorageType, NounType, value).

## Nouns

Each Storage has a NounType, which is an integer. Arbitrary noun types can be defined by the user, but there are also some basic builtin types:

- INTEGER (0) - this can be either a WORD or a BigInt represented as a WORD_ARRAY
- REAL (1) - a floating point number stands in awkwardly for a real number
- CHARACTER (2) - a Unicode character
- STRING (3) - a Unicode string
- LIST (4) - a general-purpose list, which could be specialized into a WORD_ARRAY, FLOAT_ARRAY, or MIXED_ARRAY
- DICTIONARY (5) - a mapping from Storage keys to Storage values

The StorageType determines the type of value. A WORD will always be accompanied by a machine word. However, the NounType determines how that value is interpreted. For instance, a WORD could be either an INTEGER or a CHARACTER.

## Encoding

ion follows a standard encoding for all StorageTypes, which is specifically designed to be cross-platform. It does not rely on machine word size or endianness.

### Squeeze Encoding of Unsigned Integers

A fundamental building block of the ion encoding is the cross-platform squeeze format for unsigned integers. The encoding for each unsigned integer has two parts:

- length - 1 byte
- value - variable

The length is a single byte that specifies how many bytes are used to encode the value portion. The value is the big endian encoding of the unsigned integer, dropping all leading zero bytes.

Sample encodings use decimal numbers to represent sequences of bytes. For instance, here is the decimal sequence that would be represented in hexademical as "0x01 0xFF":

```
1 255
```

This sample encoding is for the number 255. As this number fits into a single byte, the length is 1 and the value is 255.

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
8 127 255 255 255 255 255 255 255
```

This is a large number and on a 64-bit machine it would be okay, but this number is too big to be a word on a 32-bit machine This is no problem for ion as our format is cross-platform. 

### Squeeze Encoding of Signed Integers

Squeeze encoding supports signed integers by encoding the sign bit in the length field. When an integer is negative, the high bit of the length is set to 1, otherwise it is set to 0. This means that the length has a practical range of 0 to 127. The biggest number representable in ion would be 127 bytes long. If that ever becomes a practical issue, we would just update the encoding to remove this limitation. 

### Encoding Storage Values

Every encoded Storage starts with two bytes:

- StorageType - 1 byte
- NounType - 1 byte

The rest of the encoding is determined by the StorageType.

#### WORD

A word is represented as a squeezed signed integer. Here is an ion data structure with a StorageType of WORD (0), a NounType of INTEGER (0), and a value of 7:

```
0 0 1 7
```

StorageType is 0 (WORD). NounType is 0 (INTEGER). Squeezed length is 1, followed by a variable array of bytes of size 1 containing the value 7.

#### FLOAT

A float is similar to a WORD, but the length will either by 4 for a 32-bit float or 8 for a 64-bit float. The floating point value is specified in big endian IEEE floating point format. Other float sizes could be supported in the future if necessary. There are standardized formats for floats from 16 bits to 256 bits. The use of floats smaller than 16 bits curently lacks a standard format.

Examples are not given for the encoding of big-endian IEEE floats are this is covered in the IEEE 754 standard. Rather than implementing serialization directly, standard library functions should be used.

#### WORD_ARRAY

An array of words is represented by a squeezed integer specifying how many items are in the list, followed by a squeezed integer for each of those items. Here is the list [3, 4]:

```
3 4 1 2 1 3 1 4
```

First we have a StorageType 3 for WORD_ARRAY and a NounType 4 for LIST. Then we have a squeezed length "1 2", specifying 1 byte is used for the length, and the length specified is 2, meaning that there are 2 words in the list. Then we have two more squeezed integers, "1 3" uses 1 byte to encode the value 3 and "1 4" uses 1 byte to encode the value 4. Please note that the individual WORDs are unboxed. They do not carry StorageType and NounType information, just the array itself does.

#### FLOAT_ARRAY

An array of floats is the same as an array of words, except that the individual elements of the list will be FLOAT values instead of WORD values. Like in a WORD_ARRAY, the individual FLOAT values are unboxed. They do not carry StorageType or NounType information, just the array itself does.

#### MIXED_ARRAY

A mixed array is a boxed type. Each element in the list contains its own StorageType and NounType information, just like the array itself. This is a recursive data structure that can be used to build trees. Here is an example of the mixed array [[1, 2], [3, 4]]:

```
4 4 1 2 3 4 1 2 1 1 1 2 3 4 1 2 1 3 1 4
```

This is the most complicated example that we'll do for storage values. It might help to add some grouping puncutation like this:

```
(4,4): <1,2> [(3,4): <1,2> [(1,1), (1,2)], (3,4): <1,2> [(1,3), (1,4)]]
```

Let's break it down one group at a time:
- (4,4) - a MIXED_ARRAY LIST
- \<1,2\> - the mixed array has two elements
- (3,4) - the first element of the mixed array is a subarray that is a WORD_ARRAY LIST
- \<1,2\> - the first subarray has 2 elements
- (1,1) - the first element of the first subarray is 1
- (1,2) - the second element of the first subarray is 2
- (3,4) - the second element of the mixed array is a subarray that is also a WORD_ARRAY LIST
- \<1,2\> - the second subarray has 2 elements
- (1,3) - the first element of the second subarray is 3
- (1,4) - the second element of the second subarray is 4

### Encoding Noun Values

Most of the work of encoding is done using just the StorageType. The NounType mainly comes into play in the interpretation of the value. For instance, if you are deserialzing it into a native type for a specific programming language, then the difference between a WORD that is an INTEGER and a WORD that is a CHARACTER are of interest. However, there are a few notes to consider about how to encode specific nouns.

#### INTEGER

The INTEGER NounType can be implemented either as a WORD, in which case it is of machine word size, or a WORD_ARRAY to represent a BigInt larger than machine word size.

Note that if two machines are communicating using the ion format that they may not have the same machine word size. Therefore, whether a specific INTEGER is represented as a WORD or a WORD_ARRAY is dependent on the decoding side's machine word size. For encoding, the squeeze format for integers does not care about machine word size. Therefore, it is permissable to use a WORD StorageType, regardless of the size of the integer. The decoder must accommodate this and automatically promote integers to BigInts if they are too large to be machine words. Also, note that INTEGER values are signed, so the maximum size of an integer for a given machine word size is 1 bit less than that of an unsigned integer. So on a 32-bit machine the maximum size of a WORD is 31 bits. Also note that machine word size is dependent not only on the machine, but also on the programming language. In Python, for instance, all integers are automatically promoted to BigInts anyway, whereas in Java an "int" is always 32 bits, regardless of the actual machine it is running on.

#### REAL

The REAL NounType is implemented as a FLOAT. As FLOAT types can be either 32-bit or 64-bit, the encoder can choose which size to use, usually the native size of a floating point number for the machine or programming language. The decoder must be able to decode both sizes of floating point numbers, but is free to convert it to a preferred local type. This does mean that two machines that are passing REAL values back and forth may lose precision if the REAL goes from a 64-bit machine to a 32-bit machine and back again. This may seem undesirable, but if the 32-bit machine is going to be doing any computations with the value, it is likely to do them with 32-bits anyway, so this is just a reality of heterogenous computation. It is also important to note that the floating point word size and integer word size may not be the same on some machines.

#### CHARACTER

A CHARACTER is implemented as a WORD. This is in contrast to other popular representations, such as UTF-8, which is based on arrays of arrays of bytes (grapheme clusters) serialized into a complex data structure that is represented by an array of bytes but lacks the corresponding semantics. In UTF-8, individual characters are of variable length. In ion, a CHARACTER is always a WORD. It may be tempting to just pretend that UTF-8 strings are arrays of bytes and serialize them as such, but this will yield unwanted results in all cases except when the UTF-8 strings happen to coincidentally also be ASCII strings. ion encoders and decoders are required to actually support Unicode and not just ASCII.

#### STRING

Just as a CHARACTER is a WORD in ion, a STRING of characters is a WORD_ARRAY. On 32-bit machines, this is the equivalent of a UTF-32 encoding. It is relevant to note here that the squeeze format for WORD and WORD_ARRAY values does not transmit leading 0 bytes and so while it is not as efficient as UTF-8, especially for ASCII strings, we will not be encoding a full 4 bytes on 32-bit machines (or 8 bytes on 64-bit machines!) for every character in a STRING.

#### LIST

There are three types of lists: WORD_ARRAY for unboxed machine words, FLOAT_ARRAY for unboxed floating point numbers of a machine-specific size, and MIXED_ARRAY for general-purpose boxed arrays. The terms "boxed" and "unboxed" here refer to whether or not individual elements carry type information. A WORD_ARRAY can only contain WORD values and a FLOAT_ARRAY can only contain FLOAT values. If more nuanced typing is needed, then a MIXED_ARRAY provides types for each element. A MIXED_ARRAY is also the only way to have a list of lists.

#### DICTIONARY

A DICTIONARY is implemented as a MIXED_ARRAY contains two elements, both of which are LIST values representing parallel arrays. The first array contains keys and the second array, of the same length, contains associated values. This encoding is what might be referred to as column-based instead of the more popular row-based encoding as a list of key-value pairs. As a MIXED_ARRAY is used for both lists, any value can be a key and any value can be an associated value.

# Implementations

ion is portable not just across machine architectures, but also programming languages.

- [iota-cpp](https://github.com/OperatorFoundation/ion-cpp) - C++17 for both Arduino cores and desktop operating systems
- [ion-swift](https://github.com/OperatorFoundation/ion-swift) - Swift 5.9
- [iota-python](https://github.com/blanu/iota-python) - ion is included inside the iota implementation for CircuitPython for both Arduino cores and desktop operating systems

## Implementation Notes

Encoders and decoders should always map to native types in the local programming language. The only exception to this is when the local programming language lacks a corresponding type. For instance, some programming languages may lack the capacity to create a dictionary whose keys are also dictionaries. Strongly statically-typed programming languages may have trouble with even the concept of a list of lists. When possible, it would be nice to use the most colloquial local type available, such as an [[int]] (array of array of ints) for decoding a MIXED_ARRAY of WORD_ARRAY values. Similarly, when encoding it is best to use the simplest type available, for instance a WORD_ARRAY when given a python list to encode that happens to be all integers.

Also of note to the implementor is that there is one special value reserved in the squeeze encoding. A length value of 128 will never occur normally in encoding, as this is "-0", which is always represented by 0 rather than 128. Therefore, this value has been reserved for debugging of encoders. As a length of 128 should never be seen in production use, it can be followed by any debugging information that the implementor sees fit during development. A production decoder should treat a length of 128 as an unrecoverable error as it means that the other side is in debug mode and behavior following receiving this length is undefined.

# Motivation

ion is an attempt to create a truly portable data exchange format. While it is focused primarily on 32-bit and 64-bit machines and high-level programming languages, it should be implementable on an 8-bit machine. Issues which intefere with binary data format portable, such as endianness and machine word size, have been carefully avoided. It should be possible to implement the format in C, or even assembly.

Another consideration was avoiding the pitfalls of other encoding schemes, a world which has been dominated by JSON for a long time to the extent that other encodings often mimic the shortfalls of JSON, which was never intended to be a universal data exchange format between all programming languages. Two notable qualities to be avoided are arbitrary limitations, for instance on dictionary keys, and having a whole separate set of types ("JSON values") rather than using native types.

The primary use case of interest for this project is communication between diverse platforms, specifically Arduino-compatible microcontrollers, Android and iOS phones, and desktop operating systems.

# Possible Future Work

- FLOAT values of additional sizes
- STRING value alternative that uses a MIXED_ARRAY of WORD_ARRAY values to represent grapheme clusters directly
- byte arrays as a WORD_ARRAY and a count of how many bytes are actually used (or unused)
    - This could be extended to other sub-word array types, such as int16, etc.
- multi-arrays, which is to say the semantics of a single list, but a representation as a MIXED_ARRAY of LIST values, for very large lists
- slices of an array (offset and length)
- bit arrays
- booleans
- general typing for various kinds of fixed-width signed and unsigned integers
    - this could be arbitrary, it need not stick to traditional 8, 16, etc. This would be useful, for instance, in modeling TCP fields
- sets, multisets, bags, etc.
- optionals
- variants (a.k.a. unions)
- remove limitations on the sizes of integers and lists
- implementation for Kotlin/Android
