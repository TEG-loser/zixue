# Day29 完整学习笔记 + 代码 + 三次握手图解 + 口述知识点
## 一、理论基础：TCP/IP 与 Socket
### 1. TCP/IP 四层模型
1. **应用层**：HTTP、SSH、自定义协议
2. **传输层**：TCP、UDP
3. **网络层**：IP、ICMP
4. **链路层**：以太网、ARP

### 2. TCP 核心特性
- 面向**有连接**
- 可靠传输：确认应答、超时重传、去重、有序
- 全双工通信
- 基于字节流

### 3. Socket 概念
Socket 是**操作系统提供的网络编程接口**，本质是：
**Socket = IP地址 + 端口号**
用于跨主机进程间通信。

### 4. 核心 Socket API 作用
- `socket()`：创建套接字文件描述符
- `connect()`：客户端主动发起连接
- `send()/recv()`：收发数据
- `close()`：关闭套接字

---

## 二、TCP 三次握手流程（图解+文字版）
### 流程示意图
```
客户端                服务端
   |                    |
1. |------- SYN ------->|  第一次握手：客户端请求建立连接
   |                    |
2. |<----- SYN+ACK -----|  第二次握手：服务端同意并确认
   |                    |
3. |------- ACK ------->|  第三次握手：客户端确认收到
   |                    |
      连接建立成功
```

### 作用
1. 同步双方初始序列号
2. 确认双方**发送、接收能力**都正常
3. 建立可靠连接，为后续数据传输做准备

### 口述背诵版
TCP 三次握手：
第一次客户端向服务端发送 SYN 同步报文；
第二次服务端返回 SYN+ACK，同意连接并确认客户端请求；
第三次客户端再回复 ACK 确认，双方连接正式建立。

## 四、客户端执行流程
1. `socket()` 创建流式 TCP 套接字
2. 配置服务端 IP、端口
3. `connect()` 发起三次握手，建立连接
4. `send()` 发送数据
5. `recv()` 接收服务端响应
6. `close()` 断开连接

## 五、自测口述知识点
1. TCP 是**面向连接、可靠、全双工**的传输层协议。
2. Socket 是网络编程接口，由 **IP+端口** 唯一标识通信端点。
3. 三次握手目的：同步序列号、检测收发能力、建立可靠连接。
4. TCP 客户端固定流程：`socket → connect → send/recv → close`。


## 六、TCP 服务端标准流程图（Step1 完成）
```
【服务端流程】

socket()   → 创建套接字
bind()     → 绑定 IP:端口
listen()   → 开始监听（被动模式）
accept()   → 阻塞等待客户端连接（返回**新fd**用于通信）

    ↓ 循环
recv()     → 读取客户端数据
send()     → 原样回显数据
close()    → 关闭客户端连接
```

```
【客户端流程】
socket()
connect() → 主动连接服务端
send/recv
close()
```

---

## 七、TCP 回显服务器（完整版）echo_server.cpp
```cpp
#include <iostream>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

#define PORT 8888
#define BUF_SIZE 1024

int main() {
    // ===================== 1. socket =====================
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (listen_fd < 0) {
        perror("socket failed");
        return -1;
    }

    // 地址复用（避免TIME_WAIT问题）
    int opt = 1;
    setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR | SO_REUSEPORT, &opt, sizeof(opt));

    // ===================== 2. bind =====================
    struct sockaddr_in server_addr;
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;
    server_addr.sin_port = htons(PORT);

    if (bind(listen_fd, (struct sockaddr*)&server_addr, sizeof(server_addr)) < 0) {
        perror("bind failed");
        close(listen_fd);
        return -1;
    }

    // ===================== 3. listen =====================
    if (listen(listen_fd, 5) < 0) {
        perror("listen failed");
        close(listen_fd);
        return -1;
    }

    printf("TCP Echo Server running on 0.0.0.0:%d\n", PORT);
    printf("Waiting for client...\n");

    // ===================== 4. accept 循环 =====================
    while (true) {
        struct sockaddr_in client_addr;
        socklen_t client_len = sizeof(client_addr);

        // 阻塞等待客户端
        int client_fd = accept(listen_fd, (struct sockaddr*)&client_addr, &client_len);
        if (client_fd < 0) {
            perror("accept failed");
            continue;
        }

        printf("New client connected: %s:%d\n",
               inet_ntoa(client_addr.sin_addr),
               ntohs(client_addr.sin_port));

        // ===================== 5. 回显逻辑 =====================
        char buf[BUF_SIZE];
        while (true) {
            ssize_t n = recv(client_fd, buf, BUF_SIZE - 1, 0);

            if (n < 0) {     // 错误
                perror("recv error");
                break;
            } else if (n == 0) { // 客户端断开
                printf("Client disconnected\n");
                break;
            }

            buf[n] = '\0';
            printf("Recv: %s", buf);

            // 回显
            send(client_fd, buf, n, 0);
        }

        close(client_fd);
    }

    close(listen_fd);
    return 0;
}
```

---

## 八、TCP 客户端（可选）echo_client.cpp
```cpp
#include <iostream>
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

#define SERVER_IP "127.0.0.1"
#define SERVER_PORT 8888
#define BUF_SIZE 1024

int main() {
    int sockfd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in server_addr;
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(SERVER_PORT);
    inet_pton(AF_INET, SERVER_IP, &server_addr.sin_addr);

    if (connect(sockfd, (struct sockaddr*)&server_addr, sizeof(server_addr)) < 0) {
        perror("connect failed");
        return -1;
    }

    char send_buf[BUF_SIZE];
    char recv_buf[BUF_SIZE];

    while (fgets(send_buf, BUF_SIZE, stdin)) {
        send(sockfd, send_buf, strlen(send_buf), 0);

        ssize_t n = recv(sockfd, recv_buf, BUF_SIZE - 1, 0);
        if (n <= 0) break;

        recv_buf[n] = '\0';
        printf("Echo: %s", recv_buf);
    }

    close(sockfd);
    return 0;
}
```

---

## 九、编译运行
```bash
g++ echo_server.cpp -o server
g++ echo_client.cpp -o client

# 终端1
./server

# 终端2
./client
# 或
nc 127.0.0.1 8888
```

---

##十、异常处理说明（Step4 完成）
### 1. recv 返回值含义
- **n > 0**：收到 n 字节数据
- **n = 0**：**客户端正常断开**
- **n < 0**：出错

### 2. 客户端断开处理
```cpp
} else if (n == 0) {
    printf("Client disconnected\n");
    break;
}
```

### 3. 所有系统调用都判断返回值
- socket
- bind
- listen
- accept
- recv
- send

---

## 六、必背知识点（口述自测）
## 1. TCP 服务端 5 步
1. **socket**：创建文件描述符
2. **bind**：绑定 8888 端口
3. **listen**：将 socket 设为监听模式
4. **accept**：阻塞等客户端，返回**连接fd**
5. **read/write**：收发数据

## 2. 谁主动 connect？
**只有客户端调用 connect**
服务端永远是被动等待 accept


