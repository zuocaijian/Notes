> Fiddler是一款强大的抓包工具，可对HTTP和HTTPS以及FTP协议进行抓包。在Android逆向以及App分析中，经常需要对其请求接口进行分析，使用Fiddler即可完成。不过在使用之前，需要对Fiddler以及手机进行简单的设置。

# Fiddler设置
1. Tools->Fiddler options，点选Connections选项卡，配置Fiddler 监听端口为8888，同时勾选 **Allow remote computers to connect**
2. Tools->Fiddler options，点选HTTPS选项卡，勾选**Decrypt HTTPS traffic**，同时勾选**Ignore server certificate errors(unsafe)**

# 手机端设置
1. 设置代理：长按当前连接的wifi，设置代理为手动代理，代理服务器主机名为安装Fiddler的电脑ip，代理服务器端口为8888，模式为DHCP；
2. 在手机浏览器上打开安装Fiddler的电脑ip地址以及Fiddler的端口号8888，点击安装Fiddler证书

经过以上简单设置，即可使用Fiddler对手机的网络请求进行抓包了。