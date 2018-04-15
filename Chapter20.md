### Chapter20.信号：基本概念

#### 1.概念和概述

信号是事件发生时对进程的通知机制。有时也称之为软件中断。信号与硬件中断的相似之处在于打断了程序执行的正常流程，大多数情况下，无法预测信号到达的精确时间。

引发内核为进程产生信号的各类事件如下。

+ 硬件发生异常，即硬件检测到一个错误条件并通知内核，随即再由内核发送相应的信号给相关进程。
+ 用户键入了能够产生信号的终端特殊字符。
+ 发生了软件事件。例如：调整了终端窗口大小等等。



信号因某些事件而产生。信号产生后，会于稍后被传递给某一进程，而进程也会采取某些措施来相应信号。在产生和到达期间，信号处于等待状态。

通常，一旦（内核）接下来要调度该进程运行，等待信号会马上送达，或者如果进程正在运行，则会立即传递信号（例如，进程向自身发送信号）。然而，有时需要确保一段代码不为传递来的信号所中断。为了做到这一点，可以将信号添加到进程的信号掩码中——目前会阻塞该组信号的到达。如果所产生的信号属于阻塞之列，那么信号将保持等待状态，直至稍后对其解除阻塞（从信号掩码中移除）。进程可使用各种系统调用对其信号掩码添加和移除信号。



信号到达后，进程视具体信号执行如下默认操作之一：

+ 忽略信号：内核将信号丢弃，信号对进程没有产生任何影响；
+ 终止（杀死）进程：这有时是指进程异常终止，而不是进程因调用exit()而发生的正常终止。
+ 产生核心转储文件（core文件），同时进程终止：核心转储文件包含对进程虚拟内存的镜像，可将其加载到调试器中以检查进程终止时的状态。
+ 停止进程：暂停进程的执行。
+ 于之前暂停后再度恢复进程的执行。
+ 采取默认行为。
+ 执行信号处理器程序。



#### 2.改变信号处置：signal()

```c++
#include <signal.h>

void ( *signal(int sig, void (*handler)(int)) )(int);
```

这里需要对signal()函数的原型做一些解释。第一个参数sig，标识希望修改处置的信号编号，第二个参数handler，则标识信号抵达时所调用函数的地址。该函数无返回值（void），并接收一个整型参数。因此，信号处理器函数一般具有以下形式：

```c++
void handler(int sig)
{
	/* Code for the handler */
}
```

信号处理器函数指针做如下的typedef定义，有利于理解signal()的原型：

```c++
typedef void (*sighandler_t)(int);   // 定义一种新的sighandler_t类型，定义为指向某种函数的指针，以int作为函数参数，void为返回类型。

sighandler_t signal(int sig, sighandler_t handler);
```





#### 3.信号处理器

下面例子为SIGINT 信号安装一个处理器程序。

```c++
#include <signal.h>
#include "tlpi_hdr.h"

static void sigHandler(int sig)
{
  printf("Ouch!\n");
}

int main(int argc, char * argv[])
{
  int j;
  
  if (signal(SIGINT, sigHandler) == SIG_ERR)
  {
    errExit("signal");
  }
  
  for (j = 0; ; j++)
  {
    printf("%d\n", j);
    sleep(3);
  }
}
```





#### 4.发送信号：kill()

```c++
#include <signal.h>

int kill(pid_t pid, int sig);
```

+ pid > 0：发送pid指定进程；
+ pid==0：发送给与调用进程同组的每个进程，包括调用进程自身；
+ pid < -1：pid绝对值的所有下属进程；
+ pid == -1：往每个进程发生信号（需要root权限）。

此外，kill()可以使用空信号来检测具有特定进程ID的进程是否存在。



#### 5.发送信号的其他方式：raise()和killpg()

有时，进程需要向自身发送信号。

```c++
#include <signal.h>

int raise(int sig);
```

killpg()函数向某一进程组的所有成员发送一个信号。

