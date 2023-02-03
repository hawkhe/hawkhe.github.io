---
title: "SimpleFS文件系统3: 挂载文件系统"
date: 2020-07-15T00:25:43+08:00
tags: ["simplefs", "fs"]
draft: false
---



## 内核模块加载

SimpleFS的核心功能都在内核态实现，使用前需要保证该模块已经加载到内核之中。

其入口代码实现在fs.c文件中，加载SimpleFS内核模块时，会调用到module_init函数注册的simplefs_init函数上。

```c
module_init(simplefs_init); // 模块初始化
module_exit(simplefs_exit); // 模块退出

MODULE_LICENSE("Dual BSD/GPL");
MODULE_AUTHOR("National Cheng Kung University, Taiwan");
MODULE_DESCRIPTION("a simple file system");
```



## simplefs_init函数

simplefs_init函数是加载到内核时的入口函数，主要执行了以下功能:

```c
simplefs_init()
- simplefs_init_inode_cache() // 初始化文件系统根目录的inode
  - kmem_cache_create(, sizeof(struct simplefs_inode_info),...) // 申请内存
- register_filesystem(&simplefs_file_system_type) // 注册文件系统，并参数传入文件系统类型
```

注册文件系统时，传入的参数，以全局变量形式定义在fs.c中，填充了SimpleFS必要的一些信息:

```c
static struct file_system_type simplefs_file_system_type = {
    .owner = THIS_MODULE, 
    .name = "simplefs",
    .mount = simplefs_mount, // 注册执行mount命令时的回调函数
    .kill_sb = simplefs_kill_sb, // 注册执行umount时的回调函数
    .fs_flags = FS_REQUIRES_DEV,
    .next = NULL,
};
```



## simplefs_mount函数

simplefs_mount函数是在注册文件系统时，作为参数传入的，当执行mount挂载一个文件系统时，会回调到该函数上来。

simplefs_mount函数中直接调用了mount_bdev()函数:

```c
struct dentry *simplefs_mount(struct file_system_type *fs_type,
                              int flags,
                              const char *dev_name,
                              void *data)
{
    struct dentry *dentry =
        mount_bdev(fs_type, flags, dev_name, data, simplefs_fill_super);
    ...
}
```

mount_bdev是用来执行块设备挂载的函数，主要做了几件事情:

* 根据设备名称dev_name，找到已经注册的块设备block_device。
* 根据文件系统类型fs_type，对应的块设备，找到或者新建一个超级块并返回指针。
* 如果超级块指针的s_root不为空，表明已经挂载过该文件系统，需要进行挂载flag的冲突判断等处理。
* 如果为空，表示首次挂载，则在指针中填充mode, id, blocksize属性后，调用传入的回调函数，对超级块进行后续的初始化过程。
* 最终返回文件系统根目录root的指针(dentry*)。



## simplefs_fill_super函数

mount_bdev函数最终会回调到SimpleFS的simplefs_fill_super函数上，进行超级块结构体的剩余的填充工作，并完成挂载操作。主要的步骤实现如下:

```C
int simplefs_fill_super(struct super_block *sb, void *data, int silent)
{
    ...
    /* Init sb */
    sb->s_magic = SIMPLEFS_MAGIC; // 设置预定义的魔数
    sb_set_blocksize(sb, SIMPLEFS_BLOCK_SIZE); // 跟新块大小
    sb->s_maxbytes = SIMPLEFS_MAX_FILESIZE; // 设置最大文件大小
    sb->s_op = &simplefs_super_ops; // 实现super_operations中定义的接口，比如sync_fs, statfs等

    /* Read sb from disk */
    bh = sb_bread(sb, SIMPLEFS_SB_BLOCK_NR); // 从块设备的第0号block中读取数据到buffer中(已设置blocksize)
    csb = (struct simplefs_sb_info *) bh->b_data; // 指向块设备中存储的超级块信息

    /* Check magic number */
    if (csb->magic != sb->s_magic) { // 检查块设备读到的魔数，与软件定义的是否匹配(区分版本?)
        goto release;
    }

    /* Alloc sb_info */
    sbi = kzalloc(sizeof(struct simplefs_sb_info), GFP_KERNEL); // 重新申请一块内存保存超级块信息

    sbi->nr_blocks = csb->nr_blocks; // 从buffer中拷贝数据填充到新分配的内存中
    sbi->nr_inodes = csb->nr_inodes;
    sbi->nr_istore_blocks = csb->nr_istore_blocks;
    sbi->nr_ifree_blocks = csb->nr_ifree_blocks;
    sbi->nr_bfree_blocks = csb->nr_bfree_blocks;
    sbi->nr_free_inodes = csb->nr_free_inodes;
    sbi->nr_free_blocks = csb->nr_free_blocks;
    sb->s_fs_info = sbi; // 把通用的超级块结构体的s_fs_info属性指向SimpleFS定义的超级块信息中

    brelse(bh); // 释放buffer_head

    /* Alloc and copy ifree_bitmap */
    // 以下部分把存储在设备中的ifree_bitmap拷贝到内存中，全部在内存中缓存起来
    sbi->ifree_bitmap = // inode空闲位图指针分配内存, 大小与块设备中一样，即块数目*块大小(4KB)
        kzalloc(sbi->nr_ifree_blocks * SIMPLEFS_BLOCK_SIZE, GFP_KERNEL);
    ...
    for (i = 0; i < sbi->nr_ifree_blocks; i++) {
        int idx = sbi->nr_istore_blocks + i + 1;

        bh = sb_bread(sb, idx); // 根据块索引, 从块设备中读取数据
        ...
        memcpy((void *) sbi->ifree_bitmap + i * SIMPLEFS_BLOCK_SIZE, bh->b_data,
               SIMPLEFS_BLOCK_SIZE); // 填充到ifree_bitmap指针对应的offset上
        ...
    }

    /* Alloc and copy bfree_bitmap */
    // 与ifree_bitmap类似，以下部分拷贝bfree_bitmap
    sbi->bfree_bitmap =
        kzalloc(sbi->nr_bfree_blocks * SIMPLEFS_BLOCK_SIZE, GFP_KERNEL);
    ...
    for (i = 0; i < sbi->nr_bfree_blocks; i++) {
        int idx = sbi->nr_istore_blocks + sbi->nr_ifree_blocks + i + 1;

        bh = sb_bread(sb, idx);
        ...
        memcpy((void *) sbi->bfree_bitmap + i * SIMPLEFS_BLOCK_SIZE, bh->b_data,
               SIMPLEFS_BLOCK_SIZE);
        ...
    }

    /* Create root inode */
    root_inode = simplefs_iget(sb, 0); //因为0号inode在格式化时已经写入信息到块设备，故从块设备读取数据并填充内存即可。
    ...
    inode_init_owner(root_inode, NULL, root_inode->i_mode);
    sb->s_root = d_make_root(root_inode); // 使用根inode创建对应的dentry，并填充到超级块的s_root
    ...
    return 0;
// 异常处理
... 
}
```

至此，超级块的信息已经填充完毕，也就相应地完成了挂载的主要工作。

到这里，我们已经积累了足够多的经验，能够进一步探索SimpleFS是如何进行I/O操作的了。

