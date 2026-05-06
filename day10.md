# Day10：条件变量 + 生产者消费者模型

## 最终成果：producer_consumer.cpp
```cpp
#include <iostream>
#include <pthread.h>
#include <queue>
#include <cstdio>
#include <unistd.h>

// 队列最大容量
#define MAX_SIZE 5
// 生产/消费总次数
#define TOTAL_COUNT 10

// 共享队列（缓冲区）
std::queue<int> g_queue;

// 互斥锁：保护队列
pthread_mutex_t g_mutex;
// 条件变量：队列非空（消费者等待）
pthread_cond_t g_cond_not_empty;
// 条件变量：队列非满（生产者等待）
pthread_cond_t g_cond_not_full;

// 生产计数器
int g_produce_idx = 0;

// ===================== 生产者 =====================
void* producer(void* arg) {
    (void)arg;

    for (int i = 0; i < TOTAL_COUNT; ++i) {
        // 1. 上锁
        pthread_mutex_lock(&g_mutex);

        // 2. 等待：队列满了就等待
        while (g_queue.size() >= MAX_SIZE) {
            printf("[生产者] 队列已满，等待...\n");
            pthread_cond_wait(&g_cond_not_full, &g_mutex);
        }

        // 3. 生产数据
        int data = ++g_produce_idx;
        g_queue.push(data);
        printf("[生产者] 生产：%d，队列大小：%zu\n", data, g_queue.size());

        // 4. 通知消费者：队列有数据了
        pthread_cond_signal(&g_cond_not_empty);

        // 5. 解锁
        pthread_mutex_unlock(&g_mutex);

        // 模拟生产耗时
        usleep(200 * 1000);
    }

    return nullptr;
}

// ===================== 消费者 =====================
void* consumer(void* arg) {
    (void)arg;

    for (int i = 0; i < TOTAL_COUNT; ++i) {
        // 1. 上锁
        pthread_mutex_lock(&g_mutex);

        // 2. 等待：队列为空就等待
        while (g_queue.empty()) {
            printf("[消费者] 队列为空，等待...\n");
            pthread_cond_wait(&g_cond_not_empty, &g_mutex);
        }

        // 3. 消费数据
        int data = g_queue.front();
        g_queue.pop();
        printf("[消费者] 消费：%d，队列大小：%zu\n", data, g_queue.size());

        // 4. 通知生产者：队列有空位了
        pthread_cond_signal(&g_cond_not_full);

        // 5. 解锁
        pthread_mutex_unlock(&g_mutex);

        // 模拟消费耗时
        usleep(400 * 1000);
    }

    return nullptr;
}

// ===================== 主函数 =====================
int main() {
    // 初始化锁和条件变量
    pthread_mutex_init(&g_mutex, nullptr);
    pthread_cond_init(&g_cond_not_empty, nullptr);
    pthread_cond_init(&g_cond_not_full, nullptr);

    // 创建线程
    pthread_t producer_tid, consumer_tid;
    pthread_create(&producer_tid, nullptr, producer, nullptr);
    pthread_create(&consumer_tid, nullptr, consumer, nullptr);

    // 等待线程结束
    pthread_join(producer_tid, nullptr);
    pthread_join(consumer_tid, nullptr);

    // 销毁
    pthread_mutex_destroy(&g_mutex);
    pthread_cond_destroy(&g_cond_not_empty);
    pthread_cond_destroy(&g_cond_not_full);

    printf("\n=== 生产消费完成 ===\n");
    return 0;
}
```

## 编译运行
```bash
g++ producer_consumer.cpp -o pc -lpthread
./pc
```

## 运行结果（典型）
```
[生产者] 生产：1，队列大小：1
[消费者] 消费：1，队列大小：0
[生产者] 生产：2，队列大小：1
[生产者] 生产：3，队列大小：2
[消费者] 消费：2，队列大小：1
[生产者] 生产：4，队列大小：2
[生产者] 生产：5，队列大小：3
[消费者] 消费：3，队列大小：2
...
=== 生产消费完成 ===
```
**队列永远不会超过 5，不会空转，完美协作。**

---

# 核心知识点（Day10 必须掌握）

## 1. 条件变量作用
- 让线程**等待某个条件成立**
- 避免**空轮询浪费 CPU**
- 必须 **配合互斥锁一起使用**

## 2. 关键 API
```cpp
// 等待（自动解锁 + 阻塞，被唤醒后自动重新上锁）
pthread_cond_wait(&cond, &mutex);

// 唤醒一个等待的线程
pthread_cond_signal(&cond);

// 唤醒所有
pthread_cond_broadcast(&cond);
```

## 3. 为什么要用 while 而不是 if？
```cpp
// 正确
while (队列为空)
    pthread_cond_wait(...);

// 错误
if (队列为空)
    pthread_cond_wait(...);
```
原因：**虚假唤醒（spurious wakeup）**
操作系统可能在没有 signal 的情况下唤醒线程，必须用 while 再次检查条件。

## 4. 生产者消费者模型流程（背下来）
1. **生产者**
   - 上锁
   - 满了就 `wait(非满)`
   - 生产
   - `signal(非空)`
   - 解锁

2. **消费者**
   - 上锁
   - 空了就 `wait(非空)`
   - 消费
   - `signal(非满)`
   - 解锁

## 总结
- 条件变量 = **线程间同步** 的核心工具
- `cond_wait` 必须配合 **mutex**
- 必须用 **while** 判断条件
- 生产者消费者是 **最经典的多线程同步模型**
