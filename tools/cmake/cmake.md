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


