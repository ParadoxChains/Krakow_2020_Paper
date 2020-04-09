---
references:
- author:
  - family: Achten
    given: Peter
  - family: Plasmeijer
    given: Rinus
  container-title: Journal of Functional Programming
  id: cleanio
  issue: 1
  issued:
  - year: 1995
  link-citations: true
  page: '81-110'
  publisher: Cambridge University Press
  title: The Ins and Outs of Clean I/O
  type: 'article-journal'
  volume: 5
---

## Writing Wave files

### File manipulation in Clean

Due to Clean being a purely functional language, side effects such as
I/O operations are performed through uniqueness typing to preserve
referential transparency. Types which are marked as unique with `*`
cannot be used multiple times for any path of a function execution. It
guarantees that at the time a function with unique arguments is
executed, the arguments have no more than one reference each pointed to
them.

`World` is an abstract type representing the environment, or the state
of the world [@cleanio]. A program doing I/O is given an environment and
produces a new environment containing the changes. A more complicated
I/O program thus need chains the subprograms together by passing the
environment from one function to another.

``` {.clean caption="An example of passing the environment explicitly"}
echo :: *World -> *World
echo world0
  | c == '\n' = world2
              = echo world2
  where
  (c, world1) = getChar world0
  world2      = putChar c world1
```

The Clean `StdEnv` supports basic file manipulation in the `StdFile`
module. It provides operations for the `File` type, which can also be a
unique type. Opening a file requires an argument for a mode, which
distinguishes read, write, and append, and between text files and binary
data files. We will be using `FWriteData` mode for writing binary data.

There are several operations for writing data, though most of them are
not easy to work with for binary data. The smallest unit we can write is
a byte, however the byte is represented as a Clean `Char`. We will
assume a `Char` in Clean is a byte, we can denote it with a type
synonym.

``` {.clean}
:: Byte :== Char
```

There is a function for writing a string (which are unboxed `Char`
arrays in Clean) to a file, however lists are easier to work with most
of the time, so we will define a function to write a list of `Char`s to
a file.

``` {.clean}
writeBytes :: ![Byte] !*File -> *File
writeBytes []     f = f
writeBytes [b:bs] f
  #! f = fwritec b f
  = writeBytes bs f
```

`!` in the type specifies that the arguments of the function are strict,
this can improve program efficiency in places where laziness is not
needed. `#!` is a strict let notation, assigning the output of
`fwritec b f` to `f`. This `f` is not the same variable as the `f` in
the line before. In fact, it introduces a new scope and shadows the
previous variable. This is encouraged in Clean with unique types, it
makes the program somewhat resemble imperative programs besides the
explicit passing of the unique file.

We will also define a function to convert a nonnegative integer to a
list of bytes in little-endian order for later use. It will take and
argument to specify in how many bytes the number should be represented,
e.g. if the argument is 2, then the output will represent a 16-bit word,
the rest of the number is truncated. The function uses simple recursion
and basic operators from `StdEnv`.

``` {.clean}
// The first parameter is the number of bytes
// The second parameter is the integer to be converted
uintToBytesLE :: !Int !Int -> [Byte]
uintToBytesLE i n
  | i <= 0 = []
           = [toChar (n bitand 255) : uintToBytesLE (i - 1) (n >> 8)]
```

### Interface

We will implement writing to a Wave file in (L)PCM format due to its
simplicity.

The type of the function for writing a Wave file is given in a `dcl`
file (definition module). It takse some parameters that specifies the
structure of the file, and a list of bytes as the binary data in the
data chunk.

``` {.clean}
:: PcmWavParams =
  { numChannels    :: !Int // Number of channels
  , numBlocks      :: !Int // Number of samples (for each channel)
  , samplingRate   :: !Int // Sampling rate in Hz (samples per second)
  , bytesPerSample :: !Int // Number of bytes in each sample
  }

writePcmWav :: !PcmWavParams ![Byte] !*File -> *File
```

All data that the Wave file needs can be calculated from these
parameters. `numBlocks` represents the total number of blocks in the
data chunk, where each block contains `numChannels` samples.
`bytesPerSample` is how many bytes each sample contains.

### Implementation

The main function is composed of three smaller functions. The first one
writes the RIFF header into the file.

``` {.clean}
writeHeader :: !Int !*File -> *File
writeHeader l f
  #! f = fwrites "RIFF" f
  #! f = writeUint 4 l f
  #! f = fwrites "WAVE" f
  = f
```

The first argument is the length of the whole file minus the first eight
bytes. It will be calculated in the main function with 4 (the bytes
"WAVE") + 24 (size of the format chuk) + 8 (header of the data chunk) +
`bytesPerSample` $\times$ `numChannels` $\times$ `numBlocks` (size of
the binary data) + (1 or 0 depending on whether the size of the bynary
data is odd or even).

`writeUint` is a utility function that combines `writeBytes` and
`uintToBytesLE`.

The second function writes the format chunk, which contains writing in
the following data, using the same method as the previous function:

-   `"fmt "`
-   16 as a 4-byte number: the size of the format chunk after the first
    eight bytes
-   1 as a 2-byte number: this specifies the audio format to be PCM
-   `numChannels` as a 2-byte number
-   `samplingRate` as a 4-byte number
-   `samplingRate` $\times$ `bytesPerSample` $\times$ `numChannels` as a
    4-byte number: the amount of bytes processed per second
-   `bytesPerSample` $\times$ `numChannels` as a 2-byte number: the
    amount of bytes each block contains
-   8 $\times$ `bytesPerSample` as a 2-byte number: bits per sample

The last function takes in the length and the list of the binary data,
and writes it into the file using `writeBytes` after writing in the
chunk header. It also takes care of adding a padding byte if the size of
the data in bytes is odd.

The main function composes the smaller functions and evaluates the size
of the binary data so that it does not need to be calculated more than
once in the subfunctions.

``` {.clean}
writePcmWav :: !PcmWavParams ![Byte] !*File -> *File
writePcmWav p d f
  #! l = p.bytesPerSample * p.numChannels * p.numBlocks
  #! f = writeHeader (l + if (isEven l) 36 37) f
  #! f = writeFormat p f
  #! f = writeData l d f
  = f
```
