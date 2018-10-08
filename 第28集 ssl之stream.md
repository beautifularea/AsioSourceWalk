* The stream class template provides asynchronous and blocking stream-oriented
* functionality using SSL.

```
* To use the SSL stream template with an ip::tcp::socket, you would write:
 asio::io_context io_context;
 asio::ssl::context ctx(asio::ssl::context::sslv23);
 asio::ssl::stream<asio:ip::tcp::socket> sock(io_context, ctx);
```

```
template <typename Stream>
class stream :
  public stream_base,
  private noncopyable
{
};
```
` stream ` 是一个模板类， 可以提供阻塞和非阻塞的ssl流(socket)通信，比如:
```
tcp ->
asio::ssl::stream<asio::ip::tcp::socket> tstream;

upd->
asio::ssl::stream<asio::ip::upd::socket> ustream;
```
这里有个比较容易混淆的地方 ` next_layer ` vs.` lowest_layer `
```
typedef basic_socket< Protocol, SocketService > lowest_layer_type;
typedef typename next_layer_type::lowest_layer_type lowest_layer_type;
```
```
next_layer_type& next_layer()
{
  //next_layer_的类型是 ： Stream next_layer_;
  return next_layer_;
}
lowest_layer_type& lowest_layer()
{
  return next_layer_.lowest_layer();
}
```
还是以` asio::ssl::stream<asio::ip::tcp::socket> stream_ `举例子，此处   
stream 就是 ` asio::ip::tcp::socket `  ，因此  
` next_layer ` 就是表示 ` asio::ip::tcp::socket `， 而socket又是个模板类 ` typedef basic_stream_socket< tcp > socket; `  
```
template<
    typename Protocol>
class basic_stream_socket :
  public basic_socket< Protocol >
```
` lowst_layer ` 因此表示为 ` basic_socket A basic_socket is always the lowest layer. `
****可以理解为这俩个方法是逐渐找父类的！！！****

看主要方法

---
` async_handshake `, 其中同步handshake是 ` handshake `方法，调用了` detail::io `可类比着看。
```
stream_.async_handshake(boost::asio::ssl::stream_base::client, std::bind (&QuorumFetch::_handshake_handler, this, std::placeholders::_1));
```
```
template <typename HandshakeHandler>
ASIO_INITFN_RESULT_TYPE(HandshakeHandler, void (asio::error_code))
async_handshake(handshake_type type, ASIO_MOVE_ARG(HandshakeHandler) handler)
{
  //检查类型
  ASIO_HANDSHAKE_HANDLER_CHECK(HandshakeHandler, handler) type_check;

  //存放异步结束后的结果，见1
  asio::async_completion<HandshakeHandler, void (asio::error_code)> init(handler);

  //执行异步操作，见2
  detail::async_io(next_layer_, core_, detail::handshake_op(type), init.completion_handler);

  //空方法，立即返回
  return init.result.get();
}
```

****2****
```
template <typename Stream, typename Operation, typename Handler>
inline void async_io(Stream& next_layer, stream_core& core, const Operation& op, Handler& handler)
{
  //io_op 是一个函数类。（重载了operator()方法）
  //见 [第29集]()
  io_op<Stream, Operation, Handler>(next_layer, core, op, handler)(asio::error_code(), 0, 1);
}
```

****1****
```
/// Helper template to deduce the handler type from a CompletionToken, capture
/// a local copy of the handler, and then create an async_result for the
/// handler.
template <typename CompletionToken, typename Signature>
struct async_completion
{
  /// The real handler type to be used for the asynchronous operation.
  typedef typename asio::async_result<typename decay<CompletionToken>::type, Signature>::completion_handler_type completion_handler_type;

  explicit async_completion(typename decay<CompletionToken>::type& token)
  : completion_handler(token), result(completion_handler)
  {
  }

  completion_handler_type completion_handler;

  /// The result of the asynchronous operation's initiating function.
  async_result<typename decay<CompletionToken>::type, Signature> result;
};

//async_result
template <typename CompletionToken, typename Signature>
class async_result
{
  typedef CompletionToken completion_handler_type;

  /// The return type of the initiating function.
  typedef void return_type;

  //立即返回。
  return_type get()
  {
    // Nothing to do.
  }

```

---

set_verify_mode

```
stream_.set_verify_mode(asio::ssl::verify_none)
```
```
ASIO_SYNC_OP_VOID set_verify_mode(verify_mode v, asio::error_code& ec)
{
  stream_core.engine.set_verify_mode
  core_.engine_.set_verify_mode(v, ec);

  ` # define ASIO_SYNC_OP_VOID_RETURN(e) return `
  ASIO_SYNC_OP_VOID_RETURN(ec);
}

asio::error_code engine::set_verify_mode(
    verify_mode v, asio::error_code& ec)
{
  ::SSL_set_verify(ssl_, v, ::SSL_get_verify_callback(ssl_));

  ec = asio::error_code();
  return ec;
}
```

---

async_write_some

```
boost::array<char, 1> buffer;
buffer[0] = 'q'; //quorum

pSocket.async_write_some(asio::buffer(buffer),std::bind(&QuorumFetch::_write_handler, this, std::placeholders::_1, std::placeholders::_2));
```

```
template <typename ConstBufferSequence, typename WriteHandler>
ASIO_INITFN_RESULT_TYPE(WriteHandler, void (asio::error_code, std::size_t))
async_write_some(const ConstBufferSequence& buffers, ASIO_MOVE_ARG(WriteHandler) handler)
{
  ASIO_WRITE_HANDLER_CHECK(WriteHandler, handler) type_check;

  asio::async_completion<WriteHandler, void (asio::error_code, std::size_t)> init(handler);

  //同样类似与async_handshake, 用 async_io实现  
  detail::async_io(next_layer_, core_, detail::write_op<ConstBufferSequence>(buffers), init.completion_handler);

  return init.result.get();
}
```

---

async_read_some
```
template <typename MutableBufferSequence, typename ReadHandler>
ASIO_INITFN_RESULT_TYPE(ReadHandler,
    void (asio::error_code, std::size_t))
async_read_some(const MutableBufferSequence& buffers,
    ASIO_MOVE_ARG(ReadHandler) handler)
{
  // If you get an error on the following line it means that your handler does
  // not meet the documented type requirements for a ReadHandler.
  ASIO_READ_HANDLER_CHECK(ReadHandler, handler) type_check;

  asio::async_completion<ReadHandler, void (asio::error_code, std::size_t)> init(handler);

  detail::async_io(next_layer_, core_, detail::read_op<MutableBufferSequence>(buffers),
      init.completion_handler);

  return init.result.get();
}
```

分析代码可以发现，同步相关的用` detail::io `实现，异步则用` detail::async_io `实现。  
` async_handshake ` --- > ` detail_handshake_op `  
` async_write_some `--- > ` detail_write_op `  
` async_read_some ` --- > ` detail_read_op `
