# Linux 系统编程

系统编程三大基石：系统调用 (从用户空间向内核发起的函数调用)、C 库、C 编译器。

## 1 文件IO

### 1.1 open( ) 和 create( )
通过 open( ) 系统调用来打开一个文件获得一个文件描述符。
```cpp
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>

int open(const char *name, int flags);
int open(const char *name, int flags, mode_t mode);

/* O_WRONLY|O_CREAT|O_TRUNC 组合经常被使用，creat() 用来实现 */
int creat(const char *name, mode_t mode);
```
flags 参数必须是 O_RDONLY, O_WRONLY 或 O_RDWR 其中之一。
mode 参数提供新建文件的权限。

open( ) 和 creat( ) 调用成功时返回一个文件描述符，错误时都返回 -1。

### 1.2 read( )
```cpp
#include <unistd.h>

ssize_t read(int fd, void *buf, size_t len);
```
该系统调用从由 fd 指向的文件的当前偏移量至多读 len 个字节到 buf 中。成功时，将返回写入 buf 中的字节数。出错时返回 -1。

### 1.3 write( )
```cpp
#include <unistd.h>

ssize_t write(int fd, const void *buf, size_t count);
```
该系统调用从文件描述符 fd 引用文件的当前位置开始，将 buf 中之多 count 个字节写入文件中。成功时，返回写入字节数，并更新文件位置。错误时，返回 -1。

### 1.4 同步 I/O
**fsync( )**
```cpp
#include <unistd.h>

int fsync(int fd);
```
调用 fsync( ) 可以保证(阻塞) fd 对应文件的脏数据回写到磁盘上。成功时，返回 0，失败时，返回 -1。

**fdatasync( )**
```cpp
#include <unistd.h>

int sdatasync(int fd);
```
该系统调用完成的事情和 fsync( ) 一样，区别在于它仅写入数据。成功时，返回 0，失败时，返回 -1。

**sync( )**
```cpp
#include <unistd.h>

void sync(void);
```
sync( )系统调用可以用来对磁盘上的所有缓冲区进行同步。 

**区别：**
>+ fsync( ) 将所有修改过的数据，包括用户写出的数据以及文件本身的特征数据，如文件的访问时间、修改时间、文件的属主等回写到磁盘。
>+ fdatasync( ) 只强制传送用户已写出的数据至物理存储设备，不包括文件本身的特征数据。这样可以适当减少文件刷新时的数据传送量。
>+ sync( ) 确保所有的缓冲区--包括数据和元数据--都能写入磁盘。

**O_SYNC 标志：**
O_SYNC 标志在 open( ) 中使用，使所有在文件上的 I/O 操作同步。

**tips：**
>+ 一般情况下，需要确保数据写入磁盘的应用可以使用 fsync( ) 或者  fdatasync( )。因为需要较少的调用(比如，在某个决定性的操作之后)，相对于 O_SYNC 来讲，开销也更少。

### 1.5 直接 I/O
在 open( ) 中使用 O_DIRECT 标志会使内核最小化 I/O 管理的影响。

使用该标志时，I/O 操作将忽略页缓存机制，直接对用户空间缓冲区和设备进行初始化。所有的 I/O 将是同步的；操作在完成之前不会返回。

**优点：** 直接 I/O 最主要的优点就是通过减少操作系统内核缓冲区和应用程序地址空间的数据拷贝次数，降低了对文件读取和写入时所带来的 CPU 的使用以及内存带宽的占用。这对于某些特殊的应用程序，比如自缓存应用程序来说，不失为一种好的选择。如果要传输的数据量很大，使用直接 I/O 的方式进行数据传输，而不需要操作系统内核地址空间拷贝数据操作的参与，这将会大大提高性能。

**缺点：** 设置直接 I/O 的开销非常大，而直接 I/O 又不能提供缓存 I/O 的优势。缓存 I/O 的读操作可以从高速缓冲存储器中获取数据，而直接 I/O 的读数据操作会造成磁盘的同步读，这会带来性能上的差异 , 并且导致进程需要较长的时间才能执行完；对于写数据操作来说，使用直接 I/O 需要 write() 系统调用同步执行，否则应用程序将会不知道什么时候才能够再次使用它的 I/O 缓冲区。与直接 I/O 读操作类似的是，直接 I/O 写操作也会导致应用程序关闭缓慢。所以，应用程序使用直接 I/O 进行数据传输的时候通常会和使用异步 I/O 结合使用。

### 1.6 close( )
程序完成对某个文件的操作后，可以使用 close( ) 系统调用将文件描述符和对应的文件解除关联。

```cpp
#include <unistd.h>

int close(int fd);
```

成功时返回 0，失败时，返回 -1。

### 1.7 lseek( )
lseek( ) 系统调用能够对给定文件描述符引用的文件位置设置指定值。
```cpp
#include <sys/types.h>
#include <unistd.h>

off_t lseek(int fd, off_t pos, int origin);
/*
origin 参数：
    SEEK_CUR: 初始位置为当前位置，设置后位置为: 当前位置 + pos
    SEEK_END: 初始位置为文件末尾，设置后位置为: 文件末尾 + pos
    SEEK_SET：初始位置为文件起始，设置后位置为: pos
*/
```
lseek( ) 最常见的用法是来定位当前文件的开始和末尾，或确定某个文件描述符引用的当前文件位置。

### 1.8 定位读写 pread( ) 和 pwrite( )
Linux 提供了两种 read( ) 和 write( ) 的变体来替代 lseek( )，每个调用都以需要读写的文件的位置为参数。完成时，不修改文件位置。
```cpp
#define _XOPEN_SOURCE 500
#include <unistd.h>

ssize_t pread(int fd, void *buf, size_t count, off_t pos);
ssize_t pwrite(int fd, const void *buf, size_t count, off_t pos);
````
pread( ) 从文件描述符 fd 的 pos 文件位置读取 count 个字节到 buf 中。

pwrite( ) 从文件描述符 fd 的 pos 文件位置写 count 个字节到 buf 中。

### 1.9 截短文件
```cpp
#include <unistd.h>
#include <sys/types.h>

int ftruncate(int fd, off_t len);
int truncate(const char *path, off_t len);
```
两个系统调用都将文件截短到指定的长度。

这两个操作都不可修改当前文件位置。

### 1.10 I/O 多路复用
I/O 多路复用允许应用在多个文件描述符上同时阻塞，并在其中某个可以读写时收到通知。

I/O 多路复用的设计遵循以下原则：
>+ 1.I/O 多路复用：当任何文件描述符准备好 I/O 时告诉我。
>+ 2.在一个或者更多文件描述符就绪前始终处于睡眠状态。
>+ 3.唤醒：哪个准备好了？
>+ 4.在不阻塞的情况下处理所有 I/O 就绪的文件描述符。
>+ 5.返回第一步，重新开始。

#### 1.10.1 select( )
```cpp
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>

int select(int n,             /* 所有文件描述符的最大值加一 */
           fd_set *readfds,   /* 确认是否有可读数据 */
           fd_set *writefds,  /* 确认是否有写操作可不阻塞完成 */
           fd_set *exceptfds, /* 确认是否有异常发生或带外数据 */
           struct timeval *timeout);

FD_CLR(int fd, fd_set *set);
FD_ISSET(int fd, fd_set *set);
FD_SET(int fd, fd_set *set);
FD_ZERO(fd_set *set);

struct timeval
{
    long tv_sec;  /* seconds */
    long tv_usec; /* microseconds */
};
```
在指定的文件描述符准备好 I/O 之前或者超过一定的时间限制，select( ) 调用就会阻塞。

成功时，返回在所有三个集合中 I/O 就绪的文件描述符的数目。如果超出了时限，返回值可能为 0.错误时返回 -1。

```cpp
#define _XOPEN_SOURCE 600
#include <sys/select.h>

int pselect(int n,
            fd_set *readfds,
            fd_set *writefds,
            fd_set *exceptfds,
            const struct timespec *timeout,
            const sigset_t *sigmask);
```
增加 sigmask 参数，以此来解决信号和等待文件描述符之间的竞争条件。

pselect( ) 和 select( ) 有三点不同：
>+ 1.pselect( ) 的 timeout 参数使用了 timespec 结构，而不是 timeval 结构。timespec 使用秒和纳秒，而不是秒和毫秒，从理论上来讲更精确一些。
>+ 2.pselect( ) 调用并不修改 timeout 参数。这个参数在后续调用时也不需要重新初始化。
>+ 3.select( ) 调用没有 sigmask 参数。当这个参数设置为零时，pselect( ) 的行为等同于 select( )。

#### 1.10.2 poll( )
poll( ) 使用一个简单的 nfds 个 pollfd 结构体构成的数组，fds 指向该数组。

```cpp
#include <sys/poll.h>

