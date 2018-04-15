### Chapter15.文件属性



#### 1.获取文件信息：stat()

```c++
#include <sys/stat.h>

int stat(const char *pathname, struct stat *statbuf);
int lstat(const char *pathname, struct stat *statbuf);
int fstat(int fd, struct stat *statbuf);

struct stat {
  dev_t st_dev;
  ino_t st_ino;
  mode_t st_mode;
  nlink_t st_nlink;
  uid_t st_uid;
  gid_t st_gid;
  dev_t st_rdev;
  off_t st_size;
  blksize_t st_blksize;
  blkcnt_t st_blocks;
  time_t st_atime;
  time_t st_mtime;
  time_t st_ctime;
};
```

+ stat()会返回所命名文件的相关信息；
+ lstat()与stat()类似，区别在于如果文件属于符号链接，那么所返回的信息针对的是符号链接自身（而非符号链接所指向的文件）。
+ fstat()则会返回由某个打开文件描述服所指代文件的相关信息。



#### 2.使用utime()和utimes()来改变文件时间戳

```c++
#include <utime.h>

int utime(const char *pathname, const struct utimebuf *buf);

struct utimebuf {
  time_t actime;   /* Access time */
  time_t modtime;  /* Modification time */
};
```

futimes()和 lutimes()库函数的功能与utimes()大同小异。前两者与后两者之间的差异在于，用来指定要更改时间戳文件的参数不同。

```c++
#include <sys/time.h>

int futimes(int fd, const struct timeval tv[2]);
int lutimes(const char *pathname, const struct timeval tv[2]);
```



#### 3.改变文件属主：chown()，fchown()和lchown()

系统调用chown()，lchown()和fchown()可用来改变文件的属主（用户ID）和属组（组ID）。

```c++
#include <unistd.h>

int chown(const char *pathname, uid_t owner, gid_t group);

#define _XOPEN_SOURCE 500  /* Or: #define _BSD_SOURCE */
#include <unistd.h>

int lchown(const char *pathname, uid_t owner, gid_t group);
int fchown()
```





#### 4.权限检查算法

检查文件权限时，内核所遵循的规则如下：

1. 对于特权级进程，授予其所有访问权限；
2. 若进程的有效用户ID与文件的用户ID（属主）相同，内核会根据文件的属主权限，授予进程相应的访问权限。比方说，若文件权限掩码中的属主读权限位被置位，则授予进程读权限。否则，则拒绝进程对文件的读取操作。
3. 若进程的有效组ID或任一附属组ID与文件的组ID（属组）相匹配，内核会根据文件的属组权限，授予进程对文件的相应访问权限。
4. 若以上三点皆不满足，内核会根据文件的other(其他)权限，授予进程相应权限。



#### 5.检查对文件的访问权限：access()

系统调用access()就是根据进程的真实用户ID和组ID（以及附属组ID），去检查对pathname参数所指定文件的访问权限。

```c++
#include <unistd.h>

int access(const char * pathname, int mode);
```

参数mode是一堆常量相或(|)而成的位掩码。若由pathname所指定的文件具备mode参数包含的所有权限，access()将返回0；只要有一项权限未得到满足（或者有错误发生），access()则返回-1。



#### 6.文件模式创建掩码：umask()

文件模式创建掩码（umask）会对这些设置进行修改。umask 是一种进程属性，当进程新建文件或目录时，该属性用于指明应屏蔽哪些权限位。

进程的umask通常继承自其父shell，其结果往往正如人们所期望的那样：用户可以使用shell的内置命令umask来改变shell进程的umask，从而控制在shell下运行程序的umask。

大多数shell的初始化文件会将umask默认置为八进制值022。其含义对于同组或其他用户，应总是屏蔽写权限。



#### 7.更改文件权限：chmod()和fchmod()

可利用系统调用chmod()和fchmod()去修改文件权限。

```c++
#include <sys/stat.h>

int chmod(const char * pathname, mode_t mode);

#define _XOPEN_SOURCE 500
#include <sys/stat.h>

int fchmod(int fd, mode_t mode);
```

