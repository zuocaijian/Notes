## Ubuntu 16.04 64bit 安装Gogs

### 一、 安装
 1. 下载go语言源码安装包
 ```
 wget https://www.golangtc.com/static/go/1.9.2/go1.9.2.linux-amd64.tar.gz
 ```
 2. 解压包到`/usr/lib`目录下
 ```
 tar -C /usr/lib -xzf go1.9.2.linux-amd64.tar.gz
 ```
 3. 将go配置到环境变量
 ```
 	vim /etc/profile
	# 在末尾插入
	export GOROOT=/usr/lib/go
	export GOPATH=/usr/lib/gogs
	export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
 ```
 4. 下载Gogs
 ```
 wget https://dl.gogs.io/0.11.43/gogs_0.11.43_linux_amd64.tar.gz
 ```
 5. 解压包到`/usr/lib`目录下
 ```
 	tar -C /usr/lib -xzf gogs_0.11.43_linux_amd64.tar.gz
 ```
 6. 安装git(略)
 7. 安装mysql(略)，创建一个叫gogs的数据库
 ```
 	# 登录mysql
	mysql -u root -p
	# 设置数据库引擎
	SET GLOBAL default_storage_engine = 'InnoDB';
	# 创建数据库gogs表
	CREATE DATABASE gogs CHARACTER SET utf8 COLLATE utf8_bin;
 ```
 8. 启动Gogs
 ```
 cd /usr/lib/gogs
 nohup ./gogs web &
 ```
 9. 访问Gogs
 在浏览器中输入域名:3000，首次访问需要会跳转到install界面，填写一些信息，如数据库密码，名字、域名等。这些配置文件也可以在/usr/lib/gogs/custom/conf/app.ini中修改。记得要开启服务器的3000端口

### 二、 可能出现的问题及解决方案
 1. 出现 Access denied for user 'root'@'localhost'。解决方案：创建MySQL新用户`root_sql`，并赋予权限
 ```
 CREATE USER 'root_sql'@'localhost' IDENTIFIED BY 'yourpasswd';
 GRANT ALL PRIVILEGES ON *.* TO 'root_sql'@'localhost' WITH GRANT OPTION;
 FLUSH PRIVILEGES;
 ```
 2. 配置页配置失败，出现xx用户与当前登陆用户不一致的情况。解决方法是创建xx用户，并赋予xx用户对相应文件的读写权限。
 ```
 创建用户并设置密码
 useradd -d /data/git git
 passwd git
 >git
 # 赋予用户对相应文件读写的权限
 chown -R git /usr/lib/gogs
 chgrp -R git /usr/lib/gogs
 chmod a+r /usr/lib/gogs
 # 切换到git用户登录，并以git用户开启gogs服务
 su git
 cd gogs
 git>nohup ./gogs web &
 ```
 3. 邮箱配置失败，不能发送邮件。
 使用smtp.163.com:25作为host时，由于阿里云服务器默认封禁了端口号25，因此会导致邮件发送失败，官方建议将端口号25调整为465。即host为smtp.163.com:465