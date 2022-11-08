<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [1 & 2 Thread Management](#1-2-thread-management)
- [3 Data Race and Mutex](#3-data-race-and-mutex)
- [4 Deadlock](#4-deadlock)
- [5 Unique Lock and Lazy Initialization](#5-unique-lock-and-lazy-initialization)
- [6 Condition Variable](#6-condition-variable)
- [7 Future, Promise and async()](#7-future-promise-and-async)
- [8 Using Callable Objects.](#8-using-callable-objects)
- [9 package_task](#9-package_task)
- [10 Time Constrain](#10-time-constrain)

<!-- /code_chunk_output -->


# 1 & 2 Thread Management

```c++ {.line-numbers}
#include <thread>

void function_1() {
    std::cout << "Beauty is only skin-deep" << std::endl;
}

int main() {
    std::thread t1(function_1);  // t1 starts running.
    t1.join();  // main thread waits for t1 to finish.
    t1.detach();  // t1 will freely on its own -- daemon process

    return 0;
}
```

__notes:__
1. join() 和 detach() 只能二选一。

```c++ {.line-numbers}
#include <thread>

class Fctor {
public:
    void operator()(std::string& msg) {  // functional forms
        std::cout << "t1 says: " << msg << std::endl;
        msg = "Trust is the mother of deceit.";
    }
};

int main() {
    std::cout << std::thread::hardware_concurrency() << std::endl;

    std::string s = "Where there is no trust, there is no love";
    std::thread t1((Fctor()), std::ref(s));
    std::thread t2 = std::move(t1);
    t2.join();
    std::cout << s << std::endl;
    
    return 0;
}
```

__notes:__
1. hardware_concurrency() 能获取当前 CPU 最多支持的线程数量。
2. 类的函数调用运算符重载，也可以作为 thread 的参数；此时第二个参数需要转换为 ref() 以实现共享内存。
3. thread 只能 move() 不能 copy。

# 3 Data Race and Mutex

```c++ {.line-numbers}
#include <thread>
#include <mutex>

std::mutex mu;

void shared_print(std::string msg, int id) {
    mu.lock();
    std::cout << msg << id << std::endl;
    mu.unlock();
}

void shared_print_updated(std::string msg, int id) {
    // RAII: resource acquisition is initialization.
    std::lock_guard<std::mutex> guard(mu);
    std::cout << msg << id << std::endl;
}

void function_1() {
    for (int i = 0; i > -100; i--)
        shared_print(std::string("From t1: "), i);
}

int main()
{
    std::thread t1(function_1);
    for (int i = 0; i < 100; i++) 
        shared_print(std::string("From main: "), i);
    t1.join();

    return 0;
```

__notes:__
1. 当不同线程竞争同一块资源的时候，用互斥锁 mutex。
2. 但是如果 mutex 异常地错过 unlock()，那么 mutex 会永远锁上。我们类比 RAII 的策略：putting resource inside objects. 比如用 auto_ptr 来防止资源泄露。此处用 lock_guard 来防止错过 unlock()。

# 4 Deadlock

```c++ {.line-numbers}
#include <thread>
#include <mutex>

std::mutex mu, mu2;

void shared_print(std::string msg, int id) {
    std::lock(mu, mu2);
    std::lock_guard<std::mutex> guard(mu, std::adopt_lock);
    std::lock_guard<std::mutex> guard2(mu2, std::adopt_lock);
    std::cout << msg << id << std::endl;
}
```

__notes:__
1. 当有多把锁时，建议同时上锁。adopt_lock 是指：该锁已经上锁，lock_guard 只需记得开锁。

# 5 Unique Lock and Lazy Initialization

```c++ {.line-numbers}
#include <thread>
#include <mutex>

std::mutex mu;

void shared_print(std::string msg, int id) {
    std::unique_lock<std::mutex> locker(mu, std::defer_lock);
    locker.lock();
    //locker.try_lock();
    std::cout << msg << id << std::endl;
    locker.unlock();
    // do something else...
    locker.lock();

    std::unique_lock<std::mutex> locker2 = std::move(locker);
}
```

__notes:__
1. unqiue_lock 拥有更好的灵活性，可以多次 lock() 和 unlock()。
2. unique_lock 可以 move()。
3. defer_lock 是指：暂时不上锁。
4. try_lock() 更加鲁棒。

```c++ {.line-numbers}
#include <thread>
#include <mutex>

class LogFile {
    std::mutex _mu;
    //std::mutex _mu_open;
    std::once_flag _flag;
    std::ofstream _f;
public:
    LogFile() {	}
    void shared_print(std::string id, int value) {
        /*{
            std::unique_lock<std::mutex> locker2(_mu_open);
            if (!_f.is_open())
                _f.open("log.txt");
        }*/
        // file will be opened only once by one thread.
        std::call_once(_flag, [&]() {_f.open("log.txt"); });

        std::unique_lock<std::mutex> locker(_mu);
        _f << "From" << id << ": " << value << std::endl;
    }
};
```

__notes:__
1. 如果某项操作只需要被任一线程执行一次，可以用 call_once() 搭配 once_flag，以提升效率。

# 6 Condition Variable

```c++ {.line-numbers}
# include <mutex>

std::deque<int> q;
std::mutex mu;
std::condition_variable cond;

void function_1() {
    int count = 10;
    while (count > 0) {
        std::unique_lock<std::mutex> locker(mu);
        q.push_front(count);
        locker.unlock();
        cond.notify_one();  // Notify one waiting thread, if there is one.
        //cond.notify_all();
        std::this_thread::sleep_for(std::chrono::seconds(1));
        //std::chrono::steady_clock::time_point tp = 
        //    std::chrono::steady_clock::now() + std::chrono::microseconds(4);
        //std::this_thread::sleep_until(tp);
        count--;
    }
}

void function_2() {
    int data = 0;
    while (data != 1) {
        std::unique_lock<std::mutex> locker(mu);
        cond.wait(locker, []() {return !q.empty(); });  // spurious wake
        data = q.back();
        q.pop_back();
        locker.unlock();
        std::cout << "t2 got a value from t1: " << data << std::endl;   
    }
}

int main()
{
    std::thread t1(function_1);
    std::thread t2(function_2);
    t1.join();
    t2.join();
 
    return 0;
}
```

__notes:__
1. condition_variable 可以控制 2 号线程睡眠，等待 1 号线程完成任务。
2. notify_one() 提醒某一线程醒来。notify_all() 提醒所有线程醒来。
3. wait() 进入睡眠。需要一个 unique_lock 是因为：入睡前要解锁，醒来后要上锁。第二个参数用于醒来后判断是否是“虚拟唤醒”，如果得到 0，则继续入睡。
4. 注意 sleep_for(), sleep_until(), chrono 的用法。

# 7 Future, Promise and async()

```c++ {.line-numbers}
#include <future>

int factorial(int N) {
    int res = 1;
    for (int i = N; i > 1; i--)
        res *= i;
    std::cout << "Result is: " << res << std::endl;
    return res;
}

int main()
{
    std::future<int> fu = std::async(std::launch::async, factorial, 4);
    //std::future<int> fu = std::async(std::launch::deferred, factorial, 4);
    int x = fu.get();
    
    return 0;
}
```

__notes:__
1. future 和 async 搭配，能够方便地获取函数的返回值。参数 launch::async 是指：开启一个新的线程。参数 launch::deferred 是指：推迟 factorial 函数的执行，直到 get() 之前，并且在 get() 的线程中执行 factorial 函数。
2. future 对象只能调用一次 get()。

```c++ {.line-numbers}
#include <future>
#include <mutex>

std::mutex mu;

int factorial(std::shared_future<int> sf) {
    int res = 1;
    int N = sf.get();
    for (int i = N; i > 1; i--)
        res *= i;
    std::lock_guard<std::mutex> guard(mu);
    std::cout << "Result is: " << res << std::endl;
    return res;
}

int main()
{
    std::promise<int> p;
    std::future<int> f = p.get_future();
    std::shared_future<int> sf = f.share();
    std::future<int> fu = std::async(std::launch::async, factorial, sf);
    std::future<int> fu2 = std::async(std::launch::async, factorial, sf);
    // do something else...
    p.set_value(4);

    return 0;
}
```

__notes:__
1. future 和 promise 搭配，能够在未来给函数传参数。
2. shared_future 能够解决 future.get() 只能执行一次的问题。
3. 有三种获取 future 的方式：promise::get_future()；async() 返回 future；packaged_task::get_future()。

# 8 Using Callable Objects.

```c++ {.line-numbers}
# include <thread>

class A {
public:
    void f(int x, char c) {};
    int operator()(int N) { return 0; }
};

void foo(int x) {}

int main()
{
    A a;
    std::thread t1(a, 6);  // copy_of_a() in a different thread.
    std::thread t2(std::ref(a), 6);  // a() in a different thread.
    std::thread t3(A(), 6);  // temp A.
    std::thread t4([](int x) {return x * x; }, 6);
    std::thread t5(foo, 7);
    std::thread t6(&A::f, a, 8, 'w');  // copy_of_a.f(8, 'w') in a different thread.
    std::thread t7(&A::f, &a, 8, 'w');  // a.f(8, 'w') in a different thread.
    t1.join();
    t2.join();
    t3.join();
    t4.join();
    t5.join();
    t6.join();
    t7.join();

    return 0
}
```

__notes:__
1. std::bind(), std::async(), std::call_once 也有相同的用法。

# 9 package_task

# 10 Time Constrain

