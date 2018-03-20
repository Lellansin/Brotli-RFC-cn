# 4.  Encoding of Distances

As described in Section 2, one component of a compressed meta-block is a sequence of backward distances.  In this section, we provide the details to the encoding of distances.

Each distance in the compressed data part of a meta-block is represented with a pair &lt;distance code, extra bits&gt;.  The distance code and the extra bits are encoded back-to-back, the distance code is encoded using a prefix code over the distance alphabet, while the extra bits value is encoded as a fixed-width integer value.  The number of extra bits can be 0..24, and it is dependent on the distance code.

To convert a distance code and associated extra bits to a backward distance, we need the sequence of past distances and two additional parameters: the number of "postfix bits", denoted by NPOSTFIX \(0..3\), and the number of direct distance codes, denoted by NDIRECT \(0..120\). Both of these parameters are encoded in the meta-block header.  We will also use the following derived parameter:

```
POSTFIX_MASK = (1 << NPOSTFIX) - 1
```

The first 16 distance symbols are special symbols that reference past distances as follows:

```
   0: last distance
   1: second-to-last distance
   2: third-to-last distance
   3: fourth-to-last distance
   4: last distance - 1
   5: last distance + 1
   6: last distance - 2
   7: last distance + 2
   8: last distance - 3
   9: last distance + 3
  10: second-to-last distance - 1
  11: second-to-last distance + 1
  12: second-to-last distance - 2
  13: second-to-last distance + 2
  14: second-to-last distance - 3
  15: second-to-last distance + 3
```

The ring buffer of the four last distances is initialized by the values 16, 15, 11, and 4 \(i.e., the fourth-to-last is set to 16, the third-to-last to 15, the second-to-last to 11, and the last distance to 4\) at the beginning of the \*stream\* \(as opposed to the beginning of the meta-block\), and it is not reset at meta-block boundaries. When a distance symbol 0 appears, the distance it represents \(i.e., the last distance in the sequence of distances\) is not pushed to the ring buffer of last distances; in other words, the expression "second-to-last distance" means the second-to-last distance that was not represented by a 0 distance symbol \(and similar for "third-to- last distance" and "fourth-to-last distance"\).  Similarly, distances that represent static dictionary words \(see Section 8\) are not pushed to the ring buffer of last distances.

If a special distance symbol resolves to a zero or negative value, the stream should be rejected as invalid.

If NDIRECT is greater than zero, then the next NDIRECT distance symbols, from 16 to 15 + NDIRECT, represent distances from 1 to NDIRECT.  Neither the special distance symbols nor the NDIRECT direct distance symbols are followed by any extra bits.

Distance symbols 16 + NDIRECT and greater all have extra bits, where the number of extra bits for a distance symbol "dcode" is given by the following formula:

```
ndistbits = 1 + ((dcode - NDIRECT - 16) >> (NPOSTFIX + 1))
```

   The maximum number of extra bits is 24; therefore, the size of the distance symbol alphabet is \(16 + NDIRECT + \(48 &lt;&lt; NPOSTFIX\)\).

Given a distance symbol "dcode" \(&gt;= 16 + NDIRECT\), and extra bits "dextra", the backward distance is given by the following formula:

```
hcode = (dcode - NDIRECT - 16) >> NPOSTFIX
lcode = (dcode - NDIRECT - 16) & POSTFIX_MASK
offset = ((2 + (hcode & 1)) << ndistbits) - 4
distance = ((offset + dextra) << NPOSTFIX) + lcode + NDIRECT + 1
```



