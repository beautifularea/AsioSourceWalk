在[第28集]()的async_handshake中使用.

```
handshake_op  
write_op  
read_op
```
实现模式一样，都是函数类。通过` engine `参数实现读写操作。

```
template <typename ConstBufferSequence>
class write_op
{
public:
  write_op(const ConstBufferSequence& buffers)
    : buffers_(buffers)
  {
  }

  engine::want operator()(engine& eng, asio::error_code& ec, std::size_t& bytes_transferred) const
  {
    asio::const_buffer buffer = asio::detail::buffer_sequence_adapter<asio::const_buffer,  ConstBufferSequence>::first(buffers_);

    //把buffer内容，写道engine中。
    return eng.write(buffer, ec, bytes_transferred);
  }
};
```

```
engine::want operator()(engine& eng,
    asio::error_code& ec,
    std::size_t& bytes_transferred) const
{
  bytes_transferred = 0;
  return eng.handshake(type_, ec);
}

engine::want operator()(engine& eng,
    asio::error_code& ec,
    std::size_t& bytes_transferred) const
{
  asio::mutable_buffer buffer = asio::detail::buffer_sequence_adapter<asio::mutable_buffer, MutableBufferSequence>::first(buffers_);

  return eng.read(buffer, ec, bytes_transferred);
}
```

```
template <typename Handler>
void call_handler(Handler& handler,
    const asio::error_code& ec,
    const std::size_t& bytes_transferred) const
{
  handler(ec, bytes_transferred);
}
```
