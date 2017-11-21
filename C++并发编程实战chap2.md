C++并发编程实战

# 管理线程 #
## 基本线程管理 ##
### 启动线程 ###
     void do_some_work();
     std::thread my_thread(do_some_work);
其中构造函数中可以传入任何callable类型。
比如

    class background_task{
    public:
		void operator()() const
		{
			do1();
			do2();
		}

    };
	background_task f;
	std::thread my_thread(f);

但是编译器可能会解释成`std::thread my_thread(background_task())；`

正确的语法为`std::thread my_thread( (background_task()) );
std::thread my_thread{background_task()};`

**但是我试过上面3中方法都没有报错。do1();会报错找不到标识符。**

另外还可以用lambda表达式（A.5）。
    
    std::thread mythread( [] {
			do_something();
			do_something_else();
		});

**这里需要注意的是要明确决定线程是join还是detach**
**当不等待线程结束且线程有局部变量的指针或者引用时，要格外注意。处理这种情况的方法是把数据复制到线程中而不是共享。**
详见清单2.1

### join() ###

join()会清理所有与该线程相关联的存储器。只能对一个给定的线程调用一次join()

但是如果在调用join()之前发生了异常，就容易跳过对join的调用。
这时候我们可以用try/catch块开捕捉异常，但是又显得太啰嗦。

解决方法是利用标准的**资源获取即初始化RAII**语法。

	class thread_guard
	{
	    std::thread& t;
	public:
	    explicit thread_guard(std::thread& t_):
	        t(t_)
	    {}
	    ~thread_guard()
	    {
	        if(t.joinable())   //1
	        {
	            t.join();		//2
	        }
	    }
	    thread_guard(thread_guard const&)=delete;	//3
	    thread_guard& operator=(thread_guard const&)=delete;
	};


	struct func{};

	void f()
	{
	    int some_local_state;
	    func my_func(some_local_state);
	    std::thread t(my_func);
	    thread_guard g(t);
	        
	    do_something_in_current_thread();
	}//4

当执行到末尾4时，局部对象按照构造函数的逆序销毁。因此thread_guard对象g首先被销毁，并且析构函数中线程执行了join()。即使异常退出也会执行。

2调用join前首先要判断是不是joinable的。因为join只能调用一次。

3这里使用了delete(A.2)，主要的作用是控制有些构造函数不被复制。
### detach() ###

被分离的线程通常叫做守护线程，它运行在后台，可能再应用程序的整个生命周期中运行。

当调用detach()之后，线程将不能再被join()。
可以通过`t.joinalbe()`判断(false)。

案例：word文档实际上是在一个程序中开启不同线程编辑。详见清单2.4。
## 线程传参 ##
当给线程传递参数的时候，参数会以默认的方式被复制到内部存储空间，即使参数中有引用。

    void f(int i, std::string const& s);
	std::thread t(f,3,"hello");
“hello”当在新线程的上下文中才会作为char const*传送并转换成string。

    void f(int i,std::string const& s);
	void oops(int some_param)
	{
		char buf[1024];
		sprintf(buf,"%i",some_param);	//此函数的作用是将int param转换成char[]类型
		std::thread t(f,3,buf);		//wrong
		std::thread t(f,3,std::string(buf));	//right
		t.detach();
	}
函数oops会在buf在新线程上被转换成string类型之前退出，从而导致程序报错未定义。
解决方法是在buf传递给线程的之前转换成string。
    
另外可能当你希望通过引用改变对象的值时，由于复制而没有改变。

    void update_data(widget_id w, widget_data& data);
	void oops(widget_id w)
	{
		widget_data data;
		std::thread t(update_data,w,data);
		display_status();
		t.join();
		process_data(data);
	}

这里我们希望线程t调用update_data将data更新，然后传给process_data对更新过的数据进行处理。但是构造函数只是盲目地复制data的副本。因此它调用update_data时，传递的是data在内部的副本，而不是引用。当线程完成时，随着内部副本的销毁，改动都被舍弃，传递一个未改变的data给process。

解决方案用std::ref

    std::thread t(update_data, w, std::ref(data));

另外，这里的参数能被移动。一个对象内保存的数据转移到另一个对象，使原来的对象变成NULL。结合std::unique_ptr使用。

    void process_object(std::unique_ptr<object>);
	std::unique_ptr<object> p(new object);
	p->prepare_data(42);
	std::thread t(process_object,std::move(p));

object先被转移到新线程的内部存储中，然后进入process_object。

## 移动线程move() ##

线程的所有权是可以移动的。但是不能通过对一个线程赋值舍弃原来的线程。例如，

    std::thread t1(f1);
	std::thread t2(f2);
	t2 =  std::move(t1);

但是移动可以将所有权从一个函数中转移。

	std::thread f()
	{
	    void some_function();
	    return std::thread(some_function);
	}
	std::thread g()
	{
	    void some_other_function(int);
	    std::thread t(some_other_function,42);
	    return t;
	}
	
	int main()
	{
	    std::thread t1=f();
	    t1.join();
	    std::thread t2=g();
	    t2.join();
	}

