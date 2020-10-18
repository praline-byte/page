
# B+树


# BoltDB 简介

BoltDB 一个 go 实现的 key/value 数据库

目标是为不需要完整数据库服务器（如 Postgres 或MySQL）的项目提供一个简单，快速和可靠的数据库

支持事务(ACID)，使用 MVCC 和 COW，允许多个读事务和一个写事务并发执行，但是读事务会阻塞写事务

使用它的有 etcd, consul，以及我们 lark_ark 服务中使用的搜索引擎 bleve 默认使用的也是 boltdb

# 数据结构
了解任何一个工程，都要了解它的数据结构，要知道是什么对象在工程里流转，盘活整个上下文逻辑

## boltDB 数据库文件的基本格式
![](images/posts/boltDB_images/数据库文件内部布局.png)

数据库文件以页为基本单位，一个数据库文件由若干页组成。一个页的大小是由当前OS决定的，即通过os.GetpageSize()来决定，对于32位系统，它的值一般为4K字节。一个Boltdb数据库文件的前两页是meta页，第三页是记录freelist的页面，第四页及后续各页则是用于存储K/V的页面。

## page 的类型定义
```
const (
    branchPageFlag   = 0x01
    leafPageFlag     = 0x02
    metaPageFlag     = 0x04
    freelistPageFlag = 0x10
)

......

type pgid uint64

type page struct {
    id       pgid
    flags    uint16
    count    uint16
    overflow uint32
    ptr      uintptr
}
```
- id: 页面id，如0, 1, 2，...，是从数据库文件内存映射中读取一页的索引值
- flags: 页面类型，可以分为branchPageFlag、leafPageFlag、metaPageFlag 和 freelistPageFlag，表示内节点，叶子节点，meta 节点和 freelist 节点
- count: 页面内存储的元素个数，只在branchPage或leafPage中有用，对应的元素分别为branchPageElement和leafPageElement；
- overflow: 当前页是否有后续页，如果有，overflow表示后续页的数量，如果没有，则它的值为0，主要用于记录连续多页；
- ptr：用于标记页头结尾或者页内存储数据的起始处，一个页的页头就是由上述id、flags、count和overflow构成。需要注意的是，ptr本身不是页头的组成部分，它不是页的一部分，也不被存于磁盘上。


## page 的基本格式


# CRUD

初始化

增

删

改

查

# 事务

# 高性能

# 高可用

# 总结





mmap 妈妈牌技术
BoltDB 采用一个单独的文件作为持久化存储，利用mmap将文件映射到内存中，并将文件划分为大小相同的 Page 存储数据，使用写时复制技术将脏页写入文件
内存映射文件（Memory-Mapped File，mmap）技术是将一个文件映射到调用进程的虚拟内存中，通过操作相应的内存区域来访问被映射文件的内容。mmap()系统调用函数通常在需要对文件进行频繁读写时使用，用内存读写取代 I/O 读写，以获得较高的性能。
传统的 UNIX 或 Linux 系统在内核中设有多个缓冲区，当我们调用read()系统调用从文件中读取数据时，内核通常先将该数据复制到一个缓冲区中，再将数据复制到进程的内存空间。

而使用mmap时，内核会在调用进程的虚拟地址空间中创建一个内存映射区，应用进程可以直接访问这段内存获取数据，节省了从内核空间拷贝数据到用户进程空间的开销。mmap并不会真的将文件的内容实时拷贝到内存中，而是在读取数据过程中，触发缺页中断，才会将文件数据复制到内存中。

现代操作系统中常用分页技术进行内存管理，将虚拟内存空间划分成大小相同的 Page，其大小通常是 4KB


写入
BoltDB 中对文件的写入并没有使用mmap技术，而是直接通过Write()与fdatasync()这两个系统调用将数据写入文件。





Basics

There are only a few types in Bolt: DB, Bucket, Tx, and Cursor. The DB is
a collection of buckets and is represented by a single file on disk. A bucket is
a collection of unique keys that are associated with values.

Transactions provide either read-only or read-write access to the database.
Read-only transactions can retrieve key/value pairs and can use Cursors to
iterate over the dataset sequentially. Read-write transactions can create and
delete buckets and can insert and remove keys. Only one read-write transaction
is allowed at a time.


Caveats

