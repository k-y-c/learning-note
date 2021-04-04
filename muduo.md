# muduo学习

## base库

### 1. Timestamp

实现微秒级时间戳，仅含有一个64位整型成员变量。

```c++
private:

  int64_t microSecondsSinceEpoch_;
```

#### 1.1 关于操作符重载：

Timestamp类继承自boost的equality_comparable和less_than_comparable，这两个类中将运算符重载声明为友元函数

```c++
template <class T, class B = operators_detail::empty_base<T> >
struct less_than_comparable1 : B
{
     friend BOOST_OPERATORS_CONSTEXPR bool operator>(const T& x, const T& y)  { return y < x; }
     friend BOOST_OPERATORS_CONSTEXPR bool operator<=(const T& x, const T& y) { return !static_cast<bool>(y < x); }
     friend BOOST_OPERATORS_CONSTEXPR bool operator>=(const T& x, const T& y) { return !static_cast<bool>(x < y); }
};
```

所以只需要在类外定义< and ==的运算符重载即可实现其他部分。

```c++
inline bool operator<(Timestamp lhs, Timestamp rhs)
{
  return lhs.microSecondsSinceEpoch() < rhs.microSecondsSinceEpoch();
}

inline bool operator==(Timestamp lhs, Timestamp rhs)
{
  return lhs.microSecondsSinceEpoch() == rhs.microSecondsSinceEpoch();
}
```

#### 1.2 获取当前时刻

Timestamp中含有静态成员函数` static Timestamp now();`

```c++
Timestamp Timestamp::now()
{
  struct timeval tv;
  gettimeofday(&tv, NULL);
  int64_t seconds = tv.tv_sec;
  return Timestamp(seconds * kMicroSecondsPerSecond + tv.tv_usec);
}
```

`struct timeval`内部含有两个整数，分别表示s和us，gettimeofday系统调用获取当前时间，返回的`seconds * kMicroSecondsPerSecond + tv.tv_usec`即微秒级别的时间戳。

#### 1.3 获取时间差和增加时间

Timestamp头文件中提供两个全局函数`inline double timeDifference(Timestamp high, Timestamp low)`和`inline Timestamp addTime(Timestamp timestamp, double seconds)`，分别用于获取时间差和增加时间。

```c++
inline double timeDifference(Timestamp high, Timestamp low)
{
  int64_t diff = high.microSecondsSinceEpoch() - low.microSecondsSinceEpoch();
  return static_cast<double>(diff) / Timestamp::kMicroSecondsPerSecond;
}

///
/// Add @c seconds to given timestamp.
///
/// @return timestamp+seconds as Timestamp
///
inline Timestamp addTime(Timestamp timestamp, double seconds)
{
  int64_t delta = static_cast<int64_t>(seconds * Timestamp::kMicroSecondsPerSecond);
  return Timestamp(timestamp.microSecondsSinceEpoch() + delta);
}

```

### 2. Atomic（原子操作）

该头文件中包含AtomicIntegerT类，该类采用模板，含有一个成员变量。

```c++
 private:
  volatile T value_;
```

> volatile提醒编译器它后面所定义的变量随时都有可能改变，因此编译后的程序每次需要存储或读取这个变量的时候，都会直接从**变量地址**中读取数据。如果没有volatile关键字，则编译器可能优化读取和存储，可能暂时**使用寄存器中的值**，如果这个变量由别的程序更新了的话，将出现不一致的现象。

该类使用两个系统调用实现原子操作

```c++
  T get()
  {
    // in gcc >= 4.7: __atomic_load_n(&value_, __ATOMIC_SEQ_CST)
    return __sync_val_compare_and_swap(&value_, 0, 0);
  }

  T getAndAdd(T x)
  {
    // in gcc >= 4.7: __atomic_fetch_add(&value_, x, __ATOMIC_SEQ_CST)
    return __sync_fetch_and_add(&value_, x);
  }
```

get()函数获取当前值，getAndAdd(T x)为先获取后自增，由这两个原子操作可以得到++、--等加减原子操作。

### 3.  互斥锁和条件变量

Mutex.h和Condition.h分别实现了互斥锁和条件变量

#### 3.1 互斥锁

Mutex.h封装了两个类`class MutexLock`和`class MutexLockGuard`。

##### 关于MutexLock的实现

MutexLock类中含有两个成员变量和一个内部类。

