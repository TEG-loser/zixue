# Day23：exec 系列函数完整实验（fork + exec 经典组合）
包含：
- exec 函数作用
- 6 个 exec 函数区别（简化记忆）
- 完整可运行程序
- 实验现象总结

## 最终成果：exec_test.cpp
```cpp
#include <iostream>
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <cstdio>
#include <cstdlib>

int main() {
    printf("=== 主进程 PID = %d ===\n", getpid());

    // 1. 创建子进程
    pid_t pid = fork();
    if (pid < 0) {
        perror("fork failed");
        return -1;
    }

    // ==================== 子进程：执行 exec ====================
    if (pid == 0) {
        printf("【子进程】PID = %d，准备执行 exec...\n", getpid());

        // 执行 Linux 命令：/bin/ls -l
        // execl 版本：路径 + 参数列表 + NULL
        execl("/bin/ls", "ls", "-l", NULL);

        // ===================== 重要 =====================
        // 如果 exec 成功，这里**永远不会执行**
        // 只有 exec 失败才会走到这里
        perror("exec failed");
        _exit(1);
    }
    // ==================== 父进程：等待子进程 ====================
    else {
        printf("【父进程】等待子进程结束...\n");

        // 等待子进程完成
        wait(NULL);

        printf("【父进程】子进程已结束，主程序继续执行\n");
    }

    return 0;
}
```

## 编译运行
```bash
g++ exec_test.cpp -o exec_test
./exec_test
```

## 运行结果
```
=== 主进程 PID = 12345 ===
【父进程】等待子进程结束...
【子进程】PID = 12346，准备执行 exec...
总用量 48
-rwxr-xr-x 1 user user 22000  4月 28 10:00 exec_test
-rw-r--r-- 1 user user  1024  4月 28 10:00 exec_test.cpp
...
【父进程】子进程已结束，主程序继续执行
```

---

# 一、exec 核心知识点（必须掌握）
## 1. exec 是什么？
- **exec = 替换进程映像**
- 用**新程序**替换当前进程的代码段、数据段、堆栈
- **进程 PID 不变**，只是换了要运行的程序
- 一旦 exec **成功**，原程序**不再执行**

## 2. 核心作用
**fork 创建新进程 + exec 加载新程序**
这是 **Linux 所有命令（ls、pwd、ps）的启动方式**

## 3. 关键特性
- 成功：不返回，直接运行新程序
- 失败：返回 -1
- **继承原进程的 PID、文件描述符、信号处理**

---

# 二、6 个 exec 函数（超级简化记忆）
不需要全背，记住 **2 个最常用**：

| 函数名 | 参数格式 | 路径 | 用途 |
|-------|---------|------|------|
| **execl** | 列表（以 NULL 结尾） | 必须写完整路径 /bin/ls | 简单命令 |
| **execlp** | 列表 | 自动找 PATH（直接写 ls） | **最常用** |
| execv | 数组 | 完整路径 | 批量参数 |
| execvp | 数组 | 自动找 PATH | 脚本/命令 |

### 最常用示例
```cpp
// 自动搜索 PATH，最推荐
execlp("ls", "ls", "-l", NULL);
```

---

# 三、fork + exec 经典流程（Linux 进程启动流程）
## 伪代码
```
父进程
    fork() → 创建子进程

子进程
    调用 exec() → 加载新程序
    替换自己的代码/数据
    执行新程序（如 ls）

父进程
    wait() 等待子进程结束
```

## 一句话总结
**fork 生孩子，exec 让孩子去干别的事。**

---

# 四、面试必考
1. **exec 调用成功后，原来的代码还会执行吗？**
   不会，直接被覆盖。

2. **exec 会创建新进程吗？**
   不会，**PID 不变**，只是替换程序内容。

3. **fork + exec 为什么是标准组合？**
   fork 创建独立进程，exec 替换程序，实现启动新程序。

## 总结
- **exec = 替换进程运行的程序**
- **PID 不变，程序内容全变**
- 成功不返回，失败才返回
- **fork + exec = Linux 所有进程的创建方式**
- **execlp() 最常用，自动搜索 PATH**

需要我明天带你学习 **进程等待 wait / waitpid** 吗？
