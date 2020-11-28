## cache

在操作系统里，cache是最重要的一个东西。



### i/o cache

#### buffer

buffer是内存和磁盘之间的一个缓存，它加速了<b>写入</b>的过程。因为buffer的存在，不必在每次写入的时候都去寻址磁盘的位置



#### cache

cache跟buffer不太一样，它是位于内存和cpu之间的一个缓存，它加速了<b>读取</b>的过程。cache保证了每次读取的适合，也不必去进行磁盘寻址



> 注意，buffer cache都是i/o设备的块缓存

#### page cache

页缓存是在支持虚拟页面的操作系统上才有的，否则就是块缓存。页缓存是比块缓存更高级的抽象。

在现在的linux里，`page cache`和`buffer cache`可以互相映射，共同作用。

已经提到过，`page cache`是操作系统高级的抽象，而非硬件缓存。所以`page cache`是文件(虚拟)和内存之间的一层缓存。

对`page cache`而言，它的储存设备称为后备储存。

##### 读写

+ 读：一个`read`操作将直接从内存里发起请求(`page cache`)，如果命中缓存，则直接读取。否则进入后备储存读取，再缓存进`page cache`里。
+ 写：一个`write`操作将直接写入到内存`page cache`里，此时对应的后备储存标识为`dirty`，需要写入后备储存，`page cache`才能被释放。

`page cache`如何在文件读写时起作用？

##### struct page

##### address_space

```
struct address_space {
    struct inode            *host;              /* owning inode */
    struct radix_tree_root  page_tree;          /* radix tree of all pages */
    spinlock_t              tree_lock;          /* page_tree lock */
    unsigned int            i_mmap_writable;    /* VM_SHARED ma count */
    struct prio_tree_root   i_mmap;             /* list of all mappings */
    struct list_head        i_mmap_nonlinear;   /* VM_NONLINEAR ma list */
    spinlock_t              i_mmap_lock;        /* i_mmap lock */
    atomic_t                truncate_count;     /* truncate re count */
    unsigned long           nrpages;            /* total number of pages */
    pgoff_t                 writeback_index;    /* writeback start offset */
    struct address_space_operations *a_ops;     /* operations table */
    unsigned                long flags;         /* gfp_mask and error flags */
    struct backing_dev_info *backing_dev_info;  /* read-ahead information */
    spinlock_t              private_lock;       /* private lock */
    struct list_head        private_list;       /* private list */
    struct address_space    *assoc_mapping;     /* associated buffers */
};
```



##### 缓存擦除

当然，缓存必须在适当的时候被释放，否则会持续消耗内存的空间。

除了常见的`LRU`算法，`page cache`还提出了`Two List`算法：

​	维持两个队列`InActiveList`和`ActiviList`，两个队列均适用`LRU`算法进出。一个缓存块进入时，先进入到`InActiveList`，在再次使用时进入`ActiveList`。`ActiveList`里任何缓存块都不能被擦除，当两个队列差距过大时，移除`ActiveList`的部分头部数据进入到`InActiveList`，重新平衡队列后继续运行算法。



### 脏页

实际上，我们需要利用缓存来提高读写效率时，都会考虑到一个共有的问题：缓存是否真实。

我们把缓存与磁盘不相同的页面叫做脏页，把相同数据的页面叫做干净页。操作系统必须把脏页刷到磁盘上，以保证用户的修改真实有效。这个操作被叫做`flush`



我们在`page cache`一部分提到了缓存擦除。但主要介绍的是擦除算法，这里需要考虑命中的缓存页是否是脏页的情况。

+ 很显然，干净页可以直接复用
+ 脏页需要先进行`flush`，然后再被删除或复用



### VFS

我们经常说，文件读写操作需要内核态和用户态之间切换进行。

`VFS`建立了内核态到用户态的一层抽象，兼容了上层的文件系统(ext3,fat32,ntfs等)，提供给用户一套操作底层内核的API

