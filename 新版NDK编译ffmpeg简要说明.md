# 编译环境
* 系统版本： ubuntu 16.04 lts
* 编译工具： android-ndk-r17c
* 编译源码： ffmpeg-master(2019/3/12)

# 编译步骤

## 1、 生成(安装)交叉编译链工具  
	> 参考：[自定义NDK交叉编译链](https://www.jianshu.com/p/141b45d972b1)  

*Android平台目前的架构有armeabi、armeabi-v7a、arm64-v8a、x86、x86_64、mips，我们需要指定不同的交叉编译工具链来完成不同平台架构库的编译工作。具体对应的工具链地址在`$(NDK)/toolchains`目录中*  

| 架构 | 工具链名称 |
| :----: | :----: |
| 基于 ARM | arm-linux-androideabi-&lt;gcc-version&gt; |
| 基于 x86 | x86-&lt;gcc-version&gt; |
| 基于 MIPS | mipsel-linux-android-&lt;gcc-version&gt; |
| 基于 ARM64 | aarch64-linux-android-&lt;gcc-version&gt; |
| 基于 x86-64 | x86_64-&lt;gcc-version&gt; |
| 基于 MIPS64 | mips64el-linux-android-&lt;gcc-version&gt; |

*关于手机CPU和ABI的兼容性关系说明。在应用安装运行时，CPU会选择其支持的ABI架构类型的.so文件，如果CPU支持多个ABI架构，会按照优先级进行选择*  

| CPU类型 | 支持的ABI及优先级（高到低） |
| :----: | :----: |
| ARMv5 | armeabi |
| ARMv7 | armeabi、armeabi-v7a |
| ARMv8 | armeabi、armeabi-v7a、arm64-v8 |
| MIPS  | mips |
| MIPS64| mips、mips64 |
| x86   | x86、armeabi、armeabi-v7a |
| x86_64| armeabi、x86、x86_64 |

 

可以使用NDK提供的`make-standalone-toolchain.sh`脚本来生成安装定制交叉编译工具链。切换到`$(NDK)/build/tools/`目录下，执行如下命令：
	```./make-standalone-toolchain.sh --arch=arm --platform=android-21 --install-dir=./arm/ --toolchain=arm-linux-androideabi-4.9```  
- 关于该脚本命令行参数说明：
	--arch 平台架构名称  
	--platform 目标平台api  
	--install-dir 交叉编译工具链生成安装目录  
	--toolchain 工具链名称  
- 命令执行后，会在`--install-dir`指定的安装目录生成对应的工具链文件夹。我们将使用该目录下工具链来编译ffmpeg源码。  
- 我们也可以编写shell脚本，一次性生成全部平台架构的交叉编译工具链。
	```
	#!/bin/sh

	export DEV=${HOME}/Library/Android/sdk
	export NDK_HOME=${DEV}/android-ndk-r10d

	platform=android-21
	shmake=$NDK_HOME/build/tools/make-standalone-toolchain.sh

	archs=(
    	'arm'
    	'arm64'
    	'x86'
    	'x86_64'
    	'mips'
    	'mips64'
	)

	toolchains=(
    'arm-linux-androideabi-4.9'
    'aarch64-linux-android-4.9'
    'x86-4.9'
    'x86_64-4.9'
    'mipsel-linux-android-4.9'
    'mips64el-linux-android-4.9'
	)

	echo $NDK_HOME
	num=${#archs[@]}
	for ((i=0;i<$num;i++))
	do
   		sh $shmake --arch=${archs[i]} --platform=$platform --install-dir=$HOME/Chain/android-toolchain/${archs[i]} --toolchain=${toolchains[i]}
	done
	```
## 2、 生成配置文件  
1. 在FFmpeg源码目录中创建`build_android.sh`脚本文件，内容如下：
```
	#!/bin/bash
	NDK=/home/zcj/Code/android-ndk-r17c
	TOOLCHAIN=$NDK/build/tools/arm
	SYSROOT=$NDK/platforms/android-19/arch-arm/
	
	function build_one
	{
	./configure \
	    --prefix=$PREFIX \
	    --target-os=android \
	    --arch=arm \
	    --enable-cross-compile \
	    --cross-prefix=$TOOLCHAIN/bin/arm-linux-androideabi- \
	    --enable-shared \
	    --disable-static \
	    --disable-doc \
	    --disable-ffmpeg \
	    --disable-ffplay \
	    --disable-ffprobe \
	    --disable-symver \
	    --enable-gpl \
	    --sysroot=$SYSROOT \
	    --extra-cflags="-Os -fPIC $ADDI_CFLAGS" \
	    --extra-ldflags="$ADDI_LDFLAGS" \
	    $ADDITIONAL_CONFIGURE_FLAG
	}
	CPU=armv7-a
	PREFIX=$(pwd)/Android/$CPU
	ADDI_CFLAGS="-I$NDK/sysroot/usr/include/arm-linux-androideabi -isysroot $NDK/sysroot -marm"
	build_one
```
- 注意这里的`--target-os=android`这里指定了目标系统为Android，编译后会将动态链接库命名成Android系统能识别的形式。若是将其赋值为`--target-os=linux`则需要修改`configure`文件。因为Android系统加载的动态链接库格式一般为libxx.so，而这与ffmpeg默认生成的动态链接库名称不一致，所以我们需要打开源码中的`configure`文件，找到其中的
```
SLIBNAME_WITH_MAJOR='$(SLIBNAME).$(LIBMAJOR)'
LIB_INSTALL_EXTRA_CMD='$$(RANLIB) "$(LIBDIR)/$(LIBNAME)"'
SLIB_INSTALL_NAME='$(SLIBNAME_WITH_VERSION)'
SLIB_INSTALL_LINKS='$(SLIBNAME_WITH_MAJOR) $(SLIBNAME)'
```

	替换为：
```
SLIBNAME_WITH_MAJOR='$(SLIBPREF)$(FULLNAME)-$(LIBMAJOR)$(SLIBSUF)'
LIB_INSTALL_EXTRA_CMD='$$(RANLIB) "$(LIBDIR)/$(LIBNAME)"'
SLIB_INSTALL_NAME='$(SLIBNAME_WITH_MAJOR)'
SLIB_INSTALL_LINKS='$(SLIBNAME)'
```  

- 由于我们已经在第一步中生成了指定平台架构、平台架构和工具链的全部工具副本，所以我们也可以直接将`SYSROOT`的位置指向`$TOOLCHAIN/sysroot`，这样我们就不用再往`--extra-cflags`中增加额外的头文件和库文件路径了，因为第一步中已将交叉编译需要的全部头文件和库文件都拷贝了一份到`$TOOLCHAIN/sysroot`中了。  
- 从NDK 18 开始，编译器有gnu系列替换为clang，因此编译ffmpeg时会导致部分函数参数类型不匹配而失败，目前还没有找到好的方法可以解决。    

2. 修改`build_android.sh`的权限模式为可执行：`chmod +x build_android.sh`
3. 执行脚本：`.\build_android.sh`
## 3、 编译  
执行make命令，并指定线程数加快编译速度：`make -j4`
## 4、 安装  
执行make命令：`make install`，顺利完成后将在`build_android.sh`脚本文件中`--prefix=$PREFIX`指定的目录下找到生成的动态链接库。

# 编译中出现的错误及解决方法
- 出现错误：
```
In file included from libavdevice/avdevice.c:19:0:
./libavutil/avassert.h:30:20: fatal error: stdlib.h: No such file or directory
 #include <stdlib.h>
                    ^
compilation terminated.
```
> 参考：[ffmpeg使用NDK编译时遇到的一些坑](https://blog.csdn.net/luo0xue/article/details/80048847)  

出现这个错误是因为使用最新版的NDK造成的。最新版的NDK将头文件和库文件进行了分离，我们指定的sysroot文件夹下只有库文件，而头文件放在了NDK目录下的`sysroot`内，只需在`--extra-cflags`中添加`-isysroot $NDK/sysroot`即可。还有有关汇编的头文件也进行了分离，需要根据目标平台进行指定`-I$NDK/sysroot/usr/include/arm-linux-androideabi`，将`arm-linux-androideabi`改为需要的平台，就可以顺利的进行编译了。