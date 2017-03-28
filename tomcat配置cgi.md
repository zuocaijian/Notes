# 环境说明
* 操作系统：win10家庭正式版
* tomcat版本：8.0.28
* VisualStudio版本：VS2015社区免费版  

# tomcat配置
* tomcat安装及jdk安装，确保tomcat服务器可以正常开启（本地访问 http://127.0.0.1:8080 能打开tomcat主页）
* 修改tomcat安装目录/conf/web.xml文件，找到CGI的Servlet配置，将注释打开：  
```
	<servlet>
		<servlet-name>cgi</servlet-name>  
		<servlet-class>org.apache.catalina.servlets.CGIServlet</servlet-class>
        <init-param>
          <param-name>debug</param-name>
          <param-value>0</param-value>
        </init-param>
        <init-param>
          <param-name>cgiPathPrefix</param-name>
          <param-value>WEB-INF/cgi</param-value>
        </init-param>
		<init-param>
          <param-name>executable</param-name>
          <param-value></param-value>
        </init-param>
         <load-on-startup>5</load-on-startup>
    </servlet>
```  
其中参数说明如下（部分参数未配置，可视情况添加）：
	1. cgiPathPrefix：设置cgi程序在应用中的访问位置，默认访问位置为：应用名称/WEB-INF/cgi；
	2. executable：CGI程序解析器，默认为perl，如果为空，可以是任何安装在**操作系统**环境变量的脚本解析器，或是C/C++程序；
	3. parameterEncoding：访问CGI Servlet的默认参数编码，默认为utf-8；
	4. passShellEnvironment：是否开启shell环境变量，默认为false；
	5. stderrTimeout：读取标准错误信息超时时长，默认为2000毫秒；
* 同样再web.xml中找到URL映射，将注释打开：  
```
	<servlet-mapping>
        <servlet-name>cgi</servlet-name>
        <url-pattern>/cgi-bin/*</url-pattern>
    </servlet-mapping>
```  
* 修改tomcat安装目录/conf/context.xml文件，在context 节点添加属性 privileged="true"  
```
	...
	<Context privileged="true">
    <!-- Default set of monitored resources. If one of these changes, the    -->
    <!-- web application will be reloaded.                                   -->
    <WatchedResource>WEB-INF/web.xml</WatchedResource>
    <WatchedResource>${catalina.base}/conf/web.xml</WatchedResource>
	...
```
* 修改后保存配置，重启tomcat服务器

# 用C语言编写第一个cgi程序Hello World
* 在VS中新建win32控制台项目，编写代码如下：  
```
	#include<stdio.h>

	int main(int argc, char* argv[])
	{
		printf("Content-type: text/plain;charset=us-ascii\n\n");
		printf("Hello world!\r\n");
		return 0;
	}
```  
* 编译成hello.exe文件，在tomcat服务器中web项目的WEB-INF文件夹下新建cgi文件夹，将hello.exe放置在cgi目录下，并改名未hello.cgi
* 在浏览器中访问 http://127.0.0.1:8080/webproject/cgi-bin/hello.cgi ，即可看到显示 Hello world! 。