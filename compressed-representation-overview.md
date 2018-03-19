# 2. 压缩表述概览

一个压缩数据集由一个头部（header）和一系列元块（meta-block）组成。 每个元块可以解压缩为 0 到16,777,216（16 MiB）的字节序列。 最终的源数据是来自每个元块的解压序列的串联。

头部包含压缩过程中使用的滑动窗口（sliding window）的大小。 解压器必须保留那些未压缩的数据量至少在流中的当前位置之前，以便解压后面的内容。 滑动窗口的大小是 2 的 n 次幂减去16，其中 n 在 10 到 24 的范围内。所以滑动窗口的大小范围可能在 1 Kib 减 16B 到 16 Mib 减 16B 之间。

每个元块通过 LZ77 算法（Lempel-Ziv 1977，\[[LZ77](https://tools.ietf.org/html/rfc7932#ref-LZ77)\]）和霍夫曼编码的组合来压缩。 霍夫曼编码的结果在这里被称为 “前缀码”。 每个元块的前缀码与之前和之后的无关；LZ77 算法可能会复用前一个元块中出现过的字符串的引用，这取决于之前的未压缩字节的滑动窗口大小。 此外，在 brotli 格式中，字符串引用可能会替代静态字典条目的引用。

每个元块由两部分组成：元块头（描述压缩数据的表述）和被压缩的数据。 其中被压缩数据由一系列命令（command）组成。 每个命令有两个部分：（在滑动窗口中尚未检测可复用字符串的）字面的字节序列部分和指向复用字符串的指针部分，指针部分可以表述为 `<长度, 后向距离>` 的对。 命令中的字面字节序列长度可以为零。 可以复用的字符串最小长度为 2，但元块中的最后一个命令只允许有字面序列，且没有复用字符串的指针。

压缩数据中的每个命令都使用三类前缀代码表示：

1. 一组前缀码用于表示字面序列长度（也称为字面插入长度）和后向复制长度。 也就是说，单个码代表两种长度：一种是字面序列，一种是反向复制。
2. 一组前缀码用于表示字面序列。
3. 一组前缀码用于表示距离。

每一个元块的前缀码描述紧凑的出现在每个元块压缩数据之前的头部中。 插入-复制长度和距离的前缀码会跟随在额外的比特中，其值累加到由代码计算的基本值上。 额外的比特的编号由代码决定。

那么，一个元块命令可以看做一系列前缀码：

```
Insert-and-copy length, literal, literal, ..., literal, distance
```

其中，`插入-复制长度`（Insert-and-copy）定义了插入的长度和复制的长度。 插入的长度决定了紧接着的字面序列（literal）长度。 距离（distance）定义了要往回走多远可以找到复用的起始地址，复制长度决定要复制的字节数。按照这个格式产生未压缩的数据是这样的字节序列：

```
literal, literal, ..., literal, copy, copy, ..., copy
```

其中字面字节长度和复制字节长度由`插入-复制长度`前缀码确定。 （取自静态字典条目的复制字节的长度可以与复制长度不同。）

如果元块未压缩的总长度已经满了（到了元块长度的上限），则元块中的最后一个命令可以以最后的字面序列为结尾。 在这种情况下，最后一个命令中没有距离，并且复制长度会被忽略。

每个种类（category）可以有多个前缀码，其中种类中下一个元素所使用的前缀码，取决于先于那个元素之前的压缩流的上下文。 其中上下文是由三个当前块类型组成，不过每个种类只有一个类型。 块类型的范围是 `0..255` 。 对于每个种类，都有一个计数，统计该种类维持了多少个使用当前块类型进行解码的元素。 一旦该计数被消耗，再处理该类别的下一个元素之前，一个新的块类型和块计数会从马上从流中读取出来，并使用这个新的块类型。

The insert-and-copy block type directly determines which prefix code to use for the next insert-and-copy length.  For the literal and distance elements, the respective block type is used in combination with other context information to determine which prefix code to use for the next element.

Consider the following example:

```
   (IaC0, L0, L1, L2, D0)(IaC1, D1)(IaC2, L3, L4, D2)(IaC3, L5, D3)
```

 The meta-block here has four commands, contained in parentheses for clarity, where each of the three categories of symbols within these commands can be interpreted using different block types.  Here we separate out each category as its own sequence to show an example of block types assigned to those elements.  Each square-bracketed group is a block that uses the same block type:

```

    [IaC0, IaC1][IaC2, IaC3]  <-- insert-and-copy: block types 0 and 1

    [L0, L1][L2, L3, L4][L5]  <-- literals: block types 0, 1, and 0

    [D0][D1, D2, D3]          <-- distances: block types 0 and 1

```

The subsequent blocks within each block category must have different block types, but we see that block types can be reused later in the meta-block.  The block types are numbered from 0 to the maximum block type number of 255, and the first block of each block category is type 0.  The block structure of a meta-block is represented by the sequence of block-switch commands for each block category, where a block-switch command is a pair &lt;block type, block count&gt;.  The block- switch commands are represented in the compressed data before the start of each new block using a prefix code for block types and a separate prefix code for block counts for each block category.  For the above example, the physical layout of the meta-block is then:

```

      IaC0 L0 L1 LBlockSwitch(1, 3) L2 D0 IaC1 DBlockSwitch(1, 3) D1
      IaCBlockSwitch(1, 2) IaC2 L3 L4 D2 IaC3 LBlockSwitch(0, 1) L5 D3

```

where xBlockSwitch\(t, n\) switches to block type t for a count of n elements.  In this example, note that DBlockSwitch\(1, 3\) immediately precedes the next required distance, D1.  It does not follow the last distance of the previous block, D0.  Whenever an element of a category is needed, and the block count for that category has reached zero, then a new block type and count are read from the stream just before reading that next element.

The block-switch commands for the first blocks of each category are not part of the meta-block compressed data.  Instead, the first block type is defined to be 0, and the first block count for each category is encoded in the meta-block header.  The prefix codes for the block types and counts, a total of six prefix codes over the three categories, are defined in a compact form in the meta-block header.

Each category of value \(insert-and-copy lengths, literals, and distances\) can be encoded with any prefix code from a collection of prefix codes belonging to the same category appearing in the meta- block header.  The particular prefix code used can depend on two factors: the block type of the block the value appears in and the context of the value.  In the case of the literals, the context is the previous two bytes in the uncompressed data; and in the case of distances, the context is the copy length from the same command.  For insert-and-copy lengths, no context is used and the prefix code depends only on the block type.  In the case of literals and distances, the context is mapped to a context ID in the range 0..63 for literals and 0..3 for distances.  The matrix of the prefix code indexes for each block type and context ID, called the context map, is encoded in a compact form in the meta-block header.

For example, the prefix code to use to decode L2 depends on the block type \(1\), and the literal context ID determined by the two uncompressed bytes that were decoded from L0 and L1.  Similarly, the prefix code to use to decode D0 depends on the block type \(0\) and the distance context ID determined by the copy length decoded from IaC0. The prefix code to use to decode IaC3 depends only on the block type \(1\).

In addition to the parts listed above \(prefix code for insert-and- copy lengths, literals, distances, block types, block counts, and the context map\), the meta-block header contains the number of uncompressed bytes coded in the meta-block and two additional parameters used in the representation of match distances: the number of postfix bits and the number of direct distance codes.

A compressed meta-block may be marked in the header as the last meta- block, which terminates the compressed stream.

A meta-block may, instead, simply store the uncompressed data directly as bytes on byte boundaries with no coding or matching strings.  In this case, the meta-block header information only contains the number of uncompressed bytes and the indication that the meta-block is uncompressed.  An uncompressed meta-block cannot be the last meta-block.

A meta-block may also be empty, which generates no uncompressed data at all.  An empty meta-block may contain metadata information as bytes starting on byte boundaries, which are not part of either the sliding window or the uncompressed data.  Thus, these metadata bytes cannot be used to create matching strings in subsequent meta-blocks and are not used as context bytes for literals.




