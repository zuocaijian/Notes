## Ubuntu 16.04 64bit 安装GitLab

> 安装GitLab详细教程可以参考[官网安装教程](https://about.gitlab.com/installation/)

### 一、 安装
 1. 安装gitlab用到的依赖
 ```
 sudo apt-get update
 sudo apt-get install -y curl openssh-server ca-certificates
 ```
 2. 下载gitlab包并安装
 ```
 curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash
 sudo apt-get install gitlab-ce
 ```
 *这个下载的过程可能较慢，具体视网速而定。因此建议在下载安装前使用vpn加速下载安装，或者按照官网链接使用国内镜像安装*
 3. 首次使用启用GitLab配置
 ```
 sudo gitlab-ctl reconfigure
 ```
 等待一段时间后gitlab配置成功。
 4. 检测GitLab是否安装成功
 ```
 sudo gitlab-ctl status
 ```
 5. 登陆gitlab
 	* 在浏览器中输入 http://localhost
 	* 首次登录gitlab会重定向到密码设置界面，设置密码然后使用root账户登录

### 二、 配置
 1. 配置端口号及超时，避免出现502错误
	* 使用gitlab内置nginx，修改nginx默认端口，从80变为92
	```
	vim /etc/gitlab/gitlab.rb
	nginx['listen_port'] = 92
	```
	```
	vim /var/opt/gitlab/nginx/conf/gitlab-http.conf
	listen *:92;
	```
	然后重启gitlab服务 `gitlab-ctl restart`
	* 使用gitlab内置nginx，修改unicorn的默认端口，从8080改为9290，即nginx监听的rails端口
	```
	vim /etc/gitlab/gitlab.rb
	unicorn['port'] = 9290
	gitlab_workhorse['auth_backend'] = "http://localhost:9290"
	```
	```
	vim /var/opt/gitlab/gitlab-rails/etc/unicorn
	listen "127.0.0.1:9290", :tcp_nopush => true
	```
	* 调整timeout时常，默认60秒。建议设置成120s~300s
	```
	unicorn['worker_timeout'] = 120
	gitlab_rails['webhook_timeout'] = 60
	```
 2. 配置smtp邮件系统
	* 修改配置文件邮件配置相关选项
	```
	vim /etc/gitlab/gitlab.rb
	# 修改smtp配置选项
	gitlab_rails['smtp_enable'] = true
	gitlab_rails['smtp_address'] = "smtp.163.com"
	gitlab_rails['smtp_port'] = 25 
	gitlab_rails['smtp_user_name'] = "yourusername@163.com"
	gitlab_rails['smtp_password'] = "yourpassword"
	gitlab_rails['smtp_domain'] = "163.com"
	gitlab_rails['smtp_authentication'] = :login
	gitlab_rails['smtp_enable_starttls_auto'] = true
	```
	# 修改gitlab配置的发信人
	gitlab_rails['gitlab_email_from'] = "yourusername@163.com"
	user["git_user_email"] = "yourusername@163.com"
	* 更新配置，重启gitlab
	```
	gitlab-ctl reconfigure
	gitlab-ctl restart	
	```

### 三、完全卸载gitlab
 1. 停止gitlab
 ```
 gitlab-ctl stop
 ```
 2. 卸载gitlab
 ```
 dpkg -P gitlab-ce
 ```
 3. 查看gitlab进程
 ```
 ps aux | grep gitlab
 ```
 4. 杀掉第一个进程(即后面有很多....的进程)
 ```
 kill -9 18977
 ```
 杀掉后，再重复第三步，确认还有没有gitlab进程
 5. 删除所有包含gitlab的文件及目录
 ```
 find / -name "*gitlab*" | xargs rm -rf
 ```

### 四、 可能出现的问题及解决方案
> gitlab开启的服务进程非常多，内存消耗极高。因此如果内存不够，则经常会报502错误，可以通过增加虚拟内存来缓解

 ```
 # 新建2G文件
 sudo dd if=/dev/zero of=swapfile bs=1024 count2048k
 # 设置swap文件
 sudo mkswap /swapfile
 # 激活swap文件
 sudo swapon /swapfile
 # 添加开机自动启用swap文件
 sudo vim /etc/fstab
 # 末尾增加一行
 /swapfile none swap defaults 0 0
 # 关闭swap文件
 swapoff swapfile
 # 删除swap文件
 rm swapfile
 ```

> 使用smtp.163.com时，如以25作为端口号进行通信，则可能会导致邮件发送系统失败。原因是由于阿里云服务器默认封禁了端口号25官方建议将端口号25调整为465。