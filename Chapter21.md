### Chapter21.

#### 2.异常终止进程：abort()

函数abort() 终止其调用进程，并生成核心转储文件（core文件）。

```c++
#include <stdlib.h>

void abort(void);
```

函数abort()通过产生SIGABRT 信号来终止调用进程。对SIGABRT 的默认动作是产生核心转储文件，并终止进程。调试器（gdb）可以利用核心转储文件来检测调用 abort() 时的程序状态。



#### 3.在备选栈中处理信号：sigaltstack()

在调用信号处理器函数时，内核通常会在进程栈中为其创建一帧。不过，如果进程对栈的扩展突破了对栈大小的限制时，这种做法就不大可行了。例如，栈的增长过大，以至于会触及到一片映射内存或者向上增长的堆，又或者栈的大小已经直逼RLIMIT_STACK资源限制，这些都会造成这种的情况的发生。

当进程对栈的扩展试图突破其上限时，内核将为该进程产生SIGSEGV信号。不过，因为栈空间已然耗尽，内核也就无法味精厂已经安装的SIGSEGV处理器函数创建栈帧。结果是，处理器函数得不到调用，而进程也就终止了。

为此，我们需要调用 sigaltstack() 分配一块被称为“备选信号栈”的内存区域，作为信号处理器函数的栈帧。

```c++
#include <signal.h>

int sigaltstack(const stack_t *sigstack, stack_t *old_sigstack);

typedef struct {
  void **ss_sp;  // 起始地址
  int ss_flags;  // Flags
  size_t ss_size;  // 栈的大小
} stack_t;
```



#### 4.系统调用的中断和重启

考虑如下情景：

1. 为某信号创建处理器函数；
2. 发起一个阻塞的系统调用（blocking system call），例如，从终端设备调用的read()就会阻塞到有数据输入为止。
3. 当系统调用遭到阻塞时，之前创建了处理器函数的信号传递了过来，随即引发对处理器函数的调用。

信号处理器返回后又会发生什么？默认情况下，系统调用失败，并将errno置为EINTR。



不过，更为常见的情况是希望遭到中断的系统调用得以继续运行。为此，可自己利用代码进行手工重启：

```c++
while ((cnt = read(fd, buf, BUF_SIZE)) == -1 && errno == EINTR)
  continue;

if (cnt == -1)
  errExit("read");
```

也可以修改信号的**SA_RESTART**标志。

函数siginterrupt()用于改变信号的 **SA_RESTART**设置。