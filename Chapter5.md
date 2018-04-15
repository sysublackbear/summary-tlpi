### Chapter5.深入深究文件I/O

#### 1.文件控制操作：fcntl()

```c++
int flags, accessMode;
flags = fcntl(fd, F_GETFL);
if (flags == -1)
{
  errExit("fcntl");
}

if (flags & O_SYNC)
{
  printf("writes are synchronized.\n");
}
```

#### 2.复制文件描述符

```c++
#include <unistd.h>

int dup(int oldfd);

int dup2(int oldfd, int newfd);
```

#### 3.在文件特定偏移量处的I/O：pread() 和 pwrite()

相当于lseek和read(或write)合为原子性操作

#### 4.分散输入和集中输出

```c++
#include <sys/uio.h>

ssize_t readv(int fd, const struct iovec *iov, int iovcnt);

ssize_t writev(int fd, const struct iovec *iov, int iovcnt);

struct iovec {
  void *iov_base;
  size_t iov_len;
};
```

还有类似的preadv和pwritev



#### 5.截断文件：truncate()和ftruncate()系统调用

```c++
#include <unistd.h>

int truncate(const char * pathname, off_t length);
int ftruncate(int fd, off_t length);
```



#### 6.非阻塞I/O

在打开文件时指定O_NONBLOCK 标志，目的有二：

+ 若 open() 调用未能立即打开文件，则返回错误，而非陷入阻塞。有一种情况属于例外，调用open() 操作FIFO可能陷入阻塞；
+ 调用open() 成功后，后续的I/O操作也是非阻塞的。若I/O系统调用未能立即完成，则可能会只传输部分数据，或者系统调用失败，并返回EAGAIN和EWOULDBLOCK 错误。具体何种错误将依赖于系统调用。



#### 7./dev/fd目录

对于每个进程，内核都提供有一个特殊的虚拟目录/dev/fd，该目录中包含"/dev/fd/n"刑式的文件名，其中n是与进程中的打开文件描述符相对应的编号。因此，例如，/dev/fd/0就对应于进程的标准输入。



#### 8.创建临时文件

```c++
#include <stdlib.h>

int mkstemp(char * template);
```

```c++
#include <stdio.h>

FILE * tmpfile(void);
```

