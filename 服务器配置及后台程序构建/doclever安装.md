## Ubuntu 16.04 64bit 安装DOClever

### 一、 安装 Node.js
 1. 使用命令安装node.js
 ```
 sudo apt-get install nodejs-legacy
 # 查看nodejs版本
 node -v
 ```
 2. 升级node版本
 	* 首先下载npm包管理器
 	```
	sudo apt-get install npm
	```
 	* 使用npm安装n工具
 	```
	sudo npm install -g n
	```
 	* 使用n工具升级node版本至8.11.1
 	```
	sudo n 8.11.1
	# 查看nodejs版本
	node -v
	```

### 二、安装MongoDB
 1. 使用命令安装mongodb
 ```
 sudo apt-get install mongodb
 ```
 2. 查看mongodb版本
 ```
 mongo -version
 ```
 3. 查看mongodb是否启动
 ```
 pgrep mongo -l
 ```
 4. 进入mongodb命令模式。默认连接的是test数据库
 ```
 mongo shell
 ```
 5. 创建数据库doclever：use doclever，并且增加一条数据
 ```
 use doclever
 db.items.insert({"key":"value"})
 # 查看当前所有数据库
 show dbs
 # 查看当前所在数据库
 db
 ```
 6. 远程连接mongodb的doclever数据库
 ```
 mongo 127.0.0.1:27017/doclever
 ```
 7. mongodb常用操作命令
 ```
 # 显示数据库列表
 show dbs
 # 显示当前数据库中的集合（类似关系数据库中的表table）
 show collections
 # 显示所有用户
 show users
 # 切换当前数据库至yourDB
 use yourDB
 # 显示数据库操作命令
 db.help()
 # 显示集合操作命令，yourCollection是集名
 db.yourCollection.help()
 # MongoDB没有创建数据库的命令，如果想创建一个“School”的数据库，先使用 use School 命令，然后做一些操作（如：创建聚集集合 db.createCollection('teacher')）,这样就创建一个名叫“School”的数据库了。 
 ```

### 三、 安装DOClever
 1. 使用git下载DOClever源码
 ```
 git clone https://github.com/sx1989827/DOClever.git
 ```
 *直接clone整个源码仓库会非常慢，因此建议上github，下载zip包然后解压。 wget https://codeload.github.com/sx1989827/DOClever/zip/master*
 2. 进入DOClever目录，运行DOClever
 ```
 cd DOClever
 node Server/bin/www
 ```
 3. 首次运行需要进行连接数据库，设定上传文件路径和端口号等一些初始化操作
 ```
 ...
 请输入mongodb数据库地址：mongodb://localhost:27017/doclever
 ...
 请输入DOClever上传文件路径：data/files
 ...
 请输入端口号：23333
 ...
 ```
 *注意：由于文件访问权限的原因，初始化设定可能会因为没有文件读写执行权限而访问失败。为简单起见，更改整个DOClever目录的权限 sudo chmod 777 DOClever*
 4. 浏览器中输入 `http://127.0.0.1:23333` 进入登陆页面，使用默认管理员账号密码 `DOClever` 进入总后台管理页面
 5. 安装forever
 ```
 sudo npm install forever -g
 ```
 6. 使用forever启动进程
 ```
 forever start Server/bin/www
 ```