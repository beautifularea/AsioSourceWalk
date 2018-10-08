主要用来ssl lib的初始化工作。

```
单例模式
class openssl_init_base
  : private noncopyable
{
protected:
  // Class that performs the actual initialisation.
  class do_init;

  ASIO_DECL static asio::detail::shared_ptr<do_init> instance();
};
```
```
template <bool Do_Init = true>
class openssl_init : private openssl_init_base
{
public:
  // Constructor.
  openssl_init() : ref_(instance())
  {
    using namespace std; // For memmove.

    // Ensure openssl_init::instance_ is linked in.
    openssl_init* tmp = &instance_;

    memmove(&tmp, &tmp, sizeof(openssl_init*));
  }

  // Destructor.
  ~openssl_init()
  {
  }

private:
  static openssl_init instance_;

  asio::detail::shared_ptr<do_init> ref_;
};
```
```
template <bool Do_Init>
openssl_init<Do_Init> openssl_init<Do_Init>::instance_;
```

实现
```
class openssl_init_base::do_init
{
public:
  do_init()
  {
    //目前的版本是：OpenSSL 1.0.1f 6 Jan 2014
#if (OPENSSL_VERSION_NUMBER < 0x10100000L)
    ::SSL_library_init();
    ::SSL_load_error_strings();        
    ::OpenSSL_add_all_algorithms();

    mutexes_.resize(::CRYPTO_num_locks());

    for (size_t i = 0; i < mutexes_.size(); ++i)
      mutexes_[i].reset(new asio::detail::mutex);

    ::CRYPTO_set_locking_callback(&do_init::openssl_locking_func);
#endif // (OPENSSL_VERSION_NUMBER < 0x10100000L)

asio::detail::shared_ptr<openssl_init_base::do_init>
openssl_init_base::instance()
{
  static asio::detail::shared_ptr<do_init> init(new do_init);
  return init;
}

```
