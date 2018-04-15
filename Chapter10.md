### Chapter10.时间

#### 1.真实时间和进程时间

+ 真实时间：度量这一时间的起点有二：一为某个标准点：二为进程生命周期内的某个固定时点（通常为程序启动）。前者为日历时间，适用于需要对数据库记录或文件打上时间戳的程序；后者称之为流逝时间或挂钟时间，主要针对需要周期性操作或定期从外部输入设备进行度量的程序。
+ 进程时间：一个进程所使用的CPU时间总量，适用于对程序，算法性能的检查或优化。



#### 2.计算日历时间

系统调用gettimeofday()，可于tv指向的缓冲区中返回日历时间。

```c++
#include <sys/time.h>

int gettimeofday(struct timeval *tv, struct timezone *tz);

struct timeval
{
  time_t tv_sec;
  suseconds_t tv_usec;
};
```

time()系统调用返回自Epoch以来的秒数。

```c++
#include <time.h>

time_t time(time_t *timep);
```



#### 3.时间转换函数

将time_t 转换为可打印格式

```c++
#include <time.h>

char *ctime(const time_t *timep);
```

time_t 和分解时间之间的转换

```c++
#include <time.h>

struct tm *gmtime(const time_t *timep);
struct tm *localtime(const time_t *timep);

time_t mktime(struct tm *timeptr);
```

分解时间和打印格式之间的转换

```c++
#include <time.h>

char *asctime(const struct tm *timeptr);

size_t strftime(char *outstr, size_t maxsize, const char *format, 
                const struct tm *timeptr);
```

函数strptime()是strftime()的逆向函数，将包含日期和时间的字符串转换成一分解时间。

```c++
#define _XOPEN_SOURCE
#include <time.h>

char *strptime(const char *str, const char *format, struct tm *timeptr);
```

为程序设置地区

```c++
#include <locale.h>

char *setlocale(int category, const char * locale);
```

更新系统时钟

```c++
#define _BSD_SOURCE
#include <sys/time.h>

int settimeofday(const struct timeval *tv, const struct timezone *tz);
```



#### 4.进程时间

**进程时间**是进程创建后使用的CPU时间数量。出于记录的目的，内核把CPU时间分成以下两部分：

+ 用户CPU时间是在用户模式下执行所花费的时间数量。有时也称为虚拟时间，这对于程序来说，是它已经得到CPU的时间。
+ 系统CPU时间是在内核模式中执行所花费的时间数量。这是内核用于执行系统调用或代表程序执行的其他任务（例如：服务页错误）的时间。



系统调用times()，检索进程时间信息，并把结果通过buf指向的结构体返回。

```c++
#include <sys/times.h>

clock_t times(struct tms *buf);
```



通用地，函数clock()提供了一个简单的接口用于取得进程时间。它返回一个值描述了调用进程使用的总的CPU时间。（包括用户和系统）

```c++
#include <time.h>

clock_t clock(void);
```

