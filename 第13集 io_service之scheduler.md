接[第7集]()，` io_service `内部真实实现是` class scheduler `,现在开始。  

先看` class scheduler `的声明部分
```
namespace asio {
namespace detail {

struct scheduler_thread_info;

class scheduler
  : public execution_context_service_base<scheduler>,
    public thread_context //见thread_context实现
{
public:
  //见scheduler_operation实现
  typedef scheduler_operation operation;

  //构造函数
  ASIO_DECL scheduler(asio::execution_context& ctx,
      int concurrency_hint = 0);

  //销毁service持有的用户handler
  ASIO_DECL void shutdown();

  //run event loop
  ASIO_DECL std::size_t run(asio::error_code& ec);

  //run 一次，或者超时，中断退出
  ASIO_DECL std::size_t wait_one(long usec, asio::error_code& ec);

  //poll 非阻塞
  ASIO_DECL std::size_t poll(asio::error_code& ec);

  //终端event loop
  ASIO_DECL void stop();

  //判断是否stop
  ASIO_DECL bool stopped() const;

  //通知一个任务开始了
  void work_started()
  {
    ++outstanding_work_;
  }

  //对将要执行的work_finished的compensate???
  ASIO_DECL void compensating_work_started();

  //
  void work_finished()
  {
    if (--outstanding_work_ == 0)
      stop();
  }

  //提供给外界，该io_service是否运行在当前thread
  bool can_dispatch()
  {
    return thread_call_stack::contains(this) != 0;
  }

  ASIO_DECL void post_immediate_completion(
    operation* op, bool is_continuation);

  // 立即返回， work_started() has not yet been called
  ASIO_DECL void post_deferred_completion(operation* op);

  // Assumes that work_started() was previously called for each operation.
  ASIO_DECL void post_deferred_completions(op_queue<operation>& ops);

  // Enqueue the given operation following a failed attempt to dispatch the
  // operation for immediate invocation.
  ASIO_DECL void do_dispatch(operation* op);

private:
  // The mutex type used by this scheduler.
  typedef conditionally_enabled_mutex mutex;

  // The event type used by this scheduler.
  typedef conditionally_enabled_event event;

  // Structure containing thread-specific data.
  typedef scheduler_thread_info thread_info;

  // 内部类，一般是helper类
  struct task_cleanup;
  friend struct task_cleanup;

  // 内部类，一般是helper类
  struct work_cleanup;
  friend struct work_cleanup;

  // Mutex to protect access to internal data.
  mutable mutex mutex_;

  // Event to wake up blocked threads.
  event wakeup_event_;

  // The task to be run by this service.
  reactor* task_;

  // The count of unfinished work.
  atomic_count outstanding_work_;

  // The queue of handlers that are ready to be delivered.
  op_queue<operation> op_queue_;

  // Operation object to represent the position of the task in the queue.
  struct task_operation : operation
  {
    task_operation() : operation(0) {}
  } task_operation_;
};
```

---

主要方法实现
```
void scheduler::shutdown()
{
  //局部锁
  mutex::scoped_lock lock(mutex_);
  shutdown_ = true;
  lock.unlock();

  // Destroy handler objects.
  while (!op_queue_.empty())
  {
    operation* o = op_queue_.front();
    op_queue_.pop();
    if (o != &task_operation_) //正在进行的operation?
      o->destroy();
  }

  // Reset to initial state.
  task_ = 0;
}
```

run 一下吧
```
std::size_t scheduler::run(asio::error_code& ec)
{
  ec = asio::error_code();

  //如果没有未完成的work，则stop
  if (outstanding_work_ == 0)
  {
    stop();
    return 0;
  }

  //初始化thread_call_stack
  thread_info this_thread;
  this_thread.private_outstanding_work = 0;
  thread_call_stack::context ctx(this, this_thread);

  //加锁
  mutex::scoped_lock lock(mutex_);

  std::size_t n = 0;
  for (; do_run_one(lock, this_thread, ec); lock.lock()) //调用了do_run_one
    if (n != (std::numeric_limits<std::size_t>::max)())
      ++n;
  return n;
}
```

run_one
```
std::size_t scheduler::run_one(asio::error_code& ec)
{
  ec = asio::error_code();
  if (outstanding_work_ == 0)
  {
    stop();
    return 0;
  }

  thread_info this_thread;
  this_thread.private_outstanding_work = 0;
  thread_call_stack::context ctx(this, this_thread);

  mutex::scoped_lock lock(mutex_);

  //代码一样，此处之调用了一次do_run_one
  return do_run_one(lock, this_thread, ec);
}
```

