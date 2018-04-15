### Chapter32.线程：线程取消

#### 1.取消一个线程

函数 pthread_cancel() 向由 thread 指定的线程发送一个取消请求。

```c++
#include <pthread.h>

int pthread_cancel(pthread_t thread);
```

发出取消请求后，函数 pthread_cancel() 当即返回，不会等待目标线程的退出。



#### 2.取消状态及类型

函数 pthread_setcancelstate() 和 pthread_setcanceltype() 会设定标志，允许线程对取消请求响应过程加以控制。

```c++
#include <pthread.h>

int pthread_setcancelstate(int state, int * oldstate);
int pthread_setcanceltype(int type, int * oldtype);
```



#### 3.调用pthread_cancel() 取消线程的例子

```c++
#include <pthread.h>
#include "tlpi_hdr.h"

static void * threadFunc(void *arg)
{
  int j;
  printf("New thread started\n");
  for (j = 1; ; j++)
  {
    printf("Loop %d\n", j);
    sleep(1);
  }
  
  // NOTREACHED
  return NULL;
}

int main(int argc, char * argv[])
{
  pthread_t thr;
  int s;
  void *res;
  
  s = pthread_create(&thr, NULL, threadFunc, NULL);
  if (s != 0)
  {
    errExitEN(s, "pthread create");
  }
  
  sleep(3);
  
  s = pthread_cancel(thr);
  if (s != 0)
  {
    errExitEN(s, "pthread_cancel");
  }
  
  s = pthread_join(thr, &res);
  if (s != 0)
  {
    errExitEN(s, "pthread_join");
  }
  
  if (res == PTHREAD_CANCELED)
  {
    printf("Thread was canceled\n");
  }
  else
  {
    printf("Thread was not canceled (should not happen!)\n");
  }
  
  exit(EXIT_SUCCESS);
}
```



#### 4.线程可取消性的检测

上面的例子，由main()创建的线程会执行属于取消点的函数（sleep()属于取消点，printf()可能也是），因而会接受取消请求。不过，假设线程执行的是一个不含取消点的循环（计算密集型循环），这时，线程永远也不会响应取消请求。

函数 pthread_testcancel()的目的很简单，就是产生一个取消点。线程如果已有处于挂起状态的取消请求，那么只要调用该函数，线程就会随之终止。

```c++
#include <pthread.h>

void pthread_testcancel(void);
```



#### 5.清理函数

一旦处于挂起状态的取消请求，线程在执行到取消点时如果只是草草收场，这会将共享变量以及 Pthreads 对象（例如互斥量）置于一种不一致状态，可能导致进程中其他线程产生错误结果，死锁，甚至造成程序崩溃。为规避这一问题，线程可以设置一个或多个清理函数，当线程遭取消时会自动运行这些函数，在线程终止之前可执行诸如修改全局变量，解锁互斥量等动作。

每个线程都可以拥有一个清理函数栈。当线程遭取消时，会沿该栈自顶向下依次执行清理函数，首先会执行最近设置的函数，接着是次新的函数，以此类推。当执行完所有清理函数后，线程终止。

函数 pthread_cleanup_push() 和 pthread_cleanup_pop() 分别负责向调用线程的清理函数栈添加和移除清理函数。

```c++
#include <pthread.h>

void pthread_cleanup_push(void (*routine)(void*), void *arg);
void pthread_cleanup_pop(int execute);
```



#### 6.异步取消

如果设定线程为异步取消时（取消类型为PTHREAD_CANCEL_ASYNCHORONOUS），可以在任何时点将其取消（亦即，执行任何机器指令时），取消动作不会拖延到下一个取消点才执行。

异步取消的问题在于，尽管清理函数依然会得以执行，但处理函数却无从得知线程的具体状态。



#### 7.总结

函数 pthread_cancel() 允许某线程向另一个线程发送取消请求，要求目标线程终止。

至于目标线程如何响应，取决于其取消性状态和类型。如果禁用线程的取消性状态，那么请求会保持挂起（pending）状态，直至将线程的取消性状态置为启用。如果启用取消性状态，那么线程何时响应请求则依赖于取消性类型。若类型为延迟取消，则在线程下一次调用某个取消点时，取消发生。如果为异步取消类型，取消动作随时可能发生（鲜有使用）。

线程可以设置一个清理函数栈，其中的清理函数属于由开发人员定义的函数，当线程遭到取消时，会自动调用这些函数以执行清理工作（例如，恢复共享变量状态，或解锁互斥量）。