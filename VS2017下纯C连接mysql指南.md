# 在VS2017下使用C语言连接mysql
---
## 环境说明
* Windows10 64位
* Visual Studio 2017
* MySQL 5.7.17 win64位

## 连接步骤
* 在[mysql官方站点](https://dev.mysql.com/downloads/connector/c/)下载安装C语言连接mysql所需要的头文件库文件等；
* 在VS2017中设置包含头文件路径：项目右键->属性->C/C++ 常规->附件包含目录，添加mysql安装目录下的“include”文件夹；
* 在VS2017中设置库文件路径：项目右键->属性->链接器 常规->附件库目录，添加mysql安装目录下的“lib”文件夹（包含libmysql.lib）；
* 在VS2017中设置依赖项：项目右键->属性->链接器 输入->附加依赖项，手动输入添加 libmysql.lib 项目；
* 将mysql安装目录下的lib/libmysql.dll文件拷贝到项目根目录；
* 在项目中包含头文件 #include<mysql.h>

## 使用说明
* 参考[官网开发指南](https://dev.mysql.com/doc/refman/5.7/en/c-api.html) 
* 参考相关博客：
	* [C++ API方式连接mysql数据库实现增删改查](http://blog.csdn.net/android_lover2014/article/details/52760717)
	* [Linux C语言连接MySQL 增删改查操作](http://asyty.iteye.com/blog/1447092)