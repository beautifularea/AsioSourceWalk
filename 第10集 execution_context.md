` execution_context `是 ` io_context `的父类，源码注释写的太详细了，不翻译。
直接看实现
```
class execution_context
  : private noncopyable
{
public:
  //忽略这两个内部类的实现分析
  class id;
  class service; //next 节点的链表

  /// Fork-related event notifications.
  enum fork_event
  {
    /// Notify the context that the process is about to fork.
    fork_prepare,

    /// Notify the context that the process has forked and is the parent.
    fork_parent,

    /// Notify the context that the process has forked and is the child.
    fork_child
  };
private:
  // The service registry.
  asio::detail::service_registry* service_registry_;
};
```
` service_registry_ `是定义在父类中，对照 [第9集]() 查看。

` make_service `
```
template <typename Service>
Service& make_service(execution_context& e)
{
  detail::scoped_ptr<Service> svc(new Service(e));
  e.service_registry_->template add_service<Service>(svc.get());
  Service& result = *svc;
  svc.release();
  return result;
}
```

---

****主要方法实现****
```
execution_context::execution_context()
  : service_registry_(new asio::detail::service_registry(*this))
{
}

execution_context::~execution_context()
{
  shutdown();
  destroy();
  delete service_registry_;
}

void execution_context::shutdown()
{
  service_registry_->shutdown_services();
}

void execution_context::destroy()
{
  service_registry_->destroy_services();
}
```
可以发现，` execution_context ` 又是封装了一层 ` asio::detail::service_registry `
