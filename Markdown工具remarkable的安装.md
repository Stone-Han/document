#Markdown-remarkable-Python的安装
在网上搜到markdown软件比较好用的是remarkable，然后到官网上搜索。好不容易才把安装包下载下来。
赶紧备份一下[百度云链接]( https://pan.baidu.com/s/1slbKccX) 。密码na4j。
直接双击deb包，然后点击安装。
安装过程中告诉我python3-pygments没有安装。
于是又到[python下载](https://www.python.org/downloads/source/) 安装。
先解压，然后进入目录。根据给出的提示安装。

		./configure
		make
		make test
		sudo make install
		
然后重新安装remarkable
安装好之后在dash里面点击remarkable的图标没有反应。在命令行里输入才弹出来。