```c++
#include <signal.h>

int killpg(pid_t pgrp, int sig);
```



#### 6.信号集

许多信号相关的系统调用都需要能表示一组不同的信号。例如：sigaction()和sigprocmask()允许程序指定一组将由进程阻塞的信号，而sigpending()则返回一组目前正在等待送达给一进程的信号。

多个信号可使用一个称之为信号集的数据结构来表示，其系统数据类型为sigset_t。



sigemptyset() 函数初始化一个未含任何成员的信号集。sigfillset()函数初始化一个信号集，使其包含所有信号（包括所有实时信号）。

```c++
#include <signal.h>

int sigemptyset(sigset_t *set);
int sigfillset(sigset_t *set);
```

必须使用 sigemptyset() 或者 sigfillset() 来初始化信号集，单纯靠静态变量初始化为0会存在移植性问题。



信号集初始化后，可以分别使用 sigaddset() 和 sigdelset() 函数向一个集合中添加或者移除单个信号。

```c++
#include <signal.h>

int sigaddset(sigset_t *set, int sig);
int sigdelset(sigset_t *set, int sig);
```



sigismember()函数用来测试信号sig是否是信号集set的成员。

```c++
#include <signal.h>

int sigismember(const sigset_t *set, int sig);
```

```c++
#define _GNU_SOURCE
#include <signal.h>

int sigandset(sigset_t *set, sigset_t *left, sigset_t *right);
int sigorset(sigset_t *dest, sigset_t *left, sigset_t *right);

int sigisemptyset(const sigset_t *set);
```

+ sigandset()将left集和right集的交集置于dest集。
+ sigorset()将left集和right集的并集置于dest集。
+ 若set集内未包含信号，则sigisemptyset()返回true。



#### 7.信号掩码（阻塞信号传递）

内核会为每个进程维护一个信号掩码，即一组信号，并将阻塞其针对该进程的传递。如果将遭阻塞的信号发送给某进程，那么对该信号的传递将延后，直至从进程信号掩码中移除该信号，从而解除阻塞为止。

向信号掩码中添加一个信号，有如下方式：

+ 当调用信号处理程序时，可将引发调用的信号自动添加到信号掩码中。是否发生这一情况，要视sigaction()函数在安装信号处理器程序时所使用的标志而定。
+ 使用sigaction()函数建立信号处理器程序时，可以指定一组额外信号，当调用该处理器程序时会将其阻塞。
+ 使用 sigprocmask() 系统调用，随时可以显式向信号掩码中添加或移除信号。

```c++
#include <signal.h>

int sigprocmask(int how, const sigset_t *set, sigset_t *oldset);
```

使用 sigprocmask()函数即可修改进程的信号掩码，又可获取现有掩码，或者两重功效兼具。通过how参数控制。



#### 8.处于等待状态的信号

如果某进程接受了一个该进程正在阻塞的信号，那么会将该信号填加到进程的等待信号集中。当之后解除了对该信号的锁定时，会随之将信号传递给此进程。为了确定进程中处于等待状态的是哪些信号，可以使用 sigpending()。

```c++
#include <signal.h>

int sigpending(sigset_t *set)；
```



#### 9.不对信号进行排队处理

等待信号集只是一个掩码，仅表明一个信号是否发生，而未表明其发生的次数。换言之，如果同一信号在阻塞状态下产生多次，那么会将该信号记录在等待信号集中，并在稍后仅传递一次。



#### 10.改变信号处置：sigaction()

```c++
#include <signal.h>

int sigaction(int sig, const sigaction *act, struct sigaction *oldact);
```



#### 11.等待信号：pause()

调用pause()将暂停进程的执行，直至信号处理器函数中断该调用为止（或者直至一个未处理信号终止进程为止）。

函数说明：pause()会令目前的进程暂停(进入睡眠状态), 直到被信号(signal)所中断。

```c++
#include <unistd.h>

int pause(void);
```

