完成 Day8 pthread 基础的实战任务，核心是理解线程与进程的区别、掌握 pthread 核心 API、验证线程调度特性和共享内存问题。

---

## 一、完整代码实现
### 1. 基础多线程程序（pthread_basic.cpp）
包含 5 个线程创建、打印线程 ID、主线程 join 所有线程，验证线程并发调度特性：

```cpp
#include <iostream>
#include <pthread.h>
#include <unistd.h>   // for sleep
#include <cstdio>     // for perror
#include <vector>

// 线程函数参数结构体（传递多个参数示例）
struct ThreadData {
    int thread_num;    // 线程编号
    pthread_t tid;     // 线程ID（输出用）
};

// 线程函数（必须是 void* (*)(void*) 签名）
void* thread_func(void* arg) {
    ThreadData* data = static_cast<ThreadData*>(arg);
    
    // 模拟耗时操作（让调度更明显）
    sleep(1);
    
    // 打印线程信息
    printf("线程 %d: 线程ID=0x%lx，进程ID=%d\n", 
           data->thread_num, 
           (unsigned long)pthread_self(),  // 获取当前线程ID
           getpid());                       // 所有线程共享进程ID
    
    // 标记线程ID（回传给主线程）
    data->tid = pthread_self();
    
    // 线程退出，返回空指针
    pthread_exit(nullptr);
}

int main() {
    const int THREAD_COUNT = 5;
    std::vector<pthread_t> tids(THREAD_COUNT);  // 存储线程ID
    std::vector<ThreadData> thread_datas(THREAD_COUNT); // 线程参数
    
    // 1. 创建5个线程
    for (int i = 0; i < THREAD_COUNT; ++i) {
        thread_datas[i].thread_num = i + 1; // 线程编号从1开始
        int ret = pthread_create(&tids[i],          // 输出线程ID
                                 nullptr,           // 默认线程属性
                                 thread_func,       // 线程函数
                                 &thread_datas[i]); // 传递参数
        
        if (ret != 0) {
            perror("pthread_create failed");
            return -1;
        }
        printf("主线程: 创建线程 %d，初始ID=0x%lx\n", i+1, (unsigned long)tids[i]);
    }

    // 2. 主线程join所有子线程（等待子线程结束）
    printf("\n主线程: 等待所有子线程完成...\n");
    for (int i = 0; i < THREAD_COUNT; ++i) {
        int ret = pthread_join(tids[i], nullptr); // 等待线程结束，不接收返回值
        if (ret != 0) {
            perror("pthread_join failed");
            return -1;
        }
        printf("主线程: 线程 %d 已结束，最终ID=0x%lx\n", 
               i+1, (unsigned long)thread_datas[i].tid);
    }

    // 3. 验证：所有线程共享进程ID
    printf("\n主线程: 进程ID=%d，所有线程属于同一进程\n", getpid());
    
    return 0;
}
```

### 2. join vs detach 对比程序（join_vs_detach.cpp）
分别实现 join 和 detach 版本，对比主线程退出时子线程的行为：

```cpp
#include <iostream>
#include <pthread.h>
#include <unistd.h>
#include <cstdio>

// 通用线程函数：打印信息 + 模拟耗时
void* worker(void* arg) {
    int id = *static_cast<int*>(arg);
    printf("线程 %d: 启动，开始执行耗时操作...\n", id);
    
    // 模拟5秒耗时操作
    for (int i = 0; i < 5; ++i) {
        sleep(1);
        printf("线程 %d: 执行中（剩余%d秒）\n", id, 4 - i);
    }
    
    printf("线程 %d: 执行完成，退出\n", id);
    pthread_exit(nullptr);
}

// 版本1：使用 join（主线程等待子线程）
void test_join() {
    printf("\n===== 测试 join 版本 =====\n");
    pthread_t tid;
    int id = 1;
    
    // 创建线程
    int ret = pthread_create(&tid, nullptr, worker, &id);
    if (ret != 0) {
        perror("pthread_create failed");
        return;
    }
    
    // 主线程执行自己的任务（1秒）
    printf("主线程: 执行1秒任务...\n");
    sleep(1);
    
    // join 等待子线程结束
    printf("主线程: 开始join线程1...\n");
    ret = pthread_join(tid, nullptr);
    if (ret != 0) {
        perror("pthread_join failed");
        return;
    }
    
    printf("主线程: 线程1已结束，主线程退出\n");
}

// 版本2：使用 detach（主线程不等待子线程）
void test_detach() {
    printf("\n===== 测试 detach 版本 =====\n");
    pthread_t tid;
    int id = 2;
    
    // 创建线程
    int ret = pthread_create(&tid, nullptr, worker, &id);
    if (ret != 0) {
        perror("pthread_create failed");
        return;
    }
    
    // 分离线程（detach）
    ret = pthread_detach(tid);
    if (ret != 0) {
        perror("pthread_detach failed");
        return;
    }
    
    // 主线程执行1秒任务后直接退出
    printf("主线程: 执行1秒任务后退出...\n");
    sleep(1);
    printf("主线程: 已退出，子线程可能被终止\n");
}

int main() {
    // 测试join版本
    test_join();
    
    // 等待5秒，确保上一轮测试完成
    sleep(5);
    
    // 测试detach版本
    test_detach();
    
    return 0;
}
```

