# 文件说明
* downcount.mp4： 用于测试的倒计时视频文件
* ffmpeg_linux_build.tar.gz：包含linux下编译ffmpeg的全部资源。
	1. [android-ndk-r14b](https://developer.android.google.cn/ndk/downloads/index.html)：linxu下的NDK工具包
	2. [ffmpeg-3.2.4.tar.bz2](http://ffmpeg.org/download.html)：linux下版本号为3.2.4的ffmpeg发行版源码
	3. build_android.sh：用于编译android下可用的so包的脚本文件

# 编译环境
* ubuntu 16.04 lts
* ...

# 编译步骤
* 编译linxu下ffmpeg可执行文件，包括ffmpeg、ffplay、ffserver、x264、yasm。具体参考[ffmpeg的wiki官方说明](https://trac.ffmpeg.org/wiki/CompilationGuide/Ubuntu#ffmpeg)  
	* 1. 获取依赖  
	> 1. sudo apt-get update  
	> 2. sudo apt-get -y install autoconf automake build-essential libass-dev libfreetype6-dev libsdl2-dev libtheora-dev libtool libva-dev libvdpau-dev libvorbis-dev libxcb1-dev libxcb-shm0-dev libxcb-xfixes0-dev pkg-config texinfo zlib1g-dev
	* 2. 创建主目录  
	> 1. mkdir ~/ffmpeg_sources
	* 3. 下载编译安装全部类库
	> 1. **yasm**  
		sudo apt-get install yasm  
		或者  
		cd ~/ffmpeg_sources  
		wget http://www.tortall.net/projects/yasm/releases/yasm-1.3.0.tar.gz  
		tar xzvf yasm-1.3.0.tar.gz  
		cd yasm-1.3.0  
		./configure --prefix="$HOME/ffmpeg_build" --bindir="$HOME/bin"  
		make  
		make install  
	> 2. **libx264**  
		sudo apt-get install libx264-dev  
		或者  
		cd ~/ffmpeg_sources  
		wget http://download.videolan.org/pub/x264/snapshots/last_x264.tar.bz2  
		tar xjvf last_x264.tar.bz2  
		cd x264-snapshot*  
		PATH="$HOME/bin:$PATH" ./configure --prefix="$HOME/ffmpeg_build" --bindir="$HOME/bin" --enable-static --disable-opencl  
		PATH="$HOME/bin:$PATH" make  
		make install  
	> 3. **libx265**  
		sudo apt-get install libx265-dev  
		或者  
		sudo apt-get install cmake mercurial
		cd ~/ffmpeg_sources  
		hg clone https://bitbucket.org/multicoreware/x265  
		cd ~/ffmpeg_sources/x265/build/linux  
		PATH="$HOME/bin:$PATH" cmake -G "Unix Makefiles" -DCMAKE_INSTALL_PREFIX="$HOME/ffmpeg_build" -DENABLE_SHARED:bool=off ../../source make  
		make install  
	> 4. **libfdk-aac**  
		sudo apt-get install libfdk-aac-dev  
		或者  
		cd ~/ffmpeg_sources  
		wget -O fdk-aac.tar.gz https://github.com/mstorsjo/fdk-aac/tarball/master  
		tar xzvf fdk-aac.tar.gz  
		cd mstorsjo-fdk-aac*  
		autoreconf -fiv  
		./configure --prefix="$HOME/ffmpeg_build" --disable-shared  
		make  
		make install  
	> 5. **libmp3lame**  
		sudo apt-get install libmp3lame-dev  
		或者  
		sudo apt-get install nasm  
		cd ~/ffmpeg_sources  
		wget http://downloads.sourceforge.net/project/lame/lame/3.99/lame-3.99.5.tar.gz  
		tar xzvf lame-3.99.5.tar.gz  
		cd lame-3.99.5  
		autoreconf -fiv  
		./configure --prefix="$HOME/ffmpeg_build" --enable-nasm --disable-shared  
		make  
		make install  
	> 6. **libopus**  
		sudo apt-get install libopus-dev  
		或者   
		cd ~/ffmpeg_sources  
		wget http://downloads.xiph.org/releases/opus/opus-1.1.4.tar.gz  
		tar xzvf opus-1.1.4.tar.gz  
		cd opus-1.1.4  
		./configure --prefix="$HOME/ffmpeg_build" --disable-shared  
		make  
		make install  
	> 7. **libvpx**  
		sudo apt-get install libvpx-dev  
		或者   
		cd ~/ffmpeg_sources  
		wget http://storage.googleapis.com/downloads.webmproject.org/releases/webm/libvpx-1.6.1.tar.bz2  
		tar xjvf libvpx-1.6.1.tar.bz2  
		cd libvpx-1.6.1  
		PATH="$HOME/bin:$PATH" ./configure --prefix="$HOME/ffmpeg_build" --disable-examples --disable-unit-tests  
		PATH="$HOME/bin:$PATH" make  
		make install
	* 4. 下载ffmpeg源码，配置、编译并安装  
	> cd ~/ffmpeg_sources  
	> wget http://ffmpeg.org/releases/ffmpeg-snapshot.tar.bz2  
	> tar xjvf ffmpeg-snapshot.tar.bz2  
	> cd ffmpeg  
	> PATH="$HOME/bin:$PATH" PKG_CONFIG_PATH="$HOME/ffmpeg_build/lib/pkgconfig" ./configure --prefix="$HOME/ffmpeg_build" --pkg-config-flags="--static" --extra-cflags="-I$HOME/ffmpeg_build/include" --extra-ldflags="-L$HOME/ffmpeg_build/lib" --bindir="$HOME/bin" --enable-gpl --enable-libass --enable-libfdk-aac --enable-libfreetype --enable-libmp3lame --enable-libopus --enable-libtheora --enable-libvorbis --enable-libvpx --enable-libx264 --enable-libx265 --enable-nonfree  
	> PATH="$HOME/bin:$PATH" make  
	> make install  
	> hash -r  
  
	**`1. 如果全部通过sudo apt-get 安装类库，则在第4步中不用配置PATH PKG_CONFIG_PATH等变量；`**  
	**`2. 生成的可执行文件在~/bin/目录下。`**  
  
# 升级、撤销、卸载方式  
* `rm -rf ~/ffmpeg_build ~/ffmpeg_sources ~/bin/{ffmpeg,ffprobe,ffplay,ffserver,vsyasm,x264,x265,yasm,ytasm}`
* 更多细节参考[官网](https://trac.ffmpeg.org/wiki/CompilationGuide/Ubuntu#RevertingChangesMadebyThisGuide)  

# 简单使用说明
* `cd ~/bin && ./ffmpeg -i ~/input.mp4 ~/videos/output.mkv`
* 更多用法可参考[《FFMPEG视音频编解码零基础学习方法》](http://blog.csdn.net/leixiaohua1020/article/details/15811977/)以及[《ffmpeg参数中文详细解释》](http://blog.csdn.net/leixiaohua1020/article/details/12751349)
* 源码解析可参考[《 FFmpeg源代码简单分析：makefile》](http://blog.csdn.net/leixiaohua1020/article/details/44556525#comments)系列博客  

# 编译Android下可用的so动态链接库
* [官方文档](https://trac.ffmpeg.org/wiki/CompilationGuide/Android)
* 中文简单编译步骤可参考博客[《FFmpeg的Android平台移植—编译篇》](http://blog.csdn.net/gobitan/article/details/22750719)。其中build_android.sh脚本文件可在压缩包内找到，只需替换NDK路径即可