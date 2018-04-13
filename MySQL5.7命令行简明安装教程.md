### windows下安装mysql5.7
`由于MySql5.7对安全性能的增强，导致其安装步骤与MySql5.6有些变化` 

* 在[官网下载MySql5.7](http://dev.mysql.com/downloads/mysql/),然后解压；
* 将 install-dir\bin添加到环境变量；
* 在 install-dir下复制my-default.ini，修改名称为my.ini；
* 打开my.ini，并在末尾添加  
> [client]  
> default-character-set=utf8  
> [mysqld]  
> basedir=install-dir  
> datadir=install-dir\data  
> port=3306  
> default-character-set=utf8  

* 然后将my.ini放在bin目录下；
* 管理员身份进入解压目录 install-dir\bin；
* 安装/注册MySql服务。运行命令：mysqld --install；
* **初始化data目录**。运行命令：mysqld --initialize-insecure。生成无密码的root用户；
* 启动mysql服务。运行命令：net start mysql；
* 开始使用mysql。运行命令：mysql -u root -p;

### ubuntu下安装mysql最新版
- 更新软件源
	`sudo apt update; sudo apt upgrade`  
- 安装mysql  
	`sudo apt install mysql-server; sudo apt install mysql-client`

### ubuntu下修改mysql5.7 数据存储目录datadir
> 由于数据库数据较多，很多时候我们需要更改数据存放路径
> 环境：ubuntu16.04 mysql5.7.12
> 参考[ubuntu修改mysql 5.7 数据存储目录datadir](http://blog.csdn.net/qq_33571718/article/details/71425623)

- 停止mysql服务  
	`service mysql stop`
- 复制原有数据库文件到新的存储目录，以避免再次初始化，并且数据还在  
	`cp -av /var/lib/mysql/ /data/`  
- 删除日志文件（否则报错）  
	`rm /data/mysql/ib_logfile0 /data/mysql/ib_logfile1`  
- 修改mysqld.cnf文件  
	`vim /etc/mysql/mysql.conf.d/mysqld.cnf`  
	注释原来的datadir=/var/lib/mysql，复制改为新的目录，这里是    
	`datadir=/data/mysql`  
- 修改apparmor的配置文件usr.sbin.mysqld  
	`vim /etc/apparmor.d/usr.sbin.mysqld`  
	将  
	`/var/lib/mysql/ r,`  
	`/var/lib/mysql/** rwk,`  
	改为  
	`/data/mysql/ r,`  
	`/data/mysql/** rwk,`    
- 重新加载apparmor配置，然后重新启动  
	`service apparmor reload; service apparmor restart`    
- 重启mysql服务  
	`service mysql restart`