The database uses a read-only, memory-mapped data file to ensure that
applications cannot corrupt the database, however, this means that keys and
values returned from Bolt cannot be changed. Writing to a read-only byte slice
will cause Go to panic.

Keys and values retrieved from the database are only valid for the life of
the transaction. When used outside the transaction, these byte slices can
point to different data or can point to invalid memory which will cause a panic.



// b+tree 原理
查找，遍历，插入，分裂，平衡

// boltdb 的实现
boltDb 代码量不多，核心代码不到4000行，但质量很高，建议大家有时间也阅读一下，对代码能力的提升很有帮助
这里我先和大家一起，把主线走一遍，方便之后阅读的时候，更容易一些


page bucket cursor tx db

db: boltdb 的顶级对象

node 是boltDb的头等公民，
page 主要提供的是xx
先不细看，我们带着问题再回来找他们


// API们
打开 111

Bolt 会在数据文件上获得一个文件锁，所以多个进程不能同时打开同一个数据库。 打开一个已经打开的 Bolt 数据库将导致它挂起，直到另一个进程关闭它
todo 文件锁在哪里



了解一个存储引擎的时候，需要了解在这几个方面是如何实现的

// 数据结构
// CRUD
Put
    b := tx.Bucket([]byte("MyBucket"))
    err := b.Put([]byte("answer"), []byte("42"))

写入数据的时候，会生成临时文件吗？ 在工程中是非常可控的. etcd和consul都用了它, 未必是因为性能有多高而是简单可靠
we have known there's no temporary file that means BoltDB use memory or called Mmap to host temporary data. use Mmap means, your BoltDB File should not bigger than your assignable memory space

mmap
mmap也是以页为单位进行映射的，如果文件大小不是页大小的整数倍，映射的最后一页肯定超过了文件结尾处，这个时候超过部分的内存会初始化为0，对其的写操作不会写入文件。但如果映射的内存范围超过了文件大小，且超出范围大于4k，那对于超过文件所在最后一页地址空间的访问将引发异常。比如我们这里文件实际大小是16K，但我们要映射32K到进程地址空间中，那对超过16K部分的内存访问将会引发异常。实际上，我们前面分析过，Boltdb通过mmap进行了只读映射，故不会存在通过内存映射写文件的问题，同时，对db.data(即映射的内存区域)的访问是通过pgid来访问的，当前database文件里实际包含多少个page是记录在meta中的，每次通过db.data来读取一页时，boltdb均会作超限判断的，所以不会存在对超过当前文件实际页数以外的区域访问的情况

因为boltdb写文件不是通过mmap，而是直接通过fwrite写文件。强调一下，boltdb对数据库的读操作是通过读mmap内存映射区完成的；而写操作是通过文件fseek及fwrite系统调用完成的



Get
    b := tx.Bucket([]byte("MyBucket"))
    v := b.Get([]byte("answer"))

迭代
    c := tx.Bucket([]byte("MyBucket"))
    for k, v := c.First(); k != nil; k, v = c.Next() {
        fmt.Printf("key=%s, value=%s\n", k, v)
    }

前缀扫描
    c := tx.Bucket([]byte("MyBucket")).Cursor()

    prefix := []byte("1234")
    for k, v := c.Seek(prefix); k != nil && bytes.HasPrefix(k, prefix); k, v = c.Next() {
        fmt.Printf("key=%s, value=%s\n", k, v)
    }

范围扫描
	c := tx.Bucket([]byte("Events")).Cursor()
    min := []byte("1990-01-01T00:00:00Z")
    max := []byte("2000-01-01T00:00:00Z")
    for k, v := c.Seek(min); k != nil && bytes.Compare(k, max) <= 0; k, v = c.Next() {
        fmt.Printf("%s: %s\n", k, v)
    }

// 持久化

// 事务
Bolt 一次只允许一个写事务，但是一次允许多个只读事务

启动写事务
err := db.Update(func(tx *bolt.Tx) error {
    ...
    return nil
})

启动读事务
err := db.View(func(tx *bolt.Tx) error {
    ...
    return nil
})


// 高可用
 备份
Blot 是一个单一的文件，所以很容易备份
使用Tx.WriteTo()函数将数据库的一致视图写入目的地。 如果您从只读事务中调用它，它将执行热备份而不会阻止其他数据库的读写操作
Tx.CopyFile()


