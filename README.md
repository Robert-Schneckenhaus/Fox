# Fox

**Fox** is a lossless compressor written in [CShift](https://github.com/Robert-Schneckenhaus/CShift). It is the
successor of [PYC / wsc](https://github.com/Pyrdacor/wsc), a word sequence compression, and keeps its core idea:
16-bit words whose first occurrence is stored raw and every later occurrence as its index (**FOX** = *First
Occurrence indeX*). Around that idea it puts what PYC was missing:

* **LZ77 matches** in front of the entropy stage (hash chains, lazy matching, levels 0 to 9), so repeated sequences
  cost a few bits instead of one code per word. This is what makes flat images and text small.
* **Byte or word literals per block.** Text and most binary data compress better with byte literals; 16-bit data
  (tile maps, 16-bit pixels, samples) with the FOX word literals. Each block tries both on a sample and takes the
  smaller one.
* **Length-limited, table-driven Huffman codes** (at most 16 bits, two-level decoding tables) instead of bit-by-bit
  decoding with dictionaries, and no 22-bit limit that made data incompressible.
* **A real container**: magic, version, original size, stored blocks for incompressible data, CRC-32, and a decoder
  that rejects damaged data with an error instead of crashing.

The format is specified in [FORMAT.md](FORMAT.md).

## Usage

```csharp
using FoxCompression;

uint8[] packed = Fox.Compress(bytes);             // level 5
uint8[] fast = Fox.Compress(bytes, 1);
uint8[] small = Fox.Compress(bytes, 9);
uint8[] words = Fox.Compress(bytes, 5, FoxMode.Words);   // force word literals

if (Fox.Decompress(packed) is uint8[] original)
    Use(original);

// or inside a function returning an Error<T>:
var data = try Fox.Decompress(packed);
```

| Level | What it does |
|---|---|
| 0 | no matches at all: pure FOX / Huffman (the PYC idea alone) |
| 1 - 2 | fast: short hash chains, greedy matching |
| 3 - 7 | lazy matching, longer chains (5 is the default) |
| 8 - 9 | the longest chains, slow |

Errors of `Fox.Decompress` (`FoxError`): `InvalidData`, `Truncated`, `ChecksumMismatch`, `UnsupportedVersion`.

### In a project

The library is the folder [`Fox/`](Fox) (a CShift library project). Add it to your program's `cshift.json`:

```json
{
	"name": "game",
	"dependencies": ["path/to/Fox/Fox"]
}
```

### Command line

```
cd cli
cshiftc build
bin/fox c -9 input.bmp input.fox          # compress (-0 .. -9, optional --bytes / --words)
bin/fox d input.fox input.bmp             # decompress
bin/fox bench file1 file2 ...             # all levels and modes against Deflate
```

### Tests

```
cd tests
cshiftc run                    # roundtrips of all levels and modes, edge cases, 6000 damaged streams
cshiftc run -- some/files/*    # also roundtrip these files
```

## Results

Single thread, x86-64, CShift 0.33 with clang 18 (`optimize: 2`). Ratio = compressed size / original size.
`deflate -6` is CShift's own `System.Compression.Deflate`, measured the same way. zstd and LZ4 are the C libraries,
given for orientation (their ratio; their speed is in another league: zstd -3 compresses at about 180 - 350 MB/s and
decompresses at about 1 GB/s on the same files).

| File | fox -1 | fox -5 | fox -9 | deflate -6 | zstd -3 | zstd -19 | LZ4 |
|---|---|---|---|---|---|---|---|
| text (vim help, 432 KB) | 37.1 % | 27.7 % | 26.8 % | 29.8 % | 29.6 % | 24.6 % | 48.0 % |
| source code (477 KB) | 31.0 % | 23.6 % | 23.0 % | 24.6 % | 25.8 % | 20.8 % | 39.8 % |
| x86-64 binary (2 MB) | 56.5 % | 50.0 % | 49.6 % | 51.0 % | 51.7 % | 43.8 % | 70.6 % |
| graphic, 8 bpp, 32 colors | 22.1 % | 16.8 % | 15.9 % | 15.9 % | 18.2 % | 11.5 % | 34.3 % |
| photo, 8 bpp, 16 colors | 24.8 % | 21.7 % | 20.9 % | 21.3 % | 23.9 % | 19.0 % | 38.7 % |
| photo, 24 bpp | 88.9 % | 84.8 % | 84.8 % | 85.3 % | 89.2 % | 80.1 % | 96.7 % |
| flat areas, 16 bpp | 0.8 % | 0.3 % | 0.1 % | 0.9 % | 0.9 % | 0.1 % | 1.2 % |
| dithered, 16 bpp | 18.6 % | 13.8 % | 12.6 % | 14.6 % | 17.4 % | 12.0 % | 36.3 % |
| tile map, 16 bit (32 KB) | 38.1 % | 38.4 % | 38.4 % | 53.5 % | 62.6 % | 50.2 % | 78.8 % |

Speed in MB/s of original data (compress / decompress):

| File | fox -1 | fox -5 | fox -9 | deflate -6 |
|---|---|---|---|---|
| text | 68 / 229 | 22 / 259 | 7 / 277 | 15 / 166 |
| source code | 76 / 242 | 28 / 304 | 7 / 321 | 20 / 178 |
| binary | 29 / 117 | 17 / 137 | 6 / 177 | 12 / 105 |
| graphic, 8 bpp | 88 / 254 | 23 / 332 | 3 / 416 | 15 / 271 |
| flat areas, 16 bpp | 770 / 1140 | 789 / 1261 | 201 / 1258 | 68 / 815 |
| dithered, 16 bpp | 95 / 223 | 19 / 311 | 1 / 439 | 10 / 324 |
| tile map | 37 / 348 | 25 / 325 | 25 / 339 | 21 / 148 |

What this shows:

* Fox -5 compresses better than zstd -3 on all of these files, and better than Deflate -6 except on the two 8 bpp
  images, where Deflate is 2 - 5 % smaller. It decompresses up to 2 times faster than CShift's Deflate (about the
  same on the dithered pixels).
* The FOX word literals pay off where the data really consists of 16-bit values: the tile map (random tile numbers,
  hardly any repeated sequences) is 38 % with Fox and 50 % even with zstd -19. On dithered 16-bit pixels words beat
  bytes by 19 %.
* For text, binaries and photos the byte literals win, and the automatic choice takes them.
* zstd remains the better general-purpose choice when its C library is available: it is several times faster in both
  directions at a similar ratio. Fox is a pure CShift library (also for WebAssembly and the Amiga) with a simple
  format, and the better choice for 16-bit data.

## What changed from PYC

Bugs of PYC that Fox fixes by design:

* Compressed and uncompressed output could not be told apart (`Compress` returned the input unchanged when it gave
  up). Fox has a header and stored blocks.
* Data of odd length was never compressed, although the README described padding; and no original size was stored.
* Data without any repeated word crashed the compressor (an empty Huffman tree), and a single repeated word got a
  code of length 0.
* More than 22-bit codes made data incompressible; Fox limits code lengths instead.
* The end of the data was detected with exceptions and position checks; Fox knows the sizes and checks a CRC-32.

## Ideas for later

* **Filters for images:** a delta / Paeth filter (like PNG) before the compression would help photos a lot.
* **tANS / FSE** instead of Huffman: faster decoding and closer to the entropy, especially for skewed symbols.
* **Threads:** the blocks are independent apart from the match window; with a window per block (or overlapping
  windows) they could be compressed in parallel.
* **Optimal parsing** (cost-based instead of greedy/lazy) for the high levels.
* **Faster level 1:** skip ahead faster in incompressible data (like LZ4's acceleration).
