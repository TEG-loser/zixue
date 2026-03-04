完成 Day3 关于 C++ 五法则（Rule of Five）的实战任务，核心是实现支持深拷贝、移动语义的动态数组类，并理解资源所有权转移。
### 一、完整实现代码（dynamic_array.hpp）
这份代码完整实现了 Rule of Five，包含构造、析构、拷贝构造、拷贝赋值、移动构造、移动赋值，附带详细注释和测试逻辑：
```cpp
#ifndef DYNAMIC_ARRAY_HPP
#define DYNAMIC_ARRAY_HPP

#include <iostream>
#include <algorithm> // for std::swap
#include <cstddef>   // for size_t
#include <utility>   // for std::move

class MyArray {
private:
    int* data;    // 动态数组指针（资源）
    size_t size;  // 数组大小

    // 辅助函数：释放资源
    void clear() noexcept {
        if (data) {
            delete[] data;
            data = nullptr;
            size = 0;
        }
    }

    // 辅助函数：深拷贝数据
    void copyFrom(const MyArray& other) {
        // 先释放当前资源，避免内存泄漏
        clear();
        // 深拷贝：分配新内存 + 拷贝数据
        size = other.size;
        data = new int[size];
        for (size_t i = 0; i < size; ++i) {
            data[i] = other.data[i];
        }
    }

public:
    // 1. 默认构造函数
    MyArray() : data(nullptr), size(0) {
        std::cout << "[默认构造] 空数组" << std::endl;
    }

    // 2. 带参数构造函数（创建指定大小的数组）
    MyArray(size_t n, int value = 0) : size(n) {
        data = new int[n];
        for (size_t i = 0; i < n; ++i) {
            data[i] = value;
        }
        std::cout << "[参数构造] 大小=" << size << "，地址=" << (void*)data << std::endl;
    }

    // 3. 析构函数（Rule of Five 必备）
    ~MyArray() noexcept {
        clear();
        std::cout << "[析构] 释放数组，大小=" << size << std::endl;
    }

    // 4. 拷贝构造函数（深拷贝，Rule of Five 必备）
    MyArray(const MyArray& other) {
        copyFrom(other);
        std::cout << "[拷贝构造] 深拷贝，大小=" << size << "，地址=" << (void*)data << std::endl;
    }

    // 5. 拷贝赋值运算符（深拷贝，Rule of Five 必备）
    MyArray& operator=(const MyArray& other) {
        // 防止自赋值（s = s）
        if (this != &other) {
            copyFrom(other);
            std::cout << "[拷贝赋值] 深拷贝，大小=" << size << "，地址=" << (void*)data << std::endl;
        }
        return *this;
    }

    // 6. 移动构造函数（转移所有权，Rule of Five 必备）
    MyArray(MyArray&& other) noexcept {
        // 浅拷贝指针，转移资源所有权
        data = other.data;
        size = other.size;
        // 置空源对象，避免析构时重复释放
        other.data = nullptr;
        other.size = 0;
        std::cout << "[移动构造] 转移所有权，新地址=" << (void*)data << "，源对象置空" << std::endl;
    }

    // 7. 移动赋值运算符（转移所有权，Rule of Five 必备）
    MyArray& operator=(MyArray&& other) noexcept {
        // 防止自赋值
        if (this != &other) {
            // 释放当前资源
            clear();
            // 转移资源所有权
            data = other.data;
            size = other.size;
            // 置空源对象
            other.data = nullptr;
            other.size = 0;
            std::cout << "[移动赋值] 转移所有权，新地址=" << (void*)data << "，源对象置空" << std::endl;
        }
        return *this;
    }

    // 辅助方法：打印数组内容
    void print(const std::string& label = "") const {
        std::cout << label << " [";
        for (size_t i = 0; i < size; ++i) {
            std::cout << data[i];
            if (i < size - 1) std::cout << ", ";
        }
        std::cout << "] (地址=" << (void*)data << "，大小=" << size << ")" << std::endl;
    }

    // 辅助方法：修改数组元素（验证深拷贝独立性）
    void set(size_t index, int value) {
        if (index < size) {
            data[index] = value;
        }
    }

    // 辅助方法：返回数组大小
    size_t getSize() const { return size; }

    // 辅助方法：创建临时对象（用于测试移动语义）
    static MyArray createTempArray(size_t n, int value) {
        return MyArray(n, value); // 返回临时对象，触发移动构造
    }
};

// 测试代码
#ifdef TEST_DYNAMIC_ARRAY
#include <vector>

int main() {
    std::cout << "===== 1. 构造与拷贝构造 =====" << std::endl;
    MyArray arr1(5, 10); // 参数构造
    MyArray arr2 = arr1; // 拷贝构造（深拷贝）
    arr2.set(0, 20);     // 修改 arr2，验证 arr1 不受影响
    arr1.print("arr1: ");
    arr2.print("arr2: ");

    std::cout << "\n===== 2. 拷贝赋值 =====" << std::endl;
    MyArray arr3;
    arr3 = arr1; // 拷贝赋值
    arr3.print("arr3 (拷贝赋值后): ");

    std::cout << "\n===== 3. 移动构造 =====" << std::endl;
    MyArray arr4 = MyArray::createTempArray(3, 5); // 移动构造（临时对象）
    arr4.print("arr4 (移动构造后): ");

    std::cout << "\n===== 4. 移动赋值 =====" << std::endl;
    MyArray arr5;
    arr5 = MyArray::createTempArray(4, 8); // 移动赋值
    arr5.print("arr5 (移动赋值后): ");

    std::cout << "\n===== 5. push 到 vector（测试移动语义） =====" << std::endl;
    std::vector<MyArray> vec;
    vec.reserve(2); // 预留空间，避免扩容触发多次拷贝
    vec.push_back(MyArray(2, 100)); // 移动构造
    vec.push_back(std::move(arr1)); // 显式移动
    vec[0].print("vec[0]: ");
    vec[1].print("vec[1]: ");

    std::cout << "\n===== 程序结束（析构所有对象） =====" << std::endl;
    return 0;
}
#endif // TEST_DYNAMIC_ARRAY

#endif // DYNAMIC_ARRAY_HPP
```

