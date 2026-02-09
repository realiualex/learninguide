## cpu
CPU 有 C-State 和 P-State。C-State指的是能控制 cpu的空闲状态。从C0（完全活跃）

## 内存管理

### 硬件知识

* 内存条、总线与DMA

与计算机的交互，包括鼠标键盘显卡等，都是通过总线进行交互的，总线包括有：数据总线，IO总线（比如USB，PCIE总线），控制总线等。CPU和内存条之间是通过数据总线和地址总线进行直接相连。
南桥：南桥负责USB，硬盘，网卡，声卡等不太占用带宽的设备的IO。南桥除了直接控制这些设备之外，还集成了DMA控制器(DMAC)。

看一个例子：NodeJS是单线程，但文件没有读写完成之后，却能执行其他的任务，当文件读写完成之后，再进行一个回调继续操作。但正常来说，对于单线程的任务，应该一直要等到文件读写完成之后才能进行别的操作。但实际为什么没有这样呢？这是因为有DMAC硬件才完成此操作的。
看一下读文件过程：CPU发起一个读文件的请求，然后把这个操作交给DMA控制器来完成，DMA控制器可以直接访问内存，将磁盘中读取的文件放到内存中，之后以中断的形式通知CPU。因此当CPU把任务给DMA控制器之后，CPU就可以干其他事情了，节省了CPU占用的时间。
DMA控制器为什么能做到直接控制硬件呢？CPU是因为有数据总线和地址总线，因此可以直接控制硬件。当cpu把任务交给DMA控制器的时候，顺便会把总线的控制权也给到了DMA控制器，因而DMA控制器也能直接对硬件进行操作，但此时CPU就不能操作了。不过如果一直这样的话，那么在读写文件的时候，CPU就无法操作鼠标键盘了。但实际情况并非如此，这是因为在DMA控制器读写文件的时候，并非一直占有总线的控制权，而是只占有了部分的时间，比如可能只占有了1ms，下1ms就又把控制权交回给了CPU，CPU进行了一些其他操作，之后再1ms又给了DMA控制器，会这样不停的切换总线控制权，对用户来说是无感知的。

### 从操作系统角度看内存管理

操作系统是通过虚拟内存地址进行管理的，程序看到的是连续的虚拟地址，但实际上映射到物理地址上可能是不连续的，散落在各个页中。这个映射表叫做页表 PageTable。另外，虚拟地址可以是非常大的，比如物理地址只有1G，但虚拟地址可以是1TB。

* Linux操作系统中的内存管理
使用free查看Linux内存使用情况，示例如下:

```
 » free -m                                                                alex@LAPTOP-FB6FQ8MU
              total        used        free      shared  buff/cache   available
Mem:          12579         184       12108           0         286       12150
Swap:          4096           0        4096
```

其中Swap是映射到磁盘中的虚拟内存，free 12108M和available 12150不一样，那么实际可用的是多少呢？答案是12150M。free是真正的空闲，buff/cache是对磁盘或文件进行缓存的，为了加速对磁盘或文件的访问速度，这一部分内存不是强制被保留的，当另外某个程序需要调用大量内存的时候，就有可能会清掉buff/cache里面的内存，available是真正程序可使用的内存。因此一般要看可用内存多大，一定要看available，看free没有太大的意义。另外，对于used是包含了Shared。

在早期的内核中，buff/cache是分开的，buffer cache和page cache，现在的内核中已经把buffer和page cache放在一起了。buffer指的是对设备如磁盘的缓存，从磁盘角度看的，page cache指的是对文件的缓存，从文件系统的角度看的。更详细的解释可以看：http://lday.me/2019/09/09/0023_linux_page_cache_and_buffer_cache/

* Windows 系统下的内存管理

在任务管理器中观察的内存解释如下：
**使用中（已压缩）**：真正被使用了多少内存
**可用**：真正可用内存（不含被缓存的，比如不包含linux的buffer/cache）
**已提交（X/Y）**：Y指的是物理内存+虚拟内存(pagefile.sys)。应用程序提交申请的内存X，并不代表说操作系统一定会分配这么多内存。比如C语言的malloc申请了1G内存，但实际上操作系统可能只分了1M。已提交的内存可能会大于实际操作系统内存大小。
已缓存：类似linux的buffer/cache
**分页缓冲池**：分页缓冲池指的是可以映射到磁盘的。（不是很重要）
**非分页缓冲池**：无法映射到磁盘的，只能存在物理磁盘中。（不是很重要）

