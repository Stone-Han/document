# Pure virtual function called: An Explanation #

本文来自[http://www.artima.com/cppsource/pure_virtual.html](http://www.artima.com/cppsource/pure_virtual.html)

另附一个中文博客[http://blog.csdn.net/liulina603/article/details/8931483](http://blog.csdn.net/liulina603/article/details/8931483)

“纯虚函数被调用”是C++程序崩溃的时候有时弹出来的信息。这是什么意思呢？你能找到一些简单的、很好的文档来解释崩溃之后很容易就找到错误的程序。但是还有另外一种不容易被发现的错误也会产生相同的错误信息。如果你对你的错误一头雾水，这很可能意味着你的程序间接地调用了一个野指针。本文会阐明所有的原因。


## Object-Oriented C++: The Programmer's View ##

（如果你知道什么是纯虚函数和抽象类，你可以跳过这一部分）
在C++中，虚函数让相关的类的实例在运行时有不同的行为。（也就是运行时多态）：

	class Shape {
	public:
		virtual double area() const;
		double value() const;
		// Meyers 3rd Item 7:
		virtual ~Shape();
	protected:
		Shape(double valuePerSquareUnit);
	private:
		double valuePerSquareUnit_;
	};
	
	class Rectangle : public Shape {
	public:
		Rectangle(double width, double height, double valuePerSquareUnit);
		virtual double area() const;
		// Meyers 3rd Item 7:
		virtual ~Rectangle();
	// ...
	};
	
	class Circle : public Shape {
	public:
		Circle(double radius, double valuePerSquareUnit);
		virtual double area() const;
		// Meyers 3rd Item 7:
		virtual ~Circle();
	// ...
	};
	
	double
	Shape::value() const
	{
		// Area is computed differently, depending
		// on what kind of shape the object is:
		return valuePerSquareUnit_ * area();
	}

(在析构函数前被注释掉的内容参考Scott Meyers's *Effective C++*:"Declare destructors virtual in polymorphic base classes.")

在C++中，函数的接口通过函数声明实现。成员函数在类的定义中声明。函数的实现通过定义函数实现？（A function's implementation is specified by defining the function）派生类可以重新定义函数，实现针对派生类的函数。当纯虚函数被调用的时候，选择的实现不是基于静态类型指针或者引用，而是基于指向的对象，这在运行时也是可以变化的。(?)

	print(shape->area());    //Might invoke Circle::area()or Rectangle::area().


纯虚函数声明之后不一定要定义。带有纯虚函数的类叫做抽象类，因为我们不可能创造这个类的实例。派生类必须定义所有继承的纯虚函数并且具体实现。

	class AbstractShape {
	public:
		virtual double area() const = 0;
		double value() const;
		// Meyers 3rd Item 7:
		virtual ~AbstractShape();
	protected:
		AbstractShape(double valuePerSquareUnit);
	private:
		double valuePerSquareUnit_;
	protected:
		AbstractShape(double valuePerSquareUnit);
	private:
		double valuePerSquareUnit_;
	};

Circle和Rectangle类继承AbstractShape类。下面这句话无法编译，即使有对应的构造函数。

	// AbstractShape* p = new AbstractShape(value);
	
但是下面这些可以：

	Rectangle* pr = new Rectangle(height, weight, value);
	Circle* pc = new Circle(radius, value);
	
	AbstractShape* p = pr;
	p = pc;

## Object Oriented C++: Under the Covers ##

(如果你已经知道了"vtbl",你可以跳过这一部分）
这种运行时的魔法是如何发生的？
一般情况下，具有任何虚函数的类都有一个函数的指针数组，叫做"vtbl"。每一个这种类的实例都有一个类的指针，如下图所示：

![](http://ss1.sinaimg.cn/large/7f318987gy1fg5qgccku1j20d00jcgle&690)  

如果带有虚函数的抽象类没有定义这个函数，那么在vtbl中会出现什么样的情况呢？通常，C++系统提供了一个特殊的函数，会显示出“Pure virtual function called"，并且停止程序。

## Build'em Up, Tear'em Down ##

当你构造一个派生类的实例的时候，发生了什么事情？如果这个类有vtbl，整个过程大致如下：

Step 1: 构造顶层的基类：
	
	1. 让实例指针指向基类的vtbl
	2. 构造基类实例成员变量
	3. 执行基类构造函数

Step 2: 构造派生类：

	1. 将实例指针指向派生类的vtbl
	2. 构造派生类的成员变量
	3. 执行派生类的构造函数

析构通常倒叙，大概如下：

Step 1: 析构派生类：

	1. （实力指针已经指向派生类的vtbl）
	2. 执行析构函数
	3. 析构成员变量

Step 2: 析构基类：
	
	1. 指针指向基类的vtbl
	2. 执行基类析构函数
	3. 析构成员变量

## Two of the Classic Blunders ##

如果你从基类的构造函数中尝试调用一个虚函数会怎么样？

	// From sample program 1:
	AbstractShape(double valuePerSquareUnit)
		: valuePerSquareUnit_(valuePerSquareUnit)
	{
		// ERROR: Violation of Meyers 3rd Item 9!
		std::cout << "creating shape, area = " << area() << std::endl;
	}

这显然是一个调用纯虚函数的例子。编译器会提示我们这种错误。如果基类的析构函数直接调用纯虚函数你也会得到相同的提示。

如果情况稍微复杂一点，错误可能没有那么明显：

	// From sample program 3:
	AbstractShape::AbstractShape(double valuePerSquareUnit)
		: valuePerSquareUnit_(valuePerSquareUnit)
	{
		// ERROR: Indirect violation of Meyers 3rd Item 9!
		std::cout << "creating shape, value = " << value() << std::endl;
	}

这个基类的构造函数就是step 1(3)中所说的，它调用了一个成员函数的实例value()，而这个实例调用纯虚函数area()。这个对象现在仍然是AbstracShape。当它尝试去调用纯虚函数的时候发生了什么？你的程序很有可能以一个类似“pure virtual function called"的信息弹出而崩溃。

同样，在析构函数中简介地调用纯虚函数也能导致这类信息。所有通过partially-constructed调用虚函数的例子都会出这个错误。

这些就是导致pure virtual function called信息的根源。从后续程序的检查中可以很快地找出错误，堆栈调用信息能够清楚地指向问题所在。

## Pointing Out Blame ##

还有不止一种情况会导致这种错误，但是在书中或者网上都没有见到过明显的描述。

考虑下面的代码：

	// From sample program 5:
		AbstractShape* p1 = new Rectangle(width, height, valuePerSquareUnit);
		std::cout << "value = " << p1->value() << std::endl;
		AbstractShape* p2 = p1;  // Need another copy of the pointer.
		delete p1;
		std::cout << "now value = " << p2->value() << std::endl;

我们来一行一行看。

	AbstractShape* p1 = new Rectangle(width, height, valuePerSquareUnit);

创建了一个新对象。经历两个步骤：1.类似基类的行为；2. 类似派生类的行为

	std::cout << "value = " << p1->value() << std::endl;

看起来似乎一切正常。

	AbstractShape* p2 = p1;  // Need another copy of the pointer.

p1可能会发生一些事情，所以我们做一份它的拷贝。
	
	delete p1;

对象的析构也要两个步骤: 1. where the object acts like a derived class instance; 2. where it acts like a base class instance

注意到p1在delete之后有可能变化。编译器可以使指针在析构他们指向的数据之后清零。还好我们有一份拷贝p2，不变。

	std::cout << "now value = " << p2->value() << std::endl;

啊哦。
这是另一种典型错误：间接调用野指针。野指针就是它指向的对象已经被delete或者主存空间已经被释放。

所以现在p2指向的是前对象。这就好比什么？根据C++标准，这叫做“undefined”。这个技术术语的意思是，实际上任何事情都有可能发生：程序能够崩溃或者继续跑，但是生产垃圾结果，或者给Bjarne Stroustrup（C++发明者）发送一封email说你有多丑。不同的行为根据编译器或者机器或者运行的不同而不同。在实际中，可能会有一下几个可能：

- 主存可能被标记为deallocated。任何尝试获取它的都被定义为野指针。
- 主存可能被恶意scrambled。在释放之后主存管理系统可能写入一些垃圾信息
- 主存可能被重用。
- 主存可能就那样了。

最后一个情况比较有意思，什么叫“就那样了”？在这种情况下，它是一个抽象类的实例，所以也就是vtbl。那么我们从vtbl下调用一个纯虚函数会发生什么？

Pure virtual function called.

## Meanwhile, Back in the Real World ##

理论不错，在实际中会发生什么呢？

下面测试了不同错误方式在不同环境下的情况。

## Owning Up ##

那么应该如何避免这种情况呢？

前四种情况比较好解决。注意错误信息。

第五种例子的“野指针”怎么办呢？不管什么语言，编程人员都应该设计对象的归属。什么东西属于哪个对象。归属可能是：

- 转换别的东西
- 没有转换归属权的“借出”
- 共享：通过reference counts或者garbage collection

什么“东西”能拥有一个对象呢？

- 另一个对象。
- 对象的集合。比如所有的smart pointers指向被拥有的对象。
- 函数。当函数被调用的时候，它可能假设归属权（转移）或者没有（借出).函数有局部变量，但是不一定有局部变量指针指向的对象。

在我们的例子中没有清晰的归属关系。一个函数创建了一个对象，并且分配了两个指针。但是谁拥有这个对象呢？可能是函数，但是不管哪种情况，都应该负责避免这种问题。它应该用一个“dumb”指针（删除后显式归零）或者smart pointer 而不是两个指针。

在实际中，远没有这么简单。对象可能从一个模块传递到另一个完全不同的模块，代码也可能由完全不同的人写。对象的归属问题能跨越很长的距离。

任何时候你传递对象的时候都要清晰地知道归属权的问题。这是个简单的问题，但是没有问题能神奇地自己回答。思考是没有替代品的。

为自己考虑不一定要自己思考。Tom Cargill的“Localized Ownership"阐述了一些方法。Scott Meyers Iterm13 和Item14 。
 













