接上篇 [第一集 endpoint]()  
___涉及到的文件___  
[tcp.hpp](\asio\ip\tcp.hpp)  
[udp.hpp](\asio\ip\udp.hpp)  
[basic_endpoint](\asio\ip\basic_endpoint.hpp)

我们这么来使用endpoint，比如client端：
```
#include <boost/asio.hpp>
#include <iostream>
using namespace boost;
int main()
{
      std::string raw_ip_address = "127.0.0.1";
      unsigned short port_num = 3333;

      boost::system::error_code ec;
      asio::ip::address ip_address = asio::ip::address::from_string(raw_ip_address, ec);
      if (ec.value() != 0) {
          std::cout
          << "Failed to parse the IP address. Error code = "
          << ec.value() << ". Message: " << ec.message();
          return ec.value();
      }

      //此处的endpoint!
      asio::ip::tcp::endpoint ep(ip_address, port_num);
          //...TODO:
      return 0;
}
```
我们之前分析的endpoint属于detail namespace， 上面的例子为什么是tcp???
```
namespace asio {
namespace ip {
namespace detail {

// Helper class for implementating an IP endpoint.
class endpoint
{
};
} //end detail
} //end ip
} //end asio
```
先找到tcp相关代码
```
//tcp.hpp
class tcp
{
public:
  /// The type of a TCP endpoint.
  typedef basic_endpoint<tcp> endpoint;
};
```
endpoint 是类模板` basic_endpoint<> `的实例化。
再看udp部分代码
```
class udp
{
public:
  /// The type of a UDP endpoint.
  typedef basic_endpoint<udp> endpoint;
};
```
endpoint 也是basic_endpoint<>类模板的实例化。  

---

找到basic_endpoint类，它是一个模板类，包含一个private成员变量就是上篇中的endpoint类型。
```
namespace ip
{
template <typename InternetProtocol>
class basic_endpoint
{
  public:
  /// The protocol type associated with the endpoint.
  typedef InternetProtocol protocol_type;

   //TODO
   //...

   private:
    asio::ip::detail::endpoint impl_;
};
}
```

---

下面看该类的主要函数

带参数的构造函数，通过初始化列表来初始化endpoint实例
```
basic_endpoint(const InternetProtocol& internet_protocol,
    unsigned short port_num)
  : impl_(internet_protocol.family(), port_num)
{
}
```

查看协议类型
```
protocol_type protocol() const
{
  if (impl_.is_v4())
    return InternetProtocol::v4();
  return InternetProtocol::v6();
}
```

类似这样的实现，都是对endpoint类的一个封装，没有其他新的内容，涉及到的模板类也不是很复杂。****关于模板编程请自行查询****
```
void port(unsigned short port_num)
{
  impl_.port(port_num);
}

asio::ip::address address() const
{
  return impl_.address();
}
```
