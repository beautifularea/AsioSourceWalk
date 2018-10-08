接上一集。

直接看 ` engine.ipp `实现，忽略错误判断，平台判断等代码。

```
engine::engine(SSL_CTX* context)
  : ssl_(::SSL_new(context))
{
  ::SSL_set_mode(ssl_, SSL_MODE_ENABLE_PARTIAL_WRITE);
  ::SSL_set_mode(ssl_, SSL_MODE_ACCEPT_MOVING_WRITE_BUFFER);

  ::BIO* int_bio = 0;
  ::BIO_new_bio_pair(&int_bio, 0, &ext_bio_, 0);
  ::SSL_set_bio(ssl_, int_bio, int_bio);
}

engine::~engine()
{
  if (SSL_get_app_data(ssl_))
  {
    delete static_cast<verify_callback_base*>(SSL_get_app_data(ssl_));
    SSL_set_app_data(ssl_,
      0);
  }

  ::BIO_free(ext_bio_);
  ::SSL_free(ssl_);
}
```

```
SSL* engine::native_handle()
{
  return ssl_;
}
```

提一句，在` stream `中，` set_verify_mode `的实现就是调用 engine 的方法
```
asio::error_code engine::set_verify_mode(
    verify_mode v, asio::error_code& ec)
{
  ::SSL_set_verify(ssl_, v, ::SSL_get_verify_callback(ssl_));

  ec = asio::error_code();
  return ec;
}
```

handshake部分
```
engine::want engine::handshake(
    stream_base::handshake_type type, asio::error_code& ec)
{
  return perform((type == asio::ssl::stream_base::client)
      ? &engine::do_connect : &engine::do_accept, 0, 0, ec, 0);
}

int engine::do_accept(void*, std::size_t)
{
  return ::SSL_accept(ssl_);
}

int engine::do_connect(void*, std::size_t)
{
  return ::SSL_connect(ssl_);
}
```

write部分
```
engine::want engine::write(const asio::const_buffer& data, asio::error_code& ec, std::size_t& bytes_transferred)
{
  if (data.size() == 0)
  {
    ec = asio::error_code();
    return engine::want_nothing;
  }

  return perform(&engine::do_write, const_cast<void*>(data.data()), data.size(), ec, &bytes_transferred);
}

int engine::do_write(void* data, std::size_t length)
{
  return ::SSL_write(ssl_, data, length < INT_MAX ? static_cast<int>(length) : INT_MAX);
}
```

read部分
```
engine::want engine::read(const asio::mutable_buffer& data, asio::error_code& ec, std::size_t& bytes_transferred)
{
  if (data.size() == 0)
  {
    ec = asio::error_code();
    return engine::want_nothing;
  }

  return perform(&engine::do_read, data.data(), data.size(), ec, &bytes_transferred);
}

int engine::do_read(void* data, std::size_t length)
{
  return ::SSL_read(ssl_, data, length < INT_MAX ? static_cast<int>(length) : INT_MAX);
}
```

最后一个工具函数
```
engine::want engine::perform(int (engine::* op)(void*, std::size_t),
    void* data, std::size_t length, asio::error_code& ec,
    std::size_t* bytes_transferred)
{
  std::size_t pending_output_before = ::BIO_ctrl_pending(ext_bio_);
  ::ERR_clear_error();

  int result = (this->*op)(data, length);

  int ssl_error = ::SSL_get_error(ssl_, result);
  int sys_error = static_cast<int>(::ERR_get_error());
  std::size_t pending_output_after = ::BIO_ctrl_pending(ext_bio_);

  if (result > 0 && bytes_transferred)
    *bytes_transferred = static_cast<std::size_t>(result);

  //....
  return want....;
}
```

```
asio::mutable_buffer engine::get_output(const asio::mutable_buffer& data)
{
  int length = ::BIO_read(ext_bio_,  data.data(), static_cast<int>(data.size()));

  return asio::buffer(data, length > 0 ? static_cast<std::size_t>(length) : 0);
}

asio::const_buffer engine::put_input(const asio::const_buffer& data)
{
  int length = ::BIO_write(ext_bio_, data.data(), static_cast<int>(data.size()));

  return asio::buffer(data + (length > 0 ? static_cast<std::size_t>(length) : 0));
}
```
