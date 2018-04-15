### Chapter25.进程的终止

#### 1.进程的终止：_exit()和 exit()

通常，进程有两种终止方式。其一为异常（abnormal）终止，由对一信号的接收而引发，该信号的默认动作为终止当前进程，可能产生核心转储（core dump文件）。此外，进程可使用_exit()系统调用正常终止。

```c++
#include <unistd.h>

void _exit(int status);
```

_exit()的 status 参数定义了进程的终止状态，父进程可调用 wait() 以获取该状态。虽然将其定义为 int 类型，但仅有低8位可为父进程所用。

调用_exit()的程序总会成功终止（即，__exit()从不返回）。



程序一般不会直接调用_exit()，而是调用库函数exit()，它会在调用 _exit() 前执行各种动作。

```c++
#include <stdlib.h>

void exit(int status);
```

exit()会执行的动作如下：

+ 调用退出处理程序，其执行顺序与注册顺序相反。
+ 刷新stdio流缓冲区。
+ 使用由status提供的值执行_exit()系统调用。



程序的另一种终止方法是从main()函数中返回(return)，或者或明或暗地一直执行到main()函数的结尾处。执行 return n 等同于执行对 exit(n)的调用，因为调用 main() 的运行是函数会将 main() 的返回值作为 exit() 的参数。



**注册退出处理程序**

```c++
#include <stdlib.h>

int atexit(void (*func)(void));
```



#### 2.fork()，stdio缓冲区以及_exit()之间的交互

代码例子如下：

```c++
#include "tlpi_hdr.h"

int main(int argc, char * argv[])
{
  printf("Hello world!\n");
  write(STDOUT_FILENO, "Ciao\n", 5);
  
  if (fork() == -1)
  {
    errExit("fork error!");
  }
  
  exit(EXIT_SUCCESS);
}
```

程序的输出结果有点奇怪，当程序标准输出定向到终端时，会看到预期的结果：

```shell
$ ./fork_stdio_buf
Hello world
Ciao
```

不过，当重定向标准输出到一个文件时，结果如下：

```shell
$ ./fork_stdio_buf > a
$ cat a
Ciao
Hello world
Hello world
```

​       要理解为什么printf()的输出消息出现了两次，首先要记住，是在进程的用户空间内存中维护stdio缓冲区的。因此，通过fork()创建子进程时会复制这些缓冲区。当标准输出定向到终端时，因为缺省为行缓冲，所以会立即显示函数printf()输出的包含换行符的字符串。不过，当标准输出重定向到文件时，由于缺省为块缓冲，所以在本例中，当调用fork()时，printf()输出的字符串仍在父进程的stdio缓冲区中，并随子进程的创建而产生一份副本。父，子进程调用exit()时会刷新各自的stdio缓冲区，从而导致重复的输出结果。



可以才用以下任一方法来避免重复的输出结果：

+ 作为针对stdio缓冲区问题的特定解决方案，可以在调用fork()之前使用函数fflush()来刷新stdio缓冲区。作为另一种选择，也可以使用setvbuf()和setbuf()来关闭stdio流的缓冲功能。
+ 子进程可以调用_exit()而非exit()，以便不再刷新stdio缓冲区。这一技术例证了一个更为通用的原则：在创建子进程的应用中，典型情况下仅有一个进程（一般为父进程）应通过调用exit()终止，而其他进程应调用 _exit()终止，从而确保只有一个进程调用退出处理程序并刷新stdio缓冲区。



