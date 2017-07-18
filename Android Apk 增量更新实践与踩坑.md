> 本文主要思路参考张鸿洋的博客[Android 增量更新完全解析 是增量不是热修复](http://blog.csdn.net/lmj623565791/article/details/52761658)

1. 增量更新的差分包生成与合并主要用到bsdiff，官方主页为[http://www.daemonology.net/bsdiff/](http://www.daemonology.net/bsdiff/)。我们需要下载[bsdiff](http://www.daemonology.net/bsdiff/bsdiff-4.3.tar.gz)，然后解压：tar -xzvf ..。解压后编译：make，发现报错，如下所示：
```
Makefile:13: *** missing separator.  Stop.
```
原因是Makefile的缩进格式不对,打开Makefile，将install:下面的if，endif添加一个缩进。
2. 安装bsdiff需要用到的依赖库bzip2：sudo apt-get install libbz2-dev
3. 重新执行make。如果你使用的是ubuntu，那么你很可能跟我一样，发现仍然报错：
```
cc -O3 -lbz2 bsdiff.c -o bsdiff
/tmp/cc3OAqLk.o: In function `main':
bsdiff.c:(.text.startup+0x2b3): undefined reference to `BZ2_bzWriteOpen'
bsdiff.c:(.text.startup+0x968): undefined reference to `BZ2_bzWrite'
bsdiff.c:(.text.startup+0x9b0): undefined reference to `BZ2_bzWrite'
bsdiff.c:(.text.startup+0xa0c): undefined reference to `BZ2_bzWrite'
bsdiff.c:(.text.startup+0xa6b): undefined reference to `BZ2_bzWriteClose'
bsdiff.c:(.text.startup+0xabe): undefined reference to `BZ2_bzWriteOpen'
bsdiff.c:(.text.startup+0xae9): undefined reference to `BZ2_bzWrite'
bsdiff.c:(.text.startup+0xb0f): undefined reference to `BZ2_bzWriteClose'
bsdiff.c:(.text.startup+0xb61): undefined reference to `BZ2_bzWriteOpen'
bsdiff.c:(.text.startup+0xb8c): undefined reference to `BZ2_bzWrite'
bsdiff.c:(.text.startup+0xbb2): undefined reference to `BZ2_bzWriteClose'
collect2: ld 返回 1
make: *** [bsdiff] 错误 1
```
此时，有三种解决方案：
	- 方案一：参考[Android 增量更新 bsdiff bspatch](http://blog.csdn.net/shichaosong/article/details/21179703)。如果你有Android源码，那么此种解决方案是可以考虑的。不过我并没有亲自实践过，结果不予保证。
	- 方案二：手动编译。首先为了确保libbz2库正确安装，前往[bzip2主页](http://www.bzip.org/)，下载最新的bzip2，下载好了后解压(tar -xzvf ...)，编译(make),安装(sudo make install)。然后使用gcc命令分别编译bsdiff和bspatch
	```
	gcc -bsdiff.c -lbz2 -o bsdiff
	gcc -bspatch.c -lbz2 -o bspatch
	```
	- 方案三：与方案二一样，改写Makefile。将其中的bsdiff、bspatch的编译规则改写如下：
	```
	all:            bsdiff bspatch
	bsdiff:         
        cc bsdiff.c -lbz2 -o bsdiff
	bspatch:        
        cc bspatch.c -lbz2 -o bspatch
	```
编译好之后就生成了可执行文件bsdiff和bspatch。
>**猜测：进过多次试验，应该是bzip2中的头文件包含关系有问题，或者说我下载的bzip1.0.6版本与bsdiff不匹配。该错误在Android NDK编译中也会出现，下文会给出解决方案。**

4. 简单验证bsdiff和bspatch是否可用。
	- 创建文本文件test1，在其中写入：
	```
	This is the first text!
	```
	- 复制文本文件test1为test2，在结尾追加一行，变成：
	```
	This is the first text!
	modified text.
	```
	- 使用bsdiff生成patch文件：
	```
	bsdiff text1 text2 patch
	```
	可在当前文件夹下面看到生成的补丁文件 patch
	- 使用bspatch合并旧文件和patch：
	```
	bspatch text1 text3 patch
	```
	可以看到，生成了新的文件text3，并且text3内容与修改后的text2文件一摸一样。
> 为了确保合并后的文件确实与修改后的文件一致，可使用md5校验。ubuntu下可直接使用“md5sum file”命令查看文件md5。
5. bsdiff实在服务器端生成新旧文件patch包用的，而bspatch是在本地合并差分包时用的。因此，我们的应用只需要用到bspatch即可。同样有两种方式：
	- 直接用交叉编译链工具，生成Android可用的bspatch so文件；
	- copy相关源码，通过Android IDE自带的构建工具在编译Apk的时候生成bspatch。
两种方式都相对比较简单，一下主要介绍方式二。方式一只做个简单的介绍说明。**需要指出的是，因为bsdiff和bspatch都依赖到了bzip2，因此需要下载bzip2的源码，具体地址在本文前面可找到。**
6. 简单说明下ubuntu下使用交叉编译工具链生成Android可用的bspatch的方法。首先需要下载好Android提供的linux下交叉编译工具链NDK，安装与配置可参考[Ubuntu搭建Android交叉编译环境](http://www.cnblogs.com/xieyajie/p/4727706.html)。Android源码中也包含了交叉编译工具链，文件目录为/Android/prebuilts/gcc/linux-x86/arm/arm-...。linux下交叉编译比较典型的Makefile写法如下：  
```
	# Show how to cross-compile c/c++ code for android platform
	
	.PHONY: clean
	
	NDKROOT=/opt/android/ndk
	
	PLATFORM=$(NDKROOT)/platforms/android-14/arch-arm
	
	CROSS_COMPILE=$(NDKROOT)/toolchains/arm-linux-androideabi-4.6/prebuilt/linux-x86/bin/arm-linux-androideabi-
	
	CC=$(CROSS_COMPILE)gcc
	
	AR=$(CROSS_COMPILE)ar
	
	LD=$(CROSS_COMPILE)ld
	
	CFLAGS = -I$(PWD) -I$(PLATFORM)/usr/include -Wall -O2 -fPIC -DANDROID -DHAVE_PTHREAD -mfpu=neon -mfloat-abi=softfp
	
	LDFLAGS =
	
	TARGET = libmath.a
	
	SRCS = $(wildcard *.c)
	
	OBJS = $(SRCS:.c=.o)
	
	all: $(OBJS)
	
	        $(AR) -rc $(TARGET) $(OBJS)
	
	clean:
	
	        rm -f *.o *.a *.so
```
针对Android环境的Makefile文件编写注意如下：
	- **CROSS_COMPILE**  
		必须正确给出Android NDK编译工具链的路径，当在目录中执行make命令的时候，编译系统会根据 CROSS_COMPILE 前缀寻找对应的编译命令。
	- **-I$(PLATFORM)/usr/include**  
		由于Android平台没有使用传统的c语言库libc，而是自己编写了一套更加高效更适合嵌入式平台的c语言库，所以系统头文件目录不能再使用默认的路径，必须指到Android平台的头文件目录。
	- **-Wall -O2 -fPIC -DANDROID -DHAVE_PTHREAD -mfpu=neon -mfloat-abi=softfp**  
		gcc常用的参数，可自行上网搜索。
当然对于非常简单的.c文件(不涉及android特用的lib库)编译是不需要用到Makefile的，可以export交叉编译工具链到系统环境(export PATH=$PATH:toolchains/arm-linux-androideabi-4.6/prebuilt/linux-x86_64/bin/)，然后直接使用arm-linux-androideabi-gcc交叉编译命令即可。或者不导入编译环境，而是在每次使用命令时 --sysroot 指定编译时使用的头文件和库文件，即“arm-linux-androideabi-gcc --sysroot=$NDK_BASE/platforms/android-19/arch-arm hello.c -o hello”。这两种方法可参考[【ndk】直接使用ndk提供的arm-linux-androideabi-gcc编译android可执行程序](http://blog.csdn.net/smilefyx/article/details/73692126)。
7. 接下来的工作主要就是本地JNI开发了。
	1. 提取已安装应用的apk文件，代码如下：  
	```
    	public static String extract(Context context) {
    		context = context.getApplicationContext();
        	ApplicationInfo applicationInfo = context.getApplicationInfo();
        	String apkPath = applicationInfo.sourceDir;
        	Log.d("hongyang", apkPath);
        	return apkPath;
    	}
	```
	2. 生成bspatch so。各个Gradle版本的NDK配置不是本次重点，因此略过。
		1. 在Java层申明本地方法：
		```
		public native int bspatch(String oldApk, String newApk, String patch);
		```
		其中，oldApk是已安装apk文件的路径，newApk是通过差分包合成后的文件路径，patch是下载的差分包的路径。
		2. 新建execBsPatch.c文件，实现本地方法：
		```
		#include "com_example_testapp_activity_Activity_Patch_Apk.h"
		#include "bspatch.c"
		
		JNIEXPORT jint JNICALL Java_com_example_testapp_activity_Activity_1Patch_1Apk_bspatch
		  (JNIEnv *env, jobject cls, jstring old, jstring new, jstring patch)
		{
			    int argc = 4;
			    char *argv[argc];
			    argv[0] = "bspatch";
			    argv[1] = (char *)((*env)->GetStringUTFChars(env, old, 0));
			    argv[2] = (char *)((*env)->GetStringUTFChars(env, new, 0));
			    argv[3] = (char *)((*env)->GetStringUTFChars(env, patch, 0));
			
			    int ret = patchMethod(argc, argv);
			
			    (*env)->ReleaseStringUTFChars(env, old, argv[1]);
			    (*env)->ReleaseStringUTFChars(env, new, argv[2]);
			    (*env)->ReleaseStringUTFChars(env, patch, argv[3]);
			
			    return ret;
		}
		```
		其中bspatch.c即为我们下载好的bsdiff压缩包内的文件，拷贝到与execBsPatch.c同目录下即可。patchMethod即为bspatch.c的main方法，需要我们手动将mian方法改名为patchMethod。
		3. 拷贝我们下载的bzip2压缩包内的.c和.h文件到jni目录下。为方便起见，我们在jni目录下新建bzip2文件夹，专门放置bzip2相关的.c和.h文件。同时，我们需要将bspatch.c中对bzlib.h头文件包含更改为 #include "bzip2/bzlib.h"。
		到这一步，JNI层代码本该算是已经完成，可是当进行编译的时候，会报各种各样的"undefined reference to xxxx..."，让人很是崩溃。经过我几个小时的试错，一个个文件的排查，最终发现是由于头文件包含关系问题而导致的错误。最终修改如下：
			- 在bspatch.c文件中，需要同时包含bzlib.h和bzlib.c文件，即我们需要添加：
			```
			#include "bzip2/bzlib.c"
			```
			- 在bzip2/bzlib.c文件中，需要添加：
			```
			#include "compress.c"
			#include "decompress.c"
			#include "crctable.c"
			```
			- 在bzip2/compress.c中，需要添加：
			```
			#include "blocksort.c"
			#include "huffman.c"
			```
			- 在bzip2/decompress.c中，需要添加：
			```
			#include "randtable.c"
			```
		至此，最艰难的一步算是完成。
	3. 现在，只要java层调用本地方法合并差分包生成安装文件安装即可。代码如下：
	```
	private void doBspatch() {
    	final File destApk = new File(Environment.getExternalStorageDirectory(), "dest.apk");
    	final File patch = new File(Environment.getExternalStorageDirectory(), "PATCH.patch");

    	//一定要检查文件都存在

    	bspatch(extract(this),
            	destApk.getAbsolutePath(),
            	patch.getAbsolutePath());

    	if (destApk.exists())
        	install(this, destApk.getAbsolutePath());
    }

	public static void install(Context context, String apkPath) {
        Intent i = new Intent(Intent.ACTION_VIEW);
        i.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        i.setDataAndType(Uri.fromFile(new File(apkPath)),
                "application/vnd.android.package-archive");
        context.startActivity(i);
    }
	```
	4. 编译项目，生成新旧两个apk，然后在PC端做差分包，下载到手机内置储存卡，就可以愉快的检验我们的增量更新是否成功了。
8. 虽然网上有很多现成的项目可以copy，但是自己从头走一遍仍然收获颇多。如果只是简单的看一遍别人写过的代码，根本想不到这其中竟然有这么多的坑。让我开心的是，这其中大部分的坑坑，完全是我自己独立解决的，相信很多人写的类似的文章根本没有碰到(也许直接拿别人的代码改改而已)或者碰到了也没有给出解决方案。因此，此次增量更新实践虽然花费了我近一天的时间，不过填了这么多坑，也算是值了。