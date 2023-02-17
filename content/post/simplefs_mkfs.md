---
title: "SimpleFS文件系统2: 格式化"
date: 2020-05-22T00:25:43+08:00
tags: ["simplefs", "fs"]
draft: false
---



## 格式化程序mkfs.c

格式化的功能在源码中的mkfs.c中实现，其main函数中的主要环节已经写好了注释:<!--more-->

```c
int main(int argc, char **argv)
{
    ...
    /* Open disk image */
    int fd = open(argv[1], O_RDWR);
    ...
    /* Get image size */
    struct stat stat_buf;
    int ret = fstat(fd, &stat_buf);
    ...
    /* Get block device size */
    if ((stat_buf.st_mode & S_IFMT) == S_IFBLK) {
        long int blk_size = 0;
        ret = ioctl(fd, BLKGETSIZE64, &blk_size); // 使用ioctl获取块设备大小
        ...
        stat_buf.st_size = blk_size;
    }

    /* Check if image is large enough */
    long int min_size = 100 * SIMPLEFS_BLOCK_SIZE; // min_size = 400KB
    if (stat_buf.st_size <= min_size) {
        ...
        goto fclose;
    }

    /* Write superblock (block 0) */
    struct superblock *sb = write_superblock(fd, &stat_buf);
    ...
    /* Write inode store blocks (from block 1) */
    ret = write_inode_store(fd, sb);
    ...
    /* Write inode free bitmap blocks */
    ret = write_ifree_blocks(fd, sb);
    ...
    /* Write block free bitmap blocks */
    ret = write_bfree_blocks(fd, sb);
    ...
    /* Write data blocks */
    ret = write_data_blocks(fd, sb);
    ...
free_sb:
    free(sb);
fclose:
    close(fd);

    return ret;
}
```

其中，前半部分的打开块设备文件、检查文件类型、检测是否满足文件系统最小大小等步骤很好理解，这里不在赘述，后续主要分析是如何对块设备的布局进行规划，如何填充各个初始化状态下的数据结构并写入到块设备中的。



## 超级块数据写入

超级块的写入，体现了SimpleFS对整个设备分区的布局和规划，主要的步骤已经注释在核心代码中:

```C
static struct superblock *write_superblock(int fd, struct stat *fstats)
{
    struct superblock *sb = malloc(sizeof(struct superblock));
    ...
    uint32_t nr_blocks = fstats->st_size / SIMPLEFS_BLOCK_SIZE; // 块数目 = 块设备大小 / 块大小
    uint32_t nr_inodes = nr_blocks; // 定义inode数目等于块数目，这里取了理论最大值，即假设一个文件占用一个数据块
    uint32_t mod = nr_inodes % SIMPLEFS_INODES_PER_BLOCK; // 与一个块能存储的inode信息数目取模
    if (mod) // 如果未对齐(最后一个存储inode信息的块未利用完)
        nr_inodes += SIMPLEFS_INODES_PER_BLOCK - mod; // 因为存储inode信息的最后一个块有空闲空间，故能够多存储(SIMPLEFS_INODES_PER_BLOCK - mod)个inode。
    uint32_t nr_istore_blocks = idiv_ceil(nr_inodes, SIMPLEFS_INODES_PER_BLOCK); // 保存完所有inode需要的块数量
    uint32_t nr_ifree_blocks = idiv_ceil(nr_inodes, SIMPLEFS_BLOCK_SIZE * 8); // 标记完所有inode的空闲位图需要的块数量，因为位图的每个字节有8位，能标记8个inode，故一个块能标记总字节数*8个inode
    uint32_t nr_bfree_blocks = idiv_ceil(nr_blocks, SIMPLEFS_BLOCK_SIZE * 8); // 同上
    uint32_t nr_data_blocks =
        nr_blocks - 1 - nr_istore_blocks - nr_ifree_blocks - nr_bfree_blocks; // 数据块的数目 = 总的块数目 - 超级块 - 多个inode信息块 - inode空闲位图 - 数据块空闲位图

    memset(sb, 0, sizeof(struct superblock));
    sb->info = (struct simplefs_sb_info){ // 填充超级块结构体, 该结构体在以后的mount时会被作为最关键的元数据被读取出来
        .magic = htole32(SIMPLEFS_MAGIC), // 填充魔数，注意填充时使用的是32位的小端编码。
        .nr_blocks = htole32(nr_blocks),
        .nr_inodes = htole32(nr_inodes),
        .nr_istore_blocks = htole32(nr_istore_blocks),
        .nr_ifree_blocks = htole32(nr_ifree_blocks),
        .nr_bfree_blocks = htole32(nr_bfree_blocks),
        .nr_free_inodes = htole32(nr_inodes - 1), // -1是因为把编号为0的inode保留了，即文件系统根所占用的inode
        .nr_free_blocks = htole32(nr_data_blocks - 1), // 原因同上，文件系统根必然是一个目录文件，故会占用一个数据块
    };

    int ret = write(fd, sb, sizeof(struct superblock)); // 把超级块的结构体数据写入到设备文件中
    ...
    return sb;
}
```

小结：

* 主要是根据设备容量，计算数据块的数量及对应所需的inode信息的条目数
* 再进一步计算得到inode信息、inode空闲位图和数据块空闲位图的占用块数，并确定文件系统的整体布局
* 填充SimpleFS超级块结构体，并写入到设备文件中的第1个块中



## inodes信息块数据写入

