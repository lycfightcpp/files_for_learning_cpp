# 1.线程管理

**创建线程**:

首先要引入头文件#include<thread>，C++11中管理线程的函数和类在该头文件中声明，其中包括std::thread类。

语句"std::thread th1(proc1);"创建了一个名为th1的线程，并且线程th1开始执行。

实例化std::thread类对象时，至少需要传递函数名作为参数。如果函数为有参函数,如"void proc2(int a,int b)",那么实例化std::thread类对象时，则需要传递更多参数，参数顺序依次为函数名、该函数的第一个参数、该函数的第二个参数，···，如"std::thread th2(proc2,a,b);"。这里的传参，后续章节还会有详解与提升。

只要创建了线程对象（前提是，实例化std::thread对象时传递了“函数名/可调用对象”作为参数），线程就开始执行。

总之，使用C++线程库启动线程，可以归结为构造std::thread对象。

那么至此一个简单的多线程并发程序就编写完了吗？

不，还没有。当线程启动后，一定要在和线程相关联的std::thread对象销毁前，对线程运用join()或者detach()方法。

join()与detach()都是std::thread类的成员函数，是两种线程阻塞方法，两者的区别是是否等待子线程执行结束。

新手先把join()弄明白就行了，然后就可以去学习后面的章节，等过段时间再回头来学detach()。

等待调用线程运行结束后当前线程再继续运行，例如，主函数中有一条语句th1.join(),那么执行到这里，主函数阻塞，直到线程th1运行结束，主函数再继续运行。

整个过程就相当于：你在处理某件事情（你是主线程），中途你让老王帮你办一个任务（与你同时执行）（创建线程1，该线程取名老王），又叫老李帮你办一件任务（创建线程2，该线程取名老李），现在你的一部分工作做完了，剩下的工作得用到他们的处理结果，那就调用"老王.join()"与"老李.join()"，至此你就需要等待（主线程阻塞），等他们把任务做完（子线程运行结束），你就可以继续你手头的工作了（主线程不再阻塞）。

一提到join,你脑海中就想起两个字，"等待"，而不是"加入"，这样就很容易理解join的功能。

```cpp
#include<iostream>
#include<thread>
using namespace std;
void proc(int &a)
{
    cout << "我是子线程,传入参数为" << a << endl;
    cout << "子线程中显示子线程id为" << this_thread::get_id()<< endl;
}
int main()
{
    cout << "我是主线程" << endl;
    int a = 9;
    //第一个参数为函数名，第二个参数为该函数的第一个参数，如果该函数接收多个参数就依次写在后面。此时线程开始执行。
    thread th2(proc, a);
    cout << "主线程中显示子线程id为" << th2.get_id() << endl;
    th2.join()；//此时主线程被阻塞直至子线程执行结束。
    return 0;
}
```
调用join()会清理线程相关的存储部分，这代表了join()只能调用一次。使用joinable()来判断join()可否调用。同样，detach()也只能调用一次，一旦detach()后就无法join()了，有趣的是，detach()可否调用也是使用joinable()来判断。

# 2.互斥与同步

什么是互斥量？
这样比喻：单位上有一台打印机（共享数据a），你要用打印机（线程1要操作数据a），同事老王也要用打印机(线程2也要操作数据a)，但是打印机同一时间只能给一个人用，此时，规定不管是谁，在用打印机之前都要向领导申请许可证（lock），用完后再向领导归还许可证(unlock)，许可证总共只有一个,没有许可证的人就等着在用打印机的同事用完后才能申请许可证(阻塞，线程1lock互斥量后其他线程就无法lock,只能等线程1unlock后，其他线程才能lock)。那么，打印机就是共享数据，访问打印机的这段代码就是临界区，这个必须互斥使用的许可证就是互斥量。

互斥量是为了解决数据共享过程中可能存在的访问冲突的问题。这里的互斥量保证了使用打印机这一过程不被打断。

互斥量怎么使用？

首先需要#include<mutex>；（std::mutex和std::lock_guard都在<mutex>头文件中声明。）然后需要实例化std::mutex对象；最后需要在进入临界区之前对互斥量加锁，退出临界区时对互斥量解锁；至此，互斥量走完了它的一生。

