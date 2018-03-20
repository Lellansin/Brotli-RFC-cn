# 3.  前缀码的压缩表述

## 3.1.  前缀编码简介

Prefix coding represents symbols from an a priori known alphabet by bit sequences \(codes\), one code for each symbol, in a manner such that different symbols may be represented by bit sequences of

different lengths, but a parser can always parse an encoded string unambiguously symbol-by-symbol.

We define a prefix code in terms of a binary tree in which the two edges descending from each non-leaf node are labeled 0 and 1, and in which the leaf nodes correspond one-for-one with \(are labeled with\) the symbols of the alphabet.  The code for a symbol is the sequence of 0's and 1's on the edges leading from the root to the leaf labeled with that symbol.  For example:

```
               /\              Symbol    Code
              0  1             ------    ----
             /    \                A     00
            /\     B               B     1
           0  1                    C     011
          /    \                   D     010
         A     /\
              0  1
             /    \
            D      C
```

A parser can decode the next symbol from the compressed stream by walking down the tree from the root, at each step choosing the edge corresponding to the next compressed data bit.

Given an alphabet with known symbol frequencies, the Huffman algorithm allows the construction of an optimal prefix code \(one that represents strings with those symbol frequencies using the fewest

bits of any possible prefix codes for that alphabet\).  Such a prefix code is called a Huffman code.  \(See \[HUFFMAN\] for additional information on Huffman codes.\)

In the brotli format, note that the prefix codes for the various alphabets must not exceed certain maximum code lengths.  This constraint complicates the algorithm for computing code lengths from symbol frequencies.  Again, see \[HUFFMAN\] for details.

## 3.2.  在 Brotli 格式中使用前缀编码

The prefix codes used for each alphabet in the brotli format are canonical prefix codes, which have two additional rules:

* All codes of a given bit length have lexicographically consecutive values, in the same order as the symbols they represent;
* Shorter codes lexicographically precede longer codes.

  We could recode the example above to follow this rule as follows, assuming that the order of the alphabet is ABCD:

```
      Symbol  Code
      ------  ----
      A       10
      B       0
      C       110
      D       111
```

That is, 0 precedes 10, which precedes 11x, and 110 and 111 are lexicographically consecutive.

Given this rule, we can define the canonical prefix code for an alphabet just by giving the bit lengths of the codes for each symbol of the alphabet in order; this is sufficient to determine the actual codes.  In our example, the code is completely defined by the sequence of bit lengths \(2, 1, 3, 3\).  The following algorithm generates the codes as integers, intended to be read from most to least significant bit.  The code lengths are initially in tree\[I\].Len; the codes are produced in tree\[I\].Code.

```
  1\) Count the number of codes for each code length.  Let bl\_count\[N\] be the number of codes of length N, N &gt;= 1.

  2\) Find the numerical value of the smallest code for each code length:
```

```
            code = 0;
            bl_count[0] = 0;
            for (bits = 1; bits <= MAX_BITS; bits++) {
               code = (code + bl_count[bits-1]) << 1;
               next_code[bits] = code;
            }
```

```
  3\) Assign numerical values to all codes, using consecutive values for all codes of the same length with the base values determined at step 2.  Codes that are never used \(which have a bit length of zero\) must not be assigned a value.
```

```
            for (n = 0; n <= max_code; n++) {
               len = tree[n].Len;
               if (len != 0) {
                  tree[n].Code = next_code[len];
                  next_code[len]++;
               }
            }
```

Example:

Consider the alphabet ABCDEFGH, with bit lengths \(3, 3, 3, 3, 3, 2, 4, 4\).  After step 1, we have:

```
N      bl_count[N]
-      -----------
2      1
3      5
4      2
```

Step 2 computes the following next\_code values:

```
N      next_code[N]
-      ------------
1      0
2      0
3      2
4      14
```

Step 3 produces the following code values:

