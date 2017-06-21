> **使用github给开源项目添加ssh后需要在本地将私钥添加到ssh-agent，通常网上给出的方法是在git bash界面中输入：**  

	ssh-add [私匙文件路径]  
    
> 然而，在windows下通常会报错：
  
***Could not open a connection to your authentication agent***

## 解决方法：

打开git Bash命令行,依次执行：
  
	1. exec ssh-agent bash
	2. eval ssh-agent -s  
	3. ssh-add "XXX\.ssh\id_rsa"  

引号中的路径就是你私钥文件的路径