**lock()与unlock()**:

需要在进入临界区之前对互斥量lock，退出临界区时对互斥量unlock；当一个线程使用特定互斥量锁住共享数据时，其他的线程想要访问锁住的数据，都必须等到之前那个线程对数据进行解锁后，才能进行访问。

程序实例化mutex对象m,本线程调用成员函数m.lock()会发生下面 2 种情况： (1)如果该互斥量当前未上锁，则本线程将该互斥量锁住，直到调用unlock()之前，本线程一直拥有该锁。 (2)如果该互斥量当前被其他线程锁住，则本线程被阻塞,直至该互斥量被其他线程解锁，此时本线程将该互斥量锁住，直到调用unlock()之前，本线程一直拥有该锁。

不推荐实直接去调用成员函数lock()，因为如果忘记unlock()，将导致锁无法释放，使用lock_guard或者unique_lock则能避免忘记解锁带来的问题。

**lock_guard**:

std::lock_guard()是什么呢？它就像一个保姆，职责就是帮你管理互斥量，就好像小孩要玩玩具时候，保姆就帮忙把玩具找出来，孩子不玩了，保姆就把玩具收纳好。

其原理是：声明一个局部的std::lock_guard对象，在其构造函数中进行加锁，在其析构函数中进行解锁。最终的结果就是：创建即加锁，作用域结束自动解锁。从而使用std::lock_guard()就可以替代lock()与unlock()。

**通过设定作用域，使得std::lock_guard在合适的地方被析构**（在互斥量锁定到互斥量解锁之间的代码叫做临界区（需要互斥访问共享资源的那段代码称为临界区），临界区范围应该尽可能的小，即lock互斥量后应该尽早unlock），**通过使用{}来调整作用域范围，可使得互斥量m在合适的地方被解锁：**
```cpp
#include<iostream>
#include<thread>
#include<mutex>
using namespace std;
mutex m;//实例化m对象，不要理解为定义变量
void proc1(int a)
{
    //用此语句替换了m.lock()；lock_guard传入一个参数时，该参数为互斥量，此时调用了lock_guard的构造函数，申请锁定m
    lock_guard<mutex> g1(m);
    cout << "proc1函数正在改写a" << endl;
    cout << "原始a为" << a << endl;
    cout << "现在a为" << a + 2 << endl;
}//此时不需要写m.unlock(),g1出了作用域被释放，自动调用析构函数，于是m被解锁

void proc2(int a)
{
    {
        lock_guard<mutex> g2(m);
        cout << "proc2函数正在改写a" << endl;
        cout << "原始a为" << a << endl;
        cout << "现在a为" << a + 1 << endl;
    }//通过使用{}来调整作用域范围，可使得m在合适的地方被解锁
    cout << "作用域外的内容3" << endl;
    cout << "作用域外的内容4" << endl;
    cout << "作用域外的内容5" << endl;
}
int main()
{
    int a = 0;
    thread proc1(proc1, a);
    thread proc2(proc2, a);
    proc1.join();
    proc2.join();
    return 0;
}
```
std::lock_gurad也可以传入两个参数，第一个参数为adopt_lock标识时，表示构造函数中不再进行互斥量锁定，因此此时需要提前手动锁定。
```cpp
#include<iostream>
#include<thread>
#include<mutex>
using namespace std;
mutex m;//实例化m对象，不要理解为定义变量
void proc1(int a)
{
    m.lock();//手动锁定
    lock_guard<mutex> g1(m, adopt_lock);
    cout << "proc1函数正在改写a" << endl;
    cout << "原始a为" << a << endl;
    cout << "现在a为" << a + 2 << endl;
}//自动解锁

void proc2(int a)
{
    lock_guard<mutex> g2(m);//自动锁定
    cout << "proc2函数正在改写a" << endl;
    cout << "原始a为" << a << endl;
    cout << "现在a为" << a + 1 << endl;
}//自动解锁
int main()
{
    int a = 0;
    thread proc1(proc1, a);
    thread proc2(proc2, a);
    proc1.join();
    proc2.join();
    return 0;
}
```

