### Chapter29.线程：介绍

#### 1.概述

一个进程可以包含多个线程。同一程序中的所有线程均会独立执行相同程序，且共享同一份全局内存区域，其中包括初始化数据段，为初始化数据段，以及堆内存段。

同一进程中的多个线程可以并发执行。在多处理器环境下，多个线程可以同时并行。如果一线程因等待I/O操作而遭阻塞，那么其他线程依然可以继续运行。

对于某些应用而言，线程要优于进程。以网络服务器的设计为例，服务器进程（父进程）在接受客户端的连接后，会调用fork()来创建一个单独的子进程，以处理与客户端的通信。采用这种设计，服务器就能同时为多个客户端提供服务。但进程模型也会存在以下弊端：

- 进程间的信息难以共享。由于除去只读代码段外，父子进程并未共享内存，因此必须采用一些进程间通信（IPC）的方式，在进程间进行信息交换。
- 调用fork()来创建进程的代价相对较高。即便利用写时复制技术，仍然需要复制诸如内存页表和文件描述符之类的多种进程属性，这意味着fork()调用在时间上的开销依然不菲。

线程解决了上述两个问题。

- 线程之间能够方便，快速地共享信息。只需将数据复制到共享（全局或堆）变量中即可。不过，要避免出现多个线程试图同时修改同一份信息的情况。
- 创建线程比创建进程通常要快10倍甚至更多。（在Linux中，是通过系统调用clone()来实现线程的）。线程的创建之所以快，是因为调用fork()创建子进程时所需复制的诸多属性，在线程间本来就是共享的。特别是，既无需采用写时复制来复制内存也，也无需复制页表。

在Linux平台上，在编译调用了Pthreads API 的程序时，需要设置 cc -pthread 的编译选项。



#### 2.创建线程

启动程序时，产生的进程只有单条线程，称之为初始或主线程。

函数 pthread_create() 负责创建一条新线程。

```c++
#include <pthread.h>

int pthread_create(pthread_t *thread, const pthread_attr_t *attr, 
           void *(*start)(void *), void * arg);
```



#### 3.终止线程

可以如下方式终止线程的运行。

+ 线程start函数执行return语句并返回指定值。
+ 线程调用pthread_exit()
+ 调用pthread_cancel()取消线程。
+ 任意线程调用了exit()，或者主线程执行了return语句（在main函数的return语句），都会导致进程中的所有线程立即终止。

pthread_exit()函数将终止调用线程，且返回值可由另一线程通过调用pthread_join()来获取。

```c++
#include <pthread.h>

void pthread_exit(void * retval);
```

调用pthread_exit()相当于在线程的start函数中执行return，不同之处在于，可在线程start函数所调用的任意函数中调用pthread_exit()。

参数retval指定了线程的返回值。Retval所指向的内容不应分配于线程栈中，因为线程终止后，将无法确定线程栈的内容是否有效。（例如：系统可能会立即将进程虚拟内存的这片区域重新分配，供一个新的线程栈使用。）出于同样的理由，也不应在线程栈中分配线程start函数的返回值。

如果主线程调用了pthread_exit()，而非调用exit()或是执行return语句，那么其他线程将继续运行。



#### 4.线程ID

一个线程可以通过pthead_self()来获取自己的线程ID。

```c++
#include <pthread.h>

pthread_t pthread_self(void);
```

函数pthread_equal()可检查两个线程的ID是否相同。

```c++
#include <pthread.h>

int pthread_equal(pthread_t t1, pthread_t t2);
```



#### 5.连接(joining)已终止的线程

函数pthread_join()等待由 thread 标识的线程终止。（如果线程已经终止，pthread_join()会立即返回），这种操作被称为连接。

```c++
#include <pthread.h>

int pthread_join(pthread_t thread, void **retval);
```

若retval 为一非空指针，将会保存线程终止时返回值的拷贝。该返回值亦即线程调用return或pthread_exit()时所指定的值。

如向pthread_join()传入一个之前已然连接过的线程ID，将会导致无法预知的行为。例如：相同的线程ID在参与一次连接后恰好为另一新建线程所重用，再度连接的可能就是这个新线程。

