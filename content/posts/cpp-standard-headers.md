+++
title = 'c++各个标准版本的头文件'
date = 2024-01-06T00:00:00+00:00
+++
我来为你详细介绍C++各个标准版本引入的头文件，这对理解C++演进非常重要：

## C++98/03 标准头文件

### C标准库头文件 (C-style)
```cpp
#include <cassert>    // 断言
#include <cctype>     // 字符处理
#include <cerrno>     // 错误处理
#include <cfloat>     // 浮点数限制
#include <ciso646>    // 替代操作符宏
#include <climits>    // 整数限制
#include <clocale>    // 本地化
#include <cmath>      // 数学函数
#include <csetjmp>    // 跳转
#include <csignal>    // 信号处理
#include <cstdarg>    // 可变参数
#include <cstddef>    // 标准定义
#include <cstdio>     // 标准I/O
#include <cstdlib>    // 标准库
#include <cstring>    // 字符串处理
#include <ctime>      // 时间处理
#include <cwchar>     // 宽字符
#include <cwctype>    // 宽字符类型
```

### C++标准库头文件
```cpp
#include <algorithm>   // 算法
#include <bitset>      // 位集
#include <complex>     // 复数
#include <deque>       // 双端队列
#include <exception>   // 异常处理
#include <fstream>     // 文件流
#include <functional>  // 函数对象
#include <iomanip>     // I/O 操纵器
#include <ios>         // I/O 基类
#include <iosfwd>      // I/O 前向声明
#include <iostream>    // 标准I/O流
#include <istream>     // 输入流
#include <iterator>    // 迭代器
#include <limits>      // 数值限制
#include <list>        // 链表
#include <locale>      // 本地化
#include <map>         // 关联容器 map
#include <memory>      // 内存管理
#include <new>         // 动态内存分配
#include <numeric>     // 数值算法
#include <ostream>     // 输出流
#include <queue>       // 队列
#include <set>         // 集合
#include <sstream>     // 字符串流
#include <stack>       // 栈
#include <stdexcept>   // 标准异常
#include <streambuf>   // 流缓冲
#include <string>      // 字符串
#include <typeinfo>    // 类型信息
#include <utility>     // 实用程序
#include <valarray>    // 数值数组
#include <vector>      // 向量
```

## C++11 新增头文件

### 核心语言支持
```cpp
#include <initializer_list>  // 初始化列表
#include <type_traits>       // 类型特征
#include <typeindex>         // 类型索引
```

### 多线程支持
```cpp
#include <atomic>            // 原子操作
#include <thread>            // 线程
#include <mutex>             // 互斥量
#include <condition_variable> // 条件变量
#include <future>            // 异步操作
```

### 智能指针和内存
```cpp
// memory 头文件扩展了智能指针
// std::unique_ptr, std::shared_ptr, std::weak_ptr
```

### 容器
```cpp
#include <array>             // 固定大小数组
#include <forward_list>      // 单向链表
#include <unordered_map>     // 无序关联容器
#include <unordered_set>     // 无序集合
```

### 其他实用工具
```cpp
#include <chrono>            // 时间库
#include <codecvt>           // 代码转换
#include <random>            // 随机数生成
#include <ratio>             // 编译期有理数
#include <regex>             // 正则表达式
#include <system_error>      // 系统错误
#include <tuple>             // 元组
```

### 使用示例
```cpp
// C++11 特性示例
#include <vector>
#include <memory>
#include <thread>
#include <chrono>
#include <random>

int main() {
    // 智能指针
    auto ptr = std::make_unique<int>(42);
    
    // 线程
    std::thread t([]() { std::cout << "Hello from thread\n"; });
    
    // 时间
    auto start = std::chrono::high_resolution_clock::now();
    
    // 随机数
    std::random_device rd;
    std::mt19937 gen(rd());
    std::uniform_int_distribution<> dis(1, 100);
    
    t.join();
    return 0;
}
```

## C++14 新增头文件

C++14主要是对C++11的增强，没有引入新的头文件，但扩展了现有头文件的功能：

```cpp
// 扩展了 <memory>
std::make_unique<T>() // 现在可用

// 扩展了 <chrono>
using namespace std::chrono_literals;
auto duration = 100ms; // 字面量后缀

// 扩展了 <type_traits>
// 添加了 _t 和 _v 别名模板
```

## C++17 新增头文件

### 文件系统
```cpp
#include <filesystem>        // 文件系统操作
```

### 内存资源
```cpp
#include <memory_resource>   // 多态内存资源
```

### 字符串处理
```cpp
#include <string_view>       // 字符串视图
```

### 数学
```cpp
#include <cmath>             // 扩展了特殊数学函数
```

