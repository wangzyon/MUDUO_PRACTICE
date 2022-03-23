# 1 thread

```cpp
#include <thread>     //对windows或linux线程库（pthread）的封装，语言级别的跨平台

std::thread t(func);  //线程创建并启动
t.join();             //线程阻塞
t.detach();           //线程分离

std::this_thread::get_id();                            //获取当前线程id
std::this_thread::sleep_for(std::chrono::seconds(2));  //睡眠
std::this_thread::yield()                              //出让，转让线程使用权，等待CPU下一次调度
```

# 2 mutex 

0. 多线程编程有两个问题：

```
1.线程互斥：临界代码段->互斥->加锁（软件加锁-mutex,硬件加锁CAS）
临界代码段：必须确保只有一个线程能够进入的代码段，否则将无法保证多线程竞态条件（不会随CPU调用线程顺序，而产生不同的运行结果）'
另一种问法：为什么需要互斥？
多线程编程中，有时候需要保证部分代码同一时刻只能由一个线程执行，否则会造成运行结果不一致，即违反竞态条件；实现唯一线程运行临界区代码的方法就是互斥->互斥可以通过加锁->软件加锁-mutex,硬件加锁CAS->无锁atomic

2.线程同步通信:两个线程存在相互依赖关系，即一个线程的计算依赖另一个线程的计算结果
```

1. mutex: 互斥锁

```cpp
std::mutex mtx; //创建一个全局锁

void func(){
	while(count>0){
		mtx.lock();  #1
		count--;
		mtx.unlock();
	}
}
```

> 需要手动加锁和释放锁，代码逻辑错误、程序异常、忘记释放均会导致死锁（所有线程都拿不到锁，程序卡死）

2. lock_guard

```cpp
//优势：lock_guard：自动加锁释放锁，避免死锁
//缺点：lock_guard内部使用scope_ptr智能指针，虽然实现了对象管理，但由于scope_ptr关闭拷贝构造和赋值函数
//因此，lock_guard不能用于函数调用和返回值，只能应用于当前代码段

void func(){
	while(count>0){
		lock_guard<std::mutex> lock(mtx)   //出了作用域，lock_guard对象析构，会自动调用unlock释放锁
        /*
        临界区二次判断，因为锁在临界区判断while(count>0)的内部
        当count=1时，允许多个线程进入到1处，但其中一个线程拿到锁后，将count置为0,逻辑上应停止循环，
        但其他线程仍然会在0基础上进行减一操作；
        */
        if(count>0) count--; 
	}
}
```

3. unique_lock

```cpp
//unique_lock内部使用unique_ptr智能指针，实现了对象管理
//unique_lock相比于lock_guard，其内部左值拷贝和赋值关闭，但是右值拷贝构造和赋值是开启的，因此可以用于函数调用和返回值

void func(){
	while(count>0){
		unique_lock<std::mutex> lock(mtx)   //除了作用域，unique_lock对象析构，会自动调用unlock释放锁
        if(count>0) count--;
	}
}
```

# 3 condition_var

`condition_variable` ：和线程互斥锁共同使用，实现线程同步通信

```cpp
#include <iostream>
#include <thread>
#include <atomic>
#include <queue>
#include <mutex>
#include <condition_variable>

std::mutex mtx;
std::condition_variable cv;  //条件变量

class Queue
{
public:
    //通过加锁实现同一时刻只有一个线程可以对Queue进行put或get操作，保证线程安全
    void put(int val)
    {
        std::unique_lock<std::mutex> lock(mtx);
        if (!que.empty())
        {
            /*
            队列里还有未消费的对象，wait包含两个操作
            1.当前线程等待
            2.释放锁给其他线程使用（如给消费者线程消费），由于队列里有对象，消费线程不会进入同样的wait状态
            
            关键：wait是等待状态，需要其他线程notify唤醒进入阻塞状态，阻塞不能开始执行，需要拿到锁；等待不等于阻塞；
            等待->阻塞->取锁
            */
            cv.wait(lock); 
        }
        que.push(val);
        cv.notify_all(); //通知所有其他处于wait的线程，将他们的等待状态唤醒为阻塞状态
        //cv.notify_one通知指定线程
    }//出作用域，unique_lock线程锁被析构释放，此时notify唤醒的其他阻塞线程开始竞争锁执行
    int get()
    {
        std::unique_lock<std::mutex> lock(mtx);
        //队列元素为空，进入wait状态，释放锁，让生产线程开始生产
        //由于队列里没有对象，生产线程不会进入同样的wait状态
        if (que.empty())  
        {
            cv.wait(lock);
        }
        int val = que.front();
        que.pop();
        cv.notify_all();
        return val;
    }//出作用域，unique_lock线程锁被析构释放，此时notify唤醒的其他阻塞线程开始竞争锁执行

private:
    std::queue<int> que;
};

void producer(Queue *que)
{

    for (int i = 0; i < 10; ++i)
    {
        que->put(i);
        std::cout << "生产：" << i << std::endl;
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }
}

void consume(Queue *que)
{

    for (int i = 0; i < 10; ++i)
    {
        int val = que->get();
        std::cout << "消费：" << val << std::endl;
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }
}

int main()
{
    Queue que;
    std::thread t1(producer, &que);
    std::thread t2(consume, &que);

    t1.join();
    t2.join();
    return 0;
}
```

