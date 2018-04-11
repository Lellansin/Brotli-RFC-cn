# 7.  Context 建模

在第二节中已经叙述过, 用于编码字面 byte 或者距离码的前缀数取决于 block 类型以及 Context ID。 本章将指定如何计算特定字面、距离码的 Context ID 以及如何编码生成一个 Context Map，该 Map 可以在字面序列和距离前缀码的数组中将 &lt;block type, context ID&gt; 结构映射到到前缀码的索引。

## 7.1. 字面序列的 Context 模式和 Context ID 查询

用于编码下一个字面序列的 Context 是由 Stream 中的最后两个 byte 决定的 \(p1, p2, 其中 p1 是最新的 byte\), 不管这些 byte 是通过未压缩的 meta-block、反向引用、静态字典引用或者字面序列插入产生的。在 Stream 的开头，p1 和 p2 被初始化为 0。

以下四种方法，被称为 Context 模式, 用以计算 Context ID:

* LSB6, 其中  Context ID 是 p1 的 6 个最低有效位的值，
* MSB6, 其中 Context ID 是 p1 的 6 个最高有效位的值，
* UTF8, 其中 Context ID 是根据 p1, p2 进行一个复杂计算得出, 该优化用于文本压缩, 以及
* Signed, 其中 Context ID 根据 p1, p2 进行一个复杂计算得出, 用于压缩 有符号的整数序列。

The Context ID for the UTF8 and Signed context modes is computed using the following lookup tables Lut0, Lut1, and Lut2.

使用以下查找表 Lut0，Lut1 和 Lut2 来计算 UTF8 和 Signed Context Mode 的 Context ID。

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

其中每个表的长度和  CRC-32 校验值 \(见附录 C\)  如下:

```
Table    Length    CRC-32
-----    ------    ------
Lut0     256       0x8e91efb7
Lut1     256       0xd01a32f4
Lut2     256       0x0dd7a0d6
```

Given p1 is the last uncompressed byte and p2 is the second-to-last uncompressed byte, the context IDs can be computed as follows:

给定 p1 是最后一个未压缩字节，p2 是倒数第二个未压缩字节，Context ID 可按如下方式计算：

```
For LSB6:    Context ID = p1 & 0x3f
For MSB6:    Context ID = p1 >> 2
For UTF8:    Context ID = Lut0[p1] | Lut1[p2]
For Signed:  Context ID = (Lut2[p1] << 3) | Lut2[p2]
```

从上面定义的查找表和计算 Context ID 的操作中，我们可以了解到文字的 Context ID在 0到63 的范围内。

而各个模式 LSB6, MSB6, UTF8, 以及 Signed 由整数的 0, 1, 2, 3 表示。



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

