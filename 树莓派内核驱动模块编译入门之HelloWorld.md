# 环境
树莓派型号：3代B型
树莓派系统：RASPBIAN JESSIE WITH DESKTOP
```
	desc:Image with desktop based on Debian Jessie
	Version:July 2017
	Release date:2017-07-05
	Kernel version:4.9
```
主机系统：ubuntu 16.04（win10 下虚拟机）

# 内核的编译
> 由于树莓派本身资源有限，编译速度较慢，因此在主机系统中使用交叉编译的方式进行内核及内核模块的编译，可大大节省编译时间。

### 准备工作（视情况可选） 
1. 升级树莓派本身的内核版本，在树莓派终端中输入命令如下：
```
	sudo apt-get update
	sudo apt-get upgrade
	sudo rpi-update
	sudo reboot
```
> rpi-update 是树莓派系统自带的一个命令，用来同步更新内核到官方最新的内核版本。该编译好的内核版本其实是放在[github](https://github.com/raspberrypi/firmware)上的，对应于其官方给出的[最新内核源码](https://github.com/raspberrypi/linux)。这一步是后续我们自己编译内核驱动模块的基础。

### 交叉编译环境的搭建
1. 创建放置内核源码和工具的目录
```
	mkdir ~/Rpi
```
cd到该目录下进行接下来的操作。  
2. 下载内核源码。有两种方式，一是使用git clone命令：
```
	git clone git://github.com/raspberrypi/linux.git RpiLinux
```
git clone的方式会下载所有的内核分支，因此需要下载的文件会比较大，约1.5G。另一种方式是使用zip方式下载，然后使用unzip命令解压即可。这种方式只会下载当前分支代码（默认最新版本分支），因此速度会比较快。  
3. 下载编译工具链。同样有两种方式，git clone命令：
```
	git clone https://github.com/raspberrypi/tools.git RpiTools
```
同样也可以zip直接下载然后解压。  
4. 下载Module.symvers文件。命令为：
```
	wget https://github.com/raspberrypi/firmware/raw/master/extra/Module7.symvers
```
在github上树莓派官方项目[firmware](https://github.com/raspberrypi/firmware)中，/extra目录有Module.symvers和Module.symvers两个名字类似的文件，**树莓派3代需要用到的是Module7.symvers**（1代用的好像是Module.symvers，二代我也不确定，大概是这样），如果文件下载错了，等到加载编译好的内核驱动时就会报错："Invalid Module format"。下载完成后将Module7.symvers重命名为Module.symvers。
5. 到此为止，在~/Rpi/目录下就有了两个文件夹和一个文件，即RpiLinux、RpiTools和Module.symvers。  
6. 获取.config文件。在树莓派中cd到/proc目录下，找找看是否有config.gz文件。因为最新的树莓派系统为安全起见，默认是没有的。因此需要我们手动加载相关模块：
```
	sudo modprobe configs
```
之后我们就能看到config.gz文件了。使用命令将该文件内容拷贝出来：
```
	sudo zcat config.gz > ~/config
```
当然，上述命令需要我们在用户目录下提前建立一个config空文件。获取到以后通过scp命令或者其他方式拷贝到主机~/Rpi/目录下，并重命名为.config。  
7. 在RpiLinux目录下进行清理，以便重新编译：
```
	make mrproper
```
这一步主要是防止多次编译后残留一些老的配置、编译文件。如果第一次编译不清理也可以。  
8. 将之前的.config文件移动到RpiLinux目录下。  
9. 指定平台和交叉编译工具链：用Vim打开在RpiLinux目录下Makefile文件，找到并修改如下：
```
	ARCH ?= arm
	CROSS_COMPILE ?= ../RpiTools/arm-bcm2708/arm-bcm2708hardfp-linux-gnueabi/bin/arm-bcm2708hardfp-linux-gnueabi-
```
然后保存退出。  
> - 树莓派官方为我们准备了三套编译工具链，分别是arm-bcm2708hardfp-linux-gnueabi、arm-bcm2708-linux-gnueabi和gcc-linaro-arm-linux-gnueabihf-raspbian。此处我们选择的是待hardfp硬解码的编译工具链；
> - 关于Makefile文件中"=" "?=" "：=" "+="的区别：  
	= 是最基本的赋值  
	:= 是覆盖之前的值  
	?= 是如果没有被赋值过则赋值  
	+= 是添加等号后面的值
10. 准备编译，在RpiLinux目录下执行：
```
	make modules_prepare
```
11. 将之前下载好的Module.symvers移动到RpiLinux目录下。
12. 至此，整个较差编译环境就搭建好了。如果需要编译内核，可以make config进行相关配置，也可以不配置，直接make命令开始编译。同样-j参数可以指定多线程以加快编译。
13. 当然，如果想在树莓派系统中使用我们刚编译好的内核，还需要进行少量其他的工作，这里因为不是本次重点，所以就不说了，可参考其他文档或在网上查阅相关资料。

# 内核驱动之Hello world
* 在~/Rpi/目录下新建LearningDrivers目录,用于以后的驱动模块编写练习。然后新建hello子目录，创建hello.c源文件，代码如下：

```
	#include <linux/init.h>
	#include <linux/module.h>

	MODULE_LICENSE("Dual BSD/GPL");

	static int __init hello_init(void)
	{
    		printk(KERN_ALERT "Hello world\n");
    		return 0;
	}

	static void __exit hello_exit(void)
	{
    		printk(KERN_ALERT "GoodBye! Hello world\n");
	}

	module_init(hello_init);
	module_exit(hello_exit);

```
* 创建Makefile文件，如下：
```
	ifeq ($(KERNELRELEASE),)
	KDIR ?= /home/zcj/Code/Rpi/linux-rpi-4.9.y
	PWD := $(shell pwd)
	
	all:
	        make -C $(KDIR) M=$(PWD) modules
	
	clean:
	        rm -rf *.o *~ core .depend .*.cmd *.mod.c .tmp_versions *.ko *.symvers *.order
	
	.PHONY:clean
	
	else
	obj-m := hello.o
	endif
```
* 编译驱动模块，执行make命令即可，成功后会生成hello.ko文件。
* 通过scp或者其他方式将hello.ko文件拷贝到树莓派系统，然后执行命令：
```
	sudo insmod hello.ko 
```
即可成功加载我们编译好的内核驱动了。使用dmesg命令查看内核日志信息，可在日志信息的最后看到有输出"Hello world"，证明我们的编译的驱动模块已成功安装到内核中了。  


> 参考:
>  - [驱动学习-hello world](http://blog.csdn.net/xfwxqx/article/details/45338849)
>  - [树莓派内核(Kernel)的交叉编译](http://shumeipai.nxez.com/2013/10/09/raspberry-pi-kernel-cross-compiler.html)