如果要把所有权转移到函数中，只能以值的形式接收std::thread的实例作为参数。

    void f(std::thread t);
	void g()
	{
		f (std::thread(some_func);
		std::thread t(some_func);
		f(std::move(t));	
	}
也就是说可以将线程作为参数传递，但是要用移动。

可以结合thread_guard，确保退出一个作用域之前线程都已经完成。

	#include <thread>
	#include <utility>
	
	class scoped_thread
	{
	    std::thread t;
	public:
	    explicit scoped_thread(std::thread t_):
	        t(std::move(t_))
	    {
	        if(!t.joinable())
	            throw std::logic_error("No thread");
	    }
	    ~scoped_thread()
	    {
	        t.join();
	    }
	    scoped_thread(scoped_thread const&)=delete;
	    scoped_thread& operator=(scoped_thread const&)=delete;
	};
	
	struct func	{	};
		
	void f()
	{
	    int some_local_state;
	    scoped_thread t(std::thread(func(some_local_state)));
	        
	    do_something_in_current_thread();
	}
同样也可以结合vector

	#include <vector>
	#include <thread>
	#include <algorithm>
	#include <functional>
	
	void do_work(unsigned id)
	{}
	
	void f()
	{
	    std::vector<std::thread> threads;
	    for(unsigned i=0;i<20;++i)
	    {
	        threads.push_back(std::thread(do_work,i));
	    }
	    std::for_each(threads.begin(),threads.end(),
	        std::mem_fn(&std::thread::join));
	}
	
	int main()
	{
	    f();
	}
## thread::hardware_currency 线程数量 ##

该函数返回一个对于给定程序执行时能够真正并发运行的线程数量的指示。如果该信息不可用则函数可能返回0，但是这有利于线程间分割任务。比如下面是std::accumulate的简单并行版本。该实现假定所有操作没有异常。

	#include <thread>
	#include <numeric>
	#include <algorithm>
	#include <functional>
	#include <vector>
	#include <iostream>
	
	template<typename Iterator,typename T>
	struct accumulate_block
	{
	    void operator()(Iterator first,Iterator last,T& result)
	    {
	        result=std::accumulate(first,last,result);
	    }
	};
	
	template<typename Iterator,typename T>
	T parallel_accumulate(Iterator first,Iterator last,T init)
	{
	    unsigned long const length=std::distance(first,last);
	
	    if(!length)
	        return init;	//输入范围为空时返回init
	
	    unsigned long const min_per_thread=25;
	    unsigned long const max_threads=
	        (length+min_per_thread-1)/min_per_thread;	//计算最大线程数
	
	    unsigned long const hardware_threads=
	        std::thread::hardware_concurrency();

	    //实际运行的线程数取最大线程数和硬件线程数的最小值。
		//如果硬件线程返回0那么给它赋值2
	    unsigned long const num_threads=
	        std::min(hardware_threads!=0?hardware_threads:2,max_threads);	
	
	    unsigned long const block_size=length/num_threads;	//每个线程处理的范围
	
	    std::vector<T> results(num_threads);	
	    std::vector<std::thread>  threads(num_threads-1);	//创建线程，注意这里-1因为已经有一个
	
	    Iterator block_start=first;
	    for(unsigned long i=0;i<(num_threads-1);++i)	//
	    {
	        Iterator block_end=block_start;
	        std::advance(block_end,block_size);		//给迭代器增加指定偏移量
	        threads[i]=std::thread(					//启动线程累计这一块的结果
	            accumulate_block<Iterator,T>(),
	            block_start,block_end,std::ref(results[i]));
	        block_start=block_end;					//下一个块的开始
	    }
		//处理最后剩余的一块
	    accumulate_block<Iterator,T>()(block_start,last,results[num_threads-1]);
	    
	    std::for_each(threads.begin(),threads.end(),
	        std::mem_fn(&std::thread::join));	//等待所有线程完成
	
	    return std::accumulate(results.begin(),results.end(),init);		//返回累加结果
	}
	
	int main()
	{
	    std::vector<int> vi;
	    for(int i=0;i<10;++i)
	    {
	        vi.push_back(10);
	    }
	    int sum=parallel_accumulate(vi.begin(),vi.end(),5);
	    std::cout<<"sum="<<sum<<std::endl;
	}

但是这里不能直接从一个线程中返回值，所以必须将结果传入result中，我们可以用future替代。
## std::thread::id 线程id ##
方式：

1. `get_id()`
2. `std::this_thread::get_id()`

可以对id类型进行复制和比较。还可以对它进行排序、hash。

id还可以用来检车一个线程是否需要执行某些操作。

	std::thread::id master_thread;
    if(std::this_thread::get_id() == master_thread_
	{
		do_something();
	}