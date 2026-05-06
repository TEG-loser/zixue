# Day24：管道 pipe 完整实战


## 一、pipe 管道程序：pipe_demo.cpp
功能：
父进程通过管道发数据 → 子进程从管道读数据
```cpp
#include <iostream>
#include <unistd.h>
#include <sys/wait.h>
#include <cstdio>
#include <cstring>

int main()
{
    // 管道文件描述符：fd[0]读端，fd[1]写端
    int fd[2];
    char buf[128];

    // 1. 创建匿名管道
    if (pipe(fd) == -1)
    {
        perror("pipe error");
        return -1;
    }

    // 2. fork 创建子进程
    pid_t pid = fork();
    if (pid < 0)
    {
        perror("fork error");
        return -1;
    }

    // ===== 子进程：读管道 =====
    if (pid == 0)
    {
        close(fd[1]);  // 子进程关闭写端
        // 从管道读数据
        read(fd[0], buf, sizeof(buf));
        printf("子进程收到：%s\n", buf);

        close(fd[0]);
        _exit(0);
    }
    // ===== 父进程：写管道 =====
    else
    {
        close(fd[0]);  // 父进程关闭读端
        const char* msg = "Hello from parent via pipe";
        // 向管道写数据
        write(fd[1], msg, strlen(msg));

        close(fd[1]);
        wait(nullptr); // 等待子进程结束
    }

    return 0;
}
```

## 编译运行
```bash
g++ pipe_demo.cpp -o pipe_demo
./pipe_demo
```

## 运行结果
```
子进程收到：Hello from parent via pipe
```

---

# 二、管道核心知识点
## 1. 什么是匿名管道
- `pipe()` 创建**匿名管道**，用于**有亲缘关系进程间通信**（父子、兄弟进程）
- 半双工：**单向通信**
- 内核维护环形缓冲区，内存级管道，无磁盘IO

## 2. 管道文件描述符
```
int fd[2];
fd[0] : 管道 读端
fd[1] : 管道 写端
```

## 3. 通信规则
1. 数据从 **写端流入，读端流出**
2. 管道为空：`read` 阻塞等待
3. 管道写端全部关闭：`read` 返回 0，读到末尾
4. 只能用于 **亲缘进程**

## 4. 必须关闭不用的端口
- 父进程只写 → 关闭读端 `fd[0]`
- 子进程只读 → 关闭写端 `fd[1]`

---

# 三、管道通信流程伪代码
```
1. pipe(fd) 创建管道
2. fork() 创建子进程

3. 父进程：
   close(fd[0])   关闭读端
   write(fd[1])   向管道写入数据
   close(fd[1])

4. 子进程：
   close(fd[1])   关闭写端
   read(fd[0])    从管道读出数据
   close(fd[0])
```

# 四、管道特点总结
1. 匿名管道 **只能亲缘进程 IPC**
2. **半双工**，单向数据流
3. 内核缓冲，速度快
4. 自带同步：空则读阻塞，满则写阻塞
5. 生命周期随进程，无持久化
