### Chapter24.进程的创建

#### 1.fork()，exit()，wait()以及 execve() 的简介

+ 系统调用 fork() 允许一进程（父进程）创建一新进程（子进程）。具体做法是，新的子进程几近于对父进程的翻版：子进程获得父进程的栈，数据段，堆和执行文本段的拷贝。
+ 库函数exit(status)终止一进程，将进程占用的所有资源（内存，文件描述符等）归还内核，交其进行再次分配。参数status为一整形变量，表示进程的退出状态。父进程可使用系统调用wait()来获取该状态。
+ 系统调用wait(&status)的目的有二：其一，如果子进程尚未调用exit()终止，那么wait()会挂起父进程直至子进程终止；其二，子进程的终止状态通过wait()的status参数返回。
+ 系统调用execve(pathname, argv, envp) 加载一个新程序（路径名为pathname，参数列表为argv，环境变量列表为 envp到当前进程的内存。这将丢弃现存的程序文本段，并为新程序重新创建栈，数据段以及堆。通常将这一动作称为执行一个新程序。

其他一些操作系统则将 fork() 和 exec() 的功能合二为一，形成单一的spawn()操作——创建一个新进程并执行指定程序。



#### 2.创建新进程：fork()

在诸多应用中，创建多个进程是任务分解时行之有效的方法。例如，某一网络服务器进程可在侦听客户端请求的同时，为处理每一请求而创建一新的子进程，与此同时，服务器进程会继续侦听更多的客户端连接请求。以此类手法分解任务，通常会简化应用程序的设计，同时提高了系统的并发性。（即，可同时处理更多的任务或请求。）

```c++
#include <unistd.h>

pid_t fork(void);
```

完成对其调用后将存在两个进程，且每个进程都会从fork()的返回处继续执行。

这两个进程将执行相同的程序文本段，但却各自拥有不同的栈段，数据段以及堆段拷贝。子进程的栈，数据以及栈段开始时是对父进程内存相应各部分的完全赋值。执行fork()之后，每个进程均可修改各自的栈数据，以及堆段中的变量，而并不影响另一进程。

**调用fork()的基本例子**

```c++
#include "tlpi_hdr.h"

static int idata = 111;     // 数据段

int main(int argc, char * argv[])
{
  int istack = 222;
  pid_t childPid;
  
  switch (childPid = fork())
  {
    case -1:
      errExit("fork");
      
    case 0:
      idata *= 3;
      istack *= 3;
      break;
      
    default:
      sleep(3);
      break;
  }
  
  printf("PID=%ld %s idata=%d istack=%d\n", (long)getpid(), (childPid == 0) ? "(child)" : "(parent)", idata, istack);
  exit(EXIT_SUCCESS);
}
```



#### 3.父，子进程间的文件共享

​        执行fork()时，子进程会获得父进程所有文件描述符的副本。这些副本的创建方式类似于dup()，这也意味着父，子进程中对应的描述符均指向相同的打开文件句柄（即 open file description）。打开的文件句柄包含了当前文件偏移量以及文件状态标志。一个打开文件的这些属性因之而在父子进程间实现了共享。举例来说，如果子进程更新了文件偏移量，那么这种改变也会影响到父进程中相应的描述符。

​        父子进程间共享打开文件属性的妙用屡见不鲜。例如，假设父子进程同时写入一个文件，共享文件偏移量会确保二者不会覆盖彼此的输出内容。不过，这并不能阻止父子进程的输出随意混杂在一起。要想规避这一现象，需要进行进程间同步。比如，父进程可以使用系统调用wait()来暂停运行并等待子进程退出。shell就是这么做：只有当执行命令的子进程退出后，shell才会打印出提示符。

如果不需要这种对文件描述符的共享方式，那么在设计应用程序时，应于fork()调用后注意两点：其一，令父，子进程使用不同的文件描述符；其二，各自立即关闭不再使用的描述符（亦即哪些经由其他进程使用的描述符）。



