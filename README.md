# huff — a Huffman compressor in Memelang

A file-compression CLI written **100% in Memelang** (Canonical Structured
English, offline `heuristic` provider): Huffman trees built from records and
lists, canonical bit packing, and binary-safe-by-contract file handling.
One source tree, six backends — Python, Rust, C++, Go, C, TypeScript — plus a
native binary via `meme build`.

## Files

| File | Role |
|---|---|
| `huffcore.meme` | Coding core: byte mapping, frequency counts, deterministic Huffman tree, canonical table, encode/decode |
| `huffjson.meme` | Compressed-file JSON format: emission, marker parsing, ratio helper |
| `main.meme` | Thin argv-router CLI |
| `tests/meme/table_test.meme` | 18 core tests (exact hand-computed vectors) |
| `tests/meme/json_test.meme` | 14 format tests (exact document, corruption tolerance) |

Tests import the root modules directly via parent-relative `use ../../huffcore`
/ `use ../../huffjson` — no symlinks needed.

## Compressed-file format

A compressed file is one JSON document, emitted exactly as:

```json
{"table": [{"ch": 97, "code": "0"}, {"ch": 114, "code": "10"}], "bits": "0110", "original_length": 4}
```

- `table` — one entry per distinct byte: `"ch"` is the byte value (Int
  0..255), `"code"` its codeword as a `"0"/"1"` string. The table is
  **canonical**: sorted by code length ascending, then byte value ascending.
- `bits` — the concatenated codewords for the whole file, no padding.
- `original_length` — the byte count of the original file.

### Determinism rules

1. **Tree construction.** Active nodes are ordered by `(weight, smallest
   byte in subtree)`. Each step pops the two smallest by that key; the
   first-popped subtree takes bit `0`, the second bit `1`. (The
   implementation holds the queue in parallel lists and re-scans for the
   minimum each step — same ordering, no separate insert step.) Because
   each byte lives in exactly one active subtree, the `(weight, min-byte)`
   key is unique and the build is fully deterministic.
2. **Codes are tree codes**, not reassigned; the table is that set of
   codes, sorted canonically for output.
3. **One distinct byte** in the file → its code is `"0"`.
4. **Empty file** → empty table, empty bits, `original_length` 0.

### Hand-computed example: `abracadabra`

Frequencies `a=5 b=2 r=2 c=1 d=1`. Canonical table:

| ch | code |
|---|---|
| 97 (a) | `0` |
| 114 (r) | `10` |
| 98 (b) | `110` |
| 99 (c) | `1110` |
| 100 (d) | `1111` |

23 bits for 11 bytes (ratio 0.2614) — asserted byte-for-byte in
`tests/meme/table_test.meme`.

## CLI

```
huff compress <file> <out-file>      write the JSON-compressed document
huff decompress <file> <out-file>    restore the original file exactly
huff table <file>                    print the canonical table (one "<ch> <code>" line per entry)
huff stats <file> <out-file>         write: original bytes / compressed bits / ratio
```

- Unknown command or no command → usage on stderr, exit 1.
- Every error is a single `error: ...` line on stderr (via the language's
  `print error`) + exit 1.
- `ratio` = compressed bits ÷ (original bytes × 8), printed via the
  language's `format <x> with 4 decimals` (half-away-from-zero rounding,
  identical on all six targets). An empty file reports `0.0000`.
- Decompression validates: markers present, table array well-formed, bit
  characters are `0`/`1` only, codewords form a **prefix-free set** (a
  shorter code that is a prefix of a longer one is rejected — greedy
  decoding would otherwise silently return the wrong bytes), codewords
  prefix-consume the whole bit string with nothing left over, no
  duplicate codes, decoded byte count equals `original_length`, and
  every decoded byte must be encodable back to a character. Any
  violation → one error line, exit 1.

Round-trips are byte-for-byte exact — `diff` proves it in the demo below.

## Build & run

```bash
uv run --project /mnt/ssd/projects/meme meme build main.meme -o huff_bin
./huff_bin compress book.txt book.huff
./huff_bin table book.txt
./huff_bin stats book.txt book_stats.txt
./huff_bin decompress book.huff book_restored.txt
diff book.txt book_restored.txt   # identical
```

Cross-target checks:

```bash
uv run --project /mnt/ssd/projects/meme meme test tests/meme --target python  # also rust cpp go c ts
uv run --project /mnt/ssd/projects/meme meme check main.meme
```

## Documented limitations (per plan)

- **Character scope.** Input is read as text; supported characters are tab
  (9), newline (10), carriage return (13), and printable ASCII 32–126.
  Any other byte — including non-UTF-8 bytes, DEL, and multi-byte UTF-8 —
  is rejected with `error: unsupported byte in file <path>` and exit 1,
  on every backend. (Python/Rust detect invalid UTF-8 at read time; the
  other backends pass raw bytes through and are caught by the character
  check. The plan's scope is ASCII/text files.)
- **Error channel.** Errors and usage go to stderr via the language's
  `print error` statement, one line + exit 1, on every backend.
- **No byte I/O.** Memelang strings are byte-oriented but there is no
  byte→char conversion builtin; byte values come from the language's
  `number of <char>` / `character of <int>` codepoint conversions — the
  supported characters' byte values are exactly their codepoints. This is
  also why the JSON document stores the bit string plainly instead of
  packed bytes — correctness over ratio, per plan.
- **Format strictness.** The parser accepts the exact document shape this
  tool emits (fixed key order and spacing). A semantically identical JSON
  document with different whitespace, extra fields, or trailing junk
  after the closing brace is rejected cleanly rather than parsed leniently.
- **Performance.** Bit-string assembly and decoding are linear in input
  size times average codeword length (string concatenation per byte /
  prefix matching per bit); table work is bounded by the ≤98-symbol
  alphabet. This is demo-scale-appropriate, not a big-file tool.
- **Compression ratio is not the goal.** The bits are stored as a JSON
  string, so the on-disk "compressed" file is usually larger than the
  input; the ratio stat measures the coded bit count only.

## Tests

34 test cases (`meme test tests/meme`), all green on **all six targets**
(Python, Rust, C++, Go, C, TypeScript), run in the foreground:

- `table_test.meme` — exact hand-computed `abracadabra` codes and bits;
  `aaa`/single-char code "0"; empty file; `ab`; canonical-order and
  exact-bits assertions for a pangram and a 26-letters+digits file
  (vectors cross-checked by an independent reference implementation);
  round-trips; tab/newline/CR round-trip; `number of`/`character of`
  conversion coverage (tab=9, newline=10, CR=13, `"a"`=97, `~`=126, DEL
  and byte 200 rejected); truncated bits; non-`0/1` bit characters; empty
  table with bits; unknown byte in table; duplicate codes; unsupported
  character detection.
- `json_test.meme` — exact emitted document text (incl. empty); table /
  bits / length parsing; full document round-trips; truncated, garbage,
  corrupt-bits, unknown-byte, length-mismatch, table-without-bits
  documents all rejected cleanly; ratio math (23/88 → `0.2614`, 0/0 →
  `0.0000`) and corrupt ratio input.

Each test uses unique scratch paths under `tests/tmp/` (created
idempotently) and deletes them after use.
