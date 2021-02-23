## Ubuntu系统环境初始配置

#### 一、切换为国内包管理器镜像源

[ubuntu | 镜像站使用帮助 | 清华大学开源软件镜像站 | Tsinghua Open Source Mirror](https://mirror.tuna.tsinghua.edu.cn/help/ubuntu/)



#### 二、安装开发环境基础依赖库

```shell
sudo apt update // 更新软件源列表信息
sudo apt upgrade // 更新已安装软件
sudo apt install build-essential
sudo apt install manpages-dev
sudo apt install manpages-posix manpages-posix-dev
sudo apt install glibc-doc
sudo apt install gdb
sudo apt install vim
sudo apt install cmake
sudo apt install git
sudo apt install openssh-server

git config --global user.name "xxx"
git config --global user.email "xxx"
```



#### 三、设置中文输入法

设置 -> region & language -> manage installed languages -> install/remove languages -> 勾选 chinese simplified -> 点击apply等待安装语言包 -> 回退到语言设置界面添加输入选项 -> 选择Chinese，再选择 Chinese pinyin，或者在other里搜索。如果没有相关选项，则注销登陆再重新打开选择。



#### 四、vim简单配置

*主要配置文件为`/etc/vim/vimrc`以及用户目录下的`~/.vimrc`(没有可以新建)*

```shell
"额外配置
"显示行号
set number
"Tab键的宽度为4
set tabstop=4
"按C语言格式缩进
set cindent
"设置自动缩进(与上一行缩进相同)
set autoindent
"代码补全
set completeopt=preview,menu

"设置 ( 自动补全
inoremap ( ()<ESC>i
"设置 [ 自动补全
inoremap [ []<ESC>i
"设置 { 自动补全
inoremap { {}<ESC>i
"设置 < 自动补全
inoremap < <><ESC>i
"设置 ' 自动补全
inoremap ' ''<ESC>i
"设置 " 自动补全
inoremap " ""<ESC>i

"设置配色方案
"colorscheme peachpuff

```



#### 五、gcc编译相关

1. 部分函数需要启用实验性质的宏定义，在源码文件头部添加如下宏开启这些特性：

   ```c
   #define _DEFAULT_SOURCE
   ```

2. 