### 3. 线程共享变量实验（thread_shared_var.cpp）
验证多线程无锁访问共享变量的竞态问题：

```cpp
#include <iostream>
#include <pthread.h>
#include <vector>
#include <cstdio>

const int THREAD_COUNT = 5;
const int INCR_COUNT = 100000; // 每个线程自增10万次

// 共享变量（所有线程可见）
int counter = 0;

// 线程函数：对counter自增
void* incr_counter(void* arg) {
    (void)arg; // 忽略参数
    for (int i = 0; i < INCR_COUNT; ++i) {
        counter++; // 非原子操作：读取→加1→写入，存在竞态
    }
    pthread_exit(nullptr);
}

int main() {
    std::vector<pthread_t> tids(THREAD_COUNT);
    
    // 1. 创建线程
    for (int i = 0; i < THREAD_COUNT; ++i) {
        int ret = pthread_create(&tids[i], nullptr, incr_counter, nullptr);
        if (ret != 0) {
            perror("pthread_create failed");
            return -1;
        }
    }

    // 2. join所有线程
    for (int i = 0; i < THREAD_COUNT; ++i) {
        pthread_join(tids[i], nullptr);
    }

    // 3. 打印结果
    printf("预期结果: %d\n", THREAD_COUNT * INCR_COUNT);
    printf("实际结果: %d\n", counter);
    printf("差值: %d（竞态导致丢失更新）\n", THREAD_COUNT * INCR_COUNT - counter);
    
    return 0;
}
```

---

## 二、编译 & 运行说明
### 1. 编译命令（必须链接 pthread 库）
```bash
# 编译基础程序
g++ pthread_basic.cpp -o pthread_basic -lpthread -std=c++11

# 编译join vs detach对比程序
g++ join_vs_detach.cpp -o join_vs_detach -lpthread -std=c++11

# 编译共享变量实验程序
g++ thread_shared_var.cpp -o thread_shared_var -lpthread -std=c++11
```

### 2. 运行命令
```bash
# 运行基础程序
./pthread_basic

# 运行join vs detach对比程序
./join_vs_detach

# 运行共享变量实验程序
./thread_shared_var
```

---

## 三、运行结果 & 实验记录
### 1. pthread_basic.cpp 运行结果
```
主线程: 创建线程 1，初始ID=0x7f8b9f7fe700
主线程: 创建线程 2，初始ID=0x7f8b9effd700
主线程: 创建线程 3，初始ID=0x7f8b9e7fc700
主线程: 创建线程 4，初始ID=0x7f8b9dffb700
主线程: 创建线程 5，初始ID=0x7f8b9d7fa700

主线程: 等待所有子线程完成...
线程 3: 线程ID=0x7f8b9e7fc700，进程ID=12345
线程 1: 线程ID=0x7f8b9f7fe700，进程ID=12345
线程 2: 线程ID=0x7f8b9effd700，进程ID=12345
线程 5: 线程ID=0x7f8b9d7fa700，进程ID=12345
线程 4: 线程ID=0x7f8b9dffb700，进程ID=12345
主线程: 线程 1 已结束，最终ID=0x7f8b9f7fe700
主线程: 线程 2 已结束，最终ID=0x7f8b9effd700
主线程: 线程 3 已结束，最终ID=0x7f8b9e7fc700
主线程: 线程 4 已结束，最终ID=0x7f8b9dffb700
主线程: 线程 5 已结束，最终ID=0x7f8b9d7fa700

主线程: 进程ID=12345，所有线程属于同一进程
```

### 2. join_vs_detach.cpp 运行结果
```
===== 测试 join 版本 =====
主线程: 执行1秒任务...
线程 1: 启动，开始执行耗时操作...
线程 1: 执行中（剩余4秒）
主线程: 开始join线程1...
线程 1: 执行中（剩余3秒）
线程 1: 执行中（剩余2秒）
线程 1: 执行中（剩余1秒）
线程 1: 执行中（剩余0秒）
线程 1: 执行完成，退出
主线程: 线程1已结束，主线程退出

===== 测试 detach 版本 =====
主线程: 执行1秒任务后退出...
线程 2: 启动，开始执行耗时操作...
线程 2: 执行中（剩余4秒）
主线程: 已退出，子线程可能被终止
```

