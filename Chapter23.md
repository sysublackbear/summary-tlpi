### Chapter23.定时器与休眠

定时器是进程规划自己在未来某一时刻接获通知的一种机制。休眠则能使进程（或线程）暂停执行一段时间。



#### 1.间隔定时器

系统调用 setitimer() 创建一个间隔式定时器，这种定时器会在未来某个时间点到期，并于此后（可选择地）每隔一段时间到期一次。

```c++
#include <sys/time.h>

int setitimer(int which, const struct itimerval *new_value, struct itimerval *old_value);
```

可以在任何时刻调用getitimer()，以了解定时器的当前状态，距离下次到期是的剩余时间。

```c++
#include <sys/time.h>

int getitimer(int which, struct itimerval *curr_value);
```

**settimer的工作机制是，先对先对it_value倒计时，当it_value为零时触发信号，然后重置为it_interval，继续对it_value倒计时，一直这样循环下去。 机制：发送SIGALRM信号。**



利用定时器的例子如下：机制是为**SIGALRM信号**创建处理器函数。

```c++
#include <stdio.h>
#include <signal.h>
#include <sys/time.h>

void signalHandler(int signo)
{
    switch (signo){
        case SIGALRM:
            printf("Caught the SIGALRM signal!\n");
            break;
   }
}

int main(int argc, char *argv[])
{
    signal(SIGALRM, signalHandler);

    struct itimerval new_value, old_value;
    new_value.it_value.tv_sec = 0;
    new_value.it_value.tv_usec = 1;
    new_value.it_interval.tv_sec = 0;
    new_value.it_interval.tv_usec = 200000;
    setitimer(ITIMER_REAL, &new_value, &old_value);
    
    for(;;);
     
    return 0;
}
```



#### 2.更为简单的定时器接口：alarm()

```c++
#include <unistd.h>

unsigned int alarm(unsigned int seconds);
```



#### 3.暂停运行（休眠）一段固定时间

##### 低分辨率休眠：sleep()

函数sleep()可以暂停调用进程的执行达数秒之久（由参数seconds设置），或者在捕获到信号（从而中断调用）后恢复进程的运行。

```c++
#include <unistd.h>

unsigned int sleep(unsigned int seconds);
```

##### 高分辨率休眠：nanosleep()

函数 nanosleep() 的功用与 sleep()类似，但更具优势，其中包括能以更高分辨率来设定休眠间隔时间。

```c++
#define _POSIX_C_SOURCE 199309
#include <time.h>

int nanosleep(const struct timespec *request, struct timespec *remain);

struct timespec {
  time_t tv_sec;
  long tv_nsec;
};
```

