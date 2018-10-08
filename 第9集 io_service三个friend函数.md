内容暂定。
```
namespace asio {

template <typename Service>
inline Service& use_service(io_context& ioc)
{
  // Check that Service meets the necessary type requirements.
  (void)static_cast<execution_context::service*>(static_cast<Service*>(0));
  (void)static_cast<const execution_context::id*>(&Service::id);

  return ioc.service_registry_->template use_service<Service>(ioc);
}

template <>
inline detail::io_context_impl& use_service<detail::io_context_impl>(
    io_context& ioc)
{
  return ioc.impl_;
}

template <typename Service>
inline void add_service(execution_context& e, Service* svc)
{
  // Check that Service meets the necessary type requirements.
  (void)static_cast<execution_context::service*>(static_cast<Service*>(0));

  e.service_registry_->template add_service<Service>(svc);
}

template <typename Service>
inline bool has_service(execution_context& e)
{
  // Check that Service meets the necessary type requirements.
  (void)static_cast<execution_context::service*>(static_cast<Service*>(0));

  return e.service_registry_->template has_service<Service>();
}

} // namespace asio
```

---

****1****
```
template <typename Service>
Service& service_registry::use_service()
{
  execution_context::service::key key;
  init_key<Service>(key, 0);
  factory_type factory = &service_registry::create<Service, execution_context>;
  return *static_cast<Service*>(do_use_service(key, factory, &owner_));
}

template <typename Service>
Service& service_registry::use_service(io_context& owner)
{
  execution_context::service::key key;
  init_key<Service>(key, 0);
  factory_type factory = &service_registry::create<Service, io_context>;
  return *static_cast<Service*>(do_use_service(key, factory, &owner));
}

template <typename Service>
void service_registry::add_service(Service* new_service)
{
  execution_context::service::key key;
  init_key<Service>(key, 0);
  return do_add_service(key, new_service);
}

template <typename Service>
bool service_registry::has_service() const
{
  execution_context::service::key key;
  init_key<Service>(key, 0);
  return do_has_service(key);
}
```

---


****IMPL****
```

execution_context::service* service_registry::do_use_service(
    const execution_context::service::key& key,
    factory_type factory, void* owner)
{
  asio::detail::mutex::scoped_lock lock(mutex_);

  // First see if there is an existing service object with the given key.
  execution_context::service* service = first_service_;
  while (service)
  {
    if (keys_match(service->key_, key))
      return service;
    service = service->next_;
  }

  // Create a new service object. The service registry's mutex is not locked
  // at this time to allow for nested calls into this function from the new
  // service's constructor.
  lock.unlock();
  auto_service_ptr new_service = { factory(owner) };
  new_service.ptr_->key_ = key;
  lock.lock();

  // Check that nobody else created another service object of the same type
  // while the lock was released.
  service = first_service_;
  while (service)
  {
    if (keys_match(service->key_, key))
      return service;
    service = service->next_;
  }

  // Service was successfully initialised, pass ownership to registry.
  new_service.ptr_->next_ = first_service_;
  first_service_ = new_service.ptr_;
  new_service.ptr_ = 0;
  return first_service_;
}

void service_registry::do_add_service(
    const execution_context::service::key& key,
    execution_context::service* new_service)
{
  if (&owner_ != &new_service->context())
    asio::detail::throw_exception(invalid_service_owner());

  asio::detail::mutex::scoped_lock lock(mutex_);

  // Check if there is an existing service object with the given key.
  execution_context::service* service = first_service_;
  while (service)
  {
    if (keys_match(service->key_, key))
      asio::detail::throw_exception(service_already_exists());
    service = service->next_;
  }

  // Take ownership of the service object.
  new_service->key_ = key;
  new_service->next_ = first_service_;
  first_service_ = new_service;
}

bool service_registry::do_has_service(
    const execution_context::service::key& key) const
{
  asio::detail::mutex::scoped_lock lock(mutex_);

  execution_context::service* service = first_service_;
  while (service)
  {
    if (keys_match(service->key_, key))
      return true;
    service = service->next_;
  }

  return false;
}
```
