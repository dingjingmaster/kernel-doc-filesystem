## 说明

虚拟文件系统是内核中的软件层，它为用户空间程序提供统一的文件系统操作接口；为内核中各种文件系统的实现提供了一个抽象。

`VFS` 系统调用主要有：`open(2)`, `stat(2)`, `read(2)`, `write(2)`, `chmod(2)` 等等。


## 目录条目缓存(`dcache`)

VFS实现了`open(2)`、`stat(2)`、`chmod(2)`和类似的系统调用。传递给它们的`pathname`参数被`VFS`用来搜索目录条目缓存(也称为`dentry缓存`或`dcache`)。这提供了一种非常快速的查找机制，可以将路径名(filename)转换为特定的条目。`dentry`位于`RAM`中，从不保存到磁盘：它们的存在只是为了性能。

`dentry`缓存就是整个文件空间的视图，但是由于大多数计算机无法同时在`RAM`中容纳所有`dentry`条目，所以`RAM`中的`dentry`并不完整。为了将路径名解析为一个`dentry`, `VFS`可能不得不在此过程中创建`dentry`，然后加载`inode`。

## 索引节点(`Inode Object`)

一个单独的`dentry`通常有一个指向索引节点(`Inode`)的指针。`inode`是文件系统对象，比如`普通文件`、`目录`、`fifo`和其他东西。它们要么存在于磁盘上(对于块设备文件系统)，要么存在于内存中(对于伪文件系统)。在需要时，将磁盘上的`inode`复制到内存中，并将对`inode`的更改写回磁盘。单个`inode`可以被多个`dentry`指向(例如：硬链接就是这样做的)。

要查找`inode`，`VFS`需要调用父目录索引节点的`lookup()`方法。该方法由`inode`所在的特定文件系统实现安装。一旦`VFS`有了所需的`dentry`(也就有了`inode`)，我们就可以做所有无聊的事情，比如`open(2)`文件，或者`stat(2)`文件来查看`inode`数据。`stat(2)`操作相当简单:一旦`VFS`拥有了条目，它就会查看索引节点数据，并将其中一些数据传回用户空间。

## 文件对象(`File Object`)

打开文件需要另一个操作:文件结构的分配(这是文件描述符的内核端实现)。使用指向`dentry`的指针和一组`文件操作成员函数`*初始化*新分配的文件结构。这些是从索引节点数据中获取的。然后调用`open()`文件方法，以便特定的文件系统实现可以完成它的工作。文件对象结构被放置到进程的文件描述符表中。

读取、写入和关闭文件(以及其他各种`VFS`操作)是通过使用用户空间文件描述符获取适当的文件结构，然后调用所需的文件结构方法来执行所需的操作来完成的。只要文件是打开的，它就保持`dentry`被占用状态，这反过来意味着`VFS` `inode`一直使用中。

## 注册并挂载一个文件系统

注册、注销一个文件系统，使用如下函数接口：
```c
#include <linux/fs.h>

extern int register_filesystem(struct file_system_type *);
extern int unregister_filesystem(struct file_system_type *);
```

传递的 `struct file_system_type` 描述您要注册的文件系统。当要求将文件系统挂载到名称空间中的目录时，VFS将为特定的文件系统调用适当的`mount()`方法。`->mount()`返回的`vfsmount`树将附加到挂载点，因此当路径名解析到达挂载点时，它将跳转到该`vfmount`的根目录。

你可以在`/proc/filesystems`文件里看到注册到内核中的所有文件系统。

### `struct file_system_type`

此结构体描述了文件系统。从内核2.6.39开始，定义了以下成员:

```c
struct file_system_type 
{
    const char *name;
    int fs_flags;

    struct dentry *(*mount) (struct file_system_type* fs_type, int flags, const char* dev_name, void* data);
    void (*kill_sb) (struct super_block *);

    struct module *owner;
    struct file_system_type * next;
    struct list_head fs_supers;
    struct lock_class_key s_lock_key;
    struct lock_class_key s_umount_key;
};
```

 

|    参数    | 说明                                                         |
| :--------: | :----------------------------------------------------------- |
|   `name`   | 文件系统的名字，比如："`ext2`"、"`iso9660`"、"`msdos`"       |
| `fs_flags` | 各种 flags (比如：`FS_REQUIRES_DEV`, `FS_NO_DCACHE`, 等...)  |
|  `mount`   | 当一个新的文件系统实例挂载时候需要调用的方法。参数`flags`：表示挂载参数；`data`：表示挂载参数——"Mount Options"。<br/>这些参数与系统调用`mount(2)`的参数相匹配，它们的解释取决于文件系统类型。例如，对于块文件系统，`dev_name`被解释为块设备名称，该设备被打开，如果它包含合适的文件系统映像，该方法创建并初始化`struct super_block`，并将其`root dentry`返回给调用者。 |
| `kill_sb`  | 当一个文件系统实例要取消挂载（关闭）时候调用的方法           |
|  `owner`   | 对于内部`VFS`使用:在大多数情况下，您应该将其初始化为`THIS_MODULE` |
|   `next`   | 对于内部`VFS`使用:您应该将其初始化为`NULL`                   |

`mount()`方法必须返回挂载文件系统的根目录。可以获取挂载文件系统的`superblock`，并且`superblock`必须被锁定。如果失败，它应该返回`ERR_PTR(error)`。

`->mount()`一个已成功挂载的文件系统，则可以选择返回已挂载文件系统的子树（它不需要创建一个新树）；但是这一操作一定会重新创建新的超级块。

