+++
title = 'c++各个标准版本的头文件'
date = 2024-01-07T00:00:00+00:00
+++
# C++编译器支持情况详细对比

## 主要编译器概览

### 1. GCC (GNU Compiler Collection)
- **开源免费**
- **跨平台支持**：Linux、Windows (MinGW)、macOS
- **标准支持积极**
- **性能优秀**

### 2. Clang/LLVM
- **现代架构**
- **优秀的错误信息**
- **苹果主推**：macOS 默认编译器
- **模块化设计**

### 3. MSVC (Microsoft Visual C++)
- **Windows 原生**
- **Visual Studio 集成**
- **微软扩展丰富**
- **调试工具强大**

### 4. Intel C++ Compiler (ICC/ICX)
- **高性能优化**
- **Intel 硬件优化**
- **商业编译器**

## C++98/03 支持情况

### 完全支持 (所有现代编译器)
```cpp
// 所有编译器都完全支持C++98/03
GCC 3.0+    ✅ 完全支持
Clang 1.0+  ✅ 完全支持  
MSVC 6.0+   ✅ 完全支持
ICC 7.0+    ✅ 完全支持
```

## C++11 支持情况

### GCC
```cpp
GCC 4.3  (2008) - 部分支持 (auto, decltype, static_assert)
GCC 4.4  (2009) - 初始化列表, 变参模板
GCC 4.5  (2010) - Lambda表达式
GCC 4.6  (2011) - 范围for, nullptr, constexpr
GCC 4.7  (2012) - 委托构造函数, override/final
GCC 4.8  (2013) - 线程局部存储
GCC 4.9  (2014) - 正则表达式完整支持
GCC 5.0  (2015) - ✅ C++11 完全支持
```

### Clang
```cpp
Clang 2.9  (2011) - 部分支持
Clang 3.0  (2011) - 大部分C++11特性
Clang 3.1  (2012) - ✅ C++11 完全支持
Clang 3.3+ (2013) - 稳定的C++11支持
```

### MSVC
```cpp
VS 2010 (MSVC 16.0) - 部分支持 (auto, lambda, static_assert)
VS 2012 (MSVC 17.0) - 更多特性 (range-for, override)
VS 2013 (MSVC 18.0) - 委托构造函数, 变参模板
VS 2015 (MSVC 19.0) - ✅ C++11 完全支持
```

### 使用建议
```bash
# 最低版本要求 (C++11)
GCC 5.0+
Clang 3.3+
MSVC 2015+
```

## C++14 支持情况

### GCC
```cpp
GCC 4.8  - 部分支持 (返回类型推导)
GCC 4.9  - 通用lambda, 变量模板
GCC 5.0  - ✅ C++14 完全支持
```

### Clang  
```cpp
Clang 3.4 - ✅ C++14 完全支持
```

### MSVC
```cpp
VS 2015 Update 3 - ✅ C++14 完全支持
```

### 编译器标志
```bash
# GCC
g++ -std=c++14 file.cpp

# Clang  
clang++ -std=c++14 file.cpp

# MSVC
cl /std:c++14 file.cpp
```

## C++17 支持情况

### GCC
```cpp
GCC 5.0  - 部分支持 (嵌套命名空间)
GCC 6.0  - 更多特性
GCC 7.0  - ✅ C++17 完全支持
GCC 8.0+ - 稳定的C++17支持
```

### Clang
```cpp
Clang 4.0 - 部分支持
Clang 5.0 - ✅ C++17 完全支持
```

### MSVC
```cpp
VS 2017 15.0 - 部分支持
VS 2017 15.3 - 大部分特性
VS 2017 15.7 - ✅ C++17 完全支持
```

### C++17特性支持对比
```cpp
特性                    GCC 7.0  Clang 5.0  MSVC 2017.7
结构化绑定              ✅       ✅         ✅
if constexpr           ✅       ✅         ✅
折叠表达式              ✅       ✅         ✅
std::optional          ✅       ✅         ✅
std::variant           ✅       ✅         ✅
std::string_view       ✅       ✅         ✅
文件系统库              ✅       ✅         ✅
并行算法                ✅       ❌         ✅
```

