[TOC]

---

## 客户端 <--> 服务端

> 若干个客户端和一个服务端进行TCP交互。

- 每**一个**客户端和服务端的**连接** 对应于 **一个线程**；

  <img src="Reactor_pic\image-20230528223118832.png" alt="image-20230528223118832" style="zoom: 67%;" />

  > 当客户端特别少的情况下，该方案可以正常运行；但是当客户端连接比较多时（比如有10万个连接），会带来一些问题： 
  >
  > - 一是，线程处理完业务逻辑之后就会随着连接的关闭使得线程被销毁，因此不断地进行线程地创建和销毁会带来**性能开销和资源地浪费**；
  > - 二是，创建对应连接数目的线程是**不现实的**。

- 为了解决该问题，**引入线程池**，通过**资源复用**的方式解决。

  > 此时，线程池中的**一个线程** 负责 **多个连接** 的业务处理。 
  >
  > 在之前一个线程对应一个连接的时候，该线程采用 "read->业务处理->send" 的处理流程，若当前的连接没有数据可读时，线程会阻塞。
  >
  > 那么对于此时而言，若还是采用上述处理流程，则一个线程中多个连接的处理是“串行”的，即线程因为前面的连接而发生阻塞时，该连接之后的所有连接都无法处理。

- 解决方案：

  - 将**socket**改为 **非阻塞的**，然后线程 **不断地轮询** 调用read来进行判断是否有数据。

    > 该方案可以解决，但是由于线程不断地轮询操作是需要**消耗CPU**的，而且随着一个线程处理的连接数目增加时，**轮询的效率也会降低**。

  - 引入I/O多路复用。

    > 将上述轮询判断操作交给OS内核去完成。当事件就绪时才会向应用进程进行通知。

---

---



## Reactor

### 1. 单Reactor单线程

![SingleReactorSingleThread](Reactor_pic\SingleReactorSingleThread.png)

