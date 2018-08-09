## Ubuntu 16.04 64bit 安装Redis

### 第一种，源码安装(不推荐)
一、 进入[Redis中文官网](http://www.redis.cn)的[下载页面](http://www.redis.cn/download.html)。  
二、 下载最新稳定版本的Redis源码，当前稳定版本为4.0.10；
```
wget http://download.redis.io/releases/redis-4.0.10.tar.gz
```
三、 解压、编译源码
```
tar xzvf redis-4.0.10.tar.gz
cd redis-4.0.10
make
```
编译完成后将在src目录下生成两个redis-server服务程序和redis-cli客户端程序。
  
四、 启动Redis服务
```
src/redis-server
```
五、通过redis-cli客户端程序使用redis
```
src/redis-cli
redis> set foo bar
OK
redis> get foo
"bar"
...
```

### 第二种，使用包管理器安装(推荐)
一、 安装Redis
```
sudo apt-get install redis-server
```
二、 检查Redis安装是否成功以及Redis服务器状态
 1. 检查Redis服务器系统进程
 ```
 ps aux | grep redis
 ```
 2. 通过默认端口号检查Redis服务器状态
 ```
 netstat -ntl | grep 6379
 ```
 3. 通过启动命令检查Redis服务器状态
 ```
 sudo /etc/init.d/redis-server status
 ```

四、 通过命令行客户端访问Redis
```
redis-cli

# 命令行帮助
127.0.0.1:6379> help
...

# 查看所有的key列表
127.0.0.1:6379> keys *
...
```
五、 修改Redis的配置
 1. 启用Redis的访问账号
 	* 默认情况下，访问Redis服务器是不需要密码的，为了增加安全性我们需要设置Redis服务器的访问密码。访问密码设置为findpet2018。  
 	* 打开Redis服务器的配置文件/etc/redis/redis.conf
 	```
	sudo vim /etc/redis/redis.conf
	# 取消注释 requirepass
	requirepass findpet2018
	```
 2. 让Redis服务器可以被远程访问
	* 默认情况下，Redis服务器不允许远程访问，只允许本机访问，所以我们需要设置打开远程访问的功能。
	* 打开Redis服务器的配置文件/etc/redis/redis.conf
	```
	sudo vim /etc/redis/redis.conf
	# 注释bind
	# bind 127.0.0.1
	```
	* 修改后重启Redis服务器
	```
	sudo /etc/init.d/redis-server restart
	```
	* 使用密码登陆Redis服务器
	```
	redis-cli -a findpet2018
	```
	* 检查Redis的网络监听端口
	```
	netstat -ntl | grep 6379
	```
	可以看到网络监听本地地址由之前的 127.0.0.1：6379 变成 0.0.0.0：6379，表示Redis已经允许远程登陆访问。
	* 远程访问Redis服务器
	```
	redis-cli -a findpet2018 -h 192.168.xx.xx
	```
	*如果远程访问阿里云上的redis，请先确认redis端口6379已经开放*

六、 基本的Redis客户端操作命令
 1. 增加一条记录key1
 2. 删除记录
 3. ...