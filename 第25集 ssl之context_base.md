从一个demo开始讲` context `
```
asio::ssl::context m_ssl_context(asio::ssl::context::sslv23)
// Setting up the context.
m_ssl_context.set_options(boost::asio::ssl::context::default_workarounds
                          | boost::asio::ssl::context::no_sslv2
                          | boost::asio::ssl::context::single_dh_use);
m_ssl_context.set_password_callback([this](std::size_t max_length,asio::ssl::context::password_purpose purpose) -> std::string
                                          {return get_password(max_length, purpose);});
m_ssl_context.use_certificate_chain_file("server.crt");
m_ssl_context.use_private_key_file("server.key", boost::asio::ssl::context::pem);
m_ssl_context.use_tmp_dh_file("dhparams.pem");
```

其中 ` context_base `支持的方法
```
/// Different methods supported by a context.
  enum method
  {
    /// Generic SSL version 2.
    sslv2,

    /// SSL version 2 client.
    sslv2_client,

    /// SSL version 2 server.
    sslv2_server,

    /// Generic SSL version 3.
    sslv3,

    /// SSL version 3 client.
    sslv3_client,

    /// SSL version 3 server.
    sslv3_server,

    /// Generic TLS version 1.
    tlsv1,

    /// TLS version 1 client.
    tlsv1_client,

    /// TLS version 1 server.
    tlsv1_server,

    /// Generic SSL/TLS.
    sslv23, (用到了这个)

    /// SSL/TLS client.
    sslv23_client,

    /// SSL/TLS server.
    sslv23_server,

    /// Generic TLS version 1.1.
    tlsv11,

    /// TLS version 1.1 client.
    tlsv11_client,

    /// TLS version 1.1 server.
    tlsv11_server,

    /// Generic TLS version 1.2.
    tlsv12,

    /// TLS version 1.2 client.
    tlsv12_client,

    /// TLS version 1.2 server.
    tlsv12_server,

    /// Generic TLS.
    tls,

    /// TLS client.
    tls_client,

    /// TLS server.
    tls_server
  };
  ```

  ```
  /// File format types.
  enum file_format
  {
    /// ASN.1 file.
    asn1,

    /// PEM file.
    pem
  }
  ```
```
/// Purpose of PEM password.
  enum password_purpose
  {
    /// The password is needed for reading/decryption.
    for_reading,

    /// The password is needed for writing/encryption.
    for_writing
  };
  ```
注意下`
  /// Bitmask type for SSL options.  
  typedef long options; `
在设置context时候，options是一个bitmask 类型的long数据。
` context_base `使用默认构造函数，并且析构函数是protected。

可以看出该类主要定义了三个数据格式和options类型。
` method / file_format / password_purpose / options `