## C++20 支持情况

### GCC
```cpp
GCC 8.0  - 部分支持 (概念实验版)
GCC 9.0  - 更多特性
GCC 10.0 - 大部分C++20特性
GCC 11.0 - ✅ C++20 几乎完全支持
GCC 12.0+- 稳定的C++20支持
```

### Clang
```cpp
Clang 10.0 - 部分支持
Clang 12.0 - 大部分特性
Clang 14.0 - ✅ C++20 几乎完全支持
Clang 15.0+- 持续完善
```

### MSVC
```cpp
VS 2019 16.0 - 部分支持
VS 2019 16.8 - 大部分特性  
VS 2019 16.11- ✅ C++20 几乎完全支持
VS 2022      - 持续完善
```

### C++20核心特性支持
```cpp
特性                    GCC 11   Clang 14   MSVC 2019.11
概念(Concepts)          ✅       ✅         ✅
模块(Modules)           🔶       🔶         ✅
协程(Coroutines)        ✅       ✅         ✅
范围(Ranges)            ✅       ✅         ✅
std::format            ✅       ❌         ✅
三路比较(<=>)           ✅       ✅         ✅
指定初始化器            ✅       ✅         ✅
consteval              ✅       ✅         ✅
constinit              ✅       ✅         ✅
```

### 模块支持状态
```cpp
// GCC 11+
import std.core;  // 实验性支持

// Clang 14+  
import std;       // 实验性支持

// MSVC 2019.16.8+
import std.core;  // 较好支持
```

## C++23 支持情况

### GCC
```cpp
GCC 12.0 - 部分支持 (显式对象参数)
GCC 13.0 - 更多特性
GCC 14.0 - ✅ 大部分C++23特性
```

### Clang
```cpp
Clang 15.0 - 部分支持
Clang 16.0 - 更多特性  
Clang 17.0 - ✅ 大部分C++23特性
```

### MSVC
```cpp
VS 2022 17.4 - 部分支持
VS 2022 17.6+- 持续添加特性
```

### C++23特性支持
```cpp
特性                    GCC 14   Clang 17   MSVC 2022.6
显式对象参数            ✅       ✅         ✅
if consteval           ✅       ✅         ✅
多维下标运算符          ✅       ✅         ✅
std::expected          ✅       ✅         ✅
std::mdspan            ✅       ✅         ✅
std::flat_map          🔶       🔶         🔶
std::print             🔶       ❌         🔶
```

## 各编译器版本对应表

### GCC版本历史
```cpp
版本    发布年份    主要C++特性
4.3     2008       初期C++11支持
4.9     2014       完整C++11支持
5.1     2015       完整C++14支持  
7.1     2017       完整C++17支持
11.1    2021       完整C++20支持
14.1    2024       大部分C++23支持
```

### Clang版本历史
```cpp
版本    发布年份    主要C++特性
3.3     2013       完整C++11支持
3.4     2014       完整C++14支持
5.0     2017       完整C++17支持
14.0    2022       大部分C++20支持
17.0    2023       大部分C++23支持
```

### MSVC版本历史
```cpp
VS版本       MSVC版本    年份    主要C++特性
2015        19.0        2015    完整C++11/14支持
2017        19.1        2017    完整C++17支持
2019        19.2        2019    大部分C++20支持
2022        19.3        2022    持续C++20/23支持
```

## 实际开发建议

### 1. 保守选择 (最大兼容性)
```cpp
标准: C++17
编译器最低版本:
- GCC 7.0+
- Clang 5.0+  
- MSVC 2017.7+
```

### 2. 现代选择 (平衡新特性和稳定性)
```cpp
标准: C++20
编译器推荐版本:
- GCC 11.0+
- Clang 14.0+
- MSVC 2019.11+ 或 VS 2022
```