#### 4.fork()的内存语义

大部分UNIX系统的实现才用这两种技术避免fork()过程复制的浪费：

- 内核将每一进程的代码段标记为只读，从而使进程无法修改自身代码。这样，父，子进程可以共享同一代码段。系统调用fork()在为子进程创建代码段时，其所构建的一系列进程级页表项均指向与父进程相同的物理内存页帧。
- 对于父进程数据段，堆段和栈段中的各页，内核采用写时复制(copy-on-write)技术来处理。最初，内核做了一些设置，令这些段的页表项指向与父进程相同的物理内存页，并将这些页面自身标记为只读。调用fork()之后，内核会捕获所有父进程或子进程针对这些页面的修改企图，并为将要修改的页面创建拷贝。系统将新的页面拷贝分配给遭内核捕获的进程，还会对子进程的相应页表项做适当调整。从这一刻起，父，子进程可以分别修改各自的页拷贝，不再相互影响。

#### 5.系统调用vfork()

```c++
#include <unistd.h>

pid_t vfork(void);
```

vfork()因为如下两个特性而更具效率，这也是其与fork()的区别所在。

- 无需为子进程复制虚拟内存页或页表。相反，子进程共享父进程的内存，直至其成功执行了exec()或是调用_exit()退出。
- 在子进程调用exec()或_exit()之前，将暂停执行父进程。

#### 6.同步信号以规避竞争条件

例子如下：

```c++
#include <signal.h>
#include "curr_time.h"
#include "tlpi_hdr.h"

#define SYNC_SIG SIGUSR1  // 在POSIX兼容的平台上，SIGUSR1和SIGUSR2是发送给一个进程的信号，它表示用户定义的情况。

static void handler(int sig)
{
}

int main(int argc, char * argv[])
{
    pid_t childPid;
    sigset_t blockMask, origMask, emptyMask;
    struct sigaction sa;

    setbuf(stdout, NULL);   // 将标准输出缓冲区建立起来，如果第二个参数为NULL，则为无缓冲

    sigemptyset(&blockMask);  // 初始化信号集
    sigaddset(&blockMask, SYNC_SIG);  // 添加信号来屏蔽
    if (sigprocmask(SIG_BLOCK, &blockMask, &origMask) == -1)  // 添加信号进行屏蔽
    {
        errExit("sigprocmask error!");
    }

    sigemptyset(&sa.sa_mask);
    sa.sa_flags = SA_RESTART;  // 设置SA_RESTART属性，那么当信号处理函数返回后，被该信号中断的系统调用将自动恢复。
    sa.sa_handler = handler;
    if (sigaction(SYNC_SIG, &sa, NULL) == -1)
    {
        errExit("sigprocmask error!");
    }

    switch (childPid = fork())
    {
        case -1:
            errExit("fork error!");

        case 0: // Child
            printf("[%s %ld] Child started - doing some work\n", currTime("%T"), (long)getpid());
            sleep(2);

            printf("[%s %ld] Child about to signal parent\n", currTime("%T"), (long)getpid());

            if (kill(getppid(), SYNC_SIG) == -1)  // 发送信号
            {
                errExit("kill");
            }

            // Now child can do other things...

            _exit(EXIT_SUCCESS);

        default: // Parent
            printf("[%s %ld] Parent about to wait for signal\n", currTime("%T"), (long)getpid());
            sigemptyset(&emptyMask);

            if (sigsuspend(&emptyMask) == -1 && errno != EINTR) // 阻塞当前进程，等待mask信号集（信号掩码）之外的任何信号的到来
            {
                errExit("sigsuspend error");
            }

            printf("[%s %ld] Parent got signal\n", currTime("%T"), (long)getpid());

            // 如果有必要的话，解除原来的信号掩码
            if (sigprocmask(SIG_SETMASK, &origMask, NULL) == -1)
            {
                errExit("sigprocmask error")
            }

            // 父进程继续做别的事情

            exit(EXIT_SUCCESS);
    }
}
```