### 其他
```cpp
#include <any>               // 类型安全的容器
#include <optional>          // 可选值
#include <variant>           // 类型安全的联合
#include <execution>         // 并行算法执行策略
```

### 使用示例
```cpp
#include <filesystem>
#include <optional>
#include <string_view>
#include <variant>

namespace fs = std::filesystem;

std::optional<std::string> read_file(std::string_view filename) {
    if (fs::exists(filename)) {
        // 读取文件逻辑
        return "file content";
    }
    return std::nullopt;
}

int main() {
    std::variant<int, std::string> v = 42;
    
    if (auto content = read_file("test.txt")) {
        std::cout << *content << std::endl;
    }
    
    return 0;
}
```

## C++20 新增头文件

### 概念和约束
```cpp
#include <concepts>          // 概念库
```

### 协程支持
```cpp
#include <coroutine>         // 协程支持
```

### 范围和视图
```cpp
#include <ranges>            // 范围库
```

### 格式化
```cpp
#include <format>            // 字符串格式化
```

### 数值处理
```cpp
#include <numbers>           // 数学常数
#include <bit>               // 位操作工具
```

### 日历和时区
```cpp
// <chrono> 扩展了日历和时区支持
```

### 同步原语
```cpp
#include <barrier>           // 屏障
#include <latch>             // 闩锁
#include <semaphore>         // 信号量
#include <stop_token>        // 停止令牌
```

### 源码位置
```cpp
#include <source_location>   // 源码位置信息
```

### 比较
```cpp
#include <compare>           // 三路比较
```

### 使用示例
```cpp
#include <ranges>
#include <format>
#include <concepts>
#include <numbers>

template<std::integral T>
T add(T a, T b) {
    return a + b;
}

int main() {
    std::vector<int> numbers = {1, 2, 3, 4, 5};
    
    // 范围算法
    auto even_squares = numbers 
        | std::views::filter([](int x) { return x % 2 == 0; })
        | std::views::transform([](int x) { return x * x; });
    
    // 格式化输出
    for (auto x : even_squares) {
        std::cout << std::format("Square: {}\n", x);
    }
    
    // 数学常数
    std::cout << std::format("Pi: {}\n", std::numbers::pi);
    
    return 0;
}
```

## C++23 新增头文件

### 新容器
```cpp
#include <flat_map>          // 平坦映射
#include <flat_set>          // 平坦集合
```

### 多维数组支持
```cpp
#include <mdspan>            // 多维span
```

### 预期值
```cpp
#include <expected>          // 预期值类型
```

### 栈跟踪
```cpp
#include <stacktrace>        // 栈跟踪
```

### 标准库模块支持
```cpp
#include <stdfloat>          // 标准浮点类型
```

### 使用示例
```cpp
#include <expected>
#include <mdspan>
#include <print>  // 可能的新增

std::expected<int, std::string> divide(int a, int b) {
    if (b == 0) {
        return std::unexpected("Division by zero");
    }
    return a / b;
}

int main() {
    auto result = divide(10, 2);
    if (result) {
        std::println("Result: {}", *result);  // C++23 print
    } else {
        std::println("Error: {}", result.error());
    }
    
    return 0;
}
```

## 编译器支持检查

```cpp
// 检查C++版本
#if __cplusplus >= 202302L
    // C++23
    #include <expected>
    #include <mdspan>
#elif __cplusplus >= 202002L
    // C++20
    #include <ranges>
    #include <format>
    #include <concepts>
#elif __cplusplus >= 201703L
    // C++17
    #include <filesystem>
    #include <optional>
    #include <string_view>
#elif __cplusplus >= 201402L
    // C++14
#elif __cplusplus >= 201103L
    // C++11
    #include <thread>
    #include <memory>
    #include <chrono>
#endif
```

## 实际开发建议

### 1. 渐进式采用
```cpp
// 从稳定的标准开始
#include <vector>     // C++98
#include <memory>     // C++11 智能指针
#include <optional>   // C++17
#include <ranges>     // C++20 (如果编译器支持)
```

### 2. 特性检测
```cpp
#ifdef __cpp_lib_ranges
    #include <ranges>
    // 使用ranges功能
#endif

#ifdef __cpp_lib_format
    #include <format>
    // 使用format功能
#endif
```

### 3. 平台兼容性
```cpp
// 为不同平台提供兼容性
#if defined(_MSC_VER) && _MSC_VER >= 1920
    // Visual Studio 2019+
#elif defined(__GNUC__) && __GNUC__ >= 10
    // GCC 10+
#elif defined(__clang__) && __clang_major__ >= 12
    // Clang 12+
#endif
```

这样的头文件演进展示了C++语言的不断发展，每个版本都在原有基础上添加新功能，同时保持向后兼容性。
