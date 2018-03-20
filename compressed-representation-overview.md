# 2. 压缩表述概览

一个压缩数据集由一个 header （头部）和一系列 meta-block（元块）组成。 每个 meta-block 可以解压缩为 0 到16,777,216（16 MiB）的字节序列。 最终的源数据是来自每个 meta-block 的解压序列的串联。

header 包含压缩过程中使用的滑动窗口（sliding window）的大小。 解压器必须保留那些未压缩的数据量至少在流中的当前位置之前，以便解压后面的内容。 滑动窗口的大小是 2 的 n 次幂减去16，其中 n 在 10 到 24 的范围内。所以滑动窗口的大小范围可能在 1 Kib 减 16B 到 16 Mib 减 16B 之间。

每个 meta-block 通过 LZ77 算法（Lempel-Ziv 1977，\[[LZ77](https://tools.ietf.org/html/rfc7932#ref-LZ77)\]）和霍夫曼编码的组合来压缩。 霍夫曼编码的结果在这里被称为 “前缀码”。 每个 meta-block 的前缀码与之前和之后的无关；LZ77 算法可能会复用前一个 meta-block 中出现过的字符串的引用，这取决于之前的未压缩字节的滑动窗口大小。 此外，在 brotli 格式中，字符串引用可能会替代静态字典条目的引用。

每个 meta-block 由两部分组成：meta-block 的 header（描述压缩数据的表述）和被压缩的数据。 其中被压缩数据由一系列命令（command）组成。 每个命令有两个部分：（在滑动窗口中尚未检测可复用字符串的）字面的字节序列部分和指向复用字符串的指针部分，指针部分可以表述为 `<长度, 后向距离>` 的对。 命令中的字面字节序列长度可以为零。 可以复用的字符串最小长度为 2，但 meta-block 中的最后一个命令只允许有字面序列，且没有复用字符串的指针。

压缩数据中的每个命令都使用三类前缀代码表示：

1. 一组前缀码用于表示字面序列长度（也称为字面插入长度）和后向复制长度。 也就是说，单个码代表两种长度：一种是字面序列，一种是反向复制。
2. 一组前缀码用于表示字面序列。
3. 一组前缀码用于表示距离。

每一个 meta-block 的前缀码描述紧凑的出现在每个 meta-block 压缩数据之前的 header 中。 插入-复制长度和距离的前缀码会跟随在额外的 bits 中，其值累加到由代码计算的基本值上。 额外的 bits 的编号由代码决定。

那么，一个 meta-block 命令可以看做一系列前缀码：

```
Insert-and-copy length, literal, literal, ..., literal, distance
```

其中，`插入-复制长度`（Insert-and-copy）定义了插入的长度和复制的长度。 插入的长度决定了紧接着的字面序列（literal）长度。 距离（distance）定义了要往回走多远可以找到复用的起始地址，复制长度决定要复制的字节数。按照这个格式产生未压缩的数据是这样的字节序列：

```
literal, literal, ..., literal, copy, copy, ..., copy
```

其中字面字节长度和复制字节长度由`插入-复制长度`前缀码确定。 （取自静态字典条目的复制字节的长度可以与复制长度不同。）

如果 meta-block 未压缩的总长度已经满了（到了 meta-block 长度的上限），则 meta-block 中的最后一个命令可以以最后的字面序列为结尾。 在这种情况下，最后一个命令中没有距离，并且复制长度会被忽略。

每个种类（category）可以有多个前缀码，其中种类中下一个元素所使用的前缀码，取决于先于那个元素之前的压缩流的上下文。 其中上下文是由三个当前 block 类型组成，不过每个种类只有一个类型。 block 类型的范围是 `0..255` 。 对于每个种类，都有一个计数，统计该种类维持了多少个使用当前 block 类型进行解码的元素。 一旦该计数被消耗，再处理该类别的下一个元素之前，一个新的 block 类型和 block 计数会从马上从流中读取出来，并使用这个新的 block 类型。

`插入-复制` block 类型直接确定在下一个`插入-复制长度`中使用哪个前缀码。 对于字面序列和距离元素，相应的块类型用于与其他上下文信息结合，以确定下一个元素用哪个前缀码。

考虑如下的例子：

```
   (IaC0, L0, L1, L2, D0)(IaC1, D1)(IaC2, L3, L4, D2)(IaC3, L5, D3)
```

注：`IaC -> insert-and-copy`, `L -> Literal`, `D -> distance`

这里的 meta-block 有四个命令，为了清楚起见先括号括了起来，其中的三个种类的符号中的每一个都可以使用不同的 block 类型来解释。 那么，我们将他们按其自身的序列分开类展示 block 类型分配到这些元素的示例。 下面每个方括号组都是一个使用相同类型的 block：

```
    [IaC0, IaC1][IaC2, IaC3]  <-- insert-and-copy: block 类型 0 and 1

    [L0, L1][L2, L3, L4][L5]  <-- literals: block 类型 0, 1, and 0

    [D0][D1, D2, D3]          <-- distances: block 类型 0 and 1
```

每个 block 种类中后续的 block 必须具有不同的类型，但我们可以看到 block 类型可以稍后在 meta-block 中复用。 block 类型的编号从 0 到 255，同时每个 block 种类的第一个 block 是类型 0。meta-block 的 block 结构由每个 block 种类的 block-switch 命令序列表示 ，其中 block-swtich 命令是一对 `<block 类型, block 计数>`。 对于每个 block 种类，其 block-switch 命令位于压缩数据中，在每个新的 block 之前，这里新的 block 则使用一个前缀码表示类型、一个单独的前缀码表示计数。 对于上面的例子，meta-block 的物理布局是：

```
      IaC0 L0 L1 LBlockSwitch(1, 3) L2 D0 IaC1 DBlockSwitch(1, 3) D1
      IaCBlockSwitch(1, 2) IaC2 L3 L4 D2 IaC3 LBlockSwitch(0, 1) L5 D3
```

其中 `xBlockSwitch(t, n)` 切换到 block 类型 t，并计数 n 个元素。 在本例中，请注意 `DBlockSwitch(1, 3)` 紧接在下一个所需距离 D1 之前。 它没有跟在上一个 block 的最后一个距离 \(D0\) 之后。每当需要一个种类的元素，并且该种类的 block 计数已经达到零，那么在读取下一个元素之前，将从流中读取新的 block 类型和计数。

每个类别的第一个 block 的 block-switch 命令不是 meta-block 压缩数据的一部分。 相反，第一个 block 类型被定义为 0，并且每个种类的第一个 block 计数被编码在 meta-block 的 header 中。 block 的类型和计数，在这 3 个种类上共有 6 个前缀码，被紧凑格式的定义在 meta-block header 中。

每个种类的值（插入-复制长度、字面序列和距离）都可以使用 meta-block 的 header 中出现过的，同一种类的前缀码集合中的任何一个前缀码进行编码。 所使用的特定前缀码可以取决于两个因素：值出现的 block 的类型和值的上下文 \(the block type of the block the value appears in and the context of the value\)。 在字面序列的类型下，上下文是未压缩数据中的前两个字节；而在距离的类型下，上下文是来自相同命令的复制长度。 对于插入-复制长度，不使用上下文，前缀码仅依赖于 block 类型。 在字面序列和距离的情况下，上下文被映射到范围在 `0..63` 的上下文 ID 和范围在 `0..3` 的距离中。 每个 block 类型和上下文ID的前缀码索引的矩阵都被称为上下文映射，并以紧凑的形式被编码在 meta-block header 中。

例如，block 类型（1）决定了用于解码 L2 的前缀码 ，以及由 L0 和 L1 解码的两个未压缩字节决定了字面序列上下文 ID。 类似地，block 类型（0）决定了用于解码 D0 的前缀码，从 IaC0 解码的复制长度决定了距离上下文 ID。而 用于解码 IaC3 的前缀码仅取决于block 类型（1）。

除了上面列出的部分（插入-复制长度、字面序列、距离、block 类型、block 计数和上下文映射的前缀码）之外，meta-block header 还包含未压缩字节数，编码在 meta-block 中，以及用来表示匹配距离的两个附加参数：后缀 bits 的数量和直接距离代码（direct distance codes）的数量。

一个压缩后的 meta-block 可以在 header 中被标记为最后的 meta-block，这样就可以终止压缩流。

此外，meta-block 也可以将未压缩的数据直接以字节序列的形式存储在字节边界上，而不需要编码或匹配字符串。 在这种情况下， meta-block header 信息仅包含未压缩字节的数量以及 meta-block 未压缩的标记。 未压缩的 meta-block 不能是最后一个 meta-block。

meta-block 也可以是空的，不产生任何未压缩的数据。 一个空的 meta-block 可能包含以字节边界开始的字节的 meta 数据信息，该内容即不是滑动窗口的一部分也不是未压缩数据的一部分。 因此，这些 meta 数据字节不能用在随后的 meta-block 中创建匹配的字符串，也不能用作字面序列的上下文字节。

