1. 在windows下配置wsl：略
2. 在windows下安装VSCode：略
3. 在wsl中安装Makefile工具：略
4. 在VSCode中安装C/C++开发基本插件：C/C++(Mircosoft);
5. 在VSCode中安装插件：Easy C++ projects(Alejandro Charte Luque);
6. 使用Easy C++ projects插件新建一个C++项目：`Easy Cpp/C++:create new C++ project`;
7. 在Debug界面中点击`Add Configuration`，在.vscode目录下生成launch.json文件，配置如下：
	```
	{
	    // Use IntelliSense to learn about possible attributes.
	    // Hover to view descriptions of existing attributes.
	    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
	    "version": "0.2.0",
	    "configurations": [
	        {
	            "name": "(gdb) Bash on Windows Launch",
	            "type": "cppdbg",
	            "request": "launch",
	            "program": "/mnt/e/Code/VSCode/webbench/bin/main",
	            "args": [],
	            "stopAtEntry": false,
	            "cwd": "/mnt/e/Code/VSCode/webbench",
	            "environment": [],
	            "externalConsole": true,
	            "preLaunchTask": "Build C++ project",
	            "pipeTransport": {
	                "debuggerPath": "/usr/bin/gdb",
	                "pipeProgram": "${env:windir}\\system32\\bash.exe",
	                "pipeArgs": ["-c"],
	                "pipeCwd": ""
	            },
	            "setupCommands": [
	                {
	                    "description": "Enable pretty-printing for gdb",
	                    "text": "-enable-pretty-printing",
	                    "ignoreFailures": true
	                }
	            ],
	            "sourceFileMap": {
	                "/mnt/e": "d:\\"
	            }
	        }
	    ]
	}
	```
	
8. 使用C/C++插件配置环境：`C/C++:Edit Configurations (JSON)`，在.vscode目录下生成c_cpp_properties.json文件，配置如下：
   ```
	{
	    "configurations": [
	        {
	            "name": "Win32",
	            "includePath": [
	                "${workspaceFolder}/**"
	            ],
	            "defines": [
	                "_DEBUG",
	                "UNICODE",
	                "_UNICODE"
	            ],
	            "windowsSdkVersion": "10.0.14393.0",
	            "cStandard": "c11",
	            "cppStandard": "c++17",
	            "intelliSenseMode": "gcc-x64"
	        }
	    ],
	    "version": 4
	}
	```