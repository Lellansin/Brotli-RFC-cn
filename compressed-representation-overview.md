# 2. Compressed Representation Overview

A compressed data set consists of a header and a series of meta- blocks.  Each meta-block decompresses to a sequence of 0 to 16,777,216 \(16 MiB\) uncompressed bytes.  The final uncompressed data is the concatenation of the uncompressed sequences from each meta- block.

The header contains the size of the sliding window that was used during compression.  The decompressor must retain at least that amount of uncompressed data prior to the current position in the stream, in order to be able to decompress what follows.  The sliding window size is a power of two, minus 16, where the power is in the range of 10 to 24.  The possible sliding window sizes range from 1 KiB - 16 B to 16 MiB - 16 B.



