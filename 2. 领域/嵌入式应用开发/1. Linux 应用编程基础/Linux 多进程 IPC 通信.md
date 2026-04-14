#嵌入式应用开发

# 1. 进程和线程的区别

- **进程是指正在运行的程序**，它拥有独立的内存空间和系统资源，不同进程之间的数据不共享。**进程是资源分配的基本单位**。
- **线程是进程内的执行单元**，它与同一进程内的其他线程共享进程的内存空间和系统资源。**线程是调度的基本单位**。

下面用图来对比两者的关系——进程之间相互隔离，而同一进程内的线程共享地址空间：

![[resource/a88a81ad93bc17917a8e0f81b99d9af1.png]]

# 2. 进程间通信方式

**IPC （Inter-Process Communication）**

每个进程各自有不同的用户地址空间，任何一个进程的全局变量在另一个进程中都看不到，进程间通信是指在不同进程之间传播或交换信息的一种机制。

进程间通信的目的：
- **传输数据**。比如进程 A 负责生成数据，进程 B 负责处理数据，数据需要从 A 进程传输至 B 进程。
- **共享资源**。比如进程 A 与进程 B 共享某一块内存资源。
- **模块化**。将系统功能划分为多个进程模块进行开发，方便开发维护。
- **加速计算**。多核处理器环境，一个特定进程划分为几个进程并行运行。

Linux IPC（Inter-process Communication， 进程间通信）的方式：

![[resource/7d534e13e28b3e0934a87c3c4e1f3fcf.png]]

下面先通过一张对比表快速了解各种IPC方式的特点，再逐一深入讲解：

| IPC方式     | 传输速度 | 实现复杂度        | 数据拷贝次数 | 适用场景         |
| --------- | ---- | ------------ | ------ | ------------ |
| 共享内存      | 最快   | 较高（需要配合同步机制） | 0      | 大数据量、高频交互    |
| UNIX 域套接字 | 较快   | 中等           | 2      | C/S 架构、双向通信  |
| 消息队列      | 中等   | 中等           | 2      | 结构化消息、异步传递   |
| 管道        | 中等   | 较低           | 2      | 简单的父子/相关进程通信 |
| 信号量       | ——   | 较低           | ——     | 同步互斥（非数据传输）  |
| 信号        | ——   | 较低           | ——     | 事件通知（非数据传输）  |

## 2.1 管道

前面介绍的消息队列、共享内存和UNIX域套接字功能强大但使用相对复杂。如果只是父子进程间简单的数据传递，管道（Pipe）是最轻量的选择。

在内核中开辟一块缓冲区；若多个进程拿到同一个管道(缓冲区)的操作句柄，就可以访问同一个缓冲区，就可以进行通信。涉及到两次用户态与内核态之间的数据拷贝。

#### （1）匿名管道

内核中的缓冲区是没有具体的标识符的，匿名管道只能用于具有亲缘关系的进程间通信。

调用pipe接口可以创建一个匿名管道，并返回了两个描述符，一个是管道的读取端描述符 `fd[0]`，另一个是管道的写入端描述符 `fd[1]`。

> 管道是一个**半双工通信**(可以选择方向的单向传输)

匿名管道的工作原理——父进程 fork 出子进程后，两者共享管道的文件描述符：

![[resource/0b9bd871e6e583b46a5db95ebf327519.png]]

匿名管道基本API接口使用例子：父进程通过管道发送测试数据给子进程。

```
#include <stdio.h>  
#include <stdlib.h>  
#include <unistd.h>  
#include <string.h>  
  
int main()  
{  
    ///< 创建管道  
    int pipefd[2] = {-1};  
    int ret = pipe(pipefd);  
    if (ret < 0)  
    {  
       printf("pipe error\n");  
       exit(EXIT_FAILURE);  
    }  
  
    int read_fd = pipefd[0];   ///< pipefd[0] 用于从管道中读取数据  
    int write_fd = pipefd[1];  ///< pipefd[1] 用于向管道中写入数据  
  
    ///< 创建子进程  
    pid_t pid = fork();  
    if (pid == 0)  
    {  
        ///< 子进程从管道读取数据    
        char buf[128] = {0};  
        read(read_fd, buf, sizeof(buf));  
        printf("child recv data from father: %s", buf);  
    }  
    elseif (pid > 0)  
   {  
        ///< 父进程向管道写入数据  
        char *ptr = "hello88888888\n";  
        write(write_fd, ptr, strlen(ptr));  
    }  
    return 0;  
}
```

编译、运行：

