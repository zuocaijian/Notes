# jni实现照片黑白滤镜效果
> 因为比较简单，所以就直接贴代码了。本文参考[JNI处理图片——黑白滤镜](http://www.jianshu.com/p/b68ee90e6121)

1. 申明本地方法：
	```
	/**
     * 将照片处理成黑白
     * @param bitmap
     */
    public static native void processBitmap(Bitmap bitmap);
	```
2. 使用javah命令生成头文件，得到本地方法格式如下：
	```
	/*
	 * Class:     com_zcj_test_utils_JNIUtil
	 * Method:    processBitmap
	 * Signature: (Landroid/graphics/Bitmap;)V
	 */
	JNIEXPORT void JNICALL Java_com_zcj_test_utils_JNIUtil_processBitmap(JNIEnv *, jclass, jobject);
	```
3. 新建bitmap_util.c文件，全部代码如下：
	```
	#include "com_zcj_test_utils_JNIUtil.h"
	
	#include <string.h>
	#include <android/bitmap.h>
	#include <android/log.h>
	
	#ifndef loge
	#define loge(...) ((void)__android_log_print(ANDROID_LOG_ERROR, "native_bitmap_utils: ", __VA_ARGS__))
	#endif
	
	//定义一些RGB565和RGB8888的一些基本操作
	//虽然RGB565的三色只有5位信息，但其值是8位，提供的5位信息是高5位的信息。
	#define MAKE_RGB565(r, g, b) (((r) >> 3) << 11) | (((g) >> 2) << 5) | ((b) >> 3)
	#define MAKE_ARGB(a, r, g, b) ((a&0xff) << 24) | ((r&0xff) << 16) | ((g&0xff) << 8) | (b&0xff)
	
	#define RGB565_R(p) ((((p) & 0xF800) >> 11) << 3)
	#define RGB565_G(p) ((((p) & 0x7E0) >> 5) << 2)
	#define RGB565_B(p) (((p) & 0x1F) << 3)
	
	#define RGB8888_A(p) (p & (0Xff << 24) >> 24)
	#define RGB8888_R(p) (p & (0Xff << 16) >> 24)
	#define RGB8888_G(p) (p & (0Xff << 8) >> 8)
	#define RGB8888_B(p) (p & 0Xff)
	
	JNIEXPORT void JNICALL Java_com_zcj_test_utils_JNIUtil_processBitmap
	  (JNIEnv *env, jclass cls, jobject bitmap)
	  {
	    if(NULL == bitmap)
	    {
	        loge("bitmap is null!\n");
	        return;
	    }
	
	    AndroidBitmapInfo bitmapInfo;
	    memset(&bitmapInfo, 0, sizeof(bitmapInfo));
	    AndroidBitmap_getInfo(env, bitmap, &bitmapInfo);
	
	    void *pixels = NULL;
	    int res = AndroidBitmap_lockPixels(env, bitmap, &pixels);
	
	    int x = 0;
	    int y = 0;
	    for(y = 0; y < bitmapInfo.height; ++y)
	    {
	        for(x = 0; x < bitmapInfo.width; ++x)
	        {
	            //int a = 0, r = 0, g = 0, b =0;
	            void *pixel  = NULL;
	
	            //获取像素
	            if(bitmapInfo.format == ANDROID_BITMAP_FORMAT_RGBA_8888)
	            {
	                pixel = ((uint32_t *)pixels + y * bitmapInfo.width + x);
	                int r, g, b;
	                uint32_t v = *((uint32_t *)pixel);
	                r = RGB8888_R(v);
	                g = RGB8888_G(v);
	                b = RGB8888_B(v);
	                int sum = r+g+b;
	                *((uint32_t *)pixel) = MAKE_ARGB(0x1f, sum / 3, sum / 3, sum / 3);
	
	            }
	            else if(bitmapInfo.format == ANDROID_BITMAP_FORMAT_RGB_565)
	            {
	                pixel = ((uint16_t *)pixels + y * bitmapInfo.width + x);
	                int r, g, b;
	                uint16_t v = *((uint16_t *)pixel);
	                r = RGB565_R(v);
	                g = RGB565_R(v);
	                b = RGB565_R(v);
	                int sum = r+g+b;
	                *((uint16_t *)pixel) = MAKE_RGB565(sum / 3, sum / 3, sum / 3);
	            }
	        }
	    }
	
	    AndroidBitmap_unlockPixels(env, bitmap);
	  }
	```
	bitmap_util.c文件主要分为三部分，第一部分为头文件声明， 包括我们自己生成的头文件，以及处理方法中用到的头文件；第二部分包括日志打印Log.e函数宏定义，以及RGB565和RGB8888两种Bitmap格式的取像素值和生成像素值函数宏定义；第三部分为像素的实际处理函数，也很简单，显示取出整个bitmap的点像素值，然后对每个点像素值的三基色求和取均值，生成一灰值像素，最终实现整张图片的黑白效果。
4. 编译规则，gradle提供了两种方式来进行c源码编译，一种是编写Android.mk，一种是使用CMake。
	1. 使用Android.mk。mk文件编写如下：
	```
	#设置源码路径为当店mk文件所处的路径
	LOCAL_PATH := $(call my-dir)
	#清除编译环境中前面设置的变量值
	include $(CLEAR_VARS)
	#目标应用程序二进制接口
	TARGET_ARCH_ABI := armeabi
	#生成的模块名，最终名形如libxx.so或libxx.a
	LOCAL_MODULE := bitmaputils
	#编译模块的源文件
	LOCAL_SRC_FILES := bitmap_util.c
	#生成模块需要链接的库文件
	LOCAL_LDLIBS := -llog -ljnigraphics -lc -lz -landroid
	#压制编译时警告
	LOCAL_DISABLE_FATAL_LINKER_WARNINGS := true
	#说明需要编译生成的是动态链接库
	include $(BUILD_SHARED_LIBRARY)
	```
	2. 使用CMake。CMakeList文件编写如下：
	```
	add_library(bitmaputils SHARED src/main/java/com/../bitmap_util.c
	find_library(log-lib log)
	target_link_libraries(bitmaputils jnigraphics $(log-lib))
	```
5. Java层调用如下：
	```
	private void process() {
        Bitmap bitmap = BitmapFactory.decodeResource(getResources(), R.mipmap.img);
        mImg1.setImageBitmap(bitmap);
        JNIUtil.processBitmap(bitmap);
    }
	```

### 可能出现的问题
1. 原因：javah -jni生成头文件时报错
	```
	无法确定Bitmap签名
	``` 
	解决方案：指定android.jar文件
	> windows环境下
	```
	javah -classpath C:\PROGRA~2\Android\android-sdk\platforms\android-8\android.jar;. xxx(java文件全类名，不要漏掉前面的".")
	```
2. 原因：由于libc不兼容警告导致的编译错误
	```
	D:/Android/sdk/ndk-bundle/build//../toolchains/x86_64-4.9/prebuilt/windows-x86_64/lib/gcc/x86_64-linux-android/4.9.x/../../../../x86_64-linux-android/bin\ld: warning: skipping incompatible D:/Android/sdk/ndk-bundle/build//../platforms/android-21/arch-x86_64/usr
	/lib/libc.a while searching for c
	D:/Android/sdk/ndk-bundle/build//../toolchains/x86_64-4.9/prebuilt/windows-x86_64/lib/gcc/x86_64-linux-android/4.9.x/../../../../x86_64-linux-android/bin\ld: error: treating warnings as errors
	clang++.exe
	
	```
	解决方案：在mk文件配置如下，禁止警告
	```
	LOCAL_DISABLE_FATAL_LINKER_WARNINGS := true
	```
3. LOCAL_LDLIBS和LOCAL_SHARED_LIBRARIES的区别：
	- LOCAL_LDLIBS ：链接的库不产生依赖关系，一般用于不需要重新编译的库，如库不存在，则会报错找不到。**只能链接那些存在于系统目录下本模块需要连接的库。**如果某一个库既有动态库又有静态库，那么在默认情况下是链接的动态库而非静态库；
	- LOCAL_SHARED_LIBRARIES 会生成依赖关系，**当库不存在时会去编译这个库。** 