# Linux网络编程常用api

## 定时器相关

```cpp
#include <sys/timerfd.h>

// 创建定时器，返回文件描述符timerfd
// clockid: 指定定时器类型，CLOCK_REALTIME 跟随系统时间；CLOCK_MONOTONIC 不跟随系统时间
// flags: TFD_NONBLOCK 非阻塞；TFD_CLOEXEC 在执行 exec 后，关闭对应的文件描述符
int timerfd_create(int clockid, int flags);

// 设置超时时间
    struct timespec {
        time_t tv_sec;                /* Seconds */
        long   tv_nsec;               /* Nanoseconds */
    };
    struct itimerspec {
        struct timespec it_interval;  /*定时周期，若为0，则只触发一次*/
        struct timespec it_value;     /*第一次超时的时间，相对时间*/
    };
// new_value为设置的超时时间，old_value用于保存旧的超时时间
// flags: TFD_TIMER_ABSTIME 使用绝对时间
int timerfd_settime(int fd, int flags, const struct itimerspec *new_value, struct itimerspec *old_value);

//获取超时时间
int timerfd_gettime(int fd, struct itimerspec *curr_value)

```

