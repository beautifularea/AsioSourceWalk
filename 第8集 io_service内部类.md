接上篇，对内部类看下实现。
```
class executor_type;
friend class executor_type;

class work;
friend class work;

class service;

class strand;
```
#### Executor类
____Executor used to submit functions to an io_context.____
和 io_context 互为friend.
```
class io_context::executor_type
{
  /// Inform the io_context that it has some outstanding work to do.
  /**
   * This function is used to inform the io_context that some work has begun.
   * This ensures that the io_context's run() and run_one() functions do not
   * exit while the work is underway.
   */
  void on_work_started() const ASIO_NOEXCEPT
  {
    io_context_.impl_.work_started();
  }

  /// Inform the io_context that some work is no longer outstanding.
  /**
   * This function is used to inform the io_context that some work has
   * finished. Once the count of unfinished work reaches zero, the io_context
   * is stopped and the run() and run_one() functions may exit.
   */
  void on_work_finished() const ASIO_NOEXCEPT
  {
    io_context_.impl_.work_finished();
  }

  /// Request the io_context to invoke the given function object.
  /**
   * This function is used to ask the io_context to execute the given function
   * object. If the current thread is running the io_context, @c dispatch()
   * executes the function before returning. Otherwise, the function will be
   * scheduled to run on the io_context.
   *
   * @param f The function object to be called. The executor will make a copy
   * of the handler object as required. The function signature of the function
   * object must be: @code void function(); @endcode
   *
   * @param a An allocator that may be used by the executor to allocate the
   * internal storage needed for function invocation.
   */
  template <typename Function, typename Allocator>
  void dispatch(ASIO_MOVE_ARG(Function) f, const Allocator& a) const
  {
    typedef typename decay<Function>::type function_type;

    // Invoke immediately if we are already inside the thread pool.
    if (io_context_.impl_.can_dispatch())
    {
      // Make a local, non-const copy of the function.
      function_type tmp(ASIO_MOVE_CAST(Function)(f));

      //见fenced_block 实现 TODO:
      detail::fenced_block b(detail::fenced_block::full);

      //见asio_handler_invoke_helpers实现
      asio_handler_invoke_helpers::invoke(tmp, tmp);

      return;
    }

    // Allocate and construct an operation to wrap the function.
    typedef detail::executor_op<function_type, Allocator, detail::operation> op;
    typename op::ptr p = { detail::addressof(a), op::ptr::allocate(a), 0 };
    p.p = new (p.v) op(ASIO_MOVE_CAST(Function)(f), a);

    //D:\Tools\asio-1.12.1\include\asio\detail\handler_tracking.hpp
    //宏定义,见handler_tracking实现
    # define ASIO_HANDLER_CREATION(args) \
      asio::detail::handler_tracking::creation args
    ASIO_HANDLER_CREATION((this->context(), *p.p,
        "io_context", &this->context(), 0, "post"));

    io_context_.impl_.post_immediate_completion(p.p, false);
    p.v = p.p = 0;

  }

  /// Request the io_context to invoke the given function object.
  /**
   * This function is used to ask the io_context to execute the given function
   * object. The function object will never be executed inside @c post().
   * Instead, it will be scheduled to run on the io_context.
   *
   * @param f The function object to be called. The executor will make a copy
   * of the handler object as required. The function signature of the function
   * object must be: @code void function(); @endcode
   *
   * @param a An allocator that may be used by the executor to allocate the
   * internal storage needed for function invocation.
   */
  template <typename Function, typename Allocator>
  void post(ASIO_MOVE_ARG(Function) f, const Allocator& a) const
  {
    typedef typename decay<Function>::type function_type;

    // Allocate and construct an operation to wrap the function.
    typedef detail::executor_op<function_type, Allocator, detail::operation> op;
    typename op::ptr p = { detail::addressof(a), op::ptr::allocate(a), 0 };
    p.p = new (p.v) op(ASIO_MOVE_CAST(Function)(f), a);

    ASIO_HANDLER_CREATION((this->context(), *p.p,
          "io_context", &this->context(), 0, "post"));

    io_context_.impl_.post_immediate_completion(p.p, false);
    p.v = p.p = 0;

  }

  /// Request the io_context to invoke the given function object.
  /**
   * This function is used to ask the io_context to execute the given function
   * object. The function object will never be executed inside @c defer().
   * Instead, it will be scheduled to run on the io_context.
   *
   * If the current thread belongs to the io_context, @c defer() will delay
   * scheduling the function object until the current thread returns control to
   * the pool.
   *
   * @param f The function object to be called. The executor will make a copy
   * of the handler object as required. The function signature of the function
   * object must be: @code void function(); @endcode
   *
   * @param a An allocator that may be used by the executor to allocate the
   * internal storage needed for function invocation.
   */
  template <typename Function, typename Allocator>
  void defer(ASIO_MOVE_ARG(Function) f, const Allocator& a) const
  {
    typedef typename decay<Function>::type function_type;

    // Allocate and construct an operation to wrap the function.
    typedef detail::executor_op<function_type, Allocator, detail::operation> op;
    typename op::ptr p = { detail::addressof(a), op::ptr::allocate(a), 0 };
    p.p = new (p.v) op(ASIO_MOVE_CAST(Function)(f), a);

    ASIO_HANDLER_CREATION((this->context(), *p.p,
          "io_context", &this->context(), 0, "defer"));

    io_context_.impl_.post_immediate_completion(p.p, true);
    p.v = p.p = 0;
  }

  /// Determine whether the io_context is running in the current thread.
  /**
   * @return @c true if the current thread is running the io_context. Otherwise
   * returns @c false.
   */
  bool running_in_this_thread() const ASIO_NOEXCEPT;

  private:
  friend class io_context;

  // Constructor.
  explicit executor_type(io_context& i) : io_context_(i) {}

  // The underlying io_context.
  io_context& io_context_;
};
```
`# define ASIO_MOVE_CAST(type) static_cast<const type&> `

注释写的很清楚明白了，主要是` dispatch / post / defer  `有个略微的差异。
` post / defer `实现的唯一区别在 ` ASIO_HANDLER_CREATION `的参数 `post/defer`

#### work
Class to inform the io_context when it has work to do.
被遗弃了，没啥可说的，也简单。

#### service
```
/// Base class for all io_context services.
class io_context::service
  : public execution_context::service
{
  public:
    /// Get the io_context object that owns the service.
    asio::io_context& get_io_context();

  private:
  /// Destroy all user-defined handler objects owned by the service.
  ASIO_DECL virtual void shutdown();

  /// Handle notification of a fork-related event to perform any necessary
/// housekeeping.
/**
 * This function is not a pure virtual so that services only have to
 * implement it if necessary. The default implementation does nothing.
 */
ASIO_DECL virtual void notify_fork(
    execution_context::fork_event event);
};
```
