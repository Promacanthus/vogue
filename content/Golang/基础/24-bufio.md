---
title: 24-bufio
date: 2019-11-25T11:15:47.526182+08:00
draft: false
---

bufio是buffered I/O的缩写，这个代码包中的程序实体实现的I/O操作都内置了缓冲区。bufio包中的数据类型主要有：

- Reader
- Scanner
- Writer
- ReadWriter

与io包中的数据类型类似，这些类型的值也都需要在初始化的时候，包装一个或多个简单I/O接口类型的值。（简单接口类型值的就是io包中的那些简单接口。）

## `bufio.Reader`类型值中的缓冲区的作用

`bufio.Reader`类型值内的缓冲区，是一个数据存储中间，它介于底层读取器（初始化此类值的时候传入`io.Reader`类型的参数值）与读取方法及其调用方之间。

`bufio.Reader`值的读取方法一般都会先从其所属值的缓冲区中读取数据，必要的时候，它们还会预先从底层读取器那里读取一部分数据，并暂存于缓冲区中以备后用。

> 有这样一个缓冲区的好处是，可以在大多数时候降低读取方法的执行时间，虽然读取方法有时还要负责填充缓冲区，但从总体来看，读取方法平均执行时间一般会因此有大幅的缩短。

`bufio.Reader`类型并不是开箱即用的，它包含一些需要显式初始化的字段，如下：

```go
// Reader为io.Reader对象实现缓冲。
type Reader struct {
    // 字节切片，代表缓冲区，
    // 虽然这是切片类型，但是它的长度却是在初始化的时候指定，并且在之后保持不变
    buf          []byte

    rd           io.Reader //客户端提供的reader，代表底层读取器，缓冲区中的数据就是从这里拷贝来的

    r, w         int       // buf 读写位置
    // r 代表对缓冲区进行下一次读取时的开始索引，称为已读计数
    // w 代表对缓冲区进行下一次写入时的开始索引，称为已写计数

    // 它的值用于表示在从底层读取器获得数据时发生的错误
    // 这里值在被读取或忽略之后，该字段会被设置为nil
    err          error  

    // UnreadByte读取的最后一个字节； -1表示无效
    // 用于记录缓冲区中最后一个被读取的字节，读回退时会用到它的值 
    lastByte     int

    // UnreadRune读取的最后一个rune的大小； -1表示无效
    // 用于记录缓冲区中最后一个被读取的Unicode字符所占用的字节数，
    // 读回退的时候会用到它的值，这个字段只会在其所属值的ReadRune方法中才会被赋予有意义的值
    // 其他情况下，它都被置为-1
    lastRuneSize int 
}
```

bufio包提供了两个用于初始化Reader值的函数：

- NewReader：初始化的Reader值会拥有一个默认大小（4096字节，即4KB）的缓冲区，
- NewReaderSize：将缓冲区的大小的决定权交给使用方

它们都会返回一个`*bufio.Reader`类型的值。这里的缓冲区在一个Reader值的生命周期内大小是不变的，所以在有些时候需要做一些权衡。

- 读取Peek和ReadSlice方法，都会调用该类型的一个名为fill的包级私有方法，fill方法的作用是填充内部缓冲区。
- fill方法，首先检查其所属值的已读计数，如果这个计数不大于0，那么有两种可能：
    1. 缓冲区中的字节都是全新的，它们没有被读取过
    2. 缓冲区刚被压缩过，对缓冲区的压缩操作：
       1. 把缓冲区中在[已读计数，已写计数]范围之内的所有字节都一次拷贝到缓冲区的头部，这一步不会有副作用，因为：
          1. 已读计数之前的字节都已经被读取过，肯定不会再被读取，因此把它们覆盖掉是安全的
          2. 在压缩缓冲区之后，已写计数之后的字节只可能是已经被读取过的字节，或者是已被拷贝到缓冲区头部的未读字节，或者是代表未曾被填入数据的零值（0x00），所以后续的新字节可以被卸载这些位置上。
       2. fill方法会把已写计数的新值设定为原已写计数与已读计数只差，这个差锁代表的索引，就是压缩后第一次写入字节时的开始索引。

缓冲区的压缩过程，如下图所示：

![images](/images/compression.png)

实际上，fill方法只要在开始时发现其所属值的已读计数大于0，就会对缓冲区进行一次压缩，之后，如果缓冲区中还有可写的位置，那么该方法就会对其进行填充。

在填充缓冲区的时候，fill方法会试图从底层读取器哪里，读取足够多的字节，并尽量把从已写计数代表的索引位置到缓冲区末尾之间的空间都填满。

在这个过程中fill方法会及时更新已写计数，以保证填充的正确性和顺序性，它还会判断从底层读取器读取数据的时候，是否有错误发生，如果有，那么它就会把错误值赋予给其所属值的err字段，并终止填充流程。

## `bufio.Writer`类型值中缓冲的数据何时写入底层写入器

