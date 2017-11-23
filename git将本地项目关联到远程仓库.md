1. 初始化本地仓库。在本地项目目录下执行：git init；
2. 更新本地仓库状态。在本地项目目录下执行:git add . 和 git commit -m ""；
3. 创建远程仓库。(省略)；
4. 关联远程仓库。在本地执行：git remote add origin master；
	> origin 即为远程仓库的别名，可更改。master 即为远程仓库的分支。
5. 合并本地仓库和远程仓库分支。在本地执行：git pull origin master --allow-unrelated-histories；
	> 如果第一次直接执行：git pull origin master 命令会提示拉取失败'refusing to merge unrelated histories', 所以需要添加选项'--allow-unrelated-histories'，将本地分支与远程仓库分支合并。
6. 提交代码到远程仓库。在本地执行：git push -u origin master;
	> -u 选项指示本地分支默认关联的远程仓库及分支。以后拉取代码和提交代码到远程仓库只需执行'git pull'和'git push'即可。

> * 删除关联远程仓库：git remote remove origin
> * 查看远程仓库地址：git remote -v
> * 查看当前分支默认关联的远程分支：git branch -vv