`mount()`方法填充的超级块结构中，最有趣的成员是“`s_op`”字段。这是一个指向“`struct super_operations`”的指针，它描述了文件系统实现的下一级。

通常，文件系统使用通用的`mount()`实现之一，并提供`fill_super()`回调。通用变体有：

- `mount_bdev`：挂载驻留在块设备上的文件系统
- `mount_nodev`：挂载一个没有设备支持的文件系统
- `mount_single`：挂载一个在所有挂载之间共享实例的文件系统

`fill_super()`回调实现有以下参数：

- `struct super_block *sb`：超级块体结构。回调必须正确地初始化它。
- `void* data`：任意挂载选项，通常以ASCII字符串的形式出现(参见“Mount Options”一节)
- `int silent`：是否对错误保持沉默

## 超级块对象(`Superblock Object`)

一个超级块对象表示一个挂载的文件系统。

### `struct super_operations`

这描述了`VFS`如何操作文件系统的超级块。从内核`2.6.22`开始，定义了以下成员：

```
struct super_operations
{
	struct inode* (*alloc_inode)(struct super_block* sb);
    void (*destroy_inode)(struct inode*);

    void (*dirty_inode) (struct inode*, int flags);
    int (*write_inode) (struct inode*, int);
    void (*drop_inode) (struct inode*);
    void (*delete_inode) (struct inode*);
    void (*put_super) (struct super_block*);
    int (*sync_fs)(struct super_block* sb, int wait);
    int (*freeze_fs) (struct super_block*);
    int (*unfreeze_fs) (struct super_block *);
    int (*statfs) (struct dentry *, struct kstatfs *);
    int (*remount_fs) (struct super_block *, int *, char *);
    void (*clear_inode) (struct inode *);
    void (*umount_begin) (struct super_block *);

    int (*show_options)(struct seq_file *, struct dentry *);

    ssize_t (*quota_read)(struct super_block *, int, char *, size_t, loff_t);
    ssize_t (*quota_write)(struct super_block *, int, const char *, size_t, loff_t);
    int (*nr_cached_objects)(struct super_block *);
    void (*free_cached_objects)(struct super_block *, int);
};
```

除非另有说明，否则调用所有方法时不会持有任何锁。这意味着大多数方法都可以安全地阻塞。所有的方法都只能从进程上下文调用(也就是说，不能从中断处理程序或下半部调用)。

| 函数                 | 说明                                                         |
| -------------------- | ------------------------------------------------------------ |
| `alloc_inode`        | 该方法由`alloc_inode()`调用，为`struct inode`分配内存并对其进行初始化。如果没有定义此函数，则分配一个简单的 `struct inode`。通常，`alloc_inode`将用于分配一个更大的结构，其中包含嵌入的`struct inode`。 |
| `destroy_inode`      | 此方法由`destroy_inode()`调用，以释放为 `struct inode` 分配的资源。只有在定义了`->alloc_inode`时才需要它，并且简单地撤销`->alloc_inode`所做的任何事情。 |
| `dirty_inode`        | 当索引节点被标记为`dirty`时，`VFS`调用这个方法。这是专门针对被标记为`dirty`的索引节点本身，而不是它的数据。如果更新需要由`fdatasync()`持久化，那么在`flags`参数中将设置`I_DIRTY_DATASYNC`。如果`lazytime`被启用，并且`struct inode`自上次`->dirty_inode`调用以来更新了数次，则`I_DIRTY_TIME`将在`flags`中设置。 |
| `write_inode`        | 当`VFS`需要向磁盘写入索引节点时调用这个方法。第二个参数指示写是否应该是同步的，并不是所有的文件系统都检查这个标志。 |
| `drop_inode`         | 当对`inode`的最后一次访问被删除，并持有`inode->i_lock`自旋锁时调用。<br/><br/>该方法应该是`NULL`(普通的`UNIX`文件系统语义) 或 "`generic_delete_inode`"(对于不想缓存`inode`的文件系统-导致无论`i_nlink`的值如何，总是调用"`delete_inode`")。<br/><br/>“`generic_delete_inode()`”行为相当于在`put_inode()`情况下使用“`force_delete`”的旧做法，但不具有“`force_delete()`”方法所具有的竞争。 |
| `delete_inode`       | 当`VFS`想要删除索引节点时调用                                |
| `put_super`          | 当`VFS`希望释放超级块(即卸载)时调用。这是在持有超级块锁时调用的 |
| `sync_fs`            | 当`VFS`写入与超级块相关的所有脏数据时调用。第二个参数指示该方法是否应该等待直到写入完成。可选的。 |
| `freeze_fs`          | 当`VFS`锁定文件系统并强制其进入一致状态时调用。`LVM (Logical Volume Manager)`当前使用该方法。 |
| `unfreeze_fs`        | 当`VFS`解锁文件系统并使其再次可写时调用。                    |
| `statfs`             | 当`VFS`需要获取文件系统统计信息时调用。                      |
| `remount_fs`         | 在重新挂载文件系统时调用。这是在持有内核锁的情况下调用的     |
| `clear_inode`        | 当`VFS`清除索引节点时候调用。可选                            |
| `umount_begin`       | 当`VFS`卸载文件系统时调用                                    |
| `show_options`       | 由`VFS`调用，显示`/proc/<pid>/mounts`的挂载选项。(请参阅“挂载选项”一节) |
| `quota_read`         | 由`VFS`调用从文件系统配额文件中读取。                        |
| `quota_write`        | 由`VFS`调用，用于写入文件系统配额文件。                      |
| `nr_cached_objects`  | 由`sb`缓存收缩函数调用，用于文件系统返回其包含的可空闲缓存对象的数量。可选的。 |
| `free_cache_objects` | 由`sb`缓存收缩函数调用，用于文件系统扫描指定的对象数量，以尝试释放它们。可选的，但是任何实现此方法的文件系统也需要实现`->nr_cached_objects`才能正确调用它。<br/><br/>我们无法处理文件系统可能遇到的任何错误，因此使用`void`返回类型。如果`VM`在`GFP_NOFS`条件下尝试回收，则永远不会调用此方法，因此此方法不需要本身处理该情况。<br/><br/>实现必须在完成的任何扫描循环中包含条件重调度调用。这允许`VFS`确定适当的扫描批处理大小，而不必担心实现是否会由于大的扫描批处理大小而导致延迟问题。 |