若线程并未分离，则必须使用pthread_join()来进行连接。如果未能连接，那么线程终止时将产生僵尸线程，与僵尸进程概念相类似。除了浪费系统资源以外，僵尸线程若累积过多，应用将再也无法创建新的线程。

pthread_join与waitpid的区别在于：

+ 线程之间的关系是对等的，可以A连接C，C也连接A。
+ 无法连接任意线程（waitpid(-1)是可以的），也不能以非阻塞方式进行连接。



使用pthread的简单程序：

```c++
#include <pthread.h>
#include "tlpi_hdr.h"

static void * threadFunc(void * arg)
{
  char * s = (char *)arg;
  printf("%s", s);
  
  return (void *)strlen(s);
}

int main(int argc, char * argv[])
{
  pthread_t t1;
  void *res;
  int s;
  
  s = pthread_create(&t1, NULL, threadFunc, "Hello World\n");
  if (s != 0)
  {
    errExitEN(s, "pthread_create");
  }
  
  printf("Message from main()\n");
  s = pthread_join(t1, &res);
  if (s != 0)
  {
    errExitEN(s, "pthread_join");
  }
  
  printf("Thread returned %ld\n", (long)res);
  
  exit(EXIT_SUCCESS);
}
```



#### 6.线程的分离

默认情况下，线程是可连接的（joinable），也就是说，当线程退出时，其他线程可以通过调用 pthread_join() 获取其返回状态。有时，程序员并不关心线程的返回状态，只是希望系统在线程终止时能够自动清理并移除之。在这种情况下，可以调用 pthread_detach() 并向 thread 参数传入指定线程的标识符，将该线程标记为处于分离（detach）状态。

```c++
#include <pthread.h>

int pthread_detach(pthread_t thread);
```

如下例所示：使用pthread_detach()，线程可以自行分离：

pthread_detach(pthread_self());

一旦线程处于分离状态，就不能再使用 pthread_join() 来获取其状态，也无法使其重返”可连接“状态。

其他线程调用了exit()，或是主线程执行 return 语句时，即便遭到分离的线程也还是会受到影响。此时，不管线程处于可连接状态还是已分离状态，进程的所有线程会立即终止。换言之，pthread_join()只是控制线程终止后所发生的事情，而非何时或如何终止线程。



#### 7.线程的属性

前面已提及 pthread_create() 中类型为 pthread_attr_t 的 attr 参数，可利用其在创建线程时指定新线程的属性。



#### 8.线程与进程的比较

多线程方法的优点：

+ 线程间的数据共享很简单。相比之下，进程间的数据共享需要更多的投入。（例如：创建共享内存段或者使用管道pipe）；
+ 创建线程也快于创建进程。线程间的上下文切换，其消耗时间一般也比进程要短。

线程相对于进程的一些缺点如下所示：

+ 多线程编程时，需要确保调用线程安全的函数，或者以线程安全的方式来调用函数。多进程应用则无需关注这些。
+ 某个线程的bug（例如一个错误的指针来修改内存）可能会危及该进程的所有线程，因为它们共享着相同的地址空间和其他属性。相比之下，进程间的隔离更彻底。
+ 每个线程都在争用宿主进程中有限的虚拟空间地址。特别是，一旦每个线程栈以及线程特有数据（或线程本地存储）消耗掉进程虚拟地址空间的一部分，则后续线程将无缘使用这些区域。虽然有效地址空间很大，但当进程分配大量线程，亦或线程使用大量内存时，这一因素的限制作用也就突显出来。与之相反，每个进程都可以使用全部的有效虚拟内存，仅受制于实际内存和交换（swap）空间。

影响选择的还有如下几点：

+ 在多线程应用中处理信号，需要小心设计。（作为通则，一般建议在多线程程序中避免使用信号）；
+ 在多线程应用中，所有线程必须运行同一个程序（尽管可能是位于不同函数中）。对于多进程应用，不同的进程可以运行不同的程序。
+ 除了数据，线程还可以共享某些其他信息（例如，文件描述符，信号处置，当前工作目录，以及用户ID和组ID）。优劣之判，视应用而定。

