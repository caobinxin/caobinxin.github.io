
# adb
## adb 命令小结
* option
```shell
adb -d #将adb命令发给唯一的USB连接设备
adb -e #将adb命令发给唯一运行的模拟器
adb -s <serialNumber> #将adb命令发给指定的模拟器/设备。
```
* general
```shell
adb devices #将显示当前所有连接的模拟器/设备
adb connect <host>[:<port>] #通过tcp/ip连接目标设备 host 目标IP ， port 默认采用5555端口
adb disconnect <host>[:<port>] #断开与目标设备的TCP/IP 连接
adb help #显示adb的帮助信息
adb version #显示adb的版本信息
```
* debug
```shell
adb logcat <option> <filter-specs> #将log信息打印大屏幕上
adb bugreport #将 dumpsys, dumpstate 和 log信息打印到屏幕上
adb jdwp #显示可用的 jdwp 进程列表
```
* data
```shell
adb install <path-to-apk> #将指定路径下的应用程序安装到模拟器/设备中
adb pull <remote><local> #将模拟器或者设备上的指定文件复制到开发环境机器上
adb push <local><remote> #将特定文件复制到模拟器上
```
* scripting
```shell
adb get-serialno #打印出模拟器/设备的 serialno
adb get-state #打印出模拟器/设备的状态
adb wait-for-device #当设备不处于在线状态时，这个选项会暂停命令的执行，直到设备可用
```

adb wait-for-device <command> 对于那些需要在设备完全启动的状态下才能成功执行的adb 命令，这个选项可能会不起作用。比如我们之前提到的安装应用程序的命令，如果系统在还没有启动完成前adb命令就开始执行，那么安装任务很可能会失败。
* server
```shell
adb start-server #用来检查adb server 是否正在运行，如果没有，将会其启动
adb kill-server #终止正在运行的 adb server
```
* shell
```shell
adb shell #启动一个shell环境
adb shell <shellCommand> #命令会直接执行用户指定的shell指令，结束后便会退出shell环境
```
* 其余有用的adb
```shell
adb shell dumpsys activity top | head -n 10 #查询当前展示的activity
adb shell pm list packages -3 #找出设备上安装的第三方apk包名
adb shell pm path $pkg #依据apk包名找出apk
```

## wifi adb
1. 首先要用有线的方式，告诉目标设备**3399板子**在哪个端口进行监听，后续的连接命令要和这里的端口设置保持一致才能成功
```shell
adb tcpip <port> #默认是 5555
```
2. 此时拔出USB线
3. 通过wifi进行连接
```shell
adb connect <hostIp>[:<port>] #hostIP为路由器分配给目标板子的IP port 就是刚刚指定的端口号
```

通过以上三步，电脑中的 ADBserver  和 3399板子的ADBD 就实现了 tcp/ip的连接。

## adb的组成

adb 和传统的C/S 模型还不一样，在adb的框架中包含3个重要的组成部分。
* ADB client （pc）
* ADB Server (pc)
* ADB Deamon(3399板子)

### adb client
   * 运行环境： 在电脑上跑的
   * 角色： 作为发送adb 命令的一端
### adb server
   * 运行环境： 在电脑上跑的
   * 角色
        * 管理pc 和设备**3399**/模拟器的连接状态 -- 两者是 多对多的关系，所以需要一个管理者
        * 处理 adb client 的连接请求
### adb deamon
   * 运行环境： 在3399板子
   * 角色： 将和Adb server 进行通信连接以完成一些列ADB的命令请求

adb client --(多)----- adb server---(多)---- adb deamon

当用户需要发送ADB命令时，将首先启动一个adb client 。随后adb会检查 server 程序是否已经在运行，否则他还会启动server。ADB server，将绑定在TCP的5037端口上，并开始等待客户端程序的连接。
另外，adb server，也会主动扫描有效的模拟器/设备。扫描所用的端口号范围是 5555 ~ 5585的奇数号 -- 这里使用奇数号，是**因为每个模拟器/设备 都需要用到两个端口号，分别用于console 和 adb 连接**。**_这样做有效保证多个目标设备的同时运行和管理。_**

# adb USB
## Linux内核识别设备
```shell
[colbycao@colbycao-CW65S rk3399_firefly_box]$>udevadm monitor --kernel
monitor will print the received events for:
KERNEL - the kernel uevent

KERNEL[22570.552545] remove   /devices/pci0000:00/0000:00:14.0/usb3/3-2/3-2:1.0 (usb)
KERNEL[22570.552598] remove   /devices/pci0000:00/0000:00:14.0/usb3/3-2/3-2:1.1 (usb)
KERNEL[22570.552911] remove   /devices/pci0000:00/0000:00:14.0/usb3/3-2 (usb)

KERNEL[22592.706543] add      /devices/pci0000:00/0000:00:14.0/usb3/3-2 (usb)
KERNEL[22592.707323] add      /devices/pci0000:00/0000:00:14.0/usb3/3-2/3-2:1.0 (usb)
KERNEL[22592.707564] add      /devices/pci0000:00/0000:00:14.0/usb3/3-2/3-2:1.1 (usb)

```

## 查询device的信息
```shell
udevadm info -q all -p
```