设置`inode`的人负责填写“`i_op`”字段。这是一个指向“`struct inode_operations`”的指针，它描述了可以在单个`inode`上执行的方法。

### `struct xattr_handlers`

在支持扩展属性(`xattrs`)的文件系统上，`s_xattr`超级块字段指向一个以`null`结尾的`xattr`处理程序数组。扩展属性是`key:value`对。

| 键       | 说明                                                         |
| -------- | ------------------------------------------------------------ |
| `name`   | 指示处理程序匹配具有指定名称的属性(例如"`system.posix_acl_access`");前缀字段必须为`NULL`。 |
| `prefix` | 指示处理程序匹配具有指定名称前缀(如“`user`”)的所有属性;`name`字段必须为`NULL`。 |
| `list`   | 确定是否应该列出与该`xattr`处理程序匹配的属性。由一些`listxattr`实现使用，如`generic_listxattr`。 |
| `get`    | 由`VFS`调用以获取特定扩展属性的值。该方法由`getxattr(2)`系统调用调用。 |
| `set`    | 由`VFS`调用，以设置特定扩展属性的值。当新值为`NULL`时，调用以删除特定的扩展属性。该方法由`setxattr(2)`和`removexattr(2)`系统调用调用。 |

当文件系统的所有`xattr`处理程序都与指定的属性名称不匹配时，或者文件系统不支持扩展属性时，各种*`xattr(2)`系统调用返回`-EOPNOTSUPP`。

## `Inode对象(Inode Object)`

`inode object`表示文件系统中的对象。

### `struct inode_operations`

```c
struct inode_operations
{
    int (*create) (struct user_namespace *, struct inode *,struct dentry *, umode_t, bool);
    struct dentry * (*lookup) (struct inode *,struct dentry *, unsigned int);
    int (*link) (struct dentry *,struct inode *,struct dentry *);
    int (*unlink) (struct inode *,struct dentry *);
    int (*symlink) (struct user_namespace *, struct inode *,struct dentry *,const char *);
    int (*mkdir) (struct user_namespace *, struct inode *,struct dentry *,umode_t);
    int (*rmdir) (struct inode *,struct dentry *);
    int (*mknod) (struct user_namespace *, struct inode *,struct dentry *,umode_t,dev_t);
    int (*rename) (struct user_namespace *, struct inode *, struct dentry *, struct inode *, struct dentry *, unsigned int);
    int (*readlink) (struct dentry *, char __user *,int);
    const char *(*get_link) (struct dentry *, struct inode *, struct delayed_call *);
    int (*permission) (struct user_namespace *, struct inode *, int);
    struct posix_acl * (*get_acl)(struct inode *, int, bool);
    int (*setattr) (struct user_namespace *, struct dentry *, struct iattr *);
    int (*getattr) (struct user_namespace *, const struct path *, struct kstat *, u32, unsigned int);
    ssize_t (*listxattr) (struct dentry *, char *, size_t);
    void (*update_time)(struct inode *, struct timespec *, int);
    int (*atomic_open)(struct inode *, struct dentry *, struct file *, unsigned open_flag, umode_t create_mode);
    int (*tmpfile) (struct user_namespace *, struct inode *, struct file *, umode_t);
    int (*set_acl)(struct user_namespace *, struct inode *, struct posix_acl *, int);
    int (*fileattr_set)(struct user_namespace *mnt_userns, struct dentry *dentry, struct fileattr *fa);
    int (*fileattr_get)(struct dentry *dentry, struct fileattr *fa);
};
```

同样，除非另有说明，否则调用所有方法时不会持有任何锁。

