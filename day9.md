# Day9：互斥锁：线程安全计数器


## 最终成果：thread_safe_counter.cpp
```cpp
#include <iostream>
#include <pthread.h>
#include <vector>
#include <cstdio>

// 线程数量 & 每个线程自增次数
const int THREAD_NUM = 5;
const int LOOP_NUM = 100000;

// 共享计数器
int counter = 0;

// 互斥锁：保护共享变量
pthread_mutex_t mutex;

// 线程函数：安全自增
void* thread_func(void* arg) {
    (void)arg;

    for (int i = 0; i < LOOP_NUM; ++i) {
        // 上锁：进入临界区
        pthread_mutex_lock(&mutex);

        // 临界区：同一时间只有一个线程能执行
        counter++;

        // 解锁：离开临界区
        pthread_mutex_unlock(&mutex);
    }
    return nullptr;
}

int main() {
    // 1. 初始化互斥锁（默认属性）
    pthread_mutex_init(&mutex, nullptr);

    // 2. 创建线程
    std::vector<pthread_t> tids(THREAD_NUM);
    for (int i = 0; i < THREAD_NUM; ++i) {
        pthread_create(&tids[i], nullptr, thread_func, nullptr);
    }

    // 3. 等待所有线程结束
    for (int i = 0; i < THREAD_NUM; ++i) {
        pthread_join(tids[i], nullptr);
    }

    // 4. 输出结果
    printf("预期结果：%d\n", THREAD_NUM * LOOP_NUM);
    printf("实际结果：%d\n", counter);

    // 5. 销毁互斥锁
    pthread_mutex_destroy(&mutex);

    return 0;
}
```

## 编译运行（必须链接 pthread）
```bash
g++ thread_safe_counter.cpp -o safe_counter -lpthread
./safe_counter
```

## 运行结果
```
预期结果：500000
实际结果：500000
```
**完全正确，没有数据丢失！**

---

# 核心知识点讲解（Day9 必须掌握）

## 1. 为什么要互斥锁？
昨天 Day8 的无锁版本：
- `counter++` 不是原子操作
- 多线程同时读写 → **数据竞争（race condition）**
- 结果永远小于预期

今天加锁后：
- 同一时间**只有一个线程**能执行 `counter++`
- 完全避免竞争 → 结果 100% 正确

---

## 2. 互斥锁核心 API（必须背）
```cpp
// 初始化
pthread_mutex_init(&mutex, nullptr);

// 上锁（阻塞直到获取锁）
pthread_mutex_lock(&mutex);

// 解锁
pthread_mutex_unlock(&mutex);

// 销毁
pthread_mutex_destroy(&mutex);
```

---

## 3. 临界区（Critical Section）
**被锁保护的代码段**叫临界区。
规则：
- **临界区越小越好**
- 同一时间只能有一个线程进入
- 必须保证**加锁 ↔ 解锁成对出现**

---

## 4. 锁的作用总结
- 保证**原子性**
- 保证**可见性**
- 保证**有序性**
→ 实现**线程安全**

---

## 5. 常见错误（一定要避免）
1. **忘记解锁** → 死锁
2. **重复解锁** → 程序崩溃
3. **临界区太大** → 性能差
4. **不同线程用不同锁** → 等于没加锁

## 总结
- 互斥锁 = 解决多线程共享数据竞争的**标准方案**
- `lock()` + `unlock()` 必须成对
- 临界区代码必须**最小化**
- 加锁后计数器结果**100% 正确**
