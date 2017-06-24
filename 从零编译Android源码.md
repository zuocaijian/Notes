# 环境  
  1. 编译的初始环境：Windows 10 64bit;
  2. 编译的目标系统版本：Android4.4.4。

# 编译环境准备工作  
  1. [下载VMWare Workstation 12 pro 64bit](http://rj.baidu.com/soft/detail/13808.html?ald)，安装。自行添加序列号，请支持正版；
  2. [下载Ubuntu 16.04 16bit](https://www.ubuntu.com/download/desktop);
  3. 在VMWare里新建虚拟机，选择**自定义**安装。需要注意的有：
	  * 安装来源：选择稍后安装操作系统
	  * 处理器：根据物理机的配置，建议分给客户机一半的资源
	  * 虚拟内存：根据物理机的配置，建议分给客户机一半的资源
	  * 最大磁盘大小：建议为128G+，**选择 将虚拟磁盘存储为单个文件**
  4. 在新建的虚拟机里安装下载好的Ubuntu镜像：
	  * 右键新建的虚拟机，选择CD/DVD，选择“使用ISO映像文件”，选择系统镜像文件，然后确定
	  * 按正常流程在虚拟机里安装Ubuntu系统。建议选择英语，如果选择简体中文，需要花费漫长的时间来下载语言包；
	  * 安装系统镜像完成后需要安装VMWare tools增强工具。关闭虚拟机，步骤1重新到“选择ISO映像文件”，选择VMWare安装目录下的linux.iso，然后确定，重新开机。使用**sudo passwd**设置root账户初始密码，然后安装/media/用户名/VMWare tools目录下的vmware-install.pl。因为涉及到权限等问题，建议将VMWare tools整个目录拷贝到用户目录下，解压之后安装，最后重启虚拟机；
  5. 在Ubuntu镜像源列表中选择靠谱的软件源（如china下的阿里云、163等），安装vim等常用软件；
  6. 安装JDK。因为不同的Android版本会需要不同的jdk版本，因此需要安装多个不同版本的jdk。比如：
	  * 使用'sudo apt-get install openjdk-8-jdk'命令，安装openjdk8；
	  * 使用'sudo add-apt-repository ppa:webupd8team/java'命令添加ppa源，然后'sudo apt-get update'更新源列表，最后'sudo apt-get install oracle-java7-installer'安装oracle jdk8；
	  * 从网上下载jdk1.6 64bit的安装文件jdk-6u45-linux-x64.bin，然后直接安装；
	  
	  > 方式1、2安装后安装目录为/usr/lib/jvm/jdk-xx，方式3安装后安装目录为/usr/local/bin/jvm/jdk-xx。为方便起见，方式3需要将jvm目录剪切合并到/usr/lib/jvm中
	  
	  * vim打开/etc/profile文件，在文件的末尾添加配置jdk环境到系统：  
	  
			#open jdk 8
			# export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
			#oracle jdk 8
			# export JAVA_HOME=/usr/lib/jvm/java-8-oracle
			#oracle jdk 6
			export JAVA_HOME=/usr/lib/jvm/jdk1.6.0_45
			export JRE_HOME=$JAVA_HOME/jre
			export CLASSPATH=$JAVA_HOME/lib:$JRE_HOME/lib:$CLASSPATH
			export PATH=$JAVA_HOME/bin:$JRE_HOME/bin:$PATH

	    打开或关闭注释可方便的切换jdk环境，每次切换环境后记得使用 'source /etc/profile' 命令使其临时生效。建议切换jdk环境后重启虚拟机；
  7. 安装对应版本的make。编译Android4.4.4需要的make版本号为3.8.1~3.8.2，而Ubuntu 16.04自带的make版本号为4.1。因此需要到[GNU官网](http://www.gnu.org/software/make/)下载对应版本的[make源码](http://ftp.gnu.org/gnu/make/)，编译安装（./configure;make;make install）。然后将当前目录下的 make 替换到/usr/bin/make，建议替换之后重启虚拟机(如果使用了make install命令，需要注意install是否成功，如果成功，则生成的make可能不在当前目录下)；
  8. 安装编译Android源码以及Liunx内核需要用到的一个基于文本的图形终端动态库ncurses。使用'sudo apt-get install libncurses5-dev'命令即可完成安装；
  9. 根据官网建议，安装编译Android源码需要用到的一些依赖库:  
  
			sudo apt-get install libx11-dev:i386 libreadline6-dev:i386 libgl1-mesa-dev g++-multilib  
			sudo apt-get install -y git flex bison gperf build-essential libncurses5-dev:i386  
			sudo apt-get install tofrodos python-markdown libxml2-utils xsltproc zlib1g-dev:i386   
			sudo apt-get install dpkg-dev libsdl1.2-dev libesd0-dev  
			sudo apt-get install git-core gnupg flex bison gperf build-essential    
			sudo apt-get install zip curl zlib1g-dev gcc-multilib g++-multilib  
			sudo apt-get install libc6-dev-i386  
			sudo apt-get install lib32ncurses5-dev x11proto-core-dev libx11-dev   
			sudo apt-get install libgl1-mesa-dev libxml2-utils xsltproc unzip m4  
			sudo apt-get install lib32z-dev ccache  

  10. 初始化Android源码编译的环境：source build/envsetup.sh。建议初始化后重启虚拟机  

> **Tips**:  
> 1. 编译环境系统及软件最好统一使用64位，增加不必要的麻烦，提高成功率；  
> 2. 环境配置及源码编译可参考[在Ubuntu上下载、编译和安装Android最新源代码](http://blog.csdn.net/luoshengyang/article/details/6559955)、[在Ubuntu上下载、编译和安装Android最新内核源代码（Linux Kernel）](http://blog.csdn.net/luoshengyang/article/details/6564592)和[自己动手编译Android源码(超详细)](http://www.jianshu.com/p/367f0886e62b)。  

# 下载源码
  1. 下载Android源码。由于Android源码十分大，且开源服务器在国外，下载起来非常缓慢，难以忍受。因此可以寻找其他方式，如镜像源下载。关于下载方式可上网查看教程。这里建议直接从[百度网盘](http://pan.baidu.com/s/1ngsZs)下载别人打包好的源码压缩文件；
  2. 在Ubuntu中解压缩压缩包（实测，在windows下解压该压缩文件，会出现文件重复等错误，导致最终的解压缩文件有问题）。由于Ubuntu原生并没有7zip压缩工具，因此需要使用 'sudo apt-get install p7zip-full' 下载，然后使用 '7z x 压缩包 -r -o目标目录' 命令将源码压缩包解压到指定文件夹(注意-o后面没有空格，直接跟目标目录)；
  3. 关于Android源码各个目录的用途可参考[Android 5.1.1 源码目录结构](http://blog.csdn.net/tfslovexizi/article/details/51888458);
  4. 下载Android内核源码。由于Android开放源码工程中并不包括Kernel源码（系统镜像默认使用源码目录prebuild/Qume_kernel下已编译好的对应内核），因此Kernel内核需要另行下载。可使用国内镜像加快下载速度，镜像地址可参考[不翻墙下载Android内核源码](http://blog.csdn.net/sunao2002002/article/details/53057374)，使用命令 'git clone  https://aosp.tuna.tsinghua.edu.cn/kernel/common.git' 即可将项目源码克隆到本地。然后在common目录下 'git branch -a' 查看所有分支，最后 'git checkout XXX' 选择切换到需要的分支。源码大概1.5G。  

# 编译源码
  1. 选择编译目标。切换到Android源码目录下，输入 'lunch' 命令可打印出所有的编译目标，然后键入对应的数字选择相应的编译目标(一定要记得先初始化编译环境 'source envsetup.sh')；
  2. 开始编译。键入 'make' 命令即可开始编译了，同时可以使用 -j 参数来指定线程数。线程数一般为cpu core数量*2，如 'make -j8' 即是指定8条线程来编译。如果一切顺利，可在该目录下的out文件夹看到.img系统镜像了。  

> **Tips**  
> 测试1：i5 3230m，编译花了2h+；
> 测试2：i7 4790，编译花了约110min；

# 使用模拟器
  1. 编译完成之后，就可以通过以下命令运行Android模拟器了：  

		source build/envsetup.sh  
		lunch(选择刚才你设置的目标版本)  
		emulator

>建议可阅读[官方编译指南](http://source.android.com/source/building.html)  

# 编译内核

> 编译玩系统源码后打开模拟器所使用的内核是预编译好的（文件位置为），但是在实际练习的时候，我们经常会涉及到驱动文件的编写，内核文件的修改调试，这些都需要通过自定义内核来实现。因此，学习如何编译内核是非常有必要的。相对于编译系统源码来说，编译内核要简单的多。

  1. 准备环境。如上，同编译系统源码一样。不过要记得 source build/envsetup.sh，不然许多命令会找不到；  
  2. 下载内核源码。由于我们使用的是模拟器，因此需要下载对应的源码，这里推荐使用中科大的镜像源下载：?????????????????????，下载完成后源码共1.3G左右；
  3. 使用 'git branch -a' 查看所有分支。因为我们编译的系统版本为Android 4.4，在模拟器中查看到其对应的内核版本为3.4.0，因此，这里我们也需要切换到goldfish 3.4分支，使用命令 'git checkout remotes/origin/android-goldfish-3.4' 即可；  
  4. 导出交叉编译工具目录到PATH环境变量中去，执行命令：export PATH=$PATH:~XX/Android/prebuilts/gcc/linux-x86/arm/arm-eabi-4.7；  
  5. 修改kernel/common根目录下的Makefile文件，将以下两行改为：  
  
	  	 ARCH ?= arm
		 CROSS_COMPILE ?= arm-eabi-  

	注意，该两行后面不能有空格。
  6. 选择版本。使用 'lunch' 命令，然后输入数字选择对应的版本。我这里输入的为1，即 `1. aosp_arm-eng`；
  7. 配置。使用 'make goldfish_armv7_defconfig' 命令。注意 **Android系统版本4.0以下，编译内核的时候使用的配置是 goldfish_defconfig，但是4.0以上的版本的系统需要ARMv7架构或者以上才能运行，因此需要使用配饰goldfish_armv7_defconfig**;
  8. 细化配置。使用 'make menuconfig' 可修改详细的内核配置参数。该步可选，建议新手不修改；
  9.  编译。使用 'make' 命令开始编译。同样可以使用 '-j' 参数来指定编译线程数量以加快编译。不过内核编译的速度一般很快，通常几分钟就能完成，编译完成后的文件为kernel/common/arch/arm/boot/zImage；
  10.  切换到Android源码目录下，'emulator -kernel ./kernel/common/arch/arm/boot/zImage &' 即可开启模拟器并指定内核为我们刚才编译好的内核；
  11.  使用 adb 相关命令或者直接查看 设置 里的手机信息，可发现，模拟器内核已经替换为我们编译好的内核。
  	 