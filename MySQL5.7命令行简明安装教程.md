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