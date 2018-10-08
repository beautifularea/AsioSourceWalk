` tcp `也是比较简单的一个类
```
namespace asio {
namespace ip {

/// Encapsulates the flags needed for TCP.
class tcp
{
public:
  /// The type of a TCP endpoint.
  typedef basic_endpoint<tcp> endpoint;

  //static 方法
  static tcp v4()
  {
    return tcp(ASIO_OS_DEF(AF_INET));
  }

  int type() const
  {
    return ASIO_OS_DEF(SOCK_STREAM);
  }

  /// Obtain an identifier for the protocol.
  int protocol() const
  {
    return ASIO_OS_DEF(IPPROTO_TCP);
  }

  /// Obtain an identifier for the protocol family.
  int family() const
  {
    return family_;
  }

  //主要是下面三个类型。。。
  /// The TCP socket type.
  typedef basic_stream_socket<tcp> socket;

  /// The TCP acceptor type.
  typedef basic_socket_acceptor<tcp> acceptor;

  /// The TCP resolver type.
  typedef basic_resolver<tcp> resolver;

  private:
    // Construct with a specific family.
    explicit tcp(int protocol_family)
      : family_(protocol_family)
    {
    }

    //family
    int family_;
};
}
}

```
