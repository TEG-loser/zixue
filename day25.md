# Day25：Linux 信号 Signal 
## 一、核心概念
### 1. 什么是信号
信号是 **Linux 进程间异步通信方式**，内核/其他进程给目标进程发一个**软中断通知**：
- 通知进程发生了某个事件
- 进程可以：**忽略、默认处理、自定义捕获**

### 2. 常见信号
| 信号 | 含义 | 默认行为 |
|------|------|----------|
| SIGINT 2 | Ctrl+C 中断 | 终止进程 |
| SIGKILL 9 | 强制杀死 | 不可忽略、不可捕获 |
| SIGSEGV 11 | 段错误 | 终止+核心转储 |
| SIGALRM 14 | 闹钟定时信号 | 终止 |
| SIGCHLD 17 | 子进程退出 | 忽略 |

### 3. 信号处理三种方式
1. **默认处理**：系统自带行为（终止/忽略/暂停）
2. **忽略信号**：`SIG_IGN`
3. **自定义捕获**：注册信号处理函数

---

## 二、基础实验代码：signal_demo.cpp
功能：
- 自定义捕获 `SIGINT(Ctrl+C)`
- 演示忽略信号、自定义回调
```cpp
#include <iostream>
#include <signal.h>
#include <unistd.h>
#include <cstdio>

// 信号处理回调函数
void sig_handler(int sig)
{
    printf("\n收到信号：%d (SIGINT Ctrl+C)\n", sig);
    printf("进程不退出，继续运行...\n");
}

int main()
{
    // 注册信号捕获：SIGINT 走自定义处理函数
    signal(SIGINT, sig_handler);

    // 忽略 SIGCHLD 子进程退出信号
    signal(SIGCHLD, SIG_IGN);

    printf("进程 %d 正在运行，按 Ctrl+C 测试信号\n", getpid());
    while (true)
    {
        sleep(1);
        printf("running...\n");
    }

    return 0;
}
```

### 编译运行
```bash
g++ signal_demo.cpp -o signal_demo
./signal_demo
```
按下 `Ctrl+C` 不会退出，执行自定义逻辑。

---

## 三、进阶：alarm 定时信号
```cpp
#include <iostream>
#include <signal.h>
#include <unistd.h>

void alarm_handler(int sig)
{
    if (sig == SIGALRM)
    {
        printf("闹钟时间到！\n");
    }
}

int main()
{
    signal(SIGALRM, alarm_handler);

    alarm(3); // 3秒后发送 SIGALRM
    printf("设置3秒闹钟，等待...\n");
    
    // 阻塞等待信号
    pause();
    printf("程序结束\n");
    return 0;
}
```

---

## 四、信号核心知识点
1. **信号是异步的**
进程不知道何时收到信号，随时可能被打断当前执行流。

2. **SIGKILL 9 不能被捕获、不能被忽略**
系统强制杀死，用于卡死进程收尾。

3. **信号处理函数特点**
- 执行期间会**阻塞同类信号**
- 尽量简单，不要做复杂耗时操作

4. **父子进程与信号**
- 子进程默认继承父进程的信号处理方式
- `SIGCHLD` 子进程退出通知，常用 `SIG_IGN` 避免僵尸进程

---

## 五、信号工作流程伪代码
```
1. 注册信号处理函数 signal(信号号, 回调)
2. 进程正常运行
3. 内核/其他进程发送信号给本进程
4. 进程收到软中断，暂停当前流程
5. 执行自定义信号处理函数
6. 处理完回到原代码继续执行
```

---

## 六、GitHub 提交
```bash
git add signal_demo.cpp
git commit -m "Day25：Linux 信号机制实验，自定义捕获SIGINT、闹钟信号"
git push
```
