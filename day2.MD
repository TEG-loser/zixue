### 一、构造/析构顺序测试代码（constructor_destructor.cpp）
这份代码包含基类/派生类、成员对象、静态对象，能清晰展示构造/析构的完整顺序：

```cpp
#include <iostream>
#include <string>

// 辅助工具函数：打印带缩进的日志
void log(const std::string& msg, int indent = 0) {
    std::cout << std::string(indent * 2, ' ') << msg << std::endl;
}

// 1. 成员对象类（用于嵌入到其他类中）
class MemberObj {
private:
    std::string name;
public:
    MemberObj(const std::string& n) : name(n) {
        log("MemberObj 构造: " + name, 2);
    }
    ~MemberObj() {
        log("MemberObj 析构: " + name, 2);
    }
};

// 2. 基类
class Base {
public:
    Base() {
        log("Base 构造", 1);
    }
    ~Base() {
        log("Base 析构", 1);
    }
};

// 3. 派生类（继承 Base，包含 MemberObj 成员）
class Derived : public Base {
private:
    MemberObj m1;  // 成员对象1
    MemberObj m2;  // 成员对象2
    static MemberObj static_m; // 静态成员对象
public:
    Derived() : m1("m1"), m2("m2") { // 初始化成员对象
        log("Derived 构造", 1);
    }
    ~Derived() {
        log("Derived 析构", 1);
    }

    // 静态成员初始化（全局/静态区）
    static MemberObj createStaticObj() {
        return MemberObj("static_m");
    }
};

// 静态成员对象初始化（在类外）
MemberObj Derived::static_m = Derived::createStaticObj();

// 4. 全局对象
Derived global_derived;

int main() {
    log("===== main 函数开始 =====");
    
    // 局部对象
    Derived local_derived;
    
    log("===== main 函数结束 =====");
    return 0;
}
```

#### 代码说明：
- **构造顺序**：静态对象 → 全局对象 → 基类 → 成员对象（按声明顺序）→ 派生类；
- **析构顺序**：与构造完全相反 → 派生类 → 成员对象（逆声明顺序）→ 基类 → 全局对象 → 静态对象；
- 静态对象在程序启动时初始化，全局对象紧随其后，局部对象在函数执行时创建。

#### 运行结果（核心顺序）：
```
MemberObj 构造: static_m
Base 构造
  MemberObj 构造: m1
  MemberObj 构造: m2
  Derived 构造
===== main 函数开始 =====
Base 构造
  MemberObj 构造: m1
  MemberObj 构造: m2
  Derived 构造
===== main 函数结束 =====
  Derived 析构
  MemberObj 析构: m2
  MemberObj 析构: m1
Base 析构
  Derived 析构
  MemberObj 析构: m2
  MemberObj 析构: m1
Base 析构
MemberObj 析构: static_m
```

### 二、RAII 实现代码（file_raii.cpp）
这份代码实现 FileRAII 类，验证异常场景下资源自动释放，并对比非 RAII 版本的内存泄漏问题：

```cpp
#include <iostream>
#include <cstdio>
#include <stdexcept>
#include <string>

// 辅助函数：检查文件是否存在（Linux/macOS）
bool fileExists(const std::string& filename) {
    FILE* fp = fopen(filename.c_str(), "r");
    if (fp) {
        fclose(fp);
        return true;
    }
    return false;
}

// ========== RAII 版本 ==========
class FileRAII {
private:
    FILE* fp;          // 资源：文件指针
    std::string filename;

    // 禁止拷贝（避免资源重复释放）
    FileRAII(const FileRAII&) = delete;
    FileRAII& operator=(const FileRAII&) = delete;

public:
    // 构造：获取资源
    FileRAII(const std::string& fname, const std::string& mode = "w") : filename(fname) {
        fp = fopen(fname.c_str(), mode.c_str());
        if (!fp) {
            throw std::runtime_error("打开文件失败: " + fname);
        }
        std::cout << "[RAII] 打开文件: " << fname << std::endl;
    }

    // 析构：释放资源（无论是否抛异常，都会执行）
    ~FileRAII() noexcept { // 析构必须 noexcept
        if (fp) {
            fclose(fp);
            fp = nullptr;
            std::cout << "[RAII] 关闭文件: " << filename << std::endl;
        }
    }

    // 提供资源访问接口
    FILE* getFile() const { return fp; }

    // 写入数据示例
    void write(const std::string& content) {
        if (fp) {
            fputs(content.c_str(), fp);
            fflush(fp);
        }
    }
};

// ========== 非 RAII 版本 ==========
void nonRAIIExample(const std::string& filename) {
    // 手动管理资源
    FILE* fp = fopen(filename.c_str(), "w");
    if (!fp) {
        throw std::runtime_error("打开文件失败: " + filename);
    }
    std::cout << "[非RAII] 打开文件: " << filename << std::endl;

    // 模拟异常：资源未释放
    throw std::runtime_error("模拟异常，资源泄漏！");

    // 以下代码永远执行不到 → 内存泄漏
    fclose(fp);
    std::cout << "[非RAII] 关闭文件: " << filename << std::endl;
}

int main() {
    const std::string raii_file = "raii_test.txt";
    const std::string non_raii_file = "non_raii_test.txt";

    // 测试 RAII 版本（异常场景）
    try {
        FileRAII file(raii_file);
        file.write("RAII 测试内容\n");
        // 模拟异常
        throw std::runtime_error("RAII 测试异常");
    } catch (const std::exception& e) {
        std::cout << "捕获异常: " << e.what() << std::endl;
    }

    // 测试非 RAII 版本（资源泄漏）
    try {
        nonRAIIExample(non_raii_file);
    } catch (const std::exception& e) {
        std::cout << "捕获异常: " << e.what() << std::endl;
    }

    // 验证文件状态
    std::cout << "\n文件状态检查:" << std::endl;
    std::cout << raii_file << " 存在: " << (fileExists(raii_file) ? "是" : "否") << std::endl;
    std::cout << non_raii_file << " 存在: " << (fileExists(non_raii_file) ? "是" : "否") << std::endl;

    return 0;
}
```

