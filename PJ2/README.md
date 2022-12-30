CSE.30341.FA17: Project 06
==========================

This is the documentation for [Project 06] of [CSE.30341.FA17].

Members
-------

1. 程茜（19302010084@fudan.edu.cn）

Design
------

> 1. To implement `Filesystem::debug`, you will need to load the file system
>    data structures and report the **superblock** and **inodes**.
>
>       - How will you read the superblock?
>       - How will you traverse all the inodes?
>       - How will you determine all the information related to an inode?
>       - How will you determine all the blocks related to an inode?

- 磁盘的第一块为超级块，故可以通过 “**disk->read(0, block.Data)**”来读取超级块。
- 首先通过超级块的**inodeBlocks**字段知道磁盘为inode留出的块的总数，之后去遍历每一块里面的每个inode。
- inode有一个有效位valid，valid为1则为有效，为0则为无效，其后是inode的size，后面跟着是5个指向直接数据块的指针他们都在inode结构体的数组变量Direct[]中，最后还有一个指向间接块的指针Indirect。
- inode有5个直接指向数据块的指针，他们都在Direct[]数组中，故先遍历Direct数组就能得到5个数据块，之后通过Indiretc得到inode指向的间接块，每一个间接块可以存放1024个指向数据块的指针，故再遍历一次间接块的1024个指针即可。

> 2. To implement `FileSystem::format`, you will need to write the superblock
>    and clear the remaining blocks in the file system.
>      - What pre-condition must be true before this operation can succeed?
>       - What information must be written into the superblock?
>       - How would you clear all the remaining blocks?

- 传入的disk不能已经mounted。
- 超级块的MagicNumber、blocks、InodeBlocks、Inodes均需要写入超级块，blocks等于dish的大小，Inodeblocks的大小，InodeBlocks的大小等于disk的size的10%向上取整，Inodes的大小等于InodeBlocks的大小乘以128（每块含有的inode总数）。
- 将剩下的块都写入0。

> 3. To implement `FileSystem::mount`, you will need to prepare a filesystem
>    for use by reading the superblock and allocating the free block bitmap.
>
>       - What pre-condition must be true before this operation can succeed?
>       - What sanity checks must you perform?
>       - How will you record that you mounted a disk?
>       - How will you determine which blocks are free?

- 传入的disk不能已经mounted；

- 超级块的Inodes == InodeBlocks*128；

  超级块MagicNumber == 文件系统的MAGIC_NUMBER；

  blocks的数量必须大于等于0；

  超级快中的InodeBlocks == blocks*10%向上取整

- 调用Disk类的mount()方法

- 自定义了一个freemap位图数组，使用1来表示该块空闲，0表示非空闲。

> 4. To implement `FileSystem::create`, you will need to locate a free inode
>    and save a new inode into the inode table.
>      - How will you locate a free inode?
>       - What information would you see in a new inode?
>       - How will you record this new inode?

- 同debug()方法中用到的遍历inode的方法一样去遍历inode，如果找到一个inode的valid为0，则说明该inode是free的。
- 在新的inode中valid应该是1，size应该为0，直接指针和间接指针都应该为空，可以将它们设置为0。
- 在创建期间通过将 Valid 设置为 1 并设置来记录新的 inode，Inode 结构的其他字段到其适当的值。

> 5. To implement `FileSystem::remove`, you will need to locate the inode and
>    then free its associated blocks.
>      - How will you determine if the specified inode is valid?
>       - How will you free the direct blocks?
>       - How will you free the indirect blocks?
>       - How will you update the inode table?

- 先通过inumber找到inode，如果inode的valid为1则为有效，反之则为无效。
- 直接将inode的指向direct的指针设为0，在自定义的空闲位图map中将该块对应的位置设置为1，表示该块空闲。
- 先通过找到间接块，然后将间接块指向的每个指针都设置为0，然后在bitmap中将每个块以及indirect对应的位置设置为1（free）。
- 将该inode重新写入。

> 6. To implement `FileSystem::stat`, you will need to locate the inode and
>    return its size.
>
>       - How will you determine if the specified inode is valid?
>       - How will you determine the inode's size?

- 先通过inumber找到inode，如果inode的valid为1则为有效，反之则为无效。

- 每一个inode的大小为32字节

> 7. To implement `FileSystem::read`, you will need to locate the inode and
>    copy data from appropriate blocks to the user-specified data buffer.
>
>       - How will you determine if the specified inode is valid?
>       - How will you determine which block to read from?
>       - How will you handle the offset?
>       - How will you copy from a block to the data buffer?

- 通过inumber查找对应的inode，看inode的valid是否为1。
- 首先调整读取的长度为length 和 inode.Size的较小值，然后用offset除以block的大小，得到恶整数即为要读取的块号。
- 知道了首先要读取的块号后，使用offset对block的大小取余得到read_offset。
- 使用memcpy，进行copy。

> 8. To implement `FileSystem::write`, you will need to locate the inode and
>    copy data the user-specified data buffer to data blocks in the file
>    system.
>
>       - How will you determine if the specified inode is valid?
>       - How will you determine which block to write to?
>       - How will you handle the offset?
>       - How will you know if you need a new block?
>       - How will you manage allocating a new block if you need another one?
>       - How will you copy from a block to the data buffer?
>       - How will you update the inode?

- 通过inumber查找对应的inode，看inode的valid是否为1。
- 首先调整读取的长度为length 和 inode.Size的较小值，然后用offset除以block的大小，得到恶整数即为要开始写入的块号。
- 知道了首先要写入的块号后，使用offset对block的大小取余得到write_offset。
- 如果需要指向需要写入的块指针为0，则说明需要新的块。
- 去遍历bitmap，找到一个空闲的块，然后将该块全部写入0，并将bitmap对应为设置为0；
- 使用memcpy进行copy。
- inode的Size改为新的写去后的size，如果间接块有改变还要将改变写入disk，将该inode重新写入disk。

Errata
------

> Describe any known errors, bugs, or deviations from the requirements.

Extra Credit
------------

> Describe what extra credit (if any) that you implemented.

[Project 06]:       https://www3.nd.edu/~pbui/teaching/cse.30341.fa17/project06.html
[CSE.30341.FA17]:   https://www3.nd.edu/~pbui/teaching/cse.30341.fa17/
[Google Drive]:     https://drive.google.com
