### Chapter8.用户和组

#### 1.UID和GID

每个用户都拥有一个唯一的用户名和一个与之相关的数值型用户标识符（UID），用户可以隶属一个或多个组。而每个组也都拥有唯一的一个名称和一个组标识符（GID）。



#### 2.密码文本格式/etc/passwd

mtk:s:1000:101:Michael Kerrisk:/home/mtk:/bin/bash

- 登录名；
- 经过加密的密码（如果密码存在shadow文件，只会显示x）；
- 用户ID（UID）；
- 组ID（GID）；
- 主目录；
- 登录shell。



#### 3.shadow 密码文件：/etc/shadow

#### 4.组文件：/etc/group

users:y:101:jambit:y:106:claus,felix,harti,markus

+ 组名；
+ 经过加密处理的密码；
+ 组ID（GID）；
+ 用户列表；



#### 5.获取用户和组的信息

函数 getpwnam() 和 getpwuid() 的作用是从密码文件中获取记录。

```c++
#include <pwd.h>

struct passwd *getpwnam(const char* name);
struct passwd *getpwuid(uid_t uid);

// passwd结构体
struct passwd {
  char *pw_name;  // Login name
  char *pw_passwd;  // Encrypted password
  uid_t pw_uid;  // User ID
  char *pw_gecos;  // Comment (user information)
  char *pw_dir;  // Initial working(home) dir
  char *pw_shell;  // Logic shell
};
```

注意，如果在passwd文件中未发现匹配记录，那么getpwnam() 和getpwuid()将返回NULL，且不会改变errno，因此，要针对查询记录不存在和查询失败进行区分：

```c++
struct passwd *pwd;

errno = 0;
pwd = getpwnam(name);
if (pwd == NULL)
{
  if (errno == 0)
  {
    // Not Found
  }
  else
  {
    // Error
  }
}
```



函数getgrnam() 和 getgrgid() 的作用是从组文件中获取记录。

```c++
#include <grp.h>

struct group *getgrnam(const char* name);
struct group *getgrgid(gid_t gid);

// group 结构体
struct group {
  char *gr_name;  // group name
  char *gr_passwd;  // Encrypted password
  gid_t gr_gid;  // Group ID
  char **gr_mem;
};
```



#### 6.扫描密码文件和组文件中的所有记录

函数setpwent(), getpwent()和endpwent()的作用是按顺序扫描密码文件中的记录。

```c++
#include <pwd.h>

struct passwd *getpwent(void);

void setpwent(void);
void endpwent(void);
```



#### 7.从shadow密码文件中获取记录

```c++
#include <shadow.h>

struct spwd *getspnam(const char *name);
struct spwd *getspent(void);

void setspent(void);
void endspent(void);
```



#### 8.密码加密和用户认证

UNIX系统采用单向加密算法对密码进行加密，加密结果存放于/etc/shadow 中。加密算法封装于crypt()函数之中。

```c++
#define _XOPEN_SOURCE
#include <unistd.h>

char *crypt(const char *key, const char *salt);
```



读取用户密码：

```c++
#define _BSD_SOURCE
#include <unistd.h>

char *getpass(const char *prompt);
```