| 调用           | 说明                                                         |
| -------------- | ------------------------------------------------------------ |
| `create`       | 由`open(2)`和`create(2)`系统调用来调用。只有当您想要支持常规文件时才需要。您得到的`dentry`不应该有索引节点(即，它应该是一个负`dentry`)。在这里，您可能会使用`dentry`和新创建的`inode`调用`d_instantiate()` |
| `lookup`       | 当`VFS`需要在父目录中查找索引节点时调用。要查找的名称在`dentry`中找到。这个方法必须调用`d_add()`将找到的索引节点插入到条目中。`inode`结构中的“`i_count`”字段应该递增。如果指定的`inode`不存在，则应该将`NULL inode`插入到`dentry`中(这称为`负dentry`)。从这个例程返回错误代码必须只在发生实际错误时执行，否则使用`create(2)`、`mknod(2)`、`mkdir(2)`等系统调用创建索引节点将失败。如果你想重载`dentry`方法，那么你应该初始化`dentry`中的“`d_dop`”字段;这是一个指向“`struct dentry_operations`”的指针。此方法在持有目录索引节点信号量时调用 |
| `link`         | 由`link(2)`系统调用调用。只有当你想支持硬链接时才需要。您可能需要调用`d_instantiate()`，就像在`create()`方法中一样 |
| `unlink`       | `unlink(2)`系统调用调用。只有当您想要支持删除索引节点时才需要 |
| `symlink`      | 由`symlink(2)`系统调用调用。只有当您想要支持符号链接时才需要。您可能需要调用`d_instantiate()`，就像在`create()`方法中一样 |
| `mkdir`        | 由`mkdir(2)`系统调用调用。仅当您希望支持创建子目录时才需要。您可能需要调用`d_instantiate()`，就像在`create()`方法中一样 |
| `rmdir`        | 由`rmdir(2)`系统调用调用。只有当您希望支持删除子目录时才需要 |
| `mknod`        | 由`mknod(2)`系统调用调用，用于创建设备`(char, block) inode`或命名管道(`FIFO`)或`套接字`。只有当您希望支持创建这些类型的索引节点时才需要。您可能需要调用`d_instantiate()`，就像在`create()`方法中一样 |
| `rename`       | 由`rename(2)`系统调用调用，用于重命名对象，使其具有第二个`inode`和`dentry`给出的父节点和名称。<br/><br/>对于任何不支持的或未知的标志，文件系统必须返回`-EINVAL`。目前实现了以下标志:(1)`RENAME_NOREPLACE`：该标志表示如果重命名的目标存在，重命名应该使用`-EEXIST`失败，而不是替换目标。`VFS`已经检查是否存在，因此对于本地文件系统，`RENAME_NOREPLACE`实现等同于普通的重命名。(2) `RENAME_EXCHANGE`：交换源和目标。两者都必须存在;这是由`VFS`检查的。与普通的重命名不同，源和目标可以是不同的类型。 |
| `get_link`     | 由`VFS`调用，以遵循指向它所指向的索引节点的符号链接。只有当你想支持符号链接时才需要。此方法返回要遍历的符号链接主体(并可能使用`nd_jump_link()`重置当前位置)。如果在`inode`消失之前主体不会消失，则不需要其他任何东西;如果需要以其他方式固定它，则通过`get_link(…，…，done)`执行`set_delayed_call(done, destructor, argument)`来安排它的释放。在这种情况下，一旦`VFS`处理完返回的函数体，就会调用析构函数(参数)。可以在`RCU`模式下调用;由`NULL dentry`参数表示。如果请求不能在不离开`RCU`模式的情况下处理，让它返回`ERR_PTR(-ECHILD)`。<br/><br/>如果文件系统将符号链接目标存储在`->i_link`中，`VFS`可以直接使用它，而不需要调用`->get_link()`;但是，仍然必须提供`->get_link()`。`->i_link`必须在`RCU`宽限期之后才被释放。在`iget()`之后写入`->i_link`需要一个' `release` '内存屏障。 |
| `readlink`     | 当`->get_link`使用`nd_jump_link()`或`object`实际上不是符号链接时，这只是`readlink(2)`使用的覆盖。通常，文件系统应该只对符号链接实现`->get_link`, `readlink(2)`将自动使用它。 |
| `permission`   | 由`VFS`调用，用于检查类`posix`文件系统上的访问权限。<br/><br/>可以在`rcu-walk`模式下调用`(mask & MAY_NOT_BLOCK)`。如果是`rcu-walk`模式，文件系统必须在不阻塞或存储到索引节点的情况下检查权限。<br/><br/>如果遇到`rcu-walk`无法处理的情况，返回`-ECHILD`，它将在`ref-walk`模式下再次被调用。 |
| `setattr`      | 由`VFS`调用，用于设置文件的属性。这个方法由`chmod(2)`和相关的系统调用调用。 |
| `getattr`      | 由`VFS`调用以获取文件的属性。这个方法由`stat(2)`和相关的系统调用调用。 |
| `listxattr`    | 由`VFS`调用，列出给定文件的所有扩展属性。该方法由`listxattr(2)`系统调用调用。 |
| `update_time`  | 由`VFS`调用以更新`inode`的特定时间或`i_version`。如果没有定义，`VFS`将更新`inode`本身并调用`mark_inode_dirty_sync`。 |
| `atomic_open`  | 在开盘的最后一个分量上调用。使用这个可选方法，文件系统可以在一个原子操作中查找、可能创建和打开文件。如果它想把实际的打开留给调用者(例如，如果文件被证明是一个符号链接、设备，或者只是文件系统不会对其进行原子打开的东西)，它可以通过返回`finish_no_open(file, dentry)`来发出信号。此方法仅在最后一个组件为负或需要查找时调用。缓存的阳性`dentry`仍然由`f_op->open()`处理。如果文件已经创建，应该在`file->f_mode`中设置`FMODE_CREATED`标志。在`O_EXCL`的情况下，该方法必须只有在文件不存在时才成功，因此`FMODE_CREATED`必须总是在成功时设置。 |
| `tmpfile`      | 在`O_TMPFILE`的末尾调用`open()`。可选的，相当于在给定目录中自动创建、打开和取消文件链接。成功时需要带着已经打开的文件返回;这可以通过在末尾调用`finish_open_simple()`来完成。 |
| `fileattr_get` | 调用`ioctl(FS_IOC_GETFLAGS)`和`ioctl(FS_IOC_FSGETXATTR)`来检索杂项文件标志和属性。也在相关`SET`操作之前调用，以检查正在更改的内容(在本例中，`i_rwsem`被锁定为排他)。如果未设置，则返回到`f_op->ioctl()`。 |
| `fileattr_set` | 调用`ioctl(FS_IOC_SETFLAGS)`和`ioctl(FS_IOC_FSSETXATTR)`来更改杂项文件标志和属性。调用方保持`i_rwsem`独占。如果未设置，则返回到`f_op->ioctl()`。 |

