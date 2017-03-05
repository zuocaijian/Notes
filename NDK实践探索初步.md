#Android NDK开发初步探索
>在了解Anroid安全、权限与Linux进程的安全、权限之间的关联关系的时候突发奇想：是否使用socket连接来作为http客户端，以之请求网络就不需要在AndroidManifest.xml中申请Internet访问权限。为验证该想，遂作如下试验：  

1. **试验结果**：NDK下无论是调用本地方法进行socket连接，还是调用本地方法读写SD卡上的文件，均需要在AndroidManifest.xml中作相应的权限请求配置；
2. **试验环境**：AndroidStudio2.2.3 + android-ndk-r13b + Gradle2.14.1
3. **比对环境**：Ubuntu14（win10虚拟机）+ codeblock13.12  

 
####测试步骤：
1. 在codeblock中编写可执行的socket实现的http客户端文件httpGet.cpp文件；
2. 在AndroidStudio中新建NDK项目：勾选`include C++ Support`，使用frtti和fexceptions；
3. 将httpGet.cpp代码移植到Android项目的cpp文件中，建立本地方法；
4. 对比，分别测试在AndroidManifext.xml文件中配置Internet访问权限和不配置的运行结果。

####核心代码：
1.**httpGet.cpp**  
	#include<stdio.h>  
	#include<unistd.h>  
	#include<sys/types.h>  
	#include<sys/socket.h>  
	#include<sys/stat.h>  
	#include<fcntl.h>  
	#include<string.h>  
	#include<arpa/inet.h>  
	#include<netdb.h>  
	#include<stdlib.h>  
	#include<netinet/in.h>  

	#define BUFSIZE 0xF000
	void geturl(char* url)
	{
    	int cfd;
    	struct sockaddr_in cadd;
    	struct hostent* pURL = NULL;
    	char myurl[BUFSIZE];
    	char* pHost = 0;
    	char host[BUFSIZE], GET[BUFSIZE];
    	char request[BUFSIZE];
    	static char text[BUFSIZE];
    	int i, j;

    	//分离主机中的主机地址和相对路径
    	memset(myurl, 0, BUFSIZE);
    	memset(host, 0, BUFSIZE);
    	strcpy(myurl, url);
    	for(pHost = myurl; *pHost != '/' &&  *pHost != '\0'; ++pHost);

    	//获取相对路径保存到GET中
    	memset(GET, 0, BUFSIZE);
    	if((int)(pHost - myurl) == strlen(myurl))
    	{
        	strcpy(GET, "/");//即url中没有给出相对路径，需要自己手动的在url尾部加上/
    	}
    	else
    	{
        	strcpy(GET, pHost); //地址段pHost到strlen(myurl)保存的是相对路径
    	}

    	//将主机信息保存到host中
    	//此处将它置0， 即它只想的内容里面已经分离出了相对路径，剩下的为host信息（从myurl到pHost地址段存放的是HOST）
    	*pHost = '\0';
    	strcpy(host, myurl);

    	//设置socket参数
    	if(-1 == (cfd = socket(AF_INET, SOCK_STREAM, 0)))
    	{
        	printf("create socket failed of client!\n");
	        exit(-1);
    	}
    	printf("create socket success of client!\n");

    	pURL = gethostbyname(host); //讲上面获得的主机信息通过域名解析函数获得域名信息
	
    	//设置IP地址结构
    	bzero(&cadd, sizeof(struct sockaddr_in));
    	cadd.sin_family = AF_INET;
    	cadd.sin_addr.s_addr = *((unsigned long*)pURL->h_addr_list[0]);
    	cadd.sin_port = htons(80);

    	//向WEB服务器发送URL信息
    	memset(request, 0, BUFSIZE);
    	strcat(request, "GET ");
    	strcat(request, GET);
    	strcat(request," HTTP/1.1\r\n"); //设置http请求行的信息

    	strcat(request, "Accept: text/html, image/jpeg, image/png, vedio/x-mng, text/*, image/*, */*\r\n");
    	strcat(request, "Accept-Language: zh-cn\r\n");
    	strcat(request, "Accept-Encoding: x-gzip, x-deflate, gzip, deflat\r\n");
    	strcat(request, "Accpet-charset: utf-8, utf-8; q=0.5, *; q=0.5\r\n");
    	strcat(request, "HOST: ");
    	strcat(request, host);
    	strcat(request, "\r\n");
    	//strcat(request, "Content-Type: text/html\r\n");
    	//strcat(request, "Content-Length: 1024\r\n");
    	strcat(request, "Connection: Keep-Alive\r\n");
    	strcat(request, "Cache-Control: no-cache\r\n\r\n");//设置http请求头

    	//链接服务器
    	int cc;
    	if(-1 == (cc = connect(cfd, (struct sockaddr*)&cadd, (socklen_t)sizeof(cadd))))
    	{
    	    printf("connect failed of client!\n");
    	    exit(1);
    	}
    	printf("connect success!\n");

    	//向服务器发送url请求的request、
    	int cs;
    	if(-1 == (cs = send(cfd, request, strlen(request), 0)))
    	{
    	    printf("send request failed!\n");
    	    exit(1);
    	}
    	printf("send reqeust success, and send length = %d\n", cs);

    	//将http请求信息写入文件hPage
    	int chfd;
    	if(-1 == (chfd = open("hPage", O_RDWR|O_CREAT|O_TRUNC, 0766)))
    	{
    	    printf("open or create hPage failed!\n");
    	    exit(1);
    	}
    	printf("open or create hPage success!\n");

    	int chf;
    	if(-1 == (chf = write(chfd, request, strlen(request))))
    	{
    	    printf("write to hPage failed!\n");
    	    exit(1);
    	}
    	else if(chf == 0)
    	{
    	    printf("the length of written to hPage is 0 byte!\n");
    	}
    	else
    	{
    	    printf("the length of written to hPage is %d bytes!\n", chf);
    	}

    	//客户端接受服务器的返回信息
    	memset(text, 0, BUFSIZE);
    	int cr;
    	if(-1 == (cr = recv(cfd, text, BUFSIZE, 0)))
    	{
    	    printf("receive failed!\n");
    	    exit(1);
    	}
    	else
    	{
    	    printf("receive success!\n");
    	    int cofd;
    	    for(;;)
    	    {
    	        //创建一个文件mPage用于保存网页的主要信息
    	        if(-1 == (cofd = open("mPage.html", O_RDWR|O_CREAT|O_TRUNC, 0766)))
    	        {
    	            perror("open or create mPage failed!\n");
    	            exit(1);
    	        }

    	        //将网页的主要内容text写入mPage中
    	        int cf;
    	        if(-1 == (cf = write(cofd, text, strlen(text))))
    	        {
    	            perror("write to mPage failed!\n");
    	            break;
    	        }
    	        else if(cf == 0)
    	        {
    	            printf("the length of written to mPage is 0 byte!\n");
    	            break;
    	        }
    	        else
    	        {
    	            printf("the length of written to mPage is %d bytes", cf);
    	            break;
    	        }
	
    	        //将text清空，方便接受其他网页信息
    	        memset(text, 0, BUFSIZE);
	
    	        //关闭文件描述符
    	        //close(chfd);
    	        close(cofd);
    	    }
    	}

    	//free(BUFSIZE);
    	close(cfd);
	}	
