上篇 [第1集](第1集 endpoint续)中client的例子，提到了` asio::ip::address `,开始对它进行分析。  

****涉及到的问题****  
[address.hpp](asio\ip\address.hpp)  

先看address所属的namespace.
```
namespace asio {
namespace ip {
class address
{
};
} //end ip
} //end asio
```
` class address `的实例变量如下
```
private:
  // The type of the address.
  enum { ipv4, ipv6 } type_;

  // The underlying IPv4 address.
  asio::ip::address_v4 ipv4_address_;

  // The underlying IPv6 address.
  asio::ip::address_v6 ipv6_address_;
```
可以看出来，这个address的实现类似于` basic_endpoint<> `模板类，而并没有采用模板的方式。

---

下面列举几个方法，其实都是对具体IPv4/IPv6 address的封装。

不带参数的构造函数，默认还是为IPv4的地址
```
address::address()
  : type_(ipv4),
    ipv4_address_(),
    ipv6_address_()
{
}

address::address(const asio::ip::address_v4& ipv4_address)
  : type_(ipv4),
    ipv4_address_(ipv4_address),
    ipv6_address_()
{
}

address::address(const asio::ip::address_v6& ipv6_address)
  : type_(ipv6),
    ipv4_address_(),
    ipv6_address_(ipv6_address)
{
}
```

赋值运算符
```
address& address::operator=(const address& other)
{
  type_ = other.type_;
  ipv4_address_ = other.ipv4_address_;
  ipv6_address_ = other.ipv6_address_;
  return *this;
}
```

make_address实现，会先生成IPv6的地址，如果出错，则再去生成IPv4.
```
address make_address(const char* str)
{
  asio::error_code ec;
  address addr = make_address(str, ec);
  asio::detail::throw_error(ec);
  return addr;
}

address make_address(const char* str, asio::error_code& ec)
{
  asio::ip::address_v6 ipv6_address =
    asio::ip::make_address_v6(str, ec);
  if (!ec)
    return address(ipv6_address);

  asio::ip::address_v4 ipv4_address =
    asio::ip::make_address_v4(str, ec);
  if (!ec)
    return address(ipv4_address);

  return address();
}
```
地址转换
```
asio::ip::address_v4 address::to_v4() const
{
  if (type_ != ipv4)
  {
    bad_address_cast ex;
    asio::detail::throw_exception(ex);
  }
  return ipv4_address_;
}
```
变换为字符串
```
std::string address::to_string() const
{
  if (type_ == ipv6)
    return ipv6_address_.to_string();
  return ipv4_address_.to_string();
}
```
来个三连拍，判断loopback/unspecified/multicast
```
bool address::is_loopback() const
{
  return (type_ == ipv4)
    ? ipv4_address_.is_loopback()
    : ipv6_address_.is_loopback();
}

bool address::is_unspecified() const
{
  return (type_ == ipv4)
    ? ipv4_address_.is_unspecified()
    : ipv6_address_.is_unspecified();
}

bool address::is_multicast() const
{
  return (type_ == ipv4)
    ? ipv4_address_.is_multicast()
    : ipv6_address_.is_multicast();
}
```
该类比较简单，对ipv4/v6的封装，具体可看相应的实现。
