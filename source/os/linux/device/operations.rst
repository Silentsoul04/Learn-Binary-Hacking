Linux字符设备驱动 file_operations
========================================
file_operations 在 ``/include/linux/fs.h`` 中被定义，其结构如下：

.. code-block:: cpp

    struct file_operations {
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
        unsigned long mmap_supported_flags;
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
        long (*fallocate)(struct file *file, int mode, loff_t offset,
                  loff_t len);
        void (*show_fdinfo)(struct seq_file *m, struct file *f);
    #ifndef CONFIG_MMU
        unsigned (*mmap_capabilities)(struct file *);
    #endif
        ssize_t (*copy_file_range)(struct file *, loff_t, struct file *,
                loff_t, size_t, unsigned int);
        loff_t (*remap_file_range)(struct file *file_in, loff_t pos_in,
                       struct file *file_out, loff_t pos_out,
                       loff_t len, unsigned int remap_flags);
        int (*fadvise)(struct file *, loff_t, loff_t, int);
    } __randomize_layout;


owner
----------------------------------------
owner不是一个操作，它是一个指向拥有这个结构的模块的指针，这个成员用来阻止模块还在被使用时被重复卸载。在大部分场景下它被简单初始化为 THIS_MODULE, 一个在 <linux/module.h> 中定义的宏。

llseek
----------------------------------------
llseek指针参数filp为进行读取信息的目标文件结构体指针；参数 p 为文件定位的目标偏移量；参数orig为对文件定位的起始地址，这个值可以为文件开头(SEEK_SET,0)，当前位置(SEEK_CUR,1)，文件末尾(SEEK_END,2)。

llseek 方法用作改变文件中的当前读/写位置, 并且新位置作为(正的)返回值。

loff_t 参数是一个"long offset"，并且就算在 32位平台上也至少 64 位宽。错误由一个负返回值指示；如果这个函数指针是 NULL，seek调用会以潜在地无法预知的方式修改 file 结构中的位置计数器。

read
----------------------------------------
指针参数 filp 为进行读取信息的目标文件，指针参数buffer为对应放置信息的缓冲区（即用户空间内存地址），参数size为要读取的信息长度，参数p为读的位置相对于文件开头的偏移，在读取信息后，这个指针一般都会移动，移动的值为要读取信息的长度值。

这个函数用来从设备中获取数据。在这个位置的一个空指针导致 read 系统调用以 -EINVAL("Invalid argument") 失败。一个非负返回值代表了成功读取的字节数(返回值是一个 "signed size" 类型, 常常是目标平台本地的整数类型)。

aio_read
----------------------------------------
aio_read函数的第一、三个参数和本结构体中的read()函数的第一、三个参数是不同的，异步读写的第三个参数直接传递值，而同步读写的第三个参数传递的是指针，因为AIO从来不需要改变文件的位置。异步读写的第一个参数为指向kiocb结构体的指针，而同步读写的第一参数为指向file结构体的指针，每一个I/O请求都对应一个kiocb结构体。

write
----------------------------------------
参数filp为目标文件结构体指针，buffer为要写入文件的信息缓冲区，count为要写入信息的长度，ppos为当前的偏移位置，这个值通常是用来判断写文件是否越界。函数的返回值代表成功写的字节数。

aio_write
----------------------------------------
初始化设备上的一个异步写，参数类型同aio_read()函数。

readdir
----------------------------------------
对于设备文件这个成员应当为 NULL; 它用来读取目录, 并且仅对文件系统有意义。

poll
----------------------------------------
这是一个设备驱动中的轮询函数，第一个参数为file结构指针，第二个为轮询表指针。

这个函数返回设备资源的可获取状态，即POLLIN、POLLOUT、POLLPRI、POLLERR、POLLNVAL等宏的位“或”结果。每个宏都表明设备的一种状态，如POLLIN（定义为0x0001）意味着设备可以无阻塞的读，POLLOUT（定义为0x0004）意味着设备可以无阻塞的写。

poll方法是poll、epoll和select 3个系统调用的后端，都用作查询对一个或多个文件描述符的读或写是否会阻塞。

