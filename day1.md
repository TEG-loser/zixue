完成一个关于 C++ 内存模型的学习任务，核心是理解栈、堆、静态区等内存分区的特点，通过代码实验、GDB 调试验证，并整理出模型图和总结文档。

这份代码覆盖了所有要求的变量类型，打印各变量地址，能清晰看出不同内存分区的地址分布规律：

```cpp
#include <iostream>
#include <string>

// 1. 全局变量（存储在 数据区/Data Segment，未初始化的全局变量在 BSS 区）
int global_var;                // 未初始化全局变量 → BSS 区
int global_init_var = 100;     // 初始化全局变量 → Data 区
const int global_const_var = 200; // 全局常量 → 常量区/Text 区

// 函数内静态变量演示
void static_var_demo() {
    static int static_local_var;         // 未初始化静态局部变量 → BSS 区
    static int static_init_var = 300;    // 初始化静态局部变量 → Data 区
    std::cout << "[静态局部变量] 未初始化: " << &static_local_var << std::endl;
    std::cout << "[静态局部变量] 初始化:   " << &static_init_var << std::endl;
}

int main() {
    // 2. 局部变量（存储在 栈/Stack）
    int local_var = 10;                  // 局部变量 → 栈
    double local_double = 3.14;          // 局部变量 → 栈
    std::string local_str = "local string"; // 字符串对象本身在栈，内容在堆

    std::cout << "===== 全局区/静态区 =====" << std::endl;
    std::cout << "[全局变量] 未初始化:     " << &global_var << std::endl;
    std::cout << "[全局变量] 初始化:       " << &global_init_var << std::endl;
    std::cout << "[全局常量]               " << &global_const_var << std::endl;
    
    static_var_demo(); // 打印静态局部变量地址

    std::cout << "\n===== 栈区 =====" << std::endl;
    std::cout << "[局部变量] int:          " << &local_var << std::endl;
    std::cout << "[局部变量] double:       " << &local_double << std::endl;
    std::cout << "[局部变量] string对象:   " << &local_str << std::endl;
    std::cout << "[局部变量] string内容:   " << (void*)local_str.c_str() << std::endl;

    std::cout << "\n===== 堆区 =====" << std::endl;
    // 3. 堆/Heap 分配的变量（new 关键字）
    int* heap_int = new int(400);        // 堆内存
    double* heap_double = new double(2.718); // 堆内存
    char* heap_char = new char[100]{"heap string"}; // 堆数组

    std::cout << "[堆变量] int:            " << heap_int << std::endl;
    std::cout << "[堆变量] double:         " << heap_double << std::endl;
    std::cout << "[堆变量] char数组:       " << (void*)heap_char << std::endl;

    std::cout << "\n===== 常量区 =====" << std::endl;
    // 4. 字符串常量（存储在 常量区/Text 区）
    const char* const_str = "constant string";
    std::cout << "[字符串常量]             " << (void*)const_str << std::endl;

    // 释放堆内存（避免内存泄漏）
    delete heap_int;
    delete heap_double;
    delete[] heap_char;

    return 0;
}
```

#### 代码说明：
- **全局/静态区**：全局变量、静态变量（包括函数内的静态局部变量）都在这里，初始化的在 Data 区，未初始化的在 BSS 区；
- **栈区**：函数内的局部变量（包括对象本身），地址通常是**高地址向低地址增长**；
- **堆区**：`new` 分配的内存，地址通常是**低地址向高地址增长**；
- **常量区**：字符串常量、全局常量，存储在只读的 Text 区（代码区）。

### 二、GDB 调试验证指引（附关键命令）
#### 1. 编译代码
```bash
g++ -g memory_layout.cpp -o memory_layout
```

#### 2. 启动 GDB 调试
```bash
gdb ./memory_layout
```

#### 3. 关键 GDB 命令（按顺序执行）
```gdb
# 1. 查看进程内存映射（核心命令）
info proc mappings

# 2. 运行程序，打印地址
run

# 3. 查看特定变量的地址（示例）
print &global_var
print &local_var
print heap_int

# 4. 退出 GDB
quit
```

#### 4. GDB 调试截图关键信息说明
`info proc mappings` 输出示例（不同系统略有差异）：
```
Mapped address spaces:
          Start Addr           End Addr       Size     Offset objfile
0x555555554000     0x555555555000     0x1000        0x0 /home/xxx/memory_layout
0x555555555000     0x555555556000     0x1000     0x1000 /home/xxx/memory_layout
0x555555556000     0x555555557000     0x1000     0x2000 /home/xxx/memory_layout
0x555555557000     0x555555558000     0x1000     0x2000 /home/xxx/memory_layout
0x555555558000     0x555555759000   0x201000        0x0 [heap]  # 堆区
0x7ffff7a0d000     0x7ffff7bcd000   0x2c0000        0x0 /lib/x86_64-linux-gnu/libc-2.31.so
0x7ffff7ffe000     0x7ffff7fff000     0x1000        0x0 [vvar]
0x7ffffffde000     0x7ffffffff000    0x21000        0x0 [stack] # 栈区
```
- `[heap]` 行：堆区的起始/结束地址；
- `[stack]` 行：栈区的起始/结束地址；
- 代码/常量区：通常在低地址（0x555555554000 附近）；
- 静态/全局区：在代码区和堆区之间。

