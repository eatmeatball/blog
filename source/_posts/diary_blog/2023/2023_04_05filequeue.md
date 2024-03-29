---
title: filequeue
toc: true
date: 2023-04-05 19:29:23
tags:
  - other
  - blog
categories:
  - other

---


[thh/filequeue.go at master · eatmeatball/thh (github.com)](https://github.com/eatmeatball/thh/blob/master/arms/filequeue/filequeue.go)

上面是我的源码链接地址。


> 写这个文件队列的目的是为了写单体应用程序的时候可以不用依赖redis，这种额外程序。这样在开发一些单体应用的时候，可以把依赖降低到最少，也避免了内存队列的中断丢失问题。


<!--more-->



文件队列是指一种基于文件的数据结构，通常用于在磁盘上存储和管理大量数据。相对于内存队列，文件队列可以避免因为数据量过大而导致内存不足的情况。同时，文件队列也具有持久化、可靠性高等优点。

下面我们就来分析这个文件队列的实现方式。

一、整体架构

该文件队列的实现方式非常简单，只包含一个文件，其主要内容为文件头（64字节）和数据块列表。其中，文件头记录了版本号，块大小和当前队列下标信息，数据块列表则是一系列格式统一的数据块集合，每个数据块里保存了一条数据。

二、代码实现

该文件队列在代码实现中涉及到两个主要结构体：QueueHeader 和 FileQueue：

1.  QueueHeader 用于描述文件头，包括版本号，块大小等信息。其中，版本号表示文件头的版本，块大小表示数据块的大小，偏移量表示当前位于队列的哪一个数据块下，dataMaxLen 表示当前块可以存储的最大数据长度。

2.  FileQueue 是文件队列的主要实现结构体，包含了队列路径，读写锁等信息。同时，FileQueue 中的方法实现了队列的基本操作，包括入队、出队、压缩文件和初始化等操作。

三、代码核心

1.  实现入队操作 FileQueue 的 Push 方法实现了入队操作，将字符串参数转化为字节数组后，计算它的长度，并根据指定的块长生成一个数据块，将有效位设置为 1，数据长度设置为实际数据的长度，同时将数据拼接在数据块中，并写入到队列文件的末尾。

2.  实现出队操作 FileQueue 的 Pop 方法实现了出队操作，它首先计算当前数据块的起始位置，然后读取可用位、长度位和数据，接着将偏移量增加 1（表示队列已经出队了一个元素），并将新的偏移量写入到文件头中。

3.  实现数据压缩 FileQueue 的 Clean 方法实现了数据的压缩。该方法首先创建一个临时文件，然后依次将未出队的数据块从旧文件中读取，并重新写入到临时文件中。最后，将旧文件删除并重命名临时文件为新文件。

四、代码执行流程

以下是该文件队列的基本执行流程：

1.  初始化时，创建队列目录，并打开或创建队列文件。
2.  当入队时，将数据转化为字节数组，并根据指定块长生成一个数据块，写入到队列文件的末尾。
3.  当出队时，计算当前数据块的起始位置，然后读取可用位、长度位和数据。
4.  每次从队列中出队时，将偏移量增加 1，并将新的偏移量写入文件头。
5.  当需要压缩队列时，创建一个临时文件。依次将未出队的数据块从旧文件中读取，并重新写入到临时文件中。最后，将旧文件删除并重命名临时文件为新文件。

五、总结

这个 Go 实现的文件队列非常简单，实现起来主要是通过文件的读写来完成。它的使用场景主要是在本地应用程序