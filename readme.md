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

通过以上解释，得到如下示意图：

### 简述文件映射的方式如何操作文件。与普通IO区别？为什么写入完成后要调用msync？文件内容什么时候被载入内存？
(使用什么函数，函数的工作流程)  
XXXXXX

### 参考[Intel的NVM模拟教程](https://software.intel.com/zh-cn/articles/how-to-emulate-persistent-memory-on-an-intel-architecture-server)模拟NVM环境，用fio等工具测试模拟NVM的性能并与磁盘对比（关键步骤结果截图）。
（推荐Ubuntu 18.04LTS下配置，跳过内核配置，编译和安装步骤）

### 使用[PMDK的libpmem库](http://pmem.io/pmdk/libpmem/)编写样例程序操作模拟NVM（关键实验结果截图，附上编译命令和简单样例程序）。
（样例程序使用教程的即可，主要工作是编译安装并链接PMDK库）

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
