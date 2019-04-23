# 论文阅读与前期工作总结
### 姓名：覃伟，汤万鹏，韩俊柠
### 学号：16340200，16340205，16340068
---
## 前期工作

### 使用示意图展示普通文件IO方式(fwrite等)的流程，即进程与系统内核，磁盘之间的数据交换如何进行？为什么写入完成后要调用fsync？
通过博客https://www.cnblogs.com/losing-1216/p/5073051.html

我们得到示意图，如图所示，在文件IO操作中，有fopen，fwrite，fread，fclose。

![image](https://github.com/dbms-19/First-part/blob/master/pic/%E9%A2%981%E7%A4%BA%E6%84%8F%E5%9B%BE.jpg)

**fwrite是系统提供的最上层接口，也是最常用的接口。它在用户进程空间开辟一个CLib buffer，将多次小数据量相邻写操作(application buffer)先缓存起来，合并，最终调用write函数一次性写入（或者将大块数据分解多次write调用）**

**write函数通过调用系统调用接口，将数据从应用层copy到内核层，所以write会触发内核态/用户态切换。当数据到达page cache后，内核并不会立即把数据往下传递。而是返回用户空间。数据什么时候写入硬盘，有内核IO调度决定，所以write是一个异步调用**

**read调用是先检查page cache里面是否有数据，如果有，就取出来返回用户，如果没有，就同步传递下去并等待有数据，再返回用户，所以read是一个同步过程**

**fclose隐含fflush函数,fflush只负责把数据从Clibbuffer拷贝到pagecache中返回，并没有刷新到磁盘上，刷新到磁盘上可以使用fsync函数**

**因为写入完后，fclose并没有将数据刷新到磁盘上，而是将数据刷新到存储介质，fflush函数只是把数据从CLib buffer拷贝到page cache中，并没有刷新到磁盘上，所以调用fsync函数可以将数据刷新到磁盘上**

通过以上解释，得到如下示意图，内核中分页，即page：

![image](https://github.com/dbms-19/First-part/blob/master/pic/%E9%A2%981%E7%A4%BA%E6%84%8F%E5%9B%BE2.jpg)

### 简述文件映射的方式如何操作文件。与普通IO区别？为什么写入完成后要调用msync？文件内容什么时候被载入内存？
通过博客https://www.cnblogs.com/volcao/p/8818199.html 和 https://www.cnblogs.com/alantu2018/p/8506381.html

函数：
- void mmap(void *addr, size_t len, int prot,int flags, int fildes, off_t off) mmap的作用是映射文件描述符和指定文件的(off_t off)区域至调用进程的(addr,addr *len)的内存区域，mmap返回的是用户进程空间的虚拟地址，在stack和heap 之间的空闲逻辑空间(虚拟空间) 就是用来提供映射的，文件将会被映射到这一区域的某块虚拟内存上，具体哪一块若是用户没有指定，则由内核来分配。一般上，用户不该去指定这个映射的起始地址，因为栈和堆都是在向块区域进行扩展的，所以这块区域的大小会一直在变化，若是用户指定，用户根本就无法知道这块地址是否被堆用去了还是被栈用去了。
- int msync(void *addr, size_t len, int flags) 进程在映射空间的对共享内容的改变写回到磁盘文件中，此“冲洗”非彼冲洗，不同于用户缓冲区，此时的冲洗不会洗掉映射存储区的内容，会保留，此冲洗更像是复制写入文件的同步，这也是写入完成后要调用msync的原因，要写入磁盘中。
- int munmap(void *addr, size_t len) 释放存储映射区

**内存映射步骤：**
 - 用open系统调用打开文件, 并返回描述符fd.
 - 用mmap建立内存映射, 并返回映射首地址指针start.
 - 对映射(文件)进行各种操作, 显示(printf), 修改(sprintf).
 - 用munmap(void *start, size_t lenght)关闭内存映射.
 - 用close系统调用关闭文件fd（只要保证在mmap成功了之后就都可以）.
 - msync
 
 **注意事项**
 在修改映射的文件时, 只能在原长度上修改, 不能增加文件长度, 因为内存是已经分配好的.

在如图过程3时，文件内容载入内存。
- 过程1，内存映射。
- 过程2，mmap()会返回一个指针ptr，它指向进程逻辑地址空间中的一个地址，这样以后，进程无需再调用read或write对文件进行读写，而只需要通过ptr就能够操作文件。但是ptr所指向的是一个逻辑地址，要操作其中的数据，必须通过MMU将逻辑地址转换成物理地址，这个过程与内存映射无关。**
- 过程3：建立内存映射并没有实际拷贝数据，这时，MMU在地址映射表中是无法找到与ptr相对应的物理地址的，也就是MMU失败，将产生一个缺页中断，缺页中断的中断响应函数会在swap中寻找相对应的页面，如果找不到（也就是该文件从来没有被读入内存的情况），则会通过mmap()建立的映射关系，从硬盘上将文件读取到物理内存中，这个过程与内存映射无关
- 过程4：如果在拷贝数据时，发现物理内存不够用，则会通过虚拟内存机制（swap）将暂时不用的物理页面交换到硬盘上，这个过程也与内存映射无关。

![image](https://github.com/dbms-19/First-part/blob/master/pic/%E9%A2%98%E7%9B%AE2.jpg)


### 参考[Intel的NVM模拟教程](https://software.intel.com/zh-cn/articles/how-to-emulate-persistent-memory-on-an-intel-architecture-server)模拟NVM环境，用fio等工具测试模拟NVM的性能并与磁盘对比（关键步骤结果截图）。
（推荐Ubuntu 18.04LTS下配置，跳过内核配置，编译和安装步骤）
![image](https://www.baidu.com)

### 使用[PMDK的libpmem库](http://pmem.io/pmdk/libpmem/)编写样例程序操作模拟NVM（关键实验结果截图，附上编译命令和简单样例程序）。
（样例程序使用教程的即可，主要工作是编译安装并链接PMDK库）<br/>
（编译安装PMDK库的过程太过冗长，以至于没有截图，但是有成功运行的截图表明成功安装了PMDK库）
 ![image](/pic/lbpmem.png)
 <br/>
 ```php
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <stdio.h>
#include <errno.h>
#include <stdlib.h>
#ifndef _WIN32
#include <unistd.h>
#else
#include <io.h>
#endif
#include <string.h>
#include <libpmem.h>

/* just copying 4k to pmem for this example */
#define BUF_LEN 4096

int main(int argc, char *argv[])
{
        int srcfd;
        char buf[BUF_LEN];
        char *pmemaddr;
        size_t mapped_len;
        int is_pmem;
        int cc;

        if (argc != 3) {
                fprintf(stderr, "usage: %s src-file dst-file\n", argv[0]);
                exit(1);
        }

        /* open src-file */
        if ((srcfd = open(argv[1], O_RDONLY)) < 0) {
                perror(argv[1]);
                exit(1);
        }

        /* create a pmem file and memory map it */
        if ((pmemaddr = pmem_map_file(argv[2], BUF_LEN,
                                PMEM_FILE_CREATE|PMEM_FILE_EXCL,
                                0666, &mapped_len, &is_pmem)) == NULL) {
                perror("pmem_map_file");
                exit(1);
        }

        /* read up to BUF_LEN from srcfd */
        if ((cc = read(srcfd, buf, BUF_LEN)) < 0) {
                pmem_unmap(pmemaddr, mapped_len);
                perror("read");
                exit(1);
        }
        /* write it to the pmem */
        if (is_pmem) {
                pmem_memcpy_persist(pmemaddr, buf, cc);
        } else {
                memcpy(pmemaddr, buf, cc);
                pmem_msync(pmemaddr, cc);
        }

        close(srcfd);
        pmem_unmap(pmemaddr, mapped_len);

        exit(0);
}
```

---
## 论文阅读

### 总结一下本文的主要贡献和观点(500字以内)(不能翻译摘要)。
（回答本文工作的动机背景是什么，做了什么，有什么技术原理，解决了什么问题，其意义是什么） 

XXXXXX

### SCM硬件有什么特性？与普通磁盘有什么区别？普通数据库以页的粒度读写磁盘的方式适合操作SCM吗？
XXXXXX
### 操作SCM为什么要调用CLFLUSH等指令？
(写入后不调用，发生系统崩溃有什么后果)  
XXXXXX

### FPTree的指纹技术有什么重要作用？
XXXXXX

### 为了保证指纹技术的数学证明成立，哈希函数应如何选取？
（哈希函数生成的哈希值具有什么特征，能简单对键值取模生成吗？）  
XXXXXX

### 持久化指针的作用是什么？与课上学到的什么类似？
XXXXXX