```go
// Writer为io.Writer对象实现缓冲。
// 如果在写入Writer时发生错误，将不再接受任何数据，
// 并且所有后续写入和Flush都将返回错误。
// 写入所有数据之后，客户端应调用Flush方法以确保所有数据都已转发到底层io.Writer。
type Writer struct {
    err error   // 它的值用于表示在向底层写入器写数据时发生的错误
    buf []byte  // 代表缓冲区，在初始化之后，它的长度会保持不变
    n   int // 代表对缓冲区进行下一次写入时的开始索引，称为写入计数
    wr  io.Writer   // 代表底层写入器
}
```

`bufio.Writer`类型有一个名为Flush的方法，它的主要功能是把相应缓冲区中暂存的所以数据，都写到底层写入器中，数据一旦被写入底层写入器，该方法就会把它们从缓冲区中删除掉。

**这里的删除有时候只是逻辑删除**。不论是否成功写入了所有暂存数据，Flush方法都会妥当处置，并保证不会出现重写或者漏写的情况。

`bufio.Writer`类型拥有的所以数据写入方法都会在必要的时候调用它的Flush方法。

- Write方法有时候会在把数据写进缓冲区之后，调用Flush方法，以便为后续的新数据腾出空间，如果Write方法发现要写入的字节太多，同时缓冲区已空，那么会直接跨过缓冲区，直接把新的数据写到底层写入器中。
- WriteByte方法和WriteRune方法都会在发现缓冲区中的可写空间不足以容纳新的字节或Unicode字符的时候，调用Flush方法
- ReadFrom方法，会在发现底层写入器的类型是`io.ReaderFrom`接口的实现之后，直接调用其`ReadFrom`方法把参数值持有的数据写进去。

**只要缓冲区中的可写空间无法容纳需要写入的新数据，Flush方法就一定会被调用**，`bufio.Writer`类型的一些方法有时候还会试图走捷径，跨过缓冲区而直接对接数据供需方。

## `bufio.Reader`类型的读取方法

`bufio.Reader`类型拥有很多用于读取数据的指针方法，这里有四个方法可以作为不同读取流程的代表：

```go
func (b *Reader) Peek(n int) ([]byte, error)
```

- Peek：读取并返回其缓冲区中n个未读字节，并且它会从已读计数代表的索引位置开始读。

  1. 在缓冲区未被填满，并且其中的未读字节的数量小于n的时候，该方法会调用fill方法，以启动缓冲区填充流程，如果发现上次填充缓冲区时有错误，则不再填充。
  2. 如果调用方给定的n比缓冲区的长度还大，或者缓冲区中未读字节的数量小于n，那么：
     1. 所有未读字节组成的序列作为第一个结果
     2. `bufio.ErrBufferFull`变量的值作为第二个结果，用来表示虽然缓冲区被压缩和填满了，但是仍然不满足要求
  3. 上述情况都未出现，则返回**已读计数为起始的n个字节**和**表示未发生任何错误的`nil`**

**Peek方法的一个特点，即使它读取了缓冲区中的数据，也不会改变已读计数的值**。其他的读取方法不是这样的。

```go
func (b *Reader) Read(p []byte) (n int, err error)
```

- Read：把缓冲区中的未读字节，依次拷贝到其参数p代表的字节切片中，并立即根据实际拷贝的字节数增加已读计数的值。

    - 在缓冲区中还有未读字节的情况下，Read方法是这样做的。（当已读计数等于已写计数时，表示此时的缓冲区中没有任何未读的字节）
    - 当缓冲区中无未读字节时，Read方法会先检查参数p的长度是否大于或等于缓冲区的长度。
      - 如果是，Read方法放弃缓冲区中的填充数据，直接从底层读取器中读出数据并拷贝到p中，这意味着它完全跨过了缓冲区，并直连了数据供需的双方。
      - 如果否，会先把已读计数和已写计数都重置为0，然后再尝试（只进行一次）使用从底层读取器那里回去的数据，对缓冲区进行一次从头到尾的填充。

```go
func (b *Reader) ReadSlice(delim byte) (line []byte, err error)
```

- ReadSlice：持续地读取数据，直到遇到调用方给定的分隔符为止。
    
    先在缓冲区的未读部分中寻找分隔符，如果未找到，并且缓冲区未满，那么调动fill方法对缓冲区进行填充，然后再次寻找，如此往复。
    - 如果在填充的过程中遇到错误，会把未读部分作为结果返回，并返回相应的错误值。
    - 如果缓冲区被填满，仍然没有找到分隔符，那么整个缓冲区作为第一个结果，`bufio.ErrBufferFull`（缓冲区已满的错误）作为第二个结果

```go
func (b *Reader) ReadBytes(delim byte) ([]byte, error)
```

- ReadBytes：持续地读取数据，直到遇到调用方给定的分隔符为止。

    - ReadBytes方法依赖ReadSlice方法。
    - ReadLine方法依赖ReadSlice方法。
    - ReadString方法完全依赖ReadBytes方法，只是在返回的结果之上做简单的类型转换。

**Peek、ReadSlice、ReadLine方法都可能会造成内容泄露**，在正常情况下，它们都会直接返回基于缓冲区的字节切片，调用方可以通过这些方法的结果值访问到缓冲区的其他部分，甚至修改缓冲区中的内容，这是非常危险的。