## 地址空间对象（The Address Space Object）

地址空间对象用于对页缓存中的页进行分组和管理。它可以用来跟踪文件(或其他任何东西)中的页面，也可以跟踪文件部分到进程地址空间的映射。

地址空间可以提供许多不同但相关的服务。这些包括通信内存压力，按地址查找页面，以及跟踪标记为`Dirty`或`Writeback`的页面。

第一种方法可以独立于其他方法使用。`VM`可以尝试写脏页以清理它们，或者释放干净页以重用它们。要做到这一点，它可以在脏页上调用`->writepage`方法，在设置私有标志的干净页上调用`->release_folio`方法。没有`pagepprivate`和没有外部引用的干净页面将被释放，而不通知给`address_space`。

要实现此功能，需要将页面放置在`LRU`上，并且在使用页面时需要调用`lru_cache_add`和`mark_page_active`。

页面通常保存在一个基数树索引`->index`。该树维护关于每个页面的`PG_Dirty`和`PG_Writeback`状态的信息，因此可以快速找到具有这两个标志之一的页面。

脏标记主要由`mpage_writepages`(默认的`->writepages`方法)使用。它使用标记查找脏页以调用`->writepage`。如果没有使用`mpage_writepages`(即地址提供了它自己的`->writepages`)，`PAGECACHE_TAG_DIRTY`标签几乎是未使用的。`Write_inode_now`和`sync_inode`确实使用它(通过`__sync_single_inode`)来检查`->writpages`是否已经成功地写出了整个`address_space`。

`Writeback`标签由`filemapwait`和`sync_page`函数使用，通过`filemap_fdatawait_range`来等待所有回写完成。

`address_space`处理程序可以将额外的信息附加到页面上，通常使用'`struct page`'中的' `private` '字段。如果附加了此类信息，则应该设置`PG_Private`标志。这将导致各种`VM`例程对`address_space`处理程序进行额外调用，以处理该数据。

地址空间充当存储和应用程序之间的中介。数据一次整页地读入地址空间，并通过复制该页或通过对该页的内存映射提供给应用程序。数据由应用程序写入地址空间，然后通常以整个页面的形式回写到存储中，但是`address_space`对写入大小有更好的控制。

读取过程基本上只需要' `read_folio` '。写过程比较复杂，使用`write_begin`/`write_end`或`dirty_folio`将数据写入`address_space`，使用`writepage`和`writpages`将数据回写到存储。

在`address_space`中添加和删除页面受到`inode`的`i_mutex`的保护。

当数据写入页面时，应该设置`PG_Dirty`标志。它通常保持设置，直到`writepage`要求写入它。这将清除`PG_Dirty`并设置`PG_Writeback`。它实际上可以在`PG_Dirty`清除后的任何时刻写入。一旦知道它是安全的，`PG_Writeback`就会被清除。

回写使用`writeback_control`结构来指导操作。这为`writepage`和`writpages`操作提供了一些关于回写请求的性质和原因的信息，以及执行回写请求的约束条件。它还用于向调用者返回关于`writepage`或`writpages`请求结果的信息。

### 处理回写过程中的错误

大多数使用缓冲`I/O`的应用程序都会定期调用文件同步调用(`fsync`、`fdatasync`、`msync`或`sync_file_range`)，以确保写入的数据已经写入到后备存储。当回写过程中出现错误时，他们希望在发出文件同步请求时报告该错误。在一个请求报告错误后，对同一文件描述符的后续请求应该返回`0`，除非自上次文件同步以来发生了进一步的回写错误。

理想情况下，内核只会在文件描述上报告错误，这些文件描述上的写操作随后无法回写。但是，通用的`pagecache`基础结构不跟踪污染了每个单独页面的文件描述，因此不可能确定哪些文件描述符应该返回错误。

相反，内核中通用的回写错误跟踪基础结构决定在错误发生时向`fsync`报告所有打开的文件描述的错误。在有多个写入器的情况下，所有写入器都会在随后的`fsync`中返回一个错误，即使通过该特定文件描述符完成的所有写操作都成功了(或者即使根本没有对该文件描述符进行写操作)。

希望使用此基础结构的文件系统应该调用`mapping_set_error`，以便在发生错误时在`address_space`中记录错误。然后，在他们的`file->fsync`操作中从`pagecache`回写数据后，他们应该调用`file_check_and_advance_wb_err`来确保`struct`文件的错误游标已经前进到由备份设备发出的错误流中的正确位置。

### `struct address_space_operations`

这描述了`VFS`如何操作文件系统中文件到页面缓存的映射。定义了以下成员:

```c
struct address_space_operations 
{
    int (*writepage)(struct page *page, struct writeback_control *wbc);
    int (*read_folio)(struct file *, struct folio *);
    int (*writepages)(struct address_space *, struct writeback_control *);
    bool (*dirty_folio)(struct address_space *, struct folio *);
    void (*readahead)(struct readahead_control *);
    
    int (*write_begin)(struct file *, struct address_space *mapping,
                           loff_t pos, unsigned len,
                        struct page **pagep, void **fsdata);
    int (*write_end)(struct file *, struct address_space *mapping,
                         loff_t pos, unsigned len, unsigned copied,
                         struct page *page, void *fsdata);
    
    sector_t (*bmap)(struct address_space *, sector_t);
    void (*invalidate_folio) (struct folio *, size_t start, size_t len);
    bool (*release_folio)(struct folio *, gfp_t);
    void (*free_folio)(struct folio *);
    ssize_t (*direct_IO)(struct kiocb *, struct iov_iter *iter);
    int (*migrate_folio)(struct mapping *, struct folio *dst, struct folio *src, enum migrate_mode);
    int (*launder_folio) (struct folio *);

    bool (*is_partially_uptodate) (struct folio *, size_t from,
                                       size_t count);
    void (*is_dirty_writeback)(struct folio *, bool *, bool *);
    int (*error_remove_page) (struct mapping *mapping, struct page *page);
    int (*swap_activate)(struct swap_info_struct *sis, struct file *f, sector_t *span);
    int (*swap_deactivate)(struct file *);
    int (*swap_rw)(struct kiocb *iocb, struct iov_iter *iter);
};
```

