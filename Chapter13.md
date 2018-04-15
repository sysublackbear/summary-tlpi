### Chapter13.文件I/O缓冲

#### 1.文件I/O的内核缓冲：缓冲区高速缓存

read()和write()系统调用在操作磁盘文件时不会直接发起磁盘访问，而是仅仅在用户空间缓冲区与内核缓冲区高速缓存之间复制数据。例如：如下调用将3个字节的数据从用户空间内存传递到内核空间的缓冲区中。

```c++
write(fd, "abc", 3);
```

write()随即返回。在后续某个时刻，内核会将其缓冲区中的数据写入（刷新至）磁盘。（因此，可以说系统调用与磁盘操作并不同步。）如果在此期间，另一进程试图读取该文件的这几个字节，那么内核将自动从缓冲区高速缓存中提供这些数据，而不是从文件中（读取过期的内容）。

与此同理，对输入而言，内核从磁盘中读取数据并存储到内核缓冲区中。read()调用将从该缓冲区中读取数据，直至把缓冲区中的数据取完，这时，内核会将文件的下一段内容读入缓冲区高速缓存。

采用这一设计，意在使read()和write()调用的操作更为快速，因为它们不需要等待磁盘操作。同时，这一设计也极为高效，因为这减少了内核必须执行的磁盘传输次数。

内核会分配尽可能多的缓冲区高速缓存页，受限于两个因素：**可用的物理内存总量**，以及**出于其他目的对物理内存的需求**。



#### 2.stdio库的缓冲

当操作磁盘文件时，缓冲大块数据以减少系统调用，C语言函数库的I/O函数（比如，fprintf()，fscanf()，fgets()，fputs()，fgetc()）正是这么做的。因此，使用stdio库可以使编程者免于自行处理对数据的缓冲，无论是调用write()来输出，还是调用read()来输入。



**设置一个stdio流的缓冲模式**

```c++
#include <stdio.h>

int setvbuf(FILE * stream, char * buf, int mode, size_t size);

void setbuf(FILE * stream, char * buf);

void setbuffer(FILE * stream, char * buf, size_t size);
```

**刷新stdio缓冲区**

```c++
#include <stdio.h>

int fflush(FILE * stream);
```



#### 3.同步I/O数据完整性和同步I/O文件完整性

**同步I/O数据完整性(synchronized I/O data integrity completion)**：某一I/O操作，要么已成功完成到磁盘的数据传递，要么被诊断为不成功。数据完整性是指确保针对文件的每一次更新操作都传递了足够的信息落到了磁盘，以便于之后对数据的获取。

**同步I/O文件完整性(synchronized I/O file integrity completion)**：同步I/O数据完整性的超集，要将所有发生更新的文件元数据都传递到磁盘上，即使有些在后续对文件数据的读操作中并不需要（比如文件的大小信息，mtime等信息）。



#### 4.强制把缓冲区数据刷新到磁盘

文件完整性：

```c++
#include <unistd.h>

int fsync(int fd);
```

数据完整性：

```c++
#include <unistd.h>

int fdatasync(int fd);
```

sync()系统调用会使包含更新文件信息的所有内核缓冲区（即数据块，指针块，元数据等）刷新到磁盘上。

```c++
#include <unistd.h>

void sync(void);
```

使所有写入同步：O_SYNC

调用open()函数时如指定O_SYNC标志，则会使所有后续输出同步。

```c++
fd = open(pathname, O_WRONLY | O_SYNC);
```

调用open()后，每个write()调用会自动将文件数据和元数据刷新到磁盘上（即，按照同步I/O文件完整性的要求执行写操作）。

O_SYNC分O_DSYNC和O_RSYNC。



#### 5.绕过缓冲区高速缓存：直接I/O

open里面加上O_DIRECT。



#### 6.混合使用库函数和系统调用进行文件I/O

```c++
#include <stdio.h>

int fileno(FILE * stream);

FILE *fdopen(int fd, const char *mode);
```













