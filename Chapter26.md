### Chapter26.监控子进程

在很多应用程序的设计中，父进程需要知道其某个子进程于何时改变了状态——子进程终止或因收到信号而停止。包含：系统调用wait()以及信号 SIGCHLD。



#### 1.等待子进程

**系统调用 wait()**

系统调用 wait() 等待调用进程的任一子进程终止，同时在参数 status 所指向的缓冲区中返回该子进程的终止状态。

```c++
#include <sys/wait.h>

pid_t wait(int * status);
```

系统调用wait()执行如下动作：

1. 如果调用进程并无之前未被等待的子进程终止，调用将一直阻塞，直至某个子进程终止。如果调用时已有子进程终止，wait()则立即返回。
2. 如果status非空，那么关于子进程如何终止的信息则会通过status指向的整型变量返回。
3. 内核将会为父进程下的所有子进程的运行总量追加进程cpu时间以及资源使用数据。
4. 将终止子进程的ID作为wait()的结果返回。



#### 2.系统调用 waitpid()

系统调用wait()存在诸多限制，而设计waitpid()则意在突破这些限制。

+ 如果父进程已经创建了多个子进程，使用wait()将无法等待某个特定子进程的完成，只能按顺序等待下一个子进程的终止。
+ 如果没有子进程退出，wait()总是保持阻塞。有时候会希望执行非阻塞的等待：是否有子进程退出，立判可知。
+ 使用wait()只能发现那些已经终止的子进程。对于子进程因某个信号（如 SIGSTOP 或 SIGTTIN）而停止，或是已停止子进程收到 SIGCONT 信号后恢复执行的情况就无能为力了。

```c++
#include <sys/wait.h>

pid_t waitpid(pid_t pid, int * status, int options);
```

waitpid()与wait()的返回值以及参数 status 的意义相同。

+ pid > 0 , 表示等待进程ID为pid的子进程。
+ pid == 0，等待与调用进程（父进程）同一个进程组（process group）的所有子进程。
+ pid < -1，等待进程组标识符与pid绝对值相等的所有子进程。
+ pid == -1，则等待任意子进程，wait(&status)的调用与waitpid(-1, &status, 0)等价。



#### 3.孤儿进程与僵尸进程

父进程与子进程的生命周期一般都不相同，父子进程间互有长短。

+ 谁会是孤儿子进程的父进程？进程ID为1的众进程之祖——init会接管孤儿进程。换言之，某一子进程的父进程终止后，对getpid()的调用将返回1.这是判定某一子进程之“生父”是否“在世”的方法之一。
+ 在父进程执行wait()之前，其子进程就已经终止，这将会发生什么？此处的要点在于，即使子进程已经结束，系统仍然允许其父进程在之后的某一时刻去执行wait()，以确定该子进程是如何终止的。内核通过将子进程转为僵尸进程（zombie）来处理这种情况。这也将意味着将释放子进程所把持的大部分资源，以便供其他进程重新使用。该进程所唯一保留的是内核进程表中的一条记录，其中包含了子进程ID，终止状态，资源使用数据等信息。

至于僵尸进程名称的由来，则由于——无法通过信号来杀死僵尸进程，即便是（银弹）SIGKILL。这就确保了父进程总是可以执行wait()方法。

当父进程执行wait()后，由于不再需要子进程所剩余的最后信息，故而内核将删除僵尸进程。另一方面，如果父进程未执行wait()随即退出，那么init进程将接管子进程并自动调用wait()，从而从系统中移除僵尸进程。

如果父进程创建了某一子进程，但并未执行wait()，那么在内核的进程表中将为该子进程永久保留一条记录。如果存在大量此类僵尸进程，它们势必将填满内核进程表，从而阻碍新进程的创建。既然无法用信号杀死僵尸进程，那么从系统中将其移除的唯一方法就是杀掉它们的父进程（或等待其父进程终止），此时init进程将接管和等待这些僵尸进程，从而从系统中将它们清理掉。