| 函数                    | 说明                                                         |
| ----------------------- | ------------------------------------------------------------ |
| `writepage`             | 由`VM`调用，将脏页写入后备存储。这可能是出于数据完整性的原因(即“同步”)，或者是为了释放内存(刷新)。在`wbc->sync_mode`中可以看到差异。`PG_Dirty`标志已被清除，`pagellocked`为`true`。`writepage`应该开始写，应该设置`PG_Writeback`，并且应该确保在写操作完成时，页面被同步或异步地解锁。<br/><br/>如果`wbc->sync_mode`为`WB_SYNC_NONE`，则`->writepage`在出现问题时不必太过努力，如果这样做更容易(例如由于内部依赖关系)，则可以选择从映射中写出其他页面。如果它选择不开始写，它应该返回`AOP_WRITEPAGE_ACTIVATE`，这样`VM`就不会一直在该页上调用`->writepage`。<br/><br/>有关详细信息，请参阅文件“`Locking`”。 |
| `read_folio`            | 由页缓存调用，以从后备存储读取作品集。' `file` '参数为网络文件系统提供身份验证信息，通常不被基于块的文件系统使用。如果调用者没有打开的文件，它可能是`NULL`(例如，如果内核正在为自己执行读取，而不是代表具有打开文件的用户空间进程)。<br/><br/>如果映射不支持大的文件夹，那么文件夹将只包含一个页面。当调用`read_folio`时，该组合将被锁定。如果读取成功完成，则应标记为“更新”。无论读取是否成功，文件系统都应该在读取完成后解锁文件夹。文件系统不需要修改文件夹上的`refcount`;页面缓存保存一个引用计数，直到文件夹被解锁才会释放。<br/><br/>文件系统可以同步实现`->read_folio()`。在正常操作中，通过`->readahead()`方法读取文件夹。只有当这个操作失败，或者调用者需要等待读取操作完成时，页面缓存才会调用`->read_folio()`。文件系统不应该尝试在`->read_folio()`操作中执行自己的预读操作。<br/><br/>如果文件系统此时不能执行读取，它可以解锁文件夹，执行任何需要的操作以确保将来读取成功，并返回`AOP_TRUNCATED_PAGE`。在这种情况下，调用者应该查找组合，锁定它，然后再次调用`->read_folio`。<br/><br/>调用者可以直接调用`->read_folio()`方法，但使用`read_mapping_folio()`将负责锁定，等待读取完成并处理`AOP_TRUNCATED_PAGE`等情况。 |
| `writepages`            | 由`VM`调用，以写出与`address_space`对象相关联的页面。如果`wbc->sync_mode`为`WB_SYNC_ALL`，则`writeback_control`将指定必须写入的页面范围。如果是`WB_SYNC_NONE`，则给出`nr_to_write`，并且应该尽可能多地写入页面。如果没有`->writepages`，则使用`mpage_writepages`代替。这将从地址空间中选择标记为`DIRTY`的页面，并将它们传递给`->writepage`。 |
| `dirty_folio`           | 由`VM`调用，将一个文件夹标记为脏的。如果地址空间将私有数据附加到一个文件夹，并且当文件夹被污染时需要更新该数据，则特别需要这样做。例如，当内存映射页被修改时调用该函数。如果定义了，它应该设置`folio dirty`标志，并在`i_pages`中设置`PAGECACHE_TAG_DIRTY`搜索标记。 |
| `readahead`             | 由`VM`调用以读取与`address_space`对象关联的页面。这些页在页缓存中是连续的，并且被锁定。在每个页面上启动`I/O`后，实现应该减少页面重计数。通常，该页将由`I/O`完成处理程序解锁。页面集被分成一些同步页面，然后是一些异步页面，`rac->ra->async_size`给出了异步页面的数量。文件系统应该尝试读取所有同步页面，但可能会在到达异步页面时决定停止。如果它决定停止尝试`I/O`，它可以简单地返回。调用者将从地址空间中删除剩余的页面，解锁它们并减少页面重新计数。如果`I/O`成功完成，设置`pageupdate`。在任何页面上设置`PageError`都会被忽略;如果发生`I/O`错误，只需解锁页面。 |
| `write_begin`           | 由通用缓冲写代码调用，要求文件系统准备在文件中给定的偏移量处写入`len`字节。`address_space`应该通过必要时分配空间和执行任何其他内部管理来检查写操作是否能够完成。如果写操作将更新存储上任何基本块的一部分，那么应该预读这些块(如果它们还没有被读取)，以便更新的块可以正确地写出来。<br/><br/>文件系统必须为指定的偏移量(`*pagep`)返回锁定的`pagecache`页面，以便调用者写入。<br/><br/>它必须能够处理短写操作(传递给`write_begin`的长度大于复制到页面中的字节数)。<br/><br/>在`fsdata`中可能返回一个`void *`，然后传递给`write_end`。<br/><br/>成功时返回`0`;失败时`< 0`(这是错误码)，在这种情况下不会调用`write_end`。 |
| `write_end`             | 在`write_begin`和数据拷贝成功后，必须调用`write_end`。`Len`是传递给`write_begin`的原始`Len`, `copy`是能够复制的数量。<br/><br/>文件系统必须负责对页面进行解锁和释放(`refcount`)，并更新`i_size`。<br/><br/>失败时返回`< 0`，否则返回能够复制到`pagecache`中的字节数(`<= replicated`)。 |
| `bmap`                  | 由`VFS`调用，将对象内的逻辑块偏移量映射到物理块号。此方法由`FIBMAP ioctl`使用，用于处理交换文件。为了能够交换到文件，该文件必须具有到块设备的稳定映射。交换系统不遍历文件系统，而是使用`bmap`查找文件中的块的位置，并直接使用这些地址。 |
| `invalidate_folio`      | 如果一个文件夹有私有数据，那么当要从地址空间中删除部分或全部文件夹时，将调用`invalidate_folio`。这通常对应于截断，打孔或地址空间的完全无效(在后一种情况下`offset` 将始终为`0`，`length` 将为`folio_size()`)。任何与投资组合相关的私有数据都应该更新以反映这种截断。如果`offset`为`0`,`length`为`folio_size()`，那么私有数据应该被释放，因为`folio_size`必须能够被完全丢弃。这可以通过调用`->release_folio`函数来完成，但在这种情况下，`release`必须成功。 |
| `release_folio`         | 在具有私有数据的文件夹上调用`Release_folio`，以告诉文件系统该文件夹即将被释放。`->release_folio`应该从`folio`中删除所有私有数据并清除`private`标志。如果`release_folio()`失败，它应该返回`false`。`release_folio()`用于两种不同但相关的情况。第一种情况是`VM`想要释放一个没有活动用户的干净文件夹。如果`->release_folio`成功，该`folio`将从`address_space`中删除并被释放。<br/><br/>第二种情况是当请求使`address_space`中的部分或全部文件夹无效时。这可以通过`fadvise(POSIX_FADV_DONTNEED)`系统调用来实现，也可以像`nfs`和`9p`那样通过调用`invalidate_inode_pages2()`来显式请求它(当它们认为缓存可能已经过期时)。如果文件系统进行这样的调用，并且需要确定所有的`folio`都是无效的，那么它的`release_folio`将需要确保这一点。如果它还不能释放私有数据，它可能会清除`update`标志。 |
| `free_folio`            | 一旦`folio`在页面缓存中不再可见，`free_folio`将被调用，以便允许清除任何私有数据。由于它可能由内存回收器调用，因此它不应该假设原始`address_space`映射仍然存在，并且它不应该阻塞。 |
| `direct_IO`             | 由通用读/写例程调用来执行`direct_IO` -这是绕过页面缓存并直接在存储和应用程序的地址空间之间传输数据的`IO`请求。 |
| `migrate_folio`         | 这是用来压缩物理内存使用的。如果`VM`想要重新定位一个文件夹(可能是从一个即将发生故障的内存设备)，它将把一个新的文件夹和一个旧的文件夹传递给这个函数。`migrate_folio`应该传输任何私有数据，并更新它对`folio`的任何引用。 |
| `launder_folio`         | 在释放一个组合之前调用——它回写脏的组合。为了防止重新清理投资组合，它在整个操作过程中都是锁定的。 |
| `is_partially_uptodate` | 当底层块大小小于文件夹大小时，由`VM`在通过`pagecache`读取文件时调用。如果所需的块是最新的，那么读取就可以完成，而不需要`I/O`来更新整个页面。 |
| `is_dirty_writeback`    | 虚拟机在尝试回收一个文件夹时调用。虚拟机使用脏信息和回写信息来确定是否需要暂停，以便让`flush`有机会完成某些`IO`。通常它可以使用`folio_test_dirty`和`folio_test_writeback`，但是一些文件系统有更复杂的状态(`NFS`中不稳定的`folio`阻止回收)，或者由于锁定问题而没有设置这些标志。这个回调允许文件系统向`VM`指示是否应该将一个文件夹处理为脏的或回写的以达到停止的目的。 |
| `error_remove_page`     | 如果可以截断此地址空间，则通常设置为`generic_error_remove_page`。用于内存故障处理。设置此值意味着您要处理在您下面消失的页面，除非您将它们锁定或增加引用计数。 |
| `swap_activate`         | 调用以准备给定的文件进行交换。它应该执行任何必要的验证和准备，以确保可以用最小的内存分配执行写操作。它应该调用`add_swap_extent()`或助手`iomap_swapfile_activate()`，并返回添加的区段数量。如果要通过`->swap_rw()`提交`IO`，则应设置`SWP_FS_OPS`，否则`IO`将直接提交到块设备`sis->bdev`。 |
| `swap_deactivate`       | 在`swap_activate`成功执行的文件交换期间调用。                |
| `swap_rw`               | 当设置了`SWP_FS_OPS`时，调用该函数读取或写入交换页。         |