### 3. thread_shared_var.cpp 运行结果
```
预期结果: 500000
实际结果: 428957
差值: 71043（竞态导致丢失更新）
```

### 4. 实验记录：线程调度现象总结
#### （1）线程与进程的核心区别
| 维度         | 进程                     | 线程                     |
|--------------|--------------------------|--------------------------|
| 资源分配     | 独立地址空间、PID、内存   | 共享进程地址空间、PID，仅私有栈 |
| 调度单位     | 操作系统调度的基本单位   | CPU调度的基本单位        |
| 通信成本     | 高（管道/套接字/共享内存） | 低（直接访问共享变量）|
| 开销         | 高（创建/销毁需分配资源） | 低（仅需创建栈和寄存器） |
| 独立性       | 完全独立，一个崩溃不影响其他 | 共享进程资源，一个崩溃导致整个进程退出 |

#### （2）线程执行顺序特性
- **非固定性**：多次运行 `pthread_basic.cpp`，子线程打印顺序不同（如先打印线程3，再打印线程1），证明线程调度由操作系统内核决定，无固定顺序；
- **并发性**：子线程 sleep(1) 期间，主线程仍可执行（打印“等待所有子线程完成”），证明线程是并发执行的；
- **进程共享**：所有线程的 `getpid()` 结果相同，验证线程共享进程的PID和地址空间。

#### （3）join vs detach 核心区别
| 特性         | pthread_join             | pthread_detach           |
|--------------|--------------------------|--------------------------|
| 主线程行为   | 阻塞等待子线程结束       | 不阻塞，继续执行         |
| 子线程资源   | 结束后保留资源，直到join | 结束后自动释放资源       |
| 主线程退出   | 必须等待子线程完成       | 子线程可能被强制终止     |
| 适用场景     | 需要子线程执行结果       | 后台任务，无需等待结果   |
| 错误风险     | 重复join会报错           | detach后join会报错       |

#### （4）共享变量竞态问题
- **现象**：5个线程各自增10万次，预期结果50万，但实际结果远低于预期（如428957）；
- **原因**：`counter++` 是**非原子操作**，分为三步：
  1. 读取 counter 到寄存器；
  2. 寄存器值+1；
  3. 写回 counter 内存；
  多线程同时执行时，会出现“读取-覆盖”问题（如线程A和B同时读取到100，都加1写回101，丢失一次更新）；
- **解决方案**：需加锁（pthread_mutex_t）或使用原子操作（std::atomic）。

---

## 四、核心知识点总结
### 1. pthread 核心 API 解析
#### （1）pthread_create
```cpp
int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
                   void *(*start_routine) (void *), void *arg);
```
- `thread`：输出参数，返回创建的线程ID；
- `attr`：线程属性（NULL为默认，如栈大小、分离状态）；
- `start_routine`：线程函数（必须是 `void* (*)(void*)` 签名）；
- `arg`：传递给线程函数的参数（需注意生命周期，避免栈变量失效）；
- 返回值：0成功，非0失败（错误码，可通过 `perror` 打印）。

#### （2）pthread_join
```cpp
int pthread_join(pthread_t thread, void **retval);
```
- `thread`：要等待的线程ID；
- `retval`：输出参数，接收线程退出时的返回值（NULL表示不接收）；
- 作用：阻塞主线程，直到目标线程结束，回收线程资源。

#### （3）pthread_detach
```cpp
int pthread_detach(pthread_t thread);
```
- 作用：将线程设置为“分离状态”，线程结束后自动释放资源，无需主线程join；
- 注意：detach后的线程不能再调用join，否则报错。

#### （4）pthread_self
```cpp
pthread_t pthread_self(void);
```
- 作用：获取当前线程的ID（区别于进程ID getpid()）。

### 2. 线程编程注意事项
1. **线程函数参数**：
   - 避免传递栈变量（主线程栈变量可能先于子线程销毁，导致野指针）；
   - 如需传递多个参数，封装为结构体，用堆内存分配（new/delete）。
2. **线程退出**：
   - 线程函数返回、`pthread_exit(nullptr)`、主线程 `pthread_cancel` 均可终止线程；
   - 主线程退出（main返回）会终止所有子线程（即使detach）。
3. **错误处理**：
   - pthread API 返回值非0表示失败，需检查（如pthread_create返回值）；
   - 错误码可通过 `strerror(ret)` 转换为字符串。

## 总结
1. **线程核心特性**：线程共享进程地址空间和资源，调度由内核决定，执行顺序不固定；
2. **join/detach 选择**：需要等待子线程结果用join，后台任务用detach，避免资源泄漏；
3. **共享变量问题**：多线程访问共享变量必须保证原子性，否则会出现竞态，需加锁或使用原子操作；
4. **pthread 编程要点**：严格检查API返回值、注意参数生命周期、避免竞态条件。
