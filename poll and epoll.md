# 关于poll和epoll的解读

```c++
struct pollfd {
	int     fd;
	short   events;
	short   revents;
};

typedef union epoll_data
{
  void *ptr;
  int fd;
  uint32_t u32;
  uint64_t u64;
} epoll_data_t;

struct epoll_event
{
  uint32_t events;	/* Epoll events */
  epoll_data_t data;	/* User data variable */
}
```

## poll

```c++
struct pollfd {
	int     fd;
	short   events;
	short   revents;
};

int poll(struct pollfd fds[], nfds_t nfds, int timeout);
```

poll自身**不含有文件描述符** *（这与epoll不同）*

调用poll函数，第一个参数`struct pollfd *`集合了所有要关注的文件描述符，`nfds`是第一个参数数组的大小，timeout代表超时时间。函数返回值为有活动的事件个数。

`pollfd.events`代表要关注的事件，用比特位表示，由用户设置，eg.

```c++
#define POLLIN          0x0001          /* any readable data available */
#define POLLPRI         0x0002          /* OOB/Urgent readable data */
#define POLLOUT         0x0004          /* file descriptor is writeable */
#define POLLRDNORM      0x0040          /* non-OOB/URG data available */
#define POLLWRNORM      POLLOUT         /* no write type differentiation */
#define POLLRDBAND      0x0080          /* OOB/Urgent readable data */
#define POLLWRBAND      0x0100          /* OOB/Urgent data can be written */
```

`pollfd.revents`是返回该文件描述符的事件类型，若为0，则代表无事件发生，也是由bit位表示，同上。

poll模式下，在每次调用poll之后，需要遍历`struct pollfd *`，挑选出有revents的fd进行处理。

## epoll

`int epoll_create1 (int __flags) `,该函数生成一个epoll文件描述符，返回其fd。

`int epoll_ctl (int __epfd, int __op, int __fd,struct epoll_event *__event)`,用于向epollfd中增删或者修改需要关注的fd，`__epfd`是epollfd，`__op`代表需要进行的操作，see below……

```c++
#define EPOLL_CTL_ADD 1	/* Add a file descriptor to the interface.  */
#define EPOLL_CTL_DEL 2	/* Remove a file descriptor from the interface.  */
#define EPOLL_CTL_MOD 3	/* Change file descriptor epoll_event structure.  */
```

`__fd`是需要关注的文件描述符，`__event`是需要关注的事件，`epoll_event.events`用于设置关注的事件类型，同poll.events类似。`epoll_event.data`是一个union，可用于设置用户自定义的变量，在muduo当中，`epoll_event.data`设置为`*channel`

```c++
typedef union epoll_data
{
  void *ptr;
  int fd;
  uint32_t u32;
  uint64_t u64;
} epoll_data_t;

struct epoll_event
{
  uint32_t events;	/* Epoll events */
  epoll_data_t data;	/* User data variable */
}
```

进行了以上设置之后，epoll就准备好了要关注的fd，接下来进行epoll_wait（类似poll）

`int epoll_wait (int __epfd, struct epoll_event *__events, int __maxevents, int __timeout);`

该函数用于等待事件到来，用户需要准备一个容器*__events用于接收事件，`__maxevents`代表本次epoll的最大可接收事件，`__timeout`是超时时间。函数返回事件个数。

值得注意的是，epoll不同于poll，不需要遍历整个fd数组，`epoll_wait`会修改`struct epoll_event *__events`的内容，用户只需读取前`num_event`个`__events`即可，相较于poll省去了遍历fd数组的时间，可用于连接数大而活动少的情况。

`__events[i].events`是接收到的事件，类比poll的revents。`epoll_event`结构体中没有像`pollfd.fd`的成员，故其`epoll_event.data`可用于设置fd，在muduo中设置成了`*channel`,因为channel中包含了fd。



***上述内容仅解读了poll和epoll的调用过程，但对内部原理貌似并没有深入分析，留白先……***

## muduo的poll

muduo定义了`class Poller`,这个是一个抽象基类，在整个muduo库中，poll是唯一使用面向对象编程思想的。

```c++
class Poller : noncopyable
{
 public:
  typedef std::vector<Channel*> ChannelList;

  Poller(EventLoop* loop);
  virtual ~Poller();

  /// Polls the I/O events.
  /// Must be called in the loop thread.
  virtual Timestamp poll(int timeoutMs, ChannelList* activeChannels) = 0;

  /// Changes the interested I/O events.
  /// Must be called in the loop thread.
  virtual void updateChannel(Channel* channel) = 0;

  /// Remove the channel, when it destructs.
  /// Must be called in the loop thread.
  virtual void removeChannel(Channel* channel) = 0;

  virtual bool hasChannel(Channel* channel) const;

  static Poller* newDefaultPoller(EventLoop* loop);

  void assertInLoopThread() const
  {
    ownerLoop_->assertInLoopThread();
  }

 protected:
  typedef std::map<int, Channel*> ChannelMap;
  ChannelMap channels_;

 private:
  EventLoop* ownerLoop_;
};
```

`  static Poller* newDefaultPoller(EventLoop* loop);`用于创建一个poller对象，具体是poll还是epoll在函数内实现。

```c++
Poller* Poller::newDefaultPoller(EventLoop* loop)
{
  if (::getenv("MUDUO_USE_POLL"))
  {
    return new PollPoller(loop);
  }
  else
  {
    return new EPollPoller(loop);
  }
}
```

poller中管理一个ChannelMap，从下标到Channel*的映射，包含了所有要关注的channel。

poller的核心功能是得到I/O事件，并填充到`ChannelList* activeChannels`中，poll的调用在`EventLoop::loop()`中进行，activeChannels的生存期由EventLoop管理，通过传指针的方式到poller中。

***两种poll方式的原理已经在上文提到，但具体，如ChannelMap的更新以及activeChannels的更新还有很多细节未提，留后日分解***

