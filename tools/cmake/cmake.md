[[toc]]

# 第一章

## cmake特性
cmake是构建，测试，软件打包的工具

- 优势
  - 避免硬路径
  - 跨平台编译
  - 持续继承
  - 交叉编译
  - IDE可以直接利用cmake生成项目
  - 容易结合其他工具，如git，qt
  - 单元测试
- 特性
  - 自动搜索可能需要的程序，库，头文件的能力
  - 独立构建目录，不会污染源码
  - 创建复杂的自定义命令
  - 配置时选择组件的能力
  - 简单的文本文件CmakeList.txt
  - 静态库和动态库的轻松切换的能力
  - 每个IDE都支持cmake

## 原理

          ┌───────────┐
          │ CMakeCache│
          └───────────┘                             
               ▲
               │
               │ 2. 生成临时文件                    
               │
               │
               │                                    
               │         3.生成项目文件
          ┌────┴──┐                                 ┌────────────┐            ┌──────┐          ┌───────────┐        ┌──────────────┐                                    
          │ cmake ├──────────────────────┬─────────►│ build.ninja├───────────►│Ninja │────────► │ gcc/clang ├───────►│ 输出目标程序 │                                    
          └────┬──┘                      │          └────────────┘            └──────┘          └───────────┘        └──────────────┘                                    
               │                         │          ┌──────────┐              ┌───────┐                                                                                   
               │ 1.读取文件              └─────────►│ Makefile ├───────┬─────►│ make  │                                                                                   
               ▼                                    └──────────┘       │      └───────┘                                                                                   
       ┌───────────────┐                                               │      ┌──────────┐      ┌───────────┐       ┌─────────────────────┐                             
       │ CMakeLists.txt │                                               └─────►│ nmake.exe│─────►│ vs cl.exe ├──────►│ 输出windows目标程序 │                            
       └───────────────┘                                                      └──────────┘      └───────────┘       └─────────────────────┘


cmake读取CMakeList.txt生成CMakeCache，并根据执行cmake的命令和环境生成对应平台的make文件，

执行make时，会根据make文件和CMakeCache利用相关编译工具进行程序构建，

## 基础入门

使用静态库或动态库的示例
```bash
.
├── CMakeLists.txt
├── test_xlog
│   └── main.cc
└── xlog
    ├── xlog.cpp
    └── xlog.h
```

```bash
cmake_minimum_required(VERSION 3.20)
# 给project内置变量赋值
project(demo)
# 可执行程序对象，和他的依赖
add_executable(a.out ./test_xlog/main.cc)
# 静态库和他的依赖
# add_library(xlog STATIC ./xlog/xlog.cc ./xlog/xlog.h)
# 动态库和他的依赖
add_library(xlog SHARED ./xlog/xlog.cc ./xlog/xlog.h)
# 头文件搜索路径
include_directories(./xlog/)
# 库文件搜索路径
link_libraries(./xlog/build/)
# 使用的链接库，必须在 add_executable add_library 后
target_link_libraries(a.out xlog)
```

```bash
# 生成make文件
cmake -S . -B build-linux
# 执行构建
cmake --build ./build-linux
```

# 常用功能
## 注释
- 行注释
  - `# 第一行注释`
- 块注释
  - `#[[第一行注释 第二行注释]]`

## message
基础使用
```bash
message(参数1 参数2)
```
cmake命令的日志级别参数
，默认日志级别为 STATUS
`--log-level=<ERROR|WARNING|NOTICE|STATUS|VERBOSE|DEBUG|TRACE>`

CMakeLists.txt可用的日志参数
- FATAL_ERROR cmake停止运行，输出到stderr
- SEND_ERROR cmake继续运行，输出到stderr
- WARNING 输出到stderr
- NOTICE 输出到stderr
- STATUS 针对用户的简单信息 stdout
- VERBOSE 针对用户的相信信息 stdout
- DEBUG 针对开发人员简单信息 stdout
- TRACE 针对开发人员的详细信息 stdout

示例
```bash
# 会打印文件和行号
message(FATAL_ERROR "严重错误，cmake退出")
message("cmake不会执行到这里")
```
```bash
# 会打印文件和行号
# 进程不会退出，但不会生成项目文件，只是为了继续打印更多错误信息，方便调试
message(SEND_ERROR "严重错误，cmake继续执行")
message("cmake会执行到这里")
```
```bash
# 会打印文件和行号
message(WARNING "WARNING")
# 不会打印文件和行号
message(NOTICE "NOTICE")
message("NOTICE")
# 默认打印 打印加 -- 前缀 用户感兴趣
message(STATUS "STATUS")
# 默认不打印 -- 前缀 用户感兴趣的详细信息
message(VERBOSE "VERBOSE")
```
## 查找库日志