**unique_lock**:

std::unique_lock类似于lock_guard,只是std::unique_lock用法更加丰富，同时支持std::lock_guard()的原有功能。使用std::lock_guard后不能手动lock()与手动unlock();使用std::unique_lock后可以手动lock()与手动unlock(); std::unique_lock的第二个参数，除了可以是adopt_lock,还可以是try_to_lock与defer_lock;

try_to_lock: 尝试去锁定，得保证锁处于unlock的状态,然后尝试现在能不能获得锁；尝试用mutx的lock()去锁定这个mutex，但如果没有锁定成功，会立即返回，不会阻塞在那里，并继续往下执行；

defer_lock: 始化了一个没有加锁的mutex;
```cpp
#include<iostream>
#include<thread>
#include<mutex>
using namespace std;
mutex m;
void proc1(int a)
{
    unique_lock<mutex> g1(m, defer_lock);//始化了一个没有加锁的mutex
    cout << "xxxxxxxx" << endl;
    g1.lock();//手动加锁，注意，不是m.lock();注意，不是m.lock(),m已经被g1接管了;
    cout << "proc1函数正在改写a" << endl;
    cout << "原始a为" << a << endl;
    cout << "现在a为" << a + 2 << endl;
    g1.unlock();//临时解锁
    cout << "xxxxx"  << endl;
    g1.lock();
    cout << "xxxxxx" << endl;
}//自动解锁

void proc2(int a)
{
    unique_lock<mutex> g2(m,try_to_lock);//尝试加锁一次，但如果没有锁定成功，会立即返回，不会阻塞在那里，且不会再次尝试锁操作。
    if(g2.owns_lock)
    {//锁成功
        cout << "proc2函数正在改写a" << endl;
        cout << "原始a为" << a << endl;
        cout << "现在a为" << a + 1 << endl;
    }
    else
    {//锁失败则执行这段语句
        cout <<""<<endl;
    }
}//自动解锁

int main()
{
    int a = 0;
    thread proc1(proc1, a);
    thread proc2(proc2, a);
    proc1.join();
    proc2.join();
    return 0;
}
```
使用try_to_lock要小心，因为try_to_lock尝试锁失败后不会阻塞线程，而是继续往下执行程序，因此，需要使用if-else语句来判断是否锁成功,只有锁成功后才能去执行互斥代码段。而且需要注意的是，因为try_to_lock尝试锁失败后代码继续往下执行了，因此该语句不会再次去尝试锁。

std::unique_lock所有权的转移

注意，这里的转移指的是std::unique_lock对象间的转移；std::mutex对象的所有权不需要手动转移给std::unique_lock , std::unique_lock对象实例化后会直接接管std::mutex。
```cpp
mutex m;
{  
    unique_lock<mutex> g2(m,defer_lock);
    unique_lock<mutex> g3(move(g2));//所有权转移，此时由g3来管理互斥量m
    g3.lock();
    g3.unlock();
    g3.lock();
}
```

**recursive_mutex**：

std::recursive_mutex与std::mutex一样也只定义了lock，trylock，unlock和native_handle(用于返回底层实现定义的原生句柄)方法。

std::recursive_mutex与std::mutex的不同点在于std::recursive_mutex允许在一个已经锁定互斥量的线程上多次调用lock方法(与之对应的在一个已经锁定std::mutex的线程上，再次调用std::mutex的lock方法是未定义行为)，在已锁定互斥量的情况下再次调用lock方法会增加std::recursive_mutex的所有权等级。在调用std::recursive_mutex的unlock方法时，若lock与unlock调用次数匹配时(即所有权等级为1时)会解锁互斥量，否则会减少std::recursive_mutex的所有权等级。当一个线程锁定互斥量时，其他线程若尝试锁定互斥量就会被阻塞。(所有权的最大层数是未指定的。若超出此数，则可能抛std::system_error类型异常。std::recursive_mutex的lock与unlock，有点类似与COM中的AddRef和release)
```cpp
std::recursive_mutex rmutex;
void test()
{
	std::thread worker{ []() {
		//锁定互斥量，rmutex的所有权等级为1
		rmutex.lock();
		do_something();
		{
			//在锁定互斥量的情况下，再次调用lock，所有权等级增加1(为2)
			rmutex.lock();
			do_something_else();
			//所有权等级为2，不等于1，将所有权登记减少1，此次unlock调用后线程依然占有互斥量
			rmutex.unlock();
		}
		//所有权等级为1，此次unlock调用后线程解锁互斥量
		rmutex.unlock();
	} };
	worker.join();
}
```

