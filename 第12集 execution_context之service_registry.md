接[第10集]()。

` service_registry `类成员函数不多，如下：
```
class service_registry
  : private noncopyable
{
public:
  // Constructor.
  ASIO_DECL service_registry(execution_context& owner);

  // Destructor.
  ASIO_DECL ~service_registry();

  // Shutdown all services.
  ASIO_DECL void shutdown_services();

  // Destroy all services.
  ASIO_DECL void destroy_services();

  ASIO_DECL void notify_fork(execution_context::fork_event fork_ev);

  //TODO: 注意着三个函数，和[第9集]()对比着看
  template <typename Service>
  Service& use_service();

  template <typename Service>
  Service& use_service(io_context& owner);

  template <typename Service>
  void add_service(Service* new_service);

  template <typename Service>
  bool has_service() const;

private:
  template <typename Service>
  static void init_key(execution_context::service::key& key, ...);

  // Factory function for creating a service instance.
  template <typename Service, typename Owner>
  static execution_context::service* create(void* owner);

private:
  // Mutex to protect access to internal data.
  mutable asio::detail::mutex mutex_;

  // The owner of this service registry and the services it contains.
  execution_context& owner_;

  // The first service in the list of contained services.
  execution_context::service* first_service_;
```

---

```
void service_registry::destroy_services()
{
  while (first_service_)
  {
    execution_context::service* next_service = first_service_->next_;
    destroy(first_service_);
    first_service_ = next_service;
  }
}
```
```
void service_registry::notify_fork(execution_context::fork_event fork_ev)
{
  // Make a copy of all of the services while holding the lock. We don't want
  // to hold the lock while calling into each service, as it may try to call
  // back into this class.
  std::vector<execution_context::service*> services;
  {
    asio::detail::mutex::scoped_lock lock(mutex_);
    execution_context::service* service = first_service_;
    while (service)
    {
      services.push_back(service);
      service = service->next_;
    }
  }

  // If processing the fork_prepare event, we want to go in reverse order of
  // service registration, which happens to be the existing order of the
  // services in the vector. For the other events we want to go in the other
  // direction.
  std::size_t num_services = services.size();
  if (fork_ev == execution_context::fork_prepare)
    for (std::size_t i = 0; i < num_services; ++i)
      services[i]->notify_fork(fork_ev);
  else
    for (std::size_t i = num_services; i > 0; --i)
      services[i - 1]->notify_fork(fork_ev);
}

template <typename Service, typename Owner>
execution_context::service* service_registry::create(void* owner)
{
  return new Service(*static_cast<Owner*>(owner));
}
```
代码加注释写的很详细，至此，` execution_context.service_registry `完毕。