struct pollfd
{
    int fd;        /* file descriptor */
    short events;  /* requested events to watch */
    short revents; /* returned events witnessed */
};

int poll(struct pollfd *fds, unsigned int nfds, int timeout);
```

每个结构体的 events 字段是要监视的文件描述符事件的一组位掩码，用户设置这个字段。

revents 字段则是发生在该文件描述符上的事件的位掩码，内核在返回时设置这个字段。

成功时，返回具有非零 revents 字段的文件描述符个数。超时前没有任何事件发生则返回 0。失败时返回 -1。

```cpp
#define _GNU_SOURCE
#include <sys/poll.h>

int ppoll(struct pollfd *fds,
          nfds_t nfds,
          const struct timespec *timeout,
          const sigset_t *sigmask);
```
像 pselect( ) 那样，timeout 参数一秒和纳秒指定了时限， 而 sigmask 参数提供了一组等待处理的信号。

#### 1.10.3 epoll
poll( ) 和 select( ) 每次调用时都需要所有被监听的文件描述符，内核必须遍历所有被监视的文件描述符。

epoll 把监听注册从时机监听中分离出来，从而解决了这个问题。一个系统调用初始化一个 epoll 上下文，另一个从上下文中加入或者删除需要监视的文件描述符，第三个执行真正的事件等待。

```cpp
#include <sys/epoll.h>

int epoll_create(int size);
```
调用成功，epoll_create( ) 创建一个 epoll 实例，返回与该实例关联的文件描述符。size 参数为内核需要监听的文件描述符数目。出错时，返回 -1。

epoll_ctl( ) 可以向指定的 epoll 上下文中加入或者删除文件描述符。
```cpp
struct epoll_event
{
    __u32 events; /* events */
    union
    {
        void *ptr;
        int fd;
        __u32 u32;
        __u64 u64;
    } data;
};

#include <sys/epoll.h>
int epoll_ctl(int epfd,
              int op,
              int fd,
              struct epoll_event *event);

/* 
参数 op 的有效值:
    EPOLL_CTL_ADD: 把 fd 指定的文件添加到 epfd 指定的 epoll 实例监听集中，监听 event 中定义的事件。
    EPOLL_CTL_DEL: 把 fd 指定的文件从 epfd 指定的 epoll 监听集中删除。
    EPOLL_CTL_MOD: 使用 event 改变已有 fd 上的监听行为。
*/
```
data 字段由用户使用。确认监听事件后，data会返回给用户。通常将 event.data.fd 为设定 fd，这样就可以知道哪个文件描述符触发事件。

成功时，返回 0，失败时，返回 -1。

epoll_wait( ) 等待给定 epoll 实例关联的文件描述符上的事件。
```cpp
#include <sys/epoll.h>

int epoll_wait(int epfd,
               struct epoll_event *events,
               int maxevents,
               int timeout);
```
对 epoll_wait( ) 的调用等待 epoll 实例 epfd 中的文件 fd 上的事件，时限为 timeout 毫秒。

成功返回时，events 指向包含 epoll_event 结构体（该结构体描述了每个事件）的内存，且最多可以有 maxevents 个事件。返回值是时间数。出错时返回 -1。

如果 timeout 为 0，即使没有事件发生，调用也会立即返回，此时调用返回 0。如果 timeout 为 -1，调用将一直等待到有事件发生。

**高级文件IO：**
>+ **散布/聚集IO**。IO允许在单次调用中同时对多个缓冲区做读取或者写入操作，适合于聚集多个不同数据结构进行统一的IO操作。
>+ **epoll**。poll( ) 和 select( ) 的改进版，在一个程序需要处理数百个文件描述符的时候很有用。
>+ **内存映射IO**。将文件映射到内存，可以通过简单的内存管理方式来处理文件，适合有特殊要求的IO。
>+ **文件IO提示**。允许进程将文件IO使用上的一些信息提供给内核，能提升IO性能。
>+ **异步IO**。允许进程发出多个IO请求而且不必等待其完成，适用于不使用线程的情况下处理重负载IO操作。

### 1.11 散布/聚集IO
```cpp
#include <sys/uio.h>

/* 每个 iovec 结构体描述一个独立的缓冲区，称其为 段 */
struct iovec
{
    void *iov_base;
    size_t iov_len;
};

ssize_t readv(int fd, const struct iovec *iov, int count);
ssize_t writev(int fd, const struct iovec *iov, int count);
```
成功时，readv( ) 和 writev( ) 分别返回读写的字节数。返回值等于所有 iov_len 之和。出错时，返回 -1。

### 1.12 存储映射
将文件映射到内存中，即内存和文件中数据是一一对应的。

#### 1.12.1 mmap( )
mmap( )调用请求内核将 fd 表示的文件从 offset 处开始的 len 个字节数据映射到内存中。
```cpp
#include <sys/mman.h>

void* mmap(void *addr,   /* 内核映射文件的最佳地址 */
           size_t len,   /* 映射区长度 */
           int port,     /* 内存区域的访问权限 */
           int flags,    /* 指定映射对象的类型 */
           int fd,       /* 文件描述符 */
           off_t offset  /* 被映射对象内容的起点 */
           );

/*
port:
    PORT_NONE 
    PORT_READ
    PORT_WRITE
    PORT_EXEC

flags:
    MAP_SIXED 把addr看作强制性要求，而不是建议
    MAP_PRIVATE 映射区不共享
    MAP_SHARED 和其他所有映射该文件的进程共享映射内存
*/
```
调用成功，mmap( ) 返回映射区的地址，失败时，返回 MAP_FAILED。

页是内存映射的基本块，同时也是进程地址空间的基本块。mmap( ) 调用操作页。addr 和 offset 参数都必须按页大小对齐，必须是页大小的整数倍。映射区域是整数倍个页。

#### 1.12.2 munmap( )
Linux 提供了 munmap( ) 来取消 mmap( ) 的映射。
```cpp
#include <sys/mman.h>

int munmap(void *addr, size_t len);
```
munmap( )移除进程地址空间从 addr 开始，len 字节长的内存中的所有页面的映射。
成功时，返回 0，失败时，返回 -1。

**优点：**
>+ 使用 read( ) 或 write( ) 系统调用需要从用户缓冲区进行数据读写，而使用映射文件进行操作，可以避免多余的数据拷贝。
>+ 除了潜在的页错误，读写映射文件不会带来系统调用和上下文切换的开销。就像直接操作内存一样简单。
>+ 当多个进程映射同一个对象到内存中，数据在进程间共享。
>+ 在映射对象中搜索只需要一般的指针操作。

**缺陷：**
>+ 映射区域的大小通常是页大小的整数倍。
>+ 存储映射区域必须在进程地址空间内。
>+ 创建和维护映射以及相关的内核数据结构有一定的开销。

基于以上理由，处理大文件（浪费的空间只占很小的比重），或者在文件大小恰好被 page 大小整除时（没有空间浪费）优势很明显。

#### 1.12.3 调整映射的大小
Linux 提供了 mremap( ) 来扩大或减小已有映射的大小。
```cpp
#define _GNU_SOURCE
#include <unistd.h>
#include <sys/mman.h>

void* mremap(void *addr,
             size_t old_size,
             size_t new_size,
             unsigned long flags
             );
/*
prot：PROT_NONE, PROT_READ, PROT_WRITE, PROT_EXEC。
*/
```
mremap( ) 将映射区域 [addr, addr + old_szie) 的大小增加或减小到 new_size。依赖进程地址空间的可用大小和 flags，内核可以同时移动映射区域。

调用成功，返回指向新映射区域的指针，失败时，返回 MAP_FAILED。

#### 1.12.4 改变映射区域的权限
```cpp
#include <sys/mman.h>

int mprotect(const void *addr, size_t len, int prot);
```
调用 mprotect( ) 会改变 [addr, addr + len) 区域内页的访问权限，addr 是页对齐的。

prot：PROT_NONE, PROT_READ, PROT_WRITE, PROT_EXEC。

调用成功，返回 0，调用失败，返回 -1。

#### 1.12.5 使用映射机制同步文件
```cpp
#include <sys/mman.h>

int msync(void *addr, size_t len, int flags);

