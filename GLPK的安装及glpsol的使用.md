#GLPK的安装及glpsol的使用 
##安装
首先下载GLPK的安装包[官网地址](http://mirrors.ustc.edu.cn/gnu/glpk/) 
然后按照给出的说明
1. 首先解压到一个文件夹内，然后运行./configure,这个命令会生成MakeFile和config.status，以后可以运行来重新设置。
2. 然后运行make，会生成
- libglpk.a 存放所有GLPK routines的代码
- glpsol 独立的LP/MIP程序

	
	./configure
	make
	make check
	sudo make install	//make install
	
后面在编译程序的时候发现/usr/local/include下面并没有glpk.h，然后又回来看了一遍安装程序。在安装的时候已经报错
	make[2]: 没有什么可以做的为 `install-exec-am'。
	test -z "/usr/local/include" || /bin/mkdir -p "/usr/local/include"
	 /usr/bin/install -c -m 644 'glpk.h' '/usr/local/include/glpk.h'
	/usr/bin/install: 无法创建普通文件"/usr/local/include/glpk.h": 权限不够
	make[2]: *** [install-includeHEADERS] 错误 1
	make[2]:正在离开目录 `/home/stone/program/glpk-4.35/include'
	make[1]: *** [install-am] 错误 2
	make[1]:正在离开目录 `/home/stone/program/glpk-4.35/include'
	make: *** [install-recursive] 错误 1
	
然后用了sudo make install


---
##使用
关于使用有一个很好的例子，参考[IBM](https://www.ibm.com/developerworks/cn/linux/l-glpk1/) 

>Giapetto 的 Woodcarving 公司
>
>这个问题引自于 Operations Research：
>
>Giapetto 的 Woodcarving 公司生产两种木头制作的玩具：士兵和火车。一个士兵的销售价格为 27 美元，需要耗费价值 10 美元的原料。制造每个士兵需要耗费 Giapetto 的可变人力成本和间接成本一共 14 美元。一辆火车的销售价格为 21 美元，需要耗费价值 9 美元的原料。制造每辆火车需要耗费 Giapetto 的可变人力成本和间接成本一共 10 美元。这家木头士兵和火车的制造商需要两类熟练工人：木工和修整工。一个士兵需要 2 小时的修整工劳动和 1 小时的木工劳动。一辆火车需要 1 小时的修整工劳动和 1 小时的木工劳动。每周 Giapetto 可以获得所有必需的原料，但是只能提供 100 个修整工时和 80 个木工工时。市场对于火车的需求是无限的，但是每周最多可以销售 40 个士兵。Giapetto 希望能够使每周的收益（收入 - 成本）最大化。

>
>下面我们对这个问题的重要信息和假设小结一下：
>
>    有两种木制玩具：士兵和火车。
>    一个士兵的销售价格为 27 美元，需要耗费价值 10 美元的原料，另外需要耗费可变人力成本和间接成本一共 14 美元。
>    一辆火车的销售价格为 21 美元，需要耗费价值 9 美元的原料，另外需要耗费可变人力成本和间接成本一共 10 美元。
>    一个士兵需要 2 小时的修整工劳动和 1 小时的木工劳动。
>    一辆火车需要 1 小时的修整工劳动和 1 小时的木工劳动。
>    每周最多可以获得 100 个修整工时和 80 个木工工时。
>    每周对于火车的需求是无限的，但是最多可以销售 40 个士兵。 
>
>我们的目标是确定每周生产士兵和火车的数量，从而可以使每周的收益最大化。 
---
我们把

    x1：每周生产的士兵数量
    x2：每周生产的火车数量 
    
那么我们的目标就是

$$z = (27-10-14)x_1+(21-9-10)x_2 = 3 x_1 + 2 x_2$$

约束条件：

$$2x_1+x_2<=100   $$
$$x_1+x_2<=80$$
$$x_1<=40$$
$$x_1>=0,x_2>=0$$


分析完成之后就可以写程序了。我们把程序卸载.mod文件里面，然后执行程序，输出的分析结果在.sol文件里面。

---

Giapetto 问题中的数学公式需要使用 GNU MathProg 语言进行编写。需要声明的关键内容有：

- 决策变量
- 目标函数
- 约束
- 问题数据集 

**下面是.mod文件的代码**

	 1  #
	 2  # Giapetto's problem
	 3  #
	 4  # This finds the optimal solution for maximizing Giapetto's profit
	 5  #
	 6
	 7  /* Decision variables */
	 8  var x1 >=0;  /* soldier */
	 9  var x2 >=0;  /* train */
	10
	11  /* Objective function */
	12  maximize z: 3*x1 + 2*x2;
	13
	14  /* Constraints */
	15  s.t. Finishing : 2*x1 + x2 <= 100;
	16  s.t. Carpentry : x1 + x2 <= 80;
	17  s.t. Demand    : x1 <= 40;
	18
	19  end;


第1-5行是注释。

第一个 MathProg 步骤是声明决策变量。第 8 行和第 9 行声明了 x1 和 x2。决策变量的声明以关键字 var 开头。为了简化符号约束（检查不等式 9），MathProg 允许在决策变量声明中使用 >= 0 约束，正如我们在第 8 行和第 9 行中看到的一样。GNU MathProg 中的每条语句都必须以分号（;）结束。记住 x1 表示 soldier 的数量，x2 表示 train 的数量。这些变量应该分别称为 soldiers 和 trains，但是这可能会把读者中的数学界人士搞得混乱不堪。 

第 12 行声明了目标函数。线性问题可以是最大化问题，也可以是最小化问题。记住，Giapetto 的数学模型是一个最大化问题，因此我们应该使用 maximize 作为关键字，而不是相反的 minimize 关键字。目标函数称为 z，它等于 3x1 + 2x2。请注意：
- 冒号（:）字符分隔开了目标函数及其定义。
- 星号（*）字符表示乘法，类似地，加号（+）、减号（-）和斜线（/）字符分别表示加法、减法和除法。 

第 15、16、17 行定义了约束。尽管 s.t. 对于声明一个约束来说并不需要在行首，但是这样可以提高代码的可读性。 

---

然后我们在终端中

	glpsol -m giapetto.mod -o giapetto.sol

会生成一个.sol文件

	Problem:    giapetto
	Rows:       4
	Columns:    2
	Non-zeros:  7
	Status:     OPTIMAL
	Objective:  z = 180 (MAXimum)

	   No.   Row name   St   Activity     Lower bound   Upper bound    Marginal
	------ ------------ -- ------------- ------------- ------------- -------------
	     1 z            B            180
	     2 Finishing    NU           100                         100             1
	     3 Carpentry    NU            80                          80             1
	     4 Demand       B             20                          40

	   No. Column name  St   Activity     Lower bound   Upper bound    Marginal
	------ ------------ -- ------------- ------------- ------------- -------------
	     1 x1           B             20             0
	     2 x2           B             60             0

	Karush-Kuhn-Tucker optimality conditions:

	KKT.PE: max.abs.err. = 0.00e+00 on row 0
		max.rel.err. = 0.00e+00 on row 0
		High quality

	KKT.PB: max.abs.err. = 0.00e+00 on row 0
		max.rel.err. = 0.00e+00 on row 0
		High quality

	KKT.DE: max.abs.err. = 0.00e+00 on column 0
		max.rel.err. = 0.00e+00 on column 0
		High quality

	KKT.DB: max.abs.err. = 0.00e+00 on row 0
		max.rel.err. = 0.00e+00 on row 0
		High quality

	End of output


解答被划分成 4 部分：

- 有关问题和目标函数优化值的信息
- 有关目标函数状态和约束的确切信息
- 有关决策变量的优化值的确切信息
- 有关优化条件的信息（如果有的话） 

对于这个特定的问题来说，我们可以看到解决方案是 OPTIMAL 的，Giapetto 每周最大收益是 180 美元。

Finishing 约束的状态是 NU（St 列）。NU 表示这个约束的上界有一个非基本变量。如果我们了解一些基本的运筹学理论，就可以构建 simplex 场景并将之提取出来。如果不了解运筹学理论，下面是一个实际的解释。

不论何时约束达到自己的上界或下界时，我们就称之为是一个边界约束（bounded constraint）。边界约束阻碍了目标函数达到更为优化的值。例如我们可以认为它是一个音量调节旋钮，现在它已经无法再进行调节了。当这种情况发生时，glpsol 就会将约束状态显示为 NU 或 NL（分别对应上界和下界的情况），它还会显示边界（marginal）值，也称为假定价格（shadow price）。边界值是约束如果可以放松一个单位（如果音量调节旋钮可以再调节一点点）目标函数可以改进的值。注意这种改进依赖于我们的目标是要对目标函数进行最大化还是最小化。举例来说， Giapetto 问题所寻求的就是最大化，边界值为 1 就表示如果我们可以增加一个小时的修整工时，目标函数就可以增大 1（我们知道这是要多 一个小时，而不是少一个小时，这是因为修整工时约束是一个上界）。

木工和士兵的需求约束是类似的。对于木工约束来说，注意它也有一个上界。因此，这个约束中放松一个单位（增加一个小时）可以使目标函数的优化值增加边界值 1，成为 181。

然而，士兵需求是没有边界的，因此其状态是 B，这个约束中的放松不会改变目标函数的优化值。

一次只尝试放松每个边界约束一个单位，解决修改后的问题，看一下目标函数的优化值发生了什么变化。还要检查修改无边界约束的值不会对解答造成任何变化，这正是我们期望的。

最后，glpsol 的报告给出了决策变量的值：x1 = 20 和 x2 = 60。这就告诉 Giapetto 它应该生产 20 个士兵和 60 辆火车才能实现每周收益的最大化。

Giapetto 的问题很小。我们可能会纳闷，在有更多决策变量和约束的问题中，我们只能分别逐一声明每个变量和每个约束吗？如果希望调节问题中的数据（例如士兵的销售价格）应该怎样做呢？我们只能逐一修改这些值出现的地方吗？下一节将讨论这个问题。 

---

在 Giapetto 问题中使用模型和数据段

MathProg 模型通常都有一个模型段和一个数据段，有时可以在两个不同的文件中。这样 glpsol 就可以简单地使用不同的数据集来解析某个模型，从而找到对这些新数据应该采用哪种解决方案。下面的列表以更优雅的方式说明了 Giapetto 的问题： 

	1      #
	 2      # Giapetto's problem
	 3      #
	 4      # This finds the optimal solution for maximizing Giapetto's profit
	 5      #
	 6
	 7      /* Set of toys */
	 8      set TOY;
	 9
	10      /* Parameters */
	11      param Finishing_hours {i in TOY};
	12      param Carpentry_hours {i in TOY};
	13      param Demand_toys     {i in TOY};
	14      param Profit_toys     {i in TOY};
	15
	16      /* Decision variables */
	17      var x {i in TOY} >=0;
	18
	19      /* Objective function */
	20      maximize z: sum{i in TOY} Profit_toys[i]*x[i];
	21
	22      /* Constraints */
	23      s.t. Fin_hours : sum{i in TOY} Finishing_hours[i]*x[i] <= 100;
	24      s.t. Carp_hours : sum{i in TOY} Carpentry_hours[i]*x[i] <= 80;
	25      s.t. Dem {i in TOY} : x[i] <= Demand_toys[i];
	26
	27
	28      data;
	29      /* data  section */
	30
	31      set TOY := soldier train;
	32
	33      param Finishing_hours:=
	34      soldier         2
	35      train           1;
	36
	37      param Carpentry_hours:=
	38      soldier         1
	39      train           1;
	40
	41      param Demand_toys:=
	42      soldier        40
	43      train    6.02E+23;
	44
	45      param Profit_toys:=
	46      soldier         3
	47      train           2;
	48
	49      end;


我们并没有使用两个单独的文件，而是在一个具有模型段（第 1 行 到第 27 行）和一个数据段（第 28 行到第 49 行）的文件中声明了这个问题。

第 8 行定义了一个 SET。SET 是一个元素的值域。例如，如果我声明数学变量 xi, for all i in {1;2;3;4}，就说明 x 是一个包含 4 个元素的数组，因此我们就可以使用 x1、x2、x3、x4 了。在这个例子中，{1;2;3;4} 就是这个集合。因此，在 Giapetto 问题中，有一个集合 TOY。这个集合的实际值是什么呢？在这个文件的数据段中。请查看第 31 行。它定义了 TOY 集合包含 soldier 和 train。哦，这个集合不是一个数字类型的范围。那么这怎样实现呢？GLPK 是以一种非常有趣的方法来处理这个问题的。稍后我们就会看到如何使用这种方法。现在，请习惯数据段中 SET 声明所使用的语法：

set label := value1 value2 ... valueN ;

第 11、12 和 13 行定义了这个问题的参数。一共有三个参数：Finishing_hours、Carpentry_hours 和 Demand_toys。这些参数构成了这个问题的数据矩阵，并用来计算约束，正如我们稍后会看到的一样。

我们以 Finishing_hours 参数为例来看一下这个问题。它是在 TOY 集合上定义的，因此 TOY 集合中的每种玩具都有一个 Finishing_hours 值。记住每个士兵需要 2 小时的修整工作，每辆火车需要 1 小时的修整时间。现在请看一下第 33、34 和 35 行的内容。这是对每种玩具的修整工时的定义。实际上，这些行的内容声明 Finishing_hours[soldier]=2，Finishing_hours[train]=1。因此 Finishing_hours 就是一个 1 行 2 列的矩阵。

木工工时和需求参数都是类似地进行声明的。注意对于火车的需求是无限的，因此在 43 行上我们使用了一个很大的值作为上界。这个值看起来够大么？

第 17 行声明了一个变量 x，对于 TOY 中的每个 i（即 x[soldier] 和 x[train]），我们都要限制它们大于或等于 0。有了这些集合之后，声明变量就变得非常简单了，不是么？

第 20 行声明了对象函数是 z 的最大值，这是每种玩具的总体收益（一共有两种玩具：火车和士兵）。例如，对于士兵来说，收益是士兵个数乘以每个士兵的收益。

第 23、24 和 25 行上的约束也是类似地进行声明的。以修整工时约束为例：它是每种玩具的修整工时乘以所生产的这种玩具的数量之积的总和。对于这两种玩具（火车和士兵）来说，这必须小于或等于 100。类似地，木工工时综合也应该小于或等于 80。

需求约束与上面两个约束有所不同，因为它是为每种玩具而定义的，而不是为所有玩具类型总和进行定义的。因此，我们需要两个约束：一个用于火车，另外一个用于士兵，正如我们在第 25 行中看到的一样。注意索引变量（{i in TOY}）都是在 : 之前出现的。这告诉 GLPK 为 TOY 中的每种玩具类型都创建一个约束，每个约束的等式都在 : 之后出现。在本例中，GLPK 会创建

Dem[soldier] : x[soldier] <= Demand[soldier]

Dem[train] : x[train] <= Demand[train]

解析这个新模型会产生相同的结果。

#C++文件中的使用

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



在kdevelop里面建了一个工程，输入从网上找的一个程序，但是编译报了很多错。
报错类似于

	/home/stone/projects/d/main.cpp:11: undefined reference to `glp_create_prob'
	
然后我的思路是想把命令行敲的命令配置到Kdevelop下，感觉应该是路径的问题。
我尝试在CMakefilelists.txt中添加了一行

	add_librariy(/usr/local/lib/lglpk.so)
没有错误了，但是仍然无法运行。在运行配置里输入-lglpk -lm都不行。
然后在网上搜了一堆make 和cmake的语法，看了半天都没有讲怎么加参数的。
不过有几个教程特别好。
[跟我一起写makefile](http://bbs.chinaunix.net/thread-408225-1-1.html) [在Linux下使用cmake构建应用程序](https://www.ibm.com/developerworks/cn/linux/l-cn-cmake/) 

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

下一步应该是如何在其他文件中调用GLPK的计算结果。