wait_one
```
std::size_t scheduler::wait_one(long usec, asio::error_code& ec)
{
  ec = asio::error_code();
  if (outstanding_work_ == 0)
  {
    stop();
    return 0;
  }

  thread_info this_thread;
  this_thread.private_outstanding_work = 0;
  thread_call_stack::context ctx(this, this_thread);

  mutex::scoped_lock lock(mutex_);

  return do_wait_one(lock, this_thread, usec, ec);
}
```
poll 也简单
```
std::size_t scheduler::poll(asio::error_code& ec)
{
  ec = asio::error_code();
  if (outstanding_work_ == 0)
  {
    stop();
    return 0;
  }

  thread_info this_thread;
  this_thread.private_outstanding_work = 0;
  thread_call_stack::context ctx(this, this_thread);

  mutex::scoped_lock lock(mutex_);

  std::size_t n = 0;
  for (; do_poll_one(lock, this_thread, ec); lock.lock())
    if (n != (std::numeric_limits<std::size_t>::max)())
      ++n;
  return n;
}
```

post四函数
```
void scheduler::compensating_work_started()
{
  //private_outstanding_work 是在做什么
  thread_info_base* this_thread = thread_call_stack::contains(this);
  ++static_cast<thread_info*>(this_thread)->private_outstanding_work;
}

void scheduler::post_immediate_completion(
    scheduler::operation* op, bool is_continuation)
{
  work_started();
  mutex::scoped_lock lock(mutex_);

  op_queue_.push(op);

  //这是干什么
  wake_one_thread_and_unlock(lock);
}

void scheduler::post_deferred_completion(scheduler::operation* op)
{
  mutex::scoped_lock lock(mutex_);
  op_queue_.push(op);
  wake_one_thread_and_unlock(lock);
}

void scheduler::post_deferred_completions(
    op_queue<scheduler::operation>& ops)
{
  if (!ops.empty())
  {
    mutex::scoped_lock lock(mutex_);
    op_queue_.push(ops);
    wake_one_thread_and_unlock(lock);
  }
}

void scheduler::do_dispatch(
    scheduler::operation* op)
{
  work_started();
  mutex::scoped_lock lock(mutex_);
  op_queue_.push(op);
  wake_one_thread_and_unlock(lock);
}

void scheduler::abandon_operations(
    op_queue<scheduler::operation>& ops)
{
  //ops2直接销毁
  op_queue<scheduler::operation> ops2;
  ops2.push(ops);
}
```

大boss
```
std::size_t scheduler::do_run_one(mutex::scoped_lock& lock,
    scheduler::thread_info& this_thread,
    const asio::error_code& ec)
{
  while (!stopped_)
  {
    if (!op_queue_.empty())
    {
      // Prepare to execute first handler from queue.
      operation* o = op_queue_.front();
      op_queue_.pop();
      bool more_handlers = (!op_queue_.empty());

      if (o == &task_operation_)
      {
        task_interrupted_ = more_handlers;

        if (more_handlers && !one_thread_)
          wakeup_event_.unlock_and_signal_one(lock);
        else
          lock.unlock();

        task_cleanup on_exit = { this, &lock, &this_thread };
        (void)on_exit;

        // Run the task. May throw an exception. Only block if the operation
        // queue is empty and we're not polling, otherwise we want to return
        // as soon as possible.
        task_->run(more_handlers ? 0 : -1, this_thread.private_op_queue);
      }
      else
      {
        std::size_t task_result = o->task_result_;

        if (more_handlers && !one_thread_)
          wake_one_thread_and_unlock(lock);
        else
          lock.unlock();

        // Ensure the count of outstanding work is decremented on block exit.
        work_cleanup on_exit = { this, &lock, &this_thread };
        (void)on_exit;

        // Complete the operation. May throw an exception. Deletes the object.
        o->complete(this, ec, task_result);

        return 1;
      }
    }
    else
    {
      wakeup_event_.clear(lock);
      wakeup_event_.wait(lock);
    }
  }

  return 0;
}
```

看task实现。
```
void scheduler::wake_one_thread_and_unlock(
    mutex::scoped_lock& lock)
{
  if (!wakeup_event_.maybe_unlock_and_signal_one(lock))
  {
    if (!task_interrupted_ && task_)
    {
      task_interrupted_ = true;
      task_->interrupt();
    }
    lock.unlock();
  }
}
```
