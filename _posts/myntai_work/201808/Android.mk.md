# Android.mk

## 基础
* android.mk基本模板：
```shell
LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)                                                                                                                                                                          
LOCAL_SRC_FILES := $(clang_frontend_SRC_FILES)
LOCAL_MODULE:= 
LOCAL_MODULE_TAGS:= optional
include $(BUILD_xxx)
```
* LOCAL_xxx变量的定义处
FireNow-Nougat/build/core/clear_vars.mk
* BUILD_xxx 的定义处
FireNow-Nougat/build/core/config.mk
* LOCAL_MODULE_TAGS:=
 LOCAL_MODULE_TAGS ：=user eng tests optional
  * user: 指该模块只在user版本下才编译
  * eng: 指该模块只在eng版本下才编译
  * tests: 指该模块只在tests版本下才编译
  * optional:指该模块在所有版本下都编译
* 引入系统库
```shell
LOCAL_SHARED_LIBRARIES += liblog
```
**LOCAL_SHARED_LIBRARIES += liblog**** 经过测试，我们发现我们3399链接的是这个目录下的 liblog.so
```shell
/home/colbycao/cao_bin_xin/FireNow-Nougat/out/target/product/rk3399_firefly_box/obj/lib/liblog.so
```
* 引入第三方的库
通过在3399 的测试发现，会将编译后的第三方库，放置于系统目录下out/target/product/rk3399_firefly_box/obj_arm/lib/libtest.so
既然将编译好的第三方库放到系统目录下了，所以这里在Android.mk中引用的方式和引用系统库时一样的方式。
```shell
LOCAL_SHARED_LIBRARIES += liblog libtest

cp 系统库路径/libtest.so  ./lib/
LOCAL_LDFLAGS:= -L./lib/ -ltest
LOCAL_LDFLAGS:= -L$(LOCAL_PATH)/lib/ -ltest
这三种写发都是没有问题的
```
* 引入第三方的头文件
```shell
LOCAL_C_INCLUDES:= $(LOCAL_PATH)/inc
```
* 引入第三方的库
```shell

```



## 编译二进制
## 编译静态库
## 编译动态库
## 编辑jar包
## 编译app