### 三、C++ 内存模型图（文字版，可直接手绘）
```
┌─────────────────────────────────────────────────────────────┐
│ 高地址 → 栈区 (Stack)                                       │
│  - 存储：函数局部变量、函数参数、返回值                      │
│  - 特点：自动分配/释放，速度快，大小有限，高地址向低地址增长 │
│  - 示例：int local_var; 函数内的对象                         │
├─────────────────────────────────────────────────────────────┤
│ 堆区 (Heap)                                                 │
│  - 存储：new/malloc 分配的内存                              │
│  - 特点：手动分配/释放，速度慢，大小灵活，低地址向高地址增长 │
│  - 示例：int* p = new int; 动态数组                         │
├─────────────────────────────────────────────────────────────┤
│ 全局/静态区 (Data/BSS)                                      │
│  ├─ Data 区（初始化全局/静态变量）                          │
│  │  - 示例：int global_init = 10; static int s = 20;         │
│  └─ BSS 区（未初始化全局/静态变量，默认初始化为0）           │
│     - 示例：int global_var; static int s_var;                │
├─────────────────────────────────────────────────────────────┤
│ 常量区 (Constant)                                           │
│  - 存储：字符串常量、const 全局常量                          │
│  - 特点：只读，程序结束后释放                               │
│  - 示例："hello world"; const int c = 100;                   │
├─────────────────────────────────────────────────────────────┤
│ 低地址 → 代码区 (Text/Code)                                 │
│  - 存储：程序的机器指令、函数体                             │
│  - 特点：只读，不可修改                                     │
└─────────────────────────────────────────────────────────────┘
```

### 四、总结文档：《C++内存分区详解》
#### 1. 内存分区核心概念
C++ 程序运行时，操作系统会为进程分配一段连续的内存空间，并划分为不同区域，每个区域有明确的用途、生命周期和管理方式。核心分区包括：栈、堆、全局/静态区（Data/BSS）、常量区、代码区。

#### 2. 各分区详细说明
| 分区       | 存储内容                          | 生命周期                | 分配/释放方式       | 地址增长方向 |
|------------|-----------------------------------|-------------------------|---------------------|--------------|
| 栈 (Stack) | 局部变量、函数参数、返回值        | 函数调用时创建，结束销毁 | 编译器自动管理      | 高→低        |
| 堆 (Heap)  | new/malloc 分配的动态内存         | 手动分配，手动释放      | 程序员手动管理（new/delete） | 低→高        |
| Data 区    | 初始化的全局/静态变量             | 程序启动创建，结束销毁  | 编译器自动管理      | -            |
| BSS 区     | 未初始化的全局/静态变量           | 程序启动创建，结束销毁  | 编译器自动管理（默认置0） | -            |
| 常量区     | 字符串常量、const 全局常量        | 程序启动创建，结束销毁  | 编译器自动管理（只读） | -            |
| 代码区     | 程序的机器指令、函数体            | 程序启动加载，结束销毁  | 操作系统管理（只读） | -            |

#### 3. 关键区别与易错点
- **栈 vs 堆**：
  - 栈速度快但空间小，堆速度慢但空间大；
  - 栈溢出（如递归过深）会直接崩溃，堆溢出会抛出 `bad_alloc` 异常；
  - 栈变量无需手动释放，堆变量忘记释放会导致内存泄漏。
- **静态区注意事项**：
  - 静态变量（包括函数内的 static 变量）生命周期贯穿整个程序，仅初始化一次；
  - 全局变量和静态变量的初始化顺序不确定，避免跨文件依赖初始化。
- **常量区注意事项**：
  - 字符串常量存储在只读区，修改会导致程序崩溃（如 `char* p = "abc"; p[0] = 'A';`）；
  - const 局部变量存储在栈区（非只读），const 全局变量存储在常量区（只读）。

#### 4. 实战建议
- 优先使用栈变量（性能高，无需手动管理）；
- 堆变量必须配对使用 `new/delete`（或 `malloc/free`），避免内存泄漏；
- 大型数据（如大数组、复杂对象）用堆存储，小型数据用栈；
- 全局/静态变量尽量少用，避免耦合和初始化问题；
- 调试内存问题时，用 `valgrind` 检测内存泄漏，用 GDB 的 `info proc mappings` 查看内存分布。

### 总结
1. C++ 内存核心分为栈、堆、全局/静态区（Data/BSS）、常量区、代码区，各分区有明确的存储内容和生命周期；
2. 栈由编译器自动管理，堆需程序员手动管理，全局/静态区随程序生命周期存在；
3. 区分不同分区的关键：看变量类型（全局/局部/静态）、分配方式（自动/new）、是否只读（常量）。