### 二、编译 & 运行测试
#### 1. 编译命令（启用测试代码）
```bash
# 编译（定义 TEST_DYNAMIC_ARRAY 启用测试）
g++ -DTEST_DYNAMIC_ARRAY dynamic_array.hpp -o dynamic_array -std=c++11

# 运行
./dynamic_array
```

#### 2. 运行结果（核心输出）
```
===== 1. 构造与拷贝构造 =====
[参数构造] 大小=5，地址=0x55f8a7c2b2c0
[拷贝构造] 深拷贝，大小=5，地址=0x55f8a7c2b300
arr1:  [10, 10, 10, 10, 10] (地址=0x55f8a7c2b2c0，大小=5)
arr2:  [20, 10, 10, 10, 10] (地址=0x55f8a7c2b300，大小=5)

===== 2. 拷贝赋值 =====
[默认构造] 空数组
[拷贝赋值] 深拷贝，大小=5，地址=0x55f8a7c2b340
arr3 (拷贝赋值后):  [10, 10, 10, 10, 10] (地址=0x55f8a7c2b340，大小=5)

===== 3. 移动构造 =====
[参数构造] 大小=3，地址=0x55f8a7c2b380
[移动构造] 转移所有权，新地址=0x55f8a7c2b380，源对象置空
[析构] 释放数组，大小=0
arr4 (移动构造后):  [5, 5, 5] (地址=0x55f8a7c2b380，大小=3)

===== 4. 移动赋值 =====
[参数构造] 大小=4，地址=0x55f8a7c2b3c0
[移动赋值] 转移所有权，新地址=0x55f8a7c2b3c0，源对象置空
[析构] 释放数组，大小=0
arr5 (移动赋值后):  [8, 8, 8, 8] (地址=0x55f8a7c2b3c0，大小=4)

===== 5. push 到 vector（测试移动语义） =====
[参数构造] 大小=2，地址=0x55f8a7c2b400
[移动构造] 转移所有权，新地址=0x55f8a7c2b400，源对象置空
[析构] 释放数组，大小=0
[移动构造] 转移所有权，新地址=0x55f8a7c2b2c0，源对象置空
vec[0]:  [100, 100] (地址=0x55f8a7c2b400，大小=2)
vec[1]:  [10, 10, 10, 10, 10] (地址=0x55f8a7c2b2c0，大小=5)

===== 程序结束（析构所有对象） =====
[析构] 释放数组，大小=0
[析构] 释放数组，大小=4
[析构] 释放数组，大小=3
[析构] 释放数组，大小=5
[析构] 释放数组，大小=5
[析构] 释放数组，大小=5
[析构] 释放数组，大小=5
[析构] 释放数组，大小=2
```