![](https://mmbiz.qpic.cn/sz_mmbiz_png/K62vJYPSiagg5JzRZibFCEXlyXStfloosskOz25DAGkKNqCr4jm2T5sCbmD0PUsczQtbB04xoEqePohnrVeZvWJLumw7SdTYQJia47PQsPAOEY/640?wx_fmt=png&from=appmsg)

> 如果需要双向通信，则应该创建两个管道。

#### （2）命名管道

命名管道也是内核中的一块缓冲区，并且这个缓冲区具有标识符；这个标识符是一个可见于文件系统的管道文件，能够被其他进程找到并打开管道文件，则可以获取管道的操作句柄，所以该命名管道可用于同一主机上的任意进程间通信。

**命名管道与匿名管道的关键区别**——命名管道通过文件系统中可见的 FIFO 文件来建立连接，任意进程都能访问：

![[resource/Pasted image 20260410100653.png]]

创建命名管道的接口：

```
int mkfifo(const char *pathname, mode_t mode);
```

命名管道基本API接口使用例子：一个进程往管道中写入测试数据，另一个进程从管道中读取数据。

**fifo_wr.c：**

```
#include <stdio.h>  
#include <stdlib.h>  
#include <string.h>  
#include <fcntl.h>  
#include <sys/stat.h>  
#include <unistd.h>  
#include <errno.h>  
  
#define FIFO_PATH  "./fifo_file"  
  
typedef struct _msg_data  
{  
    char buf[128];  
    int cnt;  
}msg_data_t;  
  
void send_data(int fd)  
{  
    staticint cnt = 0;  
    msg_data_t send_data = {0};  
  
    cnt++;  
    strcpy(send_data.buf, "hello");  
    send_data.cnt = cnt;  
    write(fd, &send_data, sizeof(send_data));  
    printf("send msg = %s, cnt = %d\n", send_data.buf, send_data.cnt);  
}  
  
int main(void)  
{  
    ///< 创建管道文件  
    int ret = mkfifo(FIFO_PATH, 0664);  
    if (ret < 0 && errno != EEXIST)  
    {  
        printf("mkfifo error\n");  
        exit(EXIT_FAILURE);  
    }  
  
    ///< 以只写的方式打开管道文件  
    int fd = open(FIFO_PATH, O_WRONLY);  
    if (fd < 0)  
    {  
        printf("open fifo error\n");  
        exit(EXIT_FAILURE);  
    }  
    printf("open fifo success\n");  
      
    ///< 写10次  
    for (size_t i = 0; i < 10; i++)  
    {  
        send_data(fd);  
        sleep(1);  
    }  
      
    close(fd);  
  
    return 0;  
}
```

**fifo_rd.c：**

```
#include <stdio.h>  
#include <stdlib.h>  
#include <unistd.h>  
#include <string.h>  
#include <sys/stat.h>  
#include <errno.h>  
#include <fcntl.h>  
  
#define FIFO_PATH  "./fifo_file"  
  
typedef struct _msg_data  
{  
    char buf[128];  
    int cnt;  
}msg_data_t;  
  
int main(void)  
{  
    umask(0);  
    ///< 创建管道文件  
    int ret = mkfifo(FIFO_PATH,0664 );  
    if (ret < 0 && errno != EEXIST)  
    {  
        printf("mkfifo error\n");  
        exit(EXIT_FAILURE);  
    }  
  
    ///< 以只读方式获取管道文件的操作句柄  
    int fd = open(FIFO_PATH, O_RDONLY);  
    if (fd < 0)  
    {  
        printf("open error\n");  
        exit(EXIT_FAILURE);  
    }  
    printf("open fifo success\n");  
  
    while(1)  
    {  
        msg_data_t read_data = {0};  
        ///< 将从管道读取的文件写到buf中  
        int ret = read(fd, &read_data, sizeof(read_data));  
        if (ret < 0)  
        {  
            printf("read error\n");  
            exit(EXIT_FAILURE);  
        }  
        elseif (ret == 0)  
        {  
            printf("all write closed\n");  
            exit(EXIT_FAILURE);  
        }  
        printf("read_data = %s, cnt = %d\n", read_data.buf, read_data.cnt);  
        sleep(1);  
    }  
    close(fd);  
  
    return 0;  
}
```

编译、运行：

```
gcc fifo_wr.c -o fifo_wr
gcc fifo_rd.c -o fifo_rd
```

![[resource/423c81c3bf9dfd5d9264767e3aa0efa6.png]]

## 2.2 消息队列

内核中的一个优先级队列，多个进程通过访问同一个队列，进行添加结点或者获取结点实现通信。

下图展示了消息队列的工作原理——发送方将消息按优先级插入队列，接收方从队列头部取出消息：

![[resource/bf1e511a5a8b0c57fae3e591b0319165.png]]

POSIX消息队列头文件：
```
#include <fcntl.h>           /* For O_* constants */  
#include <sys/stat.h>        /* For mode constants */  
#include <mqueue.h>
```
> 编译链接需要加上 `-lrt` 链接。

消息队列基本API接口使用例子：发送进程给接收进程发送测试数据。

本例子消息队列通信的完整流程如下：

![[resource/Pasted image 20260409205650.png]]

**send.c：**
```
#include <stdio.h>  
#include <stdlib.h>  
#include <string.h>  
#include <unistd.h>  
#include <fcntl.h>           /* For O_* constants */  
#include <sys/stat.h>        /* For mode constants */  
#include <mqueue.h>  
  
#define MQ_MSG_MAX_SIZE    512   ///< 最大消息长度   
#define MQ_MSG_MAX_ITEM    5     ///< 最大消息数目  
  
staticmqd_t s_mq;  
  
typedef struct _msg_data  
{  
    char buf[128];  
    int cnt;  
}msg_data_t;  
  
  
void send_data(void)  
{  
    staticint cnt = 0;  
    msg_data_t send_data = {0};  
  
    cnt++;  
    strcpy(send_data.buf, "hello");  
    send_data.cnt = cnt;  
    int ret = mq_send(s_mq, (char*)&send_data, sizeof(send_data), 0);  
    if (ret < 0)  
    {  
        perror("mq_send error");  
        return;  
    }  
    printf("send msg = %s, cnt = %d\n", send_data.buf, send_data.cnt);  
}  
  
  
int main(void)  
{  
    int ret = 0;  
    struct mq_attr attr;  
  
    ///< 创建消息队列  
    memset(&attr, 0, sizeof(attr));  
    attr.mq_maxmsg = MQ_MSG_MAX_ITEM;  
    attr.mq_msgsize = MQ_MSG_MAX_SIZE;  
    attr.mq_flags = 0;  
    s_mq = mq_open("/mq", O_CREAT|O_RDWR, 0777, &attr);  
    if(-1 == s_mq)  
    {  
        perror("mq_open error");  
        return-1;  
    }  
  
    for (size_t i = 0; i < 10; i++)  
    {  
        send_data();  
        sleep(1);  
    }  
  
    mq_close(s_mq);  
  
    return 0;  
}
```

**recv.c：**
```
#include <stdio.h>  
#include <stdlib.h>  
#include <string.h>  
#include <unistd.h>  
#include <fcntl.h>           /* For O_* constants */  
#include <sys/stat.h>        /* For mode constants */  
#include <mqueue.h>  
  
#define MQ_MSG_MAX_SIZE    512   ///< 最大消息长度   
#define MQ_MSG_MAX_ITEM    5     ///< 最大消息数目  
  
staticmqd_t s_mq;  
  
typedef struct _msg_data  
{  
    char buf[128];  
    int cnt;  
}msg_data_t;  
  
int main(void)  
{  
    msg_data_t recv_data = {0};  
    int prio = 0;  
    ssize_t len = 0;  
  
    s_mq = mq_open("/mq", O_RDONLY);  
    if(-1 == s_mq)  
    {  
        perror("mq_open error");  
        return-1;  
    }  
  
    while (1)  
    {  
        if((len = mq_receive(s_mq, (char*)&recv_data, MQ_MSG_MAX_SIZE, &prio)) == -1)  
        {  
            perror("mq_receive error");  
            return-1;  
        }  
        printf("recv_msg = %s, cnt = %d\n", recv_data.buf, recv_data.cnt);  
        sleep(1);  
    }  
  
    mq_close(s_mq);  
    mq_unlink("/mq");  
  
    return 0;  
}
```

**编译、运行：**
`gcc send.c -o send_process -lrt`  
`gcc recv.c -o recv_process -lrt`

![[resource/Pasted image 20260409210054.png]]

## 2.3 共享内存

消息队列的读取和写入的过程，会有发生用户态与内核态之间的消息拷贝过程。而共享内存的方式则没有这个拷贝过程，进程间通信速度较快。

在物理内存上开辟一块内存空间，多个进程可以将同一块物理内存空间映射到自己的虚拟地址空间，通过自己的虚拟地址直接访问这块空间，通过这种方式实现数据共享。

下图展示了共享内存的核心原理——**多个进程的虚拟地址通过页表映射到同一块物理内存，实现零拷贝通信**：

![[resource/Pasted image 20260409210231.png]]

对比消息队列需要两次数据拷贝，共享内存的优势一目了然：

![[resource/Pasted image 20260409210251.png]]

POSIX共享内存头文件：
```
#include <sys/mman.h>  
#include <sys/stat.h>  
#include <unistd.h>
```

共享内存基本API接口使用例子：发送进程给接收进程发送测试数据。

**send.c：**
```
#include <stdio.h>  
#include <stdlib.h>  
#include <string.h>  
#include <unistd.h>  
#include <fcntl.h>           /* For O_* constants */  
#include <sys/stat.h>        /* For mode constants */  
#include <sys/mman.h>  
#include <semaphore.h>  
  
#define SHM_NAME "/shm"  
#define SEM_NAME "/shm_sem"  
  
int main(void)  
{  
    ///< 创建信号量，初始值为0，表示数据尚未就绪  
    sem_t *sem = sem_open(SEM_NAME, O_CREAT, 0666, 0);  
    if (sem == SEM_FAILED)  
    {  
        perror("sem_open error");  
        return-1;  
    }  
  
    ///< 创建和读端相同的文件标识  
    int shm_fd = shm_open(SHM_NAME, O_RDWR | O_CREAT, 0666);  
    if (shm_fd == -1)   
    {  
        perror("shm_open error");  
        return-1;  
    }  
  
    ///< 设置共享内存文件为8KB  
    ftruncate(shm_fd , 8 * 1024);  
  
    ///< 获取共享内存文件相关属性信息  
    struct stat filestat = {0};  
    fstat(shm_fd, &filestat);  
    printf("st_size = %ld\n",filestat.st_size);  
  
    ///< 内存映射  
    char *shm_ptr = (char*)mmap(NULL, filestat.st_size, PROT_READ|PROT_WRITE, MAP_SHARED, shm_fd, 0);  
    close(shm_fd);  
  
    ///< 向共享内存中写入数据  
    char buf[] = "hello world";  
    memmove(shm_ptr, buf, sizeof(buf));  
    printf("pid %d, %s\n", getpid(), shm_ptr);  
  
    ///< 写入完成，通知接收方数据已就绪  
    sem_post(sem);  
  
    ///< 解除映射，关闭信号量  
    munmap(shm_ptr, filestat.st_size);  
    sem_close(sem);  
  
    return 0;  
}
```

**recv.c：**
```
#include <stdio.h>  
#include <stdlib.h>  
#include <string.h>  
#include <unistd.h>  
#include <fcntl.h>           /* For O_* constants */  
#include <sys/stat.h>        /* For mode constants */  
#include <sys/mman.h>  
#include <semaphore.h>  
  
#define SHM_NAME "/shm"  
#define SEM_NAME "/shm_sem"  
  
int main(void)  
{  
    ///< 打开信号量  
    sem_t *sem = sem_open(SEM_NAME, O_CREAT, 0666, 0);  
    if (sem == SEM_FAILED)  
    {  
        perror("sem_open error");  
        return-1;  
    }  
  
    ///< 创建共享内存文件标识符  
    int shm_fd = shm_open(SHM_NAME, O_RDWR | O_CREAT, 0666);  
    if (shm_fd == -1)   
    {  
        perror("shm_open failed");  
        exit(EXIT_FAILURE);  
    }  
  
    ///< 设置共享内存文件为8KB  
    ftruncate(shm_fd , 8192);  
  
    ///< 获取共享内存文件相关属性信息  
    struct stat filestat;  
    fstat(shm_fd, &filestat);  
    printf("st_size = %ld\n",filestat.st_size);  
  
    ///< 映射共享内存，并获取共享内存的地址  
    char *shm_ptr = (char*)mmap(NULL, filestat.st_size, PROT_READ|PROT_WRITE, MAP_SHARED, shm_fd, 0);  
    close(shm_fd);  
  
    ///< 等待发送方写入完成  
    sem_wait(sem);  
  
    ///< 获取共享内存地址中的内容并打印  
    printf("pid = %d, %s\n", getpid(), shm_ptr);  
  
    ///< 解除映射，清理信号量和共享内存  
    munmap(shm_ptr, filestat.st_size);  
    shm_unlink(SHM_NAME);  
    sem_close(sem);  
    sem_unlink(SEM_NAME);  
  
    return 0;  
}
```

**编译、运行：**
`gcc send.c -o send_process -lrt`  
`gcc recv.c -o recv_process -lrt`

![[resource/cd9e31d3264443b874bb56d4da93cb65.png]]

对具有多个处理核系统消息传递的性能要优于共享内存。共享内存会有高速缓存一致性问题，这是由共享数据在多个高速缓存之间迁移而引起的。随着系统的处理核的数量的日益增加，可能导致消息传递作为 IPC 的首选机制。

**注意：** 共享内存**本身不提供任何同步机制**。上面的示例使用有名信号量（初始值为0）实现了"写完再读"的同步：发送方写入后 `sem_post` 将信号量加1，接收方 `sem_wait` 阻塞直到信号量大于0才读取。生产代码中如果涉及多读多写的并发场景，还需要进一步使用互斥锁保护临界区。

## 2.4 信号量

前面介绍的IPC方式都是用于进程间传输数据的，但在多进程协作中还有一个关键问题——**同步与互斥**。信号量正是解决这个问题的利器。

信号量（Semaphore）是进程和线程间同步的一种机制。

信号量本质是一个非负的整型变量。增加一个可用资源执行加一，也称为**V操作**；获取一个资源资源后执行减一，也称为**P操作**。

下图展示了信号量的 P/V 操作原理——以一个停车场为例来理解信号量：

![[resource/Pasted image 20260410101129.png]]

信号量根据信号值不同可分为两类：

- 二值信号量，信号量值只有0和1，初始值为1，1表示资源可用，0表示资源不可用；二值信号量与互斥锁类似。
- 计数信号量， 信号量的值在0到一个大于1的限制值之间，信号值表示可用的资源的数目。

信号量根据作用对象不同可分为两类：

- 有名信号量，信号值保存在文件中，用于进程间同步
- 无名信号量，又称为基于内存信号量，信号值保存在内存中，用于线程间同步

POSIX信号量头文件：

```
#include <semaphore.h>
```

> 编译链接需要加-lpthread参数。

信号量基本API接口使用例子：父子进程间通信

```
#include <stdio.h>  
#include <stdlib.h>  
#include <unistd.h>  
#include <semaphore.h>  
#include <fcntl.h>  
  
#define SEM_NAME "sem"  
  
int main (void)  
{  
    int sem_val = 0;  
  
    ///< 创建信号量  
    sem_t *sem = sem_open(SEM_NAME, O_CREAT, 0666, 1);  
    if (NULL == sem)  
    {  
       printf("sem_open error\n");  
       exit(EXIT_FAILURE);   
    }  
  
    ///< 创建子进程  
    pid_t pid = fork();  
    if (pid == -1)   
    {  
        printf("fork error\n");  
        sem_close(sem);  
        sem_unlink(SEM_NAME);  
        exit(EXIT_FAILURE);  
    }  
    elseif(pid == 0)   
    {  
        ///< 子进程进行5次P操作  
        for (size_t i = 0; i < 5; i++)  
        {  
            sem_wait(sem);  
            if (sem_getvalue(sem, &sem_val) != -1)   
            {  
                printf("child process P operation, sem_val = %d\n", sem_val);  
                sleep(1);  
            }  
        }  
        _exit(1);  
    }  
    elseif (pid > 0)  
    {  
        ///< 父进程执行5次V操作  
        for (size_t i = 0; i < 5; i++)  
        {  
            sem_post(sem);  
            if (sem_getvalue(sem, &sem_val) != -1)   
            {  
                printf("parent process V operation, sem_val = %d\n", sem_val);  
                sleep(2);  
            }  
        }  
    }  
  
    ///< 删除sem信号量  
    sem_close(sem);  
    if (sem_unlink(SEM_NAME) != -1)   
    {  
        printf("sem_unlink success\n");  
    }  
  
    return 0;  
}
```

编译、运行：

![[resource/5e0726d8749b0c49093f4113f94fa524.png]]

## 2.5 套接字（Socket）

消息队列和共享内存适合数据传输，但如果需要像网络编程一样灵活的双向通信，UNIX域套接字是更好的选择。

UNIX域套接字与传统基于TCP/IP协议栈的socket不同，unix domain socket以文件系统作为地址空间，不需经过TCP/IP的头部封装、报文ack确认、路由选择、数据校验与重传过程，因此传输速率上也不会受网卡带宽的限制。

unix domain socket 在进程间通信同样是基于“客户端—服务器”（C-S）模式。

UNIX 域套接字与 TCP/IP 套接字的对比：

![[resource/Pasted image 20260409210806.png]]

UNIX域套接字基本API接口使用例子：基于UNIX域套接字客户端进程向服务端进程发送测试数据。

下图展示了 UNIX 域套接字的 C/S 通信建立过程：

![[resource/Pasted image 20260409210853.png]]

**server.c：**
```
#include <stdio.h>  
#include <stdlib.h>  
#include <string.h>  
#include <unistd.h>  
#include <fcntl.h>           /* For O_* constants */  
#include <sys/stat.h>        /* For mode constants */  
#include <sys/socket.h>  
#include <netinet/in.h>  
#include <arpa/inet.h>  
#include <sys/un.h>  
  
#define SERVER_PATH "/tmp/server"   
  
int main(void)  
{  
    ///< 创建UNIX域字节流套接字  socket(AF_LOCAL, SOCK_DGRAM, 0)
    int server_fd = socket(AF_LOCAL, SOCK_STREAM, 0);  
    if(server_fd < 0)  
    {  
        printf("socket error\n");  
        exit(EXIT_FAILURE);  
    }  
  
    ///< 绑定服务端地址  
    unlink(SERVER_PATH);  
    struct sockaddr_un server_addr;  
    memset((char*)&server_addr, 0, sizeof(server_addr));  
    server_addr.sun_family = AF_LOCAL;  
    strncpy(server_addr.sun_path, SERVER_PATH, sizeof(server_addr.sun_path)-1);  
    if(bind(server_fd, (struct sockaddr*)&server_addr, sizeof(server_addr)) < 0)  
    {  
        printf("bind error\n");  
        close(server_fd);  
        exit(EXIT_FAILURE);  
    }  
  
    ///< 监听  
    if(listen(server_fd, 10) < 0)     
    {  
        printf("listen error\n");  
        close(server_fd);  
        exit(EXIT_FAILURE);  
    }  
  
    ///< 等待客户端连接  
    int addr_len = sizeof(struct sockaddr);  
    struct sockaddr_un client_addr;  
    int client_fd = accept(server_fd, (struct sockaddr*)&client_addr, (socklen_t *)&addr_len);  
    if(client_fd < 0)  
    {  
        printf("accept error\n");  
        close(server_fd);  
        unlink(SERVER_PATH);  
        exit(1);   
    }  
    else  
    {  
        printf("connected client: %s\n", client_addr.sun_path);  
    }  
  
    while(1)  
    {  
        char buf[128] = {0};  
  
        int recv_len = read(client_fd, buf, sizeof(buf));   
        if(recv_len <= 0)  
        {  
            printf("recv error!\n");  
            close(client_fd);  
            exit(EXIT_FAILURE);  
        }  
  
        printf("recv : %s\n", buf);  
    }  
  
    unlink(SERVER_PATH);  
    close(server_fd);  
    close(client_fd);  
  
    return 0;  
}
```

**client.c：**
```
#include <stdio.h>  
#include <stdlib.h>  
#include <string.h>  
#include <unistd.h>  
#include <fcntl.h>           /* For O_* constants */  
#include <sys/stat.h>        /* For mode constants */  
#include <sys/socket.h>  
#include <netinet/in.h>  
#include <arpa/inet.h>  
#include <sys/un.h>  
  
#define SERVER_PATH "/tmp/server"  
#define CLIENT_PATH "/tmp/client"  
  
int main(void)  
{  
    ///< 创建UNIX域字节流套接字  
    int client_fd = socket(AF_UNIX, SOCK_STREAM, 0);  
    if(client_fd < 0)  
    {  
        printf("socket error\n");  
        exit(EXIT_FAILURE);  
    }  
  
    ///< 显式绑定客户端地址  
    struct sockaddr_un client_addr;  
    memset((char*)&client_addr, 0, sizeof(client_addr));  
    client_addr.sun_family = AF_UNIX;  
    strncpy(client_addr.sun_path, CLIENT_PATH, sizeof(client_addr.sun_path)-1);  
    unlink(CLIENT_PATH);  
    if(bind(client_fd, (struct sockaddr*)&client_addr, sizeof(client_addr)) < 0)  
    {  
        printf("bind error\n");  
        close(client_fd);  
        exit(1);   
    }  
  
    ///< 连接服务端  
    struct sockaddr_un server_addr;  
    server_addr.sun_family  = AF_UNIX;  
    strncpy(server_addr.sun_path, SERVER_PATH, sizeof(server_addr.sun_path)-1);  
    int ret = connect(client_fd, (struct sockaddr*)&server_addr, sizeof(server_addr));   
    if(ret < 0)  
    {  
        printf("connect error\n");  
        close(client_fd);  
        unlink(CLIENT_PATH);  
        exit(1);    
    }   
    printf("connect to server: %s\n", server_addr.sun_path);  
  
    while(1)  
    {  
        char buf[128] = {0};  
        if (scanf("%s", buf))  
        {  
            int send_len = write(client_fd, buf, strlen(buf));  
            if (send_len <= 0)  
            {  
                printf("write error!\n");  
                close(client_fd);  
                exit(EXIT_FAILURE);    
            }  
            else  
            {  
                printf("send success! send: %s, send_len: %d\n", buf, send_len);  
            }  
        }   
    }  
  
    unlink(SERVER_PATH);  
    close(client_fd);  
  
    return 0;  
}
```

编译、运行：

```
gcc server.c -o server_process
gcc client.c -o client_process
```

![](https://mmbiz.qpic.cn/sz_mmbiz_png/K62vJYPSiagiaD9rVkQRHlMBsBmlh5p1h76yZnltaolEPBykhpdMABGsu9ibIDNcTuqVKPQxWAYqlLEsbE21WzVcSIogG1eCyiaohEorwNZ6xNI/640?wx_fmt=png&from=appmsg)

类socket的其它进程间通信方式：
[一个嵌入式系统进程间通信利器！](https://mp.weixin.qq.com/s?__biz=MzU5MzcyMjI4MA==&mid=2247524855&idx=1&sn=a1c86ea762e4fc4d2f467332ccd23e11&scene=21#wechat_redirect)

## 2.6 信号（Signal）

与上面介绍的IPC方式不同，信号不用于传输数据，而是一种轻量级的异步通知机制，用于告知目标进程某个事件已经发生。在嵌入式开发中，信号常用于优雅关闭进程（SIGTERM）、父子进程协调（SIGCHLD）、定时任务（SIGALRM）等场景。

下图展示了信号的工作原理——信号就像一个"拍肩膀"的动作，告诉目标进程"有事发生了"：

![[resource/Pasted image 20260410101731.png]]

常见的信号：

| 信号      | 值   | 说明            |
| ------- | --- | ------------- |
| SIGINT  | 2   | 终端中断（Ctrl+C）  |
| SIGKILL | 9   | 强制终止进程（不可捕获）  |
| SIGTERM | 15  | 请求终止进程（可捕获）   |
| SIGCHLD | 17  | 子进程状态改变时通知父进程 |
| SIGUSR1 | 10  | 用户自定义信号1      |
| SIGUSR2 | 12  | 用户自定义信号2      |
| SIGALRM | 14  | 定时器超时         |

信号处理的核心API：

```
#include <signal.h>  
  
// 注册信号处理函数（简易版）  
typedef void (*sighandler_t)(int);  
sighandler_t signal(int signum, sighandler_t handler);  
  
// 注册信号处理函数（推荐，功能更强大）  
int sigaction(int signum, const struct sigaction *act, struct sigaction *oldact);  
  
// 向指定进程发送信号  
int kill(pid_t pid, int sig);
```

信号基本使用例子：父进程通过信号通知子进程执行特定操作。

```
#include <stdio.h>  
#include <stdlib.h>  
#include <unistd.h>  
#include <signal.h>  
#include <sys/wait.h>  
  
void sigusr1_handler(int sig)  
{  
    constchar msg[] = "child process received SIGUSR1\n";  
    write(STDOUT_FILENO, msg, sizeof(msg) - 1);  
}  
  
int main(void)  
{  
    pid_t pid = fork();  
  
    if (pid == 0)  
    {  
        ///< 子进程注册SIGUSR1信号处理函数，然后挂起等待信号  
        signal(SIGUSR1, sigusr1_handler);  
        printf("child process waiting for signal...\n");  
        pause();  
        printf("child process exiting\n");  
        _exit(0);  
    }  
    elseif (pid > 0)  
    {  
        sleep(1);  
        ///< 父进程向子进程发送SIGUSR1信号  
        printf("parent process sending SIGUSR1 to child\n");  
        kill(pid, SIGUSR1);  
        wait(NULL);  
        printf("parent process done\n");  
    }  
  
    return 0;  
}
```

**注意：** 信号处理函数中应尽量只调用异步信号安全（async-signal-safe）的函数，避免在信号处理函数中使用 `printf`、`malloc` 等非安全函数。

## 2.7 内存映射（Memory-Mapped I/O）

通过文件或匿名映射实现共享：

```
#include <sys/mman.h>

// 文件-backed共享（持久化）或匿名映射（MAP_SHARED）
void *addr = mmap(NULL, length, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
msync(addr, length, MS_ASYNC);  // 刷盘
munmap(addr, length);
```

**优势**：共享内存的灵活性，可配合文件锁`flock()`或`fcntl()`实现同步。**POSIX 共享内存的底层实现就是基于内存映射（`mmap`）的。**

**内存映射能做但共享内存不能做的事**：

- 映射普通文件实现**零拷贝文件 I/O**（将磁盘文件直接映射到内存修改，避免 `read/write` 缓冲区拷贝）
- 执行**私有映射** (`MAP_PRIVATE`) 实现写时复制（Copy-on-Write），用于进程分叉时的内存隔离
- 映射设备文件或硬件寄存器

内存映射是**底层机制**，共享内存是**上层应用概念**。POSIX 标准选择用内存映射来实现共享内存，但内存映射本身能做更多事情。

## System V IPC 经典 trio

```
// 消息队列 - 结构化消息传递
msgget() / msgsnd() / msgrcv() / msgctl()

// 共享内存 - 最快的IPC（需配合信号量）
shmget() / shmat() / shmdt() / shmctl()

// 信号量集 - 进程同步/互斥
semget() / semop() / semctl()
```

## POSIX IPC（新标准）

```
// 消息队列
mq_open() / mq_send() / mq_receive() / mq_close()

// 共享内存（配合`mmap`使用更佳）
shm_open() / mmap() / shm_unlink()

// 无名信号量（进程内）与有名信号量（跨进程）
sem_init() / sem_post() / sem_wait() / sem_destroy()  // 内存内
sem_open() / sem_close() / sem_unlink()             // 跨进程
```
##  IPC总结

操作系统根据不同的场景提供了不同的IPC方式，下面回顾各种方式的核心特点：

**`消息队列：`** 内核中的一个优先级队列，多个进程通过访问同一个队列，在队列当中添加或者获取节点来实现进程间通信。

**`共享内存：`** 本质是一块物理内存，多个进程将同一块物理内存映射到自己的虚拟地址空间中，再通过页表映射到物理地址达到进程间通信，它是最快的进程间通信方式，相较其他通信方式少了两步数据拷贝操作。

**`UNIX域套接字：`** 与TCP/IP套接字使用方式相同，但UNIX域套接字以文件系统作为地址空间，不需经过TCP/IP的头部封装、报文ack确认、路由选择、数据校验与重传过程，因此传输速率上也不会受网卡带宽的限制。

**`管道：`** 内核中的一块缓冲区，分为匿名管道和命名管道。匿名管道只能用于具有亲缘关系的进程间；而命名管道可用于同一主机上任意进程间通信。

**`信号量：`** 本质是内核中的一个计数器，主要实现进程间的同步与互斥，对资源进行计数，有两种操作，分别是在访问资源之前进行的P操作（获取资源），还有产生资源之后的V操作（释放资源）。

**`信号：`** 一种轻量级的异步通知机制，不传输数据，仅用于通知目标进程某个事件已发生，常用于进程控制和异常处理。

# 3. 线程间通信方式

由于线程共享堆内存，通信本质上是**同步问题**——防止竞态条件（Race Condition）和确保内存可见性。

## 3.1 互斥锁（Mutex）

```
// C11标准
#include <threads.h>
mtx_t mutex;
mtx_init(&mutex, mtx_plain);
mtx_lock(&mutex);  // 阻塞
mtx_unlock(&mutex);

// C++11标准（RAII封装，推荐）
#include <mutex>
std::mutex m;
std::lock_guard<std::mutex> lock(m);       // 作用域自动释放
std::unique_lock<std::mutex> lock(m);      // 支持条件变量
```

## 3.2 条件变量（Condition Variable）

解决"等待-通知"模式（避免忙等）：

```
#include <condition_variable>
std::condition_variable cv;
std::unique_lock<std::mutex> lk(mutex);
cv.wait(lk, []{ return data_ready; });  // 原子释放锁并阻塞
cv.notify_one();  // 唤醒一个
cv.notify_all();  // 广播唤醒
```

## 3.3 读写锁（Read-Write Lock）

读并发、写独占：

```
// POSIX
pthread_rwlock_t rwlock;
pthread_rwlock_rdlock(&rwlock);   // 读锁
pthread_rwlock_wrlock(&rwlock);   // 写锁

// C++17 shared_mutex（性能优于rwlock）
std::shared_mutex sm;
std::shared_lock<std::shared_mutex> read_lock(sm);
std::unique_lock<std::shared_mutex> write_lock(sm);
```

## 3.4 屏障（Barrier）

多阶段算法同步点：

```
// POSIX线程屏障
pthread_barrier_t barrier;
pthread_barrier_init(&barrier, NULL, thread_count);
pthread_barrier_wait(&barrier);  // 到达后阻塞，直到所有线程到达

// C++20
std::barrier<> sync_point(n);
sync_point.arrive_and_wait();
```

## 3.5 原子操作与内存序（Lock-free）

无锁编程，C11/C++11标准：

```
#include <atomic>
std::atomic<int> counter{0};
counter.fetch_add(1, std::memory_order_relaxed);  // 不同内存序策略

// 标志位同步（最常用）
std::atomic<bool> flag{false};
flag.store(true, std::memory_order_release);
while(!flag.load(std::memory_order_acquire));     // 线程可见性保证
```

## 3.6 线程局部存储（TLS）

数据隔离，避免共享：

```
// GCC扩展或C11 _Thread_local
__thread int thread_local_var;  

// C++11
thread_local int tls_var = 0;
```