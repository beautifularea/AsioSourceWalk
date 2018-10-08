直接看实现
```
namespace asio {
namespace error {
namespace detail {

class ssl_category : public asio::error_category
{
public:
  const char* name() const ASIO_ERROR_CATEGORY_NOEXCEPT
  {
    return "asio.ssl";
  }

  std::string message(int value) const
  {
    //openssl 注释如下:
    /*ERR_error_string() generates a human-readable string representing the error code e, and places it at buf. buf must be at least 120 bytes long. If buf is NULL, the error string is placed in a static buffer.
    */
    const char* s = ::ERR_reason_error_string(value);
    return s ? s : "asio.ssl error";
  }
};

} // namespace detail

const asio::error_category& get_ssl_category()
{
  static detail::ssl_category instance;
  return instance;
}

} // namespace error
} // namespace asio
```

看` error_category `,
```
typedef std::error_category error_category;
```

```
namespace asio {
namespace error {

enum ssl_errors
{
  // Error numbers are those produced by openssl.
};

extern ASIO_DECL
const asio::error_category& get_ssl_category();

static const asio::error_category&
  ssl_category ASIO_UNUSED_VARIABLE
  = asio::error::get_ssl_category();

} // namespace error
} // namespace asio
```
