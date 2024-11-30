---
aliases: 
tags:
  - CMake
  - Vscode
  - GCC
create-datetime: 2024-09-13
id: "20240913162721"
url: https://zhuanlan.zhihu.com/p/621089837
---


[GCC + Vscode 搭建 STM32 开发环境（一）- 环境部署21 赞同 · 6 评论文章](https://zhuanlan.zhihu.com/p/576972892)

[GCC + Vscode 搭建 STM32 开发环境（二）- 使用Cmake管理与构建27 赞同 · 14 评论文章](https://zhuanlan.zhihu.com/p/621089837)

[GCC + Vscode 搭建 STM32 开发环境（三）- 调试6 赞同 · 1 评论文章](https://zhuanlan.zhihu.com/p/690964572)

_Cmake_ 管理工程灵活性很高，且 _Cmake_ 官方文档并没有提供一个完整的模板教用户如何去较好的组织一个项目。 结合工程实践，我整理出了一套自己的使用方法。在我的项目里面，一共有三类 _Cmake_ 文件：

1. 公共的 *_.cmake_，这部分主要提供了编译器及其参数、处理器等信息的描述；
2. 模块的 _CmakeList.txt_，用来描述项目里会引用不同的模块（自己创建的或应用第三方的库）；
3. 工程的 _CmakeList.txt_，该文件指定了具体的编译规则，并最终生成可执行文件；这个文件会引用 _1_、_2_ 两个文件；

这部分的文件后缀是 _cmake_，主要提供在使用 _Cmake_ 管理工程时的共用部分。

![https://pica.zhimg.com/80/v2-4b03ac9070e6e92929fb3936d3100a7e_720w.webp](https://pica.zhimg.com/80/v2-4b03ac9070e6e92929fb3936d3100a7e_720w.webp)

这里面包含了两类文件：编译器说明文件和内核说明文件。

### 1.1 编译器说明

这个文件说明了在编译工程时使用的编译套件以及编译参数，具体可阅读代码的注释。

_代码清单：arm-none-eabi.cmake_

```
# 编译工具链；
# 请确保已经添加到环境变量；
# 如果使用的是 linux 环境，需要将后面的 '.exe' 移除；
SET(CMAKE_C_COMPILER "arm-none-eabi-gcc.exe")
SET(CMAKE_CXX_COMPILER "arm-none-eabi-g++.exe")
SET(AS "arm-none-eabi-as.exe")
SET(AR "arm-none-eabi-ar.exe")
SET(OBJCOPY "arm-none-eabi-objcopy.exe")
SET(OBJDUMP "arm-none-eabi-objdump.exe")
SET(SIZE "arm-none-eabi-size.exe")

# 使用的 C 语言版本；
SET(CMAKE_C_STANDARD 99)
# 使用的 cpp 版本；
SET(CMAKE_CXX_STANDARD 17)
# 生成 compile_commands.json，可配合 clangd 实现精准的代码关联与跳转；
SET(CMAKE_EXPORT_COMPILE_COMMANDS True)
# 彩色日志输出；
SET(CMAKE_COLOR_DIAGNOSTICS True)

# 路径查找；
SET(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
SET(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
SET(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
SET(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)

# this makes the test compiles use static library option so that we don't need to pre-set linker flags and scripts
SET(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)

# 包含gcc头文件路径
SET(SYSTEM_PATH "-isystem C:/~Arm_Development_Toolchains/gcc-arm-none-eabi-10.3-2021.10/arm-none-eabi/include")

# 定义通用编译器参数；
# ${MCPU_FLAGS}   处理器内核信息
# ${VFP_FLAGS}    浮点运算单元类型
# ${SYSTEM_PATH}  编译器头文件路径
SET(CFCOMMON
    "${MCPU_FLAGS} ${VFP_FLAGS} ${SYSTEM_PATH} --specs=nano.specs -specs=rdimon.specs --specs=nosys.specs -Wall -fmessage-length=0 -ffunction-sections -fdata-sections"
)

# 定义最快运行速度发行模式的编译参数；
SET(CMAKE_C_FLAGS_RELEASE "-Os  ${CFCOMMON}")
SET(CMAKE_CXX_FLAGS_RELEASE "-Os  ${CFCOMMON} -fno-exceptions")
SET(CMAKE_ASM_FLAGS_RELEASE "${MCPU_FLAGS} ${VFP_FLAGS} -x assembler-with-cpp")

# 定义最小尺寸且包含调试信息的编译参数；
SET(CMAKE_C_FLAGS_RELWITHDEBINFO "-Os -g  ${CFCOMMON}")
SET(CMAKE_CXX_FLAGS_RELWITHDEBINFO "-Os -g  ${CFCOMMON} -fno-exceptions")
SET(CMAKE_ASM_FLAGS_RELWITHDEBINFO "${MCPU_FLAGS} ${VFP_FLAGS} -x assembler-with-cpp")

# 定义最小尺寸的编译参数；
SET(CMAKE_C_FLAGS_MINSIZEREL "-Os  ${CFCOMMON}")
SET(CMAKE_CXX_FLAGS_MINSIZEREL "-Os  ${CFCOMMON} -fno-exceptions")
SET(CMAKE_ASM_FLAGS_MINSIZEREL "${MCPU_FLAGS} ${VFP_FLAGS} -x assembler-with-cpp")

# 定义调试模式编译参数；
SET(CMAKE_C_FLAGS_DEBUG "-O0 -g  ${CFCOMMON}")
SET(CMAKE_CXX_FLAGS_DEBUG "-O0 -g  ${CFCOMMON} -fno-exceptions")
SET(CMAKE_ASM_FLAGS_DEBUG "${MCPU_FLAGS} ${VFP_FLAGS} -x assembler-with-cpp")

IF("${CMAKE_BUILD_TYPE}" STREQUAL "Release")
  MESSAGE(STATUS "**** Maximum optimization for speed ****")
ELSEIF("${CMAKE_BUILD_TYPE}" STREQUAL "RelWithDebInfo")
  MESSAGE(STATUS "**** Maximum optimization for size, debug info included ****")
ELSEIF("${CMAKE_BUILD_TYPE}" STREQUAL "MinSizeRel")
  MESSAGE(STATUS "**** Maximum optimization for size ****")
ELSE() # "Debug"
  MESSAGE(STATUS "**** No optimization, debug info included ****")
ENDIF()
```

### 1.2 处理器内核说明

该文件描述了当前工程使用的处理器的内核信息，如内核版本、指令集类型、浮点运算单元类型等等。

_代码清单：cortex_m4.cmake_

```
SET(CMAKE_SYSTEM_NAME Generic)
SET(CMAKE_SYSTEM_PROCESSOR cortex-m4)
SET(THREADX_ARCH "cortex_m4")
SET(THREADX_TOOLCHAIN "gnu")
ADD_DEFINITIONS(-DARM_MATH_CM4 -DARM_MATH_MATRIX_CHECK -DARM_MATH_ROUNDING -D__FPU_PRESENT=1)
SET(MCPU_FLAGS "-mthumb -mcpu=cortex-m4")
SET(VFP_FLAGS "-mfloat-abi=soft")
MESSAGE(STATUS "**** Platform: ${MCPU_FLAGS} ${VFP_FLAGS} ****")
INCLUDE(${CMAKE_CURRENT_LIST_DIR}/arm-none-eabi.cmake)
```

_代码清单：cortex_m0.cmake_

```
SET(CMAKE_SYSTEM_NAME Generic)
SET(CMAKE_SYSTEM_PROCESSOR cortex-m0)
SET(THREADX_ARCH "cortex_m0")
SET(THREADX_TOOLCHAIN "gnu")
SET(MCPU_FLAGS "-mcpu=cortex-m0 -mthumb")
SET(VFP_FLAGS "")
MESSAGE(STATUS "**** Platform: ${MCPU_FLAGS} ${VFP_FLAGS} ${FLOAT_ABI} ****")
INCLUDE(${CMAKE_CURRENT_LIST_DIR}/arm-none-eabi.cmake)
```

其他内核的写法依葫芦画瓢就可以了。

值得注意的是 _INCLUDE(${CMAKE_CURRENT_LIST_DIR}/arm-none-eabi.cmake)_ 这句话，这里是引用了第 _1.1_ 小节提到的编译器描述文件。这有点类似 _c_ 语言的 _#include “xxxxx.h”_。

而工程的 _Cmake_ 文件又会引用处理器的描述文件。通过这种层层引用的操作，会把构建的必要参数传递给构建过程。

## 2. 模块的 CmakeList.txt 文件

模块的 _CmakeList.txt_ 用来组织功能模块的编译规则。这里以一个 _LED_ 指示灯的驱动模块为例说明。 模块文件夹内容如图。

![https://pic2.zhimg.com/80/v2-8ac13df3874a013db992ade4b076ce29_720w.webp](https://pic2.zhimg.com/80/v2-8ac13df3874a013db992ade4b076ce29_720w.webp)

_drv_led.c_ 和 _drv_led.h_ 为具体的源代码，这里不展开。_CmakeLists.txt_ 就是该模块的规则描述文件了，文件内容及说明如下。

_代码清单：CmakeLists.txt_

```
# 要连接到构建目标的源文件；
TARGET_SOURCES(
  ${PROJECT_NAME}
  PRIVATE # {{BEGIN_TARGET_SOURCES}}
          ${CMAKE_CURRENT_LIST_DIR}/drv_led.c
          # {{END_TARGET_SOURCES}}
)

# 将模块头文件路径添加到目标；
TARGET_INCLUDE_DIRECTORIES(${PROJECT_NAME} PUBLIC ${CMAKE_CURRENT_LIST_DIR})
```

其中提到的 _构建目标_ 是由工程的 _CmakeLists.txt_ 指定的，在这里我们使用变量替代，在构建过程中会用实际的目标名称替换该变量。 在组织工程的时候，将需要的模块的子目录添加到工程的 _CmakeLists.txt_ 中便可以完成对该模块的调用。这类似于 _Keil_ 或 _IAR_ 中工程右键添加文件或目录，只不过他们在后台帮你完成了构建脚本的修改。

## 3. 工程的 CmakeList.txt 文件

工程的 _CmakeList.txt_ 是整个项目的编译入口，主要： - 指定工程名称（构建目标名称）； - 构建规则声明； - 依赖管理； - 预定义宏； - ....

_代码清单：CmakeList.txt_

```
# ######################################################################################################################
# 0、硬件平台信息与编译器信息；
# ######################################################################################################################

SET(PATH_WORKSPACE_ROOT ${CMAKE_SOURCE_DIR}/../../..)

INCLUDE("${PATH_WORKSPACE_ROOT}/components/toolchain/cmake/cortex_m4f.cmake")

# ######################################################################################################################
# 1、工程信息
# ######################################################################################################################

# 设置CMAKE最低版本
CMAKE_MINIMUM_REQUIRED(VERSION 3.20)

# 设置当前的工程名称
PROJECT(
  demo
  VERSION 0.0.1
  LANGUAGES C CXX ASM)
MESSAGE(STATUS "**** Building project: ${CMAKE_PROJECT_NAME}, Version: ${CMAKE_PROJECT_VERSION} ****")

# 指定链接文件；
SET(LINKER_SCRIPT ${CMAKE_SOURCE_DIR}/stm32f429bit_flash.ld)

# 指定启动文件；
SET(STARTUP_ASM ${PATH_WORKSPACE_ROOT}/components/cmsis/Device/ST/STM32F4xx/Source/Templates/gcc/startup_stm32f429xx.S)

# 项目底层公共头文件；
INCLUDE_DIRECTORIES(${PATH_WORKSPACE_ROOT}/include)

# ######################################################################################################################
# 2、编译控制；
# ######################################################################################################################

# 是否开启更详细的编译过程信息显示
SET(CMAKE_VERBOSE_MAKEFILE OFF)

# ######################################################################################################################
# 3、预定义宏；
# ######################################################################################################################

# 平台相关宏定义
ADD_DEFINITIONS(
  -DUSE_STDPERIPH_DRIVER
  -DSTM32
  -DSTM32F429_439xx
  -DHSE_VALUE=8000000
  -DCMAKE_DEMO
  -DUSING_NON_RTOS=0 # 不使用 RTOS
  -DUSING_FREERTOS=1 # 使用 FreeRTOS
  -DUSING_THREADX=2 # 使用 Threadx
  -DUSING_RTOS=USING_NON_RTOS # 选择使用的 RTOS
)

# ######################################################################################################################
# 4、差异化构建配置；
# ######################################################################################################################

OPTION(OPEN_LOG_OMN_DEBUG "Open log output for debug" OFF)

# 修改该变量的值，可以修改输出文件的名称；
SET(OUTPUT_EXE_NAME "demo")

# 优化级别的差异配置
# ######################################################################################################################
IF("${CMAKE_BUILD_TYPE}" STREQUAL "Release")
  ADD_DEFINITIONS()
ELSEIF("${CMAKE_BUILD_TYPE}" STREQUAL "RelWithDebInfo")
  ADD_DEFINITIONS()
ELSEIF("${CMAKE_BUILD_TYPE}" STREQUAL "MinSizeRel")
  ADD_DEFINITIONS()
ELSE()
  IF(OPEN_LOG_OMN_DEBUG)
    ADD_DEFINITIONS(-DLOG_BACKEND=LOG_BACKEND_NONE)
  ELSE()
    ADD_DEFINITIONS(-DLOG_BACKEND=LOG_BACKEND_NONE)
  ENDIF()
ENDIF()

MESSAGE(STATUS "**** Build for ${CMAKE_BUILD_TYPE} ****")

# ######################################################################################################################
# 5、设置文件输出路径；
# ######################################################################################################################

# 设置库输出路径
SET(LIBRARY_OUTPUT_PATH ${PROJECT_BINARY_DIR}/lib_obj)

SET(ELF_FILE ${PROJECT_BINARY_DIR}/${OUTPUT_EXE_NAME}.elf)
SET(HEX_FILE ${PROJECT_BINARY_DIR}/${OUTPUT_EXE_NAME}.hex)
SET(BIN_FILE ${PROJECT_BINARY_DIR}/${OUTPUT_EXE_NAME}.bin)

# ######################################################################################################################
# 6、组织公共库源文件；
# ######################################################################################################################

# ######################################################################################################################
# 7、组织用户源文件；
# ######################################################################################################################

# 用户源码；
# ######################################################################################################################
INCLUDE_DIRECTORIES(
  # 应用层头文件包含路径；
  ${PATH_WORKSPACE_ROOT}/projects/demo/source/applications ${PATH_WORKSPACE_ROOT}/projects/demo/source/config/board
  # 硬件驱动头文件路径；
  ${PATH_WORKSPACE_ROOT}/common_src/drivers/internal_driver ${PATH_WORKSPACE_ROOT}/common_src/drivers/module_driver)

SET(USER_SOURCE
    ${PATH_WORKSPACE_ROOT}/projects/demo/source/applications/stm32f4xx_it.c
    ${PATH_WORKSPACE_ROOT}/projects/demo/source/config/board/system_stm32f4xx.c
    ${PATH_WORKSPACE_ROOT}/projects/demo/source/main.c)

# ######################################################################################################################
# 8、编译、连接，生成可执行文件
# ######################################################################################################################

# 定义连接器参数； --gc-sections：指示链接器去掉不用的 section
SET(CMAKE_EXE_LINKER_FLAGS
    "${CMAKE_EXE_LINKER_FLAGS} -T ${LINKER_SCRIPT} -Wl,-Map=${PROJECT_BINARY_DIR}/${OUTPUT_EXE_NAME}.map -Wl,--gc-sections,--print-memory-usage"
)

# 生成可执行文件
ADD_EXECUTABLE(${PROJECT_NAME} ${COMMON_SERVICES_SOURCE} ${USER_SOURCE} ${LINKER_SCRIPT} ${STARTUP_ASM})

# 添加依赖；

SET(PATH_COMPONENTS ${PATH_WORKSPACE_ROOT}/components)
ADD_SUBDIRECTORY(${PATH_COMPONENTS}/cmsis/ ${LIBRARY_OUTPUT_PATH}/cmsis)
ADD_SUBDIRECTORY(${PATH_COMPONENTS}/soc_std_driver/stm32f4xx ${LIBRARY_OUTPUT_PATH}/soc_std_driver/stm32f4xx)
ADD_SUBDIRECTORY(${PATH_COMPONENTS}/bsp/stm32/ ${LIBRARY_OUTPUT_PATH}/bsp/stm32/)
ADD_SUBDIRECTORY(${PATH_COMPONENTS}/driver/led ${LIBRARY_OUTPUT_PATH}/led)
ADD_SUBDIRECTORY(${PATH_COMPONENTS}/libraries/fifo/ ${LIBRARY_OUTPUT_PATH}/fifo)
ADD_SUBDIRECTORY(${PATH_COMPONENTS}/libraries/link_list/ ${LIBRARY_OUTPUT_PATH}/link_list)

# ######################################################################################################################
# 9、生成 hex 和 bin 文件
# ######################################################################################################################

ADD_CUSTOM_COMMAND(
  TARGET "${PROJECT_NAME}"
  POST_BUILD
  # Build .hex and .bin files
  COMMAND ${OBJCOPY} -Obinary "${PROJECT_NAME}" "${OUTPUT_EXE_NAME}.bin"
  COMMAND ${OBJCOPY} -Oihex "${PROJECT_NAME}" "${OUTPUT_EXE_NAME}.hex"
  COMMENT "Building ${OUTPUT_EXE_NAME}.bin and ${OUTPUT_EXE_NAME}.hex"
  # Display sizes
  COMMAND ${SIZE} --format=berkeley ${PROJECT_NAME}
  COMMENT "Invoking: Cross ARM GNU Print Size")
```

## 4. vscode 工作空间的说明

我通常习惯在 _VScode_ 的工作空间放置两个目录， 具体看下图：

![image.png|1103](https://prod-files-secure.s3.us-west-2.amazonaws.com/2677eaae-d464-4f07-b1fe-9de715174bce/49cafb34-d22b-4dff-a972-9707216eff79/image.png)

第 _1_ 个目录是工程的目录，这样可以快速的查看工程的信息。

通常我的一个项目里面可能存在多个工程，而这些工程会引用一些公共过的库文件，所以这些文件会放置在项目的公共目录里面，如 _components_ 目录。

那么这个时候，我会把整个项目的目录添加到工作空间，这样我就可以方便的查阅公共模块的代码或其他文件了。你会发现，在上述截图中的第二个目录会包含第一个目录的全部内容。

这只是我的习惯，你可以按照实际情况酢情处理。

## 5. 构建与编译

到此为止，我们可以开始启动构建，并得到最终的二进制文件了。

### 5.1 使用 VScode 任务

展开 _demo/.vscode_ 目录，创建 _task,json_ 文件，这个文件会定义具体的生成与构建任务。

_代码清单：tasks.json_

```
{
   "version": "2.0.0",
   "tasks": [
      { /// 如果你使用的是 J-link 调试，并配置了 RTT 打印，那么开启该任务可以在终端打印 RTT 日志；
         "label": "0. Segger-RTT", 
         "type": "shell",
         "command": "C:/'Program Files (x86)'/SEGGER/JLink/JLinkRTTClient.exe",
         "args": [],
         "problemMatcher": [],
         "group": {
            "kind": "build",
            "isDefault": true
         }
      },
      { /// 执行该任务，你可以生成构建脚本，我这里使用的是 Ninja;
         "label": "1. Reload Cmake Project (Orion675FS)",
         "type": "shell",
         "command": "clear ; Remove-Item -Recurse ./build ; mkdir ./build ; cd ./build ; cmake -G \\"Ninja\\" -DOPEN_LOG_OMN_DEBUG=ON -DCMAKE_BUILD_TYPE=Debug ..",
         "options": {
            "cwd": "${workspaceFolder}/gcc/"
         },
         "group": {
            "kind": "build",
            "isDefault": true
         }
      },
      { /// 执行该任务，你可以执行编译，并得到可用的二进制文件；
         "label": "2. Build (Debug)",
         "type": "shell",
         "command": "clear ; cd ./build ; cmake -G \\"Ninja\\" -DOPEN_LOG_OMN_DEBUG=ON .. ; ninja -j8",
         "options": {
            "cwd": "${workspaceFolder}/gcc/"
         },
         "group": {
            "kind": "build",
            "isDefault": true
         },
      }
   ]
}
```

该文件中具体的每个参数的含义这里不解释，具体的命令的含义在命令行编译的方式里面会有详细的说明。

创建好文件后，在 _VScode_ 中使用快捷键 _ctrl + shift + B_，可以调出上述任务，点击对应的任务就可以执行了。

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/2677eaae-d464-4f07-b1fe-9de715174bce/ca692ccc-9765-4a23-9a8b-059c20916bab/image.png)

当然，你也可以通过菜单栏 _Terminal / Run Task_ 打开相同的界面。

### 5.2 命令行

在 _gcc_ 目录上右键，在弹出的菜单中点击 _Open in Integrated Terminal_，会打开一个终端。 在终端输入命令：

```
mkdir build && cd build
```

创建构建的过程文件以及最终输出文件的存放路径，你可以取其他名称。

当然了，你也可以直接在 _gcc_ 目录启动构建，但是你的目录可能变得乱七八糟。 执行完该命令后，会进入该目录。

在终端输入如下命令，生成构建脚本，这里以 _Ninja_ 为例。

```
cmake -G "Ninja" -DOPEN_LOG_OMN_DEBUG=ON -DCMAKE_BUILD_TYPE=Debug ..
```

命令内容解释：

- _cmake -G "Ninja"_ 生成适用于 _ninja_ 的构建脚本；如果需要其他的，请在终端输入 _cmake -G -help_ 查阅帮助。
- _-DOPEN_LOG_OMN_DEBUG=ON_，传递一个开关宏的值，通常我们可以在 _cmake_ 文件中定义一些开关宏，在生成的时候指定这些宏的值，这可以方便的实现差异化构建。
- _-DCMAKE_BUILD_TYPE=Debug_ 告诉 _cmake_ 在生成构建脚本时的优化类型，可选 _Debug_、_MinSizeRel_、_RelWithDebInfo_、_Release_。

目前，我们已经完成了构建脚本的生成，接下来可以启动构建了。

在终端输入如下命令，执行构建。

```
ninja -j8
```

其中， _-j8_ 是开启多线程编译，后面的数值表示使用的线程数。 编译输出的内容如下。

```
$  ninja -j8 
[51/51] Linking C executable demo
Memory region         Used Size  Region Size  %age Used
          CCMRAM:          0 GB        64 KB      0.00%
             RAM:        9264 B       192 KB      4.71%
           FLASH:        2340 B         2 MB      0.11%
          EXTRAM:          0 GB         1 MB      0.00%
   text    data     bss     dec     hex filename
   2316      24    9248   11588    2d44 demo
```

_build_ 目录的内容如图。

![image.png|1103](https://prod-files-secure.s3.us-west-2.amazonaws.com/2677eaae-d464-4f07-b1fe-9de715174bce/d5cfa944-a0d5-48cb-8aa8-298e0d6d3a04/image.png)

## 6 总结

- _Cmake_ 和 _tasks.json_ 可以有很灵活写法，文章只是写了我常用的形式，希望你可以在理解的基础上总结出适合自己的方式和方法；
    
- 上述方法对 _clion_ 同样适用；
    
- 上述 _Demo_ 的工程源码，你可以通过下面的命令获取。
    
    ```jsx
    git clone https://jihulab.com/liuy928/cmake_arm_gcc_demo.git
    ```