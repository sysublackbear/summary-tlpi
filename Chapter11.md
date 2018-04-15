### Chapter11.系统限制和选项

但凡UNIX实现，无不对各种系统特性和资源加以限制，并提供有各种标准所定义的选项。

尽管可以把假定的限制和选项硬性写入程序编码，但这将破坏程序的可移植性，因为限制和选项可能会有所不同。

sysconf()，pathconf()和fpathconf()，供应用程序调用以检查系统实现的限制和选项。



#### 1.从shell中获取限制和选项：getconf

```shell
$ getconf
usage: getconf [-v prog_env] system_var
       getconf [-v prog_env] path_var pathname
```



#### 2.在运行时获取系统限制（和选项）

sysconf()函数允许应用程序在运行时获得系统限制值。

```c++
#include <unistd.h>

long sysconf(int name);
```

pathconf()和 fpathconf() 函数允许应用程序在运行时获取文件相关的限制值。

```c++
#include <unistd.h>

long pathconf(const char *pathname, int name);
long fpathconf(int fd, int name);
```