# 4 semaphore

semaphore是另一种同步通信方式，不需要锁，且可以进程通信；

> 相乘同步通信的例子：

```cpp
#include <semaphore.h>
#include <thread.h>

sem_t sem;
sem_init(&sem,false,0);// 第二个参数是进程通信相关，0表示信号初始值为0

t = std::thread([]()
{
	do_something(); // 一些耗时操作
	sem_post(&sem); 
})

sem_wait(&sem); // 阻塞，直到do_something运行完成，sem_post使sem+1，sem_wait接触阻塞；
```

# 5 atomic

- `atomic`是C++提供的原子操作：原子，即不能被分割，原子操作，即不能被多个线程同时执行，例如A/B线程同时执行i++操作，此时必须等一个线程完成对i的全部操作，另一个线程才能对其进行操作；

- CAS是实现原子操作的一种方式，可实现无锁互斥

> 不是软件层面无锁，而是硬件加锁，即对CPU和内存交互的系统总线加锁，从而不支持多线程读写；
>
> 为了保证不受线程栈缓存的影响，原子类型需要进一步添加**volatile**，确保不经过缓存，直接读写内存，此时子线程能立刻看到变量的更新。

- atomic提供的一些原子类型：atomic_int，atomic_char，atomic_bool
- **mutex互斥锁比较重，用于一些比较复杂的场景，atomic原子操作中的CAS是轻量级的互斥**

```cpp
#include <iostream>
#include <thread>
#include <atomic>
#include <vector>

volatile std::atomic_int cnt{0};  //直接初始化，赋值初始化会报错
volatile std::atomic_bool isReady{false};

void handle()
{
    while (!isReady)  // 未准备好，出让CPU资源
    {
        std::this_thread::yield();
    }
    for (int i = 0; i < 10; ++i)
    {
        ++cnt;
    }
}

int main()
{
    // 手动启动线程的一种方式，
    std::vector<std::thread> vthread;
    for (int i = 0; i < 10; ++i)
    {
        vthread.emplace_back(std::thread(handle));
    }

    isReady = true;

    for (int i = 0; i < 10; ++i)
    {
        vthread[i].join();
    }

    std::cout << "cnt: " << cnt << std::endl;
    return 0;
}
```

# 6 __thread 关键字

__thread关键字定义的变量每一个线程独立，各个线程的值互不干扰。

1. 应用__thread缓存每个线程的id

```cpp
#include <sys/syscall.h>

namespace CurrentThread
{
    extern __thread int t_cachedTid; // 线程独立,此处使用extern声明全局变量，被多个其他文件包含不会出现重定义；

    void cacheTid()
    {
        if (t_cachedTid == 0)
        {
            // 通过linux系统调用，获取当前线程的tid值
            t_cachedTid = static_cast<pid_t>(::syscall(SYS_gettid));
        }
    }

    inline int tid()
    {
        /*
        GCC在编译过程中，会将可能性更大的代码紧跟着前面的代码，从而减少指令跳转带来的性能上的下降, 达到优化程序的目的。
        
        if (__builtin_expect(t_cachedTid == 0, 0))语句逻辑等于if(t_cachedTid == 0)
        但__builtin_expect(t_cachedTid == 0, 0)表示t_cachedTid == 0为false(0)的期望更大，因此执行cacheTid()的可能性小；缓存只在第一次获取时调用cacheTid()，自然可能性小；提前告诉编译器，优化程序；
        */
        if (__builtin_expect(t_cachedTid == 0, 0)) 
        {
            cacheTid();
        }
        return t_cachedTid;
    }
}
```

2. 应用__thread确保每个线程只能创建一个对象

```cpp
class EventLoop{
    EventLoop()
    {
        if (t_loopInThisThread) // 当t_loopInThisThread非null时，表示当前线程已创建EventLoop对象，无法再执行构造函数
        {
            LOG_FATAL("Another EventLoop %p exists in this thread %d \n", t_loopInThisThread, threadId_);
        }
        else
        {
            t_loopInThisThread = this;
        }
    }
}

// __thread关键字将全局变量变成thread local变量 
__thread EventLoop *t_loopInThisThread = nullptr;

```