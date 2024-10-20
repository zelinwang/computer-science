**CMake**是一个跨平台的安装编译工具，可义用简单的语句描述所有平台的安装

**CMake**语法格式

- 参数使用**括弧**
- 参数之间使用**空格**或者**分号**

指令是大小写无关的，参数和变量是区分大小写的

```
set(SERVER server.cpp)
add_executable(server main.cpp server.cpp)
ADD_EXECUTABLE(server main.cpp ${SERVER})

if(SERVER) { //*** }
```

变量使用${}取值，但是在if控制语句中是直接使用变量名



重要指令

- cmake_minimum_required - 指定cmake的最小版本要求

  语法：cmake_minmum_required(VERSION sersionNumer [FATAL_ERROR])

  ```cmake
  #CMake的最小版本是2.8.3
  cmake_minimum_required(VERSION 2.8.3)
  ```

- project- 定义工程名称，并指定工程支持语言

  语法：cmake_minmum_required(VERSION sersionNumer [FATAL_ERROR])

  ```cmake
  #指定工程名
  cmake_minimum_required(VERSION 2.8.3)
  ```

  

- set









