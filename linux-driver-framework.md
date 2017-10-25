# Linux驱动程序框架

Linux将所有外部设备看成是一类特殊文件，称之为“设备文件”，如果说系统调用是Linux内核和应用程序之间的接口，那么设备驱动程序则可以看成是Linux内核与外部设备之间的接口。设备驱动程序向应用程序屏蔽了硬件在实现上的细节，使得应用程序可以像操作普通文件一样来操作外部设备。
## 1. 字符设备和块设备
Linux抽象了对硬件的处理，所有的硬件设备都可以像普通文件一样来看待：它们可以使用和操作文件相同的、标准的系统调用接口来完成打开、关闭、读写和I/O控制操作，而驱动程序的主要任务也就是要实现这些系统调用函数。Linux系统中的所有硬件设备都使用一个特殊的设备文件来表示，例如，系统中的第一个IDE硬盘使用`/dev/hda`表示。每个设备文件对应有两个设备号：一个是主设备号，标识该设备的种类，也标识了该设备所使用的驱动程序；另一个是次设备号，标识使用同一设备驱动程序的不同硬件设备。设备文件的主设备号必须与设备驱动程序在登录该设备时申请的主设备号一致，否则用户进程将无法访问到设备驱动程序。
在Linux操作系统下有两类主要的设备文件：一类是字符设备，另一类则是块设备。字符设备是以字节为单位逐个进行I/O操作的设备，在对字符设备发出读写请求时，实际的硬件I/O紧接着就发生了，一般来说字符设备中的缓存是可有可无的，而且也不支持随机访问。块设备则是利用一块系统内存作为缓冲区，当用户进程对设备进行读写请求时，驱动程序先查看缓冲区中的内容，如果缓冲区中的数据能满足用户的要求就返回相应的数据，否则就调用相应的请求函数来进行实际的I/O操作。块设备主要是针对磁盘等慢速设备设计的，其目的是避免耗费过多的CPU时间来等待操作的完成。一般说来，PCI卡通常都属于字符设备。
所有已经注册（即已经加载了驱动程序）的硬件设备的主设备号可以从`/proc/devices`文件中得到。使用mknod命令可以创建指定类型的设备文件，同时为其分配相应的主设备号和次设备号。例如，下面的命令：
```
1 [root@gary root]# mknod  /dev/lp0  c  6  0
```
将建立一个主设备号为6，次设备号为0的字符设备文件`/dev/lp0`。当应用程序对某个设备文件进行系统调用时，Linux内核会根据该设备文件的设备类型和主设备号调用相应的驱动程序，并从用户态进入到核心态，再由驱动程序判断该设备的次设备号，最终完成对相应硬件的操作。
## 2. 设备驱动程序接口
Linux中的I/O子系统向内核中的其他部分提供了一个统一的标准设备接口，这是通过`include/linux/fs.h`中的数据结构`file_operations`来完成的：
```
struct file_operations {
        struct module *owner;
        loff_t (*llseek) (struct file *, loff_t, int);
        ssize_t (*read) (struct file *, char *, size_t, loff_t *);
        ssize_t (*write) (struct file *, const char *, size_t, loff_t *);
        int (*readdir) (struct file *, void *, filldir_t);
        unsigned int (*poll) (struct file *, struct poll_table_struct *);
        int (*ioctl) (struct inode *, struct file *, unsigned int, unsigned long);
        int (*mmap) (struct file *, struct vm_area_struct *);
        int (*open) (struct inode *, struct file *);
        int (*flush) (struct file *);
        int (*release) (struct inode *, struct file *);
        int (*fsync) (struct file *, struct dentry *, int datasync);
        int (*fasync) (int, struct file *, int);
        int (*lock) (struct file *, int, struct file_lock *);
        ssize_t (*readv) (struct file *, const struct iovec *, unsigned long, loff_t *);
        ssize_t (*writev) (struct file *, const struct iovec *, unsigned long, loff_t *);
        ssize_t (*sendpage) (struct file *, struct page *, int, size_t, loff_t *, int);
        unsigned long (*get_unmapped_area)(struct file *, unsigned long, 
         unsigned long, unsigned long, unsigned long);
};
```
当应用程序对设备文件进行诸如open、close、read、write等操作时，Linux内核将通过file_operations结构访问驱动程序提供的函数。例如，当应用程序对设备文件执行读操作时，内核将调用file_operations结构中的read函数。
## 3. 设备驱动程序模块
Linux下的设备驱动程序可以按照两种方式进行编译，一种是直接静态编译成内核的一部分，另一种则是编译成可以动态加载的模块。如果编译进内核的话，会增加内核的大小，还要改动内核的源文件，而且不能动态地卸载，不利于调试，所有推荐使用模块方式。
从本质上来讲，模块也是内核的一部分，它不同于普通的应用程序，不能调用位于用户态下的C或者C++库函数，而只能调用Linux内核提供的函数，在`/proc/ksyms`中可以查看到内核提供的所有函数。
在以模块方式编写驱动程序时，要实现两个必不可少的函数`init_module()`和`cleanup_module()`，而且至少要包含`<linux/krernel.h>`和`<linux/module.h>`两个头文件。在用gcc编译内核模块时，需要加上 `-DMODULE -D__KERNEL__ -DLINUX` 这几个参数，编译生成的模块（一般为.o文件）可以使用命令insmod载入Linux内核，从而成为内核的一个组成部分，此时内核会调用模块中的函数`init_module( )`。当不需要该模块时，可以使用rmmod命令进行卸载，此进内核会调用模块中的函数`cleanup_module()`。任何时候都可以使用命令来lsmod查看目前已经加载的模块以及正在使用该模块的用户数。
## 4. 设备驱动程序结构
了解设备驱动程序的基本结构（或者称为框架），对开发人员而言是非常重要的，Linux的设备驱动程序大致可以分为如下几个部分：驱动程序的注册与注销、设备的打开与释放、设备的读写操作、设备的控制操作、设备的中断和轮询处理。
### 驱动程序的注册与注销
向系统增加一个驱动程序意味着要赋予它一个主设备号，这可以通过在驱动程序的初始化过程中调用`register_chrdev()`或者`register_blkdev()`来完成。而在关闭字符设备或者块设备时，则需要通过调用`unregister_chrdev()`或`unregister_blkdev()`从内核中注销设备，同时释放占用的主设备号。
### 设备的打开与释放
打开设备是通过调用file_operations结构中的函数open( )来完成的，它是驱动程序用来为今后的操作完成初始化准备工作的。在大部分驱动程序中，open( )通常需要完成下列工作：
1. 检查设备相关错误，如设备尚未准备好等。
2. 如果是第一次打开，则初始化硬件设备。
3. 识别次设备号，如果有必要则更新读写操作的当前位置指针f_ops。
4. 分配和填写要放在file->private_data里的数据结构。
5.使用计数增1。

