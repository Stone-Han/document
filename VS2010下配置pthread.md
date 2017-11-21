---

title:windows下配置pthread
date: 2016-08-22 8:25:09
tags: [教程,windows]
categories: 教程

---
因为最近用到线程池，看了很久的C++11的线程并发，但是还是觉得自己不能快速掌握。所以还是用熟悉的pthread吧。有空再细致地学习一下C++11的新特性。


<!-- more -->

本机环境

- VS2010
- Windows10

参考[http://blog.csdn.net/npuweiwei/article/details/8666373](http://blog.csdn.net/npuweiwei/article/details/8666373 "地址")


# 下载pthread #

地址[http://sourceware.org/pthreads-win32/](http://sourceware.org/pthreads-win32/)

我用的版本是 pthreads-w32-2-9-1-release.zip

解压后有三个文件夹

- Pre-built.2
- pthreads.2
- QueueUserAPCEx


我们要用的是第一个文件夹。

# 拷贝文件 #

下一步是把Pre-built.2对应的lib,include文件分别拷贝到你安装VS的路径下。

比如我的安装路径是

- C:\Program Files (x86)\Microsoft Visual Studio 10.0\VC\lib
- C:\Program Files (x86)\Microsoft Visual Studio 10.0\VC\include

然后把dll文件对应的x86和x64文件分别拷贝到

- C:\Windows\System32
- C:\Windows\SysWOW64

# 在VS2010中配置 #

新建一个项目，比如叫Pthread_Test.

- 在Project ->Pthread_Test Properties -> Configuration Properties-> C/C++ -> General ->Additional Include Directories 中增加头文件路径。

- 在Project ->Pthread_Test Properties -> Configuration Properties-> Linker -> General-> Additional Library Directories 中增加库文件路径。我用的是x86库。

- 在Project ->Pthread_Test Properties -> Configuration Properties-> Linker -> Input ->Additional Dependencies中增加所依赖的库文件。这里我们使用的IDE是VS2010，所以我们使用pthreadVSE2.lib。各个库的不同之处在pthreads.2下的README文档中有介绍。

# 测试代码 #

下面是测试代码。

    #include "stdafx.h"
	#include <stdio.h>
	#include <pthread.h>
	#include <assert.h>
	
	void* Function_t(void* Param);
	int _tmain(int argc, _TCHAR* argv[])
	{
		pthread_t pid;
		//pthread_tpid;
		pthread_attr_t attr;
		pthread_attr_init(&attr);
		pthread_attr_setscope(&attr,PTHREAD_SCOPE_PROCESS);
		pthread_attr_setdetachstate(&attr,PTHREAD_CREATE_DETACHED);
		pthread_create(&pid, &attr,Function_t, NULL);
		printf("====\n");
		getchar();
		pthread_attr_destroy(&attr);
		return 0;
	}
	void* Function_t(void* Param)
	{
		printf("Thread Starts.\n");
		pthread_t myid = pthread_self();
		printf("Thread ID=%d ", myid);
		return NULL;
	}

