Decompressor Errata
===================

This document captures known decompressor bugs, where the decompressor rejects a valid zstd frame.
Each entry will contain:
1. The last affected decompressor versions.
2. The decompressor components affected.
2. Whether the compressed frame could ever be produced by the reference compressor.
3. An example frame (hexadecimal string when it can be short enough, link to golden file otherwise)
4. A description of the bug.

The document is in reverse chronological order, with the bugs that affect the most recent zstd decompressor versions listed first.


Compressed block with a size of exactly 128 KB
------------------------------------------------

**Last affected version**: v1.5.2

**Affected decompressor component(s)**: Library & CLI

**Produced by the reference compressor**: No

**Example Frame**: see zstd/tests/golden-decompression/block-128k.zst

The zstd decoder incorrectly rejected blocks of type `Compressed_Block` when their size was exactly 128 KB.
Note that `128 KB - 1` was accepted, and `128 KB + 1` is forbidden by the spec.

This type of block was never generated by the reference compressor.

These blocks used to be disallowed by the spec up until spec version 0.3.2 when the restriction was lifted by [PR#1689](https://github.com/facebook/zstd/pull/1689).

> A Compressed_Block has the extra restriction that Block_Size is always strictly less than the decompressed size. If this condition cannot be respected, the block must be sent uncompressed instead (Raw_Block).

Compressed block with 0 literals and 0 sequences
------------------------------------------------

**Last affected version**: v1.5.2

**Affected decompressor component(s)**: Library & CLI

**Produced by the reference compressor**: No

**Example Frame**: `28b5 2ffd 2000 1500 0000 00`

The zstd decoder incorrectly rejected blocks of type `Compressed_Block` that encodes literals as `Raw_Literals_Block` with no literals, and has no sequences.

This type of block was never generated by the reference compressor.

Additionally, these blocks were disallowed by the spec up until spec version 0.3.2 when the restriction was lifted by [PR#1689](https://github.com/facebook/zstd/pull/1689).

> A Compressed_Block has the extra restriction that Block_Size is always strictly less than the decompressed size. If this condition cannot be respected, the block must be sent uncompressed instead (Raw_Block).

First block is RLE block
------------------------

**Last affected version**: v1.4.3

**Affected decompressor component(s)**: CLI only

**Produced by the reference compressor**: No

**Example Frame**: `28b5 2ffd a001 0002 0002 0010 000b 0000 00`

The zstd CLI decompressor rejected cases where the first block was an RLE block whose `Block_Size` is 131072, and the frame contains more than one block.
This only affected the zstd CLI, and not the library.

The example is an RLE block with 131072 bytes, followed by a second RLE block with 1 byte.

The compressor currently works around this limitation by explicitly avoiding producing RLE blocks as the first
block.

https://github.com/facebook/zstd/blob/8814aa5bfa74f05a86e55e9d508da177a893ceeb/lib/compress/zstd_compress.c#L3527-L3535

Tiny FSE Table & Block
----------------------

**Last affected version**: v1.3.4

**Affected decompressor component(s)**: Library & CLI

**Produced by the reference compressor**: Possibly until version v1.3.4, but probably never

**Example Frame**: `28b5 2ffd 2027 c500 0080 f3f1 f0ec ebc6 c5c7 f09d 4300 0000 e0e0 0658 0100 603e 52`

The zstd library rejected blocks of type `Compressed_Block` whose offset of the last table with type `FSE_Compressed_Mode` was less than 4 bytes from the end of the block.

In more depth, let `Last_Table_Offset` be the offset in the compressed block (excluding the header) that
the last table with type `FSE_Compressed_Mode` started. If `Block_Content - Last_Table_Offset < 4` then
the buggy zstd decompressor would reject the block. This occurs when the last serialized table is 2 bytes
and the bitstream size is 1 byte.

For example:
* There is 1 sequence in the block
* `Literals_Lengths_Mode` is `FSE_Compressed_Mode` & the serialized table size is 2 bytes
* `Offsets_Mode` is `Predefined_Mode`
* `Match_Lengths_Mode` is `Predefined_Mode`
* The bitstream is 1 byte. E.g. there is only one sequence and it fits in 1 byte.

The total `Block_Content` is `5` bytes, and `Last_Table_Offset` is `2`.

See the compressor workaround code:

https://github.com/facebook/zstd/blob/8814aa5bfa74f05a86e55e9d508da177a893ceeb/lib/compress/zstd_compress.c#L2667-L2682