释放设备是通过调用file_operations结构中的函数release( )来完成的，这个设备方法有时也被称为close( )，它的作用正好与open( )相反，通常要完成下列工作：
1. 使用计数减1。
2. 释放在file->private_data中分配的内存。
3. 如果使用计算为0，则关闭设备。

### 设备的读写操作
字符设备的读写操作相对比较简单，直接使用函数read( )和write( )就可以了。但如果是块设备的话，则需要调用函数block_read( )和block_write( )来进行数据读写，这两个函数将向设备请求表中增加读写请求，以便Linux内核可以对请求顺序进行优化。由于是对内存缓冲区而不是直接对设备进行操作的，因此能很大程度上加快读写速度。如果内存缓冲区中没有所要读入的数据，或者需要执行写操作将数据写入设备，那么就要执行真正的数据传输，这是通过调用数据结构blk_dev_struct中的函数request_fn( )来完成的。
### 设备的控制操作
除了读写操作外，应用程序有时还需要对设备进行控制，这可以通过设备驱动程序中的函数ioctl( )来完成。ioctl( )的用法与具体设备密切关联，因此需要根据设备的实际情况进行具体分析。
### 设备的中断和轮询处理
对于不支持中断的硬件设备，读写时需要轮流查询设备状态，以便决定是否继续进行数据传输。如果设备支持中断，则可以按中断方式进行操作。