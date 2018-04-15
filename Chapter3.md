### Chapter3.系统编程概念

#### 1.系统调用

系统调用允许进程向内核请求服务。与用户空间的函数调用相比，哪怕是最简单的系统调用都会产生显著的开销，其原因是为了执行系统调用，系统需要临时性地切换到核心态，此外，内核还需验证系统调用的参数，用户内存和内核内存之间也有数据需要传递。

#### 2.处理系统级错误

系统调用失败时，会将全局整形变量errno 设置为一个正值，以标识具体的错误。程序应包含<errno.h>头文件，该文件提供了对errno的声明，以及一组针对各种错误编号而定义的常量。简单示例如下：

```c++
cnt = read(fd, buf, numbytes);
if (cnt == -1)
{
  if (errno == EINTR)
  {
    fprintf(stderr, "read was interrupted by a signal.\n");
  }
  else
  {
    /* Some other error occured */
  }
}
```



