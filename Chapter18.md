### Chapter18.目录与链接



#### 1.目录和（硬）链接

在文件系统中，目录的存储方式类似于普通文件。目录与普通文件的区别有二。

+ 在其 i-node 条目中，会将目录标记为一种不同的文件类型。
+ 目录是经特殊组织而成的文件。本质上说就是一个表格，包含文件名和i-node编号。

在大多数原生Linux文件系统上，文件名长度可达255个字符。

```shell
mikedeng@TENCENT64:~/QQMail/mmtenpay/mmpaybasic/mmpayentrust/scripts/mmpayentrust_historydata/kv_tools>ls -li
total 48
41943206 -rw-rw-rw- 1 mikedeng mikedeng   750 Feb  7 20:48 BUILD
41943166 -rw-rw-rw- 1 mikedeng mikedeng 35228 Feb 21 16:54 entrust_data_importer.cpp
41943110 -rw-rw-rw- 1 mikedeng mikedeng  3482 Feb 16 20:22 entrust_data_importer.h
41943280 -rw-rw-rw- 1 mikedeng mikedeng  3005 Feb  8 15:40 import_entrust_data_tool.cpp
```

上图第3栏代表i-node的链接计数，仅当i-node 的链接计数降为0时，也就是溢出了文件的所有名字时，才会删除（释放）文件的 i-node 记录和数据块。总结如下：rm命令从目录列表中删除一文件名，将相应 i-node 的链接计数减一，若链接计数因此而降为0，则还将释放该文件名所只带的 i-node 和数据块。



对硬链接的限制有二，均可用符号链接来加以规避。

+ 因为目录条目（硬链接）对文件的指代采用了i-node编号，而i-node编号的唯一性仅在一个文件系统之内才能得到保障，所以硬链接必须与其只带的文件驻留在同一文件系统中。
+ 不能为目录创建硬链接，从而避免出现令诸多系统程序陷于混乱的链接环路。



#### 2.创建和移除（硬）链接：link() 和 unlink()

link() 和 unlink() 系统调用分别创建和移除硬链接。

```c++
#include <unistd.h>

int link(const char *oldpath, const char *newpath);
```

```c++
#include <unistd.h>

int unlink(const char *pathname);
```



#### 3.更改文件名

借助于rename()系统调用，既可以重命名文件，又可以将文件移至同一文件系统中的另一目录。

```c++
#include <stdio.h>

int rename(const char *oldpath, const char *newpath);
```



#### 4.使用符号链接：symlink() 和 readlink()

symlink() 系统调用会针对由filepath 所指定的路径名创建一个新的符号链接——linkpath。（想移除符号链接，需使用 unlink() 调用）。

```c++
#include <unistd.h>

int symlink(const char *filepath, const char *linkpath);
```

如果指定一符号链接作为open()调用的pathname参数，那么将打开链接指向的文件。有时，倒宁愿获取链接本身的内容，即其所指向的路径名。这正是readlink()系统调用的本职工作，将符号链接字符串的一份副本置于buffer指向的字符数组中。

```c++
#include <unistd.h>

ssize_t readlink(const char *pathname, char *buffer, size_t bufsiz);
```



#### 5.创建和移除目录：mkdir()和rmdir()

mkdir()系统调用创建一个新目录。

```c++
#include <sys/stat.h>

int mkdir(const char *pathname, mode_t mode);
```

rmdir()系统调用移除由pathname 指定的目录，该目录可以是绝对路径名，也可以是相对路径名。

```c++
#include <unistd.h>

int rmdir(const char *pathname);
```



#### 6.移除一个文件或目录：remove()

remove()库函数移除一个文件或一个空目录。

```c++
#include <stdio.h>

int remove(const char *pathname);
```



#### 7.读目录：opendir() 和 readdir()

opendir()函数打开一个目录，并返回指向该目录的句柄，供后续调用使用。

```c++
#include <dirent.h>

DIR *opendir(const char *dirpath);
```

readdir()函数从一个目录流中读取连续的条目。

```c++
#include <dirent.h>

struct dirent *readdir(DIR *dirp);

struct dirent {
  ino_t d_ino;   /* File i-node number */
  char d_name[];  /* Null-terminated name of file */
}
```



#### 8.进程的当前工作目录

一个进程的当前工作目录(current working directory)定义了该进程解析相对路径名的起点。新进程的当前工作目录继承自其父进程。

**获取当前工作目录**

```c++
#include <unistd.h>

char *getcwd(char *cwdbuf, size_t size);
```



**改变当前工作目录**

chdir() 系统调用将调用进程的当前工作目录改变为由 pathname 指定的相对或绝对路径名（如属于符号链接，还会对其解除引用）。

```c++
#include <unistd.h>

int chdir(const char *pathname);
```

fchdir()系统调用与 chdir() 作用相同，只是在指定目录时使用了文件描述符，而该描述符是之前调用open()打开相应目录时获得的。

```c++
#define _XOPEN_SOURCE 500
#include <unistd.h>

int fchdir(int fd);
```





#### 9.改变进程的根目录：chroot()

每个进程都有一个根目录，该目录是解释绝对路径（即那些以/开始的目录）时的起点。默认情况下，这是文件系统的真实根目录，新进程从其父进程处继承根目录。

```c++
#define _BSD_SOURCE
#include <unistd.h>

int chroot(const char *pathname);
```

ftp程序就是应用chroot()的典型实例之一。作为一种安全措施，当用户匿名登录ftp时，ftp程序将使用 chroot() 为新进程设置根目录——一个专门预留给匿名登录用户的目录。调用chroot()后，用户将受困于文件系统中新根目录下的子树中，无法再整个文件系统中信马由缰。



#### 10.解析路径名：realpath()

realpath()库函数对 pathname（以空字符结尾的字符串）中的所有符号链接——解除引用，并解析其中所有对/.和/..的引用，从而生成一个以空字符结尾的字符串，内含相应的绝对路径名。

```c++
#include <stdlib.h>

char *realpath(const char *pathname, char *resolved_path);
```



#### 11.解析路径名字符串：dirname() 和 basename()

dirname() 和 basename() 函数将一个路径名字符串分解成目录和文件名两部分。

```c++
#include <libgen.h>

char *dirname(char * pathname);
char *basename(char * pathname);
```