**condition_variable**:

需要#include<condition_variable>，该头文件中包含了条件变量相关的类，其中包括std::condition_variable类

如何使用？std::condition_variable类搭配std::mutex类来使用，std::condition_variable对象(std::condition_variable cond;)的作用不是用来管理互斥量的，它的作用是用来同步线程，它的用法相当于编程中常见的flag标志（A、B两个人约定flag=true为行动号角，默认flag为false,A不断的检查flag的值,只要B将flag修改为true，A就开始行动）。

类比到std::condition_variable，A、B两个人约定notify_one为行动号角，A就等着（调用wait(),阻塞）,只要B一调用notify_one，A就开始行动（不再阻塞）。

std::condition_variable的具体使用代码实例可以参见文章中“生产者与消费者问题”章节。

wait(locker) :

wait函数需要传入一个std::mutex（一般会传入std::unique_lock对象）,即上述的locker。wait函数会自动调用 locker.unlock() 释放锁（因为需要释放锁，所以要传入mutex）并阻塞当前线程，本线程释放锁使得其他的线程得以继续竞争锁。一旦当前线程获得notify(通常是另外某个线程调用 notify_* 唤醒了当前线程)，wait() 函数此时再自动调用 locker.lock()上锁。

cond.notify_one(): 随机唤醒一个等待的线程

cond.notify_all(): 唤醒所有等待的线程

**async与futur**：

std::async是一个函数模板，用来启动一个异步任务，它返回一个std::future类模板对象，future对象起到了**占位**的作用（记住这点就可以了），占位是什么意思？就是说该变量现在无值，但将来会有值（好比你挤公交瞧见空了个座位，刚准备坐下去就被旁边的小伙给拦住了：“这个座位有人了”，你反驳道：”这不是空着吗？“，小伙：”等会人就来了“）,刚实例化的future是没有储存值的，但在调用std::future对象的get()成员函数时，主线程会被阻塞直到异步线程执行结束，并把返回结果传递给std::future，即通过FutureObject.get()获取函数返回值。

相当于你去办政府办业务（主线程），把资料交给了前台，前台安排了人员去给你办理（std::async创建子线程），前台给了你一个单据（std::future对象），说你的业务正在给你办（子线程正在运行），等段时间你再过来凭这个单据取结果。过了段时间，你去前台取结果（调用get()），但是结果还没出来（子线程还没return），你就在前台等着（阻塞），直到你拿到结果（子线程return），你才离开（不再阻塞）。
```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include<future>
#include<Windows.h>
using namespace std;
double t1(const double a, const double b)
{
 double c = a + b;
 Sleep(3000);//假设t1函数是个复杂的计算过程，需要消耗3秒
 return c;
}

int main() 
{
 double a = 2.3;
 double b = 6.7;
 future<double> fu = async(t1, a, b);//创建异步线程线程，并将线程的执行结果用fu占位；
 cout << "正在进行计算" << endl;
 cout << "计算结果马上就准备好，请您耐心等待" << endl;
 cout << "计算结果：" << fu.get() << endl;//阻塞主线程，直至异步线程return
//cout << "计算结果：" << fu.get() << endl;//取消该语句注释后运行会报错，因为future对象的get()方法只能调用一次。
 return 0;
}
```

**shared_future**:

std::future与std::shard_future的用途都是为了**占位**，但是两者有些许差别。std::future的get()成员函数是转移数据所有权;std::shared_future的get()成员函数是复制数据。 因此： **future对象的get()只能调用一次**；无法实现多个线程等待同一个异步线程，一旦其中一个线程获取了异步线程的返回值，其他线程就无法再次获取。 **std::shared_future对象的get()可以调用多次**；可以实现多个线程等待同一个异步线程，每个线程都可以获取异步线程的返回值。