/*
flags:
    MS_ASYNC: 指定同步操作异步发生。
    MS_INVALIDATE: 指定该块映射的其它所有拷贝都将失效。
    MS_SYNC: 指定同步操作必须同步进行。
```
调用 msync( ) 可以将 mmap( ) 生成的映射在内存中的任何修改回写到磁盘，达到同步内存中的映射和被映射的文件的目的。

成功时，返回 0，失败时，返回 -1。

#### 1.12.6 映射提示
Linux 提供了 madvise( ) 系统调用，可以让进程在如何访问映射区域上给内核一定的提示。调用会指示内核如何对起始地址为 addr，长度为 len 的内存映射区域进行操作。
```cpp
#include <sys/mman.h>

int madvise(void *addr, size_t len, int advice);

/*
advice:
    MADV_NORMAL: 按正常方式操作
    MADV_RANDOM: 应用程序将以随即顺序访问指定范围的页
    MADV_SEQUENTIAL: 应用程序意图从低地址到高地址顺序访问指定范围的页
    MADV_WILLNEED: 应用程序将很快访问指定范围的页
    MADV_DONTNEED: 应用程序短期内不会访问指定范围的页
*/
```
成功时，返回 0，失败时，返回 -1。

### 1.13 普通文件IO提示
#### 1.13.1 posix_fadvise( )
```cpp
#include <fcntl.h>

int posix_fadvise(int fd,
                  off_t offset,
                  off_t len,
                  int advice
                  );

/*
advice:
    POSIXFADV_NORMAL: 按正常方式操作
    POSIXFADV_RANDOM: 应用程序将以随即顺序访问指定范围的页
    POSIXFADV_SEQUENTIAL: 应用程序意图从低地址到高地址顺序访问指定范围的页
    POSIXFADV_WILLNEED: 应用程序将很快访问指定范围的页
    POSIXFADV_NOREUSE: 应用程序可能在最近访问给定范围，但只访问一次
    POSIXFADV_DONTNEED: 应用程序短期内不会访问指定范围的页
*/
```
调用会给出内核在文件 fd 的 [offset, offset + len) 范围内操作提示。一般用法是设置 len 和 offset 为 0，从而使设置应用到整个文件。

#### 1.13.2 readahead( )
```cpp
#include <fcntl.h>

ssize_t readahead(int fd, off64_t offset, size_t count);
```
readahead( )调用将读入 fd 表示的文件的 [offset, offset + count) 区域到页缓存中。
成功时，返回 0，失败时，返回 -1。

### 1.14 异步IO
```cpp
#include <aio.h>

/* asynchronous I/O control block */
struct aiocb
{
    int aio_filedes;  /* file descriptor */
    int aio_lio_opcode;  /* operation to perform */
    int aio_reqprio;  /* request priority offset */
    volatile void *aio_buf;  /* pointer to buffer */
    size_t aio_nbytes;  /* length of operation */
    struct sigevent aio_sigevent;  /* signal number ans value */
    /* internal, private members follow ... */
};

int aio_read(struct aiocb *aiocbp);
int aio_write(struct aiocb *aiocbp);
int aio_error(const struct aiocb *aiocbp);
int aio_return(struct aiocb *aiocbp);
int aio_cancel(int fd, struct aiocb *aiocbp);
int aio_fsync(int op, struct aiocb *aiocbp);
int aio_suspend(const struct aiocb *const cblist[],
                int n,
                const struct timespec * timeout
                );
```

### 1.15 IO调度

I/O 调度器实现两个基本操作：合并和排序。合并操作是将两个或多个相邻的 I/O 请求的过程合并为一个。排序是选取两个操作中相对更重要的一个，并按块号递增的顺序重新安排等待的 I/O 请求。按照这种方式，硬盘头的移动距离最小。

## 2 标准IO
### 2.1 打开文件
```cpp
#include <stdio.h>

FILE* fopen(const char *path, const char *mode);
FILE* fdopen(int fd, const char *mode);

/*
mode:
    r: 打开文件用来读取。
    r+: 打开文件用来读写。
    w: 打开文件用来写入，若文件存在，则会被清空，若文件不存在，则会被创建。
    w+: 打开文件用来读写，若文件存在，则会被清空，若文件不存在，则会被创建。
    a: 追加模式写入。
    a+: 追加模式读写。
*/
```

### 2.2 关闭流
关闭一个给定的流。
```cpp
#include <stdio.h>

int fclose(FILE *stream);
```
关闭所有的流。
```cpp
#define _GNU_SOURCE
#include <stdio.h>

int fcloseall(void);
```

### 2.3 从流中读取数据
#### 2.3.1 单字节读取
每次读取一个字符。
```cpp
#include <stdio.h>

int fgetc(FILE *stream);
```
从流中读取下一个字符并把该无符号字符强转为 int 返回。

#### 2.3.2 把字符回放入流中
```cpp
#include <stdio.h>

int ungetc(int c, FILE *stream);
```
每次调用把 c 强转成一个无符号字符并放回流中。

#### 2.3.3 按行读取
```cpp
#include <stdio.h>

char* fgets(char *str, int size, FILE *stream);
```
从流中读取 size - 1 个字节的数据，并把数据存入 str 中。

#### 2.3.4 读取二进制文件
```cpp
#include <stdio.h>

size_t fread(void *buf, size_t size, size_t nr, FILE *stream);
```
从输入流中读取 nr 个数据，每个数据有 size 个字节，并将数据放入到 buf 所指向的缓冲区。读入元素的个数被返回。

### 2.4 向流中写数据
#### 2.4.1 写入单个字符
```cpp
#include <stdio.h>

int fputc(int c, FILE *stream);
```
将 c 表示的字节强转为一个无符号字符，写入 stream 指向的流中。成功时，返回 c，失败时，返回 EOF。

#### 2.4.2 写入字符串
```cpp
#include <stdio.h>

int fputs(const char *str, FILE *stream);
```
成功时，返回一个非负整数，失败时，返回 EOF。

#### 2.4.3 写入二进制数据
```cpp
#include <stdio.h>

size_t fwrite(void *buf, size_t size, size_t nr, FILE *stream);
```
将 buf 指向的 nr 个元素写入到 stream 中，每个元素长度为 size。成功时，返回写入元素个数。

### 2.5 定位流
```cpp
#include <stdio.h>

int fseek(FILE *stream, long offset, int whence);

/*
whence:
    SEEK_SET: 文件位置被设置到 offset 处。
    SEEK_CUR: 文件位置被设置到 当前位置 + offset 处。
    SEEK_END: 文件位置被设置到 文件末尾 + offset 处。
*/
```
成功时，返回 0，失败时，返回 -1。

```cpp
#include <stdio.h>

int fsetpos(FILE *stream, fpos_t *pos);
```
fsetpos( ) 将流的位置设置到 pos 处。

```cpp
#include <stdio.h>

void rewind(FILE *stream);
```
rewind( ) 将位置重置到流的初始位置。

```cpp
#include <stdio.h>

long ftell(FILE *stream);
```
ftell( ) 返回当前流的位置。

### 2.6 清洗一个流
```cpp
#include <stdio.h>

int fflush(FILE *stream);
```
fflush( ) 将 stream 指向的流中所有未写入的数据会被清洗到内核中。成功时，返回 0，失败时，返回 -1。
fflush( ) 只是把用户缓冲的数据写入到内核缓冲区。

### 2.7 错误和文件结束
函数 ferror( ) 测试是否在流上设置了错误标志。
```cpp
#include <stdio.h>

int ferror(FILE *stream);
```
如果错误标志被设置，函数返回非零值，否则返回 0。

函数 feof( ) 测试文件结尾标志是否被设置。
```cpp
#include <stdio.h>

int feof(FILE *stream);
```
如果结尾标志被设置，函数返回非零值，否则返回 0。

函数 clearerr( ) 为流清空错误和文件结尾标志。
```cpp
#include <stdio.h>

void clearerr(FILE *stream);
```

### 2.8 获得关联的文件描述符
```cpp
#include <stdio.h>

int fileno(FILE *stream);
```
fileno( ) 返回和流相关联的文件描述符，失败时，返回 -1。

### 2.9 控制缓冲
```cpp
#include <stdio.h>

int setvbuf(FILE *stream, char *buf, int mode, size_t size);
/*
mode:
    _IONBF: 无缓冲
    _IOLBF: 行缓冲
    _IOFBF: 块缓冲

size: buf 指向一个 size 大小的缓冲空间，若 buf 为空，缓冲区则由 glibc 自动分配。
*/
```
成功时，返回 0，失败时，返回非零值。

## 3 文件与目录管理
### 3.1 文件及其元数据
每个文件均对应一个 inode，是文件系统中唯一数值编址。inode 存储了与文件有关的元数据，例如文件的访问权限，最后的访问时间，所有者，群组，大小以及文件数据的存储位置。

#### 3.1.1 stat 函数
Unix 提供了一组获取文件元数据的函数。
```cpp
#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>

