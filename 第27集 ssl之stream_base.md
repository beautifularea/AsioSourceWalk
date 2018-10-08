` stream_base `很简单，就是声明了` handshake_type `. 主动连接的一方叫做:active peer, 是作为客户端的角色, 为client。
被动连接方叫做: posstive peer，是作为服务端的角色, 为server.

```
/// The stream_base class is used as a base for the asio::ssl::stream
/// class template so that we have a common place to define various enums.
class stream_base
{
public:
  /// Different handshake types.
  enum handshake_type
  {
    /// Perform handshaking as a client.
    client,

    /// Perform handshaking as a server.
    server
  };

protected:
  /// Protected destructor to prevent deletion through this type.
  ~stream_base()
  {
  }
};
```
