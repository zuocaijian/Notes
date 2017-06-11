使用Google提供的MultiDex可解决一个app方法数超过65535而导致INSTALL_FAILED_DEXOPT的问题。然而该方案在dex分包时自行分析了类间的关联关系，自动将启动app所关联的类放置在主dex中。如果我们希望对dex分包做到精准控制，便需要对Gradle进行一些配置（本方法只讨论使用gradle构建项目时的dex多分包配置，不涉及使用ant构建的多分包配置）。
***
在Android插件化及热修复技术从无到有的发展过程中，有一种方案是基于dex多分包实现的：利用jvm虚拟机的类加载机制，使用DexClassLoader类加载器，配合反射技术，动态的将补丁包dex插入到dexElements中，从而屏蔽掉被替换的dex文件的加载，以达到动态修复bug或者模块热插拔的目的。
要实现以上方式的热修复，必须使用到dex多分包，且需要对dex多分包进行精准控制，保证主dex文件完全没有bug，而将项目的二级功能模块放置在从dex中。这样一旦从dex中的类出现bug，便可进行动态替换，从而实现热修复。
[彻底掌握Android多分包技术MultiDex-用Ant和Gradle分别构建（二）](http://blog.csdn.net/mynameishuangshuai/article/details/52716877)这篇博客提到了使用Gradle构建项目时的多分包配置，步骤如下：  

	1、buildToolsVersion要求21.1版本及以上；
	2、在module的build.gradle文件中defaultConfig节点下添加 multiDexEnabled true这个配置项  
	3、在module的build.gradle文件中添加multidex的依赖：  
		compile 'com.android.support:multidex:1.0.0'  
	4、在代码中加入multidex功能，共三种方式：
		1)在manifest文件中指定Application为MultiDexApplication
			<application
					android:name='android.support.multidex.MultidexApplication'
			...>
		2)让应用的Application继承MultidexApplication
			public class BaseApplication extends MultidexApplication{...}
		3)重写Application的attachBaseContext方法，并在onCreate前执行
			public class BaseApplication extends Application{
				@Override
				protected void attachBaseContext(Context base)}
					super.attachBaseContext(base);
					MultiDex.install(this);
				}
			}
	5、在module的build.gradle文件中添加afterEvaluate区域，并在其内部采用-main-dex-list选项来指定主dex中要包含的类
		afterEvaluate {
    		println("castiel--afterEvaluate")

    		tasks.matching {
       	 		println("task name: " + it.name)
        		it.name.startsWith("dex")
    		} each { dx ->
        		//定义主包包含的类列表
        		def listFile = project.rootDir.absolutePath + '/app/maindexlist.txt'
        		println("castiel--root dir: " + project.rootDir.absolutePath)
        		println("dex task found " + dx.name)
        		if (dx.additionalParameters == null) {
            		dx.additionalParameters = []
        		}
        		dx.additionalParameters += '--multi-dex' //表示当方法数越界时生成多个dex
        		dx.additionalParameters += '--main-dex-list=' + listFile
        		dx.additionalParameters += '--minimal-main-dex'
        		dx.additionalParameters += '--set-max-idx-number=48000'
    		}  
		}
实际测试中发现，经过上述配置后没有生成多个dex。 在AndroidStudio2.3.3， Gradle版本3.3， Gradle plugin插件版本2.3.3环境中，Gradle并没有以“dex”开头的任务。 Gradle插件1.3.0以前，该方法是可行的，不过在1.5.0后就失效了。取而代之的是更加简单话的dexOptions分包配置：  

	dexOptions {
		javaMaxHeapSize "4g"
        preDexLibraries = false
        additionalParameters = [
                '--multi-dex',
                '--set-max-idx-number=48000',
                '--main-dex-list=' + project.rootDir.absolutePath + '/app/maindexlist.txt',
                '--minimal-main-dex'
        ]
	}

以上，maindexlist.txt放置在module根目录下，指明了主dex文件中需要包含的class类，如下所示：  

	//主包中包含的类
	com/example/testapp/MainActivity.class

	// multidex
	android/support/multidex/MultiDex.class
	android/support/multidex/MultiDexApplication.class
	android/support/multidex/MultiDexExtractor$1.class
	android/support/multidex/MultiDexExtractor.class
	android/support/multidex/MultiDex$V14.class
	android/support/multidex/MultiDex$V19.class
	android/support/multidex/MultiDex$V4.class
	android/support/multidex/ZipUtil$CentralDirectory.class
	android/support/multidex/ZipUtil.class  

参考：
[Android Dex分包之旅](http://yydcdut.com/2016/03/20/split-dex/)
[dex分包变形记](http://blog.csdn.net/Tencent_Bugly/article/details/50068723)
[Android Studio Gradle配置dex分包](http://blog.csdn.net/q1113225201/article/details/53242203)
[Android项目构建过程](http://blog.csdn.net/qq_23547831/article/details/50634435)