struct stat
{
    dev_t st_dev;  /* ID of device containing file  设备节点*/
    ino_t st_ino;  /* inode number */
    mode_t st_mode;  /* permissions */
    nlink_t st_nlink;  /* number of hard links */
    uid_t st_uid;  /* user ID of owner */
    gid_t st_gid;  /* group ID of owner */
    dev_t st_rdev;  /* device ID (if special file) */
    off_t st_size;  /* total size in bytes */
    blksize_t st_blksize;  /* blocksize for filesystem I/O */
    blkcnt_t st_blocks;  /* number of blocks allocated */
    time_t st_atime;  /* last access time */
    time_t st_mtime;  /* last modification time */
    time_t st_ctime;  /* last status change time */
};

int stat(const char *path, struct stat *buf);
int fstat(int fd, struct stat *buf);
int lstat(const char *path, struct stat *buf);
```
stat( ) 返回由路径 path 参数指明的文件信息，fstat( ) 返回由文件描述符 fd 指向的文件信息。对于符号链接，lstat( ) 返回链接本身，而非目标文件。成功时，均返回 0，失败时，返回 -1。

#### 3.1.2 权限
chmod( ) 和 fchmod( ) 均可设置文件权限为 mode。
```cpp
#include <sys/types.h>
#include <sys/stat.h>

int chmod(const char *path, mode_t mode);
int fchmod(int fd, mode_t mode);
```
成功时，均返回 0，失败时，返回 -1。

#### 3.1.3 所有权
chown( ) 和 lchown( ) 设置有路径 path 指定的文件的所有权。
```cpp
#include <sys/types.h>
#include <unistd.h>

int chown(const char *path, uid_t owner, gid_t group);
int lchown(const char *path, uid_t owner, gid_t group);
int fchown(int fd, uid_t owner, gid_t group);
```
在符号链接情况下，lchown( ) 更改符号链接本身的所有者，而不是该符号链接所指向的文件的所有者。如果 owner 或 group 为 -1，说明值没有设定。
成功时，均返回 0，失败时，返回 -1。

### 3.2 目录
#### 3.2.1 获取当前工作目录
获取当前工作目录
```cpp
#include <unistd.h>

char* getcwd(char *buf, size_t size);
```
成功调用 getcwd( ) 会以绝对路径名形式复制当前工作目录至由 buf 指向的长度 size 字节的缓冲区，并返回一个指向 buf 的指针。

```cpp
#define _GNU_SOURCE
#include <unistd.h>

char* get_current_dir_name(void);
```
也返回当前工作目录。

#### 3.2.2 更改当前工作目录
```cpp
#include <unistd.h>

int chdir(const char *path);
int fchdir(int fd);
```
调用 chdir( ) 会更改当前工作目录为 path 指定的路径名，绝对或相对路径均可以。调用 fchdir( ) 会更改当前工作目录为文件描述符 fd 指向的路径名，而 fd 必须是打开的目录。
成功时，均返回 0，失败时，返回 -1。
这些系统调用只对当前进程有影响。

#### 3.2.3 创建目录
```cpp
#include <sys/types.h>

int mkdir(const char *path, mode_t mode);
```
成功调用 mkdir( ) 会创建目录 path（可能是相对或绝对路径），其权限位为 mode，并返回 0。失败时，返回 -1。

#### 3.2.4 移除目录
```cpp
#include <unistd.h>

int rmdir(const char *path);
```
调用成功时，rmdir( ) 从文件系统移除 path，并返回 0。失败时，返回 -1。

#### 3.2.5 读取目录内容
开始读取目录内容前，需要创建一个由 DIR 对象指向的目录流：
```cpp
#include <sys/types.h>
#include <dirent.h>

DIR* opendir(const char *name);
```
成功调用 opendir( ) 会创建 name 所指向目录的目录流。

从目录流获取文件描述符：
```cpp
#define _BSD_SOURCE  /* or _SVID_SOURCE */
#include <sys/types.h>
#include <dirent.h>

int dirfd(DIR *dir);
```
成功调用 dirfd( ) 返回目录流 dir 的文件描述符。

从目录流读取。使用 opendir( ) 创建一个目录流后，程序可以从目录中读取目录项。
```cpp
#include <sys/types.h>
#include <dirent.h>

struct dirent
{
    ino_t d_ino;  /* inode number */
    off_t d_off;  /* offset to the next dirent */
    unsigned short d_reclen;  /* length of this record */
    unsigned char d_type;  /* type of file */
    char d_name[256];  /* filename */
};

struct dirent* readdir(DIR *dir);
```
应用程序连续调用 readdir( )，获取目录下的每个文件，直至发现所需文件，或者直到整个目录已读完，此时返回 NULL。失败时，返回 NULL。

关闭目录流：
```cpp
#include <sys/types.h>
#include <dirent.h>

int closedir(DIR *dir);
```
成功调用 closedir( ) 会关闭由 dir 指向的目录流，包括目录的文件描述符，并返回 0。失败时，返回 -1。

### 3.3 链接
目录中每个名字至 inode 的映射被称为**链接**。

#### 3.3.1 硬链接
```cpp
#include <unistd.h>

int link(const char *oldpath, const char *newpath);
```
成功调用 link( ) 会为已存在的文件 oldpath 建立路径 newpath 下的新链接，并返回 0。失败时，返回 -1。
结束之后，oldpath 和 newpath 均指向同一文件。

#### 3.3.2 符号链接
符号链接，也是熟知的 symlinks 或软链接。与硬链接的相同处在于二者均指向文件系统中的文件，符号链接的不同点在于它不增加额外的目录项，而是一种特殊的文件类型。软链接与硬链接的一点很重要的区别是它可以跨越文件系统。
```cpp
#include <unistd.h>

int symlink(const char *oldpath, const char *newpath);
```
成功调用 symlink( ) 会创建指向目标 oldpath 的符号链接 newpath，并返回 0。失败时，返回 -1。

#### 3.3.3 解除链接
```cpp
#include <unistd.h>

int unlink(const char *pathname);
```
成功调用 unlink( ) 从文件系统删除 pathname，并返回 0。如果该路径是指向文件的最后一个链接，会从文件系统删除该文件。如果 pathname 指向符号链接，则只删除链接，而不删除目标文件。

为了简化对各种类型文件的删除，C 语言提供了函数 remove( ):
```cpp
#include <stdio.h>

int remove(const char *path);
```
成功调用 remove( ) 会从文件系统删除 path，并返回 0。如果 path 是个文件，remove( ) 调用 unlink( )；如果 path 是个目录，remove( ) 调用 rmdir( )。

### 3.4 复制和移动文件

#### 3.4.1 复制
下面是复制文件 src 至命名为 dst 的文件的步骤：
>+ 1.打开 src。
>+ 2.打开 dst，如果不存在则创建，如果存在则长度截断为 0。
>+ 3.将 src 数据读至内存。
>+ 4.将该数据块写入 dst。
>+ 5.继续操作，直至 src 全部已读取且已写入 dst。
>+ 6.关闭 dst。
>+ 7.关闭 src。

#### 3.4.2 移动
```cpp
#include <stdio.h>

int rename(const char *oldpath, const char *newpath);
```
成功调用 rename( ) 会将路径名 oldpath 重命名为 newpath。文件内容和 inode 保持不变。成功时，返回 0，失败时，返回 -1。

### 3.5 设备节点
设备节点是应用程序与设备驱动交互的特殊文件。

**空设备**位于/dev/null，主设备号是1，次设备号是3。该设备文件的所有者是root但所有用户均可读写。内核会忽略所有对该设备的写请求。所有对该文件的读请求则返回文件终止符（EOF）。

**零设备**位于/dev/zero，主设备号为1，次设备号为5。与空设备一样，内核忽略所有对零设备的写请求。读取该设备会返回无限null字节流。

**满设备**位于/dev/full，主设备号为1，次设备号为7。与零设备一样，读请求返回null字符（'\0'）。写请求却总触发ENOSPC错误，它表明设备已满。


### 3.6 带外通信
```cpp
#include <sys/ioctl.h>

int ioctl (int fd, int request, ...);