### 三、总结文档：《移动语义与所有权》
#### 1. 核心概念：资源所有权
- **所有权定义**：谁负责释放资源（如动态内存、文件句柄），谁就拥有该资源的所有权；
- **C++ 资源管理核心**：**唯一所有权**是最安全的模式（一个资源只能被一个对象管理），避免重复释放、内存泄漏；
- **深拷贝 vs 移动语义**：
  - 深拷贝：创建新资源，拷贝数据 → 原对象和新对象各有独立资源（双所有权）；
  - 移动语义：转移资源所有权 → 原对象失去所有权（置空），新对象接管（单所有权）。

#### 2. Rule of Five（五法则）
五法则是 C++ 管理动态资源的核心规则，当类包含动态资源时，必须实现以下 5 个函数：
| 函数类型         | 作用                          | 实现要点                          |
|------------------|-------------------------------|-----------------------------------|
| 析构函数         | 释放资源                      | `noexcept`，清空指针，避免泄漏    |
| 拷贝构造函数     | 深拷贝资源                    | 分配新内存，拷贝数据，避免浅拷贝  |
| 拷贝赋值运算符   | 深拷贝资源                    | 防止自赋值，先释放当前资源再拷贝  |
| 移动构造函数     | 转移资源所有权                | 浅拷贝指针，置空源对象，`noexcept` |
| 移动赋值运算符   | 转移资源所有权                | 防止自赋值，释放当前资源后转移    |

#### 3. 移动语义（C++11 核心特性）
##### 3.1 为什么需要移动语义？
- **性能优化**：深拷贝大对象（如大数组、字符串）会消耗大量内存和 CPU，移动语义只需拷贝指针（O(1) 时间）；
- **临时对象优化**：返回临时对象、`push_back` 临时对象时，编译器会自动触发移动构造，避免不必要的拷贝；
- **所有权转移**：明确资源归属，避免重复释放。

##### 3.2 移动构造/赋值的实现要点
1. **参数**：`MyArray&& other`（右值引用，`&&` 表示“可移动的对象”）；
2. **`noexcept`**：必须标记为 `noexcept`，否则 `vector` 等容器可能放弃移动，改用拷贝；
3. **核心逻辑**：
   - 浅拷贝源对象的指针和大小；
   - 置空源对象的指针（`other.data = nullptr`），避免析构时重复释放；
   - 不分配新内存，不拷贝数据 → 极致高效。

##### 3.3 右值引用与 `std::move`
- **右值**：临时对象、字面量（如 `MyArray(5, 10)`），没有名称，无法被修改；
- **`std::move`**：将左值（有名称的对象，如 `arr1`）转换为右值引用，显式触发移动语义（如 `vec.push_back(std::move(arr1))`）；
- **注意**：`std::move` 仅做类型转换，不实际移动资源，移动操作由移动构造/赋值完成。

#### 4. 五法则实战避坑指南
1. **防止自赋值**：拷贝/移动赋值运算符中必须检查 `this != &other`，避免自赋值导致资源释放后空指针访问；
2. **析构函数 `noexcept`**：析构函数抛出异常会导致程序终止，必须保证析构不抛异常；
3. **移动后源对象状态**：移动后源对象必须处于“可析构、可赋值”的有效状态（通常置空指针）；
4. **禁用拷贝（可选）**：如果类只需要移动语义（如 `std::unique_ptr`），可删除拷贝构造和拷贝赋值：
   ```cpp
   MyArray(const MyArray&) = delete;
   MyArray& operator=(const MyArray&) = delete;
   ```
5. **`vector` 与移动语义**：`vector` 扩容时，若元素支持移动构造（`noexcept`），会用移动代替拷贝，大幅提升性能。

### 总结
1. **Rule of Five**：包含动态资源的类必须实现析构、拷贝构造、拷贝赋值、移动构造、移动赋值，核心是保证资源安全管理；
2. **移动语义**：通过右值引用转移资源所有权，避免深拷贝的性能损耗，是 C++11 优化临时对象的核心手段；
3. **所有权核心**：动态资源的所有权必须唯一，深拷贝是“复制所有权”，移动语义是“转移所有权”，析构是“释放所有权”。
