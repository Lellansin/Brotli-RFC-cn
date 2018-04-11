## Encoding of Block-Switch Commands

As described in Section 2, a block-switch command is a pair &lt;block type, block count&gt;.  These are encoded in the compressed data part of the meta-block, right before the start of each new block of a particular block category.



Each block type in the compressed data is represented with a block type code, encoded using a prefix code over the block type code alphabet.  A block type symbol 0 means that the new block type is the same as the type of the previous block from the same block category, i.e., the block type that preceded the current type, while a block type symbol 1 means that the new block type equals the current block type plus one.  If the current block type is the maximal possible, then a block type symbol of 1 results in wrapping to a new block type of 0.  Block type symbols 2..257 represent block types 0..255, respectively.  The previous and current block types are initialized to 1 and 0, respectively, at the end of the meta-block header.



Since the first block type of each block category is 0, the block type of the first block-switch command is not encoded in the compressed data.  If a block category has only one block type, the block count of the first block-switch command is also omitted from the compressed data; otherwise, it is encoded in the meta-block header.



Since the end of the meta-block is detected by the number of uncompressed bytes produced, the block counts for any of the three categories need not count down to exactly zero at the end of the meta-block.



The number of different block types in each block category, denoted by NBLTYPESL, NBLTYPESI, and NBLTYPESD for literals, insert-and-copy lengths, and distances, respectively, is encoded in the meta-block header, and it must equal to the largest block type plus one in that block category.  In other words, the set of literal, insert-and-copy length, and distance block types must be \[0..NBLTYPESL-1\], \[0..NBLTYPESI-1\], and \[0..NBLTYPESD-1\], respectively.  From this it follows that the alphabet size of literal, insert-and-copy length, and distance block type codes is NBLTYPESL + 2, NBLTYPESI + 2, and NBLTYPESD + 2, respectively.



Each block count in the compressed data is represented with a pair &lt;block count code, extra bits&gt;.  The block count code and the extra bits are encoded back-to-back, the block count code is encoded using a prefix code over the block count code alphabet, while the extra bits value is encoded as a fixed-width integer value.  The number of extra bits can be 0..24, and it is dependent on the block count code. The symbols of the block count code alphabet along with the number of extra bits and the range of block counts are as follows:

```
        Extra              Extra               Extra
   Code Bits Lengths  Code Bits Lengths   Code Bits Lengths
   ---- ---- -------  ---- ---- -------   ---- ---- -------
    0    2    1..4     9    4   65..80    18    7   369..496
    1    2    5..8    10    4   81..96    19    8   497..752
    2    2    9..12   11    4   97..112   20    9   753..1264
    3    2   13..16   12    5  113..144   21   10   1265..2288
    4    3   17..24   13    5  145..176   22   11   2289..4336
    5    3   25..32   14    5  177..208   23   12   4337..8432
    6    3   33..40   15    5  209..240   24   13   8433..16624
    7    3   41..48   16    6  241..304   25   24   16625..16793840
    8    4   49..64   17    6  305..368
```

The first block-switch command of each block category is special in the sense that it is encoded in the meta-block header, and as described earlier, the block type code is omitted since it is an implicit zero.







