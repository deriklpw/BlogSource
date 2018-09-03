---
title: Ubuntu 14.04 Android 7.1.2源码编译
---
&emsp;&emsp;一直想入手Framework，今天终于踏上前进的步伐，从编译环境开始，记录点滴，致多年后的自己。源码的下载这里就暂时不写了，后面有需要再补上去，先介绍环境配置，编译和烧录。

<!--more-->

# 一、安装Open JDK8
1. 增加更新源仓库
```python
$ sudo add-apt-repository ppa:openjdk-r/ppa
```
2. 更新
```python
$ sudo apt-get update
```
3. 安装
```python
$ sudo apt-get install openjdk-8-jdk
```
4. 配置环境变量，可以选择不同地方进行配置，~/.bashrc，/etc/profile，或/etc/environment
```python
$ sudo gedit ~/.bashrc
```
	在其最后面添加如下环境变量
```python
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
export ANDROID_SDK=SDK的安装路径
export PATH=${JAVA_HOME}/bin:${ANDROID_SDK}/platform-tools:${ANDROID_SDK}/tools:$PATH
```
5. 更新导入，或者重启终端
```python
$ source ~/.bashrc
```
6. 查看java版本信息
```python
$ java -version
openjdk version "1.8.0_91"
OpenJDK Runtime Environment (build 1.8.0_91-8u91-b14-0ubuntu4~12.04-b14)
OpenJDK 64-Bit Server VM (build 25.91-b14, mixed mode)
```
	显示如上信息，说明配置成功

7. 如果有多个java版本，可以设置默认java版本
例如，再安装了一个java7
```python
$ sudo update-alternatives --install /usr/bin/java java /usr/lib/jvm/java-7-openjdk-amd64/bin/java 1070
$ sudo update-alternatives --install /usr/bin/javac javac /usr/lib/jvm/java-7-openjdk-amd64/bin/javac 1070

$ sudo update-alternatives --install /usr/bin/java java /usr/lib/jvm/java-8-openjdk-amd64/bin/java 600
$ sudo update-alternatives --install /usr/bin/javac javac /usr/lib/jvm/java-8-openjdk-amd64/bin/javac 600
```
	此时，可查看当前各种JDK版本和配置默认java版本：
```python
$ sudo update-alternatives --config java
$ sudo update-alternatives --config javac
```
	到这里，Java OpenJdk安装好了，接下来是安装编译时需要的各种依赖包。

# 二、安装依赖包
+ 搜集的需要安装的依赖包如下:
```python
$ sudo apt-get install git-core gnupg flex bison gperf build-essential zip curl zlib1g-dev libc6-dev lib32ncurses5-dev x11proto-core-dev libx11-dev lib32readline-gplv2-dev lib32z1-dev libgl1-mesa-dev g++-multilib mingw32 tofrodos python-markdown libxml2-utils xsltproc u-boot-tools lib32z1 lib32ncurses5 lib32bz2-1.0 gcc-arm-linux-gnueabi make ncurses-dev gawk python-lunch u-boot-tools
```
&emsp;&emsp;环境基本OK，接下来可以编译了。由于是先配好再来写的这份文档，如果有一两个细节疏忽，网上的前辈们有很多经验帖可以参考，也欢迎在下面评论区留言指正。

# 三、开始编译
1. 先编译内核
不同的厂商不一样，这里只贴全志的，仅供参考
```python
$ cd lichee
$ ./build.sh config

Welcome to mkscript setup progress
All available chips:
   0. sun50iw1p1
   1. sun8iw1p1
   2. sun8iw3p1
   3. sun8iw5p1
   4. sun8iw6p1
   5. sun8iw7p1
   6. sun8iw8p1
   7. sun8iw9p1
   8. sun9iw1p1
Choice: 4
All available platforms:
   0. android
   1. dragonboard
   2. linux
   3. camdroid
   4. secureandroid
Choice: 0
All available kernel:
   0. linux-3.4
Choice: 0
All available boards:
   0. f1
   1. fpga
   2. n1
   3. perf1_v1_0
   4. perf2_v1_0
   5. perf3_v1_0
   6. qc
Choice: 0

$ ./build.sh
```
2. 进入源码根目录，执行源码环境配置文件
```python
$ source build/envsetup.sh
```
3. 执行lunch，选择要编译的板
```python
$ lunch
```
4. 拷贝内核至输出目录
```python
$ extract-bsp
```
5. 开始编译
```python
$ make -j8 //8为线程数，可依据电脑配置设置
```
编译成功之后，生成的文件会在out/target目录下
6. 打包成烧录img
```python
$ pack -d
```

# 四、烧录
烧录的方式有两种：
+ Windows软件烧录
+ fastboot烧录，按区烧录img

第一种烧录的是上面打包好的img，下载烧录软件即可完成。下面介绍的是第二种情况的烧录：
1. 重启进入烧录模式
```python
$ adb reboot bootloader
```
2. 查看设备
```python
$ fastboot devices
```
3. 如果无驱动，增加驱动
```python
$ sudo vim /etc/udev/rules.d/50-android.rules
```
	先lsusb，查看id，在文件中加入ATTR{idVendor}和ATTRS{idProduct}，格式如下
```python
SUBSYSTEM=="usb", ATTR{idVendor}=="0012", MODE="0666", GROUP="plugdev"
SUBSYSTEM=="usb", ATTR{idVendor}=="1b8e", ATTRS{idProduct}=="c003", MODE="0666", GROUP="plugdev"
SUBSYSTEM=="usb", ATTR{idVendor}=="0489", ATTRS{idProduct}=="0007", MODE="0666", GROUP="plugdev"
SUBSYSTEM=="usb", ATTR{idVendor}=="18d1", ATTRS{idProduct}=="4ee7", MODE="0666", GROUP="plugdev"
SUBSYSTEM=="usb", ATTR{idVendor}=="1f3a", ATTRS{idProduct}=="1010", MODE="0666", GROUP="plugdev"
```
4. 重启udev
```python
$ sudo service udev restart
```
5. fastboot devices检测到设备后，开始烧录img
```python
$ fastboot flash boot out/target/product/octopus-f1/boot.img
$ fastboot flash system out/target/product/octopus-f1/system.img
$ fastboot reboot
```

# 五、编译错误处理
1. 内存不足
+ 修改文件jack-admin
```python
$ sudo gedit ./prebuilts/sdk/tools/jack-admin
```
找到参数JACK_SERVER_VM_ARGUMENTS，修改为  
JACK_SERVER_VM_ARGUMENTS="${JACK_SERVER_VM_ARGUMENTS:=-Dfile.encoding=UTF-8 -XX:+TieredCompilation <font color=#ff0000>-Xmx3g</font>}"
如果依然提示内存不足，再改大一点。
+ 重启jack server
```python
$ ./prebuilts/sdk/tools/jack-admin  stop-server
$ ./prebuilts/sdk/tools/jack-admin  start-server
```
&emsp;&emsp;编译烧录上没有很长内容，比较简单，但是在没有自己走一遍之前，免不了有无从下手的感觉，编译烧录成功了，及时记录下相关知识，方便以后查阅，也希望对有相同兴趣爱好的人有所帮助。学无止境，共勉之！

