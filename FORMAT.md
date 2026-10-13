# Fox stream format, version 0 (draft)

All multi-byte integers are **big endian**, and every block starts at an **even offset** from the start of the
stream, which itself has an even size. A 68000 can therefore read the whole stream in 16-bit words (it cannot read a
word from an odd address), and a 68020 or later can refill its bit buffer with one `move.l`. Little-endian CPUs pay
one byte swap per refill.

**Version 0 is a draft:** the format may still change without a new version number until it is declared final
(version 1). Streams from an earlier commit may not be readable by a later one.

## Stream

| Field | Size | Description |
|---|---|---|
| magic | 3 | `F` `O` `X` (0x46 0x4F 0x58) |
| version | 1 | 0 (the draft) |
| size | 4 | the size of the original data in bytes |
| blocks | ... | until `size` bytes are produced |
| crc | 4 | CRC-32 of the original data (polynomial 0xEDB88320 reflected, as in zip, gzip and PNG) |

Empty data has no blocks.

## Block

Every block starts at an even offset and covers at most 131072 (2^17) bytes of the original data.

| Field | Size | Description |
|---|---|---|
| type | 1 | 0 = stored, 1 = compressed with byte literals, 2 = compressed with word literals |
| reserved | 1 | 0 |
| raw | 4 | the bytes of original data in this block (1 to 131072) |
| payload | 4 | compressed blocks only: the size of the bit stream in bytes (always even) |
| data | raw or payload | the bytes themselves (stored) or the bit stream |
| padding | 0 or 1 | stored blocks with an odd `raw` only: a zero byte |

A compressor writes a block stored when the bit stream would not be smaller than the data.

## Bit stream

Bits are read MSB-first: from the highest bit of each byte to the lowest, byte after byte (the same order as reading
big-endian 16- or 32-bit words from the highest bit down). A field of n bits has its highest bit first. Huffman codes
are canonical (like deflate): shorter codes first, equal lengths in symbol order, written from their highest bit, so a
decoder can index a table with the top bits of its buffer.

The *unit* of a block is 1 byte for type 1 and 2 bytes (a big-endian 16-bit word: the first byte is the high byte) for type 2. Lengths and
distances are counted in units.

```
17 bits   numMain     the number of main symbols: the literal symbols + 32 match codes (see below)
 6 bits   numDist     the number of distance symbols with a code length (0 to 36)
20 x 3    precode     the code lengths of the 20 precode symbols (0 = unused, at most 7)
...       lengths     numMain + numDist code lengths (at most 16), coded with the precode
...       tokens      until the block's units are complete
 8 bits   tail        word blocks with an odd raw size only: the last byte
          padding     zero bits up to the next byte boundary, then a zero byte if needed for an even size
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

The literal symbols come first, so the most frequent symbols need no arithmetic; the 32 match codes are the last
symbols of the main alphabet. With `numLit = numMain - 32`:

**Byte blocks** (`numMain` must be 288):

| Symbol | Meaning |
|---|---|
| 0 to 255 | the literal byte with this value |
| 256 to 287 | a match, length code `symbol - 256` |

**Word blocks** (`numMain` from 33 to 65569):

| Symbol | Meaning |
|---|---|
| 0 | **new word**: 16 bits follow, the word itself; it becomes the next dictionary entry |
| 1 to numLit - 1 | **known word**: dictionary entry `symbol` (must already exist) |
| numLit to numLit + 31 | a match, length code `symbol - numLit` |

The dictionary of a word block starts empty and counts from 1, so its words are numbered in the order of their first
occurrence as a literal: the FOX scheme (*First Occurrence indeX*). Words that are only copied by matches are not in
it. An encoder writes `numLit` = 1 + the number of new words in the block.

### Matches

A match code (0 to 31) is a *bucket code* for `length - minimum`, where the minimum is 4 units for byte blocks and
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