### 3. 前沿选择 (最新特性)
```cpp
标准: C++23
编译器最新版本:
- GCC 14.0+
- Clang 17.0+
- MSVC 2022 最新版
```

## 在线编译器支持

### Compiler Explorer (godbolt.org)
```cpp
GCC:     4.4.7 到最新版本
Clang:   3.0.0 到最新版本  
MSVC:    19.0 到最新版本
ICC:     13.0.1 到最新版本
```

### 在线IDE
```cpp
Replit:    GCC 9.3, 支持到C++17
CodePen:   现代浏览器编译器
Wandbox:   多种编译器版本
```

## CMake版本要求

### C++标准与CMake
```cmake
# C++11
cmake_minimum_required(VERSION 3.1)
set(CMAKE_CXX_STANDARD 11)

# C++14  
cmake_minimum_required(VERSION 3.1)
set(CMAKE_CXX_STANDARD 14)

# C++17
cmake_minimum_required(VERSION 3.8)
set(CMAKE_CXX_STANDARD 17)

# C++20
cmake_minimum_required(VERSION 3.12)
set(CMAKE_CXX_STANDARD 20)

# C++23
cmake_minimum_required(VERSION 3.20)
set(CMAKE_CXX_STANDARD 23)
```

## 特性检测宏

### 编译器检测
```cpp
// 编译器检测
#ifdef __GNUC__
    // GCC编译器
    #define GCC_VERSION (__GNUC__ * 10000 + __GNUC_MINOR__ * 100)
#endif

#ifdef __clang__
    // Clang编译器
    #define CLANG_VERSION (__clang_major__ * 10000 + __clang_minor__ * 100)
#endif

#ifdef _MSC_VER
    // MSVC编译器
    // _MSC_VER 值对应版本
#endif
```

### C++标准检测
```cpp
// C++标准检测
#if __cplusplus >= 202302L
    #define CPP23_OR_LATER
#elif __cplusplus >= 202002L  
    #define CPP20_OR_LATER
#elif __cplusplus >= 201703L
    #define CPP17_OR_LATER
#elif __cplusplus >= 201402L
    #define CPP14_OR_LATER
#elif __cplusplus >= 201103L
    #define CPP11_OR_LATER
#endif
```

### 特性检测
```cpp
// 特性检测宏
#ifdef __cpp_concepts
    // 概念支持
#endif

#ifdef __cpp_modules  
    // 模块支持
#endif

#ifdef __cpp_coroutines
    // 协程支持  
#endif
```

## 性能对比

### 编译速度 (相对比较)
```cpp
编译器        相对速度    内存使用
Clang        ★★★★☆      ★★★☆☆
GCC          ★★★☆☆      ★★☆☆☆  
MSVC         ★★☆☆☆      ★★★★☆
ICC          ★★☆☆☆      ★★★☆☆
```

### 优化性能 (生成代码质量)
```cpp
编译器        优化质量    数学计算    并行优化
GCC          ★★★★☆      ★★★★☆      ★★★☆☆
Clang        ★★★★☆      ★★★★☆      ★★★★☆
MSVC         ★★★☆☆      ★★★☆☆      ★★★☆☆
ICC          ★★★★★      ★★★★★      ★★★★★
```

## 实用建议

### 1. 跨平台开发
```cpp
// 推荐组合
Linux:   GCC 11+ 或 Clang 14+
macOS:   Clang 14+ (Xcode Command Line Tools)
Windows: MSVC 2022 或 GCC 11+ (MinGW-w64)
```

### 2. 持续集成
```yaml
# GitHub Actions 示例
matrix:
  compiler: [gcc-11, clang-14, msvc-2022]
  standard: [17, 20]
```

### 3. 编译器切换
```bash
# 使用update-alternatives管理多版本GCC
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-11 60
sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-11 60

# 环境变量切换
export CC=clang-14
export CXX=clang++-14
```

这份文档提供了C++各版本在不同编译器中的详细支持情况，可以帮助你选择合适的编译器和C++标准版本。

