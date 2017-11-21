
#GLPK在终端及kdevelop中的使用


##终端
首先把说明文档中的程序输入GLPK.cpp
然后
	gcc -c GLPK.cpp
	gcc GLPK.o -lglpk -lm

然后生成了一个a.out命令，运行之后报错:
	error while loading shared libraries: libglpk.so.0: cannot open shared object:NO such file or directory
在网上查了，这是因为共享库文件安装到了/usr/local/lib下面[参考网站](http://blog.csdn.net/sahusoft/article/details/7388617) 

那么要把新共享库目录加入到共享库配置文件/etc/ld.so.conf中。

	cat /ect/ld.so.conf
	include ld.so.conf.d/*.conf
	echo "/usr/local/lib">>/etc.ld.so.conf
	bash: /etc.ld.so.conf: 权限不够
	
切换到root用户下设置，因为是第一次所以要设置root密码

	stone@stone-Lenovo:/$ sudo passwd root
	输入新的 UNIX 密码： 
	重新输入新的 UNIX 密码： 
	passwd：已成功更新密码
	stone@stone-Lenovo:/$ su
	root@stone-Lenovo:/# echo "/usr/local/lib">> /etc.ld.so.conf
	root@stone-Lenovo:/# cat /etc.ld.so.conf
	/usr/local/lib
	root@stone-Lenovo:/# ldconfig
	root@stone-Lenovo:/# exit
	
然后再执行`./a.out`就可以看到结果

	*     0: obj =   0.000000000e+00  infeas =  0.000e+00 (0)
	*     2: obj =   7.333333333e+02  infeas =  0.000e+00 (0)
	OPTIMAL SOLUTION FOUND

z = 733.333; x1 = 33.3333; x2 = 66.6667; x3 = 0


###kdevelop
在kdevelop里面建了一个工程，输入从网上找的一个程序，但是编译报了很多错。
报错类似于

	/home/stone/projects/d/main.cpp:11: undefined reference to `glp_create_prob'
	
然后我的思路是想把命令行敲的命令配置到Kdevelop下，感觉应该是路径的问题。
我尝试在CMakefilelists.txt中添加了一行

	add_librariy(/usr/local/lib/lglpk.so)
没有错误了，但是仍然无法运行。在运行配置里输入-lglpk -lm都不行。
然后在网上搜了一堆make 和cmake的语法，看了半天都没有讲怎么加参数的。
不过有几个教程特别好。
[跟我一起写makefile](http://bbs.chinaunix.net/thread-408225-1-1.html)
 [在Linux下使用cmake构建应用程序](https://www.ibm.com/developerworks/cn/linux/l-cn-cmake/) 

根据这里面的例子，我看到了添加库和链接库的语法

	INCLUDE_DIRECTORIES(${LIBDB_CXX_INCLUDE_DIR})
  	MESSAGE( ${LIBDB_CXX_LIBRARIES} )
   	TARGET_LINK_LIBRARIES(main ${LIBDB_CXX_LIBRARIES}18 )
   
   根据网上各种教程尝试写的都不管用，后来我看了一下在windows下的配置，发现都是配置好Include路径和Lib路径就可以了。然后我梳理了一下生成的过程。
   
   	gcc -c GLPK.cpp
	gcc GLPK.o -lglpk -lm
	
第一句话就是生成.o文件，第二句话仍然不知道-lglpk -lm是干什么用的，但是应该是把链接库链入.o文件然后生成可执行文件。
然后我在工程里写入

	include_directories(glpk.h /usr/local/include)
	target_link_libraries(d /usr/local/lib/libglpk.so)
	
然后就编译成功，执行结果也出来了。
好像第一句也不用加。不过注意第二句话要写在add_executable 后面。


下一步在需要的地方调用就可以了。
