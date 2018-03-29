# 5.  Encoding of Literal Insertion Lengths and Copy Lengths

As described in Section 2, the literal insertion lengths and backward copy lengths are encoded using a single prefix code.  This section provides the details to this encoding. Each &lt;insertion length, copy length&gt; pair in the compressed data part of a meta-block is represented with the following triplet:

```
<insert-and-copy length code, insert extra bits, copy extra bits>
```

The insert-and-copy length code, the insert extra bits, and the copy extra bits are encoded back-to-back, the insert-and-copy length code is encoded using a prefix code over the insert-and-copy length code alphabet, while the extra bits values are encoded as fixed-width integer values.  The number of insert and copy extra bits can be 0..24, and they are dependent on the insert-and-copy length code.

Some of the insert-and-copy length codes also express the fact that the distance symbol of the distance in the same command is 0, i.e., the distance component of the command is the same as that of the previous command.  In this case, the distance code and extra bits for the distance are omitted from the compressed data stream.

We describe the insert-and-copy length code alphabet in terms of the \(not directly used\) insert length code and copy length code alphabets.  The symbols of the insert length code alphabet, along with the number of insert extra bits, and the range of the insert lengths are as follows:

```
        Extra              Extra               Extra
   Code Bits Lengths  Code Bits Lengths   Code Bits Lengths
   ---- ---- -------  ---- ---- -------   ---- ---- -------
    0    0     0       8    2   10..13    16    6   130..193
    1    0     1       9    2   14..17    17    7   194..321
    2    0     2      10    3   18..25    18    8   322..577
    3    0     3      11    3   26..33    19    9   578..1089
    4    0     4      12    4   34..49    20   10   1090..2113
    5    0     5      13    4   50..65    21   12   2114..6209
    6    1    6,7     14    5   66..97    22   14   6210..22593
    7    1    8,9     15    5   98..129   23   24   22594..16799809
```

The symbols of the copy length code alphabet, along with the number of copy extra bits, and the range of copy lengths are as follows:

```
        Extra              Extra               Extra
   Code Bits Lengths  Code Bits Lengths   Code Bits Lengths
   ---- ---- -------  ---- ---- -------   ---- ---- -------
    0    0     2       8    1    10,11    16    5   70..101
    1    0     3       9    1    12,13    17    5   102..133
    2    0     4      10    2    14..17   18    6   134..197
    3    0     5      11    2    18..21   19    7   198..325
    4    0     6      12    3    22..29   20    8   326..581
    5    0     7      13    3    30..37   21    9   582..1093
    6    0     8      14    4    38..53   22   10   1094..2117
    7    0     9      15    4    54..69   23   24   2118..16779333
```



To convert an insert-and-copy length code to an insert length code and a copy length code, the following table can be used:

```
       Insert
       length        Copy length code
       code       0..7       8..15     16..23
              +----------+----------+
              |          |          |
        0..7  |   0..63  |  64..127 | <--- distance symbol 0
              |          |          |
              +----------+----------+----------+
              |          |          |          |
        0..7  | 128..191 | 192..255 | 384..447 |
              |          |          |          |
              +----------+----------+----------+
              |          |          |          |
        8..15 | 256..319 | 320..383 | 512..575 |
              |          |          |          |
              +----------+----------+----------+
              |          |          |          |
       16..23 | 448..511 | 576..639 | 640..703 |
              |          |          |          |
              +----------+----------+----------+
```

First, look up the cell with the 64 value range containing the insert-and-copy length code; this gives the insert length code and the copy length code ranges, both 8 values long.  The copy length code within its range is determined by bits 0..2 \(counted from the lsb\) of the insert-and-copy length code.  The insert length code within its range is determined by bits 3..5 \(counted from the lsb\) of the insert-and-copy length code.  Given the insert length and copy length codes, the actual insert and copy lengths can be obtained by reading the number of extra bits given by the tables above.

If the insert-and-copy length code is between 0 and 127, the distance code of the command is set to zero \(the last distance reused\).



