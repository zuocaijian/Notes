> 国内使用Genymotion下载Android镜像经常会出现速度慢，或者下载失败的情况。这时候有两种解决办法，一是使用代理下载，二是找出镜像文件的下载地址，然后找第三方工具下载，最后离线安装。这里主要介绍方法二。

1. 打开Genymotion，选择需要下载的Android镜像文件，开始下载；
2. Win+R输入%appdata%,回车打开程序配置文件夹，快捷键 Alt+↑ 打开上级目录，选择local/genymotionmobile，可以看到该文件夹下有genymotion.log的文件；
3. 查找genymotion.log文件中类似 Downloading file 的文件，Linux下可使用查找命令： find genymotion.log | xargs grep Downloading file，即可获取到镜像的下载地址 http://file*****.ova；
4. 使用迅雷等第三方下载工具下载镜像；
5. 把下载的文件复制到C:\Users\用户主目录\AppData\Local\Genymobile\Genymotion\ova 中，根据文件修改时间，覆盖里面当前正在下载的镜像。该镜像就是刚刚使用genymotion软件点击下载的那个镜像；
6. 重复步骤1，即可发现镜像不需要下载了，验证安装即可。