### Chapter12.系统和进程信息

#### 1./proc 文件系统

/proc/PID：获取与进程有关的信息；

/proc/PID/fd目录：/proc/PID/fd 目录为进程打开的每个文件描述符都包含了一个符号链接，每个符号链接的名称都与描述符的数值相匹配。例如：/proc/1968/1 是ID为1968的进程中指向标准输出的符号链接。

/proc/PID/task ：线程目录



#### 2.系统标识：uname()

uname()系统调用返回了一系列关于主机系统的标识信息。

```c++
#include <sys/utsname.h>

int uname(struct utsname *utsbuf);

// 其中utsbuf参数的定义：
#define _UTSNAME_LENGTH 65

struct utsname {
  char sysname[_UTSNAME_LENGTH];
  char nodename[_UTSNAME_LENGTH];
  char release[_UTSNAME_LENGTH];
  char version[_UTSNAME_LENGTH];
  char machine[_UTSNAME_LENGTH];
  
 #ifdef _GNU_SOURCE
  char domainname[_UTSNAME_LENGTH];
 #endif
}
```



