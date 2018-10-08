来个大活儿，那就先从io_service的例子开始吧~  
```
#include <boost/asio.hpp>
#include <iostream>
using namespace boost;
int main()
{
    // Step 1. An instance of 'io_service' class is required by
    // socket constructor.
    asio::io_service ios;
    // Step 2. Creating an object of 'tcp' class representing
    // a TCP protocol with IPv4 as underlying protocol.
    asio::ip::tcp protocol = asio::ip::tcp::v4();
    // Step 3. Instantiating an active TCP socket object.
    asio::ip::tcp::socket sock(ios);
    // Used to store information about error that happens
    // while opening the socket.
    boost::system::error_code ec;
    // Step 4. Opening the socket.
    sock.open(protocol, ec);
    if (ec.value() != 0) {
    // Failed to open the socket.
    std::cout
    << "Failed to open the socket! Error code = "
    << ec.value() << ". Message: " << ec.message();
    return ec.value();
    }
    return 0;
}
```

___涉及到的文件___  
asio\io_service.hpp  
asio\io_context.hpp

先看` io_service `的定义，可以看到 ` io_service `是` io_context `的别名。
```
namespace asio {

typedef io_context io_service;

} // namespace asio
```
直接看` io_context `的实现，在源码中作者做了一些介绍，此处就不翻译了。  
```
namespace detail {

  typedef class scheduler io_context_impl;

} // namespace detail

class io_context
  : public execution_context
{
private:
  typedef detail::io_context_impl impl_type;

  // The implementation.
  //真正实现对象
  impl_type& impl_;

//几个内部类
public:
    class executor_type;
    friend class executor_type;

    class work;
    friend class work;

    class service;

    class strand;
};
```

---

#### 1. 构造函数
创建一个` class scheduler `对象初始化impl_,  
```
io_context::io_context()
  : impl_(add_impl(new impl_type(*this, ASIO_CONCURRENCY_HINT_DEFAULT)))
{
}
```
` # define ASIO_CONCURRENCY_HINT_DEFAULT -1 `, 其中` add_impl `
```
private:
  // Helper function to add the implementation.
  io_context::impl_type& io_context::add_impl(io_context::impl_type* impl)
  {
    asio::detail::scoped_ptr<impl_type> scoped_impl(impl);
    asio::add_service<impl_type>(*this, scoped_impl.get());
    return *scoped_impl.release();
  }
```
#### 2. 析构函数
```
io_context::~io_context()
{
}
```
#### 3. run
Run the io_service object's event processing loop. 其实是调用了` class scheduler ` 的 ` run `方法开启event loop. 其他的方法： ` poll / poll_one / run_one / stop / stopped / restart / ` 都是直接调用 impl 的实现。
```
io_context::count_type io_context::run()
{
  asio::error_code ec;
  count_type s = impl_.run(ec);
  asio::detail::throw_error(ec);
  return s;
}
```
写之前觉得这篇会很难，结构它又是一个包装类，封装了真正的实现，下篇分析下它的四个内部类的实现，基本上可以断定是一些helper类。