```c++
class UnassignGuard : noncopyable
pthread_mutex_t mutex_;
pid_t holder_;
```

`class UnassignGuard`留给条件变量使用。

MutexLock提供4个外部访问接口

```c++
  bool isLockedByThisThread() const
  {
    return holder_ == CurrentThread::tid();
  }

  void assertLocked() const ASSERT_CAPABILITY(this)
  {
    assert(isLockedByThisThread());
  }

  void lock()

  void unlock()
```

加锁过程会调用`pthread_mutex_lock(&mutex_)`，并且将holder_设置为加锁线程的tid，解锁过程反之亦然。

---

##### 基于RAII实现自动加锁解锁

MutexLockGuard类中包含一个MutexLock（组合关系），基于RAII实现自动加锁解锁。

> ```c++
> // Use as a stack variable, eg.
> 
>  int Foo::size() const
> 
>  {
> 
>    MutexLockGuard lock(mutex_);
> 
>    return data_.size();
> 
>  }
> ```

MutexLockGuard的定义必须分配一个互斥锁，在其构造函数和析构函数中实现对锁的持有与释放，从而实现RAII。

```c++
class MutexLockGuard : noncopyable
{
 public:
  explicit MutexLockGuard(MutexLock& mutex)
    : mutex_(mutex)
  {
    mutex_.lock();
  }

  ~MutexLockGuard() RELEASE()
  {
    mutex_.unlock();
  }

 private:

  MutexLock& mutex_;
};
```

---

#### 3.2 条件变量

- 使用方法

  - Thread1 调用 cond.wait()，进入阻塞，等待条件到来
  - Thread2 调用 cond.notify()或者notifyAll()唤醒等待条件的线程

- 具体过程

  - Thread1

    锁住mutex

    ​		while（cond.wait()）{}

    解锁mutex

  - Thread2

    锁住mutex

    ​		更改条件[cond.notify()或者notifyAll()]

    解锁mutex

- cond.wait()里面发生了什么？

  1. 解锁mutex
  2. 等待条件（阻塞）
  3. 加锁mutex

- cond.notify()中发生了什么？

  唤醒条件（发送信号）

---

`class Condition`中包含两个成员变量。

```c++
 private:
  MutexLock& mutex_;
  pthread_cond_t pcond_;
```

提供三个接口供外部调用

```c++
  void wait()
  {
    MutexLock::UnassignGuard ug(mutex_);
    MCHECK(pthread_cond_wait(&pcond_, mutex_.getPthreadMutex()));
  }

  // returns true if time out, false otherwise.
  bool waitForSeconds(double seconds);

  void notify()
  {
    MCHECK(pthread_cond_signal(&pcond_));
  }

  void notifyAll()
  {
    MCHECK(pthread_cond_broadcast(&pcond_));
  }
```

#### 3.3 CountDownLatch

实现一个等待计数的类

```c++
class CountDownLatch : noncopyable
{
 public:

  explicit CountDownLatch(int count);

  void wait();

  void countDown();

  int getCount() const;

 private:
  mutable MutexLock mutex_;
  Condition condition_ GUARDED_BY(mutex_);
  int count_ GUARDED_BY(mutex_);
};
```

主线程调用wait()，当count_减至0是唤醒条件。

### `4`. 线程相关类

#### 4.1 CurrentThread

CurrentThread命名空间中包含4个全局变量。

```c++
namespace CurrentThread
{
  // internal
  extern __thread int t_cachedTid;
  extern __thread char t_tidString[32];
  extern __thread int t_tidStringLength;
  extern __thread const char* t_threadName;
}
```

extern关键字表示该变量将会在后面或其他文件中定义。__thread关键字表示该变量是**线程私有**的。

CurrentThread命名空间还提供几个全局函数供外部接口访问。

```c++
  inline int tid() // 当前线程id
  inline const char* tidString() // for logging
  inline int tidStringLength() // for logging
  inline const char* name()
  bool isMainThread();
  string stackTrace(bool demangle);
```

其中tid()调用cacheTid()实现对上面全局变量的更新，最底层调用`return static_cast<pid_t>(::syscall(SYS_gettid));`实现。

***Ps.cacheTid()和isMainThread()两个函数的定义在Thread.cc中实现。***

---

#### 4.2 Thread.h

Class Thread对线程进行封装，其构造函数需要传递线程函数和线程名称。

