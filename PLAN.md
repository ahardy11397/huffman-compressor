# PLAN — huffman-compressor: a Memelang graduate-level demo

Build a file compression CLI entirely in Memelang using Huffman coding.
Beta demonstration program: trees encoded in records, canonical bit
packing, and binary-safe file handling.

Language docs: /mnt/ssd/projects/meme/docs/WRITING_GUIDE.md and
docs/TUTORIAL.md (read both first). Compiler commands via
`uv run --project /mnt/ssd/projects/meme meme ...` from this repo.

## Ground rules (non-negotiable)

- Work ONLY in this repo. NO git commits.
- 100% Memelang. Bytes handled as Ints 0..255 (read the file as text; if a
  byte is not valid UTF-8 the program reports it cleanly and exits 1 —
  document this limitation; compressing ASCII/text files is the scope).
- JSON persistence in `data/` via Optional record readers/writers.
- Everything must parse with `--provider heuristic`.
- Test suites run in the FOREGROUND only.

## Format (fully specified so tests can be exact)

Compressed file = JSON document: {"table": [{"ch": <int 0..255>, "code":
"<0/1 string>"}, ...], "bits": "<0/1 string>", "original_length": <int>}.
The table is canonical: sorted by (code length ascending, char value
ascending). Codes come from a Huffman tree built with a deterministic
priority rule: among equal-weight nodes, the node whose subtree contains
the smallest char value goes first. Bits are the concatenated codewords
(whole file); no padding — the JSON stores the bit string plainly
(compression ratio is not the goal; correctness is).

## CLI (thin argv-router main + logic modules; unknown command → usage +
exit 1; every error = one stderr line + exit 1)

    huff compress <file> <out-file>
    huff decompress <file> <out-file>
    huff table <file>             # print the canonical code table
    huff stats <file> <out-file>  # original bytes, bit count, ratio as
                                  # `format <x> with 4 decimals`

Rules: empty file compresses to an empty table + empty bits and
decompresses to an empty file; single-char file gets code "0"; decompress
validates: bit length must equal sum over the implied stream, unknown chars
rejected; round-trip MUST be exact (byte-for-byte).

Suggested structures: tree nodes as records (ch, weight, left, right with
Optional children), frequency counts in `Map<Int, Int>`, the coding table
as `List<CodeEntry>` kept canonical-sorted.

## Tests — tests/meme/*.meme, `meme test`

At minimum: round-trip on "aaa", "abracadabra", a pangram, a file with all
26 letters + digits; canonical table order assertions (exact expected code
strings for "abracadabra" — hand-computed); empty file; single char;
decompress of truncated bits fails cleanly; stats ratio math; corrupt data
tolerance. Unique scratch files per test, cleaned up.

## Acceptance ("done means")

1. `meme test` python — 0 failed.
2. Same `--target rust cpp go c ts` — ALL SIX green.
3. `meme build main.meme -o huff_bin` — end-to-end demo: compress a real
   text file, show the table + stats, decompress, `diff` proves identity.
   Transcript in your final summary.
4. README.md: format spec, CLI usage, determinism rules, tests.
5. Final message: files, test count per target, demo transcript,
   deviations with reasons.