/*
fd: 文件的文件描述符。
request: 特殊请求代码，该值由内核和进程预定义，它指明对文件 fd 执行何种操作。
*/
```

### 3.7 监视文件事件
Linux提供监视文件的接口 inotify --利用它可以监控文件的移动，读取，写入，或删除操作。
通过 inotify，内核能在事件发生时通知应用程序。

#### 3.7.1 初始化 inotify
系统调用 inotify_init( ) 用来初始化 inotify 并返回初始化实例指向的文件描述符。
```cpp
#include <inotify.h>

int inotify_init(void);
```
错误时，返回 -1。

#### 3.7.2 监视
inotify 可以监视文件和目录。当监视目录时，inotify 会报告目录本身和该目录下所有文件（不包括监视目录子目录下的文件）的事件。

增加新监视。系统调用 inotify_add_watch( ) 在文件或者目录 path 上增加一个监视，监视事件由 mask 确定，监视实例由 fd 指定。
```cpp
#include <inotify.h>

int inotify_add_watch(int fd,
                      const char *path,
                      uint32_t mask
                      );

/*
mask:
    IN_ACCESS: 文件读取
    IN_MODIFY: 文件写入
    IN_ATTRIB: 文件元数据已改变
    IN_CLOSE_WRITE: 文件关闭且曾以写入模式打开
    IN_CLOSE_NOWRITE: 文件关闭且未曾以写入模式打开
    IN_OPEN: 文件已打开
    IN_MOVED_FROM: 文件从监视目录中移出
    IN_MOVED_TO: 文件已移入监视目录
    IN_CREATE: 文件已在监视目录创建
    IN_DELETE: 文件已从监视目录删除
    IN_DELETE_SELF: 监视对象本身已删除
    IN_MOVE_SELF: 监视对象本身已移动

    IN_ALL_EVENTS: 所有合法的事件
    IN_CLOSE: 所有涉及关闭的事件
    IN_MOVE: 所有涉及移动的事件
*/
```
成功时，调用返回新建的监视描述符。失败时，返回 -1。

#### 3.7.3 inotify 事件
使用结构 inotify_event 来描述 inotify 事件：
```cpp
#include <inotify.h>

struct inotify_event
{
    int wd;  /* watch descriptor */
    uint32_t mask;  /* mask of events */
    uint32_t cookie;  /* unique cookie */
    uint32_t len;  /* size of 'name' field */
    char name[];  /* null-terminated name */
};

/*
高级 inotify 事件：
    IN_IGNORED: wd 指向的监视描述符已移除
    IN_ISDIR: 作用对象是目录
    IN_Q_OVERFLOW: inotify 队列溢出
    IN_UNMOUNT: 监视对象所在的设备未挂载
*/
```

#### 3.7.4 高级监视选项
```cpp
/*
IN_DONT_FOLLOW: 
IN_MASK_ADD:
IN_ONESHOT:
IN_ONLYDIR:
*/
```

#### 3.7.5 删除 inotify 监视
```cpp
#include <inotify.h>

int inotify_rm_watch(int fd, uint32_t wd);
```
成功调用 inotify_rm_watch( ) 会从 inotify 实例（由文件描述符指向的） fd 中移除由监视描述符 wd 指向的监视，并返回 0。失败时，返回 -1。

#### 3.7.6 获取事件队列大小
```cpp
unsigned int queue_len;
int ret;
ret = ioctl(fd, FIONREAD, &queue_len);
if(ret < 0)
    perror("ioctl");
else
    printf("%u bytes pending in queue\n", queue_len);
```
返回的是队列的字节大小，而非队列的事件数。

#### 3.7.7 销毁 inotify 实例
```cpp
int ret;
/* 'fd' was obtained via inotify_init( ) */
ret = close(fd);
if(fd == -1)
    perror("close");
```

## 4 内存管理

### 4.1 进程地址空间
Linux将物理内存虚拟化。进程不能直接在物理内存上寻址，而在虚拟地址空间。虚拟空间由许多页组成。

内核将具有某些相同特征的页组织成块。这些块叫做存储器区域，段，或者映射。下面是一些在每个进程都可以见到的存储器区域：
>+ 文本段。包含一个进程的代码，字符串，常量和一些只读的数据。文本段被标记为只读，并且直接从目标文件映射到内存中。
>+ 堆栈段。包括一个进程的执行栈。执行栈中包括了程序的局部变量和函数的返回值。
>+ 数据段。又叫堆，包含着一个进程的动态存储空间。这部分空间往往由 malloc 分配。
>+ BSS段。包含了没有被初始化的全局变量。

### 4.2 动态内存分配
动态内存是在进程运行时才分配的，而不是在编译时就分配好了的，分配的大小也只有在分配时才确定。
```cpp
#include <stdlib.h>

void* malloc(size_t size);
```
成功调用后会得到一个 size 大小的内存区域，并返回一个指向这部分内存首地址的指针。该内存区域内容未定义。失败时，返回 NULL。

数组分配。数组元素大小已经确定，但元素个数却是变化的。C 提供 calloc( ) 函数。
```cpp
#include <stdlib.h>

void* calloc(size_t nr, size_t size);
```
calloc 将分配的区域全部用 0 进行初始化。失败时，返回 NULL。

调整已分配内存大小。
```cpp
#include <stdlib.h>

void* realloc(void *ptr, size_t size);
```
成功调用将会使 ptr 指向的内存区域的大小变为 size 字节。返回一个指向新空间的指针。如果 size 是 0，效果就会跟在 ptr 上调用 free( ) 相同。如果 ptr 是 NULL，结果就会跟 malloc( ) 一样。失败时，返回 -1。

动态内存的释放。
```cpp
#include <stdlib.h>

void free(void *ptr);
```
调用 free( ) 会释放 ptr 指向的内存。

对齐。数据的对齐是指数据地址和硬件确定的内存块之间的关系。

### 4.3 匿名存储器映射
对于较大的分配，glibc 并不使用堆二十创建一个匿名内存映射。一个匿名内存映射只是一块已经用 0 初始化的大的内存块，以供用户使用。因为这种映射的存储不是基于堆的，所以并不会在数据段内产生碎片。

glibc 的 malloc 使用数据段来满足效地分配，而匿名内存映射则用来满足大的分配。

### 4.4 高级存储器分配
```cpp
#include <malloc.h>

int mallopt(int param, int value);
```
调用 mallopt( ) 会将由 param 确定的存储管理相关的参数设为 value。成功时，返回非零值，失败时，返回 0。

```cpp
#include <malloc.h>

size_t malloc_usable_size(void *ptr);
```
调用成功时，返回 ptr 指向的动态内存块的实际大小。

强制 glibc 归还所有可释放的动态内存给内核：
```cpp
#include <malloc.h>

int malloc_trim(size_t padding);
```
调用成功时，数据段会尽可能地收缩，但是填充字节被保留下来，然后返回 1。失败时，返回 0。

### 4.5 基于栈的分配
```cpp
#include <alloca.h>

void* alloca(size_t size);
```
调用 alloca( )，成功时会返回一个指向 size 字节大小的内存指针。这块内存是在栈中的，当调用它的函数返回时，这块内存将被自动释放。

alloca( ) 和变长数组的主要区别在于通过前者获得的内存在函数执行过程中始终存在，而通过后者获得的内存在除了作用域后便释放了。

### 4.6 存储器操作

#### 4.6.1 字节设置
```cpp
#include <string.h>

void* memset(void *s, int c, size_t n);
```
调用 memset( ) 将把从 s 指向区域开始的 n 个字节设置为 c，并返回 s。

#### 4.6.2 字节比较
比较两块内存是否相等
```cpp
#include <string.h>

int memcmp(const void *s1, const void *s2, size_t n);
```
调用 memcmp( ) 比较 s1 和 s2 的头 n 个字节，如果两块内存相同就返回 0，如果 s1 小于 s2 就返回一个小于 0 的数，反之则返回大于 0 的数。

#### 4.6.3 字节移动
```cpp
#include <string.h>

void* memmove(void *dst, const void *src, size_t n);
```
memmove( ) 复制 src 的前 n 字节到 dst，返回 dst。

```cpp
#include <string.c>

/* dst 和 src 之间不能重叠 */
void* memcpy(void *dst, const void *src, size_t n);

/* 发现字节 c 在 src 指向的前 n 个字节中时会停止拷贝 */
void* memccpy(void *dst, const void *src, int c, size_t n);
```

#### 4.6.4 字节搜索
```cpp
#include <string.h>

void* memchr(const void *s, int c, size_t n);