## 文件对象（File Object）

`file`对象表示由进程打开的文件。这在`POSIX`术语中也称为“打开文件描述”。

### `struct file_operations`

这描述了`VFS`如何操作打开的文件。从内核`4.18`开始，定义了以下成员:

```c
struct file_operations
{
    struct module *owner;
    loff_t (*llseek) (struct file *, loff_t, int);
    ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
    ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
    ssize_t (*read_iter) (struct kiocb *, struct iov_iter *);
    ssize_t (*write_iter) (struct kiocb *, struct iov_iter *);
    int (*iopoll)(struct kiocb *kiocb, bool spin);
    int (*iterate) (struct file *, struct dir_context *);
    int (*iterate_shared) (struct file *, struct dir_context *);
    __poll_t (*poll) (struct file *, struct poll_table_struct *);
    long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
    long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
    int (*mmap) (struct file *, struct vm_area_struct *);
    int (*open) (struct inode *, struct file *);
    int (*flush) (struct file *, fl_owner_t id);
    int (*release) (struct inode *, struct file *);
    int (*fsync) (struct file *, loff_t, loff_t, int datasync);
    int (*fasync) (int, struct file *, int);
    int (*lock) (struct file *, int, struct file_lock *);
    ssize_t (*sendpage) (struct file *, struct page *, int, size_t, loff_t *, int);
    unsigned long (*get_unmapped_area)(struct file *, unsigned long, unsigned long, unsigned long, unsigned long);
    int (*check_flags)(int);
    int (*flock) (struct file *, int, struct file_lock *);
    ssize_t (*splice_write)(struct pipe_inode_info *, struct file *, loff_t *, size_t, unsigned int);
    ssize_t (*splice_read)(struct file *, loff_t *, struct pipe_inode_info *, size_t, unsigned int);
    int (*setlease)(struct file *, long, struct file_lock **, void **);
    long (*fallocate)(struct file *file, int mode, loff_t offset, loff_t len);
    void (*show_fdinfo)(struct seq_file *m, struct file *f);
#ifndef CONFIG_MMU
    unsigned (*mmap_capabilities)(struct file *);
#endif
    ssize_t (*copy_file_range)(struct file *, loff_t, struct file *, loff_t, size_t, unsigned int);
    loff_t (*remap_file_range)(struct file *file_in, loff_t pos_in,
                                   struct file *file_out, loff_t pos_out,
                                   loff_t len, unsigned int remap_flags);
    int (*fadvise)(struct file *, loff_t, loff_t, int);
};
```

同样，除非另有说明，否则调用所有方法时不会持有任何锁。

| 函数                | 说明                                                         |
| ------------------- | ------------------------------------------------------------ |
| `llseek`            |                                                              |
| `read`              |                                                              |
| `read_iter`         | 可能是以`iov_iter`作为目标的异步读取                         |
| `write`             |                                                              |
| `write_iter`        |                                                              |
| `iopoll`            | 当`io`想要轮询`HIPRI iocb`上的完成时调用                     |
| `iterate`           | 当`VFS`需要读取目录内容时调用                                |
| `iterate_shared`    | 当文件系统支持并发目录迭代器时，`VFS`需要读取目录内容时调用  |
| `poll`              | 当进程想要检查这个文件上是否有活动，并且(可选地)进入休眠状态直到有活动时，由`VFS`调用。由`select(2)`和`poll(2)`系统调用调用 |
| `unlocked_ioctl`    | 由`ioctl(2)`系统调用调用。                                   |
| `compat_ioctl`      | 当32位系统调用时，由`ioctl(2)`系统调用调用<br/>用于64位内核。 |
| `mmap`              | `mmap(2)`                                                    |
| `open`              | 在应该打开索引节点时由`VFS`调用。当`VFS`打开一个文件时，它会创建一个新的`struct file`。然后，它为新分配的文件结构调用`open`方法。您可能认为`open`方法确实属于`struct inode_operations`，您可能是对的。我认为这样做是因为它使文件系统更容易实现。如果您想要指向设备结构，那么`open()`方法是初始化文件结构中的`private_data`成员的好地方 |
| `flush`             | 由`close(2)`系统调用调用来刷新文件                           |
| `release`           | 当对打开文件的最后一个引用关闭时调用                         |
| `fsync`             | 由`fsync(2)`系统调用调用。也请参阅上面题为“回写期间处理错误”的部分。 |
| `fasync`            | 当文件启用异步(非阻塞)模式时，由`fcntl(2)`系统调用调用       |
| `lock`              | 由`fcntl(2)`系统调用`F_GETLK`、`F_SETLK`和`F_SETLKW`命令调用 |
| `get_unmapped_area` | 由`mmap(2)`系统调用调用                                      |
| `check_flags`       | 由`fcntl(2)`系统调用`F_SETFL`命令调用                        |
| `flock`             | 由`flock(2)`系统调用调用                                     |
| `splice_write`      | 由`VFS`调用，用于将数据从管道拼接到文件中。该方法由`splice(2)`系统调用使用 |
|                     |                                                              |

