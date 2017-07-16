1. check项目到本地  

    	svn co svn://repository-url  

2. 生成日志  

		svn log -v --xml F:\project > F:\project\svn.log

3. 生成详细信息  

		java -jar statsvn.jar F:\project\svn.log F:\project -include **/*.java:**/*.xml -exclude **/*.js

4. 打开project文件夹下的生成的index.html页面即可看到详细信息

> 参考[用StatSVN统计SVN服务器项目的代码量](http://www.cnblogs.com/seer/p/5735501.html)