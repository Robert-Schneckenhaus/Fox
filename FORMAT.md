# Fox stream format, version 1

All multi-byte integers are little endian. `varint` is an unsigned LEB128 number (7 bits per byte, lowest group
first, the high bit set on all bytes but the last).

## Stream

| Field | Size | Description |
|---|---|---|
| magic | 3 | `F` `O` `X` (0x46 0x4F 0x58) |
| version | 1 | 1 |
| size | varint | the size of the original data in bytes |
| blocks | ... | until `size` bytes are produced |
| crc | 4 | CRC-32 of the original data (polynomial 0xEDB88320 reflected, as in zip, gzip and PNG) |

Empty data has no blocks.

## Block

Every block starts at a byte boundary and covers at most 131072 (2^17) bytes of the original data.

| Field | Size | Description |
|---|---|---|
| type | 1 | 0 = stored, 1 = compressed with byte literals, 2 = compressed with word literals |
| raw | varint | the bytes of original data in this block (1 to 131072) |
| payload | varint | compressed blocks only: the size of the bit stream in bytes |
| data | raw or payload | the bytes themselves (stored) or the bit stream |

A compressor writes a block stored when the bit stream would not be smaller than the data.

## Bit stream

Bits are read from the lowest bit of each byte to the highest, byte after byte; a field of n bits has its lowest bit
first. Huffman codes are canonical (like deflate): shorter codes first, equal lengths in symbol order, and their bits
are stored from the highest code bit to the lowest (i.e. the code is bit-reversed in the stream, as in deflate).

The *unit* of a block is 1 byte for type 1 and 2 bytes (a little-endian 16-bit word) for type 2. Lengths and
distances are counted in units.

```
17 bits   numMain     the number of main symbols with a code length (the rest have length 0)
 6 bits   numDist     the number of distance symbols with a code length (0 to 36)
20 x 3    precode     the code lengths of the 20 precode symbols (0 = unused, at most 7)
...       lengths     numMain + numDist code lengths (at most 16), coded with the precode
...       tokens      until the block's units are complete
 8 bits   tail        word blocks with an odd raw size only: the last byte
          padding     zero bits up to the next byte boundary
```

### Code lengths

The main and distance lengths form one sequence (so a run may continue from one into the other):

| Precode symbol | Meaning |
|---|---|
| 0 to 16 | one length of this value |
| 17 + 3 bits | 3 to 10 lengths of 0 |
| 18 + 7 bits | 11 to 138 lengths of 0 |
| 19 + 2 bits | 3 to 6 more of the previous length (which must not be 0) |

Over-subscribed codes are invalid. Incomplete codes are allowed (for example a single used symbol has a code of
length 1); their unused bit patterns are invalid in the data.

### Main symbols

| Symbol | Byte blocks | Word blocks |
|---|---|---|
| 0 to 31 | match, length code | match, length code |
| 32 to 287 | literal byte `symbol - 32` | |
| 32 | | **new word**: 16 bits follow, the word itself; it is appended to the block's dictionary |
| 33 to 32 + 65536 | | **known word**: dictionary entry `symbol - 33` (must already exist) |

The dictionary of a word block starts empty, so its words are numbered in the order of their first occurrence as a
literal: the FOX scheme (*First Occurrence indeX*). Words that are only copied by matches are not in it.

### Matches

A match symbol (0 to 31) is a *bucket code* for `length - minimum`, where the minimum is 4 units for byte blocks and
2 units for word blocks (4 bytes either way). It is followed by its extra bits, then one distance symbol (bucket code
for `distance - 1`) and its extra bits. The match copies `length` units starting `distance` units before the current
position; it may overlap the bytes it writes, reach into earlier blocks, but not before the start of the data, and
not past the end of the current block.

Bucket codes, for a value v:

| Code | Values | Extra bits |
|---|---|---|
| 0, 1, 2, 3 | 0, 1, 2, 3 | 0 |
| 2n + b, n >= 2 | `(2 + b) << (n - 1)` + extra | n - 1 |

So 4 covers 4..5, 5 covers 6..7, 6 covers 8..11, 7 covers 12..15, and so on. Lengths use codes 0 to 31
(`length - minimum` < 65536; encoders write at most 32768 units), distances codes 0 to 35 (`distance - 1` < 2^18).