```
Symbol Length   Code
------ ------   ----
A       3       010
B       3       011
C       3       100
D       3       101
E       3       110
F       2       00
G       4       1110
H       4       1111
```

## 3.3.  字母大小

Prefix codes are used for different purposes in the brotli format, and each purpose has a different alphabet size.  For literal codes, the alphabet size is 256.  For insert-and-copy length codes, the alphabet size is 704.  For block count codes, the alphabet size is 26.  For distance codes, block type codes, and the prefix codes used in compressing the context map, the alphabet size is dynamic and is based on parameters defined in later sections.  The following table summarizes the alphabet sizes for the various prefix codes and the sections of this document in which they are defined.

```
+-----------------+-------------------------+------------+
| Prefix Code     | Alphabet Size           | Definition |
+-----------------+-------------------------+------------+
| literal         | 256                     |            |
+-----------------+-------------------------+------------+
| distance        | 16 + NDIRECT +          | Section 4  |
|                 | (48 << NPOSTFIX)        |            |
+-----------------+-------------------------+------------+
| insert-and-copy | 704                     | Section 5  |
| length          |                         |            |
+-----------------+-------------------------+------------+
| block count     | 26                      | Section 6  |
+-----------------+-------------------------+------------+
| block type      | NBLTYPESx + 2,          | Section 6  |
|                 | (where x is I, L, or D) |            |
+-----------------+-------------------------+------------+
| context map     | NTREESx + RLEMAXx       | Section 7  |
|                 | (where x is L or D)     |            |
+-----------------+-------------------------+------------+
```

## 3.4.  简单前缀码

The first two bits of the compressed representation of each prefix code distinguish between simple and complex prefix codes.  If this value is 1, then a simple prefix code follows as described in this section.  Otherwise, a complex prefix code follows as described in Section 3.5.

A simple prefix code can have up to four symbols with non-zero code length.  The format of the simple prefix code is as follows:

```
  2 bits: value of 1 indicates a simple prefix code

  2 bits: NSYM - 1, where NSYM = number of symbols coded
```

NSYM symbols, each encoded using ALPHABET\_BITS bits

```
  1 bit:  tree-select, present only for NSYM = 4
```

The value of ALPHABET\_BITS depends on the alphabet of the prefix code: it is the smallest number of bits that can represent all symbols in the alphabet.  For example, for the alphabet of literal bytes, ALPHABET\_BITS is 8.  The value of each of the NSYM symbols above is the value of the ALPHABET\_BITS width integer value.  If the integer value is greater than or equal to the alphabet size, or the value is identical to a previous value, then the stream should be rejected as invalid.

Note that the NSYM symbols may not be presented in sorted order. Prefix codes of the same bit length must be assigned to the symbols in sorted order.

The \(non-zero\) code lengths of the symbols can be reconstructed as follows:

```
  \*  if NSYM = 1, the code length for the one symbol is zero -- when encoding this symbol in the compressed data stream using this prefix code, no actual bits are emitted.  Similarly, when decoding a symbol using this prefix code, no bits are read and the one symbol is returned.

  \*  if NSYM = 2, both symbols have code length 1.

  \*  if NSYM = 3, the code lengths for the symbols are 1, 2, 2 in the order they appear in the representation of the simple prefix code.

  \*  if NSYM = 4, the code lengths \(in order of symbols decoded\) depend on the tree-select bit: 2, 2, 2, 2 \(tree-select bit 0\), or 1, 2, 3, 3 \(tree-select bit 1\).
```

## 3.5.  复杂前缀码

A complex prefix code is a canonical prefix code, defined by the sequence of code lengths, as discussed in Section 3.2.  For even greater compactness, the code length sequences themselves are compressed using a prefix code.  The alphabet for code lengths is as follows:

