### Chapter14.系统编程概念



#### 1.设备文件和设备ID

在目录 /dev 中。



#### 2.磁盘驱动器

尽管现代磁盘速度很快，但读写磁盘信息耗时依然不菲。首先，磁头要移动到相应磁道（寻道时间）；然后，在相应扇区旋转到磁头下之前，驱动器必须一直等待（旋转延迟）；然后，还要从所请求的块上传输数据（传输时间）。执行上述操作所耗费的时间总量通常是以毫秒为单位的。相比之下，同样的时间可供现代cpu执行数百万条指令。



#### 3.文件系统

文件系统是对常规文件和目录的组织集合。

文件系统由以下几部分组成：

+ 引导块；
+ 超级快，包含；
  + i节点表容量；
  + 文件系统中逻辑块的大小；
  + 以逻辑块计，文件系统的大小。
+ i节点表；
+ 数据块。



#### 4.i节点

i节点所维护的信息如下：

+ 文件类型（比如，常规文件，目录，符号链接，以及字符设备等）。
+ 文件属主（亦称为用户ID或UID）；
+ 文件属组（亦称为组ID或GID）。
+ 3类用户的访问权限：属主，属组以及其他用户。
+ 3个时间戳：atime（最后访问时间），mtime（最后修改时间）和ctime（状态最后改变时间）。
+ 指向文件的硬链接数量。
+ 文件的大小，以字节为单位。
+ 实际分配给文件的块数量，以512字节块为单位。
+ 指向文件数据块的指针。



#### 5.挂载文件系统：mount()

mount()系统调用将由source指定设备所包含的文件系统，挂载到由target指定的目录下。

```c++
#include <sys/mount.h>

int mount(const char * source, const char * target, const char * fstype, 
          unsigned long mountflags, const void * data);
```





#### 6.卸载文件系统：umount()和umount2()

umount()系统调用用于卸载已挂载的文件系统

```c++
#include <sys/mount.h>

int umount(const char *target);
```

系统调用umount2()是umount()的扩展版。通过flags参数，umount2()可对卸载操作施以更精密的控制。

```c++
#include <sys/mount.h>

int umount2(const char *target, int flags);
```



