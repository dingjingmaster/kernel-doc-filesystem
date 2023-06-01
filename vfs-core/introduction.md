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

