下面看` address_v4 `的具体实现  
```
namespace asio {
namespace ip {
  class address_v4
  {
    //TODO:
  };
}
}
```

成员变量相关内容
```
//C++0x 中使用std::array
#if defined(GENERATING_DOCUMENTATION)
  typedef array<unsigned char, 4> bytes_type;
#else
  typedef asio::detail::array<unsigned char, 4> bytes_type;
#endif

asio::detail::in4_addr_type addr_;
```
在` socket_types `中定义 ` typedef in_addr in4_addr_type; `
```
struct in_addr {
    //in_addr_t一般为32位的unsigned int，其字节顺序为网络字节序，即该无符号数采用大端字节序。
    in_addr_t s_addr;

};
```
看主要方法实现

---
```
/// Default constructor.
  address_v4()
  {
    addr_.s_addr = 0;
  }

address_v4::address_v4(const address_v4::bytes_type& bytes)
{
  using namespace std; // For memcpy.
  memcpy(&addr_.s_addr, bytes.data(), 4);
}

address_v4::address_v4(address_v4::uint_type addr)
{
  addr_.s_addr = asio::detail::socket_ops::host_to_network_long(
      static_cast<asio::detail::u_long_type>(addr));
}
```
三连拍实现
```
/// Obtain an address object that represents the loopback address.
static address_v4 loopback()
{
  return address_v4(0x7F000001);
}

/// Obtain an address object that represents the broadcast address.
static address_v4 broadcast()
{
  return address_v4(0xFFFFFFFF);
}

bool address_v4::is_loopback() const
{
  return (to_uint() & 0xFF000000) == 0x7F000000;
}

bool address_v4::is_unspecified() const
{
  return to_uint() == 0;
}

bool address_v4::is_class_a() const
{
  return (to_uint() & 0x80000000) == 0;
}

bool address_v4::is_class_b() const
{
  return (to_uint() & 0xC0000000) == 0x80000000;
}

bool address_v4::is_class_c() const
{
  return (to_uint() & 0xE0000000) == 0xC0000000;
}

bool address_v4::is_multicast() const
{
  return (to_uint() & 0xF0000000) == 0xE0000000;
}
```
to_bytes
```
address_v4::bytes_type address_v4::to_bytes() const
{
  using namespace std; // For memcpy.
  bytes_type bytes;
  memcpy(bytes.elems, &addr_.s_addr, 4);
  return bytes;
}
```
address ---> to_string 用inet_ntop来实现
```
std::string address_v4::to_string() const
{
  asio::error_code ec;
  char addr_str[asio::detail::max_addr_v4_str_len];
  const char* addr =
    asio::detail::socket_ops::inet_ntop(
        ASIO_OS_DEF(AF_INET), &addr_, addr_str,
        asio::detail::max_addr_v4_str_len, 0, ec);
  if (addr == 0)
    asio::detail::throw_error(ec);
  return addr;
}
```
char * ---> to address用inet_pton实现
```
address_v4 make_address_v4(
    const char* str, asio::error_code& ec)
{
  address_v4::bytes_type bytes;
  if (asio::detail::socket_ops::inet_pton(
        ASIO_OS_DEF(AF_INET), str, &bytes, 0, ec) <= 0)
    return address_v4();
  return address_v4(bytes);
}
```
string ---> to address
```
inline address_v4 address_v4::from_string(const std::string& str)
{
  return asio::ip::make_address_v4(str);
}
```
这个地方又用到了` inline `而不是 ` ASIO_DECL `，有点儿乱。

最后一个方法
```
template <typename Elem, typename Traits>
std::basic_ostream<Elem, Traits>& operator<<(
    std::basic_ostream<Elem, Traits>& os, const address_v4& addr)
{
  return os << addr.to_string().c_str();
}
```