> - Reactor 对象 通过 select 监听事件，收到事件的通知后通过 dispatch 进行分发，具体的分发是根据事件类型来将事件发生给 Acceptor 对象或 Handler 对象。
> - 若该事件是连接建立的事件，则交由Acceptor对象进行处理。(Accept对象通过 accept获取新连接，并为之创建一个Handle对象来处理后续的响应事件）
> - 否则，则交由当前连接的对应的 Handle对象 进行响应。(Handle对象 进行I/O以及 相关的业务处理)

---

### 2. 单Reactor多线程

![image-20230528164041933](Reactor_pic\SingleReactorMultiThread.png)

> - 此时，与但Reactor单线程相比， Handle 不再负责业务的处理工作， 而是将 业务处理工作 交由线程池去完成，自己只负责数据的接收操作。 
> - 线程池处理完业务之后，将数据发送给 Hanle 对象，由 Handle 对象进行发送给客户端。

---

### 3. 多Reactor多线程

![MutiReactorMultiThread](Reactor_pic\MutiReactorMultiThread.png)

> - 主线程 中的 `MainReactor` 对象通过 `select` 监听 **连接建立事件**， 收到事件通知后通过 `Acceptor` 对象中的 accept方法来获取 新连接，将新连接分配给某个子线程。
> - 子线程 中的 `SubReactor` 对象将从 MainReactor对象分配的新连接 加入 select 继续进行监听，并创建一个 `Handle` 用于处理连接的响应事件。
> - 如果有新的事件发生（不是连接建立事件），则 SubReactor 对象会调用当前连接对应的 Handle 对象来进行响应。
> - Handle对象通过 “read->业务处理->send" 的流程来完成完整的业务流程。（当然，可以将该业务的处理交由线程池去完成）



![image-20230528223353149](Reactor_pic\image-20230528223353149.png)

![image-20230528223458785](Reactor_pic\image-20230528223458785.png)

![image-20230528223544685](Reactor_pic\image-20230528223544685.png)

![image-20230528223741061](Reactor_pic\image-20230528223741061.png)

> 如果业务逻辑本身是阻塞的，在将线程占满的情况下，有可能导致“雪崩效应”。

---



## Proactor

![Proactor](\Reactor_pic\Proactor.png)

> - `Proactor initiator` 负责创建 `Proactor` 和 `Handle` 对象， 并将 `Proactor` 和 `Handle` 都通过 `Asynchronous Operation Proccessor` 注册到内核。
>
> - `Asynchronous Operation Proccessor` 负责处理注册请求，并处理I/O操作。
> - `Asynchronous Operation Proccessor` 完成I/O操作之后通知 `Proactor`。
>
> - `Proactor` 根据 不同的事件类型 来**回调**不同的 Handle 来进行业务的处理。
> - `Handle` 完成业务的处理。

---

---





## 总结

`Reactor`，`Proactor` 都是一种 **基于事件分发** 的网络编程模式，区别在于Reactor是基于 **待完成** 的IO事件，Proactor是基于 **已完成** 的IO事件。

> `Reacotr` 是 **同步** **非阻塞** 网络模式，感知的是 **就绪的可读事件**；
>
> > 由于Reactor需要通过IO多路复用机制去监听TCP连接，当监听的文件描述符就绪之后，Reactor就将数据从socket接收缓存中读取到应用进程的内存之中。这个过程是同步的，读取完数据之后应用进程才能对数据进行处理。
> >
> > 由于Reactor不断地进行epoll_wait的循环监听，因此处理Reactor的进程不会因为读事件没有就绪而发生阻塞，因此可以将其认为是非阻塞（事实上在epoll_wait的时候还是会发生阻塞）。
>
> `Proactor` 是 **异步** 网络模式，感知的是 **已完成的读写事件**；
>
> > 在发起异步读写请求时，需要传入数据缓冲区的地址（用来存放结果数据等）等信息，这样OS内核才可以完成数据的读写工作。在os内核完S成读写操作后，会向应用进程**发送一个通知**来通知应用进程处理数据。
>
> >**Reactor**: 来了 **IO事件**（有新连接、有数据可读、有数据可写），操作系统通知应用进程，让应用进程来处理。
> >
> >**Proactor**: 来了 **IO事件**，操作系统进行处理，处理完之后再通知应用进程。

---





## Linux下的异步I/O

<img src="Reactor_pic\image-20230528223850680.png" alt="image-20230528223850680" style="zoom:80%;" />

> `Linux` 下的 异步I/O 是**不完善的**， `aio` 系列函数是由 **POSIX** 定义的异步操作接口， **不是真正的OS级别支持的**， 而是在用户空间**模拟**出来的异步， 并且仅仅支持 **基于本地文件** 的`aio`异步操作， 对于网络编程中的 socket 是不支持的，这使得 基于Linux的高性能网络程序 都是使用Reactor方案的。

---

## 运用

开源软件 **`Netty`** , **`Memcache`** 都是采用了 **多Reactor多线程** 方案的。

**`Nginux`** 采用了 **多Reactor多进程** 方案，不过该方案 和标准的方案有些差异。（表现在：主进程 **仅仅是用来初始化**socket的，并没有创建 `MainReactor` 来进行 accept 连接， 而是由子进程的 `SubReactor` 来进行 accept 连接，通过**锁**来控制一次仅有一个子进程进行accept（防止出现**惊群现象**）, 子进程 `accpet` 新连接后就放到自己的 `SubReactor` 进行处理，不会将其再分配给其他子进程。）

> 惊群现象：多进程/线程 在同时阻塞等待同一个事件时，若该事件发生，那么就会唤醒等待的所有进程/线程；但是最终只可能有一个进程/线程获得“控制权”，而其他的进程/线程将会失败，从而只能重新进入等待状态。这种现象和性能的浪费就称为惊群。
>
> ![
>
> ](Reactor_pic\image-20230528224036232.png)

![image-20230528224154669](Reactor_pic\image-20230528224154669.png)

> 使用`EPOLLEXCLUSIVE`时，只有一个线程能够处理监听同一个文件描述符就绪事件。
>
> - 在不同OS上实现可以有所差异
> - 使用其，可能会导致负载不均衡的问题。
> - 并不能完全消除惊群问题。

---



[(3 封私信) 如何深刻理解Reactor和Proactor？ - 知乎 (zhihu.com)](https://www.zhihu.com/question/26943938)









# -----------------

## Implement

### Thread

#### OO

##### Thread.hpp

```c++
#ifndef __THREAD_HPP__
#define __THREAD_HPP__

#include<pthread.h>
class Thread {
public:
    Thread();
    virtual ~Thread();
    void StartUp();
    void Join();
private:
    virtual void run() = 0;
    static void* threadFun(void * arg);
private:
    pthread_t _tId;
    bool _isRunning;
};

#endif
```

##### Thread.cc

```c++
#include"Thread.hpp"
#include<string.h>
#include<iostream>
using std::cerr;
using std::endl;

Thread::Thread() 
: _tId(0)
, _isRunning(false)
{}
Thread::~Thread() {

}
void Thread::StartUp() {
    int res = pthread_create(&_tId,nullptr,threadFun,this);
    if(res != 0) {
        cerr << "[pthread_create in StartUp] " << strerror(res) << endl;
        return;
    }
    _isRunning = true;
}
void Thread::Join() {
    int res = pthread_join(_tId,nullptr);
    if(res != 0) {
        cerr << "[pthread_join in Join] " << strerror(res) << endl;
        return;
    }
    _isRunning = false;
}
void* Thread::threadFun(void * arg) {
    Thread* p = static_cast<Thread*>(arg);
    if(p) {
        p->run();
    }
    pthread_exit(nullptr);
}

```



#### BO

##### Thread.hpp

```c++
#ifndef __THREAD_HPP__
#define __THREAD_HPP__

#include<functional>
#include<pthread.h>
class Thread {
    using CallBackFunc = std::function<void()>;
public:
    Thread(CallBackFunc && cb);
    virtual ~Thread();
    void StartUp();
    void Join();
private:
    static void* threadFun(void * arg);
private:
    pthread_t _tId;
    bool _isRunning;
    CallBackFunc _cb;
};

#endif
```

##### Thread.cc

```c++
#include"Thread.hpp"
#include<string.h>
#include<iostream>
using std::cerr;
using std::endl;

Thread::Thread(CallBackFunc && cb) 
: _tId(0)
, _isRunning(false)
, _cb(std::move(cb))
{}
Thread::~Thread() {

}
void Thread::StartUp() {
    int res = pthread_create(&_tId,nullptr,threadFun,this);
    if(res != 0) {
        cerr << "[pthread_create in StartUp] " << strerror(res) << endl;
        return;
    }
    _isRunning = true;
}
void Thread::Join() {
    int res = pthread_join(_tId,nullptr);
    if(res != 0) {
        cerr << "[pthread_join in Join] " << strerror(res) << endl;
        return;
    }
    _isRunning = false;
}
void* Thread::threadFun(void * arg) {
    Thread* p = static_cast<Thread*>(arg);
    if(p) {
        p->_cb();
    }
    pthread_exit(nullptr);
}
```

---



### Producer_Consumer

#### OO

![image-20230528220139939](C:\Users\Administrator\Desktop\cpp_advance\Reactor_pic\image-20230528220139939.png)

##### MutexLock.hpp

```c++
#ifndef __MUTEXLOCK_HPP__
#define __MUTEXLOCK_HPP__

#include"NonCopyable.hpp"
#include<pthread.h>
class MutexLock 
:public NonCopyable
{
public:
    MutexLock();
    ~MutexLock();
    void Lock();
    void TryLock();
    void UnLock();
    pthread_mutex_t* getMutexPtr();
private:
    pthread_mutex_t _mutex;
};

class MutexLockGuard {
public:
    MutexLockGuard(MutexLock &mutex)
    : _mutex(mutex)
    {   _mutex.Lock();  }
    ~MutexLockGuard() {
        _mutex.UnLock();
    }
private:
    MutexLock & _mutex;
};

#endif
```

##### Condition.hpp

```c++
#ifndef __CONDITION_HPP__
#define __CONDITION_HPP__

#include"NonCopyable.hpp"
#include<pthread.h>
class MutexLock;
class Condition 
: public NonCopyable
{
public:
    Condition(MutexLock &mutex);
    ~Condition();
    void Wait();
    void Notify();
    void NotifyAll();
private:
    pthread_cond_t _condition;   
    MutexLock & _mutex;
};

#endif
```

##### TaskQueue.hpp

```c++
#ifndef __TASKQUEUE_HPP__
#define __TASKQUEUE_HPP__

#include"NonCopyable.hpp"
#include<queue>
#include"MutexLock.hpp"
#include"Condition.hpp"
class TaskQueue 
: public NonCopyable
{
public:
    TaskQueue(std::size_t queueCapacity);
    ~TaskQueue();
    bool Empty() const;
    bool Full() const;
    void Push(const int & val);
    int Pop();
private:
    std::size_t _queueCapacity;
    std::queue<int> _que;
    MutexLock _mutex;
    Condition _notFull;
    Condition _notEmpty;
};

#endif
```

##### NonCopyable.hpp

```c++
#ifndef __NONCOPYABLE_HPP__
#define __NONCOPYABLE_HPP__

class NonCopyable {
public:
    NonCopyable(){}
    ~NonCopyable(){}
    NonCopyable(const NonCopyable &) = delete;
    NonCopyable & operator=(const NonCopyable&) = delete;
};

#endif
```



##### MutexLock.cc

```c++
#include"MutexLock.hpp"
#include<stdio.h>
MutexLock::MutexLock() {
    pthread_mutex_init(&_mutex,nullptr);    //always success
}
MutexLock::~MutexLock() {
    int res = pthread_mutex_destroy(&_mutex);
    if(res) {
        perror("MutexLock_Destroy");
        return;
    }
}
void MutexLock::Lock() {
    int res = pthread_mutex_lock(&_mutex);
    if(res) {
        perror("MutexLock_Lock");
        return;
    }
}
void MutexLock::TryLock() {
    int res = pthread_mutex_trylock(&_mutex);
    if(res) {
        perror("MutexLock_TryLock");
        return;
    }
}
void MutexLock::UnLock() {
    int res = pthread_mutex_unlock(&_mutex);
    if(res) {
        perror("MutexLock_UnLock");
        return;
    }

}
pthread_mutex_t* MutexLock::getMutexPtr() {
    return &_mutex;
}
```

##### Condition.cc

```c++
#include"Condition.hpp"
#include"MutexLock.hpp"
#include<stdio.h>

Condition::Condition(MutexLock &mutex) 
: _mutex(mutex)
{
    int res = pthread_cond_init(&_condition,nullptr);
    if(res) {
        perror("Condition_Init");
        return;
    }
}
Condition::~Condition() {
    int res = pthread_cond_destroy(&_condition); 
    if(res) {
        perror("Condition_Destroy");
        return;
    }
}
void Condition::Wait() {
    int res = pthread_cond_wait(&_condition,_mutex.getMutexPtr()); 
    if(res) {
        perror("Condition_Wait");
        return;
    }
}
void Condition::Notify() {
    int res = pthread_cond_signal(&_condition); 
    if(res) {
        perror("Condition_Signal");
        return;
    }
}
void Condition::NotifyAll() {
    int res = pthread_cond_broadcast(&_condition); 
    if(res) {
        perror("Condition_BroadCast");
        return;
    }
}
```

##### TaskQueue.cc

```c++
#include"TaskQueue.hpp"

TaskQueue::TaskQueue(std::size_t queueCapacity) 
: _queueCapacity(queueCapacity)
, _que()
,_mutex()
,_notFull(_mutex)
,_notEmpty(_mutex)
{}
TaskQueue::~TaskQueue() {

}
bool TaskQueue::Empty() const {
    return _que.size() == 0;
}
bool TaskQueue::Full() const {
    return _que.size() == _queueCapacity;
}
void TaskQueue::Push(const int & val) {
    /* _mutex.Lock(); */
    MutexLockGuard mutexGuard(_mutex);
    while(Full()) {
        _notFull.Wait();
    }
    _que.push(val);
    _notEmpty.Notify();
    /* _mutex.UnLock(); */
}
int TaskQueue::Pop() {
    /* _mutex.Lock(); */
    MutexLockGuard mutexGuard(_mutex);
    while(Empty()) {
        _notEmpty.Wait();
    }
    int tmp = _que.front();
    _que.pop();
    _notFull.Notify();
    /* _mutex.UnLock(); */
    return tmp;
}
```

##### Producer.hpp

```c++
#ifndef __PRODUCER_HPP__
#define __PRODUCER_HPP__

#include<iostream>
#include"Thread.hpp"
#include"TaskQueue.hpp"
#include<stdlib.h>
#include<time.h>
#include<unistd.h>
#include"MutexLock.hpp"

class Producer 
: public Thread
{
public:
   Producer(TaskQueue & taskque, MutexLock & mutex)
    : _taskque(taskque)
    , _mutex(mutex)
    {}
    ~Producer(){}
    void run() override;
private:
    TaskQueue &_taskque;
    MutexLock &_mutex;
};

void Producer::run() {
    srand(clock());
    int cnt = 20;
    while(cnt--) {
        int tmp = rand() % 100;
        _taskque.Push(tmp);
        _mutex.Lock();
        std::cout << "Producer: product " << tmp << std::endl;
        _mutex.UnLock();
        sleep(1);
    }
}

#endif
```

##### Customer.hpp

```c++
#ifndef __CUSTOMER_HPP__
#define __CUSTOMER_HPP__

#include<iostream>
#include"Thread.hpp"
#include"TaskQueue.hpp"
#include<unistd.h>
#include"MutexLock.hpp"
class Customer 
: public Thread
{
public:
    Customer(TaskQueue & taskque, MutexLock & mutex)
    : _taskque(taskque)
    , _mutex(mutex)
    {}
    ~Customer(){}
    void run() override;
private:
    TaskQueue &_taskque;
    MutexLock & _mutex;
};

void Customer::run() {
    int cnt = 20;
    while(cnt--) {
        int tmp = _taskque.Pop();
        _mutex.Lock();
        std::cout << "Customer: consume " << tmp << std::endl;
        _mutex.UnLock();
        sleep(1);
    }
}

#endif
```

##### test_file

```c++
#include<iostream>
#include"TaskQueue.hpp"
#include"Customer.hpp"
#include"Producer.hpp"
#include"MutexLock.hpp"
using std::cout;
using std::endl;

void test(){
    TaskQueue taskque(10);
    MutexLock mutex;
    Customer consumer(taskque, mutex);
    Producer producer(taskque, mutex);
    
    producer.StartUp();
    consumer.StartUp();

    consumer.Join();
    producer.Join();
}
int main(void){
    test();
    return 0;
}
```



#### BO

<img src="C:\Users\Administrator\Desktop\cpp_advance\Reactor_pic\image-20230528220208277.png" alt="image-20230528220208277" style="zoom:80%;" />

##### Customer.hpp

```C++
#ifndef __CUSTOMER_HPP__
#define __CUSTOMER_HPP__

#include<iostream>
#include"TaskQueue.hpp"
#include<unistd.h>
#include"MutexLock.hpp"
class Customer 
{
public:
    Customer(TaskQueue & taskque, MutexLock &mutex)
    : _taskque(taskque)
    , _mutex(mutex)
    {}
    ~Customer(){}
    void run();
private:
    TaskQueue &_taskque;
    MutexLock &_mutex;
};

void Customer::run() {
    int cnt = 20;
    while(cnt--) {
        int tmp = _taskque.Pop();
        //<<是运算符重载函数，因此不是原子操作，在多线程的情况下需要进行加锁
        _mutex.Lock();
        std::cout << "Customer: consume " << tmp << std::endl;
        _mutex.UnLock();
        sleep(1);
    }
}

#endif
```

##### Producer.hpp

```c++
#ifndef __PRODUCER_HPP__
#define __PRODUCER_HPP__

#include<iostream>
#include"TaskQueue.hpp"
#include<stdlib.h>
#include<time.h>
#include<unistd.h>
#include"MutexLock.hpp"

class Producer 
{
public:
   Producer(TaskQueue & taskque, MutexLock &mutex)
    : _taskque(taskque)
    , _mutex(mutex)
    {}
    ~Producer(){}
    void run();
private:
    TaskQueue &_taskque;
    MutexLock &_mutex;
};

void Producer::run() {
    srand(clock());
    int cnt = 20;
    while(cnt--) {
        int tmp = rand() % 100;
        _taskque.Push(tmp);
        //<<是运算符重载函数，因此不是原子操作，在多线程的情况下需要进行加锁
        _mutex.Lock();
        std::cout << "Producer: product " << tmp << std::endl;
        _mutex.UnLock();
        sleep(1);
    }
}

#endif
```

##### test_file

```c++
#include<iostream>
#include"TaskQueue.hpp"
#include"Thread.hpp"
#include"Customer.hpp"
#include"Producer.hpp"
#include<functional>
#include"MutexLock.hpp"
using std::cout;
using std::endl;

void test(){
    MutexLock mutex;
    TaskQueue taskque(10);
    Customer consumer(taskque, mutex);
    Producer producer(taskque, mutex);
    Thread myThread1(std::bind(&Customer::run,consumer));
    Thread myThread2(std::bind(&Producer::run,producer));
    
    myThread1.StartUp();
    myThread2.StartUp();

    myThread2.Join();
    myThread1.Join();
}
int main(void){
    test();
    return 0;
}
```

---







### ThreadPool

#### OO

![image-20230528220020784](C:\Users\Administrator\Desktop\cpp_advance\Reactor_pic\image-20230528220020784.png)

##### ThreadPool.hpp

```c++
#ifndef __THREADPOOL_HPP__
#define __THREADPOOL_HPP__

#include"TaskQueue.hpp"
#include"Thread.hpp"
#include"Task.hpp"
#include<vector>
#include<memory>
#include"MutexLock.hpp"
using std::vector;
using std::unique_ptr;

class ThreadPool {
    friend class WorkThread;
public:
    ThreadPool(size_t queueSize, size_t threadNum);
    ~ThreadPool();
    void Start();
    void Stop();
    void AddTask(Task*);
private:
    Task* GetTask();
    void threadFunc();
private:
    size_t _threadsNum;
    size_t _queueSize;
    vector<unique_ptr<Thread>> _threads;
    TaskQueue<Task*> _taskQueue;
    bool _isExit;
};
#endif
```

##### WorkThread.hpp

```c++
#ifndef __WORKTHREAD_HPP__
#define __WORKTHREAD_HPP__

#include"Thread.hpp"
class ThreadPool;
class WorkThread 
: public Thread
{
public:
    WorkThread(ThreadPool & threadPool);
    ~WorkThread();
    void run() override;
private:
    ThreadPool & _threadPool;
};

#endif
```

##### Task.hpp

```c++
#ifndef __TASK_HPP__
#define __TASK_HPP__

class Task {
public:
    virtual void process() = 0;
    virtual ~Task(){}
};

#endif
```

##### MyTask.hpp

```c++
#ifndef __MYTASK_HPP__
#define __MYTASK_HPP__

#include"Task.hpp"
class MyTask 
: public Task
{
public:
    MyTask();
    void process() override;
    ~MyTask();
};
#endif
```

##### ThreadPool.cc

```c++
#include"ThreadPool.hpp"
#include"WorkThread.hpp"
#include<unistd.h>

#include<iostream>
using std::cout;
using std::endl;

ThreadPool::ThreadPool(size_t queueSize, size_t threadNum) 
: _threadsNum(threadNum)
, _queueSize(queueSize)
, _taskQueue(_queueSize)
, _isExit(false)
{
}
ThreadPool::~ThreadPool() {
}
void ThreadPool::Start() {
    //预留空间
    _threads.reserve(_threadsNum);
    //创建线程池
    for(size_t i = _threadsNum; i > 0; --i) {
        _threads.emplace_back(new WorkThread(*this));
    }
    //启动线程池中的线程
    for(auto &i : _threads) {
        i->StartUp();
    }
}
void ThreadPool::Stop() {
    //保证所有的任务都执行完
    while(!_taskQueue.Empty()) {
        sleep(1);
    }
    //退出线程池
    _isExit = true;
    //将等待线程唤醒
    _taskQueue.WaitUpAllThreadOn_notEmpty();
    //
    for(auto &i : _threads) {
        i->Join();
    }
}
void ThreadPool::AddTask(Task* pTask) {
    //增加一个任务
    _taskQueue.Push(pTask);
}
Task* ThreadPool::GetTask() {
    //获得一个任务
    return _taskQueue.Pop();
}
void ThreadPool::threadFunc() {
    //工作线程循环执行
    //当任务工作process执行速度很快时，会在_isExit=true之前再次进入循环，使得该线程在GetTask()处处于等待状态。
    //解决办法：
    // 1. 添加sleep机制使得process的执行速度变慢
    // 2. 在释放线程池时，需要将所有由于过快导致等待的线程唤醒，由于唤醒之后，由于虚假唤醒的机制使得无法达到目的，因此增加标记flag
    while(!_isExit) {
        Task *pTask = GetTask();
        if(pTask) { 
            pTask->process();
        }
    }
}
```

##### WorkThread.cc

```c++
#include"WorkThread.hpp"
#include"ThreadPool.hpp"
WorkThread::WorkThread(ThreadPool & threadPool)
: _threadPool(threadPool)
{}

WorkThread::~WorkThread()
{}

void WorkThread::run() {
    _threadPool.threadFunc();
}
```

##### MyTask.cc

```c++
#include"MyTask.hpp"
#include<iostream>
#include<stdlib.h>
using std::cout;
using std::endl;

MyTask::MyTask()
{ 
    srand(clock());
}
MyTask::~MyTask()
{}
void MyTask::process() {
    cout << "MyTask: randNumber = " << rand()%100 << endl;
}
```

```c++
#ifndef __TASKQUEUE_HPP__
#define __TASKQUEUE_HPP__

#include"NonCopyable.hpp"
#include<queue>
#include"MutexLock.hpp"
#include"Condition.hpp"
template<typename T>
class TaskQueue 
: public NonCopyable
{
    using ElemType = T;
public:
    TaskQueue(std::size_t queueCapacity);
    ~TaskQueue();
    bool Empty() const;
    bool Full() const;
    void Push(const ElemType & val);
    ElemType Pop();
    void WaitUpAllThreadOn_notEmpty();
private:
    size_t _queueCapacity;
    std::queue<ElemType> _que;
    MutexLock _mutex;
    Condition _notFull;
    Condition _notEmpty;
    bool _flag;
};

template<typename T>
TaskQueue<T>::TaskQueue(std::size_t queueCapacity) 
: _queueCapacity(queueCapacity)
, _que()
, _mutex()
, _notFull(_mutex)
, _notEmpty(_mutex)
, _flag(true)
{}
template<typename T>
TaskQueue<T>::~TaskQueue() {

}
template<typename T>
bool TaskQueue<T>::Empty() const {
    return _que.size() == 0;
}
template<typename T>
bool TaskQueue<T>::Full() const {
    return _que.size() == _queueCapacity;
}
template<typename T>
void TaskQueue<T>::Push(const ElemType & val) {
    /* _mutex.Lock(); */
    MutexLockGuard mutexGuard(_mutex);
    while(Full()) {
        _notFull.Wait();
    }
    _que.push(val);
    _notEmpty.Notify();
    /* _mutex.UnLock(); */
}
template<typename T>
typename TaskQueue<T>::ElemType TaskQueue<T>::Pop() {
    /* _mutex.Lock(); */
    MutexLockGuard mutexGuard(_mutex);
    while(Empty() && _flag) {
        _notEmpty.Wait();
    }
    if(!_flag) {
        return nullptr;
    }
    ElemType tmp = std::move(_que.front());
    _que.pop();
    _notFull.Notify();
    /* _mutex.UnLock(); */
    return std::move(tmp);
}
template<typename T>
void TaskQueue<T>::WaitUpAllThreadOn_notEmpty() {
    _flag = false;
    _notEmpty.NotifyAll();
}
#endif
```

##### test_file

```c++
#include<iostream>
#include"MyTask.hpp"
#include"ThreadPool.hpp"
#include<memory>
using std::unique_ptr;
using std::cout;
using std::endl;

void test(){
    std::unique_ptr<Task> pTask(new MyTask());
    ThreadPool threadPool(10,3);
    threadPool.Start();

    int cnt = 20;
    while(cnt--) {
        threadPool.AddTask(pTask.get());
    }
    threadPool.Stop();
}
int main(void){
    test();
    return 0;
}
```



#### BO

![image-20230528220053514](C:\Users\Administrator\Desktop\cpp_advance\Reactor_pic\image-20230528220053514.png)

##### TaskQueue.hpp

```c++
#ifndef __TASKQUEUE_HPP__
#define __TASKQUEUE_HPP__

#include"MutexLock.hpp"
#include"Condition.hpp"
#include<queue>
#include<functional>
class TaskQueue 
{
public:
    using ElemType = std::function<void()>;
public:
    TaskQueue(size_t queueSize);
    ~TaskQueue();
    bool Empty();
    bool Full();
    void Push(const ElemType &&);
    ElemType Pop();
    void WakeUpThreadOn_notEmpty();
private:
    TaskQueue(const TaskQueue &) = delete;
    TaskQueue& operator=(const TaskQueue &) = delete;
private:
    size_t _queueSize;
    std::queue<ElemType> _que;
    MutexLock _mutex;
    Condition _notFull;
    Condition _notEmpty;
    bool _flag;
};

#endif
```

##### TaskQueue.cc

```c++
#include"TaskQueue.hpp"

TaskQueue::TaskQueue(size_t queueSize)
: _queueSize(queueSize)
, _que()
, _mutex()
, _notFull(_mutex)
, _notEmpty(_mutex)
, _flag(true)
{}
TaskQueue::~TaskQueue() 
{}
bool TaskQueue::Empty() {
    return _que.size() == 0;
}
bool TaskQueue::Full() {
    return _que.size() == _queueSize;
}
void TaskQueue::Push(const ElemType && elem) {
    MutexLockGuard autoLock(_mutex);
    while(Full()) {
        _notFull.Wait();
    }
    _que.push(std::move(elem));
    _notEmpty.Notify();
}
typename TaskQueue::ElemType TaskQueue::Pop() {
    MutexLockGuard autoLock(_mutex);
    while(Empty() && _flag) {
        _notEmpty.Wait();
    }
    //flag是在结束后唤醒虚假等待所需的
    if (_flag) {
        ElemType tmp = _que.front();
        _que.pop();
        _notFull.Notify();
        return tmp;
    } else {
        return nullptr;
    }
}
void TaskQueue::WakeUpThreadOn_notEmpty() {
    _flag = false;
    _notEmpty.NotifyAll();
}
```

##### test_file

```c++
#include<iostream>
#include"ThreadPool.hpp"
#include<stdlib.h>
#include<memory>
#include<functional>
#include"MutexLock.hpp"
using std::function;
using std::bind;
using std::unique_ptr;
using std::cout;
using std::endl;

class MyTask {
public:
    MyTask(MutexLock & mutex) 
    : _mutex(mutex)
    {
        srand(clock());       
    }
    void process() {
        int randNum = rand() % 100;
        _mutex.Lock();
        cout <<"(" << ++_cnt << ")  ";
        cout << "MyTask rand_number = " << randNum << endl;
        _mutex.UnLock();
    }
private:
    MutexLock & _mutex;
    static int _cnt;
};
int MyTask::_cnt = 0;

void test0(){
    MutexLock mutex;
    unique_ptr<MyTask> pTask(new MyTask(mutex));
    ThreadPool threadPool(4,10);
    int cnt = 100;
    threadPool.Start();
    while(cnt--) {
        threadPool.AddTask(bind(&MyTask::process,pTask.get()));
    }
    threadPool.Stop();
}
int main(void) {
    test0();
    return 0;
}
```

---



### Reactor(v1)

```c++
#ifndef __SOCKET_HPP__
#define __SOCKET_HPP__

class Socket {
public:
    Socket();
    ~Socket();
    explicit Socket(const int & fd);
    int fd() const;
private:
    int _listenFd;
};

#endif
```

```c++
#ifndef __INETADDRESS_HPP__
#define __INETADDRESS_HPP__

#include<arpa/inet.h>
#include<string>
using std::string;
class InetAddress {
public:
    InetAddress(const string & ip, const unsigned short & port);
    InetAddress(struct sockaddr_in & addr);
    ~InetAddress();
    string Ip() const;
    unsigned short Port() const;
    struct sockaddr_in* GetAddrPtr();
private:
    struct sockaddr_in _addr;
};

#endif
```

```c++
#ifndef __ACCRPTOR_HPP__
#define __ACCRPTOR_HPP__

#include"InetAddress.hpp"
#include"Socket.hpp"
class Acceptor {
public:
    Acceptor(const string & ip, const unsigned short & port);
    ~Acceptor();
    void Ready();
    int Accept();
private:
    void SetReuseAddr() const;
    void SetReusePort() const;
    void Bind() ;
    void Listen() const;
private:
    Socket _socket;
    InetAddress _inetAddr;
};

#endif
```

```c++
#ifndef __SOCKETIO_HPP__
#define __SOCKETIO_HPP__

class SocketIO {
public:
    SocketIO(int fd);
    ~SocketIO();
    int readn(char * buff, int len);
    int readLine(char * buff, int len);
    int writen(const char * buff, int len);
private:
    int _netFd;
};

#endif
```

```c++
#ifndef __TCPCONNECTION_HPP__
#define __TCPCONNECTION_HPP__

#include<string>
#include"SocketIO.hpp"
using std::string;
class TcpConnection {
public:
    explicit TcpConnection(int fd);
    ~TcpConnection();
    string Recv();
    void Send(const string & msg);
private:
    SocketIO _socketIO;
};

#endif
```

```c++
#include"Socket.hpp"
#include<unistd.h>
#include<sys/types.h>
#include<sys/socket.h>

Socket::Socket()
{
    _listenFd = socket(AF_INET,SOCK_STREAM,0);
    if(-1 == _listenFd) {
        return;
    }
}
Socket::~Socket(){
    close(_listenFd);
}
Socket::Socket(const int & fd)
: _listenFd(fd)
{}
int Socket::fd() const {
    return _listenFd;
}
```

```c++
#include"InetAddress.hpp"

InetAddress::InetAddress(const string & ip, const unsigned short & port) {
    _addr.sin_family = AF_INET;
    _addr.sin_addr.s_addr = inet_addr(ip.c_str());
    _addr.sin_port = htons(port);
}
InetAddress::InetAddress(struct sockaddr_in & addr) 
: _addr(addr)
{}
InetAddress::~InetAddress() {

}
std::string InetAddress::Ip() const {
    return inet_ntoa(_addr.sin_addr);
}
unsigned short InetAddress::Port() const {
    return ntohs(_addr.sin_port);
}
struct sockaddr_in* InetAddress::GetAddrPtr() {
    return &_addr;
}
```

```c++
#include"Acceptor.hpp"
#include<stdio.h>

Acceptor::Acceptor(const string & ip, const unsigned short & port) 
: _inetAddr(ip,port)
{}
Acceptor::~Acceptor() {

}
void Acceptor::Ready() {
    SetReuseAddr();
    SetReusePort();
    Bind();
    Listen();
}
int Acceptor::Accept() {
    int connectFd = accept(_socket.fd(),nullptr,nullptr);
    if(-1 == connectFd) {
        perror("accept");
        return -1;
    }
    return connectFd;
}
void Acceptor::SetReuseAddr() const {
    int val = 1;
    int res = setsockopt(_socket.fd(),SOL_SOCKET,SO_REUSEADDR,&val,sizeof(val));
    if(res < 0) {
        perror("setsockopt");
        return;
    }
} 
void Acceptor::SetReusePort() const {
    int val = 1;
    int res = setsockopt(_socket.fd(),SOL_SOCKET,SO_REUSEPORT,&val,sizeof(val));
    if(res < 0) {
        perror("setsockopt");
        return;
    }
}
void Acceptor::Bind() {
    int res = bind(_socket.fd(),(struct sockaddr*)(_inetAddr.GetAddrPtr()),sizeof(struct sockaddr_in));
    if(res < 0) {
        perror("bind");
        return;
    }
}
void Acceptor::Listen() const {
    int res = listen(_socket.fd(),128);
    if(res < 0) {
        perror("listen");
        return;
    }
}
```

```c++
#include"SocketIO.hpp"
#include<unistd.h>
#include<errno.h>
#include<stdio.h>
#include<sys/types.h>
#include<sys/socket.h>
#include<string.h>

SocketIO::SocketIO(int fd)
: _netFd(fd)
{}
SocketIO::~SocketIO() {
    close(_netFd);
}
int SocketIO::readn(char * buff, int len) {
    int sret = 0;
    int length = len;
    char *p = buff;
    while(length > 0){
        sret = read(_netFd,p,length);
        if(-1 == sret && errno == EINTR) {
            continue;
        } else if (-1 == sret) {
            perror("read error -1");
            return len - sret;
        } else if (0 == sret) {
            break;
        } else {
            p += sret;
            length -= sret;
        }
    }
    return len - length;
}
int SocketIO::readLine(char * buff, int len) {
    int sret = 0;
    int length = len;
    int total = 0;
    char * p = buff;
    
    while(length > 0) {
        sret = recv(_netFd,p,length,MSG_PEEK);

        if(-1 == sret && errno == EINTR) {
            fprintf(stderr,"signal,errno=EINTR\n");
            continue;
        } else if (-1 == sret) {
            perror("recv error -1");
            return len - sret;
        } else if (0 == sret) {
            fprintf(stderr,"sret = 0\n");
            break;
        } else {
            for(int i = 0; i < sret; ++i) {
                if(p[i] == '\n') {
                    int sz = i + 1;
                    readn(p,sz);
                    p += sz;
                    p[0] = '\0';
                    return total + sz;
                }
            }
            readn(p,sret);
            total += sret;
            length -= sret;
            p += sret;
        }
    }
    p[0] = '\0';
    return total;
}
int SocketIO::writen(const char * buff, int len) {
    int sret = 0;
    int length = len;
    const char *p = buff;
    while(length > 0) {
        sret = write(_netFd,p,length);
        if(-1 == sret && errno == EINTR) {
            continue;
        } else if (-1 == sret) {
            perror("write error -1");
            return len - sret;
        } else if (0 == sret) {
            break;
        } else {
            p += sret;
            length -= sret;
        }
    }
    return len - length;
}
```

```c++
#include"TcpConnection.hpp"

TcpConnection::TcpConnection(int fd) 
: _socketIO(fd)
{}
TcpConnection::~TcpConnection() {

}
string TcpConnection::Recv() {
    char buf[65535] = {0};
    _socketIO.readLine(buf,sizeof(buf));
    return buf;
}
void TcpConnection::Send(const string & msg) {
    _socketIO.writen(msg.c_str(),msg.size());
}
```

##### test_file

```c++
#include<iostream>
#include"Acceptor.hpp"
#include"TcpConnection.hpp"
using std::cout;
using std::endl;

void test(){
    Acceptor acceptor("127.0.0.1", 8888);
    acceptor.Ready();
    TcpConnection tcpConnection(acceptor.Accept());
    while(1){
        /* cout << "...while..." << endl; */
        string str = tcpConnection.Recv();
        if(str == "\0") {
            cout << "null string" << endl;
        }
        cout << str;
        tcpConnection.Send(">>(server) nihaoya");
    }
}
int main(void){
    test();
    return 0;
}
```

---





### Reactor(v2/v3)

<img src="C:\Users\Administrator\Desktop\cpp_advance\Reactor_pic\image-20230528210605116.png" alt="image-20230528210605116" style="zoom: 80%;" />

header_file

```c++
#ifndef __SOCKET_HPP__
#define __SOCKET_HPP__ 

//adpat the RAII method
class Socket {
public:
    Socket();       //托管资源
    ~Socket();      //回收资源
    explicit Socket(int fd);
    int Fd() const; //获取socket的对应的文件描述符
private:
    int _listenFd;
};

#endif
```

```c++
#ifndef __INETADDRESS_HPP__
#define __INETADDRESS_HPP__

#include<arpa/inet.h>
#include<string>
using std::string;

class InetAddress {
public:
    InetAddress(const string & ip, const unsigned short & port);
    InetAddress(const struct sockaddr_in & addr);
    ~InetAddress();
    string Ip() const;
    unsigned short Port() const;
private:
    struct sockaddr_in _addr;
};

#endif
```

```c++
#ifndef __ACCEPTOR_HPP__
#define __ACCEPTOR_HPP__

#include"Socket.hpp"
#include"InetAddress.hpp"

class Acceptor {
public:
    Acceptor(const string & ip, const unsigned short & port);
    ~Acceptor();
    void Ready();
    int Accept();
    int Fd() const;
private:
    void Bind();
    void Listen();
    void SetReuseAddr();
    void SetReusePort();
private:
    Socket _socket;
    InetAddress _inetAddr;
};

#endif
```

```c++
#ifndef __SOCKETIO_HPP__
#define __SOCKETIO_HPP__

#include<string>
using std::string;

class SocketIO {
public:
    SocketIO(int fd);
    ~SocketIO();
    int ReadN(char * buff, int len);
    int ReadLine(char * buff, int len);
    int WriteN(const char * buff, int len);
private:
    int _connectFd;
};

#endif 
```

##### TcpConnection.hpp

```c++
#ifndef __TCPCONNECTION_HPP__
#define __TCPCONNECTION_HPP__

#include"SocketIO.hpp"
#include"Socket.hpp"
#include"InetAddress.hpp"
#include<string>
#include<memory>
#include<functional>
using std::function;
using std::string;
using std::shared_ptr;

class TcpConnection 
: public std::enable_shared_from_this<TcpConnection>
{
public:
    using TcpConnectionPtr = shared_ptr<TcpConnection>;
    using CallBackFunc = function<void(TcpConnectionPtr)>;
public:
    explicit TcpConnection(int fd);
    ~TcpConnection();
    string Recieve();
    bool IsConnectionClosed() const;
    void Send(const string & msg);
    void SetOnConnectionCallBack(const CallBackFunc & cb);
    void SetOnMessageCallBack(const CallBackFunc & cb);
    void SetOnCloseCallBack(const CallBackFunc & cb);
    void HandleConnectionCallBack();
    void HandleMessageCallBack();
    void HandleCloseCallBack();

    string InfoStringPrint() const;     // debug:test_function
private:
    InetAddress GetSelfAddr();     // debug:test_function
    InetAddress GetPeerAddr();     // debug:test_function
private:
    SocketIO _sockIO;
    Socket _socket;
    InetAddress _selfAddr;
    InetAddress _peerAddr;
    CallBackFunc _onConnection;
    CallBackFunc _onMessage;
    CallBackFunc _onClose;
};

#endif
```

##### EventEpoll.hpp

```c++
#ifndef __EVENTEPOLL_HPP__
#define __EVENTEPOLL_HPP__

#include"TcpConnection.hpp"
#include"Acceptor.hpp"
#include<vector>
#include<map>
using std::vector;
using std::map;

class EventEpoll {
public:
    using TcpConnectionPtr = typename TcpConnection::TcpConnectionPtr;
    using CallBackFunc = typename TcpConnection::CallBackFunc;
public:
    EventEpoll(Acceptor & acceptor);
    ~EventEpoll();
    int CreateEpoll();
    void AddFdToEpollIn(int fd);
    void DelFdFromEpollIn(int fd);
    void Loop();
    void UnLoop();
    void EpollWait();   
    void SetOnConnectionCallBack(CallBackFunc && cb);
    void SetOnMessageCallBack(CallBackFunc && cb);
    void SetOnCloseCallBack(CallBackFunc && cb);

private:
    int _epfd;
    bool _isLooping;
    Acceptor & _acceptor;
    vector<struct epoll_event> _readyEvents;
    map<int,TcpConnectionPtr> _connectLists;
    CallBackFunc _onConnection;
    CallBackFunc _onMessage;
    CallBackFunc _onClose;
};

#endif
```

##### TcpServer.hpp

```c++
#ifndef __TCPSERVER_HPP__
#define __TCPSERVER_HPP__

#include"EventEpoll.hpp"
#include"Acceptor.hpp"
#include<utility>
using TcpConnectionPtr = typename EventEpoll::TcpConnectionPtr;
using CallBackFunc = typename EventEpoll::CallBackFunc;

class TcpServer {
public:
    TcpServer(const string & ip, const unsigned short & port);
    ~TcpServer();
    void Start();
    void Stop();
    void SetAllCallBack(CallBackFunc && onConnection, CallBackFunc && onMessageFunc, CallBackFunc && onCloseFunc);
private:
    Acceptor _acceptor;
    EventEpoll _eventEpool;
};

#endif
```

---

##### ------

implement_file

```c++
#include"Socket.hpp"
#include<sys/types.h>
#include<sys/socket.h>
#include<unistd.h>
#include<stdio.h>

Socket::Socket() 
{
    _listenFd = socket(AF_INET,SOCK_STREAM,0);
    if(-1 == _listenFd) {
        perror("socket");
    }
}      
Socket::~Socket() {
    if(_listenFd >= 0) {
        close(_listenFd);
    }
}
Socket::Socket(int fd) 
: _listenFd(fd)
{}
int Socket::Fd() const {
    return _listenFd;
}
```

```c++
#include"InetAddress.hpp"

InetAddress::InetAddress(const string & ip, const unsigned short & port) {
    _addr.sin_family = AF_INET;
    _addr.sin_addr.s_addr = inet_addr(ip.c_str());
    _addr.sin_port = htons(port);
}
InetAddress::~InetAddress() {

}
InetAddress::InetAddress(const struct sockaddr_in & addr) 
: _addr(addr)
{
}
string InetAddress::Ip() const {
    return inet_ntoa(_addr.sin_addr);
}
unsigned short InetAddress::Port() const {
    return ntohs(_addr.sin_port);
}
```

```c++
#include"Acceptor.hpp"
#include<stdio.h>
Acceptor::Acceptor(const string & ip, const unsigned short & port) 
: _inetAddr(ip,port)
{}

Acceptor::~Acceptor() {

}

void Acceptor::Ready() {
    SetReuseAddr();
    SetReusePort();
    Bind();
    Listen();
}
int Acceptor::Accept() {
    int connectFd = accept(_socket.Fd(),nullptr,nullptr);
    if(-1 == connectFd) {
        perror("accept in Accept");
    }
    return connectFd;
}
int Acceptor::Fd() const {
    return _socket.Fd();
}
void Acceptor::Bind() {
    int res = bind(_socket.Fd(),(struct sockaddr*)&_inetAddr, sizeof(struct sockaddr_in));
    if(res < 0) {
        perror("bind in Bind");
    }
}
void Acceptor::Listen() {
    int res = listen(_socket.Fd(),128);
    if(res < 0) {
        perror("listen in Listen");
    }
}
void Acceptor::SetReuseAddr() {
    int opt = 1;
    int res = setsockopt(_socket.Fd(),SOL_SOCKET,SO_REUSEADDR,&opt,sizeof(opt));
    if(res < 0) {
        perror("setsockopt in SetReuseAddr");
    }
}
void Acceptor::SetReusePort() {
    int opt = 1;
    int res = setsockopt(_socket.Fd(),SOL_SOCKET,SO_REUSEPORT,&opt,sizeof(opt));
    if(res < 0) {
        perror("setsockopt in SetReuseAddr");
    }
}
```

```c++
#include"SocketIO.hpp"
#include<unistd.h>
#include<errno.h>
#include<stdio.h>
#include<sys/types.h>
#include<sys/socket.h>

SocketIO::SocketIO(int fd) 
: _connectFd(fd)
{}
SocketIO::~SocketIO() {
    if(_connectFd >= 0) {
        close(_connectFd);
    }
}
int SocketIO::ReadN(char * buff, int len) {
    int sret = 0;
    int length = len;
    char *p = buff;
    while(length > 0){
        sret = read(_connectFd,p,length);
        if(-1 == sret && errno == EINTR) {
            continue;
        } else if (-1 == sret) {
            perror("read error -1");
            return len - sret;
        } else if (0 == sret) {
            break;
        } else {
            p += sret;
            length -= sret;
        }
    }
    return len - length;
}
int SocketIO::ReadLine(char * buff, int len) {
    int sret = 0;
    int length = len;
    int total = 0;
    char * p = buff;
    
    while(length > 0) {
        sret = recv(_connectFd,p,length,MSG_PEEK);

        if(-1 == sret && errno == EINTR) {
            fprintf(stderr,"signal,errno=EINTR\n");
            continue;
        } else if (-1 == sret) {
            perror("recv error -1");
            return len - sret;
        } else if (0 == sret) {
            fprintf(stderr,"sret = 0\n");
            break;
        } else {
            for(int i = 0; i < sret; ++i) {
                if(p[i] == '\n') {
                    int sz = i + 1;
                    ReadN(p,sz);
                    p += sz;
                    p[0] = '\0';
                    return total + sz;
                }
            }
            ReadN(p,sret);
            total += sret;
            length -= sret;
            p += sret;
        }
    }
    p[0] = '\0';
    return total;
}
int SocketIO::WriteN(const char * buff, int len) {
    int sret = 0;
    int length = len;
    const char *p = buff;
    while(length > 0) {
        sret = write(_connectFd,p,length);
        if(-1 == sret && errno == EINTR) {
            continue;
        } else if (-1 == sret) {
            perror("write error -1");
            return len - sret;
        } else if (0 == sret) {
            break;
        } else {
            p += sret;
            length -= sret;
        }
    }
    return len - length;
}
```

##### TcpConnection.cc

```c++
#include "TcpConnection.hpp"
#include<string.h>

TcpConnection::TcpConnection(int fd) 
: _sockIO(fd)
, _socket(fd)
, _selfAddr(GetSelfAddr())
, _peerAddr(GetPeerAddr())
{}

TcpConnection::~TcpConnection() {

}

string TcpConnection::Recieve() {
    char buf[65535] = {0};
    _sockIO.ReadLine(buf,sizeof(buf));
    return string(buf);
}

bool TcpConnection::IsConnectionClosed() const {
    char buf[10] = {0};
    int res = recv(_socket.Fd(),buf,sizeof(buf),MSG_PEEK);
    if(res) {
        return false;
    } else {
        return true;
    }
}
void TcpConnection::Send(const string & msg) {
    _sockIO.WriteN(msg.c_str(), msg.size());
}
void TcpConnection::SetOnConnectionCallBack(const CallBackFunc & cb) {
    _onConnection = cb;   
}
void TcpConnection::SetOnMessageCallBack(const CallBackFunc & cb) {
    _onMessage = cb;
}
void TcpConnection::SetOnCloseCallBack(const CallBackFunc & cb) {
    _onClose = cb;
}
void TcpConnection::HandleConnectionCallBack() {
    _onConnection(shared_from_this());
}
void TcpConnection::HandleMessageCallBack() {
    _onMessage(shared_from_this());
}
void TcpConnection::HandleCloseCallBack() {
    _onClose(shared_from_this());
}
string TcpConnection::InfoStringPrint() const {
    string str;
    str += _selfAddr.Ip();
    str += ":";
    str += std::to_string(_selfAddr.Port());
    str += "<------>";
    str += _peerAddr.Ip();
    str += ":";
    str += std::to_string(_peerAddr.Port());
    return str;
}
InetAddress TcpConnection::GetSelfAddr() {
    struct sockaddr_in addr;
    socklen_t len;
    memset(&addr,0,sizeof(addr));
    getsockname(_socket.Fd(),(struct sockaddr*)&addr,&len);
    return addr;
}
InetAddress TcpConnection::GetPeerAddr() {
    struct sockaddr_in addr;
    socklen_t len;
    memset(&addr,0,sizeof(addr));
    getpeername(_socket.Fd(),(struct sockaddr*)&addr,&len);
    return addr;

}
```

##### EventEpoll.cc

```c++
#include"EventEpoll.hpp"
#include<iostream>
#include<sys/epoll.h>
#include<string.h>
using std::cerr;
using std::cout;
using std::endl;

EventEpoll::EventEpoll(Acceptor & acceptor) 
: _epfd(CreateEpoll())
, _isLooping(false)
, _acceptor(acceptor)
, _readyEvents(1024)
{
    AddFdToEpollIn(_acceptor.Fd());
}

EventEpoll::~EventEpoll() {

}

int EventEpoll::CreateEpoll() {
    int epfd = epoll_create1(0);
    if(epfd < 0) {
        perror("epoll_create1 in CreateEpoll");
    }
    return epfd;
}

void EventEpoll::AddFdToEpollIn(int fd) {
    struct epoll_event evt;
    memset(&evt,0,sizeof(evt));
    evt.events = EPOLLIN;
    evt.data.fd = fd;
    int res = epoll_ctl(_epfd,EPOLL_CTL_ADD,fd,&evt);
    if(res < 0) {
        perror("epoll_ctl in AddFdToEpollIn");
    }
}
void EventEpoll::DelFdFromEpollIn(int fd) {
    struct epoll_event evt;
    memset(&evt,0,sizeof(evt));
    evt.events = EPOLLIN;
    evt.data.fd = fd;
    int res = epoll_ctl(_epfd,EPOLL_CTL_DEL,fd,&evt);
    if(res < 0) {
        perror("epoll_ctl in AddFdToEpollIn");
    }
}
void EventEpoll::Loop() {
    _isLooping = true;
    while(_isLooping) {
        EpollWait();
    }
}
void EventEpoll::UnLoop() {
    _isLooping = false;
}
void EventEpoll::EpollWait() {
    int readyNum;
    
    do {
        readyNum = epoll_wait(_epfd,&*_readyEvents.begin(),_readyEvents.size(),3000);
    } while(-1 == readyNum && errno == EINTR);
    
    if(-1 == readyNum) {
        perror("epoll_wait in EpollWait");
        UnLoop();
        return;
    } else if (0 == readyNum) {
        cerr << "epoll_wait over time" << endl;
    } else {
        for(int i = 0; i < readyNum; ++i) {
            if(_readyEvents[i].data.fd == _acceptor.Fd()
               && _readyEvents[i].events & EPOLLIN) {
                
               //初始vector只有1024大小，当vector可能不够存放新的连接时，扩充vector大小
                if(readyNum ==static_cast<int>(_readyEvents.size())) {
                    _readyEvents.resize(2 * readyNum);
                }

                int connectFd = _acceptor.Accept();
                if(connectFd < 0) {
                    cerr << "Accept failed" << endl;
                    continue;
                }
                
                //加入红黑树监听集合
                AddFdToEpollIn(connectFd);
            
                TcpConnectionPtr ptcpConnectionPtr(new TcpConnection(connectFd));
               
                //注册三个半之三
                ptcpConnectionPtr->SetOnConnectionCallBack(_onConnection);
                ptcpConnectionPtr->SetOnMessageCallBack(_onMessage);
                ptcpConnectionPtr->SetOnCloseCallBack(_onClose);

                _connectLists.insert(make_pair(connectFd,ptcpConnectionPtr));

                //三个半 之 连接建立
                ptcpConnectionPtr->HandleConnectionCallBack();
            } else {
                auto it = _connectLists.find(_readyEvents[i].data.fd);
                if (it == _connectLists.end()) {
                    cerr << "tcp connection not exit" << endl;
                } else {
                    if(it->second->IsConnectionClosed()) {
                        cerr << "tcp connection have closed" << endl;

                         //三个半 之 连接关闭
                        it->second->HandleCloseCallBack();

                        DelFdFromEpollIn(it->first);
                        _connectLists.erase(it);
                    } else {
                        //三个半 之 连接正常
                        it->second->HandleMessageCallBack();
                    }
                }
            }
        }
    }
}   
void EventEpoll::SetOnConnectionCallBack(CallBackFunc && cb) {
    _onConnection = std::move(cb);
}
void EventEpoll::SetOnMessageCallBack(CallBackFunc && cb) {
    _onMessage = std::move(cb);
}
void EventEpoll::SetOnCloseCallBack(CallBackFunc && cb) {
    _onClose = std::move(cb);
}
```

##### TcpServer.cc

```c++
#include"TcpServer.hpp"

TcpServer::TcpServer(const string & ip, const unsigned short & port)
: _acceptor(ip,port)
, _eventEpool(_acceptor)
{
}

TcpServer::~TcpServer() {

}

void TcpServer::Start() {
    _acceptor.Ready();
    _eventEpool.Loop();
}

void TcpServer::Stop() {
    _eventEpool.UnLoop();
}

void TcpServer::SetAllCallBack(CallBackFunc && onConnection
                               , CallBackFunc && onMessageFunc
                               , CallBackFunc && onCloseFunc) 
{
    _eventEpool.SetOnConnectionCallBack(std::move(onConnection));
    _eventEpool.SetOnMessageCallBack(std::move(onMessageFunc));
    _eventEpool.SetOnCloseCallBack(std::move(onCloseFunc));
}
```

##### test_file

```c++
#include<iostream>
#include"TcpServer.hpp"
using std::cout;
using std::endl;
using TcpConnectionPtr = typename EventEpoll::TcpConnectionPtr;
using CallBackFunc = typename EventEpoll::CallBackFunc;

void onConnectionFunc(const TcpConnectionPtr &con) {
    cout << con->InfoStringPrint() << " have connected" << endl;
}
void onMessageFunc(const TcpConnectionPtr &con) {
    string msg = con->Recieve();
    //处理业务 
    con->Send(msg);
}
void onCloseFunc(const TcpConnectionPtr &con) {
    cout << con->InfoStringPrint() << " have closed" << endl;
}

void test() {
    TcpServer tcpServer("127.0.0.1", 8888);
    tcpServer.SetAllCallBack(std::move(onConnectionFunc),std::move(onMessageFunc),std::move(onCloseFunc));
    tcpServer.Start();
    
    tcpServer.Stop();
}
int main(void) {
    test();
    return 0;
}
```

---



### Reactor(v4/v5)

![image-20230528223239476](Reactor_pic\image-20230528223239476.png)

<img src="C:\Users\Administrator\Desktop\cpp_advance\Reactor_pic\image-20230528211554891.png" alt="image-20230528211554891" style="zoom:80%;" />





##### Socket.hpp

```c++
#ifndef __SOCKET_HPP__
#define __SOCKET_HPP__ 

//adpat the RAII method
class Socket {
public:
    Socket();       //托管资源
    ~Socket();      //回收资源
    explicit Socket(int fd);
    int Fd() const; //获取socket的对应的文件描述符
private:
    int _listenFd;
};

#endif
```

##### InetAddress.hpp

```c++
#ifndef __INETADDRESS_HPP__
#define __INETADDRESS_HPP__

#include<arpa/inet.h>
#include<string>
using std::string;

class InetAddress {
public:
    InetAddress(const string & ip, const unsigned short & port);
    InetAddress(const struct sockaddr_in & addr);
    ~InetAddress();
    string Ip() const;
    unsigned short Port() const;
private:
    struct sockaddr_in _addr;
};

#endif
```

##### Acceptor.hpp

```c++
#ifndef __ACCEPTOR_HPP__
#define __ACCEPTOR_HPP__

#include"Socket.hpp"
#include"InetAddress.hpp"

class Acceptor {
public:
    Acceptor(const string & ip, const unsigned short & port);
    ~Acceptor();
    void Ready();
    int Accept();
    int Fd() const;
private:
    void Bind();
    void Listen();
    void SetReuseAddr();
    void SetReusePort();
private:
    Socket _socket;
    InetAddress _inetAddr;
};

#endif
```

##### SockIO.hpp

```c++
#ifndef __SOCKETIO_HPP__
#define __SOCKETIO_HPP__

#include<string>
using std::string;

class SocketIO {
public:
    SocketIO(int fd);
    ~SocketIO();
    int ReadN(char * buff, int len);
    int ReadLine(char * buff, int len);
    int WriteN(const char * buff, int len);
private:
    int _connectFd;
};
```

##### TcpConnection.hpp

```c++
#ifndef __TCPCONNECTION_HPP__
#define __TCPCONNECTION_HPP__

#include"SocketIO.hpp"
#include"Socket.hpp"
#include"InetAddress.hpp"
#include<string>
#include<memory>
#include<functional>
using std::function;
using std::string;
using std::shared_ptr;

class EventEpoll;
class TcpConnection 
: public std::enable_shared_from_this<TcpConnection>
{
public:
    using TcpConnectionPtr = shared_ptr<TcpConnection>;
    using CallBackFunc = function<void(TcpConnectionPtr)>;
public:
    explicit TcpConnection(int fd, EventEpoll *pEventEpoll);
    ~TcpConnection();

    string Recieve();
    //判断当前连接（对方）是否断开
    bool IsConnectionClosed() const;
    void Send(const string & msg);
    
    //线程池将业务处理后的结果发送给EventEpoll/Reactor
    void SendInLoop(const string & msg);

    //三个事件的注册
    void SetOnConnectionCallBack(const CallBackFunc & cb);
    void SetOnMessageCallBack(const CallBackFunc & cb);
    void SetOnCloseCallBack(const CallBackFunc & cb);
    //三个事件的执行
    void HandleConnectionCallBack();
    void HandleMessageCallBack();
    void HandleCloseCallBack();

    string InfoStringPrint() const;     // debug:test_function
private:
    InetAddress GetSelfAddr();     // debug:test_function
    InetAddress GetPeerAddr();     // debug:test_function
private:
    SocketIO _sockIO;
    Socket _socket;
    InetAddress _selfAddr;
    InetAddress _peerAddr;
    EventEpoll *_pEventEpoll;       //线程池将业务处理的结果发送给Reactor/EventEpoll,需要知道EventEpoll的存在(关联)
    CallBackFunc _onConnection;
    CallBackFunc _onMessage;
    CallBackFunc _onClose;
};

#endif
```

##### EventEpoll.hpp

```c++
#ifndef __EVENTEPOLL_HPP__
#define __EVENTEPOLL_HPP__

#include"TcpConnection.hpp"
#include"Acceptor.hpp"
#include"MutexLock.hpp"
#include<vector>
#include<map>
#include<memory>
#include<sys/eventfd.h>
using std::vector;
using std::map;
using std::shared_ptr;
using std::function;

class EventEpoll {
public:
    using TcpConnectionPtr = typename TcpConnection::TcpConnectionPtr;
    using CallBackFunc = typename TcpConnection::CallBackFunc;
    using EventCallBackFunc = std::function<void()>;
public:
    EventEpoll(Acceptor & acceptor);
    ~EventEpoll();
    
    //创建一个epoll文件描述符
    int CreateEpoll();
    //将要监听的读阻塞行为的文件描述符加入到监听集合中(红黑树)
    void AddFdToEpollIn(int fd);
    void DelFdFromEpollIn(int fd);

    void Loop();
    void UnLoop();
    void EpollWait();

    //用于处理新的连接
    void HandleNewConnection();
    //用于处理老的连接
    void HandleMessage(int fd);
    
    //三个事件的注册
    void SetOnConnectionCallBack(CallBackFunc && cb);
    void SetOnMessageCallBack(CallBackFunc && cb);
    void SetOnCloseCallBack(CallBackFunc && cb);
    
    //创建一个eventfd，用于线程间通信（线程池和Reactor/EventEpoll）
    int CreateEventFd();
    //对eventfd进行读操作
    void HandleRead();
    //对eventfd进行写操作
    void WakeUp();
    //Reactor/EventEpoll执行待执行函数
    void DoPendingFunc();
    
    //线程池将待执行事件发送给EventEpoll/Reactor
    void RunInLoop(EventCallBackFunc && ecb);

private:
    int _epfd;              //epoll文件描述符
    int _eventFd;           //eventfd文件描述符
    bool _isLooping;        //epoll事件循环是否继续进行的标志
    Acceptor & _acceptor;   //需要通过Accpet函数获取新的连接
    vector<struct epoll_event> _readyEvents;    //就绪队列
    map<int,TcpConnectionPtr> _connectLists;    //存储文件描述符和对应的连接
    CallBackFunc _onConnection;     //连接建立事件的函数对象
    CallBackFunc _onMessage;        //消息完成事件的函数对象
    CallBackFunc _onClose;          //连接断开事件的函数对象
    vector<EventCallBackFunc> _pending;         //Reactor用于存储待执行函数的集合
    MutexLock _mutex;       //由于对于_pending是多线程（线程池和Reactor）访问的，因此必须要加锁
};

#endif
```

##### TcpServer.hpp

```c++
#ifndef __TCPSERVER_HPP__
#define __TCPSERVER_HPP__

#include"EventEpoll.hpp"
#include"Acceptor.hpp"
#include<utility>
using TcpConnectionPtr = typename EventEpoll::TcpConnectionPtr;
using CallBackFunc = typename EventEpoll::CallBackFunc;

class TcpServer {
public:
    TcpServer(const string & ip, const unsigned short & port);
    ~TcpServer();
    void Start();
    void Stop();
    void SetAllCallBack(CallBackFunc && onConnection, CallBackFunc && onMessageFunc, CallBackFunc && onCloseFunc);
private:
    Acceptor _acceptor;
    EventEpoll _eventEpool;
};

#endif
```

##### ToUpperServer.hpp

```c++
#ifndef __TOUPPERSERVER_Hpp__
#define __TOUPPERSERVER_Hpp__

#include"TcpServer.hpp"
#include"ThreadPool.hpp"
#include<iostream>
#include<string.h>
using std::cout;
using std::endl;
using namespace std::placeholders;
using TcpConnectionPtr = typename EventEpoll::TcpConnectionPtr;
using CallBackFunc = typename EventEpoll::CallBackFunc;

class MyTask {
public:
    MyTask(const string &str, const TcpConnectionPtr & tcpConnection)
    : _str(str)
    , _tcpConnection(tcpConnection)
    {}

    ~MyTask(){
    }

    void process() {
        //处理业务逻辑
        for(auto & c : _str) {
            c = ::toupper(c);
        }
        //线程池将数据结果发送给Reactor
        _tcpConnection->SendInLoop(_str);
    }
private:
    string _str;
    const TcpConnectionPtr & _tcpConnection;
};


class ToUpperServer {
public:
    ToUpperServer(size_t threadNum, size_t queueSize,const string & ip, const unsigned short & port) 
    : _threadPool(threadNum,queueSize)
    , _tcpServer(ip,port)
    {
    }
    
    ~ToUpperServer() {

    }
    
    void Start() {
        _threadPool.Start();
        _tcpServer.SetAllCallBack(std::bind(&ToUpperServer::onConnectionFunc,this,_1)
                                  , std::bind(&ToUpperServer::onMessageFunc,this,_1)
                                  , std::bind(&ToUpperServer::onCloseFunc,this,_1));
        _tcpServer.Start();
    }

    void Stop() {
        _tcpServer.Stop();
        _threadPool.Stop();
    }


    void onConnectionFunc(const TcpConnectionPtr &con) {
        cout << con->InfoStringPrint() << " have connected" << endl;
    }

    void onMessageFunc(const TcpConnectionPtr &con) {
        string msg = con->Recieve();
        //...
        //业务处理逻辑
        //...

        //将业务处理后的结果发送给EventLoop/Reactor,然后由EventLoop/Reactor将其发送给客户端
        //业务逻辑交给线程池去做

        MyTask task(msg,con);
        _threadPool.AddTask(std::bind(&MyTask::process, task));
    }

    void onCloseFunc(const TcpConnectionPtr &con) {
        cout << con->InfoStringPrint() << " have closed" << endl;
    }
private:
    ThreadPool _threadPool;
    TcpServer _tcpServer;
};
#endif
```

---

##### ------

##### Socket.cc

```c++
#include"Socket.hpp"
#include<sys/types.h>
#include<sys/socket.h>
#include<unistd.h>
#include<stdio.h>

Socket::Socket() 
{
    _listenFd = socket(AF_INET,SOCK_STREAM,0);
    if(-1 == _listenFd) {
        perror("socket");
    }
}      
Socket::~Socket() {
    if(_listenFd >= 0) {
        close(_listenFd);
    }
}
Socket::Socket(int fd) 
: _listenFd(fd)
{}
int Socket::Fd() const {
    return _listenFd;
}
```

##### InetAddress.cc

```c++
#include"InetAddress.hpp"

InetAddress::InetAddress(const string & ip, const unsigned short & port) {
    _addr.sin_family = AF_INET;
    _addr.sin_addr.s_addr = inet_addr(ip.c_str());
    _addr.sin_port = htons(port);
}
InetAddress::~InetAddress() {

}
InetAddress::InetAddress(const struct sockaddr_in & addr) 
: _addr(addr)
{
}
string InetAddress::Ip() const {
    return inet_ntoa(_addr.sin_addr);
}
unsigned short InetAddress::Port() const {
    return ntohs(_addr.sin_port);
}
```

##### Acceptor.cc

```c++
#include"Acceptor.hpp"
#include<stdio.h>
Acceptor::Acceptor(const string & ip, const unsigned short & port) 
: _inetAddr(ip,port)
{}

Acceptor::~Acceptor() {

}

void Acceptor::Ready() {
    SetReuseAddr();
    SetReusePort();
    Bind();
    Listen();
}
int Acceptor::Accept() {
    int connectFd = accept(_socket.Fd(),nullptr,nullptr);
    if(-1 == connectFd) {
        perror("accept in Accept");
    }
    return connectFd;
}
int Acceptor::Fd() const {
    return _socket.Fd();
}
void Acceptor::Bind() {
    int res = bind(_socket.Fd(),(struct sockaddr*)&_inetAddr, sizeof(struct sockaddr_in));
    if(res < 0) {
        perror("bind in Bind");
    }
}
void Acceptor::Listen() {
    int res = listen(_socket.Fd(),128);
    if(res < 0) {
        perror("listen in Listen");
    }
}
void Acceptor::SetReuseAddr() {
    int opt = 1;
    int res = setsockopt(_socket.Fd(),SOL_SOCKET,SO_REUSEADDR,&opt,sizeof(opt));
    if(res < 0) {
        perror("setsockopt in SetReuseAddr");
    }
}
void Acceptor::SetReusePort() {
    int opt = 1;
    int res = setsockopt(_socket.Fd(),SOL_SOCKET,SO_REUSEPORT,&opt,sizeof(opt));
    if(res < 0) {
        perror("setsockopt in SetReuseAddr");
    }
}
```

##### SocketIO.cc

```c++
#include"SocketIO.hpp"
#include<unistd.h>
#include<errno.h>
#include<stdio.h>
#include<sys/types.h>
#include<sys/socket.h>

SocketIO::SocketIO(int fd) 
: _connectFd(fd)
{}
SocketIO::~SocketIO() {
    if(_connectFd >= 0) {
        close(_connectFd);
    }
}
int SocketIO::ReadN(char * buff, int len) {
    int sret = 0;
    int length = len;
    char *p = buff;
    while(length > 0){
        sret = read(_connectFd,p,length);
        if(-1 == sret && errno == EINTR) {
            continue;
        } else if (-1 == sret) {
            perror("read error -1");
            return len - sret;
        } else if (0 == sret) {
            break;
        } else {
            p += sret;
            length -= sret;
        }
    }
    return len - length;
}
int SocketIO::ReadLine(char * buff, int len) {
    int sret = 0;
    int length = len;
    int total = 0;
    char * p = buff;
    
    while(length > 0) {
        sret = recv(_connectFd,p,length,MSG_PEEK);

        if(-1 == sret && errno == EINTR) {
            fprintf(stderr,"signal,errno=EINTR\n");
            continue;
        } else if (-1 == sret) {
            perror("recv error -1");
            return len - sret;
        } else if (0 == sret) {
            fprintf(stderr,"sret = 0\n");
            break;
        } else {
            for(int i = 0; i < sret; ++i) {
                if(p[i] == '\n') {
                    int sz = i + 1;
                    ReadN(p,sz);
                    p += sz;
                    p[0] = '\0';
                    return total + sz;
                }
            }
            ReadN(p,sret);
            total += sret;
            length -= sret;
            p += sret;
        }
    }
    p[0] = '\0';
    return total;
}
int SocketIO::WriteN(const char * buff, int len) {
    int sret = 0;
    int length = len;
    const char *p = buff;
    while(length > 0) {
        sret = write(_connectFd,p,length);
        if(-1 == sret && errno == EINTR) {
            continue;
        } else if (-1 == sret) {
            perror("write error -1");
            return len - sret;
        } else if (0 == sret) {
            break;
        } else {
            p += sret;
            length -= sret;
        }
    }
    return len - length;
}
```

##### TcpConnection.cc

```c++
#include "TcpConnection.hpp"
#include "EventEpoll.hpp"
#include <string.h>

TcpConnection::TcpConnection(int fd, EventEpoll *pEventEpoll) 
: _sockIO(fd)
, _socket(fd)
, _selfAddr(GetSelfAddr())
, _peerAddr(GetPeerAddr())
, _pEventEpoll(pEventEpoll)
{}

TcpConnection::~TcpConnection() {

}

string TcpConnection::Recieve() {
    char buf[65535] = {0};
    _sockIO.ReadLine(buf,sizeof(buf));
    return string(buf);
}

bool TcpConnection::IsConnectionClosed() const {
    char buf[10] = {0};
    //MSG_PEEK从sock内核文件缓冲区读数据不取走
    int res = recv(_socket.Fd(),buf,sizeof(buf),MSG_PEEK);
    if(res) {
        return false;
    } else {
        return true;
    }
}
void TcpConnection::Send(const string & msg) {
    _sockIO.WriteN(msg.c_str(), msg.size());
}

void TcpConnection::SendInLoop(const string & msg) {
    //将数据和发送数据的能力即TCP连接的Send方法发送给EventLoop
    _pEventEpoll->RunInLoop(std::bind(&TcpConnection::Send,this,msg));
}

void TcpConnection::SetOnConnectionCallBack(const CallBackFunc & cb) {
    _onConnection = cb;   
}
void TcpConnection::SetOnMessageCallBack(const CallBackFunc & cb) {
    _onMessage = cb;
}
void TcpConnection::SetOnCloseCallBack(const CallBackFunc & cb) {
    _onClose = cb;
}

void TcpConnection::HandleConnectionCallBack() {
    //由于this已经被某个智能指针shared_ptr托管了，
    //因此需要通过shared_from_this()来获取其对应的智能指针
    _onConnection(shared_from_this());
}
void TcpConnection::HandleMessageCallBack() {
    _onMessage(shared_from_this());
}
void TcpConnection::HandleCloseCallBack() {
    _onClose(shared_from_this());
}

string TcpConnection::InfoStringPrint() const {
    string str;
    str += _selfAddr.Ip();
    str += ":";
    str += std::to_string(_selfAddr.Port());
    str += "<------>";
    str += _peerAddr.Ip();
    str += ":";
    str += std::to_string(_peerAddr.Port());
    return str;
}

InetAddress TcpConnection::GetSelfAddr() {
    struct sockaddr_in addr;
    socklen_t len;
    memset(&addr,0,sizeof(addr));
    getsockname(_socket.Fd(),(struct sockaddr*)&addr,&len);
    return addr;
}

InetAddress TcpConnection::GetPeerAddr() {
    struct sockaddr_in addr;
    socklen_t len;
    memset(&addr,0,sizeof(addr));
    getpeername(_socket.Fd(),(struct sockaddr*)&addr,&len);
    return addr;

}
```

##### EventEpoll.cc

```c++
#include"EventEpoll.hpp"
#include<iostream>
#include<sys/epoll.h>
#include<string.h>
#include<unistd.h>
using std::cerr;
using std::cout;
using std::endl;

EventEpoll::EventEpoll(Acceptor & acceptor) 
: _epfd(CreateEpoll())
, _eventFd(CreateEventFd())
, _isLooping(false)
, _acceptor(acceptor)
, _readyEvents(1024)
, _mutex()
{
    //监听_listenFd
    AddFdToEpollIn(_acceptor.Fd());
    //监听_eventFd
    AddFdToEpollIn(_eventFd);
}

EventEpoll::~EventEpoll() {
    if(-1 != _epfd) {
        close(_epfd);
    }
    if(-1 != _eventFd) {
        close(_eventFd);
    }
}

int EventEpoll::CreateEpoll() {
    int epfd = epoll_create1(0);
    if(epfd < 0) {
        perror("epoll_create1 in CreateEpoll");
    }
    return epfd;
}

void EventEpoll::AddFdToEpollIn(int fd) {
    struct epoll_event evt;
    memset(&evt,0,sizeof(evt));
    evt.events = EPOLLIN;
    evt.data.fd = fd;
    int res = epoll_ctl(_epfd,EPOLL_CTL_ADD,fd,&evt);
    if(res < 0) {
        perror("epoll_ctl in AddFdToEpollIn");
    }
}

void EventEpoll::DelFdFromEpollIn(int fd) {
    struct epoll_event evt;
    memset(&evt,0,sizeof(evt));
    evt.events = EPOLLIN;
    evt.data.fd = fd;
    int res = epoll_ctl(_epfd,EPOLL_CTL_DEL,fd,&evt);
    if(res < 0) {
        perror("epoll_ctl in AddFdToEpollIn");
    }
}

void EventEpoll::Loop() {
    _isLooping = true;
    while(_isLooping) {
        EpollWait();
    }
}

void EventEpoll::UnLoop() {
    _isLooping = false;
}

void EventEpoll::EpollWait() {
    int readyNum;
    
    do {
        readyNum = epoll_wait(_epfd,&*_readyEvents.begin(),_readyEvents.size(),3000);
    } while(-1 == readyNum && errno == EINTR);
    
    if(-1 == readyNum) {
        perror("epoll_wait in EpollWait");
        UnLoop();
        return;
    } else if (0 == readyNum) {
        cerr << "epoll_wait over time" << endl;
    } else {
        for(int i = 0; i < readyNum; ++i) {
            if(_readyEvents[i].data.fd == _acceptor.Fd()
               && _readyEvents[i].events & EPOLLIN) {
                
                //初始vector只有1024大小，当vector可能不够存放新的连接时，扩充vector大小
                if(readyNum ==static_cast<int>(_readyEvents.size())) {
                    _readyEvents.resize(2 * readyNum);
                }

                HandleNewConnection();

            } else if (_readyEvents[i].data.fd == _eventFd 
                       && _readyEvents[i].events & EPOLLIN) {

                //此处对于_eventfd的read操作不会阻塞，只是起了将与之相关的内核计数器值清零
                HandleRead();
                
                DoPendingFunc();

            } else {
                HandleMessage(_readyEvents[i].data.fd);
            }
        }
    }
}

void EventEpoll::HandleNewConnection() {
    int connectFd = _acceptor.Accept();
    if(connectFd < 0) {
        perror("Accept in HandleNewConnection");
        return;
    }

    //加入红黑树监听集合
    AddFdToEpollIn(connectFd);

    TcpConnectionPtr ptcpConnectionPtr(new TcpConnection(connectFd,this));

    //注册三个半事件之三 注册
    ptcpConnectionPtr->SetOnConnectionCallBack(_onConnection);
    ptcpConnectionPtr->SetOnMessageCallBack(_onMessage);
    ptcpConnectionPtr->SetOnCloseCallBack(_onClose);

    _connectLists.insert(make_pair(connectFd,ptcpConnectionPtr));

    //三个半 之 连接建立 执行
    ptcpConnectionPtr->HandleConnectionCallBack();

}

void EventEpoll::HandleMessage(int fd) {
    auto it = _connectLists.find(fd);
    if (it == _connectLists.end()) {
        cerr << "tcp connection not exit" << endl;
    } else {

        if(it->second->IsConnectionClosed()) {
            cerr << "tcp connection have closed" << endl;

            //三个半 之 连接关闭 执行
            it->second->HandleCloseCallBack();
            //从监听集合以及_connectLists中删除
            DelFdFromEpollIn(it->first);
            _connectLists.erase(it);

        } else {
            //三个半 之 消息完成 执行
            it->second->HandleMessageCallBack();
        }
    }

}

void EventEpoll::SetOnConnectionCallBack(CallBackFunc && cb) {
    _onConnection = std::move(cb);
}
void EventEpoll::SetOnMessageCallBack(CallBackFunc && cb) {
    _onMessage = std::move(cb);
}
void EventEpoll::SetOnCloseCallBack(CallBackFunc && cb) {
    _onClose = std::move(cb);
}

void EventEpoll::RunInLoop(EventCallBackFunc && ecb) {
    //加入EventEpoll的等待处理事件集合中
    //由于该函数是在TcpConnection::SendInLoop中调用，从线程池调用关系来看，该函数是属于线程池的范围
    {
        MutexLockGuard autoLock(_mutex);
        _pending.push_back(std::move(ecb));
    }

    //此时EventEpoll的等待处理事件集合不为空，线程池将进行通知EventEpoll进行相应处理
    WakeUp();
}

int EventEpoll::CreateEventFd() {
    int fd = eventfd(0,0);
    if(fd < 0) {
        perror("eventfd in CreateEventFd");
    }
    return fd;
}
void EventEpoll::HandleRead() {
    uint64_t u = 1;
    ssize_t res = read(_eventFd,&u,sizeof(uint64_t));
    if(res != sizeof(uint64_t)) {
         perror("read in HandleRead");
         return;
    } 
}
void EventEpoll::WakeUp() {
    uint64_t u = 1;
    ssize_t res = write(_eventFd,&u,sizeof(uint64_t));
    if(res != sizeof(uint64_t)) {
         perror("write in WakeUp");
         return;
    } 
}
void EventEpoll::DoPendingFunc() {
    //由于对于_pending的访问是多线程进行的，因此进行访问时需要加锁
    //此函数完成两个操作：将待执行函数从_pending中取出，然后进行执行
    //由于访问_pending时，_mutex一直处于加锁状态，那么会限制其他线程对于_pending的访问
    //因此需要将这两个操作进行分离
    
    vector<EventCallBackFunc> tmp;
    {
        MutexLockGuard autoLock(_mutex);
        tmp.swap(_pending);
    }

    for(auto & ecb : tmp) {
        ecb();    
    }
}
```

##### TcpServer.cc

```c++
#include"TcpServer.hpp"

TcpServer::TcpServer(const string & ip, const unsigned short & port)
: _acceptor(ip,port)
, _eventEpool(_acceptor)
{
}

TcpServer::~TcpServer() {

}

void TcpServer::Start() {
    _acceptor.Ready();
    _eventEpool.Loop();
}

void TcpServer::Stop() {
    _eventEpool.UnLoop();
}

void TcpServer::SetAllCallBack(CallBackFunc && onConnection
                               , CallBackFunc && onMessageFunc
                               , CallBackFunc && onCloseFunc) 
{
    _eventEpool.SetOnConnectionCallBack(std::move(onConnection));
    _eventEpool.SetOnMessageCallBack(std::move(onMessageFunc));
    _eventEpool.SetOnCloseCallBack(std::move(onCloseFunc));
}
```

---



##### test_file

```c++
#include"ToUpperServer.hpp"

int main(void) {
    ToUpperServer toUpperServer(4,10,"127.0.0.1",8888);
    toUpperServer.Start();
    return 0;
}
```

---

































