### Chapter27.程序的执行

#### 1.执行新程序：execve()

​        系统调用execve()可以将新程序加载到某一进程的内存空间。在这一操作过程中，将丢弃旧有程序，而进程的栈，数据以及堆段会被新程序的相应部件所替换。在执行了各种C语言函数库的运行时启动代码以及程序的初始化代码后，例如，C++静态构造函数，或者以gcc constructor属性声明的C语言函数，新程序会从main()函数处开始执行。

​        由fork()生成的子进程对execve()的调用最为频繁，不以fork()调用为先导而单独调用execve()的做法在应用中实属罕见。

基于系统调用execve()，还提供了一系列冠以exec来命名的上层库函数，虽然接口方式各异，但功能相同。通常将调用这些函数加载一个新程序的过程称作exec操作，或是简单地以exec()来表示。

```c++
#include <unistd.h>

int execve(const char * pathname, char *const argv[], char *const envp[]);
```

值得一提的是，如果对pathname所指定的程序文件设置了 set-user-ID(set-group-ID)权限位，那么系统调用会在执行此文件时将进程的有效用户（组）ID置为程序文件的属主（组）ID。利用这一机制，可令用户在运行特定程序是临时获取特权。



使用execve()例子：

```c++
#include "tlpi_hdr.h"

int main(int argc, char * argv[])
{
  char *argVec[10];
  char *envVec[] = { "GREET=salut", "BYE=adieu", NULL };
  
  if (argc != 2 || strcmp(argv[1], "--help") == 0)
  {
    usageErr("%s pathname\n", argv[0]);
  }
  
  argVec[0] = strrchr(argv[1], '/');
  if (argVec[0] != NULL)
  {
    argVec[0]++;
  }
  else
  {
    argVec[0] = argv[1];
  }
  
  argVec[1] = "hello world";
  argVec[2] = "goodbye";
  argVec[3] = NULL;
  
  execve(argv[1], argVec, envVec);
  errExit("execve");
}
```

另一个程序：显示参数列表和环境列表

```c++
#include "tlpi_hdr.h"

extern char **environ;

int main(int argc, char * argv[])
{
  int j;
  char **ep;
  
  for (j = 0; j < argc; j++)
  {
    printf("argv[%d] = %s\n", j, argv[j]);
  }
  
  for (ep = environ; *ep != NULL; ep++)
  {
   	printf("environ: %s\n", *ep);
  }
  
  exit(EXIT_SUCCESS);
}
```

如何调用：

```shell
$ ./t_execve ./envargs
argv[0] = envargs
argv[1] = hello world
argv[2] = goodbye
environ: GREET=salut
environ: BYE=adieu
```



#### 2.上层exec()库函数

包括：execle, execlp, execvp, execv, execl。



#### 3.执行shell命令：system()

程序可通过调用system()函数来执行任意的shell命令。

```c++
#include <stdlib.h>

int system(const char * command);
```

函数system()创建一个子进程来运行shell，并以之执行命令 command。

system()的主要优点在于简便。

+ 无需处理对fork()，exec()，wait()和exit()的调用细节。
+ system()会代为处理错误和信号。
+ 因为system()使用shell来执行命令，所以会在执行command之前对其进行所有的常规shell处理，替换以及重定向操作。
+ 但是有个缺点就是低效率。



#### 4.system()的实现

```c++
#include <unistd.h>
#include <signal.h>
#include <sys/wait.h>
#include <sys/types.h>
#include <errno.h>

int system(const char * command)
{
  sigset_t blockMask, origMask;
  struct sigaction saIgnore, saOrigQuit, saOrigInt, saDefault;
  pid_t childPid;
  int status, savedErrno;
  
  if (command == NULL)  // 判断shell程序是否可用
  {
    return system(":") == 0;
  }
  
  sigemptyset(&blockMask);  // block住SIGCHLD（子进程结束的信号）
  sigaddset(&blockMask, SIGCHLD);
  sigprocmask(SIG_BLOCK, &blockMask, &origMask);
  
  saIgnore.sa_handler = SIG_IGN;
  saIgnore.sa_flags = 0;
  sigemptyset(&saIgnore.sa_mask);
  sigaction(SIGINT, &saIgnore, &saOrigInt);
  sigaction(SIGQUIT, &saIgnore, &saOrigQuit);  // 忽略ctrl-c，ctrl-\ 信号
  
  switch (childPid = fork()) {
    case -1: // fork() failed
      status = -1;
      break;
      
    case 0:  // Child Process
      saDefault.sa_handler = SIG_DFL;
      saDefault.sa_flags = 0;
      sigemptyset(&saDefault.sa_mask);
      
      if (saOrigInt.sa_handler != SIG_IGN)
        sigaction(SIGINT, &saDefault, NULL);
      if (saOrigQuit.sa_handler != SIG_IGN)
        sigaction(SIGQUIT, &saDefault, NULL);
      
      sigprocmask(SIG_SETMASK, &origMask, NULL);
      
      // 子进程刚从fork()返回时，会将对SIGINT和SIGQUIT的处置置为SIG_IGN（继承自父进程）。
      // 不过，fork()不会改变子进程对这些信号的处理方式，而exec()则会将对已处理信号的处置重置为默认值，
      // 但不改变对其他信号的处置。
      // 因此，如果调用者对SIGINT和SIGQUIT的处置设置并非SIG_IGN,那么子进程会将其置为SIG_DFL
      
      // SIG_DFL:默认信号的处理程序
      // SIG_IGN:忽略信号的处理程序
      
      excel("/bin/sh", "sh", "-c", command, (char *)NULL);
      _exit(127);
    default:  // Parent process
      while (waitpid(childPid, &status, 0) == -1)
      {
        if (errno != EINTR)
        {
          status = -1;
          break;
        }
      }
      break;
  }

  savedErrno = errno;   // 下面的过程有可能会修改到errno的值
    
  // 去除屏蔽
  sigprocmask(SIG_SETMASK, &origMask, NULL);
  sigaction(SIGINT, &saOrigInt, NULL);
  sigaction(SIGQUIT, &saOrigQuit, NULL);
  
  errno = savedErrno;
  
  return status;
  
}
```







