### Chapter19.监控文件事件

​        某些应用程序需要对文件或目录进行监控，已侦测其是否发生了特定事件。例如，当把文件加入或移出一目录时，图形化文件管理器应能判定此目录是否在其当前显示之列，而守护进程可能也想要监控自己的配置文件，以了解其是否被修改。

​        自内核2.6.13起，Linux开始提供inotify机制，以允许应用程序监控文件事件。inotify都是Linux专有机制。



#### 1.概述

使用inotify API 有以下几个关键步骤：

1.应用程序使用inotify_init()来创建一 inotify 实例，该系统调用所返回的文件描述符用于在后续操作中指代该实例。

2.应用程序使用inotify_add_watch()向inotify实例的监控列表添加条目，籍此告知内核哪些文件是自己的兴趣所在。每个监控项都包含一个路径名以及相关的位掩码。位掩码针对路径名指明了所要监控的事件集合。作为函数结果，inotify_add_watch()将返回一监控描述符，用于在后续操作中指代该监控项。

3.为获得事件通知，应用程序需针对inotify文件描述符执行read()操作。每次对read()的成功调用，都会返回一个或多个inotify_event结构，其中各自记录了处于inotify实例监控之下的某一路径名所发生的事件。

4.应用程序在结束监控时会关闭inotify文件描述符。这会自动清除与inotify实例相关的所有监控项。



**inotify机制可用于监控文件或目录，且非递归的，并不能监控子目录。**可使用select()，poll()，epoll以及由信号驱动的I/O来监控inotify文件描述符。



#### 2.inotify API

inotify_init() 系统调用可创建一新的 inotify 实例。

```c++
#include <sys/inotify.h>

int inotify_init(void);
```

 作为函数结果，inotify_init()会返回一个文件描述符（句柄），用于在后续操作中指代此inotify 实例。



针对文件描述符 fd 所指代 inotify 实例的监控列表，系统调用 inotify_add_watch() 既可以追加新的监控项，也可以修改现有监控项。

```c++
#include <sys/inotify.h>

int inotify_add_watch(int fd, const char *pathname, uint32_t mask);
```



系统调用 inotify_rm_watch() 会从文件描述符 fd 所指代的 inotify 实例中，删除由wd所定义的监控项。

```c++
#include <sys/inotify.h>

int inotify_rm_watch(int fd, uint32_t wd);
```

参数 wd 是一监控描述符，由之前对 inotify_add_watch() 的调用返回。



#### 3.运用 inotify API 的例子

```c++
#include <sys/inotify.h>
#include <limits.h>
#include "tlpi_hdr.h"

static void displayInotifyEvent(struct inotify_event *i)
{
  printf("	wd =%2d; ", i->wd);
  if (i->cookie > 0)
  {
    printf("cookie =%4d; ", i->cookie);
  }
  
  printf("mask = ");
  if (i->mask & IN_ACCESS)	printf("IN_ACCESS ");
  if (i->mask & IN_ATTRIB)	printf("IN_ATTRIB ");
  if (i->mask & IN_CLOSE_NOWRITE) printf("IN_CLOSE_NOWRITE ");
  if (i->mask & IN_CREATE)	printf("IN_CREATE ");
  if (i->mask & IN_DELETE)	printf("IN_DELETE ");
  if (i->mask & IN_DELETE_SELF)	printf("IN_DELTE_SELF ");
  if (i->mask & IN_IGNORED)	printf("IN_IGNORED ");
  if (i->mask & IN_ISDIR)	printf("IN_ISDIR ");
  if (i->mask & IN_MODIFY)	printf("IN_MODIFY ");
  if (i->mask & IN_MOVE_SELF)	printf("IN_MOVE_SELF ");
  if (i->mask & IN_MOVED_FROM)	printf("IN_MOVED_FROM");
  if (i->mask & IN_MOVED_TO)	printf("IN_MOVED_TO ");
  if (i->mask & IN_OPEN)	printf("IN_OPEN ");
  if (i->mask & IN_Q_OVERFLOW)	printf("IN_Q_OVERFLOW ");
  if (i->mask & IN_UNMOUNT)	printf("IN_UNMOUNT ");
  printf("\n");
  
  if (i->len > 0)
  {
    printf("	name = %s\n", i->name);
  }
}

#define BUF_LEN (10 * (sizeof(struct inotify_event) + NAME_MAX + 1))

int main(int argc, char *argv[])
{
  int inotifyFd, wd, j;
  char buf[BUF_LEN];
  ssize_t numRead;
  char * p;
  struct inotify_event *event;
  
  if (argc < 2 || strcmp(argv[1], "--help") == 0)
  {
    usageErr("%s pathname...\n", argv[0]);
  }
  
  inotifyFd = inotify_init();
  if (inotifyFd == -1)
  {
    errExit("inotify_init");
  }
  
  for (j = 1; j < argc; j++)
  {
    wd = inotify_add_watch(inotifyFd, argv[1], IN_ALL_EVENTS);
    if (wd == -1)
    {
      errExit("inotify_add_watch");
    }
    printf("Watching %s using wd %d\n", argv[j], wd);
  }
  
  for (;;)
  {
    numRead = read(inotifyFd, buf, BUF_LEN);
    if (numRead == 0)
    {
      fatal("read() from inotify fd returned 0!");
    }
    
    if (numRead == -1)
    {
      errExit("read");
    }
    
    printf("Read %ld bytes from inotify fd\n", (long)numRead);
    
    /* Process all of the events in buffer returned by read() */
    
    for (p = buf; p < buf + numRead; )
    {
      event = (struct inotify_event *)p;
      displayInotifyEvent(event);
      
      p += sizeof(struct inotify_event) + event->len;
    }
  }
  
  exit(EXIT_SUCCESS);
}
```





