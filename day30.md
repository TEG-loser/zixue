# Day30 完整交付：**多客户端并发 TCP 服务器（fork 版）**


---

# 一、核心知识点（必理解）
## 1. 阻塞 accept
- **默认 socket 都是阻塞的**
- `accept()` 没有客户端连接时 **一直等**
- 处理一个客户端时，**其他客户端连不进来**（串行）

## 2. 并发方案
- **fork()** 创建子进程处理客户端
- 父进程继续回去 `accept()`
- 实现 **真正多客户端同时连接**

## 3. 必须处理
- 子进程退出 → 防止**僵尸进程**
- 关闭多余 fd → 防止文件描述符泄漏
---

# 二、最终完整代码：**tcp_server_fork.cpp**
```cpp
#include <iostream>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <signal.h>
#include <sys/wait.h>

#define PORT 8888
#define BUF_SIZE 1024

// 处理僵尸进程
void sig_child(int sig) {
    while (waitpid(-1, NULL, WNOHANG) > 0);
}

// 客户端处理逻辑（回显）
void handle_client(int client_fd) {
    char buf[BUF_SIZE];
    while (true) {
        ssize_t n = recv(client_fd, buf, BUF_SIZE - 1, 0);

        if (n < 0) {
            perror("recv error");
            break;
        } else if (n == 0) {
            printf("客户端断开\n");
            break;
        }

        buf[n] = '\0';
        printf("recv: %s", buf);
        send(client_fd, buf, n, 0);
    }
    close(client_fd);
}

int main() {
    // 处理僵尸进程
    signal(SIGCHLD, sig_child);

    // 1. socket
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (listen_fd < 0) {
        perror("socket failed");
        return -1;
    }

    // 端口复用
    int opt = 1;
    setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    // 2. bind
    struct sockaddr_in server_addr;
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;
    server_addr.sin_port = htons(PORT);

    if (bind(listen_fd, (struct sockaddr*)&server_addr, sizeof(server_addr)) < 0) {
        perror("bind failed");
        close(listen_fd);
        return -1;
    }

    // 3. listen
    if (listen(listen_fd, 5) < 0) {
        perror("listen failed");
        close(listen_fd);
        return -1;
    }

    printf("TCP Server run on 0.0.0.0:%d\n", PORT);

    // ===================== Step1：循环 accept =====================
    while (true) {
        struct sockaddr_in client_addr;
        socklen_t client_len = sizeof(client_addr);

        // 阻塞等待客户端
        int client_fd = accept(listen_fd, (struct sockaddr*)&client_addr, &client_len);
        if (client_fd < 0) {
            perror("accept error");
            continue;
        }

        printf("新连接: %s:%d\n", inet_ntoa(client_addr.sin_addr), ntohs(client_addr.sin_port));

        // ===================== Step3：fork 并发 =====================
        pid_t pid = fork();
        if (pid < 0) {
            perror("fork failed");
            close(client_fd);
            continue;
        }

        if (pid == 0) {
            // 子进程：处理客户端
            close(listen_fd); // 子进程不需要监听fd
            handle_client(client_fd);
            exit(0); // 处理完退出
        } else {
            // 父进程：关闭客户端fd，继续accept
            close(client_fd);
        }
    }

    close(listen_fd);
    return 0;
}
```

---

# 三、编译运行
```bash
g++ tcp_server_fork.cpp -o server
./server
```

开 **3 个终端**测试：
```bash
nc 127.0.0.1 8888
```
**全部同时连接、同时收发，不卡顿！**

---

# 四、按你的路线一步步讲解

## Step1：循环 accept
```cpp
while (true) {
    int client_fd = accept(listen_fd, ...);
}
```
- 服务器永远运行
- 来一个客户端处理一个

## Step2：串行版本问题
**不 fork** 的话：
- 第一个客户端连接后，服务器进入 `while(recv)`
- **第二个客户端连不上**，一直阻塞在 `accept`
- 必须等第一个客户端断开，才能处理下一个

这就是 **串行（阻塞）服务器**。

## Step3：fork 实现并发（核心）
```cpp
pid = fork();

if (子进程) {
    关闭 listen_fd
    处理客户端收发
    退出
} else {
    关闭 client_fd
    继续 accept
}
```

### 关键要点
- **子进程关闭 listen_fd**（不需要）
- **父进程关闭 client_fd**（文件描述符泄漏）
- `SIGCHLD` + `waitpid` 处理**僵尸进程**

## Step4：测试并发
```bash
ps -ef | grep server
```
你会看到：
- 1 个主进程（父进程）
- N 个子进程（对应 N 个客户端）
- 客户端断开 → 子进程消失

---

# 五、自测口述知识点（Day30 达标）
## 1. accept 阻塞行为
**没有客户端连接时，accept 会一直阻塞等待，不占CPU。**

## 2. 串行 vs 并发
- **串行**：同一时间只能处理一个客户端
- **并发（fork）**：多个客户端同时连接、同时通信

## 3. 服务端标准流程（必背）
```
socket → bind → listen → while(accept) → fork 处理客户端
```

## 4. 为什么要关闭多余 fd？
- 防止文件描述符耗尽
- 父子进程共享文件描述符表，不关闭会泄漏


# 你已完成：
✅ TCP 服务端全流程
✅ 阻塞 / 非阻塞理解
✅ 多客户端并发
✅ fork 进程模型
✅ 僵尸进程处理
✅ 文件描述符管理
