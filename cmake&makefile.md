# cmake笔记

## 一些网址

3.CMake官方教程— CMake 3.17.0-rc1文档
https://cmake.org/cmake/help/latest/guide/tutorial/index.html

5.cmake使用示例与整理总结
https://blog.csdn.net/QTVLC/article/details/82380413

6.CMake命令/函数汇总（翻译自官方手册）
https://www.cnblogs.com/52php/p/5684588.html

## 大致结构

项目目录下应包含src、incl、bin、lib、build

- src：源码
- incl：头文件（暂时不用）
- bin：执行码
- lib：静态/动态库
- build：cmake产生的文件

### 示例

- 主目录下的CMakeList.txt

```cmake
CMAKE_MINIMUM_REQUIRED(VERSION 3.10)#版本控制

PROJECT(demo3)#项目名称

ADD_SUBDIRECTORY(./mylib)#添加子目录

ADD_SUBDIRECTORY(./src)#添加子目录
```

- ./mylib下的CMakeList.txt

```cmake
AUX_SOURCE_DIRECTORY(. DIR_LIB_SRCS)
#把一个目录下所有的源代码文件并将列表存储在一个变量中(不含头文件)

SET(LIBRARY_OUTPUT_PATH ${PROJECT_BINARY_DIR}/lib)
#LIBRARY_OUTPUT_PATH:重新定义目标链接库文件的存放位置
#PROJECT_SOURCE_DIR:工程的根目录
#PROJECT_BINARY_DIR:运行cmake命令的目录,通常是${PROJECT_SOURCE_DIR}/build

ADD_LIBRARY(Mylib STATIC ${DIR_LIB_SRCS})
#生成动态库或静态库
#SHARED 动态库
#STATIC 静态库
```

- ./src下的CMakeList.txt

```cmake
INCLUDE_DIRECTORIES(${PROJECT_SOURCE_DIR}/mylib)
#向工程添加特定的头文件搜索路径

SET(EXECUTABLE_OUTPUT_PATH ${PROJECT_BINARY_DIR}/bin)
#EXECUTABLE_OUTPUT_PATH:重新定义可执行文件的存放位置

AUX_SOURCE_DIRECTORY(./ DIR_SRCS)

ADD_EXECUTABLE(demo4 ${DIR_SRCS})
#生成可执行文件
TARGET_LINK_LIBRARIES(demo4 Mylib)
#为target添加需要链接的共享库

```