```bash
# 开始查找
message(CHECK_START "查找lib1")
# 设置缩进
set(CMAKE_MESSAGE_INDENT "--")
message(CHECK_START "查找lib2")
message(CHECK_PASS "成功")
# 取消缩进
set(CMAKE_MESSAGE_INDENT "")
message(CHECK_FAIL "失败")
```

## 变量
- `set(<var> <val>)` 创建变量var，并设置值val
  - 如果没有指定val，则变量会被撤销，而不是被设置为空
- `unset(<var>)` 撤销变量
- `${var}` 读取变量，若变量不存在，则返回空字符串
  - 变量名大小写不敏感
  - 变量可以存储变量名，以实现嵌套使用

## 内部变量
- 提供信息的变量
  - PROJECT_NAME project()设置的项目名称
- 控制make行为的变量
  - BUILD_SHARED_LIBS 控制 add_library 默认生成动态库还是静态库
    - ON 创建动态库
    - OFF 创建静态库
- 描述系统的变量
  - WIN
  - UNIX
- 控制生成make文件行为的变量
  - CMAKE_COLOR_MAKEFILE 控制生成makefile文件时，过程输出是否带颜色

## include

```bash
include(file [OPTIONAL] [RESUTL_VARIABLE var])
```
- OPTIONAL : 如果加OPTIONAL，则file是可选的，若file不存在则忽略
- RESUTL_VARIABLE var : 如果file不存在，则将 NOT FOUND写入var，否则将file的绝对路径写入var

```bash
.
├── CMakeLists.txt
└── common
    └── test.cmake
```

```bash
# common/test.cmake
message("test cmake")
```

```bash
# CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(cmake_include)
message("begin include")
include(./common/test.cmake OPTIONAL RESULT_VARIABLE ret)
# ret : /root/cmake/src/107-include/common/test.cmake
message("ret : ${ret}")
message("after include")
# 报错并终止进程
# include(./common/test1.cmake)
# 不会报错
include(./common/test1.cmake OPTIONAL)
include(./common/test1.cmake OPTIONAL RESULT_VARIABLE ret)
# ret : NOTFOUND
message("ret : ${ret}")
message("end include")
```

## 自动加载源文件和头文件
将src下所有源码（不包括头文件）存入LIB_SRCS变量
aux_source_directory("./src" LIB_SRCS)

头文件通过查找的方式获得路径，存入H_FILE变量
FILE(GLOB H_FILE "${INCLUDE_PATH}/xxx/*.h")
FILE(GLOB H_FILE "${INCLUDE_PATH}/*.h")

```bash
cmake_minimum_required(VERSION 3.20)
project(test)
aux_source_directory("./xlog/" LIB_SRCS)
FILE(GLOB LIB_H "./xlog/*.h")
add_library(xlog SHARED ${LIB_SRCS} ${LIB_H})
include_directories(./xlog)
link_directories(./xlog/build/)
aux_source_directory("./test_xlog/" PRJ_SRCS)
FILE(GLOB PRJ_H "./test_xlog/*.h")
add_executable(a.out ${PRJ_SRCS} ${PRJ_H})
target_link_libraries(a.out xlog)
```

## 分步构建

```bash
# 构建make
cmake -S . -B build
# 查看可选的分步构建
cmake --build ./build --target help
The following are some of the valid targets for this Makefile:
... all (the default if no target is provided)
... clean
... depend
... edit_cache
... rebuild_cache
... a.out
... main.o
... main.i
... main.s

# 预编译
cmake --build ./build --target main.i
# 编译
cmake --build ./build --target main.s
# 汇编
cmake --build ./build --target main.o
# 链接
cmake --build ./build
```
## 调试
make时显示具体的构建过程

- 方法1 ： 开启缓存变量 CMAKE_VERBOSE_MAKEFILE
```bash
cmake_minimum_required(VERSION 3.20)
project(firstdemo)
add_executable(a.out main.cpp)

# 开启调试
set(CMAKE_VERBOSE_MAKEFILE ON)
```
- 方法2 : `cmake --build . -v`

## 设置输出目录

- CMAKE_CURRENT_LIST_DIR
  - CMakeLists.txt的目录
- CMAKE_LIBRARY_OUTPUT_DIRECTORY
  - linux的动态库
- CMAKE_RUNTIME_OUTPUT_DIRECTORY
  - 执行程序输出路径
- CMAKE_ARCHIVE_OUTPUT_DIRECTORY
  - 归档路径windows的.lib dll pdb调试文件, linux的.a静态库