Windows内存管理参考文档：https://docs.microsoft.com/zh-cn/archive/blogs/markrussinovich/pushing-the-limits-of-windows-paged-and-nonpaged-pool


### 与内存相关的系统调用

* 用户态调用内核态的几种方式：1. 系统调用，2. 中断，3.异常
malloc 本质上是一个库函数，不是直接的系统调用，在申请128K以下的内存，调用的是brk这个系统调用，申请128K以上的内存，通过mmap系统调用。

**测试一：**
brk在C语言中，有一个sbrk的库函数，可以用sbrk申请内存。示例如下：
先申请0个字节给first，然后再申请1个字节给second，再申请0个地址给third。
```
#include <stdio.h>
#include <unistd.h>
int main(){
    void *first = sbrk(0);
    void *second = sbrk(1);
    void *third = sbrk(0);
    printf("%p\n",first);
    printf("%p\n",second);
    printf("%p\n",third);
}
```
将文件保存成1B_memory.c, 然后执行  gcc 1B_memory.c -o 1B_memory 进行编译，编译之后默认文件名为 1B_memory，在shell里面执行 .\1B_memory 可以看到地址如下：
```
~ » ./1B_memory
0x55c39bc75000
0x55c39bc75000
0x55c39bc75001
```
从这个测试上看，brk申请的内存地址是连续的，brk提高了heap的上界。

**测试二：**
使用gcc编译一下文件，然后直接执行，可以正常输出结果123.
```
#include <stdio.h>
#include <unistd.h>
int main(){
    int *first=(int *)sbrk(1);
    *(first+1) = 123;
    printf("%d\n",*(first+1));
}
```
代码解释：先申请1个字节的内存地址空间，然后将指针转换为int类型的指针，first代表1个int值。first+1相当于往后移动了4个字节，也就是相当于现在位于第5个字节的位置（first代表int类型，int类型是4个字节，+1的话就是加1个int，即加4个字节），*(first+1)代表的是对第5--8个字节进行赋值，从执行结果中并没有报错。这是因为操作系统对内存的分页管理导致的，操作系统申请内存的最小单位是1页，一般系统中页大小是4k，因而brk看起来申请的是1个字节，实际是申请的是4k即4096个字节，第5-8个字节属于4k以内，所以不会报错。

**测试三：**
```
#include <stdio.h>
#include <unistd.h>
int main(){
    int *first=(int *)sbrk(1);
    *first -1;
    *(first+1024) = 123;
    printf("%d",*(first+1024));
}
```
代码解释：这次移动了4096个字节(first代表int类型，占4个字节，4个1024就是4096)，所以这次就报错了。

**测试四：**
使用mmap来申请128K以上的内存。
mmap使用的说明:
//addr 传NULL则不关心起始地址，关心地址的话，应该传个4k的倍数，不然也会归到4k背书的起始地址。
viod *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
//释放内存munmap
int munmap(void *addr, size_t length);

* 代码示例
```
#include <stdio.h>
#include <unistd.h>
#include <sys/mman.h>
int main(){
    int* a =(int *) mmap(NULL, 100 * 4096, PROT_READ| PROT_WRITE, MAP_PRIVATE| MAP_ANONYMOUS, -1, 0);
    int* b =a;
    for(int i=0; i < 100; i++){
        b=(void *)a + (i*4096);
        *b =1;
    }
    while(1){
        sleep(1);
    }
}
```
代码说明：fd是文件描述符，因为mmap是将文件映射到内存，因为这里不需要文件，所以写-1就是直接申请内存了。申请内存的时候，因为fd为-1，也不需要offset，因而offset为0。 100* 4096 代表申请了100页的内存（1页为4k），然后对这100页中的每一页都进行赋值(不赋值的话，虽然申请了，但操作系统并不会实际给到程序这么多内存)。比如这个代码中，将每一页的第一个地址都复制为1，所以操作系统会在每一次的时候都开辟出这一页。