void* memrchr(const void *s, int c, size_t n);
```
函数 memchr( ) 从 s 指向的区域的 n 个字节中寻找 c，c 将被转换为 unsigned char。
memrchr( ) 反向搜索。

#### 4.6.5 字节加密
```cpp
#define _GNU_SOURCE
#include <string.h>

void* memfrob(void *s, size_t n);
```
memfrob( ) 函数将 s 指向的位置开始的 n 个字节，每个都与 42 进行异或操作来对数据进行加密。再次调用，可以将其转换回来。

### 4.7 内存锁定

#### 4.7.1 锁定部分地址空间
```cpp
#include <sys/mman.h>

int mlock(const void *addr, size_t len);
```
mlock( ) 将锁定 addr 开始长度为 len 个字节的虚拟内存。成功时，返回 0，失败时，返回 -1。
一个由 fork( ) 产生的子进程并不从父进程处继承锁定的内存。

#### 4.7.2 锁定全部地址空间
```cpp
#include <sys/mman.h>

int mlockall(int flags);

/*
flags:
    MCL_CURRENT: 将所有已被映射的页面锁定在进程地址空间中
    MCL_FUTURE: 将所有未来映射的页面也锁定在进程地址空间中
*/
```
mlockall( ) 函数锁定一个进程在现有地址空间在物理内存中的所有界面。

#### 4.7.3 内存解锁
```cpp
#include <sys/mman.h>

int munlock(const void *addr, size_t len);
int munlock(void);
```
调用成功时都返回 0，失败时，返回 -1。

## 5 进程管理

### 5.1 进程ID
每一个进程都有一个唯一的标识符表示，即进程 ID，简称 pid。

空闲进程 pid 为 0，init 进程 pid 为 1。

每个进程都被一个用户和组所拥有。每个子进程都继承了父进程的用户和组。

每个进程都是某个进程组的一部分。

获取进程 ID 和父进程 ID
```cpp
#include <sys/types.h>
#include <unistd.h>

pid_t getpid(void);
pid_t getppid(void);
```

### 5.2 运行新进程

#### 5.2.1 exec
execl( ) 会将 path 所指路径的映像载入内存，替换当前进程的映像。
```cpp
#include <unistd.h>

int execl(const char *path, const char *arg, ...);
```
成功的调用会以跳到新的程序的入口点作为结束，而刚刚才被运行的代码是不会存在于进程的地址空间中的。pid、父进程 pid、优先级、所属的用户和组不改变。

其他系统调用：
```cpp
#include <unistd.h>

int execlp(const char *file, const char *arg, ...);
int execle(const char *path, const char *arg, ..., char *const envp[]);
int execv(const char *path, char *const argv[]);
int execvp(const char *file, char *const argv[]);
int execve(const char *filename, char *const argv[], char *const envp[]);
```

#### 5.2.2 fork( )
创建一个和当前进程映像一样的进程
```cpp
#include <sys/types.h>
#include <unistd.h>

pid_t fork(void);
```
在子进程中，成功的 fork( ) 调用会返回 0，在父进程中 fork( ) 返回子进程的 pid。

Linux采用写时复制的方法，而不是对父进程空间进行整体复制。如果一个进程要修改自己的那份资源，那么就会复制那份资源，并把复制的那份提供给进程。

```cpp
#include <sys/types.h>
#include <unistd.h>

pid_t vfork(void);
```
vfork( ) 会挂起父进程直到子进程终止或者运行了一个新的可执行文件的映像。vfork( ) 避免了地址空间的按页复制。父进程和子进程共享相同的地址空间和页表项。

vfork( ) 只完成了一件事：复制内部的内核数据结构。因此，子进程也就不能修改地址空间中的任何内存。

### 5.3 终止进程
```cpp
#include <stdlib.h>

void exit(int status);
```
exit( ) 调用通常会执行一些基本的终止进程的步骤，然后通知内核终止这个进程。

atexit( ) 用来注册一些在进程结束时要调用的函数。
```cpp
#include <stdlib.h>

int atexit(void(*function)(void));
```
atexit( ) 成功调用会把指定的函数注册到进程正常结束时调用的函数中。

当一个子进程终止时，内核会向其父进程发送 SIGCHILD 信号。

### 5.4 等待终止的子进程
```cpp
#include <sys/types.h>
#include <sys/wait.h>

pid_t wait(int *status);
```
wait( ) 返回已终止子进程的 pid，或者返回 -1表示出错。如果没有子进程终止，调用会被阻塞，直到一个子进程终止。如果有子进程终止了，它就会立刻返回。

如果知道需要等待进程的 pid，可以使用 waitpid( ) 系统调用：
```cpp
#include <sys/types.h>
#include <sys/wait.h>

pid_t waitpid(pid_t pid, int *status, int options);

/*
pid:
    <-1: 等待所有组 ID 是 pid 绝对值的子进程
    -1: 等待任一子进程，和 wait( ) 等效
    0: 等待与调用进程处于同一进程组的任一进程
    >0: 等待进程 pid 为传入值的那个子进程

options:
    WNOHANG: 不阻塞，立刻返回
    WUNTRACED: 即使调用进程没有跟踪子进程，WIFSTOPPED 也一样被设置
    WCONTINUED: 即使调用进程没有跟踪子进程，WIFCONTINUED 也一样被设置
*/
```
参数 pid 是需要等待的一个或多个进程的 pid。

Linux提供了 waitid( )。
```cpp
#include <sys/wait.h>

int waitid(idtype_t idtype,
           id_t id,
           siginfo_t *infop,
           int options
           );

/*
idtype:
    P_PID: 等待 pid 值是 id 的子进程
    P_GID: 等待进程组 ID 是 id 那些子进程
    P_ALL: 等待所有子进程，参数 id 被忽略
*/
```

如果一个进程创建了新进程然后立刻开始等待它的结束，那么使用下面的函数就很合适：
```cpp
#define _XOPEN_SOURCE
#include <stdlib.h>

int system(const char *command);
```
由参数 command 指定的程序会得到执行。

### 5.5 用户和组
改变实际用户（组）ID 和保存设置的用户（组）ID。
```cpp
#include <sys/types.h>
#include <unistd.h>

int setuid(uid_t uid);
int setgid(gid_t gid);
```
setuid( ) 用来设定当前进程的有效用户 ID。成功时，返回 0，失败时，返回 -1。
setgid( ) 用来设定当前进程的有效用户 ID。成功时，返回 0，失败时，返回 -1。

改变有效用户和组 ID。
```cpp
#include <sys/types.h>
#include <unistd.h>

int seteuid(uid_t euid);
int setegid(gid_t egid);
```
seteuid( ) 将有效用户 ID 的值设置为 euid。成功时，返回 0，失败时，返回 -1。
setegid( ) 将有效用户 ID 的值设置为 euid。成功时，返回 0，失败时，返回 -1。

非 root 用户应该使用 seteuid( ) 来设置有效用户 ID。如果有 root 权限的进程希望改变三种用户 ID，那么应该使用 setuid( )；而只是想临时改变有效用户 ID，那么最好使用 seteuid( )。

获取用户和组ID
```cpp
#include <unistd.h>
#include <sys/types.h>

uid_t getuid(void);
gid_t getgid(void);

uid_t geteuid(void);
gid_t getegid(void);
```

### 5.6 会话和进程组
每个进程都属于某个进程组。每个进程组都由进程组 ID（pgid）唯一标识，并且有一个组长进程。进程组 ID 就是组长进程的 pid。

一个会话就是一个或多个进程组的集合。

创建一个新的会话：
```cpp
#include <unistd.h>

pid_t setsid(void);
```
setsig( ) 创建新会话，并在其中创建一个新的进程组，而且使得调用进程成为新会话的首进程和新进程组的组长进程。

setpgid( ) 将 pid 进程的进程组 ID 设置为 pgid。
```cpp
#define _XOPEN_SOURCE 500
#include <unistd.h>

int setpgid(pid_t pid, pid_t pgid);
```
如果 pid 是 0，则使用调用者的进程 ID。如果 pgid 是 0，则将 pid 进程的进程 ID 设置为进程组 ID。

### 5.7 进程调度
就绪进程：非阻塞，且有部分时间片可用。内核用一个就绪队列维护所有的就绪进程，一旦某进程耗光它的时间片，内核就将其移出队列，直到所有就绪进程都耗光时间片才考虑将其放回队列。

多任务操作系统分为两大类：协同式和抢占式。Linux为抢占式。

Linux通过动态分配进程时间片，兼顾吞吐率（长时间片）与交互性能（短时间片）。

偏向于计算的进程称为“处理器约束进程”（倾向于长时间片），偏向于IO的进程称为“IO约束进程”（倾向于短时间片）。

**所有的进程都必须运行**。

不会有就绪却没有运行的较高优先级进程，系统中的运行进程一定是最高优先级的可运行进程。

### 5.8 让出处理器
调用 sched_yield( ) 函数将终端当前进程，运行一个新进程，就和内核主动抢占进程一样。
```cpp
#include <sched.h>

