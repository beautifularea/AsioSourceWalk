` typedef class epoll_reactor reactor; `
asio针对不同的平台，实现了不同的reactor实现方式，比如` poll select epoll iocp `，咱就分析看epoll的代码。

```
```
