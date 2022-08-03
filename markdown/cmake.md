<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [CMake 笔记](#cmake-笔记)
  - [lib 文件编译](#lib-文件编译)
  - [调用链接库](#调用链接库)
  - [lib 与 src 同时编译](#lib-与-src-同时编译)

<!-- /code_chunk_output -->



# CMake 笔记

___

## lib 文件编译

lib 文件夹下有 hello.cpp 与 hello.h 两个文件。将其编译为动态与静态链接库。

__主目录下：__

```cmake {.line-numbers}
CMAKE_MINIMUM_REQUIRED(VERSION 3.16)

PROJECT(HELLOLIB)
ADD_SUBDIRECTORY(lib)
```

__lib 目录下：__

```cmake {.line-numbers}
SET(LIBHELLO_SRC hello.cpp)
ADD_LIBRARY(hello SHARED ${LIBHELLO_SRC})  # 动态链接库
SET_TARGET_PROPERTIES(hello PROPERTIES CLEAN_DIRECT_OUTPUT 1)
SET_TARGET_PROPERTIES(hello PROPERTIES VERSION 1.2 SOVERSION 1)  # 指定动态库版本

ADD_LIBRARY(hello_static STATIC ${LIBHELLO_SRC})  # 静态链接库
SET_TARGET_PROPERTIES(hello_static PROPERTIES OUTPUT_NAME hello)  # 指定静态库名称
SET_TARGET_PROPERTIES(hello_static PROPERTIES CLEAN_DIRECT_OUTPUT 1)

# 安装头文件和共享库
INSTALL(TARGETS hello hello_static LIBRARY DESTINATION lib ARCHIVE DESTINATION lib)  # 静态库要指定 ARCHIVE 关键字
INSTALL(FILES hello.h DESTINATION include/hello)
```

__命令行输入：__

```
cmake -DCMAKE_INSTALL_PREFIX=/home/neal/usr  ..
make
mak install
```

__最终运行得到：__

```
/home/neal/usr/lib/libhello.a
/home/neal/usr/lib/libhello.so
/home/neal/usr/lib/libhello.so.1
/home/neal/usr/lib/libhello.so.1.2

/home/neal/usr/include/hello/hello.h
```

## 调用链接库

src 文件夹下有 helloworld.cpp 文件，该文件 # include <hello.h>。

__主目录下：__

```cmake {.line-numbers}
CMAKE_MINIMUM_REQUIRED(VERSION 3.16)

PROJECT(HELLOMAIN)
ADD_SUBDIRECTORY(src bin)  # 将 src 编到 bin 文件夹下。
```

__src 目录下：__

```cmake {.line-numbers}
ADD_EXECUTABLE(hello helloworld.cpp)

FIND_PATH(HELLO_HEADER NAMES hello.h PATHS /home/neal/usr/include/hello)  # 查找头文件。
INCLUDE_DIRECTORIES(${HELLO_HEADER})  # include 头文件。

FIND_LIBRARY(HELLO_LIBRARY hello HINTS /home/neal/usr/lib)  # 查找链接库
TARGET_LINK_LIBRARIES(hello ${HELLO_LIBRARY})  # 添加链接库
```

__最终运行得到：__

```
/home/neal/project/helloworld/build/bin/hello
```

## lib 与 src 同时编译

__主目录下：__

```cmake {.line-numbers}
CMAKE_MINIMUM_REQUIRED(VERSION 3.16)

PROJECT(HELLOLIB)
ADD_SUBDIRECTORY(lib)
ADD_SUBDIRECTORY(src bin)
```

__lib 目录下：__

```cmake {.line-numbers}
SET(LIBHELLO_SRC hello.cpp)
ADD_LIBRARY(hello_lib SHARED ${LIBHELLO_SRC})
SET_TARGET_PROPERTIES(hello_lib PROPERTIES VERSION 1.2 SOVERSION 1)
```

__src 目录下：__

```cmake {.line-numbers}
ADD_EXECUTABLE(hello helloworld.cpp)

FIND_PATH(HELLO_HEADER NAMES hello.h PATHS /home/neal/projects/helloworld/lib)
INCLUDE_DIRECTORIES(${HELLO_HEADER})
TARGET_LINK_LIBRARIES(hello hello_lib)

FIND_PACKAGE(OpenCV REQUIRED)
INCLUDE_DIRECTORIES(${OpenCV_INCLUDE_DIRS})
TARGET_LINK_LIBRARIES(hello ${OpenCV_LIBS})
```