```c++
Thread::Thread(ThreadFunc func, const string& n)
  //typedef std::function<void ()> ThreadFunc;
  //typedef boost::function<void ()> ThreadFunc;两种做法
  : started_(false),
    joined_(false),
    pthreadId_(0),
    tid_(0),
    func_(std::move(func)),//move为右值引用
    name_(n),
    latch_(1)//用于等待线程启动
{
  setDefaultName();//构造函数的第二个参数有设置默认值string()，若未设置线程名，此处将为其赋值默认名
  //snprintf(buf, sizeof buf, "Thread%d", num);
}
```

通过调用thread.start()和thread.join()实现线程启动和阻塞。

##### Thread::start()实现原理

```c++
void Thread::start()
{//……
  detail::ThreadData* data = new detail::ThreadData(func_, name_, &tid_, &latch_);//新建一个ThreadData对象，将Thread类中的线程函数等参数传入该对象。
  if (pthread_create(&pthreadId_, NULL, &detail::startThread, data))//将data作为参数传入全局函数startThread（void *)    
  {
    //thread start failed……
  }
  else
  {
    latch_.wait();//等待线程启动
    assert(tid_ > 0);
  }
}

void* startThread(void* obj)//ThreadData*隐式转换为void*
{
  ThreadData* data = static_cast<ThreadData*>(obj);//再转回ThreadData*，实现绑定
  data->runInThread();
  delete data;
  return NULL;
}

void ThreadData::runInThread()
  {
    *tid_ = muduo::CurrentThread::tid();
    tid_ = NULL;
    latch_->countDown();//唤醒主线程
    latch_ = NULL;

    muduo::CurrentThread::t_threadName = name_.empty() ? "muduoThread" : name_.c_str();
    ::prctl(PR_SET_NAME, muduo::CurrentThread::t_threadName);//设置进程名称
    try
    {
      func_();//用户定义的子线程函数
    }
  //exception……
	}
```

***为什么作者对于线程的封装要利用ThreadData中间类的指针作为参数进行传递，里面是否包含何种设计模式？***

#### 4.3 ThreadPool（线程池）

线程池的用法类似生产者消费者模型。

![fig1](/Users/chenkeyu/Desktop/c++/mynote/fig/fig1.png)

用法：

```c++
void foo(){};

int main(){
  muduo::ThreadPool threadpool;
  treadpool.start(5);//启动5个工作线程
  for(int i = 0;i<1000:++i){
    threadpool.run(foo);//往task queue中添加任务，
  }
}
```

```c++
void ThreadPool::start(int numThreads) //该函数用于启动工作线程
{
  assert(threads_.empty());
  running_ = true;
  threads_.reserve(numThreads);
  for (int i = 0; i < numThreads; ++i)
  {
    char id[32];
    snprintf(id, sizeof id, "%d", i+1);
    threads_.emplace_back(new muduo::Thread(std::bind(&ThreadPool::runInThread, this), name_+id));//往线程池中添加工作线程
    //std::vector<std::unique_ptr<muduo::Thread>> threads_;使用智能指针管理Thread*
    threads_[i]->start();
  }
  if (numThreads == 0 && threadInitCallback_)
  {
    threadInitCallback_();
  }
}
```

```c++
void ThreadPool::runInThread()
{
  try
  {
    if (threadInitCallback_)
    {
      threadInitCallback_();//线程初始化callback
    }
    while (running_)
    {
      Task task(take());//take()从task queue中取任务
      if (task)
      {
        task();
      }
    }
  }
  //exception……
}
```

`void ThreadPool::run(Task task)`和`ThreadPool::Task ThreadPool::take()`分别代表生产者和消费者，一方往task queue中添加任务，一方从task queue中取出任务

```c++
void ThreadPool::run(Task task)
{
  if (threads_.empty())
  {
    task(); // 若工作线程为0，则直接执行任务，该类支持0工作线程的情况。
  }
  else
  {
    MutexLockGuard lock(mutex_);	//	上锁,当lock离开其生存期时，锁自动销毁
    while (isFull() && running_)	//	当task queue满，则等待条件
    {													/*		mutable MutexLock mutex_;		*/
      notFull_.wait();				/*		Condition notEmpty_;				*/
  	}													/*		Condition notFull_;					*/
    if (!running_) return;
    assert(!isFull());

    queue_.push_back(std::move(task));	//	右值引用，添加任务
    notEmpty_.notify();	//	发送不为空条件信号
  }
}
```