在设计长生命周期的父进程（例如：会创建众多子进程的网络服务器和shell）时，这些语义具有重要意义。换句话说，此类应用中，父进程应执行wait()方法，以确保系统总是能够清理那些死去的子进程，避免使其成为长寿僵尸。父进程在处理SIGCHLD信号时，对wait()的调用既可同步，也可异步。



#### 4.SIGCHLD信号

子进程的终止属于异步事件。父进程无法预知其子进程何时终止。（即使父进程向子进程发送 SIGKILL 信号，子进程终止的确切时间还依赖于系统的调度：子进程下一次在何时使用CPU。）之前已经论及，父进程应使用wait()（或类似调用）来防止僵尸子进程的累积，以及采用如下两种方法来避免这一问题。

+ 父进程调用不带WNOHANG标志的wait()，或waitpid()方法，此时如果尚无已经终止的子进程，那么调用将会阻塞。
+ 父进程周期性第调用带有WHOHANG标志的waitpid()，执行针对已终止子进程的非阻塞检查（轮询）。

这两种方法使用起来都有所不便。一方面，可能并不希望父进程以阻塞的方式来等待子进程的终止。另一方面，反复调用非阻塞的waitpid()会造成CPU资源的浪费，并增加应用程序设计的复杂度。为了规避这些问题，可以采用针对SIGCHLD信号的处理程序。



**通过SIGCHLD信号处理程序捕获已终止的子进程的例子：**

```c++
#include <signal.h>
#include <sys/wait.h>
#include "print_wait_status.h"
#include "curr_time.h"
#include "tlpi_hdr.h"

static volatile int numLiveChildren = 0;  // Number of children started but not yet waited on

static void sigchldHandler(int sig)
{
  int status, savedErrno;
  pid_t childPid;
  
  savedErrno = errno;
  
  printf("%s handler: Caught SIGCHLD\n", currTime("%T"));
  
  while ((childPid = waitpid(-1, &status, WNOHANG)) > 0)  // -1 代表等待所有的子进程
  {
    printf("%s handler: Reaped child %ld - ", currTime("%T"), (long)childPid);
    printWaitStatus(NULL, status);
    numLiveChildren--;
  }
  
  if (childPid == -1 && errno != ECHILD)
  {
    errMsg("waitpid");
  }
  
  sleep(5);
  printf("%s handler: returning\n", currTime("%T"));
  
  errno = savedErrno;
}

int main(int argc, char * argv[])
{
  int j, sigCnt;
  sigset_t blockMask, emptyMask;
  struct sigaction sa;
  
  if (argc < 2 || strcmp(argv[1], "--help") == 0)
  {
    usageErr("%s child-sleep-time...\n", argv[0]);
  }
  
  setbuf(stdout, NULL);
  
  sigCnt = 0;
  numLiveChildren = argc - 1;
  
  sigemptyset(&sa.sa_mask);
  sa.sa_flags = 0;
  sa.sa_handler = sigchldHandler;
  if (sigaction(SIGCHLD, &sa, NULL) == -1)
  {
    errExit("sigaction");
  }
  
  sigemptyset(&blockMask);
  sigaddset(&blockMask, SIGCHLD);
  if (sigprocmask(SIG_SETMASK, &blockMask, NULL) == -1)
  {
    errExit("sigprocmask");
  }
  
  for (j = 1; j < argc; j++)
  {
    switch(fork())
    {
      case -1:
        errExit("fork");
        
      case 0:
        sleep(getInt(argv[j], GN_NONNEG, "child-sleep-time"));
        printf("%s Child %d (PID=%ld) exiting\n", currTime("%T"), j, (long)getpid());
        _exit(EXIT_SUCCESS);
        
      default:  // Parent - loops to create next child
        break;
    }
  }
  
  sigemptyset(&emptyMask);
  while (numLiveChildren > 0)
  {
    if (sigsuspend(&emptyMask) == -1 && errno != EINTR)  // 阻塞信号掩码
      errExit("sigsuspend");
    sigCnt++;
  }
  
  printf("%s All %d children have terminated; SIGCHLD was caught "
        "%d times\n", currTime("%T"), argc - 1, sigCnt);
  exit(EXIT_SUCCESS);
}
```

