poll方法应当返回一个位掩码指示是否非阻塞的读或写是可能的，并且提供给内核信息用来使调用进程睡眠直到 I/O 变为可能。如果一个驱动的 poll 方法为 NULL, 设备假定为不阻塞地可读可写。

这里通常将设备看作一个文件进行相关的操作，而轮询操作的取值直接关系到设备的响应情况，可以是阻塞操作结果，同时也可以是非阻塞操作结果。

ioctl
----------------------------------------
inode 和 filp 指针是对应应用程序传递的文件描述符 fd 的值, 和传递给 open 方法的相同参数。cmd 参数从用户那里不改变地传下来, 并且可选的参数 arg 参数以一个 unsigned long 的形式传递，不管它是否由用户给定为一个整数或一个指针。如果调用程序不传递第 3 个参数，被驱动操作收到的arg值是无定义的。

因为类型检查在这个额外参数上被关闭，编译器不能警告你如果一个无效的参数被传递给ioctl，并且任何关联的错误将难以查找。

ioctl 系统调用提供了发出设备特定命令的方法。另外, 几个 ioctl 命令被内核识别而不必引用 fops 表。如果设备不提供 ioctl 方法, 对于任何未事先定义的请求，系统调用返回一个错误。

mmap
----------------------------------------
mmap 用来请求将设备内存映射到进程的地址空间。 

open
----------------------------------------
inode 为文件节点,这个节点只有一个，无论用户打开多少个文件，都只是对应着一个inode结构；但是filp只要打开一个文件，就对应着一个file结构体，file结构体通常用来追踪文件在运行时的状态信息

flush
----------------------------------------
flush 操作在进程关闭它的设备文件描述符的拷贝时调用，它执行并且等待设备的任何未完成的操作。

release
----------------------------------------
当最后一个打开设备的用户进程执行close()系统调用的时候，内核将调用驱动程序release()函数。release函数的主要任务是清理未结束的输入输出操作，释放资源，用户自定义排他标志的复位等。在文件结构被释放时引用这个操作。如同 open，release可以为 NULL。

synch
----------------------------------------
刷新待处理的数据,允许进程把所有的脏缓冲区刷新到磁盘。 

aio_fsync
----------------------------------------
aio_fsync是fsync方法的异步版本。所谓的fsync方法是一个系统调用函数。系统调用fsync把文件所指定的文件的所有脏缓冲区写到磁盘中（如果需要，还包括存有索引节点的缓冲区）。相应的服务例程获得文件对象的地址，并随后调用fsync方法。通常这个方法以调用函数__writeback_single_inode()结束，这个函数把与被选中的索引节点相关的脏页和索引节点本身都写回磁盘。

fasync
----------------------------------------
fasync函数是系统支持异步通知的设备驱动。

lock
----------------------------------------
lock 方法用来实现文件加锁。

readv / writev
----------------------------------------
readv / writev实现发散/汇聚读和写操作。应用程序偶尔需要做一个包含多个内存区的单个读或写操作，这些系统调用允许它们这样做而不必对数据进行额外拷贝。如果参数中的函数指针为 NULL，read 和 write 方法被调用。

sendfile
----------------------------------------
sendfile实现 sendfile 系统调用的读, 使用最少的拷贝从一个文件描述符搬移数据到另一个。

sendpage
----------------------------------------
sendpage 是 sendfile 的另一半; 它由内核调用来发送数据，一次一页，到对应的文件。

get_unmapped_area
----------------------------------------
这个方法的目的是在进程的地址空间找一个合适的位置来映射在底层设备上的内存段中。这个任务通常由内存管理代码进行，这个方法存在为了使驱动能强制特殊设备可能有的任何的对齐请求。大部分驱动可以置这个方法为 NULL。

check_flags
----------------------------------------
check_flags用于模块检查传递给 fnctl(F_SETFL...) 调用的标志。

dir_notify
----------------------------------------
dir_notify在应用程序使用 fcntl 来请求目录改变通知时调用。只对文件系统有用，驱动不需要实现 dir_notify。
