1.  Introduction

1.1.  Purpose

   The purpose of this specification is to define a lossless compressed
   data format that:

      *  is independent of CPU type, operating system, file system, and
         character set; hence, it can be used for interchange.

      *  can be produced or consumed, even for an arbitrarily long,
         sequentially presented input data stream, using only an a
         priori bounded amount of intermediate storage; hence, it can be
         used in data communications or similar structures, such as Unix
         filters.

      *  compresses data with a compression ratio comparable to the best
         currently available general-purpose compression methods, in
         particular, considerably better than the gzip program.

      *  decompresses much faster than current LZMA implementations.

   The data format defined by this specification does not attempt to:

      *  allow random access to compressed data.

      *  compress specialized data (e.g., raster graphics) as densely as
         the best currently available specialized algorithms.

   This document is the authoritative specification of the brotli
   compressed data format.  It defines the set of valid brotli
   compressed data streams and a decoder algorithm that produces the
   uncompressed data stream from a valid brotli compressed data stream.