```
0..15: Represent code lengths of 0..15
 16: Copy the previous non-zero code length 3..6 times.
     The next 2 bits indicate repeat length
           (0 = 3, ... , 3 = 6)
     If this is the first code length, or all previous
     code lengths are zero, a code length of 8 is
     repeated 3..6 times.
     A repeated code length code of 16 modifies the
     repeat count of the previous one as follows:
        repeat count = (4 * (repeat count - 2)) +
                       (3..6 on the next 2 bits)
     Example:  Codes 7, 16 (+2 bits 11), 16 (+2 bits 10)
               will expand to 22 code lengths of 7
               (1 + 4 * (6 - 2) + 5)
 17: Repeat a code length of 0 for 3..10 times.
     The next 3 bits indicate repeat length
           (0 = 3, ... , 7 = 10)
     A repeated code length code of 17 modifies the
     repeat count of the previous one as follows:
        repeat count = (8 * (repeat count - 2)) +
                       (3..10 on the next 3 bits)
```

```
Note that a code of 16 that follows an immediately preceding 16 modifies the previous repeat count, which becomes the new repeat count.  The same is true for a 17 following a 17.  A sequence of three or more 16 codes in a row or three of more 17 codes in a row is possible, modifying the count each time.  Only the final repeat count is used.  The modification only applies if the same code follows.  A 16 repeat does not modify an immediately preceding 17 count nor vice versa.
```

A code length of 0 indicates that the corresponding symbol in the alphabet will not occur in the compressed data, and it should not participate in the prefix code construction algorithm given earlier. A complex prefix code must have at least two non-zero code lengths. The bit lengths of the prefix code over the code length alphabet are compressed with the following variable-length code \(as it appears in the compressed data, where the bits are parsed from right to left\):

```
Symbol   Code
------   ----
0          00
1        0111
2         011
3          10
4          01
5        1111
```

We can now define the format of the complex prefix code as follows:

* 2 bits: HSKIP, the number of skipped code lengths, can have values of 0, 2, or 3.  The skipped lengths are taken to be zero.  \(An HSKIP of 1 indicates a Simple prefix code.\)
* Code lengths for symbols in the code length alphabet given just above, in the order: 1, 2, 3, 4, 0, 5, 17, 6, 16, 7, 8, 9, 10, 11, 12, 13, 14, 15.  If HSKIP is 2, then the code lengths for symbols 1 and 2 are zero, and the first code length is for symbol 3.  If HSKIP is 3, then the code length for symbol 3 is also zero, and the first code length is for symbol 4.

```
  The code lengths of code length symbols are between 0 and 5, and they are represented with 2..4 bits according to the variable- length code above.  A code length of 0 means the corresponding code length symbol is not used.

  If HSKIP is 2 or 3, a respective number of leading code lengths are implicit zeros and are not present in the code length sequence above.

  If there are at least two non-zero code lengths, any trailing zero code lengths are omitted, i.e., the last code length in the sequence must be non-zero.  In this case, the sum of \(32 &gt;&gt; code length\) over all the non-zero code lengths must equal to 32.

  If the lengths have been read for the entire code length alphabet and there was only one non-zero code length, then the prefix code has one symbol whose code has zero length.  In this case, that symbol results in no bits being emitted by the compressor and no bits consumed by the decompressor.  That single symbol is immediately returned when this code is decoded.  An example of where this occurs is if the entire code to be represented has symbols of length 8.  For example, a literal code that represents all literal values with equal probability.  In this case the single symbol is 16, which repeats the previous length.  The previous length is taken to be 8 before any code length code lengths are read.
```

* Sequence of code length symbols, which is at most the size of the alphabet, encoded using the code length prefix code.  Any trailing 0 or 17 must be omitted, i.e., the last encoded code length symbol must be between 1 and 16.  The sum of \(32768 &gt;&gt; code length\) over all the non-zero code lengths in the alphabet, including those encoded using repeat code\(s\) of 16, must be equal to 32768.  If the number of times to repeat the previous length or repeat a zero length would result in more lengths in total than the number of symbols in the alphabet, then the stream should be rejected as invalid.



