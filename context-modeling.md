# 7.  上下文建模

As described in Section 2, the prefix tree used to encode a literal byte or a distance code depends on the block type and the context ID. This section specifies how to compute the context ID for a particular literal and distance code and how to encode the context map that maps a &lt;block type, context ID&gt; pair to the index of a prefix code in the array of literal and distance prefix codes.

## 7.1.  Context Modes and Context ID Lookup for Literals

The context for encoding the next literal is defined by the last two bytes in the stream \(p1, p2, where p1 is the most recent byte\), regardless of whether these bytes are produced by uncompressed meta- blocks, backward references, static dictionary references, or by literal insertions.  At the start of the stream, p1 and p2 are initialized to zero.

There are four methods, called context modes, to compute the Context ID:

* LSB6, where the Context ID is the value of six least significant bits of p1,
* MSB6, where the Context ID is the value of six most significant bits of p1,
* UTF8, where the Context ID is a complex function of p1, p2, optimized for text compression, and
* Signed, where Context ID is a complex function of p1, p2, optimized for compressing sequences of signed integers.

The Context ID for the UTF8 and Signed context modes is computed using the following lookup tables Lut0, Lut1, and Lut2.

```
Lut0 :=
   0,  0,  0,  0,  0,  0,  0,  0,  0,  4,  4,  0,  0,  4,  0,  0,
   0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,
   8, 12, 16, 12, 12, 20, 12, 16, 24, 28, 12, 12, 32, 12, 36, 12,
  44, 44, 44, 44, 44, 44, 44, 44, 44, 44, 32, 32, 24, 40, 28, 12,
  12, 48, 52, 52, 52, 48, 52, 52, 52, 48, 52, 52, 52, 52, 52, 48,
  52, 52, 52, 52, 52, 48, 52, 52, 52, 52, 52, 24, 12, 28, 12, 12,
  12, 56, 60, 60, 60, 56, 60, 60, 60, 56, 60, 60, 60, 60, 60, 56,
  60, 60, 60, 60, 60, 56, 60, 60, 60, 60, 60, 24, 12, 28, 12,  0,
   0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1,
   0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1,
   0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1,
   0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1,
   2, 3, 2, 3, 2, 3, 2, 3, 2, 3, 2, 3, 2, 3, 2, 3,
   2, 3, 2, 3, 2, 3, 2, 3, 2, 3, 2, 3, 2, 3, 2, 3,
   2, 3, 2, 3, 2, 3, 2, 3, 2, 3, 2, 3, 2, 3, 2, 3,
   2, 3, 2, 3, 2, 3, 2, 3, 2, 3, 2, 3, 2, 3, 2, 3

Lut1 :=
   0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
   0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
   0, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,
   2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 1, 1, 1, 1, 1, 1,
   1, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2,
   2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 1, 1, 1, 1, 1,
   1, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3,
   3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 1, 1, 1, 1, 0,
   0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
   0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
   0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
   0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
   0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
   0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
   2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2,
   2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2

Lut2 :=
   0, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,
   2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2,
   2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2,
   2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2,
   3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3,
   3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3,
   3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3,
   3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3,
   4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4,
   4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4,
   4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4,
   4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4,
   5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5,
   5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5,
   5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5,
   6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 7
```

The lengths and the CRC-32 check values \(see Appendix C\) of each of these tables as a sequence of bytes are as follows:

```
Table    Length    CRC-32
-----    ------    ------
Lut0     256       0x8e91efb7
Lut1     256       0xd01a32f4
Lut2     256       0x0dd7a0d6
```

Given p1 is the last uncompressed byte and p2 is the second-to-last uncompressed byte, the context IDs can be computed as follows:

```
For LSB6:    Context ID = p1 & 0x3f
For MSB6:    Context ID = p1 >> 2
For UTF8:    Context ID = Lut0[p1] | Lut1[p2]
For Signed:  Context ID = (Lut2[p1] << 3) | Lut2[p2]
```

From the lookup tables defined above and the operations to compute the context IDs, we can see that context IDs for literals are in the range of 0..63.

The context modes LSB6, MSB6, UTF8, and Signed are denoted by integers 0, 1, 2, 3.

A context mode is defined for each literal block type and they are stored in a consecutive array of bits in the meta-block header, always two bits per block type.

## 7.2.  Context ID for Distances

The context for encoding a distance code is defined by the copy length corresponding to the distance.  The context IDs are 0, 1, 2, and 3 for copy lengths 2, 3, 4, and more than 4, respectively.

## 7.3.  Encoding of the Context Map

There are two context maps, one for literals and one for distances. The size of the context map is 64 \* NBLTYPESL for literals, and 4 \* NBLTYPESD for distances.  Each value in the context map is an integer between 0 and 255, indicating the index of the prefix code to be used when encoding the next literal or distance.

The context maps are two-dimensional matrices, encoded as one- dimensional arrays:

```
CMAPL[0..(64 * NBLTYPESL - 1)]
CMAPD[0..(4 * NBLTYPESD - 1)]
```

The index of the prefix code for encoding a literal or distance code with block type, BTYPE\_x, and context ID, CIDx, is:

```
index of literal prefix code = CMAPL[64 * BTYPE_L + CIDL]
index of distance prefix code = CMAPD[4 * BTYPE_D + CIDD]
```

The values of the context map are encoded with the combination of run length encoding for zero values and prefix coding.  Let RLEMAX denote the number of run length codes and NTREES denote the maximum value in the context map plus one.  NTREES must equal the number of different values in the context map; in other words, the different values in the context map must be the \[0..NTREES-1\] interval.  The alphabet of the prefix code has the following RLEMAX + NTREES symbols:

```
0: value zero
1: repeat a zero 2 to 3 times, read 1 bit for repeat length
2: repeat a zero 4 to 7 times, read 2 bits for repeat length
...
RLEMAX: repeat a zero (1 << RLEMAX) to (1 << (RLEMAX+1))-1
        times, read RLEMAX bits for repeat length
RLEMAX + 1: value 1
...
RLEMAX + NTREES - 1: value NTREES - 1
```

If RLEMAX = 0, the run length coding is not used and the symbols of the alphabet are directly the values in the context map.  We can now define the format of the context map \(the same format is used for literal and distance context maps\):

```
1..5 bits: RLEMAX, 0 is encoded with one 0 bit, and values 1..16
           are encoded with bit pattern xxxx1 (so 01001 is 5)

Prefix code with alphabet size NTREES + RLEMAX

Context map size values encoded with the above prefix code and run
   length coding for zero values.  If a run length would result in
   more lengths in total than the size of the context map, then
   the stream should be rejected as invalid.

1 bit:  IMTF bit, if set, we do an inverse move-to-front transform
        on the values in the context map to get the prefix code
        indexes.
```

Note that RLEMAX may be larger than the value necessary to represent the longest sequence of zero values.  Also, the NTREES value is encoded right before the context map as described in Section 9.2.

We define the inverse move-to-front transform used in this specification by the following C language function:

```cpp
void InverseMoveToFrontTransform(uint8_t* v, int v_len) {
   uint8_t mtf[256];
   int i;
   for (i = 0; i < 256; ++i) {
      mtf[i] = (uint8_t)i;
   }
   for (i = 0; i < v_len; ++i) {
      uint8_t index = v[i];
      uint8_t value = mtf[index];
      v[i] = value;
      for (; index; --index) {
         mtf[index] = mtf[index - 1];
      }
      mtf[0] = value;
   }
}
```

Note that the inverse move-to-front transform will not produce values outside the \[0..NTREES-1\] interval.