```C
static int write_inode_store(int fd, struct superblock *sb)
{
    /* Allocate a zeroed block for inode store */
    char *block = malloc(SIMPLEFS_BLOCK_SIZE);
    ...
    memset(block, 0, SIMPLEFS_BLOCK_SIZE);

    /* Root inode (inode 0) */
    struct simplefs_inode *inode = (struct simplefs_inode *) block;
    uint32_t first_data_block = 1 + le32toh(sb->info.nr_bfree_blocks) + // 超级块占用1个块，其他区域从sb->info中读取
                                le32toh(sb->info.nr_ifree_blocks) +
                                le32toh(sb->info.nr_istore_blocks);
    inode->i_mode = htole32(S_IFDIR | S_IRUSR | S_IRGRP | S_IROTH | S_IWUSR |
                            S_IWGRP | S_IXUSR | S_IXGRP | S_IXOTH);
    inode->i_uid = 0;
    inode->i_gid = 0;
    inode->i_size = htole32(SIMPLEFS_BLOCK_SIZE);
    inode->i_ctime = inode->i_atime = inode->i_mtime = htole32(0);
    inode->i_blocks = htole32(1); // 根inode占用1个数据块
    inode->i_nlink = htole32(2); // 默认会有 . 与 .. 两个软链接
    inode->dir_block = htole32(first_data_block); // 设置根inode的目录块的索引，即第几个数据块

    int ret = write(fd, block, SIMPLEFS_BLOCK_SIZE); // 在第1个inode信息块中写入根inode的数据
    ...
    /* Reset inode store blocks to zero */
    memset(block, 0, SIMPLEFS_BLOCK_SIZE);
    uint32_t i;
    for (i = 1; i < sb->info.nr_istore_blocks; i++) { // 把剩余inode信息块置零
        ret = write(fd, block, SIMPLEFS_BLOCK_SIZE);
        ...
    }
    ...
end:
    free(block);
    return ret;
}
```

小结

* 主要是向设备文件中写入第0号inode，即该文件系统根目录的元数据信息。



## inode/数据块空闲位图数据写入

inode空闲位图的写入

```c
static int write_ifree_blocks(int fd, struct superblock *sb)
{
    char *block = malloc(SIMPLEFS_BLOCK_SIZE);
    ...
    uint64_t *ifree = (uint64_t *) block;

    /* Set all bits to 1 */
    memset(ifree, 0xff, SIMPLEFS_BLOCK_SIZE); // 1表示空闲，0表示已使用

    /* First ifree block, containing first used inode */
    ifree[0] = htole64(0xfffffffffffffffe); // 最低位为0，表示第0号inode已被使用
    int ret = write(fd, ifree, SIMPLEFS_BLOCK_SIZE);
    ...
    /* All ifree blocks except the one containing 2 first inodes */
    ifree[0] = 0xffffffffffffffff;
    uint32_t i;
    for (i = 1; i < le32toh(sb->info.nr_ifree_blocks); i++) { //填充剩余的inode空闲位图块，全部置为1表示空闲
        ret = write(fd, ifree, SIMPLEFS_BLOCK_SIZE);
        ...
    }
    ret = 0;
    ...
end:
    free(block);

    return ret;
}
```

数据块空闲位图的写入

```C
static int write_bfree_blocks(int fd, struct superblock *sb)
{
    uint32_t nr_used = le32toh(sb->info.nr_istore_blocks) +
                       le32toh(sb->info.nr_ifree_blocks) +
                       le32toh(sb->info.nr_bfree_blocks) + 2; // 2中包含1个超级块和1个根目录

    char *block = malloc(SIMPLEFS_BLOCK_SIZE);
    ...
    uint64_t *bfree = (uint64_t *) block;

    /*
     * First blocks (incl. sb + istore + ifree + bfree + 1 used block)
     * we suppose it won't go further than the first block
     */
    memset(bfree, 0xff, SIMPLEFS_BLOCK_SIZE);
    uint32_t i = 0;
    while (nr_used) {
        uint64_t line = 0xffffffffffffffff;
        for (uint64_t mask = 0x1; mask; mask <<= 1) { // 个位为1的64位掩码，并不断左移
            line &= ~mask; // 置当前掩码所在位置为0，即表示该数据块已使用
            nr_used--;
            if (!nr_used)
                break;
            if (line == (uint64_t)0x00) // break if this line is used up?
                break;
        }
        bfree[i] = htole64(line); // 以64位长度为单位，不断更新块内的位图信息，直到所有已使用数据块被都被标记
        i++;
    }
    int ret = write(fd, bfree, SIMPLEFS_BLOCK_SIZE); // 写入第1个数据块空闲位图到设备上
    ...
    /* other blocks */
    memset(bfree, 0xff, SIMPLEFS_BLOCK_SIZE);
    for (i = 1; i < le32toh(sb->info.nr_bfree_blocks); i++) { // 写入1到剩余的数据块空闲位图，表示都未使用
        ret = write(fd, bfree, SIMPLEFS_BLOCK_SIZE);
        ...
    }
    ret = 0;
    ...
    free(block);
    return ret;
}
```

小结:

* 根据sb->info中的inode及块的使用统计，分别写入位图到inode及数据块的空闲位图块中。



## 数据块数据写入

```c
static int write_data_blocks(int fd, struct superblock *sb)
{
    /* FIXME: unimplemented */
    return 0;
}
```

当前的实现是没有做任何事情直接返回的。



## 小结

* SimpleFS的格式化，主要是根据设备容量计算出文件系统的布局
* 根据布局，把超级块结构体写入超级块所在块中(第1号块)
* 根据超级块的定义，把inode信息块、inode空闲位图、数据块空闲位图写入相应的数据
* 格式化中写入的结构体与位图数据，可以在后面梳理挂载功能的时候，进行进一步理解

