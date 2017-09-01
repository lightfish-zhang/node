# node与libuv

## 前言

node的最重要的事件循环`event loop`，是通过`libuv`库实现的，这是google开发的C语言库，兼容不同的操作系统

## 事件类型

- Network I/O事件， 根据OS平台不同，分别使用Linux上的epoll，OSX和BSD类OS上的kqueue，SunOS上的event ports以及Windows上的IOCP机制。
- File I/O事件，则使用thread pool，在其中进行阻塞式I/O。利用thread pool的方式实现异步请求处理，在各类OS上都能获得很好的支持。
    + 社区有人提出，在Linux下使用原生的NIO(非阻塞io)替换thread pool的建议 ,测试发现有3%的提升. 考虑到 NIO 对内核版本的依赖，利用thread pool的方式实现异步请求处理，在各类OS上都能获得很好的支持，相信是 libuv 作者权衡再三的结果。

- 时间事件， 在`event loop`处理完后，从时间事件的表中获取下一次最近事件的时间间隔`timeout`，将`timeout`设置为等待事件的超时时间`uv_run(loop, timeout);`，周而复始

## 源码分析

### 学习资源

[官方文档](http://docs.libuv.org)
[社区文档](http://nikhilm.github.io/uvbook/)
[libuv常见示例](https://github.com/thlorenz/libuv-dox)

### 从Linux角度去看

- 笔者是Linux爱好者（其实不懂其他操作系统），基于Linux环境下去研究libuv源码
- libuv中对Linux封装的代码主要声明在`libuv/src/unix/linux-core.c`
- 笔者一开始就去找epoll系统调用，发现，使用了`syscall`调用，原来还有这种操作，学习了

```c
int uv__epoll_create(int size) {
#if defined(__NR_epoll_create)
  return syscall(__NR_epoll_create, size);
#else
  return errno = ENOSYS, -1;
#endif
}

int uv__epoll_ctl(int epfd, int op, int fd, struct uv__epoll_event* events) {
#if defined(__NR_epoll_ctl)
  return syscall(__NR_epoll_ctl, epfd, op, fd, events);
#else
  return errno = ENOSYS, -1;
#endif
}

int uv__epoll_wait(int epfd,
                   struct uv__epoll_event* events,
                   int nevents,
                   int timeout) {
#if defined(__NR_epoll_wait)
  return syscall(__NR_epoll_wait, epfd, events, nevents, timeout);
#else
  return errno = ENOSYS, -1;
#endif
}
```

- 另外，多进程之间分享文件描述符的`sendmsg/resvmsg`

```c
int uv__sendmmsg(int fd,
                 struct uv__mmsghdr* mmsg,
                 unsigned int vlen,
                 unsigned int flags);
int uv__utimesat(int dirfd,
                 const char* path,
                 const struct timespec times[2],
                 int flags);
```

- 获取时间`int clock_gettime(clockid_t clk_id, struct timespec *tp)`
- 修改进程的名字`int prctl(PR_SET_NAME, title)`
- 获取cpu信息，长见识了，还有这写法
```c
static unsigned long read_cpufreq(unsigned int cpunum) {
  unsigned long val;
  char buf[1024];
  FILE* fp;

  snprintf(buf,
           sizeof(buf),
           "/sys/devices/system/cpu/cpu%u/cpufreq/scaling_cur_freq",
           cpunum);

  fp = fopen(buf, "r");
  if (fp == NULL)
    return 0;

  if (fscanf(fp, "%lu", &val) != 1)
    val = 0;

  fclose(fp);

  return val;
}
```