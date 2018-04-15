### Chapter9.进程凭证

#### 1.实际用户ID和实际组ID

实际用户ID和实际组ID确定了进程所属的用户和组。作为登录过程的步骤之一，登录shell从/etc/passwd 文件中读取相应的用户密码记录的第三和第四字段，置为其实际用户ID和实际组ID。当创建新进程（比如，shell执行依程序）时，将其从父进程中继承这些ID。



#### 2.有效用户ID和有效组ID

在大多数UNIX实现中，当进程尝试执行各种操作（即系统调用）时，将结合有效用户ID，有效组ID，连同辅助组ID一起来确定授予进程的权限。

有效用户ID为0（root的用户ID）的进程拥有超级用户的所有权限。这样的进程又称为特权级进程。而某些系统调用只能有特权级进程执行。

通常，有效用户ID及组ID与其相应的实际ID相等，但有两种方法能够致使二者不同。其一是使用系统调用，其二是执行set-user-ID 和 set-group-ID程序。



#### 3.Set-User-ID 和 Set-Group-ID 程序

set-user-ID程序会将进程的有效用户ID置为可执行文件的用户ID（属主），从而获得常规情况下并不具有的权限。

Linux系统中经常使用的set-user-ID程序包括:passwd,用于更改用户密码；mount和umount，用于加载和卸载文件系统；su，允许用户以另一用户的身份运行shell。



#### 4.获取和修改实际，有效和保存设置标识

获取实际和有效ID

```c++
#include <unistd.h>

uid_t getuid(void);  // 返回调用进程的实际用户ID和组ID

uid_t geteuid(void);  // 返回进程实际有效用户id

gid_t getgid(void);  

gid_t getegid(void);

```

修改进程的实际id

```c++
#include <unistd.h>

int setuid(uid_t uid);
int setgid(gid_t gid);
```

修改进程的有效用户id

```c++
#include <unistd.h>

int seteuid(uid_t euid);
int setegid(gid_t egid);
```

修改实际ID和有效ID

```c++
#include <unistd.h>

int setreuid(uid_t ruid, uid_t euid);
int setregid(gid_t rgid, gid_t egid);
```

这两个系统调用的第一个参数都是新的实际ID，第二个参数都是新的有效ID。若只想修改其中的一个ID，可以将另外一个参数指定为-1。



获取实际，有效和保存设置ID

在大多数UNIX实现中，进程不能直接获取（或修改）其保存set-user-ID 和保存set-group-ID的值。然而，Linux提供了两个（非标准的）系统调用来实现此项功能：getresuid()和getresgid()。

```c++
#define _GNU_SOURCE
#include <unistd.h>

int getresuid(uid_t *ruid, uid_t *suid);
int getresgid(gid_t *rgid, gid_t *sgid);
```



修改实际，有效和保存设置ID

```c++
#define _GNU_SOURCE
#include <unistd.h>

int setresuid(uid_t *ruid, uid_t *euid, uid_t *suid);
int setresgid(gid_t *rgid, gid_t *egid, gid_t *sgid);
```



#### 5.获取和修改文件系统ID

```c++
#include <sys/fsuid.h>

int setfsuid(uid_t fsuid);
int setfsgid(gid_t fsgid);
```



#### 6.获取和修改辅助组ID

```c++
#include <unistd.h>

int getgroups(int gidsetsize, gid_t grouplist);
```







