
cmake_minimum_required(VERSION 3.10.1)
#set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)
#将当前目录附加到CMAKE_MODULE_PATH，以使特定于设备的cmake模块可见
list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_LIST_DIR})
# Target definition
set(CMAKE_SYSTEM_NAME  Generic)
set(CMAKE_SYSTEM_PROCESSOR cortex-a9)
#---------------------------------------------------------------------------------------
# Set toolchain paths
#---------------------------------------------------------------------------------------
set(TOOLCHAIN arm-none-eabi)
if(NOT DEFINED TOOLCHAIN_PREFIX)
    if(CMAKE_HOST_SYSTEM_NAME STREQUAL Linux)
        set(TOOLCHAIN_PREFIX "/home/ares/gcc-arm-none-eabi-9-2019-q4-major")
    elseif(CMAKE_HOST_SYSTEM_NAME STREQUAL Darwin)
        set(TOOLCHAIN_PREFIX "/usr/local")
    elseif(CMAKE_HOST_SYSTEM_NAME STREQUAL Windows)
        set(TOOLCHAIN_PREFIX "C:/Program Files/PROS/toolchain/usr")
    else()
        set(TOOLCHAIN_PREFIX "/usr")
        message(STATUS "No TOOLCHAIN_PREFIX specified, using default: " ${TOOLCHAIN_PREFIX})
    endif()
endif()
#设置工具链系统路径
set(TOOLCHAIN_BIN_DIR ${TOOLCHAIN_PREFIX}/bin)
set(TOOLCHAIN_INC_DIR ${TOOLCHAIN_PREFIX}/${TOOLCHAIN}/include)
set(TOOLCHAIN_LIB_DIR ${TOOLCHAIN_PREFIX}/${TOOLCHAIN}/lib)
# 设置系统依赖的扩展
if(WIN32)
    set(TOOLCHAIN_EXT ".exe" )
else()
    set(TOOLCHAIN_EXT "" )
endif()
# 使用静态库执行编译器测试
set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)
#---------------------------------------------------------------------------------------
# 设置系统编译器
#---------------------------------------------------------------------------------------
set(CMAKE_C_COMPILER ${TOOLCHAIN_BIN_DIR}/${TOOLCHAIN}-gcc${TOOLCHAIN_EXT} CACHE INTERNAL "C Compiler")
set(CMAKE_CXX_COMPILER ${TOOLCHAIN_BIN_DIR}/${TOOLCHAIN}-g++${TOOLCHAIN_EXT} CACHE INTERNAL "C++ Compiler")
set(CMAKE_ASM_COMPILER ${TOOLCHAIN_BIN_DIR}/${TOOLCHAIN}-gcc${TOOLCHAIN_EXT} CACHE INTERNAL "ASM Compiler")
set(CMAKE_OBJCOPY  ${TOOLCHAIN_BIN_DIR}/${TOOLCHAIN}-objcopy${TOOLCHAIN_EXT} CACHE INTERNAL "objcopy")
set(CMAKE_SIZE  ${TOOLCHAIN_BIN_DIR}/${TOOLCHAIN}-size${TOOLCHAIN_EXT} CACHE INTERNAL "size")

set(CMAKE_FIND_ROOT_PATH ${TOOLCHAIN_PREFIX}/${${TOOLCHAIN}} ${CMAKE_PREFIX_PATH})
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)


add_definitions(
    -D_POSIX_THREADS 
    -D_UNIX98_THREAD_MUTEX_ATTRIBUTES
)
#不知道是干嘛
set(LINKER_SCRIPT "${CMAKE_SOURCE_DIR}/firmware/v5.ld")
#设置CPU参数
SET(MCU_FLAGS "-mcpu=cortex-a9 -mfpu=neon-fp16 -mfloat-abi=softfp -Os -g -Wno-psabi -ffunction-sections -fdata-sections -fdiagnostics-color -funwind-tables -Wl,--allow-multiple-definition")
SET(CMAKE_C_FLAGS_INIT "${MCU_FLAGS} --std=gnu11")
SET(CMAKE_CXX_FLAGS_INIT "${MCU_FLAGS} --std=gnu++17")
#设置LINK的FALGS
SET(CMAKE_EXE_LINKER_FLAGS_INIT "-Wl,-gc-sections,--allow-multiple-definition -Wl,-T${LINKER_SCRIPT}")

project(monolith)
ENABLE_LANGUAGE(ASM)
#头文件路径
include_directories(
    ${TOOLCHAIN_INC_DIR} 
    "./include"
)
#设置项目本身的路径你
SET(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/bin) #设置二进制文件存放路径 
SET(LIBRARY_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/firmware) #设置各种库存放路径
#添加需要链接的库文件目录 新建好了 链接的时候用
LINK_DIRECTORIES(${LIBRARY_OUTPUT_PATH})
#aux_source_directory(${PROJECT_SOURCE_DIR}/src SOURCES)
#枚举头文件
file(GLOB_RECURSE INCLUDES "include/*.h" "include/*.hpp")
#枚举源文件
file(GLOB_RECURSE SOURCES "src/*.c" "src/*.cpp")

#制作.ELF
add_executable(${PROJECT_NAME}.elf ${SOURCES} ${INCLUDES})


#链接静态库
set_target_properties(${PROJECT_NAME}.elf PROPERTIES LINK_DEPENDS ${LINKER_SCRIPT})
target_link_libraries(${PROJECT_NAME}.elf c m pros)
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -lc -lm -lpros -lgcc -lstdc++ -lsupc++")
#设置路径
set(ELF_FILE ${EXECUTABLE_OUTPUT_PATH}/${PROJECT_NAME}.elf)
set(HEX_FILE ${EXECUTABLE_OUTPUT_PATH}/${PROJECT_NAME}.hex)
set(BIN_FILE ${EXECUTABLE_OUTPUT_PATH}/${PROJECT_NAME}.bin)
#生成.HEX .BIN怎么写不知道 目前只会生成.ELF 
add_custom_command(TARGET ${PROJECT_NAME}.elf POST_BUILD
        COMMAND ${CMAKE_OBJCOPY} -Oihex $<TARGET_FILE:${PROJECT_NAME}.elf> ${HEX_FILE}
        COMMAND ${CMAKE_OBJCOPY} -Obinary $<TARGET_FILE:${PROJECT_NAME}.elf> ${BIN_FILE}
        COMMENT "Building ${HEX_FILE}
Building ${BIN_FILE}")

add_custom_target(UPLOAD
        arm-none-eabi-gdb -iex "target remote tcp:127.0.0.1:3333"
        -iex "monitor program $<TARGET_FILE:${PROJECT_NAME}.elf>"
        -iex "monitor reset init"
        -iex "disconnect" -iex "quit")