// 高性能
// 统计
当我们运行一个xx时，需要观测的指标  split, rebalanced

// 使用场景
就像每个算法模型都有假设空间一样，数据引擎也有一定的使用场景限制


所以 boltdb 可以更容易的，与其他组件结合提供更强大的功能
比如，lark_ark 服务中使用的 bleve 搜索引擎，就提供 boltdb 作为 kvstore 层的支持

boltdb 大概就是这样，我们也可以对其二次开发，进行改造，支持我们提到的特性

在评估和使用 Bolt 时，需要注意以下几点
Bolt 适合读取密集型工作负载。顺序写入性能也很快，但随机写入可能会很慢。您可以使用DB.Batch()或添加预写日志来帮助缓解此问题。
尽量避免长时间运行读取事务。 Bolt使用copy-on-write技术，旧的事务正在使用，旧的页面不能被回收。
Bolt在数据库文件上使用独占写入锁，因此不能被多个进程共享
将大量批量随机写入加载到新存储区可能会很慢，因为页面在事务提交之前不会分裂。不建议在单个事务中将超过 100,000 个键/值对随机插入单个新 bucket中。
Bolt使用内存映射文件，以便底层操作系统处理数据的缓存。 通常情况下，操作系统将缓存尽可能多的文件，并在需要时释放内存到其他进程。 这意味着Bolt在处理大型数据库时可以显示非常高的内存使用率。 但是，这是预期的，操作系统将根据需要释放内存。 Bolt可以处理比可用物理RAM大得多的数据库，只要它的内存映射适合进程虚拟地址空间

Bolt数据库中的数据结构是存储器映射的，所以数据文件将是endian特定的。 这意味着你不能将Bolt文件从一个小端机器复制到一个大端机器并使其工作。 对于大多数用户来说，这不是一个问题，因为大多数现代的CPU都是小端的
// todo 存储器映射？
由于页面在磁盘上的布局方式，Bolt无法截断数据文件并将空闲页面返回到磁盘。 相反，Bolt 在其数据文件中保留一个未使用页面的空闲列表。 这些免费页面可以被以后的交易重复使用。 由于数据库通常会增长，所以对于许多用例来说，这是很好的方法 但是，需要注意的是，删除大块数据不会让您回收磁盘上的空间




对比
如果您需要高随机写入吞吐量（> 10,000 w / sec）或者您需要使用旋转磁盘，则 LevelDB可能是一个不错的选择。 如果你的应用程序是重读的，或者做了很多范围扫描，Bolt 可能是一个不错的选择
内存 > 数据库 && 读稳定 > 写稳定 && 读性能 > 写性能
因此在之后的搜人场景，可以使用

缺点
没有WAL/redo log, 变动直接打入B+Tree

一方面看这确实是优点, 另一方面也导致了很多更好的优化做不了. 因为没有log保护中间状态, B+Tree自身必须具有原子性. BoltDB通过COW(copy on write)来达成, 带来了可观的开销. 即使只对某个page改了1 byte也必须重写整页, 连带parent到root都需要重写, 直至meta page, 有写放大的问题. meta page更新成功是操作成功的依据.

COW使得BoltDB几乎没有成本地支持一写多读的事务, 但也做不了并发写事务了, 因为双meta page只能确保至少一份是有效的. BoltDB提供了batch write可以自我安慰一下



总结：
原理
K/V 型存储，使用 B+ 树索引。
支持事务(ACID)，使用 MVCC 和 COW，允许多个读事务和一个写事务并发执行，但是读事务有可能会阻塞写事务，适合读多写少的场景

BoltDB 的写事务实现比较巧妙，利用 meta 副本和 freelist 机制实现并发控制，提供了一种解决问题的思路。操作系统通过 COW (Copy-On-Write) 技术进行 Page 管理，通过写时复制技术，系统可实现无锁的读写并发，但是无法实现无锁的写写并发，这就注定了这类数据库读性能很高，但是随机写的性能较差，因此非常适合于『读多写少』的场景

在工程中是非常可控的. etcd 和 consul 都用了它, 未必是因为性能有多高而是简单可靠


实现
交流了看源码的一些技巧
希望可以互相交流讨论

最后，系统设计没有银弹，适合的就是最好的