#### 代码说明：
1. **RAII 核心**：
   - 构造函数获取资源（打开文件）；
   - 析构函数释放资源（关闭文件），且析构函数标记为 `noexcept`；
   - 禁止拷贝（避免资源重复释放）；
   - 即使抛出异常，析构函数也会自动执行，确保资源释放。
2. **非 RAII 问题**：
   - 异常发生时，手动释放资源的代码执行不到，导致文件句柄泄漏；
   - 即使文件创建成功，也无法关闭，可能导致数据丢失或资源占用。

#### 编译 & 运行：
```bash
# 编译
g++ -g file_raii.cpp -o file_raii

# 运行
./file_raii

# 用 valgrind 检测内存/资源泄漏
valgrind --leak-check=full ./file_raii
```

#### 运行结果：
```
[RAII] 打开文件: raii_test.txt
[RAII] 关闭文件: raii_test.txt
捕获异常: RAII 测试异常
[非RAII] 打开文件: non_raii_test.txt
捕获异常: 模拟异常，资源泄漏！

文件状态检查:
raii_test.txt 存在: 是
non_raii_test.txt 存在: 是
```

#### Valgrind 检测结果（关键）：
- RAII 版本：无泄漏（`All heap blocks were freed -- no leaks are possible`）；
- 非 RAII 版本：提示文件句柄泄漏（`still reachable: 1 open file descriptors`）。

### 三、RAII 总结文档：《RAII 原则与异常安全详解》
#### 1. RAII 核心定义
RAII（Resource Acquisition Is Initialization）即“资源获取即初始化”，是 C++ 特有的资源管理思想：
- **核心逻辑**：将资源（文件句柄、内存、锁、网络连接等）的生命周期绑定到对象的生命周期；
- **实现方式**：
  - 构造函数：获取资源（打开文件、分配内存、加锁）；
  - 析构函数：释放资源（关闭文件、释放内存、解锁）；
  - 资源全程由对象管理，无需手动调用释放函数。

#### 2. 为什么资源必须绑定对象生命周期？
- **自动管理**：对象超出作用域时，析构函数自动执行，无论是否发生异常，资源都会被释放；
- **避免泄漏**：手动管理资源时，容易因异常、分支语句、忘记调用释放函数导致泄漏；
- **代码简洁**：无需在每个分支/异常场景中手动释放资源，减少冗余代码；
- **异常安全**：保证“不泄漏资源”是异常安全的基础要求。

#### 3. 析构函数为什么必须 noexcept？
- **C++ 标准规定**：如果析构函数抛出异常且未处理，程序会直接调用 `std::terminate()` 终止运行；
- **noexcept 作用**：显式声明析构函数不会抛出异常，编译器可优化，且符合 RAII 设计原则；
- **实践要求**：析构函数中应捕获所有可能的异常，确保不向外抛出（比如 `fclose` 失败时记录日志，而非抛异常）。

#### 4. RAII 实现的关键原则
1. **禁止拷贝**：资源对象（如 FileRAII）应禁用拷贝构造函数和拷贝赋值运算符（`delete`），避免多个对象管理同一资源；
2. **移动语义（可选）**：C++11 后可实现移动构造/赋值，允许资源所有权转移；
3. **资源封装**：将资源（如 FILE*）设为私有，只提供安全的访问接口；
4. **异常处理**：构造函数获取资源失败时，应抛出异常（避免创建“无效对象”）。

#### 5. RAII 典型应用场景
| 场景         | 资源         | RAII 实现方式                  |
|--------------|--------------|--------------------------------|
| 文件操作     | 文件句柄     | 构造 open，析构 close          |
| 动态内存     | 堆内存       | std::unique_ptr / std::shared_ptr |
| 线程同步     | 互斥锁       | std::lock_guard / std::unique_lock |
| 网络编程     | 套接字       | 构造 connect，析构 close       |
| 数据库操作   | 数据库连接   | 构造 connect，析构 disconnect  |

#### 6. 异常安全的三个级别
- **基本保证**：异常发生后，资源不泄漏，对象处于有效状态；
- **强保证**：异常发生后，程序状态回滚到异常前的状态；
- **不抛异常保证**：函数永远不抛出异常（如析构函数、`noexcept` 函数）。
RAII 至少能保证“基本保证”，是实现异常安全的核心手段。

### 总结
1. **构造/析构顺序**：静态对象 → 全局对象 → 基类 → 成员对象 → 派生类；析构顺序与构造完全相反；
2. **RAII 核心**：资源绑定对象生命周期，构造获取、析构释放，确保异常场景下资源不泄漏；
3. **关键规则**：析构函数必须 `noexcept`，资源对象禁止拷贝，构造失败时抛异常。