```c++
ThreadPool::Task ThreadPool::take()
{
  MutexLockGuard lock(mutex_);	//	上锁
  // always use a while-loop, due to spurious wakeup
  while (queue_.empty() && running_)
  {
    notEmpty_.wait();		//	task queue为空，等待notEmpty_信号
  }
  Task task;
  if (!queue_.empty())	//	此处条件应该是比为true……
  {
    task = queue_.front();
    queue_.pop_front();	//	取任务
    if (maxQueueSize_ > 0)
    {
      notFull_.notify();	//	通知task queue不为满	
    }
  }
  return task;
}
```

---

### 5. singleton（单例）

##### 进程单例对象

单例模式保证了在某进程中最多只存在一个对象，muduo的singleton使用了`pthread_once(&ponce_, &Singleton::init);`,该函数保证Singleton::init()在进程中只执行一次。

```c++
class Singleton : noncopyable
{
 public:
  Singleton() = delete;
  ~Singleton() = delete;

  static T& instance()
  {
    pthread_once(&ponce_, &Singleton::init);
    assert(value_ != NULL);
    return *value_;
  }

 private:
  static void init()
  {
    value_ = new T();
    if (!detail::has_no_destroy<T>::value)
    {
      ::atexit(destroy);
    }
  }
```

值得注意的是，在`if (!detail::has_no_destroy<T>::value)`处，作者使用了**元编程**（黑魔法？？），用于判断模板T中是否含有no_destroy成员，以便于为其注册析构函数，删除指针，[具体详见这里](https://blog.csdn.net/qu1993/article/details/109474327)。

---

##### 线程本地单例（每个线程中的单例对象）

***此处先留白***

---

### 6. 日志

***留白……***

-----

## net库

### 1. IP地址封装

muduo将IP地址封装到`class InetAddress`，这个类的成员变量是一个union（同时只存在一个数据）。

```c++
 private:
  union
  {
    struct sockaddr_in addr_;
    struct sockaddr_in6 addr6_;
  };
```

该类封装了3个构造函数（除开ivp6）。

```c++
 /// Mostly used in TcpServer listening.
 /// eg. InetAddress ip = InetAddress(8099)
  explicit InetAddress(uint16_t port = 0, bool loopbackOnly = false, bool ipv6 = false);

  /// Constructs an endpoint with given ip and port.
  /// @c ip should be "1.2.3.4"
  InetAddress(StringArg ip, uint16_t port, bool ipv6 = false);

  /// Constructs an endpoint with given struct @c sockaddr_in
  /// Mostly used when accepting new connections
  explicit InetAddress(const struct sockaddr_in& addr)
    : addr_(addr)
  { }
```

### 2. Socket封装

muduo封装了`class Socket`，其成员变量为sock的文件描述符。

```c++
 private:
  const int sockfd_;
```

`SocketsOps.h`和`SocketsOps.cc`在sockets命名空间内封装了一些接口函数，可供调用。

套接字的绑定由`class Acceptor`的构造函数实现。

```c++
Acceptor::Acceptor(EventLoop* loop, const InetAddress& listenAddr, bool reuseport)
  : loop_(loop),
    acceptSocket_(sockets::createNonblockingOrDie(listenAddr.family())),
    acceptChannel_(loop, acceptSocket_.fd()),
    listening_(false),
    idleFd_(::open("/dev/null", O_RDONLY | O_CLOEXEC))
{
  assert(idleFd_ >= 0);
  acceptSocket_.setReuseAddr(true);
  acceptSocket_.setReusePort(reuseport);
  acceptSocket_.bindAddress(listenAddr);
  acceptChannel_.setReadCallback(
      std::bind(&Acceptor::handleRead, this));
}
```



### 3.关于shutdown

TcpConnection中的shutdown函数会调用`::shutdown(sockfd, SHUT_WR)`，关闭当前连接fd的写事件，但不会直接关闭连接。

测试：在echo案例中，使用Telnet通信时，TcpConnection调用shutdown会直接导致Telnet关闭连接，而使用nc时，则仅仅是无法接受到msg。

### 4. TcpServer 连接到来时的流程

![Tcpnewconn](/Users/chenkeyu/Desktop/c++/mynote/fig/Tcpnewconn.jpg)

---

### 5. TcpServer连接被动关闭时流程

![Tcpcloseconn](/Users/chenkeyu/Desktop/c++/mynote/fig/Tcpcloseconn.jpg)