int sched_yield(void);
```
调用成功返回 0，失败返回 -1。

### 5.9 进程优先级
“nice value”在程序运行的时候指定，Linux调度器基于这样的原则来调度：高优先级的程序总是先运行。

合法的优先级在 -20 到 19 之间，默认为 0。nice 值越低，优先级越高，时间片越长。

获取和设置进程 nice 值的系统调用。
```cpp
#include <unistd.h>

int nice(int inc);
```
成功调用 nice( ) 将在现有优先级上增加 inc，并返回新值。

```cpp
#include <sys/time.h>
#include <sys/resource.h>

int getpriority(int which, int who);
int setpriority(int which, int who, int prio);

/*
which:
    PRIO_PROCESS: 
    PRIO_PGRP: 
    PRIO_USER: 

who:
    进程 ID
    进程组 ID
    用户 ID
*/
```

### 5.10 处理器亲和度
进程调度器应该尽量让进程停留在一个处理器。
处理器亲和度表明一个进程停留在同一个处理器上的可能性。只有当负载极端不平衡的时候，才考虑迁移进程。
进程从父进程继承处理器亲和度。

### 5.11 实时系统
实时系统分为软硬实时系统两大类。硬实时系统对于操作期限非常严格，超过期限就会产生失败。软实时系统不认为超过期限是一个严重的失败。
软硬实时系统的区分也不等同于操作时限的长短。

每一个进程都有一个与 nice 值无关的静态优先级，对于普通程序，值为 0，对于实时程序，值为 1 到 99。

### 5.12 调度策略
**先进先出策略**。
**轮转策略**。
**普通调度策略**。
**批调度策略**。

### 5.13 资源限制
Linux提供了两个操作资源限制的系统调用。
```cpp
#include <sys/time.h>
#include <sys/resource.h>

struct rlimit
{
    rlim_t rlim_cur;  /* soft limit */
    rlim_t rlim_max;  /* hard limit */
};

int getrlimit(int resource, struct rlimit *rlim);
int setrlimit(int resource, const struct rlimit *rlim);
```
## 6 线程
线程共享资源:
>+ 1.文件描述符表
>+ 2.每种信号的处理方式
>+ 3.当前工作目录
>+ 4.用户ID和组ID
>+ 5.内存地址空间 (.text/.data/.bss/heap/共享库)

线程非共享资源:
>+ 1.线程id
>+ 2.处理器现场和栈指针(内核栈)
>+ 3.独立的栈空间(用户空间栈)
>+ 4.errno变量
>+ 5.信号屏蔽字
>+ 6.调度优先级

线程优、缺点:
>+ 优点： 1. 提高程序并发性 2. 开销小 3. 数据通信、共享数据方便
>+ 缺点： 1. 库函数，不稳定 2. 调试、编写困难、gdb不支持 3. 对信号支持不好
>+ 优点相对突出，缺点均不是硬伤。Linux下由于实现方法导致进程、线程差别不是很大。

### 6.1 pthread_self
**pthread_self**函数  获取线程ID。pthread_self函数  获取线程ID。其作用对应进程中 getpid() 函数。线程ID是进程内部，识别标志。(两个进程间，线程ID允许相同)
```cpp
#include <pthread.h>

pthread_t pthread_self(void);
```

### 6.2 pthread_create
**pthread_create**函数  创建一个新线程。 其作用，对应进程中 fork() 函数。
```cpp
#include <pthread.h>

int pthread_create(pthread_t *tidp,
                   const pthread_attr_t *attr,
                   void *(*start_rtn)(void*),
                   void *arg
                   );
/*
参数:
    第一个参数为指向线程标识符的指针。
    第二个参数用来设置线程属性。
    第三个参数是线程运行函数的起始地址。
    最后一个参数是运行函数的参数。
*/

typedef struct
{
    int detachstate;     线程的分离状态
    int schedpolicy;     线程调度策略
    struct sched_param schedparam;  线程的调度参数
    int inheritsched;    线程的继承性
    int scope;           线程的作用域
    size_t guardsize;    线程栈末尾的警戒缓冲区大小
    int    stackaddr_set;
    void* stackaddr;     线程栈的位置
    size_t stacksize;    线程栈的大小
}pthread_attr_t;
```
若线程创建成功，则返回0。若线程创建失败，则返回出错编号。

返回成功时，由tidp指向的内存单元被设置为新创建线程的线程ID。attr参数用于指定各种不同的线程属性。新创建的线程从start_rtn函数的地址开始运行，该函数只有一个万能指针参数arg，如果需要向start_rtn函数传递的参数不止一个，那么需要把这些参数放到一个结构中，然后把这个结构的地址作为arg的参数传入。

### 6.3 pthread_exit
**pthread_exit**函数  将单个线程退出。
```cpp
#include <pthread.h>

void pthread_exit(void* retval);

/* 参数：retval表示线程退出状态，通常传NULL */
```
多线程环境中，应尽量少用，或者不使用exit函数，取而代之使用pthread_exit函数，将单个线程退出。任何线程里exit导致进程退出，其他线程未工作结束，主控线程退出时不能return或exit。

另注意，pthread_exit或者return返回的指针所指向的内存单元必须是全局的或者是用malloc分配的，不能在线程函数的栈上分配，因为当其它线程得到这个返回指针时线程函数已经退出了。

### 6.4 pthread_join
**pthread_join**函数  阻塞等待线程退出，获取线程退出状态 其作用，对应进程中 waitpid() 函数。
```cpp
#include <pthread.h>

int pthread_join(pthread_t thread, void **retval);

/* 参数：thread：线程ID  retval：存储线程结束状态。 */
```
以阻塞的方式等待thread指定的线程结束。当函数返回时，被等待线程的资源被收回。如果线程已经结束，那么该函数会立即返回。并且thread指定的线程必须是joinable的。

### 6.5 pthread_cancel
**pthread_cancel**函数  杀死(取消)线程 其作用，对应进程中 kill() 函数。
```cpp
#include <pthread.h>

int pthread_cancel(pthread_t thread);
```
线程的取消并不是实时的，而有一定的延时。需要等待线程到达某个取消点(检查点)。

### 6.6 pthread_detach
**pthread_detach**函数  实现线程分离。
```cpp
#include <pthread.h>

int pthread_detach(pthread_t thread); 
```
线程分离状态：指定该状态，线程主动与主控线程断开关系。线程结束后，其退出状态不由其他线程获取，而直接自己自动释放。网络、多线程服务器常用。

在线程被分离后，不能使用pthread_join等待它的终止状态。

## 7 信号
信号是提供处理异步事件机制的软件中断。

### 7.1 信号概念
内核会暂停该进程正在执行的代码，并跳转到先前注册过的函数。接下来进程会执行这个函数。一旦进程从这个函数返回，它会条回到捕获信号的地方继续执行。

### 7.2 基本信号管理
signal( )
```cpp
#include <signal.h>

typedef void(*sighandler_t)(int);

sighandler_t signal(int signo, sighandler_t handler);
```
调用 signal( ) 会移除接收 signo 信号的当前操作，并以 handler 指定的新信号处理程序代替它。

pause( ) 可以使进程睡眠，直到进程接受到处理或终止进程的信号。
```cpp
#include <unistd.h>

int pause(void);
```
pause( ) 只在接收到可捕获的信号时返回。

在新进程中任何父进程捕获的信号都被重置为默认操作。
当进程调用 fork( ) 时，子进程继承完全同样的信号语义。

### 7.3 发送信号
kill( ) 从一个进程向另一个进程发送信号。
```cpp
#include <sys/types.h>
#include <signal.h>

int kill(pid_t pid, int signo);
```
kill( ) 给 pid 代表的进程发送信号 signo。

raise( ) 是一种进程给自己发送信号的方法。
```cpp
#include <signal.h>

int raise(int signo);
```

给整个进程组发送信号。
```cpp
#include <signal.h>

int killpg(int pgrp, int signo);
```

### 7.4 重入
**可重入函数**是指可以安全的调用自身（或者从同一个进程的另一个线程）的函数。为了使函数可重入，函数决不能操作静态数据，必须只操作栈分配的数据或者调用者提供的数据，不得调用任何不可重入的函数。

信号处理程序必须只使用可重入函数。

### 7.5 信号集
信号集合操作用来管理信号集。
```cpp
#include <signal.h>

