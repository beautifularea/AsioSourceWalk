` context `类主要是封装了openssl中关于` SSL_CTX `封装，继承自 ` context_base `和` noncopyable ` ,属于多继承，C++是支持的，但是要注意一个钻石继承的问题.
```
class context
  : public context_base,
    private noncopyable
```

构造方法
```
ASIO_DECL explicit context(method m);
ASIO_DECL context(context&& other);
ASIO_DECL context& operator=(context&& other);
```

清理options的方法
```
ASIO_DECL void clear_options(options o);
```
设置options的方法，参数为bitmask。
```
ASIO_DECL void set_options(options o);
```

设置验证模式，参考[第23集]()
```
ASIO_DECL void set_verify_mode(verify_mode v);
```
验证证书链的深度
```
ASIO_DECL void set_verify_depth(int depth);
```
```
template <typename VerifyCallback>
  void set_verify_callback(VerifyCallback callback);
```
```
ASIO_DECL void load_verify_file(const std::string& filename);
```
```
ASIO_DECL void add_certificate_authority(const const_buffer& ca);
```

Configures the context to use the default directories for finding certification authority certificates.
```
ASIO_DECL void set_default_verify_paths();
```
```
ASIO_DECL void add_verify_path(const std::string& path);
```

Use a certificate from a memory buffer.
```
ASIO_DECL void use_certificate(const const_buffer& certificate, file_format format);
```
Use a certificate from a memory buffer.
```
ASIO_DECL ASIO_SYNC_OP_VOID use_certificate(
      const const_buffer& certificate, file_format format,
      asio::error_code& ec);
```
Use a certificate from a file.
```
ASIO_DECL void use_certificate_file(const std::string& filename, file_format format);
```

主要private成员函数
```
private:
  struct bio_cleanup;
  struct x509_cleanup;
  struct evp_pkey_cleanup;
  struct rsa_cleanup;
  struct dh_cleanup;
```

---

实现
```
template <typename VerifyCallback>
ASIO_SYNC_OP_VOID context::set_verify_callback(
    VerifyCallback callback, asio::error_code& ec)
{
  do_set_verify_callback(
      new detail::verify_callback<VerifyCallback>(callback), ec);
  ASIO_SYNC_OP_VOID_RETURN(ec);
}
```

构造函数
```
context::context(context::method m)
  : handle_(0)
{
  //清楚error
  ::ERR_clear_error();

  switch(m)
  {
  case context::sslv23:
      handle_ = ::SSL_CTX_new(::SSLv23_method());
      break;

  default:
    handle_ = ::SSL_CTX_new(0);
    break;
  }

  if (handle_ == 0)
  {
    asio::error_code ec(
        static_cast<int>(::ERR_get_error()),
        asio::error::get_ssl_category());

    asio::detail::throw_error(ec, "context");
  }

  set_options(no_compression);
}
```

```
typedef SSL_CTX* native_handle_type;
native_handle_type handle_;
context::native_handle_type context::native_handle()
{
  return handle_;
}
```

```

ASIO_SYNC_OP_VOID context::set_options(context::options o, asio::error_code& ec)
{
  ::SSL_CTX_set_options(handle_, o);

  ec = asio::error_code();
  ASIO_SYNC_OP_VOID_RETURN(ec);
}
```

```
ASIO_SYNC_OP_VOID context::set_verify_mode(
    verify_mode v, asio::error_code& ec)
{
  ::SSL_CTX_set_verify(handle_, v, ::SSL_CTX_get_verify_callback(handle_));

  ec = asio::error_code();
  ASIO_SYNC_OP_VOID_RETURN(ec);
}
```

```
ASIO_SYNC_OP_VOID context::set_verify_depth(
    int depth, asio::error_code& ec)
{
  ::SSL_CTX_set_verify_depth(handle_, depth);

  ec = asio::error_code();
  ASIO_SYNC_OP_VOID_RETURN(ec);
}
```

```
ASIO_SYNC_OP_VOID context::load_verify_file(
    const std::string& filename, asio::error_code& ec)
{
  ::ERR_clear_error();

  if (::SSL_CTX_load_verify_locations(handle_, filename.c_str(), 0) != 1)
  {
    ec = asio::error_code(
        static_cast<int>(::ERR_get_error()),
        asio::error::get_ssl_category());
    ASIO_SYNC_OP_VOID_RETURN(ec);
  }

  ec = asio::error_code();
  ASIO_SYNC_OP_VOID_RETURN(ec);
}
```

```
ASIO_SYNC_OP_VOID context::add_verify_path(
    const std::string& path, asio::error_code& ec)
{
  ::ERR_clear_error();

  if (::SSL_CTX_load_verify_locations(handle_, 0, path.c_str()) != 1)
  {
    ec = asio::error_code(
        static_cast<int>(::ERR_get_error()),
        asio::error::get_ssl_category());
    ASIO_SYNC_OP_VOID_RETURN(ec);
  }

  ec = asio::error_code();
  ASIO_SYNC_OP_VOID_RETURN(ec);
}
```

剩下的方法都是对OPenssl的封装了，没啥可分析的，忽略就不分析了。