**atomic**:

原子操作指“不可分割的操作”，也就是说这种操作状态要么是完成的，要么是没完成的，不存在“操作完成了一半”这种状况。互斥量的加锁一般是针对一个代码段，而原子操作针对的一般都是一个变量(操作变量时加锁防止他人干扰)。 std::atomic<>是一个模板类，使用该模板类实例化的对象，提供了一些保证原子性的成员函数来实现共享数据的常用操作。

可以这样理解： 在以前，定义了一个共享的变量(int i=0)，多个线程会用到这个变量，那么每次操作这个变量时，都需要lock加锁，操作完毕unlock解锁，以保证线程之间不会冲突；但是这样每次加锁解锁、加锁解锁就显得很麻烦，那怎么办呢？ 现在，实例化了一个类对象(std::atomic<int> I=0)来代替以前的那个变量（这里的对象I你就把它看作一个变量，看作对象反而难以理解了），每次操作这个对象时，就不用lock与unlock，这个对象自身就具有原子性（相当于加锁解锁操作不用你写代码实现，能自动加锁解锁了），以保证线程之间不会冲突。

提到std::atomic<>，你脑海里就想到一点就可以了：std::atomic<>用来定义一个自动加锁解锁的共享变量（“定义”“变量”用词在这里是不准确的，但是更加贴切它的实际功能），供多个线程访问而不发生冲突。
```cpp
//原子类型的简单使用
std::atomic<bool> b(true);
b = false;
```
std::atomic<>对象提供了常见的原子操作（通过调用成员函数实现对数据的原子操作）： store是原子写操作，load是原子读操作。exchange是于两个数值进行交换的原子操作。**即使使用了std::atomic<>，也要注意执行的操作是否支持原子性**，也就是说，你不要觉得用的是具有原子性的变量（准确说是对象）就可以为所欲为了，你对它进行的运算不支持原子性的话，也不能实现其原子效果。一般针对++，–，+=，-=，&=，|=，^=是支持的，这些原子操作是通过在std::atomic<>对象内部进行运算符重载实现的。

# 3.生产者消费者问题

生产者-消费者模型是经典的多线程并发协作模型。生产者用于生产数据，生产一个就往共享数据区存一个，如果共享数据区已满的话，生产者就暂停生产；消费者用于消费数据，一个一个的从共享数据区取，如果共享数据区为空的话，消费者就暂停取数据，且生产者与消费者不能直接交互。

```cpp
/* 生产者消费者问题 */
#include <iostream>
#include <deque>
#include <thread>
#include <mutex>
#include <condition_variable>
#include<Windows.h>
using namespace std;

deque<int> q;
mutex mu;
condition_variable cond;
int c = 0;  //缓冲区的产品个数

void producer() 
{ 
    int data1;
    while (1) 
    {//通过外层循环，能保证生产不停止
        if(c < 3) 
        {//限流
            {
            data1 = rand();
            unique_lock<mutex> locker(mu);//锁
            q.push_front(data1);
            cout << "存了" << data1 << endl;
            cond.notify_one();  // 通知取
            ++c;
            }
            Sleep(500);
        }
    }
}

void consumer() 
{
    int data2;//data用来覆盖存放取的数据
    while (1) 
    {
        {
        unique_lock<mutex> locker(mu);
        while(q.empty())
            //wait()阻塞前先会解锁,解锁后生产者才能获得锁来放产品到缓冲区；生产者notify后，将不再阻塞，且自动又获得了锁。
            cond.wait(locker); 
        data2 = q.back();//取的第一步
        q.pop_back();//取的第二步
        cout << "取了" << data2 <<endl;
        --c;
        }
        Sleep(1500);
    }
}
int main() 
{
    thread t1(producer);
    thread t2(consumer);
    t1.join();
    t2.join();
    return 0;
}
```

# 4.线程池

**不采用线程池时**：

创建线程 -> 由该线程执行任务 -> 任务执行完毕后销毁线程。即使需要使用到大量线程，每个线程都要按照这个流程来创建、执行与销毁。