int sigemptyset(sigset_t *set);
int sigfillset(sigset_t *set);
int sigaddset(sigset_t *set, int signo);
int sigdelset(sigset_t *set, int signo);
int sigismember(const sigset_t *set, int signo);
```

Linux提供了一些非标准的函数：
```cpp
#define _GNU_SOURCE
#include <signal.h>

int sigisemptyset(sigset_t *set);
int sigorset(sigset_t *dest, sigset_t *left, sigset_t *right);
int sigandset(sigset_t *dest, sigset_t *left, sigset_t *right);
```

###  7.6 阻塞信号
任何被挂起的信号都不会被处理，直到它们被解除阻塞。进程可以阻塞任意多个信号，被进程阻塞的信号叫做该进程的信号掩码。

Linux实现了一个管理进程信号掩码的函数：
```cpp
#include <signal.h>

int sigprocmask(int how,
                const sigset_t *set,
                sigset_t *oldset);

/*
how:
    SIG_SETMASK: 调用进程的信号掩码变成 set
    SIG_BLOCK: set 中的信号被加入到调用进程的信号掩码中
    SIG_UNBLOCK: set 中的信号被从调用进程的信号掩码中移除
*/
```
如果 oldset 是非空的，该函数将 oldset 设置为先前的信号集。如果 set 是空的，该函数会忽略 how，并且不改变信号掩码，但仍然设置 oldset 的信号掩码。成功时，返回 0，失败时，返回 -1。

当内核产生一个被阻塞的信号时，该信号未被发送，被叫做待处理信号。当一个待处理信号解除阻塞时，内核会把它发送给进程处理。

检测待处理信号集的函数：
```cpp
#include <signal.h>

int sigpending(sigset_t *set);
```
sigpending( ) 会将 set 设置为待处理的信号集，返回 0，失败时，返回 -1。

该函数始终处于等待状态，直到产生一个终止该进程或被该进程处理的信号：
```cpp
#include <signal.h>

int sigsuspend(const sigset_t *set);
```
如果一个信号终止了进程，sigsuspend( ) 不返回。如果一个信号被发送和处理了，sigsuspend( ) 在信号处理程序返回后，返回 -1。

### 7.7 高级信号管理
sigaction( ) 系统调用，提供了更加强大的信号管理能力。除此之外，当信号处理程序运行时，可以用它来阻塞特定信号的接收，也可以用它来获取信号发送时各种操作系统和进程状态的信息：
```cpp
#include <signal.h>

int sigaction(int signo,
              const struct sigaction *act,
              struct sigaction *oldact);

struct sigaction
{
    void (*sa_handler)(int);  /* 信号处理程序或操作 */
    void (*sa_sigaction)(int, siginfo_t*, void*);
    sigset_t sa_mask;  /* 阻塞的信号 */
    int sa_flags;  /* 标志 */
    void (*sa_restorer)(void);
};
```

### 7.8 发送带附加信息的信号
```cpp
#include <signal.h>

union sigval
{
    int sival_int;
    void* *sival_ptr;
};

int sigqueue(pid_t pid,
             int signo,
             const union sigval value);

union sigval
{
    int sigval_int;
    void *sigval_ptr;
};
```
成功时，由 signo 表示的信号被加入到由 pid 指定的进程或进程组的队列中，并返回 0。失败时，返回 -1。

## 8 Linux 网络编程

**字节序转换**
```cpp
#include <netinet/in.h>

unsigned long int htonl(unsigned long int hostlong);
unsigned short int htons(unsigned short int hostshort);
unsigned long int ntohl(unsigned long int netlong);
unsigned short int ntohs(unsigned short int nethort);
```

**socket 地址**
```cpp
#include <bits/socket.h>

struct sockaddr
{
    sa_family_t sa_family;
    char sa_data[14];
}
```

**IP 地址转换函数**
```cpp
#include <arpa/inet.h>

/* IPv4 */
in_addr_t inet_addr(const char* strptr);
int inet_aton(const char* cp, struct in_addr* inp);
char* inet_ntoa(struct in_addr in);

/* IPv4 && IPv6 */
int inet_pton(int af, const char* src, void* dst);
const char* inet_ntop(int af, const void* src, char* dst, socklen_t cnt);
```

**创建 socket**
```cpp
#include <sys/types.h>
#include <sys/socket.h>

int socket(int domain, int type, int protocol);
```

**绑定 socket**
```cpp
#include <sys/types.h>
#include <sys/socket.h>

int bind(int sockfd,
         const struct sockaddr* my_addr,
         socklen_t addrlen);
```

**监听 socket**
```cpp
#include <sys/socket.h>

int listen(int sockfd, int backlog);
```

**接受连接**
```cpp
#include <sys/types.h>
#include <sys/socket.h>

int accept(int sockfd,
           struct sockaddr *addr,
           socklen_t *addrlen);
```

**发起连接**
```cpp
#include <sys/types.h>
#include <sys/socket.h>

int connect(int sockfd,
            const struct sockaddr *serv_addr,
            socklen_t addrlen);
```

**关闭连接**
```cpp
#include <unistd.h>

int close(int fd);
```
```cpp
#include <sys/socket.h>

int shutdown(int sockfd, int howto);
```

**TCP 数据读写**
```cpp
#include <sys/types.h>
#include <sys/socket.h>

ssize_t recv(int sockfd, void *buf, size_t len, int flags);
ssize_t send(int sockfd, const void *buf, size_t len, int flags);
```

**UDP 数据读写**
```cpp
#include <sys/types.h>
#include <sys/socket.h>

ssize_t recvfrom(int sockfd,
                 void *buf,
                 size_t len,
                 int flags,
                 struct sockaddr* src_addr,
                 socklen_t *addrlen);

ssize_t sendto(int sockfd,
               const void *buf,
               size_t len,
               int flags,
               const struct sockaddr *dest_addr,
               socklen_t addrlen);
```

**通用数据读写函数**
```cpp
#include <sys/socket.h>

ssize_t recvmsg(int sockfd, struct msghdr* msg, int flags);
ssize_t sendmsg(int sockfd, struct msghdr* msg, int flags);

struct msghdr
{
    void* msg_name;           /* socket 地址 */
    socklen_t msg_namelen;    /* socket 地址长度 */
    struct iovec* msg_iov;    /* 分散的内存块 */
    int msg_iovlen;           /* 分散内存块的数量 */
    void* msg_control;        /* 指向辅助数据的起始位置 */
    socklen_t msg_controllen; /* 辅助数据的大小 */
    int msg_flags;            /* 复制函数中的flags参数，并在调用过程中更新 */
};

struct iovec
{
    void* iov_base;  /* 内存起始地址 */
    size_t iov_len;  /* 内存长度 */
}
```

**带外标记**
```cpp
#include <sys/socket.h>

int sockatmark(int sockfd);

/*
sockfd 处于带外标记，返回 1；
否则，返回 0.
*/
```

**获取地址信息**
```cpp
#include <sys/socket.h>

int getsockname(int sockfd,
                struct sockaddr* address,
                socklen_t* address_len);

int getpeername(int sockfd,
                struct sockaddr* address,
                socklen_t* address_len);
```

**socket 选项**
```cpp
#include <sys/socket.h>

int getsockopt(int sockfd,
               int level,
               int option_name,
               void* option_value,
               socklen_t* restrict option_len);

int setsockopt(int sockfd,
               int level,
               int option_name,
               const void* option_value,
               socklen_t option_len);
```

**获取主机信息**
```cpp
#include <netdb.h>

struct hostent* gethostbyname(const char* name);
struct hostent* gethostbyaddr(const void* addr, size_t len, int type);

struct hostent
{
    char* h_name;        /* 主机名 */
    char** h_aliases;    /* 主机别名列表 */
    int h_addrtype;      /* 地址类型（地址族） */
    int h_length;        /* 地址长度 */
    char** h_addr_list;  /* 主机 IP 地址列表 */
};
```

**获取服务信息**
```cpp
#include <netdb.h>

struct servent* getservbyname(const char* name, const char* proto);
struct servent* getservbyport(int port, const char* proto);

struct servent
{
    char* s_name;      /* 服务名称 */
    char** s_aliases;  /* 服务的别名列表 */
    int s_port;        /* 端口号 */
    char* s_proto;     /* 服务类型，通常是 tcp 或者 udp */
};
```