2.**native-lib.cpp(AndroidStudio jni)**  
  
	#include"httpGet.cpp"  
	#define OK  1  
	#define ERROR  0  
	...
	extern "C"
	jint
	Java_com_xxx_ndk_MainActivity_urlFromJNI(JNIEnv* env, jobject thiz){
		char address[] = "www.baidu.com";
		char* pAdd = address;
		return geturl(pAdd);
	}	

*说明：linux下的httpGet.cpp移植到AndroidStudio下的native-lib.cpp文件需要作一些调整：1.需要修改open()方法里文件的创建位置(/sdcard/xxx.xx); 2.为方便起见，本地方法调用返回值为int类型，对于exit()方法，需要替换为return OK或者return ERROR。*
***
编译.so文件  
1.编译环境：Ubuntu14 + gcc；  
2.编译步骤：  
>  
1. 首先， 编译mylib.c：  
$gcc -c -fPIC -o mylib.o mylib.c  
-c表示只编译(compile)，而不是链接。-o选项用于说明输出(output)文件名。gcc将生成一个目标(object)文件mylib.o。注意-fPIC选项。PIC指Position Independent Code。共享库要求有此选项，以便实现动态链接(dynamic linking)。
2. 生成共享库：  
$gcc -shared -o mylib.so mylib.o
库文件以lib开始。共享库文件以.so为后缀。-shared表示生成一个共享库。注意，由于该文件并不包含main()函数，因此，生词.so文件必须分两步，先编译，后链接。