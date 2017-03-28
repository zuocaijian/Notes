# 常用命令
* pwd：显示当前路径
* passwd user：修改用户密码
* su：switch user 切换用户
* useradd -m user：root下添加用户
* userdel user：root下删除用户
* echo $PATH：查看系统环境配置
* export PATH=$PATH:/home/work/：将work目录添加至环境变量
* whereis：搜寻命令的位置
* touch：创建文件
* cat：查看文本文件
* file /hello.txt：查看文件类型
* du -sh：查看当前文件下所有文件的大小
* tar -cf xx.tar xx：打包xx并创建文件xx.tar
* tar -tvf xx.tar：列出xx.tar的信息
* tar -cjf xx.bz2 xx1 xx2：压缩文件xx1和xx2到xx.bz2
* tar -czf xx.bz2 xx1 xx2：压缩文件xx1和xx2到xx.bz2
* tar -xf xx.tar -C xx：解压xx.tar到文件xx
* diff：比较两个文件/文件夹的差异
* find /dir -name xx：再dir目录下查找名为xx的文件
* wc -l xx 查看文件的行数
* find /dir -name "*.c" -exec wc -l {} \;

# Vim编辑器
* %!xxd：查看当前文本的二进制  

# shell编程  
	#!/bin/bash
	  
	read a
	read b
	c=`expr $a + $b`
	echo $c  
> **注意 = 两边没有空格，而 + 两边有空格**  

* 