虽然创建与销毁线程消耗的时间 远小于 线程执行的时间，但是对于需要频繁创建大量线程的任务，创建与销毁线程 所占用的时间与CPU资源也会有很大占比。

**为了减少创建与销毁线程所带来的时间消耗与资源消耗**，因此采用线程池的策略：

程序启动后，预先创建一定数量的线程放入空闲队列中，这些线程都是处于阻塞状态，基本不消耗CPU，只占用较小的内存空间。

接收到任务后，任务被挂在任务队列，线程池选择一个空闲线程来执行此任务。

任务执行完毕后，不销毁线程，线程继续保持在池中等待下一次的任务。

**线程池所解决的问题**：

(1) 需要频繁创建与销毁大量线程的情况下，由于线程预先就创建好了，接到任务就能马上从线程池中调用线程来处理任务，减少了创建与销毁线程带来的时间开销和CPU资源占用。

(2) 需要并发的任务很多时候，无法为每个任务指定一个线程（线程不够分），使用线程池可以将提交的任务挂在任务队列上，等到池中有空闲线程时就可以为该任务指定线程。

## 线程池的实现

```cpp
#ifndef THREAD_POOL_H
#define THREAD_POOL_H

#include <vector>
#include <queue>
#include <memory>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <future>
#include <functional>
#include <stdexcept>

class ThreadPool {
public:
    ThreadPool(size_t);
    template<class F, class... Args>
    auto enqueue(F&& f, Args&&... args) 
        -> std::future<typename std::result_of<F(Args...)>::type>;
    ~ThreadPool();
private:
    // need to keep track of threads so we can join them
    std::vector< std::thread > workers;
    // the task queue
    std::queue< std::function<void()> > tasks;
    
    // synchronization
    std::mutex queue_mutex;
    std::condition_variable condition;
    bool stop;
};
 
// the constructor just launches some amount of workers
inline ThreadPool::ThreadPool(size_t threads)
    :   stop(false)
{
    for(size_t i = 0;i<threads;++i)
        workers.emplace_back(
            [this]
            {
                for(;;)
                {
                    std::function<void()> task;

                    {
                        std::unique_lock<std::mutex> lock(this->queue_mutex);
                        this->condition.wait(lock,
                            [this]{ return this->stop || !this->tasks.empty(); });
                        if(this->stop && this->tasks.empty())
                            return;
                        task = std::move(this->tasks.front());
                        this->tasks.pop();
                    }

                    task();
                }
            }
        );
}

// add new work item to the pool
template<class F, class... Args>
auto ThreadPool::enqueue(F&& f, Args&&... args) 
    -> std::future<typename std::result_of<F(Args...)>::type>
{
    using return_type = typename std::result_of<F(Args...)>::type;

    auto task = std::make_shared< std::packaged_task<return_type()> >(
            std::bind(std::forward<F>(f), std::forward<Args>(args)...)
        );
        
    std::future<return_type> res = task->get_future();
    {
        std::unique_lock<std::mutex> lock(queue_mutex);

        // don't allow enqueueing after stopping the pool
        if(stop)
            throw std::runtime_error("enqueue on stopped ThreadPool");

        tasks.emplace([task](){ (*task)(); });
    }
    condition.notify_one();
    return res;
}

// the destructor joins all threads
inline ThreadPool::~ThreadPool()
{
    {
        std::unique_lock<std::mutex> lock(queue_mutex);
        stop = true;
    }
    condition.notify_all();
    for(std::thread &worker: workers)
        worker.join();
}

#endif
```
example:
```cpp
#include <iostream>
#include <vector>
#include <chrono>

#include "ThreadPool.h"

int main()
{
    
    ThreadPool pool(4);
    std::vector< std::future<int> > results;

    for(int i = 0; i < 8; ++i) {
        results.emplace_back(
            pool.enqueue([i] {
                std::cout << "hello " << i << std::endl;
                std::this_thread::sleep_for(std::chrono::seconds(1));
                std::cout << "world " << i << std::endl;
                return i*i;
            })
        );
    }

    for(auto && result: results)
        std::cout << result.get() << ' ';
    std::cout << std::endl;
    
    return 0;
}
```