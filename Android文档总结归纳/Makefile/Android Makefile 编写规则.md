#Android Makefile 编写规则
```
LOCAL_PATH := $(call my-dir) //得到本地目录，即该Android.mk文件所处的目录
LOCAL_DIR_PATH:= $(call my-dir)

include $(CLEAR_VARS) //编译的开始
LOCAL_CFLAGS := -DXXX  //表示在所有的源文件中增加一个宏定义 #define XXX

LOCAL_C_INCLUDES := xxx.h  //表示包含的头文件

LOCAL_SRC_FILES += xxx.cpp xxx.c //表示源文件路径
LOCAL_SHARED_LIBRARIES := libdl libcutils liblog  //表示依赖的共享库(动态库)
LOCAL_STATIC_LIBRARIES := libxxx //表示该模块需要使用哪些静态库，以便在编译时进行链接。

LOCAL_MODULE := libxxx //表示该 makefile文件生成的库文件
LOCAL_MODULE_TAGS := optional 
//eng 表示eng版本会编译
//user 表示user版本下编译
//tests 表示tests版本下会编译
//optional 表示所有版本都会编译

include $(BUILD_SHARED_LIBRARY) //编译的结束
//BUILD_SHARED_LIBRARY 生成共享库(.so)
//BUILD_EXECUTABLE 生成可执行文件
//BUILD_STATIC_JAVA_LIBRARY 生成jar包
//BUILD_STATIC_LIBRARY 生成静态库(.a)
```
